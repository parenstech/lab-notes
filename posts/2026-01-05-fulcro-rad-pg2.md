Title: Fulcro-RAD-PG2: When JDBC Becomes the Bottleneck
Date: 2026-01-05
Tags: clojure, fulcro, postgresql, performance
Preview: true

[Fulcro RAD](https://github.com/fulcrologic/fulcro-rad) provides automatic CRUD generation from attribute definitions - you declare your data model, RAD generates the database operations. The default SQL adapter uses JDBC, which is fine until it is not. In benchmarks, we measured 70-99% of request time spent in JDBC overhead for simple single-row reads. The database was waiting; Java was not.

[Fulcro-RAD-PG2](https://github.com/parenstech/fulcro-rad-pg2) replaces the JDBC layer with [PG2](https://github.com/igrishaev/pg2), a native PostgreSQL driver for Clojure that speaks the Postgres wire protocol directly. Single-row reads that took 12ms now take 0.3ms. The bottleneck moved from "talking to the database" back to "actually doing work."

Beyond raw performance, the adapter simplifies the development workflow. It auto-generates PostgreSQL schema from RAD attribute definitions and auto-generates [Pathom3](https://github.com/wilkerlucio/pathom3) resolvers that handle the common patterns. Custom bidirectional transformers support JSON columns, CSV arrays, and field-level encryption. Full relationship support covers to-one, to-many, self-referential hierarchies, and many-to-many through join tables - including cascade delete and collection ordering.

If your Fulcro RAD application is I/O bound on simple queries, the problem might not be your database. It might be how you are talking to it.
