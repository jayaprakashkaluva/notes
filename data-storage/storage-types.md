# Storage Types: What They Are, Why They Exist, and What They Solve

A conceptual overview of the major kinds of storage in a computer system.
It starts with the physical **memory hierarchy** (why there are several
tiers of storage at all), then covers the three **access models** that
software uses to talk to persistent storage (block, file, object), and
finally the **deployment topologies** that connect storage to machines
(DAS, NAS, SAN, cloud). Each section explains the problem that the
storage type was invented to solve and the trade-offs it accepts.

Section 3 is the core. It builds the three access models up from the bare
disk, shows what breaks if each one is missing, and explains **POSIX**,
the file-API contract that separates file storage from the other two.

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

This is the heart of the document. The physical tiers in section 2 answer
"where do the bytes physically live". The **access model** answers a
different question: **how does a program name a piece of data and get at
it?** Three answers have survived decades of use because each one is the
simplest possible contract for a distinct kind of consumer:

```
 Block   "Give me a disk."
         A numbered array of fixed-size blocks. No names, no structure.

 File    "Give me a shared tree of named files that behaves like Unix."
         Paths, directories, permissions, byte-level edits, POSIX rules.

 Object  "Give me a place to put a blob under a key and get it back,
          from anywhere, forever, at any scale."
         Flat key space, whole-object reads and writes, HTTP.
```

Seen from the application, the three differ in what a "thing" is and how
it is addressed:

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

They are not three competing products. They are three **layers of
abstraction**, and each one exists because the layer below it is too
primitive for a whole class of users while the layer above it is too
expensive or too restrictive for another class. To see why, start with what
the hardware actually gives you and remove the abstractions one at a time.

### 3.0 Starting point: what a disk really offers

Strip away every piece of software and a disk (HDD or SSD) is a device that
understands exactly two commands: *read sector N* and *write sector N*,
where every sector is the same size (512 bytes or 4 KiB). There are no
names, no notion of "this range belongs to program X", no sizes other than
"the whole device", no permissions, and no way to write a single byte
without rewriting its whole sector.

```
 sector:   0     1     2     3     4     5     6   ...   N-1
         ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬───┬─────┐
         │     │     │     │     │     │     │     │   │     │
         └─────┴─────┴─────┴─────┴─────┴─────┴─────┴───┴─────┘
                                 ▲
             every read/write is "sector number + whole sector of bytes"
```

Imagine writing a text editor against this interface. You would have to
decide which sectors hold your document, remember that decision somewhere
(in another sector), make sure no other program picks the same sectors,
handle the document growing past the sectors you reserved, and rewrite a
whole sector to change one character. Every program on the machine would
have to solve exactly the same problems, and any two programs that made
different choices could not read each other's data and could silently
overwrite each other. Early computers really did work like this, and it is
the reason the abstractions below were invented.

### 3.1 Block storage

**What it is.** Block storage is the disk interface from section 3.0 with
two things added: a clean, device-independent API, and **virtualisation**
of the "disk" itself.

- The API: the consumer sees a **block device** of some size, addresses it
  with a **logical block address** (LBA) from 0 to N-1, and reads or writes
  one or more whole blocks at a time. Nothing else. The block device does
  not know or care what the bytes mean.
- The virtualisation: the block device the consumer sees no longer has to
  be one physical drive. It can be a partition of a drive, a stripe across
  several drives (RAID), a logical volume that can be resized, a volume
  exported over a network (a SAN LUN, an iSCSI target), or a virtual disk
  attached to a virtual machine that is really backed by objects in a
  cluster. To the consumer it is still just "a disk".

```
 consumer (filesystem / database / VM)
   │  read(lba, count)  /  write(lba, count, data)
   ▼
 block device  ─────  "a disk", size S, block size B
   │
   ▼  may be:
   physical drive │ partition │ RAID set │ LVM volume │ SAN LUN │ cloud volume
```

**Why it exists.**

1. **It is the native language of the medium.** Anything that wants full
   control over layout (a filesystem, a database engine, a swap area) needs
   to speak in blocks, and a stable block API means they can do so without
   knowing which drive model or controller is underneath.
2. **Operating systems boot from it.** A machine needs *something* to load
   before there is a filesystem to read. That something is a block device.
3. **Virtual machines need a disk.** A guest operating system expects a
   disk it can partition, format and boot. The only thing that satisfies
   that expectation is a block device.
4. **Some workloads want no help.** A database already has its own page
   cache, its own write-ahead log, its own free-space management and its
   own crash-recovery. A filesystem in between would duplicate all of that
   and get in the way (double caching, unpredictable write ordering). Such
   workloads want raw blocks and precise control over when data hits the
   medium.
5. **Volume-level operations.** Because the block layer has no idea what
   the data means, it can snapshot, clone, replicate, encrypt, thin-provision
   or migrate an entire volume with a single, simple mechanism, and it works
   identically for every filesystem or database placed on top.

**What happens if we do not have it.** Every filesystem and database ships
its own driver for every disk controller, so nothing is portable. There is
no way to hand a "disk" to a virtual machine, so virtualisation does not
work. A disk cannot be moved between hosts, resized, or snapshotted as a
unit. Databases must live inside a filesystem whose caching and ordering
rules they cannot control, so durability guarantees become harder to
reason about.

**What it deliberately does not do.** Block storage has no names, no
sizes other than "the whole device", no permissions, no notion of
ownership, and no rule for how two hosts can share it. A block volume is
normally attached to exactly one host at a time; if two hosts write to the
same volume without a cluster-aware filesystem coordinating them, each one
overwrites the other's blocks and both copies are corrupted. These gaps are
not flaws. They are precisely the things the file layer is for.

### 3.2 File storage

**What it is.** A **filesystem** is a program (usually part of the
operating system kernel) that takes a block device and turns it into a
**tree of named files**. Each file has a variable length, a set of
metadata (owner, permissions, timestamps, size) and a body that can be
read and written at any byte offset. Directories hold names that point to
files or other directories. Internally the filesystem keeps its own
bookkeeping on the same block device: which blocks are free, which blocks
belong to which file and in which order, and where each directory lives.

```
                     block device (from 3.1)
 ┌──────────────────────────────────────────────────────────────────┐
 │ superblock │ free-block map │ inode table │ data blocks ...      │
 └──────────────────────────────────────────────────────────────────┘
        │              │              │             │
        │              │              │             └─ file contents, directory entries
        │              │              └─ one record per file: size, owner, perms,
        │              │                 timestamps, list of data blocks
        │              └─ which blocks are in use
        └─ where everything else is, block size, filesystem type

 what the user sees instead:
   /
   ├── home/
   │   └── alice/
   │       ├── report.txt      (1,532 bytes, rw-r--r--, alice)
   │       └── photos/
   └── var/log/syslog          (grows continuously)
```

**Why it exists.** It solves every problem listed in section 3.0 once, in
one place, for all programs:

1. **Naming.** `/home/alice/report.txt` is meaningful to a person and
   stable across reboots. Block 8,204,113 is neither.
2. **Space management.** The filesystem tracks which blocks are free and
   hands them out as files grow, reclaims them on delete, and tries to keep
   a file's blocks near each other so sequential reads stay fast. No
   program has to think about it.
3. **Variable size and byte-level access.** A program can append a line to
   a log or overwrite ten bytes in the middle of a file. The filesystem
   translates that into whole-block reads and writes to the block device.
4. **Sharing and protection.** Many users and processes use one volume.
   The filesystem enforces who may read or write what, and it arbitrates
   concurrent access so two processes appending to the same file do not
   corrupt it.
5. **Interoperability.** Because every program uses the same file API
   (section 3.3), a file written by one program can be read by any other.
   A shell, an editor, a compiler and a backup tool all agree on what a
   file is.
6. **Crash safety.** Journaling filesystems make sure that the bookkeeping
   is never left half-updated by a crash, so a power cut does not turn the
   whole volume into garbage.
7. **Network sharing.** File protocols such as NFS and SMB carry the same
   file API across a network, so hundreds of machines can mount one
   directory tree. This is how shared home directories, build farms and
   most enterprise shared drives work.

**What happens if we do not have it.** You are back in section 3.0. Every
application invents its own on-disk layout; two applications cannot share
a disk safely; there is no `ls`, no `cp`, no backup tool that works on
everything; there are no permissions, so any program can read any other
program's data; a crash mid-write can destroy unrelated data because there
is no shared journal; and "how much space is left" is a question nobody
can answer. The operating system itself cannot be installed, because an OS
is thousands of named files.

**What it costs.** All of that convenience is bought with **metadata** and
**semantic guarantees**, and both become expensive at scale.

- Every file needs an inode; every lookup walks a directory path component
  by component; every create, rename or delete updates directory entries,
  free maps and journals. On one machine that is cheap. Spread across a
  cluster of servers, every one of those updates needs coordination.
- The API promises that a write is immediately visible to every reader,
  that a byte range can be changed in place, that a file can be locked,
  that a rename is atomic. Keeping those promises when the readers and
  writers are on different machines in different buildings requires
  distributed locks and cache invalidation, which limits how far a
  filesystem can scale and how fast it can go.
- A hierarchical namespace does not split cleanly. If one directory holds
  a hundred million entries, or one hot directory takes all the traffic,
  there is no natural way to spread it over many servers.

Those promises are what people mean by **POSIX semantics**, which deserves
its own section.

### 3.3 POSIX: the contract that defines "a file"

**What POSIX is.** POSIX (Portable Operating System Interface) is a family
of IEEE standards, first published in 1988 and derived from Unix, that
defines the programming interface an operating system offers to programs:
the system calls, their arguments, their error codes and, crucially, their
**semantics** (what must be true after the call returns). Linux, the BSDs,
macOS and most other Unix-like systems implement it; Windows has partial
compatibility layers. Its purpose was portability: a program written
against POSIX should compile and behave the same on any compliant system.

For storage, the relevant part of POSIX is the **file API**. The core calls
are few and every programmer has used them, often without knowing they are
a standard:

```
 open(path, flags)        → file descriptor      name a file, get a handle
 read(fd, buf, n)         → bytes                read at the current offset
 write(fd, buf, n)        → bytes                write at the current offset
 lseek(fd, off, whence)                          move the offset anywhere
 close(fd)
 stat(path)               → size, owner, mode, timestamps
 mkdir / rmdir / readdir                         directory operations
 rename(old, new)                                atomic move / replace
 unlink(path)                                    delete a name
 link(path, newpath)                             hard link: a second name
 chmod / chown                                   permissions and ownership
 fsync(fd)                                       force data to durable media
 fcntl / flock                                   advisory file locking
 mmap(fd, ...)                                   map a file into memory
```

**Why it matters for storage.** POSIX is the reason "file storage" is a
single well-defined thing rather than a dozen incompatible ones. When a
filesystem is described as *POSIX-compliant* it means that all of the
guarantees below hold, and therefore any program written for Unix will
work on it unchanged. The guarantees are the important part, and they are
also the expensive part:

- **Byte-granular, in-place writes.** `write()` at any offset changes just
  those bytes; the rest of the file is untouched. The filesystem must do
  read-modify-write on the underlying blocks.
- **Read-after-write consistency.** Once `write()` returns, any `read()` by
  any process on the system sees the new bytes. Across a network this
  means caches on other machines must be invalidated before the write is
  acknowledged.
- **A shared, coherent file offset and metadata.** `stat()` must report the
  size that includes the write that just happened, from any process.
- **Atomic `rename()`.** Replacing a file by writing a temporary and
  renaming it over the original must be all-or-nothing. Every "save file
  safely" routine and most package managers rely on it.
- **Hard links and unlink-while-open.** A file can have several names, and
  a file that is deleted while a process still has it open stays alive
  until the last handle closes.
- **Locking.** Processes can coordinate through `flock()`/`fcntl()` locks
  on files or byte ranges.
- **Durability on request.** `fsync()` must not return until the data is
  on stable storage. This is how databases and mail servers implement
  "committed".
- **Permissions and ownership.** Every operation is checked against the
  owner, group and mode bits.

**Why these are hard in a distributed system.** On one machine all of this
is enforced by one kernel holding one set of in-memory structures, so it
is nearly free. Spread the same file across ten servers with a thousand
clients and every guarantee becomes a coordination problem:

```
 client A (host 1)                 client B (host 2)
   write(fd, "hello", off=0)          read(fd, buf, off=0)
        │                                   │
        ▼                                   ▼
   POSIX says B must see "hello" if its read starts after A's write returned.
   Therefore A's write cannot return until every cached copy of that range
   on every other host has been invalidated or updated. That is a network
   round trip to a lock or lease manager on every write, or a design that
   forbids caching. Either choice caps throughput.
```

That is why distributed and network filesystems fall into three camps:
those that implement full POSIX with distributed locking and accept the
cost; those that relax specific guarantees (for example, close-to-open
consistency, where a write is only guaranteed visible after the writer
closes and the reader opens) and call themselves *near-POSIX*; and those
that give up the file model entirely, which is where object storage comes
from.

**How POSIX relates to the other two models.** Block storage sits *below*
POSIX: it provides the raw blocks that a POSIX filesystem is built on and
knows nothing about files. Object storage sits *beside* it and
deliberately drops almost every guarantee above in exchange for scale.

### 3.4 Object storage

**What it is.** An object store holds **objects**. Each object is an
opaque blob of bytes plus a set of user-defined **metadata** (key-value
pairs) plus a unique **key** (its name). Keys live in a **flat namespace**,
usually partitioned into buckets. The API is tiny and usually spoken over
HTTP:

```
 PUT    /bucket/key        store a whole object (creates or replaces)
 GET    /bucket/key        fetch a whole object (or a byte range of it)
 DELETE /bucket/key        remove it
 HEAD   /bucket/key        fetch its metadata only
 LIST   /bucket?prefix=    enumerate keys that start with a prefix
```

There is no `open`, no file offset, no `lseek`, no `write` at an offset,
no `rename`, no directory, no lock, no `fsync`. An object is created whole
and, if it changes, replaced whole. Keys may contain slashes, and tools
will *display* `photos/2024/beach.jpg` as if it lived in folders, but the
store has no folders: it is one key with slashes in it, and "listing a
folder" is a prefix query.

```
 bucket "media"
   key                          metadata                   data
   ──────────────────────────   ────────────────────────   ─────────────
   photos/2024/beach.jpg        content-type=image/jpeg    <2.3 MiB blob>
                                owner=alice
   photos/2024/city.jpg         content-type=image/jpeg    <1.8 MiB blob>
   backups/db-2024-06-01.tar    retention=90d              <40 GiB blob>
   logs/web/2024-06-01.gz       …                          <300 MiB blob>

 flat: no directory objects exist; the prefix "photos/2024/" is just a filter
```

**Why it exists.** Object storage was invented when the web made it
necessary to store billions of items, spread over thousands of servers in
several data centres, written once and read from anywhere. The file model
was tried first, and every one of its strengths turned into a limit:

1. **The hierarchical namespace does not partition.** Directory trees have
   hot spots and shared parents. A flat key space can be hashed, and each
   key sent to whichever set of servers owns that hash, with no server
   needing to know about any other key. That is what makes "add more
   machines to get more capacity and throughput" work indefinitely.
2. **In-place writes and locks require coordination.** Section 3.3 showed
   that byte-level writes with read-after-write visibility need distributed
   locking. If an object can only ever be replaced whole, there is nothing
   to lock: each PUT is an independent, self-contained event that can be
   accepted by any node and reconciled later.
3. **Whole objects are easy to make durable.** A whole, immutable blob can
   be split into fragments, erasure-coded and spread across machines and
   sites in one pass at write time. Doing that for a file whose byte 500
   might be rewritten a millisecond later is far harder. This is why object
   stores can promise very high durability cheaply.
4. **Metadata should travel with the data.** A photo's content type, the
   user who uploaded it, its retention rule and its checksum are attached
   to the object itself, so no separate database or filesystem attribute
   scheme is needed and the object is self-describing wherever it is
   copied.
5. **The client is anywhere.** The consumer of a web application's data is
   a browser, a phone or a server in another region. Mounting a filesystem
   across the internet is impractical and insecure; an authenticated HTTP
   request works from every language on every network and passes through
   firewalls and load balancers that already exist.
6. **Cost.** By refusing to do the expensive things (locks, in-place
   updates, directory consistency), an object store can run on the cheapest
   servers with the largest, slowest disks and still keep every object
   online. It is usually the least expensive tier that answers a request in
   under a second.

**What happens if we do not have it.** Every large-scale system ends up
rebuilding it badly. Storing billions of photos in a filesystem hits
directory and inode limits, funnels all metadata through one server, and
forces every web server to mount the same share. Storing them in a database
bloats the database with blobs it was never designed to page through.
Storing them on block volumes means every volume is captive to one host
and there is no shared namespace at all. Backups, media, logs, datasets
and static web content all have the same shape (large, written once, read
many times, kept for years, accessed over a network) and none of the older
models fit that shape without serious contortions.

**What it deliberately does not do.**

- **No partial updates.** Changing one byte of a 40 GiB backup means
  uploading 40 GiB. Databases, virtual machine disks and anything that
  edits files in place cannot live directly on object storage.
- **No mounting or booting.** There is no block device and no POSIX API.
  Tools exist that present a bucket as a filesystem, but they emulate
  directories with prefix listings and emulate writes with whole-object
  replacement, and their performance and semantics reflect that.
- **Per-request latency.** Every operation is an HTTP request through
  authentication, load balancing and often a metadata lookup. Reading a
  hundred million tiny objects one by one is slow; reading one large
  object is fast. Applications batch and design keys accordingly.
- **Consistency by design, not by default.** Historically some object
  stores returned stale listings or an old version shortly after an
  overwrite (eventual consistency). Many modern systems now offer strong
  read-after-write consistency for single objects, but the interface does
  not promise anything about groups of objects, and there is no atomic
  rename or multi-object transaction.

### 3.5 Why exactly three, and why not just one

Each model is the minimum contract for one kind of consumer, and each
one's deliberate gaps are the next one's reason to exist:

```
                 names?   sizes?   shared?   in-place   POSIX?   scale-out   who wants it
                                             edits?              namespace?
 Block            no       no       one host   yes       below    no          OS, DB, VM
 File             yes      yes      yes        yes       yes      hard        people, programs
 Object           key      yes      yes        no        no       yes         web-scale apps
```

- Take **block** away and nothing can boot, no VM has a disk, and databases
  lose control of their I/O.
- Take **file** away and there is no shared, named, protected place for the
  thousands of small mutable things that people and programs work with
  every second.
- Take **object** away and there is no affordable, durable, internet-facing
  home for the enormous volume of write-once data that modern applications
  generate.

Could one model do all three jobs? A POSIX filesystem *can* be made to
scale out, but only by paying for distributed locking on every operation
that object-storage workloads never needed. An object store *can* be given
a filesystem front end, but partial writes then cost a full-object rewrite.
A block device *can* be shared, but only by putting a cluster filesystem
on it, which is just the file model again. Every attempt to collapse the
three into one ends up re-implementing the others on top, which is exactly
how real systems are built:

```
   application
      │
   object store  ── each object is stored on ── local filesystem or raw block device
                                                    on each storage node
   block volume  ── may be striped across ─────── objects in a cluster
   filesystem    ── always sits on ────────────── a block device
   file gateway  ── translates POSIX calls into ── object PUT / GET
```

The layering is circular on purpose. The models are not a ladder from
worse to better; they are three interfaces, and the right one is the one
whose contract matches the consumer.

### 3.6 Choosing between them: a worked example

Consider a photo-sharing service. It needs all three at once:

- **Block**: the relational database that holds users, albums and
  comments runs on a block volume. The database engine wants raw,
  low-latency, in-place writes with precise `fsync` control, and it runs
  on one host at a time (with a replica on another volume for failover).
  The virtual machines that host the web tier also boot from block
  volumes.
- **File**: the build servers, the shared configuration, the scratch
  space where an image-processing pipeline writes intermediate files, and
  the developers' home directories all live on a network filesystem. The
  programs involved expect paths, byte-level edits and POSIX behaviour, and
  the data is modest in size and heavily mutated.
- **Object**: every uploaded photo, every generated thumbnail, the nightly
  database backups, the access logs and the training data for the
  recommendation model go into an object store. Each item is written once,
  read many times from anywhere in the world, must survive the loss of a
  data centre, and there are billions of them.

Put the database on object storage and every row update becomes a whole
file upload. Put the photos on the database's block volume and the volume
is captive to one host and fills within a week. Put them on the network
filesystem and the metadata server collapses under the item count. Each
type is doing the one job it was designed for.

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
