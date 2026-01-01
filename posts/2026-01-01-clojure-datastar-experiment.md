Title: The Clojure Datastar Experiment: When Language Loyalty Becomes a Trap
Date: 2026-01-01
Tags: clojure, clojurescript, datastar, experiment

I spent weeks building what I thought would be the obvious next step: a Clojure-native version of [Datastar](https://data-star.dev/). Same architecture, but with Clojure expressions instead of JavaScript, Transit instead of JSON, hiccup helpers instead of raw HTML attributes.

It was a waste of time. Here's why.

## The Siren Song

Datastar is a ~10KB JavaScript library for server-driven UI. The server owns all state. The client is a thin rendering layer. Click a button, POST to the server, receive SSE updates, morph the DOM. No React, no Redux, no client-side state management.

For a Clojure developer, there's a natural next question: *What if we had this, but in Clojure?*

The pitch writes itself:
- Clojure expressions instead of JavaScript (`(toggle! :_open)` instead of `$_open = !$_open`)
- Transit for richer data types
- Hiccup helpers for ergonomic server rendering
- ClojureScript on the client for consistency

I called the experiment [Aleth](https://github.com/parenstech/aleth) and got to work.

## What I Built

The core took about 2,000 lines across client and server modules:

```
Client modules:
  core.cljs    482 lines  (bindings, discovery)
  eval.cljs    308 lines  (expression evaluator)
  local.cljs   229 lines  (local signals)
  signals.cljs 164 lines  (signal store)
  sse.cljs     194 lines  (SSE client)
  morph.cljs   160 lines  (DOM morphing wrapper)

Server modules:
  core.clj     117 lines
  hiccup.clj   290 lines
  sse.clj      211 lines

Total: 2,155 lines
```

The expression evaluator alone - allowing Clojure syntax in DOM attributes - required:
- A whitelist of ~50 safe functions
- Special form handling (if, when, let, do, and, or, cond)
- Symbol resolution against signal maps
- Reactive expression watching

When the "server owns truth" principle made simple UI patterns sluggish (every dropdown toggle required a server round-trip), I implemented local signals - Datastar's solution to the same problem:

```clojure
;; Aleth's attempted local signals
[:div (a/local {:_open false})
 [:button (a/on-local :click '(toggle! :_open)) "Toggle"]
 [:div (a/show-expr '_open) "Content"]]
```

Four hundred lines later, it worked. I felt accomplished.

Then I looked at the bundle size.

## The Numbers Don't Lie

| Library | Size (gzipped) | Ratio |
|---------|----------------|-------|
| Datastar | ~10.76 KB | 1x |
| Aleth | 80 KB | 7.4x |

The gap is structural, not fixable. Aleth includes:
- ClojureScript core (~50KB alone)
- Transit encoding/decoding
- cljs.reader for parsing expressions
- Custom evaluator
- [Malli](https://github.com/metosin/malli) schemas

Even stripping everything optional, ClojureScript's baseline makes parity impossible.

## The Fundamental Problem

Every Aleth expression goes through:

```
String -> cljs.reader/read-string -> AST -> tree-walk evaluation -> result
```

Every click, every reactive update, every signal change pays this parsing tax. I'm interpreting an interpreter.

Datastar's expressions are native JavaScript:

```javascript
$_open = !$_open
```

Evaluated by the browser's JavaScript engine via `Function()` constructor. Zero parsing overhead. Battle-tested. Every edge case handled by decades of browser development.

**You cannot beat native JavaScript at being JavaScript.**

This should have been obvious from the start. I was so focused on the elegance of unified syntax that I ignored the fundamental constraint: the browser already has an expression language. It's optimized. It works. Adding a layer on top is pure overhead.

## The Honest Comparison

When I forced myself to answer "What does Aleth offer over Datastar?", the answer was deflating:

| Aspect | Aleth | Datastar |
|--------|-------|----------|
| DOM morphing | Idiomorph | Idiomorph (same) |
| SSE protocol | Custom events | Custom events (same) |
| Declarative attributes | Yes | Yes (same) |
| Local signals | Yes | Yes (same) |
| Bundle size | 80KB | 10KB |
| Expression parsing | Custom interpreter | Native browser |
| Community | Just me | Growing ecosystem |
| Backend SDKs | Clojure only | Go, Python, PHP, Java, etc. |

The differentiator is "Clojure syntax for expressions." That's it. And that differentiator adds complexity, size, and overhead without benefiting users.

## The Trap Pattern

I fell into a trap I've seen before. Call it "language loyalty syndrome."

The pattern:
1. Discover a tool that works well
2. Notice it's not in your preferred language
3. Conclude the solution is to rewrite it
4. Spend weeks reimplementing what already exists
5. End up with a worse version that you now have to maintain

The justification sounds reasonable: "We'll have Clojure all the way down!" But the justification conflates two different things:
- **Server code**, where language choice matters (you write a lot of it, it's complex, types and tooling matter)
- **Client expressions**, which are trivial one-liners (`$count++`, `$_open = !$_open`)

Nobody writes complex logic in Datastar expressions. They're not meant for that. The server handles complexity. The client handles `$visible = true`.

Optimizing for "Clojure syntax" in the client is optimizing for something that doesn't need optimization.

## What I Should Have Built

The valuable part of Aleth is the server side:

```clojure
;; This is actually useful
(defn counter-view [count]
  [:div {:data-signals (json/encode {:count count})}
   [:button {:data-on-click "$count++"} "+"]
   [:span {:data-text "$count"} count]])

;; SSE helpers are useful
(a/sse-response
  (fn [sse]
    (a/patch! sse "#counter" (counter-view new-count))))
```

A Clojure SDK for Datastar would be:
- Hiccup helpers that emit Datastar-compatible attributes
- Ring middleware for SSE responses
- Transit encoding if you want richer data types

Use Datastar's 10KB client as-is. Don't rewrite it. Don't wrap it. Include the CDN script and move on.

## The Lesson

Before rewriting an existing tool in your preferred language, ask:

1. **Where is the value?** (Server logic vs. client expressions)
2. **What am I actually gaining?** (Syntax consistency? Is that worth 8x bundle size?)
3. **What is the maintenance cost?** (Tracking upstream changes, fixing edge cases, security audits)
4. **Who benefits?** (You as the developer, or actual users?)

The value of server-driven UI is in the *architecture*, not the *syntax*. Datastar already nailed the architecture. Wrapping it in Clojure syntax adds complexity without improving the architecture.

## The Hard Part

Abandoning the experiment was harder than I expected. I had working code. I had solved interesting problems. The expression evaluator was elegant in its way.

But "I built a thing" is not the same as "I built a thing worth using."

The honest answer to "what if Datastar, but in Clojure?" is: "Use Datastar. Write your server in Clojure. The expressions on the client are JavaScript, and that's fine."

Sometimes the right answer is to not build the thing.

---

*The Aleth experiment is preserved at [github.com/parenstech/aleth](https://github.com/parenstech/aleth). The server-side SDK approach - hiccup helpers and SSE middleware for Datastar - is what I'll build next.*
