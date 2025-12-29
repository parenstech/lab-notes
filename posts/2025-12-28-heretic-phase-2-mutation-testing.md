Title: Heretic Phase 2: When Your Tests Pass, But Your Code Is Broken
Date: 2025-12-28
Tags: clojure, testing, rewrite-clj
Preview: true

Phase 1 gave us a map: every test linked to the exact code it exercises. Now we put it to work.

What if you swapped every `+` to a `-` in your arithmetic functions? Would your tests catch it? What about replacing `<` with `<=`? If your test suite checks that code *runs* rather than that it *works*, these mutations slip through unnoticed.

Phase 2 brings the mutation engine online. We break your code systematically, one expression at a time, and find out which tests actually notice.

## The Heresy in Action

Here's the loop: mutate, test, revert. One change at a time. Never corrupt source files.

```
1. Scan source files for mutation sites ([rewrite-clj](https://github.com/clj-commons/rewrite-clj))
2. For each mutation:
   a. Apply mutation to file (with backup)
   b. Hot-reload changed namespaces ([clj-reload](https://github.com/tonsky/clj-reload))
   c. Look up covering tests (from Phase 1's index)
   d. Run those tests (with timeout)
   e. Record: killed or survived
   f. Revert mutation (restore backup)
3. Report mutation score
```

The key insight from Phase 1 pays off immediately: instead of running your entire test suite for each mutation (which would take forever), we run only the tests that touch that code. A function covered by 2 tests? We run 2 tests. Not 200.

## 18 Ways to Break Your Code

Mutation operators are the vocabulary of mutation testing. Each one represents a common programming mistake:

**Arithmetic**: `+` becomes `-`, `*` becomes `/`

**Boolean**: `and` becomes `or`, `true` becomes `false`

**Comparison**: `<` becomes `>`, `=` becomes `not=`

**Boundary** (the subtle ones): `<` becomes `<=`, `>` becomes `>=`

That last category is where most surviving mutants hide. Off-by-one errors are the cockroaches of programming bugs: hard to spot, impossible to eradicate entirely. Your tests probably check that `validate-age` accepts 18 and rejects 10, but do they check exactly at the boundary?

Each operator is just data:

```clojure
(def swap-plus-minus
  {:id :swap-plus-minus
   :original '+
   :replacement '-
   :description "Replace + with -"
   :matcher (symbol-matcher '+)})
```

The matcher checks if a rewrite-clj zipper location is a valid target. The replacement is the mutation. Simple.

## Surgery Without Scarring

Here's the tricky part: applying mutations to real source files without losing whitespace, comments, or formatting. A naive approach would parse to an AST, modify it, and pretty-print back. Your carefully formatted code would come out looking like machine-generated slop.

[rewrite-clj](https://github.com/clj-commons/rewrite-clj) solves this elegantly. It parses Clojure into a zipper structure that preserves *everything* - not just the code, but the spaces between tokens, the newlines, the comments. You navigate, you edit, you serialize back. The unchanged parts stay exactly as they were.

We navigate to the exact subexpression using coordinates from Phase 1: `form-id` finds the top-level form, `coord` navigates within it:

```clojure
(defn apply-mutation! [mutation]
  (let [{:keys [file form-id coord operator]} mutation
        original-content (slurp file)
        zloc (z/of-string original-content {:track-position? true})

        ;; Navigate to mutation location
        form-zloc (find-form-by-id zloc form-id)
        target-zloc (coord->zloc form-zloc coord)

        ;; Apply replacement
        replacement-str (apply-operator operator target-zloc)
        modified-zloc (z/replace target-zloc (n/token-node (symbol replacement-str)))
        modified-content (z/root-string modified-zloc)]

    (spit file modified-content)
    (assoc mutation :backup original-content)))
```

The `:backup` key is our safety net. Reverting is just `(spit file backup)`.

But side effects demand guarantees. What if a test throws? What if we crash mid-run? A macro ensures we always revert:

```clojure
(defmacro with-mutation [[binding mutation] & body]
  `(let [~binding (apply-mutation! ~mutation)]
     (try
       ~@body
       (finally
         (revert-mutation! ~binding)))))
```

No orphaned mutations. No corrupted source files. Ever.

## The Hot Reload Dance

Mutating a file on disk means nothing if the JVM keeps running the old bytecode. We need the runtime to pick up the change.

[clj-reload](https://github.com/tonsky/clj-reload) handles this:

```clojure
(defn reload-after-mutation! []
  (reloader/reload!))
```

It tracks file timestamps and reloads only what changed, in dependency order. More importantly, it handles protocols and multimethods correctly - a common pitfall where naive reloading leaves stale dispatch tables that cause mysterious test failures.

## Targeted Testing: The Phase 1 Payoff

This is where the coverage index from Phase 1 becomes essential. Given a mutation at `form-id + coord`, we look up which tests exercise that code:

```clojure
(defn tests-for-mutation [index mutation]
  (let [form-id (resolve-form-id index mutation)]
    (coverage/tests-for-location index form-id)))
```

If `sample.core/add` is covered by `sample.core-test/test-add` and `sample.core-test/test-compute`, we run exactly those two tests. Not the whole suite. Not even close.

Test execution wraps in a timeout. Mutations can turn terminating code into infinite loops - imagine swapping `(> counter 0)` to `(< counter 0)` in a while condition:

```clojure
(defn run-test-with-timeout [test-var timeout-ms]
  (let [f (future (run-single-test test-var))
        result (deref f timeout-ms ::timeout)]
    (if (= result ::timeout)
      (do (future-cancel f) {:status :timeout})
      {:status :completed :results result})))
```

## Reading the Verdict

Each mutation gets a status:

- **:killed** - A test failed. Good. Your tests caught the bug.
- **:survived** - All tests passed. Bad. Your tests missed it.
- **:no-coverage** - No tests cover this location. Maybe intentional, maybe not.
- **:timeout** - Tests took too long. Usually means an infinite loop.
- **:error** - Something broke. Investigate.

The mutation score is straightforward: `killed / (killed + survived)`. It answers one question: *what percentage of detectable bugs would your tests actually catch?*

```clojure
(defn mutation-score [results]
  (let [killed (count (filter #(= :killed (:status %)) results))
        survived (count (filter #(= :survived (:status %)) results))
        total (+ killed survived)]
    (if (zero? total)
      0.0
      (/ (double killed) total))))
```

## What the Output Looks Like

```
Heretic Mutation Testing

Coverage data is up to date.
Initializing namespace reloader...
Reloader ready.

Scanning for mutation sites...
Found 47 mutation sites.

Running mutation tests...
[100%] 47/47 mutations tested

Mutation Testing Results

Total: 47 mutations

  Killed:      42 (89%)
  Survived:     3 (6%)
  No Coverage:  2 (4%)

Score: 93.3%

Surviving Mutations:
-------------------

1. src/sample/core.clj:27 - swap-lt-lte (< -> <=)
   Not killed by: sample.core-test/test-safe-divide

2. src/sample/core.clj:37 - swap-and-or (and -> or)
   Not killed by: sample.core-test/test-check-range
```

Those three survivors are your action items. Each one is a mutation your tests missed - which means a real bug with that shape could slip through too.

## What Survivors Tell You

Not every survivor needs a new test. For each one, ask:

1. **Is this actually a bug?** Sometimes `<` vs `<=` genuinely doesn't matter for your semantics. Document why if so.

2. **Should I add a test?** If this mutation represents a real bug category, write a test that would fail with the mutation applied.

3. **Is the code unreachable?** Sometimes survivors point to dead code. Maybe delete it.

4. **Is it an equivalent mutant?** Some mutations produce identical behavior to the original. Swapping `(>= x 0)` for `(> x -1)` is logically equivalent for integers - no test can distinguish them. These are false positives, not test gaps. You can safely ignore them, though they do inflate your survivor count.

The goal isn't 100% mutation score. It's knowing where your tests are weak and deciding what to do about it.

## The Speed Question

"Running tests for every mutation must take forever."

It would, if we ran the full suite each time. But we don't:

- 47 mutations in a sample project
- 2 tests average per mutation
- Full run in seconds, not minutes

Without targeted testing, you'd run the full suite 47 times. The coverage index from Phase 1 transforms mutation testing from overnight-batch-job to interactive-feedback-loop. But even this isn't fast enough for continuous development - Phase 3 will push further.

## Getting Started

[Heretic](https://github.com/parenstech/heretic) is available on GitHub.

```clojure
(require '[heretic.core :as heretic])

;; Load config from heretic.edn
(def config (heretic/load-config))

;; Run mutation testing (auto-collects coverage if needed)
(heretic/mutate! config)

;; Or with options
(heretic/mutate! config
  :files ["src/my/critical/code.clj"]  ; Specific files only
  :verbose true)
```

The `mutate!` function handles everything: checking coverage freshness, initializing the reloader, generating mutations, running tests, and reporting results.

## What's Next

Phase 2 proves the concept works. But "seconds, not minutes" still means context-switching. Phase 3 will make it truly fast:

- **Subsumption analysis** - if one mutation dominates another, skip the redundant one
- **Mutant clustering** - group similar mutations, test representatives only
- **Incremental testing** - only re-test forms that changed since last run
- **Watch mode** - mutation test on save, results while you're still thinking about the change

Because mutation testing shouldn't be something you run once a quarter. It should be part of your development loop.

## In a Language Built on Immutability

Mutation is heresy. But now it's *productive* heresy.

---

## Series

This post is part of the Heretic series on building a mutation testing tool for Clojure:

- [Introduction](/posts/2025-12-28-heretic-mutation-testing.html)
- [Phase 1: Coverage Mapping](/posts/2025-12-28-heretic-phase-1-complete.html)
- **Phase 2: Mutation Testing** (this post)
- [Phase 3: Optimization](/posts/2025-12-29-heretic-phase-3-optimization.html)
