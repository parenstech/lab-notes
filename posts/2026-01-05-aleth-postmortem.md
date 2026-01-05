Title: Aleth: A Failed Experiment in Server-Driven UI
Date: 2026-01-05
Tags: clojure, clojurescript, ui, postmortem
Preview: true

Aleth - from Greek *aletheia*, meaning truth as unconcealment - was an attempt to build a server-driven UI library for Clojure that eliminated client-side computation entirely. The server would own all truth. The client would be a pure projection, showing exactly what it was told.

The architecture was sound in theory: deterministic rendering, trivial testability, observable state transitions, schema-validated signals. For AI-supervised development where you want the simplest possible model for an LLM to reason about, having no client-side logic seemed appealing.

The project failed for two reasons. First, [Datastar](https://data-star.dev/) already exists and does this job well - it is mature, actively maintained, and battle-tested. Building a competitor from scratch was not a good use of time when adoption of an existing solution was possible. Second, the latency trade-off proved more limiting than expected. Every interaction requires a server round-trip, which feels broken for common patterns like search-as-you-type or drag-and-drop.

**What I learned:** Before building infrastructure, verify that no suitable solution exists. The "not invented here" instinct is expensive. Datastar now handles our server-driven UI needs, and the time spent on Aleth would have been better spent elsewhere.
