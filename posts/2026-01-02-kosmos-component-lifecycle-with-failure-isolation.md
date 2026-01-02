Title: Kosmos: Component Lifecycle Management with Failure Domain Isolation
Date: 2026-01-02
Tags: clojure, system-design, missionary
Preview: true

Every morning I run `bb dev` to start my development environment. It boots PostgreSQL, spawns two ClojureScript compilers, launches two backend services (Brian and Inbox Hero), and wires them all together. When it works, it is seamless. When it does not, things get interesting.

The problem emerged gradually. Brian's migration might fail because I forgot to run `bb db:snapshot` after a schema change. The previous system's response? Roll back *everything* - including Inbox Hero, which was running perfectly fine and had nothing to do with Brian's database. I would find myself waiting for components to restart that never needed to stop in the first place.

This is the story of [Kosmos](https://github.com/yenda/kosmos), a component lifecycle manager for Clojure that treats failure isolation as a first-class concern.

## The Vocabulary of Order

In ancient Greek cosmology, **Kosmos** (kosmos) meant not just the universe, but *order itself* - the underlying structure that makes a system coherent and comprehensible. The philosophers distinguished it from chaos (khaos), the formless void.

Kosmos provides that structure for component systems: the graph of dependencies, the state of each component, the rules for startup and shutdown. It is the abstract order that process supervisors enforce.

## The Problem: Aggressive Rollback

Existing component lifecycle managers for Clojure - [Mount](https://github.com/tolitius/mount), [Component](https://github.com/stuartsierra/component), [Integrant](https://github.com/weavejester/integrant) - handle dependencies well but treat the system as monolithic. When something fails, the whole thing comes down.

For production, that might be exactly what you want. A partial system in an undefined state is arguably worse than no system at all.

For development, it is a productivity killer. I have two unrelated applications sharing infrastructure. If Brian's migration fails, why should Inbox Hero stop? If the ClojureScript compiler crashes, why should the database connection pool close?

The answer is: it should not. What I needed was **failure domain isolation** - the ability to define boundaries where failures are contained.

## Groups: Failure Domains Made Explicit

Kosmos introduces the concept of **groups**. A group is a failure domain - components in the same group fail together, but different groups are isolated.

```clojure
(def components
  {:postgres     {:id :postgres :shared true :deps #{}}
   :brian-clj    {:id :brian-clj :group :brian :deps #{:postgres}}
   :brian-cljs   {:id :brian-cljs :group :brian :deps #{}}
   :inbox-hero   {:id :inbox-hero :group :inbox-hero :deps #{:postgres}}})
```

Three concepts here:

1. **`:group`** - Tags a component with its failure domain
2. **`:shared true`** - Marks infrastructure that belongs to all groups
3. **`:deps`** - Hard dependencies that must be running first

When you start the system with `start-by-groups!`, Kosmos:

1. Starts all shared components first (infrastructure layer)
2. If shared fails, abort everything (database down = nothing works)
3. Starts groups in parallel - one group failing does NOT cancel others

```clojure
(def system (kosmos/make-system components))
(kosmos/start-coordinator! system)
(kosmos/start-by-groups! system)
;; => {:shared {:status :ok :started [:postgres]}
;;     :groups {:brian {:status :error :error #error{...}}
;;              :inbox-hero {:status :ok :started [:inbox-hero]}}}
```

Brian failed. Inbox Hero kept running. Exactly what I wanted.

## The m/join Insight

Here is the architectural decision that makes group isolation work. It is subtle, but essential.

Kosmos uses [Missionary](https://github.com/leonoel/missionary) for async coordination. Missionary's `m/join` combinator runs multiple tasks in parallel. But here is the catch: when one task throws, `m/join` cancels all the siblings.

```clojure
;; This would NOT work for group isolation:
(m/? (m/join vector
       (?start-brian-group system)    ;; throws!
       (?start-inbox-hero-group system))) ;; gets cancelled
```

The solution: groups never throw. They return status maps instead.

```clojure
(defn ?start-group
  "CRITICAL: Returns a status map instead of throwing on failure.
   This is essential because m/join cancels siblings when one throws."
  [system group-id group-component-ids]
  (m/sp
   (try
     ;; ... start components ...
     {:group group-id :status :ok :started started-ids}
     (catch Exception e
       ;; Rollback this group only
       {:group group-id :status :error :error e}))))
```

Now `m/join` runs all groups to completion. Each group handles its own failures internally. The caller gets a map of results and can decide what to do.

This pattern - returning status maps instead of throwing - is the key to composition in concurrent systems. Exceptions do not compose; data does.

## The Architecture: Pure Core, Effectful Shell

Kosmos follows a strict separation between pure logic and side effects.

<div class="mermaid">
flowchart TB
    subgraph Pure["Pure Core (testable, no mocks)"]
        G["graph.clj<br/>Kahn's algorithm<br/>dependency queries"]
        S["state.clj<br/>state machine<br/>valid transitions"]
        GR["group.clj<br/>partitioning<br/>isolation validation"]
        SC["schema.clj<br/>Malli schemas<br/>defaults"]
    end

    subgraph Effectful["Effectful Shell"]
        C["coordinator.clj<br/>mailbox-based state<br/>serialized mutations"]
        L["lifecycle.clj<br/>start/stop tasks<br/>ready polling"]
    end

    L --> C
    C --> S
    L --> G
    L --> GR
    C --> SC
    GR --> G
</div>

The pure core answers questions without doing anything:

- "What order should I start these components?" (graph)
- "Is this state transition valid?" (state)
- "Which groups are affected by this failure?" (group)

The effectful shell does the actual work:

- Spawn tasks, poll for readiness, manage timeouts (lifecycle)
- Serialize state mutations through a single mailbox (coordinator)

This separation has a practical benefit: the core is trivially testable. No mocks, no fixtures, just pure functions and data.

```clojure
(deftest topo-levels-test
  (let [components {:a {:id :a :deps #{}}
                    :b {:id :b :deps #{:a}}
                    :c {:id :c :deps #{:a}}}]
    (is (= [#{:a} #{:b :c}]
           (graph/topo-levels components)))))
```

## The Coordinator Pattern

Here is a race condition that bites every stateful system eventually:

```clojure
;; Thread 1
(when (= :running (get-status system :app))  ;; check
  (transition! system :app :stopping))        ;; act

;; Thread 2 (concurrent)
(when (= :running (get-status system :app))  ;; check - also true!
  (transition! system :app :stopping))        ;; act - invalid transition!
```

Both threads see `:running`, both try to transition. One fails because you cannot transition from `:stopping` to `:stopping`.

The solution is the **coordinator pattern**: all state mutations go through a single task that processes commands from a mailbox.

```clojure
;; Commands go into the mailbox (a Missionary task)
(coord/cmd! system {:cmd :transition
                    :id :app
                    :new-status :stopping})

;; Coordinator processes them sequentially
(defn ?coordinator [{:keys [mailbox snapshot]}]
  (m/sp
   (loop [state {}]
     (let [cmd (m/? mailbox)  ;; await next command
           {:keys [state result]} (process-command state cmd)]
       (reset! snapshot state)  ;; update read view
       (when-let [reply-to (:reply-to cmd)]
         (reply-to result))     ;; respond if requested
       (if (= result ::shutdown)
         state
         (recur state))))))
```

This gives us:

1. **Serialized mutations** - No races, ever
2. **Atomic check-and-act** - The `?transition-if!` command checks and transitions in one step
3. **Fast reads** - Queries read from a snapshot atom (eventually consistent, but fast)

The coordinator is the single source of truth. Everything else is derived.

## The State Machine

Components move through seven states:

<div class="mermaid">
stateDiagram-v2
    [*] --> stopped
    stopped --> starting: start
    starting --> running: ready
    starting --> error: failure
    running --> draining: stop requested
    running --> degraded: dep failed
    running --> error: failure
    degraded --> running: dep recovered
    degraded --> draining: stop requested
    degraded --> error: failure
    draining --> stopping: drained
    draining --> error: failure
    stopping --> stopped: stopped
    stopping --> error: failure
    error --> starting: restart
    error --> stopped: give up
</div>

The interesting one is **:degraded**. When a dependency fails, dependents can either:

- **:stop** - Shut down (the default)
- **:degraded** - Keep running with reduced functionality
- **:ignore** - Pretend nothing happened

```clojure
{:id :api-server
 :deps #{:database :cache}
 :on-dep-failure :degraded  ;; keep serving if cache dies
 :start-fn (fn [{:keys [database cache]}]
             (start-server {:db database
                            :cache (or cache (no-op-cache))}))}
```

For development, `:degraded` mode means the REPL stays alive even when parts of the application fail. You can fix the problem and restart just the broken component.

## Dependency Ordering with Maximum Parallelism

Kosmos uses Kahn's algorithm for topological sorting, but with a twist: instead of producing a flat order, it produces **levels**.

```clojure
(graph/topo-levels components)
;; => [#{:postgres :redis}     ;; level 0: no deps
;;     #{:cache :auth}         ;; level 1: depends on level 0
;;     #{:api :worker}]        ;; level 2: depends on levels 0-1
```

Components within a level start in parallel. Levels are sequential. This maximizes parallelism while respecting dependencies.

The same algorithm runs in reverse for shutdown:

```clojure
(graph/reverse-topo-levels components)
;; => [#{:api :worker}         ;; stop dependents first
;;     #{:cache :auth}
;;     #{:postgres :redis}]    ;; infrastructure last
```

## Cross-Group Dependency Validation

Groups only provide isolation if components respect group boundaries. A component in `:brian` depending on a component in `:inbox-hero` would violate the isolation guarantee - if Inbox Hero fails, Brian would be affected.

Kosmos validates this at system creation:

```clojure
(group/validate-group-isolation! components)
;; Throws if:
;; - Component in :brian has hard dep (:deps) on component in :inbox-hero
;; - Neither component is :shared

;; OK:
;; - Cross-group soft deps (:wants) - these degrade gracefully
;; - Deps on :shared components - those are global infrastructure
```

This validation runs before any components start. Fail fast, fail clearly.

## Restart Strategies (OTP-Inspired)

Erlang/OTP got restart strategies right decades ago. Kosmos borrows the vocabulary:

| Strategy | Behavior |
|----------|----------|
| `:permanent` | Always restart on failure |
| `:transient` | Restart on abnormal exit only (exception or non-zero) |
| `:temporary` | Never restart |

Combined with exponential backoff:

```clojure
{:id :flaky-service
 :restart :permanent
 :restart-delays [1000 2000 5000 10000 30000]  ;; last value repeats
 :max-restarts 5}  ;; nil = unlimited
```

After 5 failures within the backoff window, the component stays in `:error` and stops trying. This prevents a broken component from consuming resources in an infinite restart loop.

## When to Use Kosmos

Kosmos is designed for development environments where:

- Multiple applications share infrastructure
- Partial availability is acceptable
- Fast feedback matters more than bulletproof recovery

For production systems, you might want:

- **Simpler**: Mount/Integrant if you do not need group isolation
- **More mature**: Kubernetes if you need distributed orchestration
- **Different tradeoffs**: Custom supervision if you need different semantics

Kosmos fills a specific niche: development environments with multiple failure domains. If that is your situation, it might save you the frustration that motivated its creation.

## The Takeaway

The core insight behind Kosmos is simple: **failure domains should be explicit, not implicit**.

When you mark a component with `:group :brian`, you are making a claim about failure boundaries. When you mark infrastructure as `:shared true`, you are declaring it critical to everything. The system validates these claims and enforces them.

Most importantly, the architecture makes these guarantees possible through careful design:

- Pure core for testability
- Coordinator pattern for race-free state
- Status maps instead of exceptions for composition
- Level-based parallelism for speed

The result is a component manager that understands that not all failures are equal - and that sometimes the best response to one thing breaking is to let everything else keep running.
