# Notes

Technical deep-dive notes on the architecture and internals of databases, analytics engines, and AI systems.

## Datastores

| Note | Topic |
|---|---|
| [MySQL Server Architecture](datastores/MYSQL_SERVER_ARCHITECTURE.md) | MySQL Server — architecture & implementation deep dive |
| [PostgreSQL Source Overview](datastores/POSTGRES_OVERVIEW.md) | PostgreSQL source tree — code overview |
| [PostgreSQL Internals](datastores/postgress_ARCHITECTURE_ANALYSIS.md) | How four decades of design decisions fit together |
| [Cassandra Architecture](datastores/cassandra-architecture.md) | Apache Cassandra — architecture & implementation |
| [Redis Architecture](datastores/redis-architecture.md) | Redis — architecture & implementation deep dive |
| [RocksDB Internals](datastores/rocksdb-internals.md) | RocksDB internal architecture walkthrough |
| [YugabyteDB Architecture](datastores/yugabytedb-architecture.md) | YugabyteDB — architecture and technical implementation |
| [Ceph Architecture](datastores/ceph-architecture.md) | Ceph — architecture and internal implementation |

## Analytics

| Note | Topic |
|---|---|
| [ClickHouse Internals](analytics/clickhouse-architecture-internals.md) | ClickHouse architecture & internal implementation |
| [Flink Architecture](analytics/flink-architecture.md) | Apache Flink — architecture |
| [Flink Internals](analytics/flink-internals.md) | Apache Flink — internal implementation details |
| [Presto Architecture](analytics/presto-technical-architecture.md) | Presto (prestodb) — technical architecture deep dive |

## Computer Architecture

| Note | Topic |
|---|---|
| [Computer Architecture](computer-architecture/computer-architecture.md) | Hardware and software as one system — CPU, memory, I/O, firmware, kernel, runtimes, and how they interact |

## Fundamentals

| Note | Topic |
|---|---|
| [What Happens When You Run a Program](fundamentals/what-happens-when-you-run-a-program.md) | End-to-end trace of `./hello` — shell, fork/exec, ELF loader, scheduler, page faults, CPU pipeline, caches, DRAM, syscalls, exit |

## Data Structures

| Note | Topic |
|---|---|
| [Probabilistic Data Structures](datastructures/probablic-datastructures/probabilistic-data-structures.md) | Overview, taxonomy & decision guide — start here |
| [Bloom Filters & Cuckoo Filters](datastructures/probablic-datastructures/bloom-filters.md) | Approximate membership — Bloom, counting, blocked, Ribbon, cuckoo |
| [HyperLogLog](datastructures/probablic-datastructures/hyperloglog.md) | Cardinality estimation — FM sketch, LogLog, HLL, HLL++ |
| [Count-Min Sketch & Heavy Hitters](datastructures/probablic-datastructures/count-min-sketch.md) | Frequency estimation — Count-Min, Count Sketch, SpaceSaving, TinyLFU |
| [Quantile Sketches](datastructures/probablic-datastructures/quantile-sketches.md) | Percentile estimation — t-digest, DDSketch, KLL |
| [MinHash, SimHash & LSH](datastructures/probablic-datastructures/minhash-simhash-lsh.md) | Similarity sketches & near-duplicate detection |
| [Skip Lists](datastructures/probablic-datastructures/skip-lists.md) | Probabilistic balancing with exact answers |

## Software Engineering

| Note | Topic |
|---|---|
| [REST API Best Practices](software-engineering/rest-api-best-practices.md) | HTTP/REST API design grounded in RFC 9110/9457, Fielding, and the Microsoft, Google, Zalando, Stripe, and OWASP guidelines; includes granularity and mobile-client sections |
| [GraphQL Best Practices](software-engineering/graphql-best-practices.md) | GraphQL schema, pagination, mutation, error, performance, and security practices grounded in the spec, graphql.org, Relay, Shopify, GitHub, Apollo, and OWASP |

## AI

| Note | Topic |
|---|---|
| [llama.cpp Inference Internals](ai/llama-cpp-inference-internals.md) | llama.cpp inference architecture and internals |
| [Code Puppy Architecture](ai/code_puppy_architecture.md) | Code Puppy — architecture & technical implementation |
