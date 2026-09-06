# Computer Science: From Zero to Expert

A self-contained curriculum for someone with **no prior computer science background** who wants to reach genuine expertise. Each document assumes only the ones before it. Read them in order the first time; use them as references afterward.

## The curriculum

| # | Document | What it covers | Why this position |
|---|---|---|---|
| 1 | [How Computers Work](01-how-computers-work.md) | Binary, logic gates, CPU, memory, assembly | Everything runs on this machine |
| 2 | [Programming Fundamentals](02-programming-fundamentals.md) | Variables, control flow, functions, OOP, functional programming, paradigms | The craft everything else is practiced in |
| 3 | [Data Structures](03-data-structures.md) | Arrays, lists, hash tables, trees, heaps, graphs | How programs organize data |
| 4 | [Algorithms](04-algorithms.md) | Big-O, sorting, searching, graphs, dynamic programming | How programs solve problems efficiently |
| 5 | [Operating Systems](operating-systems.md) | Processes, memory, files, concurrency, the kernel | How programs share one machine |
| 5b | [Virtualization](virtualization.md) | Hypervisors, VT-x/EPT, virtio, containers, microVMs, live migration | How one machine becomes many (deep dive after OS) |
| 6 | [Computer Networks](05-computer-networks.md) | IP, TCP, DNS, HTTP, how the internet works | How machines talk to each other |
| 7 | [Databases](06-databases.md) | Relational model, SQL, transactions, indexing, internals | How data survives and scales |
| 8 | [Theory of Computation](07-theory-of-computation.md) | Discrete math, automata, computability, P vs NP | What computers can and cannot do |
| 9 | [Compilers & Programming Languages](08-compilers-and-languages.md) | Parsing, type systems, interpreters, runtimes, GC | How code becomes execution |
| 10 | [Software Engineering](09-software-engineering.md) | Version control, testing, design, architecture, working in teams | How real software gets built and survives |
| 11 | [Security & Cryptography](10-security-and-cryptography.md) | Crypto primitives, TLS, common attacks, secure design | How systems are attacked and defended |

## Where to go next (deeper dives already in these notes)

- **Distributed systems** — [../distributed-systems/](../distributed-systems/) (time & ordering, replication, consensus, stream processing)
- **Database internals** — [../datastores/](../datastores/) (PostgreSQL, MySQL, Redis, RocksDB, Cassandra...)
- **Probabilistic data structures** — [../datastructures/probablic-datastructures/](../datastructures/probablic-datastructures/) (bloom filters, HyperLogLog, sketches)
- **Big-data engines** — [../analytics/](../analytics/) (Spark, Flink, ClickHouse, Presto)
- **Containers** — [../infra/docker-deep-dive.md](../infra/docker-deep-dive.md)
- **AI/LLM applications** — [../ai/](../ai/)

## How to use this curriculum

1. **Read actively.** Every document has runnable commands and exercises — do them. Reading about a hash table teaches you *about* hash tables; implementing one teaches you hash tables.
2. **Learn one language early** (Python is the friendliest start; add C during the OS document; add one typed language — Go, Java, or Rust — during Software Engineering).
3. **Build things between documents.** After Algorithms: solve problems on LeetCode/Advent of Code. After Networks: build an HTTP server. After Databases: build something with PostgreSQL. Projects are where the knowledge compounds.
4. **Expect spiral learning.** Concepts reappear at increasing depth (caching appears in hardware, OS, databases, and networks). That repetition is the point — each pass deepens the model.

## The ten-thousand-foot view

Computer science is one long story about **layers of abstraction**: physics → transistors → logic gates → CPU → machine code → operating system → programming language → data structures & algorithms → applications → networks of applications. Each layer hides the complexity of the one below and exposes a simpler interface above. Expertise is (a) knowing your layer deeply, (b) understanding one or two layers below it, and (c) knowing when an abstraction is leaking.
