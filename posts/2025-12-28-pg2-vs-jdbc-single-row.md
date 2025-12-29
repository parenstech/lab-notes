Title: I Benchmarked PG2 vs JDBC and Found... Nothing
Date: 2025-12-28
Tags: clojure, postgresql, benchmarks, pg2
Preview: true

I spent an evening benchmarking [pg2](https://github.com/igrishaev/pg2) against [next.jdbc](https://github.com/seancorfield/next-jdbc). PG2 claims 2-3x better performance thanks to PostgreSQL's binary protocol. The author's benchmarks prove it. I was ready to rewrite our database layer.

Then I ran my benchmarks on our typical web app workload and stared at the results: **they were identical**.

Not "roughly similar." Not "within 20%." Identical. Within statistical noise. I triple-checked my setup. I wasn't doing anything wrong.

## The Benchmarks

I tested what most web apps actually do: single-row CRUD operations.

```clojure
;; SELECT one organization by ID
(jdbc/execute-one! conn
  ["SELECT id, name, active FROM organizations WHERE id = ?" org-id]
  {:builder-fn rs/as-unqualified-kebab-maps})

(pg/execute conn
  "SELECT id, name, active FROM organizations WHERE id = $1"
  {:params [org-id]})
```

```clojure
;; INSERT + DELETE in a transaction
(jdbc/with-transaction [tx conn]
  (jdbc/execute-one! tx
    ["INSERT INTO addresses (id, name, city, state, zip, country) VALUES (?, ?, ?, ?, ?, ?)"
     id "Test" "City" "CA" "00000" "USA"])
  (jdbc/execute-one! tx ["DELETE FROM addresses WHERE id = ?" id]))

(pg/with-tx [conn]
  (pg/execute conn
    "INSERT INTO addresses (id, name, city, state, zip, country) VALUES ($1, $2, $3, $4, $5, $6)"
    {:params [id "Test" "City" "CA" "00000" "USA"]})
  (pg/execute conn "DELETE FROM addresses WHERE id = $1" {:params [id]}))
```

## The (Anticlimactic) Results

Full [Criterium](https://github.com/hugoduncan/criterium) benchmarks, 60 samples each:

**SELECT Single Row**

| Pattern | JDBC | PG2 | Difference |
|---------|------|-----|------------|
| Dedicated connection | 23.7 us | 26.7 us | JDBC 11% faster |
| Connection pool | 22.9 us | 20.8 us | PG2 9% faster |
| Prepared statement | 20.3 us | 22.9 us | JDBC 11% faster |

**INSERT + DELETE**

| Pattern | JDBC | PG2 | Difference |
|---------|------|-----|------------|
| Autocommit | 11.7 ms | 11.1 ms | Noise |
| Explicit transaction | 3.17 ms | 3.10 ms | Noise |
| Pool + transaction | 3.07 ms | 6.14 ms | JDBC 2x faster |

Swapping leaders. Within noise. **No clear winner.**

## Where Does the Time Actually Go?

This is where I stopped being confused and started being curious.

For a single-row query returning three columns, here's the breakdown:

<div class="mermaid">
flowchart LR
    A["Your Code<br/>~0%"] --> B["Driver<br/>~5%"]
    B --> C["TCP<br/>~40%"]
    C --> D["PostgreSQL<br/>~10%"]
    D --> E["TCP<br/>~40%"]
    E --> F["Driver<br/>~5%"]
    F --> G["Your Code<br/>~0%"]
</div>

The driver is maybe 5% of total time. Even if PG2's binary protocol is 3x faster at encoding and decoding, that's a 3x improvement on 5%.

On a 3ms operation, you save roughly **0.15ms**.

You cannot reliably measure that. It disappears into network jitter.

## So Is PG2 Lying?

No. Look at what [PG2's benchmarks](https://github.com/igrishaev/pg2#performance) actually test:

```sql
SELECT x FROM generate_series(1,50000) as s(x)
SELECT row_to_json(row(1, random(), 2, random())) FROM generate_series(1,50000)
```

**50,000 rows.** The math changes completely:

```
Driver overhead = 50,000 x per-row encoding cost
```

With 50,000 rows:
- Text protocol: Parse 50k strings, convert each type through string parsing
- Binary protocol: Direct memory mapping, native types

The binary protocol advantage scales linearly with row count. For 50k rows, driver overhead dominates. For 1 row, it vanishes into network latency.

## The Payload Math

Let's make this concrete. Single row with a UUID, string, and boolean:

| Protocol | Payload |
|----------|---------|
| Text | `"550e8400-e29b-41d4-a716-446655440000", "Test Org", "t"` (~50 bytes) |
| Binary | 16 bytes + length-prefixed string + 1 byte (~25 bytes) |

Saving 25 bytes at localhost speeds: ~0.0002ms. Completely unmeasurable.

Now multiply by 50,000:

- Text: ~2.5 MB of string parsing
- Binary: ~1.2 MB of direct memory reads

Suddenly you're saving real milliseconds of CPU time.

## When Each Driver Wins

**PG2 shines when you're moving data:**
- Analytics queries returning thousands of rows
- Report generation
- CSV/JSON exports
- COPY operations for bulk imports
- Streaming large result sets

**JDBC is fine when you're serving web requests:**
- Typical CRUD (fetch one user, update one order)
- When you need the JDBC ecosystem (connection pools, monitoring, etc.)
- Existing codebases that already work

## The Benchmark Code

Full source: [github.com/yenda/pg2-vs-jdbc](https://github.com/yenda/pg2-vs-jdbc)

```clojure
;; PG2 config (following official recommendations)
{:binary-encode? true
 :binary-decode? true
 :pool-min-size 2
 :pool-max-size 8}

;; Run it yourself
clojure -M:run      ;; Full Criterium bench
clojure -M:quick    ;; Quick bench
```

## What I Actually Learned

PG2's claims are valid. The benchmarks are honest. The 2-3x improvement is real.

It just doesn't apply to *my* workload.

Most web apps are network-bound, not driver-bound. We spend our time waiting for TCP round-trips and PostgreSQL query planning, not encoding result sets. For single-row CRUD, the driver is the wrong thing to optimize.

Before reaching for a faster library, ask: where does the time actually go? The answer might save you a rewrite.
