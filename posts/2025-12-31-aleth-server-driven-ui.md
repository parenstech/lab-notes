Title: Aleth: Server-Driven UI Without Client Lies
Date: 2025-12-31
Tags: clojure, clojurescript, ui, server-driven

The ancient Greeks had a word for truth: *aletheia* - literally "un-concealment." Truth wasn't something you asserted; it was something you revealed by stripping away what hid it.

This is the premise behind [Aleth](https://github.com/parenstech/aleth), a new server-driven UI library for Clojure. The core bet: **no client-side computation**. The server owns all truth. The client is a pure projection - it shows what it's told, nothing more.

In an age of increasingly complex frontend frameworks, this sounds almost naive. But there's a specific context where it makes profound sense: systems where humans supervise and AI writes the code.

## The Architectural Bet

Modern web UIs are distributed systems hiding inside your browser. State lives in Redux stores, component local state, URL parameters, localStorage, and server databases. When something goes wrong, you triangulate across all of them.

Aleth eliminates this by making a radical constraint: the client cannot compute.

Where [Datastar](https://data-star.dev/) allows `$count + 1` in attributes, Aleth has no expressions. Where React components derive state locally, Aleth demands the server send exactly what to display. The client becomes a terminal - it receives instructions and renders them.

```clojure
;; Server computes everything
(defn increment-handler [req]
  (sse-response
    (fn [sse]
      (let [{:keys [count]} (signals req)
            new-count (inc count)]
        (signals! sse {:count new-count})
        (patch! sse "#counter" [:div {:id "counter"} [:span (str new-count)]])
        (close! sse)))))
```

The client receives two things: a signal update (`{:count 6}`) and a DOM patch. It applies both. No logic, no decisions, no opportunities to diverge from truth.

## How It Works

Aleth uses Transit+JSON over Server-Sent Events. The wire protocol has three operations:

- **patch** - Update DOM via hiccup
- **signals** - Update reactive client state
- **execute** - Run JavaScript (the escape hatch, discussed later)

The server sends hiccup, the client morphs it into the DOM using [Idiomorph](https://github.com/bigskysoftware/idiomorph). Signal changes trigger reactive bindings. That's the entire runtime.

```clojure
;; Server renders initial page
(defn counter-page [count]
  [:html
   [:body
    [:div (a/signals {:count count})
     [:span (a/text :count) (str count)]
     [:button (a/action "/increment") "+"]
     [:button (a/action "/decrement") "-"]]]])
```

The `a/signals`, `a/text`, and `a/action` helpers emit `data-*` attributes. When Aleth's JavaScript loads, it discovers these attributes and wires them up. Click the button, POST to `/increment`, receive SSE response, update DOM. The HTML works before JavaScript loads; Aleth progressively enhances it.

## What's Good About This

**Determinism.** `state -> UI` is a pure function. Same signals, same render. No "it works if you refresh," no race conditions between client and server state.

**Testability.** You can property-test your entire UI:

```clojure
(defspec render-is-deterministic 100
  (prop/for-all [state (mg/generator signals-schema)]
    (= (render state) (render state))))
```

Visual regression testing becomes trivial - render HTML, snapshot, compare. No client timing issues, no flaky tests.

**Observability.** Everything is inspectable. Aleth includes devtools (SSE inspector, signal viewer, schema panel) that show every state transition. Debug by reading the event stream, not by reproducing timing-dependent bugs.

**Schema validation.** [Malli](https://github.com/metosin/malli) validates signals on both ends. Invalid states are rejected, not silently accepted.

**Clean API.** The library exports a single entry point with sensible helpers. Hot reload works correctly (using WeakSet to prevent duplicate bindings). The devtools use Shadow DOM for style isolation.

## What's Concerning

This is an early-stage library with some serious issues.

**No tests.** For a library whose core value proposition is correctness and determinism, this is ironic. The spec includes property-based test examples, but the implementation has none.

**Memory leaks.** Signal watchers are never cleaned up. In a long-running session, this will accumulate. Multiple locations in the codebase add watchers without corresponding removal logic.

**The `execute` escape hatch.** The wire protocol includes an `execute` operation that runs arbitrary JavaScript via `js/eval`. This is a security risk in any production system, even if intended only for debugging. It's the kind of backdoor that gets forgotten about.

**No recovery after SSE failure.** The connection has retry logic with exponential backoff, but there's no recovery mechanism once retries exhaust. The client just stops.

**Stale DOM references.** After morphing, references to old DOM nodes aren't invalidated. This can cause silent failures in bindings.

**URL injection.** The redirect handler doesn't sanitize URLs, creating a potential vector for malicious redirects.

## Who Should Consider This

Aleth is right for:

- **Admin panels and internal tools** - where latency to the server is low and instant UI response isn't critical
- **AI-supervised development** - where you want the simplest possible model for an LLM to reason about
- **Forms-heavy applications** - where most interactions are "submit and wait for server response"
- **Dashboards with real-time data** - the SSE broadcast pattern handles this cleanly

Aleth is wrong for:

- **Consumer applications** requiring instant responsiveness
- **Offline-first or PWA** - the library explicitly doesn't support offline (server owns truth)
- **Complex interactions** like drag-and-drop, real-time drawing, gaming
- **Production use** - the current implementation has too many gaps

The spec is honest about this: "For offline-first, consider [Fulcro](https://fulcro.fulcrologic.com/)."

## The Latency Trade-off

Every click goes to the server and back. On a local network, this is imperceptible. Over a 200ms round-trip, it's noticeable. The library's answer is "optimize the server," not "add client computation."

This is philosophically consistent but practically limiting. There are interactions where even 50ms of latency feels broken - typing in a search box, dragging to reorder a list, hovering to preview. Aleth doesn't try to solve these cases.

The comparison to [Phoenix LiveView](https://hexdocs.pm/phoenix_live_view/) is instructive. LiveView makes the same server-centric bet but in Elixir's ecosystem where lightweight processes and low-latency WebSockets are first-class. Aleth is swimming upstream against browser realities.

## Conclusion

Aleth represents an interesting point in the design space: what if we maximally simplified the client at the cost of server round-trips? For the right use cases - internal tools, AI-assisted development, admin interfaces - this trade-off makes sense.

The ideas are sound. The implementation needs work.

If you're building something in the sweet spot (low-latency server, forms-heavy workflow, prioritizing correctness over responsiveness), Aleth is worth watching. If you need production-ready today, wait for the tests, the memory leak fixes, and the security audit.

The name promises truth as unconcealment. The library isn't there yet - but the architecture points in an interesting direction.
