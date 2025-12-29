Title: Heretic Phase 1: The Mutation Testing Problem Nobody Talks About
Date: 2025-12-28
Tags: clojure, testing, clojurestorm
Preview: true

Mutation testing has a dirty secret: it's too slow to actually use.

The idea is elegant. Take your code, introduce small bugs (mutations), and check if your tests catch them. If a test fails, the mutant is "killed." If all tests pass, you've found a gap in your coverage. Simple.

Here's the problem. For each mutation, you run your entire test suite. Got 500 mutations and a 30-second test suite? That's over four hours of CPU time. For a realistic codebase, we're talking days.

This is why mutation testing remains a curiosity rather than a standard practice. And this is what [Heretic](https://github.com/parenstech/heretic) fixes.

## The Insight: Most Mutations Only Need 2-3 Tests

When you mutate a single expression, you don't need to run every test in your codebase. You only need the tests that actually exercise that expression. Usually that's 2 or 3 tests, not 200.

The challenge is knowing which tests cover which expressions. Not just which functions - which *subexpressions*. The `+` inside `(if condition (+ a b) (* a b))` might only be covered by half the tests that hit the `*`.

Phase 1 of Heretic builds this map. Given any code location - down to individual subexpressions - it instantly tells you which tests exercise it.

## ClojureStorm: The Compiler That Watches Everything

This is where Clojure's ecosystem gives us something special. [ClojureStorm](https://github.com/flow-storm/clojure) is an instrumented Clojure compiler that powers [FlowStorm](https://github.com/flow-storm/flow-storm-debugger), the time-traveling debugger. It emits callbacks for every expression evaluation - exactly the granularity we need.

Heretic hooks into these callbacks:

```clojure
(Tracer/setTraceFnsCallbacks
 {:trace-expr-fn      (fn [_ _ coord form-id] (record-hit! form-id coord))
  :trace-fn-return-fn (fn [_ _ coord form-id] (record-hit! form-id coord))
  :trace-fn-unwind-fn (fn [_ _ coord form-id] (record-hit! form-id coord))})
```

Each callback receives a `form-id` (unique identifier for the top-level form) and a `coord` (path into the AST like `"3,2,1"`). That coordinate pinpoints the exact subexpression - the third element's second element's first element.

With these coordinates, we can track precisely what code runs when each test executes.

## Running Tests One at a Time

My first attempt tried to run all tests together while tracking which test was "active" using dynamic bindings. This failed spectacularly. Test runners parallelize. Fixtures share state. Dynamic bindings don't cross thread boundaries cleanly.

The solution is embarrassingly simple: run each test in isolation.

```clojure
(defn run-test-with-coverage [test-var]
  (tracer/reset-current-coverage!)
  (try
    (test-var)
    (catch Exception e
      (println "Test threw exception:" (.getMessage e))))
  {(symbol test-var) (tracer/get-current-coverage)})
```

This works with any test runner - [Kaocha](https://github.com/lambdaisland/kaocha), [Cognitect test-runner](https://github.com/cognitect-labs/test-runner), vanilla [clojure.test](https://clojure.github.io/clojure/clojure.test-api.html). No special integration required.

## Inverting the Index

The raw coverage data maps tests to the forms they exercise:

```clojure
;; test -> form -> coordinates
{sample.core-test/test-add
  {1991344213 #{"3" "3,1" "3,2"}}}
```

But for mutation testing, we need the reverse lookup. Given a mutation at `[form-id coord]`, which tests should we run?

```clojure
;; [form-id coord] -> tests
{[1991344213 "3,2"] #{sample.core-test/test-add}
 [1991344213 "3,1"] #{sample.core-test/test-add}}
```

This inversion turns O(n) scans into O(1) lookups. Mutate the expression at coordinate `"3,2"` in form `1991344213`? Run `test-add`. Done.

## Making It Fast: Incremental Collection

Running every test one at a time sounds slow. It is - for a full collection. The key is avoiding full collections.

Each test namespace gets its own coverage file with hashes of its dependencies:

```clojure
(defn test-ns-stale? [heretic-dir test-ns test-paths source-paths config]
  (let [coverage-data (load-test-ns-coverage heretic-dir test-ns)
        test-file-path (ns->test-file test-ns test-paths)]
    (if (nil? coverage-data)
      true  ;; No coverage file -> stale
      (let [{stored-hashes :hashes
             source-deps :source-deps} coverage-data]
        (or (not= (:test-file stored-hashes) (hash-file test-file-path))
            (not= (:source-files stored-hashes) (hash-files source-deps))
            (not= (:config stored-hashes) (hash config)))))))
```

Change a test file? Only that namespace recollects. Change a source file? Only namespaces that touched it recollect. In practice, you're recollecting a handful of namespaces, not the entire suite.

## The Storage Layout

```
.heretic/
+-- meta.edn           # Form registry + global metadata
+-- index.edn          # Inverse index (coord -> tests)
+-- coverage/
    +-- my-app-core-test.edn
    +-- my-app-api-test.edn
```

Per-namespace files enable surgical updates. The index is derived data - rebuilt from coverage files after each collection. Change detection is cheap; only recollection is expensive.

## Seeing It Work

Running against a sample project with arithmetic, conditionals, collections, and recursion:

```
Collecting coverage...
  Tracer initialized
  Discovering test namespaces in: [test]
  Found namespaces: [sample.core-test]
  Loading 1 stale namespaces...
  Getting form registry...
  Found 47 forms
  Collecting sample.core-test ...

Collection complete:
  Test namespaces: 1 total, 1 stale, 1 collected
  Forms registered: 47
  Duration: 892ms
```

The inverse index maps 150+ coordinate entries to their covering tests. Even the tricky cases work - ClojureStorm represents map and set coordinates specially (like `"V3902528943"`) since hash-based iteration order is undefined.

## Current Limitations

Instrumentation adds overhead. Tests that depend on precise timing - benchmarks, race condition tests, timeout-sensitive assertions - may behave differently under ClojureStorm's tracing. Run performance-critical tests without instrumentation.

Per-test isolation assumes clean test boundaries. If your tests share mutable state or have external side effects (database writes, API calls), the coverage map may not accurately reflect which tests exercise which code paths.

Those special map/set coordinates (`"V3902528943"`) work correctly in Heretic, but they won't map intuitively to source positions if you're inspecting raw coverage data.

## What This Enables

With the coverage map in place, Phase 2 can:

1. Parse source files with [rewrite-clj](https://github.com/clj-commons/rewrite-clj)
2. Identify mutation points (swap `+` for `-`, `>` for `>=`, etc.)
3. For each mutation, look up covering tests from the index
4. Apply the mutation and run only those tests
5. Report which mutations survived

Instead of running 200 tests per mutation, we run 2. Instead of days, minutes.

## Try It

Add to your deps.edn:

```clojure
{:aliases
 {:heretic
  {:extra-deps
   {com.github.flow-storm/clojure {:mvn/version "1.12.0-1"}
    io.github.yenda/heretic {:git/sha "..."}}
   :jvm-opts ["-Dclojure.storm.instrumentEnable=true"
              "-Dclojure.storm.instrumentOnlyPrefixes=your.app"]}}}
```

Create `heretic.edn`:

```clojure
{:source-paths ["src"]
 :test-paths ["test"]
 :instrument-prefixes ["your.app"]}
```

Collect coverage:

```clojure
(require '[heretic.core :as heretic])
(heretic/collect! (heretic/load-config))
```

The `.heretic/` directory now contains your coverage map, ready for mutation testing.

[Phase 2](/posts/2025-12-28-heretic-phase-2-mutation-testing.html) brings the mutation engine online - using rewrite-clj to systematically break your code and the inverse index to test each mutation in milliseconds instead of minutes.

## In a Language Built on Immutability

Mutation is heresy. That's the point.

The irony of mutation testing is that it validates your tests by breaking your code. The philosophy of immutability is that mutation is the source of bugs. Heretic embraces both: we mutate deliberately to prove we'd catch it accidentally.

---

## Series

This post is part of the Heretic series on building a mutation testing tool for Clojure:

- [Introduction](/posts/2025-12-28-heretic-mutation-testing.html)
- **Phase 1: Coverage Mapping** (this post)
- [Phase 2: Mutation Testing](/posts/2025-12-28-heretic-phase-2-mutation-testing.html)
- [Phase 3: Optimization](/posts/2025-12-29-heretic-phase-3-optimization.html)
