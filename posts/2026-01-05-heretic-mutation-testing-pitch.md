Title: Heretic: Mutation Testing That Actually Runs
Date: 2026-01-05
Tags: clojure, testing, mutation-testing
Preview: true

Mutation testing is the gold standard for test quality measurement. Introduce bugs into your code, check if your tests catch them. A test suite that passes when the code is broken is not protecting you. The problem: it is slow. Run your entire test suite for every mutation, and a 1000-line project with 5000 possible mutations becomes a multi-hour exercise.

[Heretic](https://github.com/parenstech/heretic) makes mutation testing practical for Clojure through one key innovation: it knows which tests exercise which expressions. Using [ClojureStorm](https://github.com/flow-storm/clojure) instrumentation, Heretic builds a coverage map at expression granularity - not line-level, but every subexpression. When a mutation changes `(+ a b)` to `(- a b)`, Heretic runs only the tests that actually touched that `+`. Two tests instead of two hundred. Milliseconds instead of seconds per mutation.

The library includes 80+ Clojure-specific mutation operators targeting the bugs Clojure developers actually make: threading macro confusion (`->` vs `->>`), nil-handling mistakes (`seq` vs truthy check), lazy/eager mismatches, destructuring key typos. Results come as HTML reports with heatmaps, JSON for CI integration, or EDN for programmatic analysis.

Mutation testing moved from "interesting academic technique" to "thing you run in CI." That is the point.
