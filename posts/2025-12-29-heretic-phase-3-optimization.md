Title: Heretic Phase 3: Making Mutation Testing Actually Usable
Date: 2025-12-29
Tags: clojure, testing, beholder
Preview: true

Mutation testing has an image problem. Mention it at a conference and watch developers nod approvingly before asking, "But doesn't it take forever?" They're not wrong. Run a thousand mutations, execute your test suite for each one, and suddenly you're measuring build times in hours rather than seconds. It's the kind of technique everyone admires but nobody actually uses.

Phase 3 of [Heretic](https://github.com/yenda/heretic) is my answer to that cynicism. What if mutation testing could run in seconds instead of hours?

## The Math That Kills Mutation Testing

Let's be honest about the problem. A moderately-sized Clojure project generates maybe 500 mutations. Your test suite takes 10 seconds. That's 83 minutes of mutation testing before you see a single result. For larger codebases? Hours. Overnight jobs. The kind of feedback loop that makes developers stop caring.

But here's what I noticed while staring at mutation runs: most of that work is wasted.

Three observations changed how I think about this:

1. **Many mutations are redundant** - Swapping `<` to `<=` and `<` to `!=` usually get killed by the same boundary tests. Why test both?

2. **Some mutations can never die** - `(+ x 0)` and `(- x 0)` produce identical behavior. No test will ever distinguish them.

3. **Most mutations don't change between runs** - If I edited one function, why re-mutate the other 500?

Each observation points to an optimization. Together, they make mutation testing fast enough to run while your coffee cools.

## Subsumption: The Hierarchy of Mutations

Mutation testing researchers discovered something powerful: operator subsumption. If mutation A surviving implies mutation B survives, then A **dominates** B. Test A, skip B.

For relational operators, this follows the RORG schema (Relational Operator Replacement with Guard). The idea: some mutations are strictly harder to kill than others, so testing the hard one tells you everything you need to know about the easy ones:

```clojure
(def relational-operator-subsumption
  {'<  [:swap-lt-lte :swap-lt-neq :replace-comparison-false]
   '>  [:swap-gt-gte :swap-gt-neq :replace-comparison-false]
   '<= [:swap-lte-lt :swap-lte-eq :replace-comparison-true]
   '>= [:swap-gte-gt :swap-gte-eq :replace-comparison-true]
   '=  [:swap-eq-lte :swap-eq-gte :replace-comparison-true]
   'not= [:swap-neq-lt :swap-neq-gt :replace-comparison-false]})
```

The intuition: if `(< x 5)` mutated to `(<= x 5)` survives your tests, you've already proven your tests miss boundary conditions. Testing `(< x 5)` mutated to `(not= x 5)` won't reveal anything new - the boundary mutation already exposed the gap.

Heretic computes the minimal set that still gives you full mutation coverage:

```clojure
(defn minimal-operator-set
  "Compute minimal set of operators needed for full coverage."
  [operators]
  (let [dominated (set (mapcat dominated-operators operators))
        minimal (remove dominated operators)]
    (set minimal)))
```

This typically cuts operator count by 30-40% - not by sacrificing thoroughness, but by eliminating redundancy.

## Clustering: Test Representatives, Not Every Variant

Beyond subsumption, mutations cluster naturally. Ten mutations all swapping `+` to `-` in different arithmetic expressions? They'll likely be killed by similar tests. What if we picked a representative and let its fate speak for the group?

Heretic offers several clustering strategies:

```clojure
(defmethod cluster-mutations :operator [mutations _]
  ;; Group by operator type
  (group-by :operator mutations))

(defmethod cluster-mutations :location [mutations _]
  ;; Group by file + line + form-id
  (group-by mutation-location mutations))

(defmethod cluster-mutations :similarity [mutations _]
  ;; Group by operator category + parent form context
  (group-by (juxt :operator-category :parent-form-type) mutations))
```

The key insight: pick the *hardest* mutation from each cluster as representative. If the hard one dies, the easy ones would too. If the hard one survives, you've found a gap worth investigating.

```clojure
(def ^:private operator-hardness
  {;; Boundary mutations are subtle (hard to kill)
   :swap-lt-lte 90
   :swap-gt-gte 90
   ;; RORG mutations (medium-hard)
   :swap-lt-neq 80
   ;; Simple swaps (easier)
   :swap-plus-minus 50
   :swap-lt-gt 50})

(defn select-representative [cluster]
  (->> cluster
       (sort-by mutation-difficulty-score >)
       first))
```

This trades precision for speed. But the point of mutation testing isn't to find every theoretical gap - it's to find the gaps that matter. Representatives surface those.

## Equivalent Mutations: The Unkillable Dead

Some mutations can never die. They're *equivalent* - semantically identical to the original code. No test can kill them because there's nothing to detect.

Heretic uses [rewrite-clj](https://github.com/clj-commons/rewrite-clj) zippers to analyze the syntactic context of each mutation:

```clojure
(def equivalent-patterns
  [{:operator :swap-plus-minus
    :context (fn [zloc]
               (let [sibling (z/right zloc)]
                 (and sibling
                      (= 0 (z/sexpr sibling)))))
    :reason "Adding or subtracting zero has no effect"}

   {:operator :swap-mult-div
    :context (fn [zloc]
               (let [sibling (z/right zloc)]
                 (and sibling
                      (= 1 (z/sexpr sibling)))))
    :reason "Multiplying or dividing by one has no effect"}

   {:operator :negate-conditional
    :context (fn [zloc]
               ;; (if (not (not x)) ...) -> removing outer not is equivalent
               (when-let [child (z/down zloc)]
                 (and (= 'not (z/sexpr child))
                      (when-let [inner (-> child z/right z/down)]
                        (= 'not (z/sexpr inner))))))
    :reason "Double negation is equivalent to no negation"}])
```

Heretic identifies these patterns statically and filters them before testing. This isn't just about speed - it's about trust. Every survivor should mean something. False alarms from equivalent mutants erode confidence in the tool itself.

## Incremental Testing: Only Re-Mutate What Changed

The biggest win comes from a simple observation: code doesn't change that much between runs.

Heretic tracks SHA-256 hashes of every top-level form in your codebase:

```clojure
(defn hash-file-forms [file-path]
  (when-let [zloc (parser/parse-file file-path)]
    (loop [current (z/down zloc)
           form-id 0
           result {}]
      (if (nil? current)
        result
        (recur (z/right current)
               (inc form-id)
               (assoc result form-id (compute-form-hash current)))))))
```

On the next run, only forms with changed hashes get re-mutated:

```clojure
(defn changed-forms [heretic-dir source-paths]
  (let [previous (load-form-hashes heretic-dir)
        current (compute-current-hashes source-paths)]
    {:changed-files (find-changed-files previous current)
     :changed-forms (find-changed-forms previous current)
     :current-hashes current}))
```

Fixed a bug in one function? That's the only function that gets mutated. The hundreds of mutations from unchanged code? Already cached. For typical development - editing one or two functions at a time - mutation count collapses from hundreds to single digits.

## Watch Mode: Where It All Comes Together

All these optimizations enable something that would have seemed absurd when we started: continuous mutation testing.

```clojure
(defn start-watch! [config & {:keys [verbose debounce-ms]
                              :or {verbose true debounce-ms 300}}]
  (let [index (load-coverage-index config)
        paths (concat (:source-paths config) (:test-paths config))
        handler (fn [event]
                  (let [file (:path event)]
                    (if (test-file? file)
                      (handle-test-change config file verbose)
                      (handle-source-change config index file verbose))))
        debounced (debounce handler debounce-ms)
        watcher (apply beholder/watch debounced paths)]
    (println "Watch mode started. Edit source files to trigger mutation testing.")
    watcher))
```

Using [beholder](https://github.com/nextjournal/beholder) (the same file watcher as [Kaocha](https://github.com/lambdaisland/kaocha)), Heretic monitors your source and test directories. Save a file, mutations run. 300ms debounce, targeted tests from the Phase 1 coverage map, results in seconds.

This is where all three phases pay off:
- **Phase 1** tells us exactly which tests cover the changed code
- **Phase 2** defines the mutations to apply
- **Phase 3** makes it fast enough to run on every save

## The Complete Pipeline

A Phase 3 mutation run now looks like this:

1. Load previous form hashes, identify what changed
2. Generate mutations only for changed code
3. Filter out equivalent mutations (the unkillable dead)
4. Apply subsumption analysis (the dominated redundant)
5. Cluster remaining mutations by similarity
6. Select hardest-to-kill representatives from each cluster
7. Run targeted tests using the coverage map
8. Propagate results to all cluster members
9. Update hashes for next incremental run

What took 60 minutes now takes 30 seconds. That's not an incremental improvement - it's a different tool entirely.

## When This Doesn't Work

These optimizations involve real tradeoffs:

- **Clustering trades precision for speed** - Representatives may miss edge cases that other cluster members would have caught. If you need exhaustive coverage, disable clustering.

- **Subsumption assumes standard operator semantics** - Custom operators or macros that redefine comparison behavior won't follow the RORG hierarchy. Heretic can't know that your `my-less-than` doesn't behave like `<`.

- **Equivalent mutation detection is static analysis** - It catches obvious patterns like `(+ x 0)` but can't detect runtime equivalence. A mutation that's equivalent due to data constraints (e.g., a value that's always positive) will still run.

For most projects, a 100x speedup is worth these tradeoffs. But if you're testing safety-critical code, run a full unoptimized pass periodically - perhaps nightly or before releases.

## Getting Started

Enable incremental mode and watch mode together for the best development experience:

```clojure
(heretic/mutate! config
  :incremental true
  :watch true)
```

This starts Heretic in watch mode with all Phase 3 optimizations active. Edit a file, save it, and see mutation results in seconds.

## What's Next

Phase 3 completes the core mutation testing pipeline. The roadmap includes parallel execution across multiple JVMs, AI-powered semantic mutations (using LLMs to generate realistic bugs rather than operator swaps), IDE integration for real-time feedback, and historical mutation score trends.

But the main point is already proven: mutation testing doesn't have to be slow. With the right optimizations, it belongs in your development loop, not relegated to your CI pipeline's graveyard shift.

The series started with a question: what if coverage is measuring the wrong thing? Tests that execute code without verifying behavior give false confidence. Mutation testing answers a harder question: would your tests catch real bugs?

Now that question can be answered in seconds, not hours.

In a language built on immutability, mutation remains heresy. But it's become fast heresy - the kind you can practice daily.

---

## Series

This post is part of the Heretic series on building a mutation testing tool for Clojure:

- [Introduction](/posts/2025-12-28-heretic-mutation-testing.html)
- [Phase 1: Coverage Mapping](/posts/2025-12-28-heretic-phase-1-complete.html)
- [Phase 2: Mutation Testing](/posts/2025-12-28-heretic-phase-2-mutation-testing.html)
- **Phase 3: Optimization** (this post)
