Title: Kosmos and Sentinel: Component Lifecycle Done Right
Date: 2026-01-05
Tags: clojure, system-design, devtools
Preview: true

Every Clojure project needs to start things in order and stop them gracefully. [Mount](https://github.com/tolitius/mount), [Component](https://github.com/stuartsierra/component), and [Integrant](https://github.com/weavejester/integrant) handle this for application components. But development environments have different requirements: multiple applications sharing infrastructure, partial availability when one thing breaks, automatic recovery after crashes. The existing tools treat the system as monolithic - one failure brings everything down.

[Kosmos](https://github.com/parenstech/kosmos) is a component lifecycle library that treats failure isolation as a first-class concern. Components belong to groups (failure domains), and a failure in one group does not cascade to others. The architecture separates pure logic (dependency graphs, state machines, group validation) from effects (starting processes, polling for readiness). This makes the core trivially testable - no mocks, just data in and data out. Restart strategies borrowed from Erlang/OTP (`:permanent`, `:transient`, `:temporary`) control recovery behavior. The coordinator pattern serializes all state mutations through a mailbox, eliminating race conditions.

[Sentinel](https://github.com/parenstech/sentinel) builds on Kosmos to supervise OS processes. It handles dependency-aware startup, health monitoring with automatic restart, graceful recovery after crashes, zombie process cleanup, and fork management for parallel development. The key innovation is worktree-based identity: each git worktree gets isolated processes, ports, and databases, enabling true parallel development without conflicts.

Together they form a cohesive system: Kosmos provides the pure lifecycle logic, Sentinel adds the process supervision layer. The combination replaces ad-hoc shell scripts with something that understands dependencies, handles failures gracefully, and keeps your development environment running when parts of it break.
