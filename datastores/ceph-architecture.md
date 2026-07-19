# Ceph — Architecture and Internal Implementation

> Analysis of the Ceph source tree at `C:\opensource\cpeh` (version **20.0.0, "Tentacle"**, dev branch).
> Depth target: staff/principal engineer. All class names and paths refer to actual files in this tree.

---

## Table of Contents

1. [What Ceph Is, and the Core Design Bet](#1-what-ceph-is-and-the-core-design-bet)
2. [Repository Layout](#2-repository-layout)
3. [RADOS Object Model and Addressing](#3-rados-object-model-and-addressing)
4. [CRUSH: Deterministic Placement](#4-crush-deterministic-placement)
5. [Cluster Maps and Epoch-Based Coordination](#5-cluster-maps-and-epoch-based-coordination)
6. [The Monitor: Paxos and Cluster State](#6-the-monitor-paxos-and-cluster-state)
7. [The Messenger: Wire Protocol and Networking](#7-the-messenger-wire-protocol-and-networking)
8. [Authentication: CephX](#8-authentication-cephx)
9. [The OSD: Op Pipeline, PGs, and Peering](#9-the-osd-op-pipeline-pgs-and-peering)
10. [Replication and Erasure-Coded Backends](#10-replication-and-erasure-coded-backends)
11. [BlueStore: The Storage Engine](#11-bluestore-the-storage-engine)
12. [Client Side: librados and the Objecter](#12-client-side-librados-and-the-objecter)
13. [End-to-End Write Path Walkthrough](#13-end-to-end-write-path-walkthrough)
14. [RBD: Block Storage](#14-rbd-block-storage)
15. [CephFS: The MDS and POSIX Semantics](#15-cephfs-the-mds-and-posix-semantics)
16. [RGW: Object Gateway](#16-rgw-object-gateway)
17. [ceph-mgr and Orchestration](#17-ceph-mgr-and-orchestration)
18. [Crimson and SeaStore: The Next-Generation OSD](#18-crimson-and-seastore-the-next-generation-osd)
19. [Cross-Cutting Infrastructure](#19-cross-cutting-infrastructure)
20. [Consistency Model and Failure Handling — Summary](#20-consistency-model-and-failure-handling--summary)

---

## 1. What Ceph Is, and the Core Design Bet

Ceph is a distributed storage system exposing three interfaces over a single substrate:

- **RADOS** — a strongly-consistent, self-healing object store (the substrate itself, via `librados`)
- **RBD** — virtual block devices striped over RADOS objects
- **CephFS** — a POSIX filesystem with a separate metadata server tier (MDS)
- **RGW** — an S3/Swift-compatible HTTP object gateway layered on RADOS

The foundational design decision, which explains nearly every internal mechanism, is:

> **There is no metadata lookup on the data path.** Data placement is *computed*, not *looked up*.

A client holding a compact **cluster map** can independently compute which OSDs (Object Storage Daemons) hold any object, using the **CRUSH** pseudo-random placement function. Consequences:

1. No central directory/lookup service to bottleneck or shard (contrast: HDFS NameNode, GFS master).
2. Clients talk **directly** to the primary OSD for each object; the monitor cluster is only a coordination/consensus service, off the data path.
3. All coordination reduces to **map epoch agreement**: every message carries the sender's map epoch; peers with stale maps fetch increments before proceeding. This is the system-wide mechanism for handling topology changes.
4. Recovery is fully decentralized: each placement group (PG) heals itself via peer-to-peer protocols among its member OSDs.

The second big bet is **primary-copy replication with synchronous acknowledgment**: RADOS is CP in CAP terms. A write is acknowledged only after every OSD in the acting set has it durable. Availability during faults is restored by *reconfiguring* (peering onto a new acting set) rather than by weakening consistency.

The codebase is C++ (C++20/23-era), ~4M+ LOC including bundled submodules, built with CMake (`CMakeLists.txt`, min 3.22). Daemon entry points are thin: `src/ceph_osd.cc`, `src/ceph_mon.cc`, `src/ceph_mds.cc`, `src/ceph_mgr.cc`, plus `radosgw` under `src/rgw`.

---

## 2. Repository Layout

| Path | Contents |
|---|---|
| `src/mon/` | Monitor: Paxos, elections, all cluster-state services (`*Monitor.cc`) |
| `src/osd/` | Classic OSD: PG machinery, peering, replication/EC backends, scrub, scheduler |
| `src/os/` | `ObjectStore` abstraction; `bluestore/` (production engine), `memstore/`, `fs/` |
| `src/kv/` | `KeyValueDB` abstraction; `RocksDBStore` wrapper |
| `src/msg/` | Messenger: `async/` event-driven network stack, protocol v1/v2 |
| `src/osdc/` | Client-side RADOS machinery: `Objecter`, `Striper`, `ObjectCacher` |
| `src/librados/`, `src/libradosstriper/` | Public RADOS client API |
| `src/librbd/` | Block device client library |
| `src/mds/`, `src/client/` | CephFS metadata server and POSIX client (`Client.cc`, fuse) |
| `src/rgw/` | Object gateway; `driver/` holds SAL backends (rados, dbstore, posix, d4n…) |
| `src/mgr/`, `src/pybind/mgr/` | Manager daemon C++ core + Python modules (dashboard, cephadm, balancer…) |
| `src/crimson/` | Next-gen Seastar-based OSD (crimson-osd) and SeaStore engine |
| `src/crush/` | CRUSH map + mapping algorithm (core in C: `mapper.c`, `crush.c`) |
| `src/erasure-code/` | EC plugin framework: jerasure, isa-l, clay, shec, lrc |
| `src/cls/` | "Object classes" — server-side stored procedures (cls_rbd, cls_rgw, cls_lock…) |
| `src/common/`, `src/include/` | Config, throttles, perf counters, encoding, `buffer::list`, types |
| `src/messages/` | All wire message definitions (`MOSDOp.h`, `MOSDRepOp.h`, …) |
| `src/auth/` | CephX protocol implementation |
| `src/cephadm/` | Deployment tool (Python zipapp), pairs with `pybind/mgr/cephadm` |
| `src/dmclock/` | dmClock QoS library used by the mClock op scheduler |
| Submodules | `rocksdb`, `seastar`, `spdk`, `dpdk`, `arrow`, `fmt`, `googletest`, `isa-l`, `zstd`, `s3select`, … |
| `qa/` | Teuthology integration test suites; `src/test/` unit tests |
| `doc/` | Sphinx docs, including `doc/dev/` internals notes |

---

## 3. RADOS Object Model and Addressing

### 3.1 Objects, pools, namespaces

The unit of storage is an **object**: named blob (typically ≤ 4 MiB by convention of the upper layers) with three co-located facets, all updatable in one atomic transaction:

- **data** — a byte extent (sparse-capable)
- **xattrs** — small key/value attributes (stored inline in the onode in BlueStore)
- **omap** — an ordered key/value map of arbitrary size (backed by RocksDB in BlueStore; this is what makes RGW bucket indexes and CephFS directory objects feasible)

Objects live in **pools** (an int64 id + name), each with its own replication/EC policy, CRUSH rule, and PG count. Within a pool, an optional **namespace** string partitions the name space (used for cap-based multi-tenancy).

### 3.2 hobject_t / ghobject_t

The canonical internal object key (`src/common/hobject.h`) is:

```
hobject_t  = { pool, namespace, oid(name), key(optional locator), snap, hash }
ghobject_t = { hobject_t, generation, shard_id }
```

- `hash` is the rjenkins hash of the name (or the explicit locator key), computed once and carried around — it determines PG membership and sort order.
- `snap` distinguishes HEAD (`CEPH_NOSNAP`), snapshot clones (snapid), and `snapdir`.
- `ghobject_t` adds `shard_id` (which EC shard, `NO_SHARD` for replicated) and `generation` (used for rollback of EC objects). This is what the ObjectStore layer keys on.

### 3.3 Placement groups

Objects are not placed individually — they are grouped into **placement groups (PGs)**:

```
pg = (pool, ps)   where ps = stable_mod(hash(object_name), pg_num, pgp_num_mask)
OSD set = CRUSH(cluster_map, pg, rule)
```

`stable_mod` (see `src/osd/osd_types.h`) is designed so that doubling `pg_num` splits each PG into exactly two children (PG "splitting"), keeping data movement local to the split. PGs are the unit of: peering, replication ordering, recovery, scrub, and stats. A pool has `pg_num` PGs; `pgp_num` controls how many placement seeds CRUSH sees (allowing split-then-rebalance in two phases). The `pg_autoscaler` mgr module adjusts these online.

Per-PG ordering is a load-bearing invariant: **all replicated writes within a PG are applied in a single total order defined by the primary** (`eversion_t = (epoch, version)` monotonic per PG). This gives RADOS per-object linearizability at low bookkeeping cost.

### 3.4 Snapshots

RADOS has native COW snapshot support: each object logically has a HEAD plus **clones** keyed by snapid. Snap metadata is either **self-managed** (client-provided `SnapContext`, used by RBD/CephFS) or **pool snaps**. On write, the OSD compares the op's SnapContext with the object's `snapset` and clones the object if needed (`PrimaryLogPG::make_writeable`). Deleted snaps are lazily reclaimed by **snaptrim**, driven by `SnapMapper` (`src/osd/SnapMapper.cc`), which maintains a reverse snapid→objects index in omap.

---

## 4. CRUSH: Deterministic Placement

CRUSH (`src/crush/`) maps `(pg, rule, crush_map)` → an ordered list of OSDs. The compute core is deliberately plain C (`mapper.c`) so the same code runs in the Linux kernel client.

### 4.1 Structure

The CRUSH map is a weighted hierarchy of **buckets** (root → datacenter → rack → host → osd, types are user-definable). Devices carry a **device class** (hdd/ssd/nvme); internally each class gets a "shadow tree" so rules can target a class without separate hierarchies. Each **rule** is a small program:

```
take <root>
choose/chooseleaf firstn|indep <n> type <bucket-type>
emit
```

- `firstn` — for replicated pools (order matters little, gaps compacted)
- `indep` — for EC pools (positional: a failed choice yields a hole, preserving shard positions)

### 4.2 straw2

The dominant bucket algorithm is **straw2**: for each child `i`, compute `draw_i = ln(hash(pg, i, r)) / weight_i` (fixed-point via `crush_ln()`), pick the max. Key property: changing one child's weight only moves items into/out of *that child* — provably minimal data movement, and the reason weight changes don't reshuffle the world. Legacy algorithms (uniform, list, tree, straw1) survive for compat; behavior knobs are versioned as **tunables** (profiles named by release) so old clients aren't broken silently — clients must advertise matching feature bits.

### 4.3 Corrections on top of CRUSH

Pure CRUSH output is statistically balanced but not perfectly, and sometimes infeasible (too many failed retries). Three override layers live in the OSDMap, applied in order:

1. **`pg_temp`** — temporary acting set published during recovery (e.g., keep an old replica primary while the CRUSH-chosen one backfills); `primary_temp` overrides just the primary.
2. **`pg_upmap` / `pg_upmap_items`** — explicit per-PG remap exceptions, the mechanism behind the `balancer` mgr module (upmap mode achieves near-perfect PG distribution). `pg_upmap_primary` similarly balances read/primary load.
3. **reweight** — legacy per-OSD probabilistic rejection factor.

`OSDMap::pg_to_up_acting_osds()` (`src/osd/OSDMap.cc`) is the single authority combining all of these. **up set** = what CRUSH+upmap says; **acting set** = up set as modified by pg_temp — the set actually serving I/O.

---

## 5. Cluster Maps and Epoch-Based Coordination

Every subsystem's authoritative state is a versioned map, owned by the monitors:

| Map | Owner service | Consumers |
|---|---|---|
| `MonMap` | MonmapMonitor | everyone (bootstrap) |
| `OSDMap` | OSDMonitor | OSDs, all clients |
| `FSMap`/`MDSMap` | MDSMonitor | MDS, CephFS clients |
| `MgrMap` | MgrMonitor | mgr, daemons reporting metrics |
| CRUSH map | embedded in OSDMap | everyone |

`OSDMap` (`src/osd/OSDMap.h`) is the heart: OSD up/down + in/out states (orthogonal: *down* is liveness, *out* triggers data re-placement), addresses, weights, pools (`pg_pool_t`), CRUSH map blob, upmaps, pg_temp, blocklist (fencing dead clients — critical for RBD exclusive locking), `up_thru` vector (see peering), and required feature/compat flags.

Maps evolve by **epoch** with **incremental deltas** (`OSDMap::Incremental`) distributed lazily: there is no broadcast. OSDs gossip map increments to peers; clients get them from OSD replies or monitor subscriptions (`MonClient::sub_want("osdmap", ...)`). Every `MOSDOp`, `MOSDRepOp`, and peering message carries an epoch; a recipient that's behind stalls the op, fetches increments, applies them, and re-evaluates. Correctness never depends on freshness — only on the rule that *an OSD will not serve a PG in an interval it can't prove it is the acting primary for*.

---

## 6. The Monitor: Paxos and Cluster State

`src/mon/`. Monitors form a small quorum (3 or 5) running **one shared Paxos instance** (`Paxos.cc`) — not multi-decree parallel Paxos; it's a serialized, batched, leader-based variant where each "commit" is one transaction blob applied to the local store.

### 6.1 Storage

`MonitorDBStore.h` wraps RocksDB. Every Paxos commit is a `MonitorDBStore::Transaction` (encoded bufferlist) written atomically. Committed Paxos values themselves are stored under `paxos/<version>`, so lagging peers catch up by replaying stored transactions; a `stashed` full snapshot bounds replay.

### 6.2 Services

Each area of cluster state is a `PaxosService` subclass multiplexed over the single Paxos: `OSDMonitor` (by far the largest — owns the OSDMap and all `ceph osd ...` commands), `MonmapMonitor`, `AuthMonitor` (cephx keys), `MDSMonitor`, `MgrMonitor`, `ConfigMonitor` (centralized `ceph config` database), `LogMonitor`, `HealthMonitor`, `KVMonitor`, `NVMeofGwMon`. The pattern:

- each service accumulates `pending` state; `propose_pending()` encodes it into the shared transaction and Paxos-commits it;
- reads are served from any monitor holding a valid **lease** granted by the leader (bounded staleness reads without a quorum round-trip);
- writes are forwarded to the leader.

`OSDMonitor` also implements failure detection *policy*: OSDs heartbeat each other (not the mons) and file `MOSDFailure` reports; the monitor requires enough distinct reporters (`mon_osd_min_down_reporters`, weighted by reporter subtree) before marking an OSD down — protecting against a partitioned OSD accusing everyone.

### 6.3 Elections

`Elector.cc` / `ElectionLogic.cc`. Rank-based election with three strategies: **classic** (lowest rank wins), **disallow** (blacklist certain mons from leading), and **connectivity** — each monitor maintains a `ConnectionTracker` (`ConnectionTracker.cc`) scoring peer reachability (exponentially decayed ping success), and the election prefers the best-connected monitor. This fixed the classic pathology where a network-flapping rank-0 monitor caused election storms.

Monitor↔monitor sync for a new/lagged mon is a full store sync (`Monitor::sync_*`), chunked over the wire.

---

## 7. The Messenger: Wire Protocol and Networking

`src/msg/`. All inter-daemon and client communication uses the `Messenger` abstraction: entity-addressed (`entity_name_t` like `osd.3`, `client.4123`), typed messages (`src/messages/M*.h`, ~200 types), with per-peer-type **`Policy`** declaring lossy/lossless, client/server, and throttle limits.

- **Lossy** (client↔OSD): on failure, drop state; the *client* (Objecter) owns resends. Sessions are cheap.
- **Lossless peer** (OSD↔OSD, mon↔mon): the messenger itself guarantees exactly-once ordered delivery across reconnects — connections carry sequence numbers, unacked messages are replayed on reconnect, and connection races are resolved deterministically (by address comparison + cookies). Higher layers (peering) get to assume FIFO reliable channels.

### 7.1 AsyncMessenger

The only production implementation (`msg/async/`): a small pool of worker threads (default 3) each running an `EventCenter` epoll/kqueue loop; every connection is pinned to one worker (no cross-thread migration ⇒ per-connection state is single-threaded). Pluggable `NetworkStack` backends: POSIX (production), RDMA (ibverbs), DPDK (userspace TCP). **Fast dispatch** delivers latency-critical messages (`MOSDOp`) directly on the messenger thread into the OSD, bypassing the ordered `DispatchQueue` used for control-plane messages.

### 7.2 Protocol v2

`ProtocolV2.cc` + `frames_v2.{h,cc}` (port 3300, `msgr2`). Frame = preamble (tag, 1–4 segment descriptors, CRC) + segments + epilogue. Properties worth knowing:

- **Negotiation phase** (banner/HELLO/AUTH) is extensible and encrypted-capable from the start, fixing v1's rigid handshake.
- **Modes**: `crc` (CRC32C integrity only) or `secure` — full-frame **AES-128-GCM** encryption (`crypto_onwire.cc`) with keys derived from the cephx session; on-wire compression is negotiable per connection (`compressor_registry.cc`).
- **Segment scatter/gather**: message header/front/middle/data are separate segments so the receiver can land data payloads into page-aligned buffers (important for zero-copy toward BlueStore and RDMA).
- **Reconnect/session continuation** carries `connect_seq`/`msg_seq` so lossless sessions resume without replay ambiguity.

Message memory uses `ceph::buffer::list` (`src/include/buffer.h`) — a rope of refcounted raw buffers; encode/decode and network I/O are all zero-copy appends/splices of these ptrs. This type is pervasive; nearly every serialized structure has `encode(..., bufferlist&)`/`decode`.

---

## 8. Authentication: CephX

`src/auth/`. Kerberos-like tickets without a synchronous KDC on every connection:

1. Client authenticates to a monitor with its secret key (`AuthMonitor` holds the key database); receives a **ticket** for the desired services plus a session key.
2. Tickets are encrypted with **rotating service keys** shared by all daemons of a type (rotated on `auth_service_ticket_ttl`); any OSD can validate a client ticket offline.
3. Tickets embed **caps** (capability strings parsed by `MonCap`/`OSDCap`/`MDSAuthCaps`), e.g. `osd 'allow rwx pool=rbd namespace=foo'` — enforced at the daemon on every op (`OSDCap.cc` grammar is a boost::spirit parser).
4. msgr2 secure mode binds the cephx session key into the AES-GCM channel keys, giving mutual auth + confidentiality.

---

## 9. The OSD: Op Pipeline, PGs, and Peering

`src/osd/` — the most intricate subsystem. One `ceph-osd` process per storage device.

### 9.1 Threading and op scheduling

```
msgr worker (fast dispatch)
  └─ OSD::ms_fast_dispatch → enqueue into one of N OSDShards (hash of pgid)
       └─ shard worker threads dequeue via OpScheduler
            └─ take PG lock → PrimaryLogPG::do_request(...)
                 └─ ObjectStore::queue_transaction(...)   (async)
                      └─ commit callbacks re-enqueue completions
```

- **`OSDShard`** (in `OSD.h`): the OSD is sharded (`osd_op_num_shards_*`); each shard owns a slice of PGs, its own scheduler queue and worker threads (`osd_op_num_threads_per_shard`). A PG is only ever touched under its `PG::lock` — coarse-grained but simple; the shard design bounds contention.
- **`OpScheduler`** (`osd/scheduler/`): default is **mClock** (`mClockScheduler.cc`, using `src/dmclock`) — reservation/weight/limit QoS across client ops, recovery, scrub, snaptrim, pg deletion. Profiles (`high_client_ops`, `balanced`, `high_recovery_ops`) auto-derive per-class parameters from a measured/configured per-OSD IOPS capacity (`osd_mclock_max_capacity_iops_*`). Cost is modeled in scaled units so 4K and 4M ops compete fairly. WPQ remains as fallback.
- Everything that runs in the pipeline is an `OpSchedulerItem` wrapping either a client `OpRequest` or an internal event (peering message, recovery, scrub chunk) — internal work is *scheduled*, not privileged.

### 9.2 PG core: PrimaryLogPG

`PrimaryLogPG.cc` (~16 kLOC) implements the primary-copy protocol for both replicated and EC pools:

- `do_op()` performs admission checks: map epoch, PG state (must be *active*), object context (`ObjectContext` with per-object rw locks and the `object_info_t`/`SnapSet` metadata), ordering (writes to one object serialize; `RWState` allows read parallelism), and dup detection.
- **Dup detection / exactly-once**: client ops carry a `reqid_t (client, tid)`. The PG log retains recent reqids (including compacted `pg_log_dup_t` entries beyond the log window). A resent, already-committed op is answered from the log with its original result code and version — this is what makes client resends across primary failover safe.
- `execute_ctx()` evaluates the op vector (RADOS ops are vectors of sub-ops — `CEPH_OSD_OP_WRITE`, `CMPXATTR`, `OMAPSETKEYS`, class calls, etc.) against the object, producing an `OpContext` holding a `PGTransaction` (logical, backend-agnostic mutation description) plus the `pg_log_entry_t`.
- **Object classes** (`src/cls/`, loaded via `ClassHandler`): server-side plugins invoked as ops (`CEPH_OSD_OP_CALL`). `cls_rbd` and `cls_rgw` implement atomic read-modify-write logic (e.g., bucket-index updates) *inside* the OSD, which is Ceph's answer to distributed transactions for upper layers: turn a multi-node transaction into a single-object atomic op.
- Writes become a `RepGather` and go to the backend (`issue_repop`); the reply is sent when all acting-set members commit.

`PGBackend` abstracts replication strategy: `ReplicatedBackend` vs `ECBackend` (§10). Reads are served by the primary by default; **balanced reads** (replicas) and localized reads are allowed for replicated pools when the client sets flags; since Reef/Squid EC pools can also serve some reads from shards.

### 9.3 PG log

`PGLog.{h,cc}`: a bounded in-memory + on-disk (pgmeta omap) log of recent operations per PG — entries `pg_log_entry_t {op, soid, version(eversion_t), prior_version, reqid, mod_desc...}`. It serves three roles:

1. **Recovery by log diff** — a restarting replica whose last version is inside the primary's log window computes exactly the objects it's missing (the `missing` set) and copies only those (log-based recovery), versus **backfill** (full ordered scan) when the divergence exceeds the log (`osd_min_pg_log_entries`…`osd_max_pg_log_entries`, trimmed against `min_last_complete_ondisk`).
2. **Divergent entry rollback** — entries carry enough (`ObjectModDesc`) to locally roll back ops that a new authoritative log doesn't contain.
3. **Dup-op detection** (above).

### 9.4 Peering — the consensus mechanism for PGs

`PeeringState.{h,cc}` (~7 kLOC header) — a **boost::statechart** hierarchical state machine (`PeeringMachine`), shared verbatim between the classic OSD and crimson (that's why it was extracted out of `PG`). The machine (states at `PeeringState.h:740+`):

```
Started
├─ Primary
│   ├─ Peering
│   │   ├─ GetInfo      – query pg_info_t from all live members of past intervals
│   │   ├─ GetLog       – choose authoritative log (find_best_info), pull it
│   │   ├─ GetMissing   – fetch log tails of peers to compute their missing sets
│   │   └─ WaitUpThru   – wait for monitor to record up_thru for this interval
│   └─ Active
│       ├─ Activating / Recovering / Backfilling / Clean ...
└─ ReplicaActive / Stray / ...
```

Concepts a reviewer must hold:

- **Past intervals**: the sequence of `(epoch range, acting set)` since the PG was last known clean. Any interval in which the PG *may have gone active* could hold writes; peering must contact **at least one survivor of every maybe-active interval** (`PastIntervals::PriorSet`) or refuse to proceed (the alternative is silent data loss). If required survivors are down and `osd_find_best_info_ignore_history_les` isn't forced, the PG stays `down` / becomes `incomplete` — availability sacrificed for consistency, by design.
- **`up_thru`**: an OSD publishes into the OSDMap "I was alive through epoch E" *before* activating a PG. This closes the race where an interval existed on paper but no writes could have happened in it — peering can prune such intervals. It's the subtle piece that makes the "maybe-active" computation sound.
- **`find_best_info()`**: picks the authoritative log by max `last_epoch_started`, then max `last_update`, then longest log tail — i.e., the replica provably containing every acknowledged write.
- **`last_epoch_started` (LES)**: recorded on activation; the fencepost separating "history that peering already reconciled" from the current interval.
- Result of peering: an **acting set** with complete knowledge of each member's `missing` set. The PG then goes `active` (serves I/O immediately — missing objects are recovered on demand if a client touches them, `wait_for_unreadable_object`) and drains recovery/backfill in the background. If the acting set is smaller than `min_size`, the PG stays inactive (`undersized`+`peered`), refusing writes rather than risking split-brain on subsequent failures.

Peering is *not* Paxos — it needs no quorum among OSDs because the monitor's OSDMap serializes interval membership; peering is the deterministic recovery procedure over that history.

### 9.5 Scrub, recovery mechanics, watch/notify

- **Scrub** (`osd/scrubber/`): its own state machine (also boost::statechart, `ScrubMachine`). Chunked: reserve replicas → freeze writes for a small object range → collect `ScrubMap`s from all shards → compare (shallow: sizes/attrs/omap digests; deep: full-data CRC32C against BlueStore's stored checksums) → persist inconsistencies to `ScrubStore` (omap of the pgmeta object), queryable via `rados list-inconsistent-obj`. `repair` re-runs and rewrites bad shards from the authoritative copy.
- **Recovery/backfill throttling**: `osd_max_backfills`, `osd_recovery_max_active`, plus async reservation protocol (local + remote `AsyncReserver`) so a flapping node can't stampede the cluster; all recovery I/O flows through the mClock scheduler as background-class ops.
- **Watch/Notify** (`Watch.cc`): clients register persistent watches on an object; a `notify` op fans out to all watchers and aggregates acks with timeout. This is RADOS's pub/sub primitive — RBD uses it for header invalidation and exclusive-lock handoff; the mons/mgr use it internally. Watch state survives primary failover via the object's `watchers` metadata and client ping/reconnect.

---

## 10. Replication and Erasure-Coded Backends

### 10.1 ReplicatedBackend

`ReplicatedBackend.cc`: primary encodes the `PGTransaction` into a per-replica `MOSDRepOp` containing the serialized **ObjectStore transaction** + log entries; replicas apply blindly (no re-execution — replicas are passive appliers, so replica state is byte-identical by construction) and reply on commit. The primary acks the client when *all* acting members (including itself) commit. Since BlueStore, there is no separate "applied vs committed" ack — commit implies readable.

### 10.2 Erasure coding — two backends, one switch

Tentacle ships **two** EC implementations, selected per-pool by `ECSwitch.h` (`allows_ecoptimizations` pool flag):

- **Legacy** (`ECBackendL.cc`, `ECCommonL.cc`, `ECTransactionL.cc`, `ECUtilL.cc`): full-stripe RMW — any sub-stripe write reads the whole stripe, re-encodes, rewrites every shard. Reads must reconstruct from `k` shards through the primary. Rollback support via object **generations** (`ghobject_t.generation`): shards keep old generations until the write is fully committed cluster-wide, enabling per-shard rollback during peering.
- **Optimized / "fast EC"** (new `ECBackend.cc` + `ECExtentCache.cc`, the Tentacle headline feature): **partial reads** (read only the shards/extents needed, direct from shard OSDs) and **partial writes** (update only affected data shards + parity deltas), an extent cache to coalesce sub-stripe RMW, and avoidance of touching empty-beyond-EOF shards. Turns EC from "archive tier" into something plausible for RBD/CephFS block-ish workloads.

Plugin layer (`src/erasure-code/`): `ErasureCodeInterface` with plugins **jerasure** (+`gf-complete` SIMD Galois-field kernels), **isa-l** (Intel-optimized), **clay** (coupled-layer, bandwidth-optimal repair), **shec**, **lrc** (locally repairable, hierarchy-aware). The EC profile (`k`, `m`, plugin, `crush-failure-domain`, `stripe_unit`) is stamped into the pool.

EC pools use `indep` CRUSH placement (shard position = CRUSH position); each shard is a distinct `ghobject_t` with `shard_id`; the PG identity for shard s is `pgid.s`.

---

## 11. BlueStore: The Storage Engine

`src/os/bluestore/` — the production `ObjectStore`. Design premise: **filesystems are the wrong substrate for an object store** (double journaling, no atomic data+metadata transactions). BlueStore owns a raw block device and gets transactional metadata from an embedded RocksDB.

### 11.1 Layout and components

```
┌────────────────────────────── raw device ──────────────────────────────┐
│ superblock │  BlueFS region(s) — hosts RocksDB files  │  data extents  │
└─────────────────────────────────────────────────────────────────────────┘
                    ▲                                        ▲
                 RocksDB  ◄── BlueRocksEnv ── BlueFS      Allocator (in-mem)
                (all metadata, WAL, deferred writes)      FreelistManager (persisted)
```

- **BlueFS** (`BlueFS.cc`): a deliberately primitive filesystem — no directories-in-directories, whole-file-ish allocation (large extents), a single replayable journal of metadata ops — just enough to host RocksDB's `.sst`/`.log` files. Supports up to three devices by speed class (`db.wal` → NVMe/NVRAM, `db` → SSD, `db.slow` → spillover to the main device), with RocksDB level-targeted placement. `BlueRocksEnv` adapts it to RocksDB's `Env` API. BlueFS and BlueStore share the main device via `bluefs_shared_alloc` — one allocator, two clients.
- **RocksDB schema** (prefixes at `BlueStore.cc:131`): `S` superblock fields, `T` statfs, `C` collections (`cnode_t` per PG), `O` **onodes**, `M`/`m`/`p`/`P` omap data (global / per-pool / per-PG / pgmeta variants — the per-PG `p` prefix keys omap by `(pool, hash, onode-id)` so a PG's omap is physically clustered, making PG deletion and splitting range-deletes), `L` deferred-write intents, `B`/`b` freelist, `X` shared blobs.
- **Onode key encoding**: `(shard, reversed-pool, reversed-hash-nibbles, namespace, name, snap, gen)` — hash nibbles are reversed so that RocksDB's sort order equals PG-collection enumeration order (`ghobject_t` ordering), letting collection listing and PG splits be prefix scans.

### 11.2 Data structures: onode → extent map → blob → pextent

```
Onode (per object, cached; `O` key in RocksDB)
 └─ ExtentMap  — logical offset → lextent   (sharded: large maps split into
                                             separately-encoded/loaded shards)
      └─ Blob  — unit of allocation/checksum/compression state (bluestore_blob_t)
           ├─ csum: type (crc32c default) + chunk size (4K) + vector of checksums
           ├─ optional compression header (alg, logical vs compressed length)
           └─ pextents: vector of (device offset, length)
 SharedBlob (`X` key) — refcounted extents shared between clones (snapshots);
                        ref counting via bluestore_extent_ref_map_t
```

The **extent-map sharding** matters: a 4 MiB RBD object at 4K random-write granularity produces a large lextent map; encoding it monolithically per update would dominate write amplification, so it's split into shards (~`bluestore_extent_map_shard_target_size`) that are dirtied/re-encoded independently, and only decoded on demand ("spanning blobs" handle blobs crossing shard boundaries).

**Caches** (`BlueStore.h`): sharded onode cache (2Q/LRU variants) + buffer (data) cache, with a `PriorityCache` arbiter that dynamically splits `osd_memory_target` between onode cache, buffer cache, and RocksDB block cache based on hit-rate pressure — this is the OSD's memory governor.

### 11.3 Write path

Min allocation unit `min_alloc_size` (4K on both HDD and SSD nowadays). Two regimes, chosen in `_do_write()` / the refactored `Writer.cc` (v2 write path):

1. **New-allocation write ("big write")**: aligned or beyond-EOF data goes to freshly allocated space (COW — never overwrite live extents in place), checksummed per 4K chunk, then the metadata (onode/blob updates) commits via RocksDB. Data hits disk *before* the KV commit references it; crash ⇒ orphaned extents at worst (reclaimed since the allocation is only persisted transactionally).
2. **Deferred write ("small write")**: sub-`min_alloc_size` overwrites of allocated space would need read-modify-write; instead the payload is embedded in the RocksDB transaction under prefix `L` (WAL), acked on KV commit, and later written in place and the `L` key deleted (`deferred_try_submit`, batched). Threshold `prefer_deferred_size` (HDD: 64K default — HDDs benefit from turning random writes into journal appends; SSD: small).

**Transaction state machine** (`TransContext` in `BlueStore.h`): `PREPARE → AIO_WAIT → IO_DONE → KV_QUEUED → KV_SUBMITTED → KV_DONE → (DEFERRED_QUEUED → DEFERRED_CLEANUP) → DONE`. A single **`_kv_sync_thread`** batches all pending txcs' RocksDB writes into one WAL sync (group commit — the central latency/throughput tradeoff point of the whole OSD), and a `_kv_finalize_thread` runs completions. Per-collection (=per-PG) sequencers preserve transaction order; unrelated PGs proceed in parallel.

**Checksums are end-to-end at rest**: every read verifies crc32c against blob metadata; scrub deep-verifies cross-replica. **Compression** (lz4/snappy/zstd, per-pool hints/modes) operates per blob with min-ratio gating (`compression_required_ratio`); partially-overwritten compressed blobs are handled by punching holes in the ref map and garbage-collecting via blob GC heuristics.

### 11.4 Allocators and freelist

In-memory `Allocator` hierarchy (`Allocator.h`): **HybridAllocator** (default — AVL range tree for the common case, degrading to bitmap for fragmented regions), `AvlAllocator`, `BitmapAllocator` (fast, constant-memory), `BtreeAllocator`, `Btree2Allocator`, legacy `StupidAllocator`. Persistence is decoupled: `BitmapFreelistManager` mirrors allocation state in RocksDB (`b` keys, XOR-merge operator so alloc/free are blind writes) — or, with **NCB (null freelist)** mode, BlueStore skips persisting the freelist entirely and rebuilds the allocation map at startup from onode metadata (fast path via an allocation-map file on clean shutdown; full onode scan after a crash). `fsck`/`repair` (invocable offline via `ceph-bluestore-tool`, `bluestore_tool.cc`) cross-checks onodes ↔ freelist ↔ BlueFS.

`src/blk/` provides the block layer under BlueStore: libaio/io_uring (`KernelDevice`), SPDK NVMe userspace (`NVMEDevice`), PMEM.

---

## 12. Client Side: librados and the Objecter

`src/librados/` is a thin veneer; the real machinery is **`Objecter`** (`src/osdc/Objecter.cc`, ~4 kLOC of protocol logic):

- Holds the current `OSDMap`; `_calc_target()` computes `(pg, primary/replica, epoch)` per op — including locator/namespace hashing, upmap application, and read-from-replica policy.
- Maintains one lossy session per OSD; ops are tagged with `tid` and tracked in `inflight_ops`.
- **Resend logic** is where correctness lives: on any map increment, every in-flight op's target is recomputed; if the acting set or primary changed (interval change), the op is resent to the new primary. Writes are only resent when safe (the new primary's dup detection dedups); `PAUSERD/PAUSEWR`/full-pool flags gate admission. `last_force_op_resend` in the pool epoch forces resend on certain pool changes (e.g., PG splits).
- **Linger ops** implement watch/notify persistence: a watch is an op that re-registers itself on every interval change, transparently surviving primary failover.
- Backpressure: byte-budget throttle on in-flight ops (`objecter_inflight_op_bytes`).

`MonClient` (`src/mon/MonClient.cc`) handles monitor session, authentication, map subscriptions, and command routing. `libradosstriper` adds client-side striping over librados; `Striper` (`src/osdc/Striper.cc`) is the shared file→object extent mapper used by RBD and CephFS (`ceph_file_layout`: `stripe_unit`, `stripe_count`, `object_size`).

---

## 13. End-to-End Write Path Walkthrough

A 4 KiB RBD write, replicated ×3, as it actually executes:

1. **librbd** maps image offset → object (`rbd_data.<id>.<objno>`, 4 MiB default) via `Striper`; issues `write(off,len)` through its `ImageRequest`/io_object pipeline (acquiring the exclusive lock if configured).
2. **Objecter**: `hash(name) → ps`; `stable_mod(ps, pg_num) → pg`; CRUSH+upmap+pg_temp → acting `[osd.7*, osd.2, osd.19]`. Sends `MOSDOp{epoch, pg, oid, ops[], reqid, snapc}` to osd.7 over msgr2 (AES-GCM if secure mode).
3. **osd.7**: fast dispatch → shard queue → mClock dequeue → PG lock. `PrimaryLogPG::do_op`: epoch check (fetch increments if behind), active check, `ObjectContext` + write lock, dup check against PG log, SnapContext vs SnapSet (clone if a new snap exists — COW happens *here*, on the OSD).
4. `execute_ctx` → `PGTransaction` + `pg_log_entry_t{version=(e,v+1)}` → `RepGather`; backend encodes an ObjectStore `Transaction` (data write + object_info xattr update + pg log omap append) and sends `MOSDRepOp` to osd.2, osd.19; queues its own transaction locally.
5. **Each OSD's BlueStore**: 4 KiB overwrite of an allocated extent → **deferred**: payload into the `L` key of the RocksDB batch alongside onode/log mutations; `_kv_sync_thread` group-commits the WAL → txc `KV_SUBMITTED` → commit callback. (An aligned/new write would instead do aio to fresh extents, *then* KV commit.)
6. Replicas send `MOSDRepOpReply(committed)`; when all three commits are in, osd.7 replies `MOSDOpReply{result, version}` to the client and unblocks any ops queued behind this one on the same object.
7. In-place disk write for the deferred payload happens later, batched; the `L` intent is deleted in a subsequent KV transaction.

Failure at any point resolves through the same two mechanisms: monitor marks osd down → new OSDMap epoch → Objecter resends to new primary → dup detection ensures exactly-once; peering reconciles the PG's log across the interval boundary.

---

## 14. RBD: Block Storage

`src/librbd/` (heavily subdirectoried: `io/`, `image/`, `operation/`, `exclusive_lock/`, `journal/`, `mirror/`, `cache/`, `migration/`, `crypto/`).

**On-disk format (v2)**: `rbd_id.<name>` → image id; `rbd_header.<id>` holds size/features/snaps/parent as omap (all header mutations go through `cls_rbd` for atomicity); `rbd_object_map.<id>` bitmap (2 bits/object: exists/clean) accelerates diff, flatten, delete; data objects `rbd_data.<id>.<objno>` are created lazily (thin provisioning is just RADOS sparseness).

Key mechanisms:

- **Layering/clones**: a clone's header references parent image+snap; reads fall through to the parent on ENOENT (client-side, using the object map to short-circuit); first write triggers **copy-up** (read parent object, write to child, then apply the op). `flatten` is a background copy-up sweep.
- **Exclusive lock** (`exclusive_lock/`, `ManagedLock`): cooperative single-writer via `cls_lock` on the header + watch/notify for handoff requests; features like object-map, fast-diff, journaling require it. Dead clients are fenced via OSDMap **blocklist** (the lock breaker blocklists the old owner's address — this is why RADOS blocklisting exists).
- **Journaling & mirroring**: with the journaling feature, every mutation is first appended to a RADOS-backed journal (`src/journal/`) then applied; `rbd-mirror` daemons tail the journal for async cross-cluster replication. **Snapshot-based mirroring** (the now-preferred mode) instead periodically mirror-snapshots and ships deltas (fast-diff), trading RPO granularity for much lower overhead.
- **Caching**: legacy shared `ObjectCacher`; modern **pwl** (persistent write-log) cache in `cache/pwl/` — a client-local, crash-consistent (PMEM or SSD) write-back log preserving barrier semantics.
- `krbd`/`rbd-nbd`/tcmu map images into kernels; `rbd migration` does live image moves (source→target with layered redirection).

---

## 15. CephFS: The MDS and POSIX Semantics

`src/mds/` + `src/client/`. Architectural split: **file data goes client↔OSD directly** (striped via `ceph_file_layout`); the MDS handles namespace + coherence only, and its own state lives in RADOS (a metadata pool) — the MDS is a *cache and lock manager*, not a store. `ceph-mds` daemons are therefore stateless-ish and rapidly replaceable.

### 15.1 Metadata representation

- In-memory: `CInode`, `CDentry`, `CDir` (`MDCache.cc` is a 14 kLOC cache of these). On-disk: each directory fragment is **one RADOS object** whose omap maps dentry name → embedded inode (inodes are stored *inside* their primary dentry — no separate inode table lookup on path traversal; hard links use "remote" dentries).
- **Dirfrags**: a directory can be split into power-of-two hash fragments (`frag_t`), each its own object — how huge directories scale and how hot directories parallelize across MDS ranks.
- **`MDLog`** (`MDLog.cc`, `events/`): every mutation is journaled (as `EMetaBlob` deltas inside `LogEvent`s) into a RADOS-backed journal *before* being acked, then lazily flushed to the dirfrag objects. `LogSegment`s track dirty items; journal replay reconstructs cache state on failover. Standby-replay MDSs tail the active's journal continuously for hot failover.
- Auxiliary tables: `InoTable` (free inode ranges per rank), `SnapServer`, `SessionMap`; `PurgeQueue` asynchronously deletes file data of unlinked inodes ("stray" dentries → purge).

### 15.2 Distributed cache coherence: caps and locks

The genuinely hard part. Two interlocking protocols:

- **Client capabilities** (`Capability.cc`): per-(inode, client) grants encoded as bitfields — `p`in, `A`uth (mode/uid), `L`ink, `X`attr, `F`ile, each with shared/exclusive plus file-specific rights (read/write/cache/buffer/lazy). E.g. a sole writer gets `Fcb` (may buffer writes client-side); when a second client opens the file, the MDS *revokes* buffering, forcing flush, and may put the inode in LAZY/mixed mode where all I/O goes synchronous to OSDs. POSIX coherence emerges from cap grant/revoke, not from per-op MDS round-trips.
- **Internal MDS locks** (`Locker.cc`, ~30 lock states in `locks.c`): each inode field group has a lock instance (`SimpleLock`, `ScatterLock`, `LocalLock`) governing which MDS rank and which clients may read/mutate. **ScatterLock** solves a CephFS-specific problem: directory stats (rstats — recursive size/ctime/count rollups, a headline feature) are updated by *children* potentially on other ranks; scatter state lets writes scatter to replicas and later gather for a coherent read.

### 15.3 Multi-MDS: dynamic subtree partitioning

The namespace is partitioned by **subtree**, dynamically: `MDBalancer.cc` measures per-dirfrag temperature (popularity decaying counters) and triggers `Migrator.cc` to **export** subtrees between ranks — a two-phase handoff (freeze subtree → transfer cache state + journal EExport/EImport on both sides) that's crash-consistent on either end. Ephemeral/static pins (`ceph.dir.pin` xattrs) let operators override the balancer, which in practice many deployments do. Directory *reads* scale further via replication of hot dirfrags to non-authoritative ranks.

Snapshots: `SnapRealm` hierarchy — a snap on a directory implicitly covers its subtree; clients attach the realm's `SnapContext` to data writes so OSD-side COW captures file data; metadata pastness is handled by versioned "old_inode" copies in the dentry.

---

## 16. RGW: Object Gateway

`src/rgw/`. An HTTP daemon (Boost.Asio/Beast frontend, `rgw_asio_frontend.cc`, coroutine-per-request on `spawn::spawn`) translating S3/Swift/IAM/STS dialects onto RADOS.

- **Layered request pipeline**: frontend → REST dialect dispatch (`rgw_rest_s3.cc` etc.) → auth engine chain (`rgw_auth*.cc`: v2/v4 signatures, keystone, STS/temp creds, external authenticators) → `RGWOp` subclasses (`rgw_op.cc`, ~10 kLOC of S3 semantics) → **SAL** (Store Abstraction Layer, `rgw_sal.h`) → driver.
- **SAL drivers** (`rgw/driver/`): `rados` (production), `dbstore` (SQLite standalone), `posix`, `d4n` (distributed cache over Redis), plus filters. SAL was a major Reef-era refactor to decouple S3 semantics from RADOS specifics.
- **RADOS layout**: user metadata + per-user bucket list (omap); bucket → *entrypoint* + versioned *bucket instance* metadata; **bucket index** = N shard objects whose omap maps object name → `rgw_bucket_dir_entry`, updated via `cls_rgw` in a **prepare/complete two-phase** dance around the data write (with "dir suggest" lazy repair for orphaned prepares — this is an *eventually-reconciled index over consistent data*). Object data: head object (first ~4 MiB + manifest xattr) + tail objects (immutable, refcounted via `cls_refcount` for server-side copy); multipart assembles a manifest referencing part tails. Object versioning adds olh (object logical head) indirection objects.
- **Background machinery**: garbage collection (deferred tail deletion via `cls_rgw_gc` omap queues), lifecycle (`rgw_lc.cc`, per-bucket shard-parallel), quota caches, usage logging, bucket notifications (`rgw_pubsub*`: kafka/amqp/http endpoints with reliable [cls_2pc_queue] delivery).
- **Multisite** (`rgw_sync.cc`, `driver/rados/rgw_data_sync.cc`, `rgw_coroutine.cc` — a bespoke stackless coroutine framework predating C++20 coros): realm → zonegroups → zones. Metadata ops funnel through the master zone (mdlog); data changes append to per-shard **datalogs** and per-bucket **bilogs**; peer zones run pull-based sync agents (full-sync then incremental log tailing) with per-shard markers and error retry queues. Active-active, last-writer-wins per object version.
- Also here: `rgw-orphan-list`/`rgw-gap-list` offline consistency tooling, S3 Select (`s3select` submodule), `radosgw-admin` (its own subdir).

---

## 17. ceph-mgr and Orchestration

`src/mgr/` (C++ host) + `src/pybind/mgr/` (Python modules). The mgr exists to keep *non-critical, stateful-ish, high-cardinality* work off the monitors:

- Every daemon maintains an `MgrClient` session, streaming perf counters and (from OSDs) PG stats; the mgr aggregates and only digests reach the monitor (`MgrStatMonitor`) — this is why mon DB size stays bounded on large clusters.
- `ActivePyModules` embeds CPython; modules get `get(...)` snapshots of maps/stats, config, and a KV store (`MonKVStore` via `KVMonitor`). Notable modules: **dashboard** (full web UI + REST, Angular frontend in `src/pybind/mgr/dashboard/frontend`), **prometheus** exporter, **balancer** (upmap optimizer), **pg_autoscaler**, **devicehealth** (SMART → failure prediction), **volumes** (CephFS subvolume API consumed by CSI), **rbd_support** (mirror scheduling, trash purge), **telemetry**, **rook**, and **orchestrator**/**cephadm**.
- **cephadm** (`src/cephadm/cephadm.py` zipapp + `pybind/mgr/cephadm`): the built-in deployment plane. The mgr module SSHes to hosts, runs the cephadm binary to manage **containerized** daemons (podman/docker + systemd units), reconciles declarative **service specs** (`ServiceSpec` YAML: placement, counts, networks) against inventory — a small Kubernetes-shaped control loop without Kubernetes. Also handles upgrades (staggered, `ceph orch upgrade`), cert management, node-proxy (Redfish) hardware monitoring, and the NVMe-oF/iSCSI/NFS(ganesha)/SMB/ingress(haproxy+keepalived) gateway stacks.
- Failover: mgrs are active/standby (`MgrMonitor` arbitrates); modules must tolerate restart-anywhere (state in mon KV or RADOS).

---

## 18. Crimson and SeaStore: The Next-Generation OSD

`src/crimson/` — a ground-up OSD rewrite targeting NVMe-era hardware, where the classic OSD's locks, queues, and thread handoffs dominate latency.

### 18.1 Execution model

Built on **Seastar** (bundled submodule): shard-per-core, one pinned reactor thread per CPU, **no locks, no cross-core shared memory by default** — cross-core communication is explicit message passing (`smp::submit_to`). All I/O and logic are futures/continuations; blocking is structurally impossible. Every PG is owned by exactly one shard; a `MOSDOp` is routed to its PG's core and runs to completion there. The peering logic is *reused* from the classic OSD (`PeeringState` was deliberately extracted to be host-agnostic), while everything around it (`crimson/osd/`, `crimson/net/` — a native msgr2 implementation on Seastar sockets) is new. Op ordering uses "pipeline" stage abstractions (`osd_operation.h`) instead of PG-wide mutexes — ops declare the stages (obc lock, wait-for-active, etc.) they pass through, giving fine-grained, deterministic interleaving.

Interim compatibility: **AlienStore** runs classic BlueStore in a foreign thread pool bridged to the reactor ("alien" threads), so crimson-osd can run today on BlueStore while SeaStore matures.

### 18.2 SeaStore

`crimson/os/seastore/` — a from-scratch, log-structured, copy-on-write object store designed for the reactor model (per-shard instances, no global state) and for NVMe/ZNS:

- **TransactionManager** provides CoW transactions over **CachedExtent**s (fixed-size typed extents forming B-trees). All mutation is: open transaction → obtain extents (from `Cache`) → mutate copies → commit appends a delta/record to the **Journal**. Conflict detection is optimistic — transactions that raced a committed write are invalidated and restarted (`transaction_interruptor`).
- **LBA manager** (`lba/`, btree): logical→physical indirection tree, itself stored in extents (self-referential; bootstrapped from a root block). Because *everything* moves (log-structured), a **backref manager** maintains reverse mappings so the cleaner can relocate live extents.
- **Journal** flavors: segmented (for SMR/ZNS-friendly sequential segments) and circular-bounded (for the `random_block_manager` path on conventional SSDs, where cold data can be written in place — "RBM").
- **ExtentPlacementManager** + `async_cleaner`: generational/hot-cold segregation of extents into segments; the cleaner does GC (segment compaction) with usage tracked via backrefs — the classic LFS cleaning problem, handled with hot/cold hints from extent types (metadata vs onode-data vs cold data). `extent_pinboard` keeps hot extents pinned in cache.
- **Onode tree**: staged FLTree (`onode_manager/staged-fltree/`) — a cache-friendly, dense B-tree keyed by ghobject with staged (prefix-factored) key comparison; **OMap manager** implements omap as another extent B-tree.

Status in this tree: functional and under heavy development (crimson is testable via `vstart.sh --crimson`), not yet the default production OSD.

---

## 19. Cross-Cutting Infrastructure

- **Serialization/compat**: `encode/decode` free functions + `DENC` fast-path (`src/include/denc.h` — bounds precomputation, contiguous decode). Versioned structs use `ENCODE_START(v, compat_v, bl)`/`DECODE_FINISH` wrappers embedding version + compat floor + length (skippable unknown tails ⇒ rolling upgrades work). Feature-conditional encoding keys off the peer's advertised `ceph_features.h` bit set (64-bit, heavily recycled). `ceph-dencoder` + `ceph-object-corpus` regression-test wire compat across releases — treat any encoder change without a corpus update as a red flag in review.
- **Config**: options defined declaratively in `src/common/options/*.yaml.in` (global/osd/mds/rgw…), code-generated; runtime `md_config_t` supports typed values, live `observers`, and level/flags metadata. Precedence: compiled default < mon `ConfigMonitor` store (`ceph config set`, with mask syntax like `osd/class:ssd`) < local `ceph.conf` < env < CLI override. Most options apply live via observer callbacks.
- **Observability**: `dout()` logging with per-subsystem levels *and* an always-on in-memory ring of recent high-verbosity entries dumped on crash (`log/`); **admin socket** per daemon (`ceph daemon <name> ...` — perf dump, dump_ops_in_flight, config get/set, state machine dumps); `PerfCounters` (lock-free counters/averages/histograms) exported via mgr→prometheus; distributed tracing via Jaeger/OpenTelemetry (`jaegertracing/`, `osd_tracer.cc`) and blkin/LTTng at the block layer; `objectstore-tool`/`ceph-bluestore-tool`/`ceph-monstore-tool` for offline surgery.
- **Failure containment primitives**: `Throttle`/`BackoffThrottle` (`common/Throttle.h`) at every ingress (messenger dispatch bytes, objecter in-flight, bluestore deferred bytes); OSD **backoff** messages tell clients to stop sending for a PG/object range during peering instead of queueing unboundedly; heartbeat grace adaptivity; `osd_op_thread_timeout`/suicide timeouts convert livelocks into crashes (crash-and-recover philosophy — daemons are designed to die loudly and restart cheaply, with `ceph-crash` shipping reports).
- **Testing**: `src/test/` unit tests (gtest); `qa/suites/` teuthology matrices (the yaml-fragment convolution under `qa/suites/{rados,rbd,fs,rgw,upgrade}` *is* the release qualification); `vstart.sh` spins a full dev cluster from a build tree; `ceph_test_rados` (the "RadosModel" random-op consistency checker) and thrashers (kill/revive OSDs mid-workload) are the workhorses that make peering changes shippable.

---

## 20. Consistency Model and Failure Handling — Summary

The invariant chain to keep in your head when reasoning about any Ceph change:

1. A write is acked ⇔ every acting-set member has committed it durably (BlueStore KV commit).
2. The monitor's Paxos-serialized OSDMap defines PG intervals; `up_thru` prunes never-active intervals.
3. Peering contacts a survivor of every maybe-active interval and adopts the best log (`find_best_info`) — so an acked write can never be silently lost while any of its holders survives; if none survive, the PG blocks (`incomplete`/`down`) rather than lie.
4. PG log dup entries make client resends idempotent across primary failover.
5. `min_size` prevents going active with too few replicas (split-brain guard); blocklisting fences dead lock-holders (RBD/MDS).
6. Scrub + BlueStore checksums close the loop against silent corruption.

Everything else — CRUSH, upmap, mClock, BlueFS, caps, multisite logs — is performance and operability engineering layered around that chain.

---

*Generated 2026-07-18 from source analysis of the Ceph "Tentacle" (v20.0.0) development tree.*
