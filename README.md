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

## Fundamentals

| Note | Topic |
|---|---|
| [Probabilistic Data Structures](fundamentals/probabilistic-data-structures.md) | Overview, taxonomy & decision guide — start here |
| [Bloom Filters & Cuckoo Filters](fundamentals/bloom-filters.md) | Approximate membership — Bloom, counting, blocked, Ribbon, cuckoo |
| [HyperLogLog](fundamentals/hyperloglog.md) | Cardinality estimation — FM sketch, LogLog, HLL, HLL++ |
| [Count-Min Sketch & Heavy Hitters](fundamentals/count-min-sketch.md) | Frequency estimation — Count-Min, Count Sketch, SpaceSaving, TinyLFU |
| [Quantile Sketches](fundamentals/quantile-sketches.md) | Percentile estimation — t-digest, DDSketch, KLL |
| [MinHash, SimHash & LSH](fundamentals/minhash-simhash-lsh.md) | Similarity sketches & near-duplicate detection |
| [Skip Lists](fundamentals/skip-lists.md) | Probabilistic balancing with exact answers |

## AI

| Note | Topic |
|---|---|
| [llama.cpp Inference Internals](ai/llama-cpp-inference-internals.md) | llama.cpp inference architecture and internals |
| [Code Puppy Architecture](ai/code_puppy_architecture.md) | Code Puppy — architecture & technical implementation |
