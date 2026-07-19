# PostgreSQL Source Tree — Code Overview

> A high-level, developer-oriented walk-through of what lives in this repository
> and how the pieces fit together.
>
> Repository: **PostgreSQL** (version marker in `configure.ac`: `20devel`)
> License: PostgreSQL License (see `COPYRIGHT`)
> Upstream docs: <https://www.postgresql.org/docs/devel/>

---

## 1. What this repository is

This repository contains the full source distribution of **PostgreSQL**, an
advanced open-source object-relational database management system (ORDBMS).
The codebase implements:

- A SQL-standard-compliant relational engine with extensions (JSON, arrays,
  ranges, geometry, full-text search, etc.).
- **ACID transactions** with multi-version concurrency control (MVCC).
- **Write-ahead logging (WAL)**, physical and logical replication, and
  point-in-time recovery.
- Multiple **index access methods** (B-tree, hash, GiST, GIN, SP-GiST, BRIN).
- A **cost-based query planner/optimizer** with an alternative genetic planner
  (GEQO).
- **Extensibility hooks** for user-defined types, operators, functions,
  aggregates, index methods, foreign data wrappers, and procedural languages.
- Client libraries (`libpq`, ECPG), a rich set of command-line tools, and a
  large `contrib/` collection of first-party extensions.

The project is written primarily in **C**, with Perl (build/tests), Python
(tests), and a smattering of yacc/lex, SQL, and shell.

---

## 2. Top-level layout

```
postgres/
├── COPYRIGHT           License (PostgreSQL License, BSD-style)
├── HISTORY             Pointer to release notes online
├── README.md           One-page project description
├── Makefile /          Autotools + Meson build entry points
│   GNUmakefile.in
├── configure(.ac)      Autoconf-generated build configuration
├── meson.build /       Alternative Meson build system
│   meson_options.txt
├── config/             Autoconf macros
├── doc/                SGML/XML source for the official manual
├── contrib/            First-party extensions & optional modules
└── src/                The actual server, client, and library sources
```

Two build systems coexist: the classic **autoconf + make** flow
(`./configure && make`) and a newer **Meson** flow (`meson setup build`). Both
build the same binaries; developers may use whichever is more convenient.

---

## 3. The `src/` tree

`src/` is the heart of the project. Each subtree has a well-defined role:

| Path                | What it contains                                                                 |
|---------------------|----------------------------------------------------------------------------------|
| `src/backend/`      | The database server itself (the `postgres` executable)                           |
| `src/bin/`          | Command-line tools shipped with the server (`psql`, `pg_dump`, `initdb`, …)      |
| `src/interfaces/`   | Client-side libraries: `libpq` (C), `ecpg` (embedded SQL in C), `libpq-oauth`    |
| `src/include/`      | Public and internal C headers                                                    |
| `src/common/`       | Code compiled into both the backend and frontend tools                           |
| `src/fe_utils/`     | Utilities shared across frontend programs                                        |
| `src/pl/`           | Server-side procedural languages (`plpgsql`, `plperl`, `plpython`, `pltcl`)      |
| `src/port/`         | Portability shims for functions that vary across operating systems               |
| `src/makefiles/`    | Platform-specific make fragments                                                 |
| `src/template/`     | Per-platform build templates used by `configure`                                 |
| `src/test/`         | Regression, isolation, TAP, SSL, LDAP, Kerberos, subscription, and other tests   |
| `src/timezone/`     | Bundled IANA time-zone data and lookup code                                      |
| `src/tools/`        | Developer tools (code checks, PGXS helpers, MSVC helpers, etc.)                  |
| `src/tutorial/`     | Small SQL/C examples used by the documentation                                   |

---

## 4. Inside the backend (`src/backend/`)

The backend is where SQL becomes data on disk. It is organised as a pipeline
from statement text to storage, with supporting subsystems around it.

### 4.1 Query lifecycle at a glance

```
   client
     │  SQL text (via libpq)
     ▼
 ┌───────────┐    ┌───────────┐    ┌────────────┐    ┌───────────┐    ┌───────────┐
 │  parser   │──▶│  analyze  │──▶│  rewriter  │──▶│ optimizer │──▶│ executor  │
 └───────────┘    └───────────┘    └────────────┘    └───────────┘    └───────────┘
                                                                          │
                                                                          ▼
                                                                    access / storage
                                                                    (heap, indexes,
                                                                     buffers, WAL)
```

### 4.2 Key backend subdirectories

- **`main/`** — Entry point (`main.c`). Chooses whether to run as postmaster,
  bootstrap, single-user backend, etc.

- **`postmaster/`** — The supervisor process. Listens for connections and forks
  a backend per client. Also starts auxiliary processes: **WAL writer**,
  **checkpointer**, **bgwriter**, **autovacuum launcher/workers**,
  **archiver**, **logical replication launcher**, **stats collector**, etc.

- **`tcop/`** — "Traffic cop" main loop of a backend session (`postgres.c`).
  Reads a message from the client, dispatches to parser/planner/executor,
  returns results, repeats. Also handles simple utility statements directly.

- **`parser/`** — Lexes and parses SQL, then performs *parse analysis* to
  produce a `Query` tree. See `parser/README` for a file-by-file map
  (`scan.l`, `gram.y`, `analyze.c`, `parse_expr.c`, `parse_clause.c`, …).

- **`rewrite/`** — Applies rules and expands views (the *query rewriter*).

- **`optimizer/`** — Cost-based planner. Turns `Query` trees into `Plan` trees.
  - `path/` — enumerates join orders and access paths.
  - `plan/` — turns the cheapest paths into a plan tree.
  - `prep/` — pre-processing (constant folding, subquery pull-up, …).
  - `util/` — shared helpers (pathnode, restrictinfo, relnode, …).
  - `geqo/` — genetic query optimizer for large join searches.

- **`executor/`** — Runs plan trees using a *demand-pull* pipeline: each
  `PlanState` node produces one tuple per call to its parent. Every plan node
  type has a matching state node and a `nodeXxx.c` file (e.g., `nodeSeqscan.c`,
  `nodeHashjoin.c`, `nodeAgg.c`, `nodeModifyTable.c`). See `executor/README`
  for the model in detail.

- **`access/`** — Storage-facing access methods:
  - `heap/` — the default **heap** table access method, MVCC row storage,
    HOT updates (`README.HOT`), tuple locking (`README.tuplock`), TOAST
    (out-of-line large values), vacuum/prune, and WAL replay.
  - `nbtree/`, `hash/`, `gin/`, `gist/`, `spgist/`, `brin/` — index AMs.
  - `transam/` — the **transaction manager**: XID/CLOG bookkeeping, subxacts,
    WAL emission, two-phase commit, snapshots (`xact.c`, `xlog.c`, `clog.c`,
    `slru.c`, `twophase.c`, `varsup.c`, …). See `access/transam/README`.
  - `table/`, `index/`, `common/`, `sequence/`, `tablesample/`, `rmgrdesc/` —
    generic table/index API layers, sequence AM, TABLESAMPLE, WAL
    resource-manager description code.

- **`storage/`** — Below the access methods:
  - `buffer/` — shared **buffer manager** (pin counts, content locks, clock
    sweep, ring buffers; see `buffer/README`).
  - `smgr/` — storage manager abstraction over relation files (`md.c` = magnetic
    disk).
  - `file/` — virtual file descriptors and temporary files.
  - `freespace/` — free-space maps.
  - `page/` — page-level utilities and page layout.
  - `lmgr/` — heavyweight lock manager (relation, tuple, advisory locks) and
    lightweight LWLocks.
  - `ipc/` — shared-memory setup, latches, dynamic shared memory (DSM),
    process barriers.
  - `sync/` — fsync request queue used by checkpointer.
  - `aio/` — asynchronous I/O infrastructure.
  - `large_object/` — legacy LOB API backing store.

- **`catalog/`** — The **system catalogs** (`pg_class`, `pg_attribute`,
  `pg_type`, …). Contains the `.dat` seed files, `postgres.bki` generator, and
  the `heap`, `index`, `namespace`, `dependency`, `objectaddress`, and
  `partition` helpers that create/alter catalog rows.

- **`commands/`** — Implementations of DDL and other utility SQL statements:
  `CREATE`/`ALTER`/`DROP` for tables, indexes, views, extensions, roles,
  functions, publications, subscriptions; also `COPY`, `VACUUM`, `ANALYZE`,
  `CLUSTER`, `EXPLAIN`, `PREPARE`, `TRUNCATE`, etc.

- **`nodes/`** — Definition of the internal AST/plan node types plus
  copy/equal/read/out functions. `README` describes the node system.

- **`utils/`** — Server-wide utilities:
  - `adt/` — built-in data-type input/output and operator implementations.
  - `cache/` — relcache, catcache, syscache, plancache, typcache.
  - `mmgr/` — memory contexts (`palloc`/`pfree` and friends).
  - `time/` — snapshot management.
  - `sort/` — external merge sort, disk-spillable tuplestore/tuplesort.
  - `hash/`, `mb/`, `misc/`, `error/`, `resowner/`, `activity/`, `fmgr/`, …

- **`libpq/`** — Backend side of the client wire protocol (authentication,
  message dispatch). Client-side counterpart lives in `src/interfaces/libpq/`.

- **`replication/`** — Physical WAL streaming (`walsender.c`, `walreceiver.c`),
  logical decoding, slot management, publications/subscriptions,
  synchronous replication.

- **`backup/`** — `pg_basebackup` server support, incremental backup, and
  WAL archiving glue.

- **`archive/`** — Framework for shipping WAL to an archive location.

- **`partitioning/`** — Partitioning bookkeeping and pruning helpers used by
  the planner and executor.

- **`statistics/`** — Extended, multi-column statistics for the planner.

- **`foreign/`** — Foreign data wrapper (FDW) infrastructure; concrete FDWs
  live under `contrib/` (`file_fdw`, `postgres_fdw`).

- **`jit/`** — Just-in-time compilation of expression and tuple deforming
  code via LLVM (optional at build time).

- **`bootstrap/`** — Special bootstrap-mode backend used by `initdb` to seed
  the initial catalogs.

- **`snowball/`, `regex/`, `tsearch/`, `port/`, `lib/`, `nls.mk`, `po/`** —
  Stemmers, regex engine, full-text search, portability helpers, generic
  data structures, and translated messages.

---

## 5. Client tools (`src/bin/`)

Programs that ship with the server, most implemented as `libpq` clients:

| Tool                | Purpose                                                             |
|---------------------|---------------------------------------------------------------------|
| `psql`              | Interactive SQL terminal                                            |
| `initdb`            | Initialise a new database cluster on disk                           |
| `pg_ctl`            | Start/stop/reload/status control for a running cluster              |
| `pg_dump` / `pg_dumpall` / `pg_restore` | Logical backup and restore                      |
| `pg_basebackup`     | Take a physical base backup over the replication protocol           |
| `pg_verifybackup`   | Verify a base backup's manifest                                     |
| `pg_combinebackup`  | Combine incremental backups into a full backup                      |
| `pg_rewind`         | Resync a stale standby against the current primary                  |
| `pg_upgrade`        | In-place upgrade of a cluster between major versions                |
| `pg_checksums`      | Enable/disable/verify data-page checksums                           |
| `pg_amcheck`        | Corruption checks over heap and B-tree indexes                      |
| `pg_waldump`        | Decode WAL segment files for inspection                             |
| `pg_walsummary`     | Read WAL summary files produced by the summarizer                   |
| `pg_archivecleanup` | Remove old WAL files from an archive                                |
| `pg_resetwal`       | Emergency: reset the WAL and control information                    |
| `pg_controldata`    | Print the contents of `pg_control`                                  |
| `pg_config`         | Print installed configuration/paths                                 |
| `pg_test_fsync` / `pg_test_timing` | Micro-benchmarks for storage and clocks               |
| `pgbench`           | Standard OLTP benchmark tool                                        |
| `pgevent`           | Windows event-log resource module                                   |
| `scripts/`          | Small wrappers such as `createdb`, `dropdb`, `reindexdb`, `vacuumdb`|

---

## 6. Client interfaces (`src/interfaces/`)

- **`libpq/`** — The canonical C client library. Nearly every other driver
  (JDBC, psycopg, npgsql, node-postgres, etc. — external projects) either
  wraps `libpq` or reimplements its wire protocol.
- **`ecpg/`** — **E**mbedded **S**QL in **C** preprocessor and runtime.
- **`libpq-oauth/`** — Optional OAuth 2.0 client-side helper for `libpq`.

---

## 7. Procedural languages (`src/pl/`)

Server-side languages that can be used to write functions, procedures, and
triggers. All four ship with the source tree; each is packaged as an
extension (`CREATE EXTENSION plpgsql;`, etc.).

- **`plpgsql/`** — The default procedural language (SQL-like with control
  flow). Enabled by default in every database.
- **`plperl/`** — Perl.
- **`plpython/`** — Python 3.
- **`tcl/`** — Tcl.

---

## 8. Contrib extensions (`contrib/`)

Optional first-party modules built and shipped alongside the server. A few
highlights:

| Category            | Modules                                                               |
|---------------------|-----------------------------------------------------------------------|
| Data types          | `citext`, `cube`, `hstore`, `isn`, `ltree`, `seg`, `uuid-ossp`, `xml2`|
| Full text / search  | `pg_trgm`, `unaccent`, `dict_int`, `dict_xsyn`, `fuzzystrmatch`       |
| Index methods       | `bloom`, `btree_gin`, `btree_gist`                                    |
| Foreign data        | `postgres_fdw`, `file_fdw`, `dblink`                                  |
| Monitoring/insight  | `pg_stat_statements`, `pageinspect`, `pgstattuple`, `pg_visibility`,  |
|                     | `pg_buffercache`, `pg_freespacemap`, `pg_walinspect`,                 |
|                     | `pg_logicalinspect`                                                   |
| Ops / admin         | `amcheck`, `pg_prewarm`, `pg_surgery`, `oid2name`, `vacuumlo`,        |
|                     | `pgrowlocks`, `passwordcheck`, `auth_delay`, `auto_explain`           |
| Backup / archive    | `basic_archive`, `basebackup_to_shell`                                |
| Crypto              | `pgcrypto`, `sslinfo`                                                 |
| Query tooling       | `pg_overexplain`, `pg_plan_advice`, `pg_stash_advice`                 |
| PL bridges          | `bool_plperl`, `hstore_plperl`, `hstore_plpython`, `jsonb_plperl`,    |
|                     | `jsonb_plpython`, `ltree_plpython`                                    |
| Sampling / misc     | `tsm_system_rows`, `tsm_system_time`, `intagg`, `intarray`,           |
|                     | `tablefunc`, `earthdistance`, `lo`, `tcn`, `test_decoding`,           |
|                     | `spi`, `sepgsql`, `start-scripts`                                     |

Each module lives in its own subdirectory with a `Makefile` (or
`meson.build`), a `.control` file, SQL install scripts, and C sources.

---

## 9. Documentation (`doc/`)

- `doc/src/sgml/` — Source for the official manual, written in DocBook XML.
  Built into HTML, PDF, and man pages.
- `doc/KNOWN_BUGS`, `doc/MISSING_FEATURES`, `doc/TODO` — Long-lived developer
  tracking files; the canonical TODO is maintained on the wiki.

---

## 10. Testing (`src/test/`)

PostgreSQL has a deep testing infrastructure:

- **`regress/`** — SQL-driven regression suite (run via `make check`). Golden
  output comparison against `expected/` files.
- **`isolation/`** — Multi-session concurrency tests using a small permutation
  DSL, targeting locking/MVCC edge cases.
- **`recovery/`, `subscription/`, `authentication/`, `ssl/`, `ldap/`,
  `kerberos/`, `icu/`, `modules/`, `perl/`** — Perl **TAP** tests exercising
  full clusters, replication, security integrations, and helper modules.
- **`locale/`, `mb/`** — Localization and multibyte-encoding tests.
- Extension-specific tests live under each extension's directory
  (`contrib/<name>/expected/` or `t/`).

---

## 11. Build systems

- **Autoconf + Make (classic):**
  ```sh
  ./configure --prefix=/usr/local/pgsql
  make -j
  make check
  make install
  ```
- **Meson (newer):**
  ```sh
  meson setup build --prefix=/usr/local/pgsql
  ninja -C build
  meson test -C build
  ninja -C build install
  ```

Both produce the same server binary, `contrib` shared libraries, and client
tools. Meson tends to be faster on Windows and has cleaner incremental builds.

---

## 12. How a query flows through the code — a concrete walk

1. A client sends `SELECT * FROM t WHERE id = 1;` over a TCP or Unix socket.
2. The **postmaster** (`src/backend/postmaster/postmaster.c`) accepts the
   connection and forks a backend.
3. The backend authenticates via `libpq` (`src/backend/libpq/`).
4. The **traffic cop** (`src/backend/tcop/postgres.c`) reads the query.
5. **Parser** (`src/backend/parser/`): `scan.l` → `gram.y` produces a raw
   parse tree; `analyze.c` turns it into a `Query` node.
6. **Rewriter** (`src/backend/rewrite/`): applies rules and expands views.
7. **Planner** (`src/backend/optimizer/`): builds `Path`s for each
   `RelOptInfo`, picks the cheapest, converts to a `Plan` tree.
8. **Executor** (`src/backend/executor/`): initialises a `PlanState` tree,
   pulls tuples from the top node until exhausted; the `SeqScan` state
   fetches pages via the **buffer manager** (`storage/buffer/`); the buffer
   manager asks the **storage manager** (`storage/smgr/md.c`) which issues
   OS reads.
9. Tuples pass through executor nodes, transformed and filtered, until the
   top node yields them to the destination receiver, which serialises them
   to the client over the wire protocol.
10. If it were a write, the **transaction manager** (`access/transam/`) would
    have logged the change to **WAL** (`xlog.c`); the **checkpointer** and
    **bgwriter** later flush dirty buffers; **autovacuum** eventually cleans
    up dead tuples.

---

## 13. Extending PostgreSQL

Common extension points, each grounded in a specific area of the source:

- **User-defined types / functions / operators** — implemented as SQL
  or as C functions using the `V1` calling convention (`fmgr.h`).
- **Procedural languages** — model after `src/pl/plpgsql`.
- **Index access methods** — implement the AM API used by `access/nbtree`,
  `access/gin`, etc.
- **Table access methods** — implement the API defined in
  `src/include/access/tableam.h`; the heap AM in `access/heap` is the
  reference implementation.
- **Foreign data wrappers** — implement the FDW API from
  `src/backend/foreign/`; see `contrib/postgres_fdw` for a real example.
- **Background workers** — register with the postmaster to run custom
  server processes.
- **Hooks** — many subsystems expose function pointers (planner, executor
  start/end, ProcessUtility, ClientAuthentication, emit_log, …) that
  extensions can chain into. `contrib/auto_explain` and
  `contrib/pg_stat_statements` are canonical examples.
- **Logical decoding output plugins** — see `contrib/test_decoding`.
- **WAL resource managers**, **custom scan nodes**, **archive modules**, and
  **basebackup targets** are also pluggable.

---

## 14. Where to look next

- `src/backend/executor/README` — the executor model.
- `src/backend/optimizer/README` — planner architecture, paths vs. plans.
- `src/backend/parser/README` — parser file map.
- `src/backend/access/transam/README` — transactions, XIDs, WAL layering.
- `src/backend/access/heap/README.HOT` and `README.tuplock` — heap internals.
- `src/backend/storage/buffer/README` — buffer manager rules.
- `src/backend/nodes/README` — internal node system.
- `doc/src/sgml/` — official manual sources (also published at
  <https://www.postgresql.org/docs/devel/>).
- The [PostgreSQL wiki](https://wiki.postgresql.org/wiki/Development_information)
  for style, review process, and mailing-list conventions.

---

*Generated overview based on the current state of the source tree.*
