# Storage Types: What They Are, Why They Exist, and What They Solve

A conceptual overview of the major kinds of storage in a computer system.
It starts with the physical **memory hierarchy** (why there are several
tiers of storage at all), then covers the three **access models** that
software uses to talk to persistent storage (block, file, object), and
finally the **deployment topologies** that connect storage to machines
(DAS, NAS, SAN, cloud). Each section explains the problem that the
storage type was invented to solve and the trade-offs it accepts.

The document is vendor-neutral. It does not describe any specific product.

---

## 1. Why there is more than one kind of storage

Every storage technology sits somewhere on three axes that pull against
each other:

- **Speed**: how quickly a byte can be read or written (latency) and how
  many bytes per second can move (throughput).
- **Capacity and cost**: how many bytes you get per unit of money and per
  unit of physical space.
- **Durability**: whether the data survives a power loss, a crash, a disk
  failure, or a data-centre fire.

No single technology wins on all three. Fast storage is expensive and small.
Cheap storage is slow. Durable storage requires redundancy, which costs
capacity and write speed. Because of this, real systems stack several kinds
of storage into a **hierarchy** and move data between tiers so that the
data being used right now lives in the fast tier and everything else lives
in the cheap tier.

```
            fast, small, expensive, volatile
                    ▲
   CPU registers    │   a few hundred bytes, accessed every cycle
   L1 / L2 / L3     │   KiB to tens of MiB of SRAM on the CPU die
   Main memory      │   GiB of DRAM; lost on power off
   ─────────────────┼───────────────────────────  volatile / persistent line
   SSD (flash)      │   hundreds of GiB to TiB; microseconds
   HDD (magnetic)   │   TiB; milliseconds (seek + rotation)
   Tape / archive   │   PiB; seconds to minutes to reach the first byte
                    ▼
            slow, huge, cheap, persistent
```

The hierarchy works because of **locality**: programs tend to reuse the
same data (temporal locality) and data near what they just used (spatial
locality). Caches and page caches exploit this so that most accesses hit a
fast tier even though most bytes live in a slow one.

---

## 2. Volatile vs persistent storage

### 2.1 Volatile storage (registers, caches, RAM)

**What it is.** Memory that holds data only while powered. Registers and
SRAM caches live on the processor; DRAM main memory sits on the memory bus.

**Why it exists.** The CPU executes an instruction in a fraction of a
nanosecond. Persistent media respond in microseconds (flash) or
milliseconds (spinning disk). Without a fast working area the CPU would
spend almost all of its time waiting. Volatile memory is the working area.

**Problem it solves.** Bridging the speed gap between the processor and
everything else. It is byte-addressable and random-access, so a program can
read or write any location directly.

**Trade-off.** Contents vanish on power loss or crash. Anything that must
survive has to be written to persistent storage, which is why databases,
filesystems and applications distinguish "in memory" from "committed".

### 2.2 Persistent storage (SSD, HDD, tape, optical)

**What it is.** Media that retain data without power.

- **Hard disk drives (HDD)** store bits magnetically on spinning platters.
  A read head must physically move to the right track (seek) and wait for
  the platter to rotate to the right sector. Sequential access is fast;
  random access is slow.
- **Solid-state drives (SSD)** store bits as charge in NAND flash cells.
  There are no moving parts, so random reads are fast. Flash cannot be
  overwritten in place: a block must be erased before it is rewritten, and
  each cell tolerates a limited number of erase cycles. The drive's
  firmware (the flash translation layer) hides this by remapping writes
  and wear-levelling across cells.
- **Magnetic tape** stores bits on a long ribbon that is read
  sequentially. Very cheap per byte and long-lived, but reaching a given
  byte can take a long time. Used for backup and archive.
- **Optical media** (CD, DVD, Blu-ray) are read with a laser. Mostly used
  for distribution and long-term write-once archives.

**Why it exists.** Data has to outlive the process that created it, the
machine's uptime, and often the machine itself.

**Problem it solves.** Durability and capacity at a price that makes
storing large amounts of data feasible.

**Trade-off.** Persistent media are addressed in fixed-size **sectors** or
**pages** (commonly 512 bytes or 4 KiB), not bytes. Software must therefore
read and write whole blocks, and every higher-level storage abstraction is
built on top of that block interface.

---

## 3. The three access models: block, file, object

Persistent media expose a raw array of blocks. Almost no application wants
to program against that directly. Three abstractions have emerged, each
answering the question "how does software name and reach a piece of data?"
differently.

```
 Application view                 Unit of access       Typical consumer
 ─────────────────────────────    ──────────────────   ─────────────────────
 Block:  disk = array of blocks   fixed-size block     filesystems, databases,
         addressed by number      (e.g. 4 KiB)         virtual machine disks

 File:   tree of directories      byte range within    people, shell tools,
         and named files          a named file         most applications

 Object: flat namespace of        whole object,        web / cloud apps,
         key -> blob + metadata   by key               backups, media, data lakes
```

### 3.1 Block storage

**What it is.** The storage presents itself as a linear sequence of
fixed-size blocks, each identified by a number (a logical block address).
The consumer reads or writes any block by number. The storage layer knows
nothing about what the blocks contain.

**Why it exists.** It is the natural interface of the underlying medium.
Disks, SSDs and volumes carved out of them all speak in blocks. Anything
that wants full control over layout, such as a filesystem or a database
engine, needs this level.

**Problems it solves.**

- **Low latency and high control.** No intermediate layer interprets the
  data. A database can lay out its pages exactly where it wants and issue
  I/O in the order it needs for consistency.
- **Bootable and mountable volumes.** An operating system installs onto a
  block device and puts its own filesystem on it. Virtual machines are
  given block devices that look exactly like a physical disk.
- **Snapshots and cloning at the volume level.** Because the block layer
  is simple, it is easy to implement copy-on-write snapshots of an entire
  volume.

**Trade-offs.**

- A block device is normally attached to **one** host at a time. Two hosts
  writing to the same block device without a coordinating filesystem
  corrupt each other's data.
- There is no notion of a "file" or "who owns this". All structure has to
  be supplied by whatever sits on top.
- Capacity is provisioned as a fixed-size volume; growing it usually
  requires resizing the filesystem above it as well.

### 3.2 File storage

**What it is.** Data is organised into **files**, each with a name and a
byte-addressable body, arranged in a **hierarchy of directories**. A
filesystem maps this structure onto blocks and keeps metadata such as
size, timestamps, ownership and permissions.

**Why it exists.** Humans and most programs think in terms of named
documents in folders, not block numbers. A filesystem turns raw blocks into
something people can navigate and programs can share.

**Problems it solves.**

- **Naming and organisation.** A path like `/home/alice/report.txt` is
  meaningful. Block 8,204,113 is not.
- **Sharing with access control.** Multiple users and processes can read
  and write files on the same volume, with the filesystem enforcing
  permissions and coordinating concurrent access.
- **Variable-size data.** A file can be any length and can grow; the
  filesystem handles allocating and freeing blocks behind it.
- **Network sharing.** Network file protocols (such as NFS and SMB) let
  many machines mount the same directory tree, which is how home
  directories and shared project folders are usually delivered.

**Trade-offs.**

- Directory hierarchies and metadata add overhead. Listing a directory with
  millions of entries, or keeping metadata consistent across a cluster, is
  a hard problem and becomes the bottleneck at large scale.
- Strong POSIX semantics (a write is immediately visible to every reader,
  files can be locked, byte ranges can be rewritten in place) are expensive
  to guarantee across a network and across many servers.
- The interface was designed for a single machine and stretches, rather
  than scales, to the internet.

### 3.3 Object storage

**What it is.** Data is stored as **objects**, each consisting of the data
itself, a set of metadata, and a unique **key**. Objects live in a flat
namespace (often grouped into buckets or containers). There is no
directory tree and no in-place modification: you create, read, replace, or
delete a whole object, usually over an HTTP API.

**Why it exists.** Web-scale applications needed to store enormous numbers
of items (photos, videos, backups, log archives, machine-learning
datasets) across many servers and many data centres, accessed from
anywhere over the network. Filesystem semantics were more than those
workloads needed and got in the way of scaling.

**Problems it solves.**

- **Massive horizontal scale.** A flat key space is easy to partition
  across thousands of nodes. There is no directory metadata to keep
  consistent.
- **Durability by design.** Each object is typically replicated or
  erasure-coded across machines and locations, so the loss of a disk, a
  server or a site does not lose data.
- **Rich metadata.** Arbitrary key-value metadata travels with each object,
  which supports indexing, lifecycle rules and content tagging without a
  separate database.
- **Simple, universal access.** An HTTP GET or PUT works from any language
  on any network, with authentication built into the protocol.
- **Cost.** By dropping in-place writes, locking and hierarchical metadata,
  object stores run on cheap commodity hardware and are usually the
  cheapest persistent tier that is still online.

**Trade-offs.**

- **No partial updates.** Changing one byte means rewriting the whole
  object. Workloads that modify data in place (databases, most
  applications expecting a filesystem) do not fit.
- **Higher latency per request.** Each access is a network round trip
  through an HTTP stack, so object storage suits large reads and writes,
  not many tiny ones.
- **Weaker consistency historically.** Some object stores have offered
  eventual consistency for listings or overwrites, which applications must
  be designed around. Many modern systems now offer strong consistency,
  but the interface does not promise POSIX behaviour.
- It cannot be booted from or mounted as a normal disk without an
  adapter layer.

### 3.4 Choosing between them

```
 Need                                          Fit
 ────────────────────────────────────────────  ─────────────────
 Database data files, VM disks, boot volumes   Block
 Shared home directories, POSIX applications,  File
   tools that expect paths and byte writes
 Backups, media, logs, datasets, static web    Object
   content, anything huge and write-once

 Many real systems layer them: a filesystem on a block volume,
 a block volume striped across objects, or a file gateway in
 front of an object store.
```

---

## 4. Deployment topologies: where the storage lives

The access model says *how* software addresses data. The topology says
*where the storage hardware sits relative to the computer using it*.

```
 DAS         host ──── disk                       one host, direct cable / bus
 NAS         hosts ── LAN ── file server ── disks  file protocol over the network
 SAN         hosts ── dedicated storage network ── disk arrays   block protocol
 Cloud       hosts ── internet / API ── provider-managed pools   any model
```

### 4.1 Direct-attached storage (DAS)

**What it is.** Disks connected directly to a single computer over an
internal bus or a short cable (for example SATA, SAS, NVMe, USB).

**Why it exists.** It is the simplest and fastest way to attach storage.
There is no network in the path.

**Problem it solves.** Lowest latency and lowest cost per attached disk.
Fine for a laptop, a workstation, or a server whose data does not need to
be shared.

**Trade-off.** The storage is captive to one host. If the host is down, the
data is unreachable. Capacity cannot be pooled across machines, so some
servers run out of space while others sit half empty.

### 4.2 Network-attached storage (NAS)

**What it is.** A dedicated server (or appliance) that owns disks and
exports them over the ordinary local network as **file shares** using a
file protocol such as NFS or SMB.

**Why it exists.** Organisations needed many users and machines to see the
same files, with central backup and central access control.

**Problem it solves.** Shared, centrally managed file storage. Any machine
on the network can mount the share and see the same directory tree.

**Trade-off.** Every access is a network round trip, and the NAS server is
a single point of contention and, unless clustered, a single point of
failure. It provides files, not raw block devices, so it is not the right
fit for databases or virtual machine disks that want block semantics.

### 4.3 Storage area network (SAN)

**What it is.** A dedicated, high-speed network (traditionally Fibre
Channel, or iSCSI / NVMe over Fabrics over Ethernet) that connects servers
to shared disk arrays and presents them as **block devices**.

**Why it exists.** Data centres wanted the performance and control of
block storage together with the pooling and central management of a
shared array. Servers needed to fail over: if one host dies, another host
must be able to attach the same volume and carry on.

**Problem it solves.**

- **Pooled block storage.** Capacity is allocated to hosts as logical
  volumes from a shared pool rather than being locked into each chassis.
- **High availability.** A volume can be re-attached to a different host,
  which is the foundation of clustered databases and virtual machine
  migration.
- **Consistent performance.** Storage traffic runs on its own network and
  does not compete with application traffic.

**Trade-off.** Expensive hardware and specialist skills. The volumes are
still block devices, so sharing one between hosts simultaneously requires
a cluster-aware filesystem on top.

### 4.4 Cloud and software-defined storage

**What it is.** Storage delivered as a service by a provider, or built in
software on top of a fleet of ordinary servers with local disks. All three
access models are offered: block volumes attachable to virtual machines,
managed file shares, and object storage over HTTP.

**Why it exists.** Buying, housing and operating storage arrays is capital
intensive and slow to change. Software-defined storage lets a pool of
commodity machines behave like an array, and cloud providers rent that
pool out by the gigabyte-month.

**Problem it solves.**

- **Elasticity.** Capacity can grow or shrink on demand instead of being
  bought years in advance.
- **Durability and geography.** Providers replicate across data centres,
  which few organisations could do on their own.
- **Operational simplicity.** Failed disks, firmware, and capacity
  planning become the provider's problem.

**Trade-off.** Latency to remote storage, ongoing cost that never
amortises, dependence on a provider's availability and pricing, and the
need to think about data egress and residency. Software-defined systems
run on premises avoid the vendor dependence but take on the operational
work.

---

## 5. Cross-cutting techniques that every storage type uses

The problems below appear regardless of tier or access model, and the same
families of solutions recur.

- **Redundancy against failure.** Disks fail. RAID (within one machine),
  replication (copies on several machines) and erasure coding (data plus
  parity fragments spread across machines) all trade extra capacity for
  the ability to survive lost hardware. Replication is simple and fast to
  read; erasure coding uses far less space but costs CPU and network to
  rebuild.
- **Caching and tiering.** Keep hot data on the fast, expensive tier and
  cold data on the cheap one, and move it automatically as access patterns
  change. The CPU cache, the operating system page cache, an SSD cache in
  front of HDDs, and a lifecycle policy that moves old objects to archive
  storage are all the same idea at different scales.
- **Write-ahead logging / journaling.** To make an update atomic and
  durable despite crashes mid-write, record the intent in a sequential log
  first, then apply it. Filesystems, databases and object stores all do a
  version of this.
- **Snapshots and copy-on-write.** Instead of overwriting data in place,
  write the new version elsewhere and update pointers. Old versions can be
  kept cheaply as snapshots, and a partial write can never corrupt the
  previous good state.
- **Checksums.** Store a hash alongside data so that silent corruption on
  the medium (bit rot) is detected on read rather than passed to the
  application.
- **Compression and deduplication.** Reduce the bytes actually stored by
  compressing them or by storing identical blocks only once. Common in
  backup systems and in storage arrays that hold many similar virtual
  machine images.

---

## 6. Summary

| Type            | Unit of access      | Solves                                   | Main limitation                          |
|-----------------|---------------------|------------------------------------------|------------------------------------------|
| Registers/cache | word / cache line   | CPU speed gap                            | tiny, volatile                           |
| RAM             | byte                | fast working set                         | volatile, costly per GiB                 |
| SSD             | page / block        | fast persistent random access            | limited erase cycles, cost per TiB       |
| HDD             | sector              | cheap persistent capacity                | slow random access, moving parts         |
| Tape            | sequential stream   | cheapest long-term archive               | very high first-byte latency             |
| Block storage   | numbered block      | raw, low-latency volumes for OS/DB/VM    | single host, no structure                |
| File storage    | path + byte range   | human naming, sharing, POSIX programs    | metadata scaling, strict semantics cost  |
| Object storage  | key -> whole object | internet-scale durable blobs             | no partial writes, per-request latency   |
| DAS             | local bus           | simplest, fastest attachment             | captive to one host                      |
| NAS             | file share over LAN | central shared files                     | contention, file semantics only          |
| SAN             | block over fabric   | pooled, highly available block volumes   | cost and complexity                      |
| Cloud / SDS     | any, via API        | elasticity, geographic durability        | latency, ongoing cost, provider lock-in  |

Storage types exist because speed, capacity, cost and durability cannot all
be maximised at once. Each type is a deliberate position on those axes, and
a well-designed system combines several of them so that every piece of
data sits on the tier and behind the interface that fits how it is used.
