Title: Building Heretic: From ClojureStorm to Mutant Schemata
Date: 2025-12-30
Tags: clojure, testing, mutation-testing, clojurestorm

*This is Part 2 of a series on mutation testing in Clojure. [Part 1](/posts/2025-12-28-heretic-mutation-testing.html) introduced the concept and why Clojure needed a purpose-built tool.*

The previous post made a claim: mutation testing can be fast if you know which tests to run. This post shows how [Heretic](https://github.com/parenstech/heretic) makes that happen.

We'll walk through the three core phases: collecting expression-level coverage with [ClojureStorm](https://github.com/flow-storm/clojure), transforming source code with [rewrite-clj](https://github.com/clj-commons/rewrite-clj), and the optimization techniques that keep mutation counts manageable.

## Phase 1: Coverage Collection

Traditional coverage tools track lines. Heretic tracks expressions.

The difference matters. Consider:

```clojure
(defn apply-discount [price user]
  (if (:premium user)
    (* price 0.8)    ;; <- Line 3
    price))
```

Line-level coverage would show line 3 as "covered" if any test enters the `if`-true branch. Expression-level coverage distinguishes between tests that evaluate `*`, `price`, and `0.8`. When we later mutate `0.8` to `1.2`, we can run only the tests that actually touched that specific literal.

### ClojureStorm's Instrumented Compiler

ClojureStorm is a fork of the Clojure compiler that instruments every expression during compilation. Originally built for the [FlowStorm](https://github.com/flow-storm/flow-storm-debugger) debugger, it provides exactly the hooks Heretic needs.

The integration is surprisingly minimal:

```clojure
(ns heretic.tracer
  (:import [clojure.storm Emitter Tracer]))

(def ^:private current-coverage
  "Atom of {form-id #{coords}} for the currently running test."
  (atom {}))

(defn record-hit! [form-id coord]
  (swap! current-coverage
         update form-id
         (fnil conj #{})
         coord))

(defn init! []
  ;; Configure what gets instrumented
  (Emitter/setInstrumentationEnable true)
  (Emitter/setFnReturnInstrumentationEnable true)
  (Emitter/setExprInstrumentationEnable true)

  ;; Set up callbacks
  (Tracer/setTraceFnsCallbacks
   {:trace-expr-fn (fn [_ _ coord form-id]
                     (record-hit! form-id coord))
    :trace-fn-return-fn (fn [_ _ coord form-id]
                          (record-hit! form-id coord))}))
```

When any instrumented expression evaluates, ClojureStorm calls our callback with two pieces of information:

- **form-id**: A unique identifier for the top-level form (computed by hashing the form's source text)
- **coord**: A path into the form's AST, like `"3,2,1"` meaning "third child, second child, first child"

### The Coordinate System

ClojureStorm coordinates are strings representing paths into the AST:

```clojure
(defn foo [a b] (+ a b))
;;  0    1   2     3
;;              3,0 3,1 3,2
```

For the function above:
- `"3"` points to `(+ a b)`
- `"3,0"` points to `+`
- `"3,1"` points to `a`
- `"3,2"` points to `b`

Maps and sets, being unordered, use hash-based coordinates. The key `:user` might be addressed as `"K2847561"` where the number is a hash of the key's printed representation.

### The Form-Location Bridge

Here's a problem we discovered during implementation: ClojureStorm assigns form-ids at compile time, while [rewrite-clj](https://github.com/clj-commons/rewrite-clj) computes its own hashes from source text. They don't match.

The solution was building a bridge through file locations. ClojureStorm's FormRegistry stores the file and line for each form. We build an index:

```clojure
(defn build-form-location-index [forms source-paths]
  (into {}
        (for [[form-id {:keys [form/file form/line]}] forms
              :when (and file line)
              :let [abs-path (resolve-path source-paths file)]
              :when abs-path]
          [[abs-path line] form-id])))
```

When the mutation engine finds a mutation site at `src/my/app.clj` line 42, it looks up the ClojureStorm form-id via this index. The lookup finds the form whose start line is the largest value less than or equal to the mutation's line - the containing form.

### Collection Workflow

Coverage collection runs each test individually and captures what it touches:

```clojure
(defn run-test-with-coverage [test-var]
  (tracer/reset-current-coverage!)
  (try
    (test-var)
    (catch Throwable t
      (println "Test threw exception:" (.getMessage t))))
  {(symbol test-var) (tracer/get-current-coverage)})
```

The result is a map from test symbol to coverage data:

```clojure
{my.app-test/test-addition
  {12345 #{"3" "3,1" "3,2"}    ;; form-id -> coords touched
   12346 #{"1" "2,1"}}
 my.app-test/test-subtraction
  {12345 #{"3" "4"}
   12347 #{"1"}}}
```

This gets persisted to `.heretic/coverage/` with one file per test namespace, enabling incremental updates. Change a test file? Only that namespace gets recollected.

## Phase 2: The Mutation Engine

With coverage data in hand, we need to actually mutate the code. This means:

1. Parsing Clojure source into a navigable structure
2. Finding locations where operators apply
3. Transforming the source
4. Hot-swapping the modified code into the running JVM

### Parsing with rewrite-clj

[rewrite-clj](https://github.com/clj-commons/rewrite-clj) gives us a zipper over Clojure source that preserves whitespace and comments - essential for producing readable diffs:

```clojure
(defn parse-file [path]
  (z/of-file path {:track-position? true}))

(defn find-mutation-sites [zloc]
  (->> (walk-form zloc)
       (remove in-quoted-form?)  ;; Skip '(...) and `(...)
       (mapcat (fn [z]
                 (let [applicable (ops/applicable-operators z)]
                   (map #(make-mutation-site z %) applicable))))))
```

The `walk-form` function traverses the zipper depth-first. At each node, we check which operators match. An operator is a data map with a matcher predicate:

```clojure
(def swap-plus-minus
  {:id :swap-plus-minus
   :original '+
   :replacement '-
   :description "Replace + with -"
   :matcher (fn [zloc]
              (and (= :token (z/tag zloc))
                   (symbol? (z/sexpr zloc))
                   (= '+ (z/sexpr zloc))))})
```

### Coordinate Mapping

The tricky part is converting between rewrite-clj's zipper positions and ClojureStorm's coordinate strings. We need bidirectional conversion for the round-trip:

```clojure
(defn coord->zloc [zloc coord]
  (let [parts (parse-coord coord)]  ;; "3,2,1" -> [3 2 1]
    (reduce
     (fn [z part]
       (when z
         (if (string? part)      ;; Hash-based for maps/sets
           (find-by-hash z part)
           (nth-child z part)))) ;; Integer index for lists/vectors
     zloc
     parts)))

(defn zloc->coord [zloc]
  (loop [z zloc
         coord []]
    (cond
      (root-form? z) (vec coord)
      (z/up z)
      (let [part (if (is-unordered-collection? z)
                   (compute-hash-coord z)
                   (child-index z))]
        (recur (z/up z) (cons part coord)))
      :else (vec coord))))
```

The validation requirement is that these must be inverses:

```clojure
(= coord (zloc->coord (coord->zloc zloc coord)))
```

### Applying Mutations

Once we find a mutation site and can navigate to it, the actual transformation is straightforward:

```clojure
(defn apply-mutation! [mutation]
  (let [{:keys [file form-id coord operator]} mutation
        operator-def (get ops/operators-by-id operator)
        original-content (slurp file)
        zloc (z/of-string original-content {:track-position? true})
        form-zloc (find-form-by-id zloc form-id)
        target-zloc (coord/coord->zloc form-zloc coord)
        replacement-str (ops/apply-operator operator-def target-zloc)
        modified-zloc (z/replace target-zloc
                                 (n/token-node (symbol replacement-str)))
        modified-content (z/root-string modified-zloc)]
    (spit file modified-content)
    (assoc mutation :backup original-content)))
```

### Hot-Swapping with clj-reload

After modifying the source file, we need the JVM to see the change. [clj-reload](https://github.com/tonsky/clj-reload) handles this correctly:

```clojure
(ns heretic.reloader
  (:require [clj-reload.core :as reload]))

(defn init! [source-paths]
  (reload/init {:dirs source-paths}))

(defn reload-after-mutation! []
  (reload/reload {:throw false}))
```

Why clj-reload specifically? It solves problems that `require :reload` doesn't:

1. **Proper unloading**: Calls `remove-ns` before reloading, preventing protocol/multimethod accumulation
2. **Dependency ordering**: Topologically sorts namespaces, unloading dependents first
3. **Transitive closure**: Automatically reloads namespaces that depend on the changed one

The mutation workflow becomes:

```clojure
(with-mutation [m mutation]
  (reloader/reload-after-mutation!)
  (run-relevant-tests m))
;; Mutation automatically reverted in finally block
```

### 80+ Clojure-Specific Operators

The operator library is where Heretic's Clojure focus shows. Beyond the standard arithmetic and comparison swaps, we have:

**Threading operators** - catch `->`/`->>` confusion:
```clojure
(-> data (get :users) first)   ;; Original
(->> data (get :users) first)  ;; Mutant: wrong arg position
```

**Nil-handling operators** - expose nil punning mistakes:
```clojure
(when (seq users) ...)   ;; Original: handles empty list
(when users ...)         ;; Mutant: breaks on empty list (truthy)
```

**Lazy/eager operators** - catch chunking and realization bugs:
```clojure
(map process items)    ;; Original: lazy
(mapv process items)   ;; Mutant: eager, different memory profile
```

**Destructuring operators** - expose JSON interop issues:
```clojure
{:keys [user-id]}   ;; Original: kebab-case
{:keys [userId]}    ;; Mutant: camelCase from JSON
```

The full set includes `first`/`last`, `rest`/`next`, `filter`/`remove`, `conj`/`disj`, `some->`/`->`, and qualified keyword mutations. These are the mistakes Clojure developers actually make.

## Phase 3: Optimization Techniques

With 80+ operators and a real codebase, mutation counts get large fast. A 1000-line project might generate 5000 mutations. Running the full test suite 5000 times is not practical.

Heretic uses several techniques to make this manageable.

### Targeted Test Execution

This is the big one, enabled by Phase 1. Instead of running all tests for every mutation, we query the coverage index:

```clojure
(defn tests-for-mutation [coverage-map mutation]
  (let [form-id (resolve-form-id (:form-location-index coverage-map) mutation)
        coord (:coord mutation)]
    (get-in coverage-map [:coord-to-tests [form-id coord]] #{})))
```

A mutation at `(+ a b)` might only be covered by 2 tests out of 200. We run those 2 tests in milliseconds instead of the full suite in seconds.

### Equivalent Mutation Detection

Some mutations produce semantically identical code. Detecting these upfront avoids wasted test runs:

```clojure
;; (* x 0) -> (/ x 0) is NOT equivalent (divide by zero)
;; (* x 1) -> (/ x 1) IS equivalent (both return x)

(def equivalent-patterns
  [{:operator :swap-mult-div
    :context (fn [zloc]
               (some #(= 1 %) (rest (z/child-sexprs (z/up zloc)))))
    :reason "Multiplying or dividing by one has no effect"}

   {:operator :swap-lt-lte
    :context (fn [zloc]
               (let [[_ left right] (z/child-sexprs (z/up zloc))]
                 (and (= 0 right)
                      (non-negative-fn? (first left)))))
    :reason "(< (count x) 0) is always false"}])
```

The patterns cover boundary comparisons (`(>= (count x) 0)` is always true), function contracts (`(nil? (str x))` is always false), and lazy/eager equivalences (`(vec (map f xs))` equals `(vec (mapv f xs))`).

### Subsumption Analysis

Subsumption identifies when killing one mutation implies another would also be killed. If swapping `<` to `<=` is caught by a test, then swapping `<` to `>` would likely be caught too.

Based on the RORG (Relational Operator Replacement with Guard) research, we define subsumption relationships:

```clojure
(def relational-operator-subsumption
  {'<  [:swap-lt-lte :swap-lt-neq :replace-comparison-false]
   '>  [:swap-gt-gte :swap-gt-neq :replace-comparison-false]
   '<= [:swap-lte-lt :swap-lte-eq :replace-comparison-true]
   ;; ...
   })
```

For each comparison operator, we only need to test the minimal set. The research shows this achieves roughly the same fault detection with 40% fewer mutations.

The subsumption graph also enables intelligent mutation selection:

```clojure
(defn minimal-operator-set [operators]
  (set/difference
   operators
   ;; Remove any operator dominated by another in the set
   (reduce
    (fn [dominated op]
      (into dominated
            (set/intersection (dominated-operators op) operators)))
    #{}
    operators)))
```

### Mutant Schemata: Compile Once, Select at Runtime

The most sophisticated optimization is mutant schemata. Instead of applying one mutation, reloading, testing, reverting, reloading for each mutation, we embed multiple mutations into a single compilation:

```clojure
;; Original
(defn calculate [x] (+ x 1))

;; Schematized (with 3 mutations)
(defn calculate [x]
  (case heretic.schemata/*active-mutant*
    :mut-42-5-plus-minus (- x 1)
    :mut-42-5-1-to-0     (+ x 0)
    :mut-42-5-1-to-2     (+ x 2)
    (+ x 1)))  ;; original (default)
```

We reload once, then switch between mutations by binding a dynamic var:

```clojure
(def ^:dynamic *active-mutant* nil)

(defmacro with-mutant [mutation-id & body]
  `(binding [*active-mutant* ~mutation-id]
     ~@body))
```

The workflow becomes:

```clojure
(defn run-mutation-batch [file mutations test-fn]
  (let [schemata-info (schematize-file! file mutations)]
    (try
      (reload!)  ;; Once!
      (doseq [[id mutation] (:mutation-map schemata-info)]
        (with-mutant id
          (test-fn id mutation)))
      (finally
        (restore-file! schemata-info)
        (reload!)))))  ;; Once!
```

For a file with 50 mutations, this means 2 reloads instead of 100. The overhead of `case` dispatch at runtime is negligible compared to compilation cost.

### Operator Presets

Finally, we offer presets that trade thoroughness for speed:

```clojure
(def presets
  {:fast #{:swap-plus-minus :swap-minus-plus
           :swap-lt-gt :swap-gt-lt
           :swap-and-or :swap-or-and
           :swap-nil-some :swap-some-nil}

   :minimal minimal-preset-operators  ;; Subsumption-aware

   :standard #{;; :fast plus...
               :swap-first-last :swap-rest-next
               :swap-thread-first-last}

   :comprehensive (set (map :id all-operators))})
```

The `:fast` preset uses ~15 operators that research shows catch roughly 99% of bugs. The `:minimal` preset uses subsumption analysis to eliminate redundant mutations. Both run much faster than `:comprehensive` while maintaining detection power.

## Putting It Together

A mutation testing run with Heretic looks like:

1. **Collect coverage** (once, cached): Run tests under ClojureStorm instrumentation, build expression-level coverage map
2. **Generate mutations**: Parse source files, find all applicable operator sites
3. **Filter**: Remove equivalent mutations, apply subsumption to reduce set
4. **Group by file**: Prepare for schemata optimization
5. **For each file**:
   - Build schematized source with all mutations
   - Reload once
   - For each mutation: bind `*active-mutant*`, run targeted tests
   - Restore and reload
6. **Report**: Mutation score, surviving mutations, test effectiveness

The result is mutation testing that runs in seconds for typical projects instead of hours.

---

This covers the core implementation. The [next post](/posts/2025-01-06-heretic-ai-mutations.html) will explore Phase 4: AI-powered semantic mutations and hybrid equivalent detection - using LLMs to generate the subtle, domain-aware mutations that traditional operators miss.

**Previously:** [Part 1 - Heretic: Mutation Testing in Clojure](/posts/2025-12-28-heretic-mutation-testing.html)
