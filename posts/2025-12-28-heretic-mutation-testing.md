Title: Heretic: Mutation Testing in Clojure
Date: 2025-12-28
Tags: clojure, testing, mutation-testing
Image: img/heretic-logo.webp

![Heretic](/img/heretic-logo.webp)

Your tests pass. Your coverage is high. You deploy.

Three days later, a bug surfaces in a function your tests definitely executed. The coverage report confirms it: that line is green. Your test ran the code. So how did a bug slip through?

Because coverage measures execution, not verification.

```clojure
(defn apply-discount [price user]
  (if (:premium user)
    (* price 0.8)
    price))

(deftest apply-discount-test
  (is (number? (apply-discount 100 {:premium true})))
  (is (number? (apply-discount 100 {:premium false}))))
```

Coverage: 100%. Every branch executed. Tests: green.

But swap `0.8` for `1.2`? Tests pass. Change `*` to `/`? Tests pass. Flip `(:premium user)` to `(not (:premium user))`? Tests pass.

The tests prove *some* number comes back. They say nothing about whether it's the right number.

## The Question Nobody's Asking

Mutation testing asks a harder question: if I introduced a bug, would any test notice?

The technique is simple. Take your code, introduce a small change (a "mutant"), and run your tests. If a test fails, the mutant is "killed" - your tests caught the bug. If all tests pass, the mutant "survived" - you've found a gap in your verification.

This isn't new. [PIT](https://pitest.org/) does it for Java. [Stryker](https://stryker-mutator.io/) does it for JavaScript. [cargo-mutants](https://mutants.rs/) does it for Rust.

Clojure hasn't had a practical option.

The only dedicated tool, [jstepien/mutant](https://github.com/jstepien/mutant), was archived this year as "wildly experimental." You can run PIT on Clojure bytecode, but bytecode mutations bear no relationship to mistakes Clojure developers actually make. You'll get mutations like "swap IADD for ISUB" when what you want is "swap `->` for `->>` " or "change `:user-id` to `:userId`."

## Why Clojure Makes This Hard

Mutation testing has a performance problem everywhere. Run 500 mutations, execute your full test suite for each one, and you're measuring build times in hours. Most developers try it once, watch the clock, and never run it again.

But Clojure adds unique challenges:

**Homoiconicity cuts both ways.** Code-as-data makes programmatic transformation elegant, but distinguishing "meaningful mutation" from "syntactic noise" gets subtle when everything is just nested lists.

**Macros muddy the waters.** A mutation to macro input might not change the expanded code. A mutation inside a macro definition might break in ways that have nothing to do with your production logic.

**The bugs we make are language-specific.** Threading macro confusion, nil punning traps, destructuring gotchas from JSON interop, keyword naming collisions - these aren't `+` becoming `-`. They're mistakes that come from thinking in Clojure.

## What If It Could Be Fast?

The insight that makes [Heretic](https://github.com/parenstech/heretic) practical: most mutations only need 2-3 tests.

When you mutate a single expression, you don't need your entire test suite. You need only the tests that exercise that expression. Usually that's a handful of tests, not hundreds.

The challenge is knowing which ones. Not just which functions they call, but which *subexpressions* they touch. The `+` inside `(if condition (+ a b) (* a b))` might be covered by different tests than the `*`.

Heretic builds this map using [ClojureStorm](https://github.com/flow-storm/clojure), the instrumented compiler behind [FlowStorm](https://github.com/flow-storm/flow-storm-debugger). Run your tests once under instrumentation. From then on, each mutation runs only the tests that actually touch that code.

Instead of running 200 tests per mutation, we run 2. Instead of hours, seconds.

## What If It Understood Clojure?

Generic operators miss the bugs we actually make:

```clojure
;; The mutation you want: threading macro confusion
(-> data (get :users) first)     ; Original
(->> data (get :users) first)    ; Mutant: wrong arg position, wrong result

;; The mutation you want: nil punning trap
(when (seq users) (map :name users))   ; Original (handles empty)
(when users (map :name users))         ; Mutant (breaks on empty list)

;; The mutation you want: destructuring gotcha
{:keys [user-id name]}           ; Original (kebab-case)
{:keys [userId name]}            ; Mutant (camelCase from JSON)
```

Heretic has 65+ mutation operators designed for Clojure idioms. Swap `first` for `last`. Change `rest` to `next`. Replace `->` with `some->`. Mutate qualified keywords. The mutations you see will be the bugs you recognize.

## What If It Could Think?

Here's a finding that should worry anyone relying on traditional mutation testing: [research shows](https://research.chalmers.se/en/publication/536348) that nearly half of real-world faults have no strongly coupled traditional mutant. The bugs that escape to production aren't the ones that flip operators. They're the ones that invert business logic.

```clojure
;; Traditional mutation: swap * for /
(* price 0.8)  -->  (/ price 0.8)     ; Absurd. Nobody writes this bug.

;; Semantic mutation: invert the discount
(* price 0.8)  -->  (* price 1.2)     ; Premium users pay MORE. Plausible bug.
```

A function called `apply-discount` should never increase the price. That's the invariant tests should verify. An AI can read function names, docstrings, and context to generate the mutations that *test whether your tests understand the code's purpose*.

This hybrid approach - fast deterministic mutations for the common cases, intelligent semantic mutations for the subtle ones - is where Heretic is heading. [Meta's ACH system](https://engineering.fb.com/2025/02/05/security/revolutionizing-software-testing-llm-powered-bug-catchers-meta-ach/) proved the pattern works at industrial scale.

## Why "Heretic"?

Clojure discourages mutation. Values are immutable. State changes through controlled transitions. The design philosophy is that uncontrolled mutation leads to bugs.

So there's something a bit ironic about a tool that deliberately introduces mutations to find those bugs. We mutate your code to prove your tests would catch it if it happened accidentally - to verify that the discipline holds.

---

This is the first in a series on building Heretic. Upcoming posts will cover how ClojureStorm enables expression-level coverage mapping, how we use [rewrite-clj](https://github.com/clj-commons/rewrite-clj) and [clj-reload](https://github.com/tonsky/clj-reload) for hot-swapping mutants, and the optimization techniques that make this practical for real codebases.

If your coverage is high but bugs still slip through, you're measuring the wrong thing.
