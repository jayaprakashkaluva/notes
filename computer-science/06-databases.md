# Databases: Durable, Queryable, Concurrent State

Assumes [Data Structures](03-data-structures.md) (B-trees, hash tables), [Algorithms](04-algorithms.md), and [Operating Systems](operating-systems.md) (files, fsync, concurrency). A database is what you use when data must **survive crashes**, be **queried flexibly**, stay **consistent under concurrent access**, and **outgrow memory** — the four problems application code solves badly by hand. This document covers the relational model and SQL, how databases work inside, and the modern landscape. (Per-system internals live in [../datastores/](../datastores/).)

---

## Table of Contents

1. [Why Files Aren't Enough](#1--why-files-arent-enough)
2. [The Relational Model](#2--the-relational-model)
3. [SQL](#3--sql)
4. [Schema Design and Normalization](#4--schema-design-and-normalization)
5. [Indexes](#5--indexes)
6. [How Queries Execute](#6--how-queries-execute)
7. [Transactions: ACID](#7--transactions-acid)
8. [Concurrency Control](#8--concurrency-control)
9. [Crash Recovery: Logging](#9--crash-recovery-logging)
10. [Storage Engines: B-trees vs. LSM-trees](#10--storage-engines-b-trees-vs-lsm-trees)
11. [Beyond One Node, Beyond Relational](#11--beyond-one-node-beyond-relational)
12. [The Road to Expertise](#12--the-road-to-expertise)

---

# 1 — Why Files Aren't Enough

Try building a store's records as CSV files and you re-derive the whole field: finding one customer = scan everything (no **indexes**); "orders joined to customers" = hand-written nested loops (no **query language**); crash mid-write = half-written corruption (no **atomicity**); two processes writing = lost updates ([OS doc Part 6](operating-systems.md)'s races, now on disk); "every order must reference a real customer" = enforced nowhere (no **integrity**). A database is the reusable, ferociously-engineered answer to all five, behind one interface.

# 2 — The Relational Model

Codd's 1970 insight: store data as plain **tables** (relations) — rows (tuples) with typed columns — and *describe* queries in terms of table math, letting the system figure out execution.

```
customers                          orders
 id | name  | city                  id | customer_id | total | placed_at
----+-------+--------              ----+-------------+-------+-----------
  1 | Ana   | Lisbon                501|      1      | 40.00 | 2026-01-05
  2 | Bo    | Austin                502|      1      | 12.50 | 2026-02-11
```

- **Primary key**: column(s) uniquely identifying each row (`customers.id`).
- **Foreign key**: a column referencing another table's key (`orders.customer_id` → `customers.id`) — relationships are *data*, not pointers, and the database can *enforce* them (referential integrity: no order for a nonexistent customer).
- The revolutionary part is **data independence**: tables have no inherent order, no access paths in the model — *how* data is stored (heap files, B-trees, sort orders) can change freely without breaking queries. The model is the interface; storage is the implementation. (The same abstraction move as every other document — applied to data, and worth a Turing Award.)

# 3 — SQL

The declarative language of the model — you state *what*, the optimizer (§6) decides *how*:

```sql
SELECT   c.city, COUNT(*) AS orders, SUM(o.total) AS revenue
FROM     orders o
JOIN     customers c ON c.id = o.customer_id
WHERE    o.placed_at >= '2026-01-01'
GROUP BY c.city
HAVING   SUM(o.total) > 1000
ORDER BY revenue DESC
LIMIT    10;
```

The mental model that makes SQL click — **logical evaluation order** (≠ writing order): `FROM`+`JOIN` (build the combined rows) → `WHERE` (filter rows) → `GROUP BY` (collapse into groups) → `HAVING` (filter *groups*) → `SELECT` (compute output columns) → `ORDER BY`/`LIMIT`. Most SQL confusion (why can't `WHERE` see an aggregate? why does `HAVING` exist?) dissolves against this pipeline.

Core vocabulary beyond the above:

- **JOIN flavors**: `INNER` (only matches), `LEFT OUTER` (keep all left rows, NULLs for missing matches — "customers *including* those with no orders"), `FULL`, `CROSS`. Joins are the model's superpower: relationships reassembled at query time, any shape, no pre-decided access path.
- **NULL** — SQL's three-valued logic: `NULL = NULL` is not true but *unknown*; use `IS NULL`. (Doc 02 §10.3's null problem, in its original habitat; a top source of subtle query bugs, e.g. `NOT IN` with NULLs matching nothing.)
- **Subqueries and CTEs** (`WITH ... AS`) — named intermediate results; window functions (`SUM(...) OVER (PARTITION BY ...)`) — aggregates *without* collapsing rows (running totals, rank-within-group; the feature that graduates you from intermediate SQL).
- **DDL/DML**: `CREATE TABLE` (with types, keys, `NOT NULL`, `CHECK`, `UNIQUE` constraints — push integrity into the database; it's the only layer that sees *all* writers), `INSERT`/`UPDATE`/`DELETE`.
- Never build SQL by string concatenation with user input — **parameterized queries only** (SQL injection: [doc 10 §6](10-security-and-cryptography.md)).

# 4 — Schema Design and Normalization

Design question: which tables, which columns where? The failure mode is **redundancy** — storing a customer's city on every order means updates must find every copy (miss one → contradiction: *update anomaly*), and facts appear/vanish with unrelated rows (*insertion/deletion anomalies*).

**Normalization** is redundancy elimination, formalized. The working core (3NF/BCNF, minus the ceremony): **every non-key fact should be stated once, in the table whose key it depends on** — "the key, the whole key, and nothing but the key." City is a fact about a *customer* → customers table only; orders *reference* the customer. Practical method: name the entities (customer, order, product), one table each; one table per many-to-many relationship (order_items: order_id, product_id, qty); facts attach to the entity they're *about*.

**Denormalization** — deliberately reintroducing redundancy for read speed (a precomputed `order_count` on customers; a star-schema warehouse) — is a legitimate optimization *after* a normalized baseline, with the update-consistency cost consciously accepted. Analytics systems ([../analytics/](../analytics/)) live further along this trade-off than transactional ones.

# 5 — Indexes

The single highest-leverage database skill. Without an index, `WHERE email = '...'` **scans every row** — O(n) with n on disk. An **index** is an auxiliary structure mapping column value → row location.

- The default structure is the **B+tree** (doc 03 §8.3 realized): hundreds of keys per disk-page-sized node, 3–4 levels for billions of rows, values only in leaves, leaves linked for in-order scans. Supports equality *and* ranges/prefixes/ORDER BY (`placed_at BETWEEN ...` — walk the leaves). Hash indexes: equality only, occasionally faster.
- **Composite indexes** index column *tuples* — `(customer_id, placed_at)` sorts by customer, then date within customer. **Leftmost-prefix rule**: that index accelerates `customer_id = ?` and `customer_id = ? AND placed_at > ?`, but *not* `placed_at > ?` alone (like a phone book: useless for searching by first name). Column order in composite indexes is a real design decision.
- A **covering index** contains every column a query needs → the query never touches the table ("index-only scan") — a favorite production fix.
- **The price**: every write updates every index on the table (slower inserts/updates, more space). Indexing is a *read-vs-write trade*, so index what queries actually filter/join/sort on — found via `EXPLAIN` (§6) and slow-query logs, not guessed.
- The primary key usually *is* the table's physical order (clustered index — InnoDB) or its row locator; UUIDs as primary keys scatter inserts across the tree (page splits, cache misses) — why sequential IDs/ULIDs are friendlier at scale.

# 6 — How Queries Execute

The pipeline behind every query — and behind `EXPLAIN`, your window into it:

1. **Parse** SQL into a tree ([doc 08](08-compilers-and-languages.md)'s parsing, applied).
2. **Optimize** — the crown jewel: enumerate equivalent plans (which index? which join *order*? which join *algorithm*?), estimate each cost from **statistics** (row counts, value distributions/histograms per table), pick the cheapest. Join-order search across many tables is doc 04 §7's **dynamic programming**, live in production. When a query is inexplicably slow, the classic causes read straight off this design: stale statistics (run `ANALYZE`), a mis-estimated filter, or a plan flip.
3. **Execute** the plan, classically as an iterator tree (each operator pulls rows from its children; modern engines compile or vectorize instead — [../analytics/clickhouse-architecture-internals.md](../analytics/clickhouse-architecture-internals.md)).

The three join algorithms (recognize them in `EXPLAIN` output):

| Algorithm | How | When the optimizer picks it |
|---|---|---|
| **Nested loop** | for each outer row, probe inner (via index) | small outer × indexed inner — OLTP's bread and butter |
| **Hash join** | build hash table on smaller side, probe with larger | big unsorted inputs, equality joins — analytics' default |
| **Merge join** | both sides sorted, zip them | inputs already sorted (indexes) or sorting pays for itself |

`EXPLAIN ANALYZE` (Postgres) shows the chosen plan *with actual row counts and timings* — reading it is the debugging skill; "estimated 3 rows, actual 3 million" points at the villain.

# 7 — Transactions: ACID

A **transaction** groups statements into one all-or-nothing unit: `BEGIN ... COMMIT` (or `ROLLBACK`). The contract, ACID:

- **Atomicity** — all effects or none, even across a crash mid-transaction. The bank transfer (debit A, credit B) either fully happens or fully doesn't; no state where money vanished. Mechanism: the log (§9).
- **Consistency** — transactions move the database between states satisfying all declared constraints (keys, checks); the database enforces what it's told, the application supplies the rest.
- **Isolation** — concurrent transactions don't see each other's half-finished work; the headline illusion, with fine print important enough to get §8.
- **Durability** — once `COMMIT` returns, the data survives power loss. Mechanism: `fsync` of the log before acknowledging ([OS doc §7.3](operating-systems.md)'s `write ≠ durable` lesson is *why* this is explicit work).

The power move ACID enables: application code can be written *as if* it runs alone and never crashes — the two hardest systems problems, abstracted away. That's why transactions, not tables, are the deepest reason databases exist.

# 8 — Concurrency Control

Full isolation ("serializability" — the outcome equals *some* one-at-a-time ordering) costs performance, so SQL defines **isolation levels** trading correctness for speed, each permitting named **anomalies**:

| Level | Permits | Notes |
|---|---|---|
| Read uncommitted | dirty reads (seeing uncommitted data) | rarely used |
| Read committed | non-repeatable reads (re-read a row → it changed) | Postgres default |
| Repeatable read / snapshot | phantoms (re-run a *query* → new rows), write skew | MySQL default; Postgres RR is snapshot |
| Serializable | nothing | the real thing; costs retries/locks |

The mechanisms:

- **Locking (2-phase locking)**: readers/writers take shared/exclusive row locks, hold to commit. Correct; readers block writers and vice versa; deadlocks are detected and a victim aborted ([OS doc §6.3](operating-systems.md) — "deadlock detected, transaction rolled back" is this).
- **MVCC (multi-version concurrency control)** — the modern default (Postgres, MySQL/InnoDB, Oracle): writers create *new versions* of rows instead of overwriting; each transaction reads a consistent **snapshot** as of its start. **Readers never block writers, writers never block readers** — a copy-on-write idea (the OS doc's §5.4 pattern, again) that made high-concurrency OLTP practical. Costs: old versions accumulate (Postgres VACUUM exists to reap them), and snapshots permit **write skew** — two transactions each read, then write disjoint rows, jointly violating a constraint neither saw broken (the classic: two on-call doctors each see "another doctor is on call" in their snapshot and both sign off). Fixes: `SELECT ... FOR UPDATE` (explicit lock), or serializable level (Postgres SSI detects the dangerous patterns and aborts one).

Practical discipline: keep transactions **short** (no user think-time inside them, no external API calls); handle serialization-failure/deadlock errors by **retrying**; know your engine's default level and its permitted anomalies — silent write skew in a booking system is a real-money bug.

# 9 — Crash Recovery: Logging

How atomicity + durability actually work — the database's deepest engineering, and (like [OS doc §7.3](operating-systems.md)'s journaling, same idea, database-grade) built on one principle:

**Write-ahead logging (WAL)**: before modifying any data page, append a description of the change to a sequential **log**; `COMMIT` = the log records are fsync'd. Data pages themselves can be written back lazily, in any order, whenever convenient — because after a crash, **recovery replays the log**: redo committed changes not yet in the data pages, undo changes from uncommitted transactions (ARIES is the canonical algorithm: analysis, redo-everything, undo-losers; **checkpoints** bound how much log replay is needed).

Why this design wins twice: correctness (the log is the single source of truth ordered before acknowledgment) *and* performance (one sequential fsync'd append per commit — sequential I/O, the disk's favorite — instead of scattered random page writes; group commit amortizes further). And the log turns out to be a gift that keeps giving: **replication** ships it to replicas, **change-data-capture** feeds it to downstream systems, **point-in-time recovery** replays it onto backups — "the log is the database; tables are a cache of it" is a legitimately deep systems slogan (see [../distributed-systems/](../distributed-systems/) and Kafka's whole worldview).

# 10 — Storage Engines: B-trees vs. LSM-trees

The two ways to organize table/index data on disk — the axis along which modern engines divide:

- **B-tree engines** (Postgres, InnoDB, SQL Server): update pages **in place** (WAL-protected). Reads are direct (one tree descent); writes cost random page I/O. The OLTP default for decades.
- **LSM-tree engines** (RocksDB, Cassandra, LevelDB, HBase — [../datastores/rocksdb-internals.md](../datastores/rocksdb-internals.md)): writes go to an in-memory sorted buffer (*memtable*) + WAL; when full, flush as an immutable sorted file (*SSTable*); background **compaction** merges files. Writes become **pure sequential I/O** (superb write throughput, SSD-friendly); reads may check several files (mitigated by **bloom filters** — [../fundamentals/bloom-filters.md](../fundamentals/bloom-filters.md), earning their keep in every LSM read path); compaction consumes background I/O (*write amplification* — the tuning obsession of LSM operators).

Rule of thumb: read-heavy/point-lookup OLTP → B-trees; write-heavy/ingest-heavy → LSM. Analytics adds a third axis — **columnar storage** (store each *column* contiguously: scan only needed columns, compress ruthlessly, vectorize — ClickHouse, Parquet; [../analytics/](../analytics/)) versus row storage for transactional point access. OLTP row-stores and OLAP column-stores are different machines because §1's four problems weight differently.

# 11 — Beyond One Node, Beyond Relational

The one-node story ends where data or load exceeds one machine — the full treatment is [../distributed-systems/](../distributed-systems/) (partitioning, replication, consensus) and [../datastores/](../datastores/) (Cassandra, Yugabyte, Redis...); the map from here:

- **Replication** (copies for failover + read scaling; sync vs. async → replica lag, the first distributed consistency lesson) and **partitioning/sharding** (split by key across nodes; cross-shard queries and transactions get hard — the price of scale).
- **NoSQL** families and what each trades away: **key-value** (Redis, DynamoDB — hash-table-as-a-service: speed, no joins), **document** (MongoDB — nested JSON per record: schema flexibility, weaker cross-document guarantees), **wide-column** (Cassandra — write-optimized partitioned rows: scale, eventual consistency), **graph** (Neo4j — relationship traversal as the primary operation). The 2010s pendulum: NoSQL dropped joins/transactions/SQL for scale; **NewSQL/distributed SQL** (Spanner, CockroachDB, Yugabyte) spent a decade engineering them back on distributed foundations. The relational model, it turns out, was never the bottleneck — single-node implementations were.
- **CAP intuition** (properly: [../distributed-systems/02](../distributed-systems/02-partitioning-and-replication.md)): under a network partition, choose consistency or availability — the one-sentence version of why eventually-consistent systems exist.
- Choosing: default to **Postgres** until a *measured* requirement says otherwise — the boring-technology principle. Most systems never outgrow one well-indexed node plus replicas.

# 12 — The Road to Expertise

## 12.1 Books

1. **SQL in practice first**: *SQLBolt* / *pgexercises.com* (free, interactive) then **SQL Antipatterns** (Karwin) — design mistakes as war stories.
2. **Designing Data-Intensive Applications** (Kleppmann) — the modern classic: storage engines, replication, transactions, the distributed landscape. The single best book adjacent to this document; read after it, before the distributed-systems notes.
3. **Database Internals** (Petrov) — B-trees, LSMs, WAL, recovery, distributed algorithms — one level deeper than this document.
4. **CMU 15-445** (Pavlo, free lectures + projects) — *the* database internals course; its projects have you build a buffer pool, B+tree, and query executor.
5. **Use The Index, Luke** (free online) — everything about indexes from the practitioner's side.

## 12.2 Doing

1. Install **Postgres**. Import a real dataset; write 50 queries of escalating ambition (joins → aggregates → window functions → CTEs).
2. **Make `EXPLAIN ANALYZE` a reflex**: for five of your queries, read the plan, add an index, watch the plan and timing change; construct a case where the composite-index leftmost rule bites.
3. **Break isolation on purpose**: two psql sessions, `BEGIN` in both; produce a lost update at read-committed, then a write skew at repeatable read; fix each with `FOR UPDATE`/serializable. (One evening; permanent intuition.)
4. Kill -9 Postgres mid-write; watch WAL recovery in the logs.
5. Build the CMU 15-445 projects, or write a toy KV store with a WAL and crash-recovery test — then a memtable+SSTable LSM version.
6. Then: [../datastores/](../datastores/) for real engines' internals, [../distributed-systems/](../distributed-systems/) for many-node truth.

## 12.3 Ideas to retain forever

1. **The relational model = data independence** — describe *what*, let the system choose *how*; the abstraction that made data outlive programs.
2. **SQL is a pipeline** (FROM→WHERE→GROUP→HAVING→SELECT→ORDER) — hold it and the language holds together.
3. **State every fact once** (normalize); buy read speed with redundancy only knowingly (denormalize).
4. **Indexes are the read/write trade** — B+trees give equality *and* range; composite order matters (leftmost prefix); `EXPLAIN` is how you know rather than guess.
5. **The optimizer is statistics + DP** — cost-based plans from table stats; stale stats = mystery slowness.
6. **ACID lets you code as if alone and crash-free** — the two hardest problems, rented from the engine.
7. **Isolation has levels with named anomalies** — know your default; write skew is real money; retry serialization failures.
8. **Write-ahead, fsync, replay** — the log is the truth; tables are its cache; replication/CDC/PITR fall out for free.
9. **B-tree vs. LSM vs. columnar = read vs. write vs. scan** — storage engines are the memory-hierarchy lesson, disk edition.
10. **Default to Postgres; distribute only when measurement forces you** — then read the distributed-systems notes first.

---

*Next: [Theory of Computation](07-theory-of-computation.md) — what any computer can compute at all.*
