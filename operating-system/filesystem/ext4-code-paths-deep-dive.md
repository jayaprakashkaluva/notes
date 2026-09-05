# ext4 Code Paths — Deep Dive (Companion Volume)

> Companion to `ext4-filesystem-internals.md`. That document covers architecture; this one
> traces the actual code paths function-by-function, with the invariants and failure
> handling that the architecture summary glosses over. Line references are into
> `fs/ext4/` at kernel ~7.3-rc (2026).

---

## 1. Anatomy of `ext4_map_blocks()` — the central dispatcher

`inode.c:700`. Every I/O path (buffered writeback, DIO via iomap, fiemap, fallocate,
readahead) funnels through this one function. Its structure is a textbook
**optimistic three-phase lookup**, each phase more expensive and more exclusive:

### Phase 0 — lockless ES-tree hit (`inode.c:738`)

```c
if (ext4_es_lookup_extent(inode, map->m_lblk, NULL, &es, &map->m_seq)) { ... }
```

No `i_data_sem` at all. The extent status tree answers from memory, and the result is
stamped with `map->m_seq` = `i_es_seq` (a per-inode sequence counter bumped on every ES
mutation). Callers that defer acting on the mapping (iomap writeback, DIO) can later
detect staleness by comparing sequence numbers instead of holding locks across I/O —
this is the same *seqcount validation* idiom as `d_seq` in path walking.

Four cache outcomes: written/unwritten → return mapping directly; delayed/hole → return 0
with `m_len` = usable hole/delayed length (so callers can batch). `else BUG()` — the ES
tree is trusted absolutely (see the consistency contract, companion doc §9).

`EXT4_GET_BLOCKS_CACHED_NOWAIT` (`inode.c:761,777`): pure cache probe, returns without
ever touching disk or sleeping — used by paths that can't block (e.g. `->map_pages`-style
opportunistic mapping).

Also note the debug assertion strategy: `ES_AGGRESSIVE_TEST` recheck (`inode.c:763`)
re-executes the slow path and compares against the cache answer — cache coherence bugs
are made loud rather than silently corrupting.

### Phase 1 — read-locked on-disk query (`inode.c:784`)

```c
down_read(&EXT4_I(inode)->i_data_sem);
retval = ext4_map_query_blocks(handle, inode, map, flags);
up_read(&EXT4_I(inode)->i_data_sem);
```

`ext4_map_query_blocks` (`inode.c:560`) dispatches on `EXT4_INODE_EXTENTS` to
`ext4_ext_map_blocks` or `ext4_ind_map_blocks` *without CREATE*, then **populates the ES
cache** with the result (`ext4_es_cache_extent`, `inode.c:594`) so the next lookup takes
phase 0. The `QUERY_LAST_IN_LEAF` refinement: if the found extent is the last in its leaf
and didn't cover the whole request, query the *next* leaf too before caching — avoids
caching artificially short extents at leaf boundaries.

Every result is checked against `block_validity.c`'s system-zone rbtree
(`check_block_validity`, `inode.c:790`) — a corrupted extent tree pointing into bitmaps
or the journal is caught here, returning `-EFSCORRUPTED` before any I/O touches metadata.

### Phase 2 — write-locked allocation (`inode.c:822`)

Only if `EXT4_GET_BLOCKS_CREATE` and the lookup didn't fully satisfy. Note the
re-dispatch inside `ext4_map_create_blocks` (`inode.c:629`) re-tests `EXT4_INODE_EXTENTS`
— **the inode's mapping format can change between phase 1 and phase 2** (online
migration, `migrate.c`), because the lock was dropped in between. This
drop-and-revalidate is a recurring ext4 pattern; forgetting the revalidation after a lock
gap is a classic source of ext4 CVEs.

Post-allocation obligations, in order (`inode.c:660-860`):

1. **Zeroout before ES insert** (`inode.c:653-666`): if the caller demanded zeroed blocks,
   they must be zeroed *before* the mapping becomes visible in the ES tree, else a
   concurrent lookup could read stale disk contents. The comment also notes metadata
   buffers must be unmapped before zeroing or writeback could overwrite zeros with stale
   bh data — a two-way ordering constraint.
2. **ES insert** with the delalloc-reserve flag threaded through (quota/cluster
   accounting must know whether this allocation consumes a prior reservation).
3. **Ordered-mode data tracking** (`inode.c:841-857`): freshly allocated *written* blocks
   on a `data=ordered` inode are attached to the transaction's inode list via
   `ext4_jbd2_inode_add_write/wait` — this is the actual mechanism of ordered mode. It's
   a *byte range* on a `jbd2_inode`, not journaled data blocks: at commit time jbd2
   writes those page-cache ranges out before committing the metadata that references
   them. `IO_SUBMIT` callers use the `_wait` variant since their data is already in
   flight.
4. **Fast-commit range tracking** (`inode.c:859`).

### The locking exception worth memorizing

`inode.c:727-735`: mapping queries normally require `i_rwsem` or `invalidate_lock` or a
folio lock (asserted by `ext4_check_map_extents_env`). The *only* exception is
`EXT4_GET_BLOCKS_IO_SUBMIT` (writeback data submission, which holds only folio locks) —
and it is required to pass `EXT4_EX_NOCACHE`, i.e. it may look but must not pollute the
cache with ranges it doesn't hold locks for. Locking exemptions being *paired with
caching restrictions* is the giveaway that the ES tree's correctness argument depends on
who is allowed to insert.

---

## 2. `ext4_ext_map_blocks()` walkthrough (`extents.c:4277`)

The extent-tree side of phase 1/2. Control flow:

### Found-extent fast path (`extents.c:4316-4366`)

`ext4_find_extent` builds the path array; if the target lblk falls inside the found
extent:

- **Initialized extent** → return mapping, clamp `m_len` to what remains of the extent.
- **Initialized + `CONVERT_UNWRITTEN`** → `convert_initialized_extent` (the reverse
  conversion, used by e.g. atomic-write requirements).
- **Unwritten extent** → `ext4_ext_handle_unwritten_extents`: the hard case. Depending on
  flags, either return it as-is (reads see a hole; DIO into it proceeds and converts at
  I/O end), split it (write covers only part → up to three pieces, only the middle
  converting), or convert with zeroout: if the split would produce tiny fragments, ext4
  prefers to **write zeros to disk** over creating more extents
  (`EXT4_EXT_ZERO_LEN` heuristic) — trading I/O for tree compactness.

Note `extents.c:4302-4314`: an empty leaf with depth > 0 is `-EFSCORRUPTED`, but the
comment explains why this *can't* be asserted inside `ext4_find_extent` — the tree is
legitimately transiently empty during modification. Knowing where an invariant is
checkable is itself documented knowledge.

### Allocation path (`extents.c:4383-4527`)

1. **Bigalloc first-chance** (`extents.c:4389-4419`): with clusters > 1 block, the
   request may fall in a cluster that's *already allocated* to this file (its head or
   tail used by a neighboring extent). `get_implied_cluster_alloc` detects this and skips
   the allocator entirely — the physical blocks exist, only the mapping is missing. This
   is checked twice: against the left neighbor from the path, and against the right
   neighbor found by `ext4_ext_search_right`.
2. **Goal computation**: `ext4_ext_find_goal` picks a target near the logical
   predecessor's physical block; `ar.lleft/lright` + `pleft/pright` give mballoc the
   physical neighbors so allocation continues contiguously in *both* directions.
3. **Length clamping** to `EXT_INIT_MAX_LEN`/`EXT_UNWRITTEN_MAX_LEN` (32768/32767) and
   overlap check against the next existing extent.
4. **Cluster-aligned request construction** (`extents.c:4446-4457`): goal, logical, and
   length are all shifted down by the intra-cluster offset so the allocated cluster's
   phase matches the logical phase — required for future `get_implied_cluster_alloc`
   correctness. Bigalloc is not a local hack; it distorts every calculation in this
   function (`allocated_clusters` vs `ar.len` bookkeeping).
5. **`ext4_mb_new_blocks`** does the actual allocation (§4).
6. **Insert + rollback** (`extents.c:4490-4513`): `ext4_ext_insert_extent` may itself
   need to allocate tree blocks and can fail with ENOSPC *after* data blocks were
   allocated. Recovery: discard this inode's preallocations and free the just-allocated
   blocks — but only for ENOSPC/EDQUOT. For other errors (corruption), it deliberately
   **leaks the blocks**: "If the filesystem is inconsistent, we'll just leak allocated
   blocks to avoid causing even more damage" (`extents.c:4494`). Choosing the leak over
   the risky repair when invariants are already broken is a policy worth noticing.
7. **fsync-relevance tracking** (`extents.c:4516-4522`): `ext4_update_inode_fsync_trans`
   records the transaction tid in `i_sync_tid` and — only if the extent is written, i.e.
   changes data visibility — `i_datasync_tid`. This is what makes
   `fdatasync()` cheaper than `fsync()` (§6): unwritten-extent allocation doesn't
   obligate a datasync commit because reads see zeros either way.

---

## 3. The writeback state machine (`inode.c:2764`, `ext4_do_writepages`)

The `mpage_da_data` struct is a cursor over the mapping: `start_pos/next_pos/end_pos`,
the accumulated `map` (extent needing allocation), `io_submit` (bio + `io_end` being
built), and mode bits `do_map`/`can_map`/`scanned_until_end`.

### Two-pass structure

**Pass A — no transaction** (`inode.c:2871-2892`): walk dirty folios and submit anything
*already fully mapped* with `do_map = 0`. Rationale in the comment: don't start a jbd2
handle (which pins the running transaction and can stall every other filesystem user)
just to discover you're blocked on device congestion writing already-mapped pages.
Transaction hold time is treated as a global resource to be minimized.

**Pass B — per-extent transactions** (`inode.c:2894-2991`): loop while there are
unmapped-dirty pages and quota (`nr_to_write`) remains:

1. Fresh `io_end` per extent (unwritten-conversion completion object).
2. `ext4_journal_start_with_reserve(inode, EXT4_HT_WRITE_PAGE, needed_blocks, rsv_blocks)`
   — credits for one extent of `MAX_WRITEPAGES_EXTENT_LEN` (2048) blocks, plus a
   **reserved sub-handle** (`rsv_blocks`) when `dioread_nolock` is on: the credits that
   the *end-of-I/O unwritten conversion* will need are reserved now, up front
   (`inode.c:2841-2848`), because the conversion runs later from a workqueue when asking
   for fresh credits might deadlock against this very transaction.
3. `mpage_prepare_extent_to_map` (`inode.c:2608`): gathers the next run of dirty pages,
   locking them, accumulating contiguous delayed/unwritten buffers into `mpd->map`. Pages
   already mapped are submitted on the fly. In `data=journal` mode (`can_map = 0`) pages
   are never mapped here — journalled data doesn't do delalloc at all; the function
   instead re-journals page buffers (`mpage_journal_page_buffers`, `inode.c:2570`).
4. `mpage_map_and_submit_extent` (`inode.c:2452`): loop `mpage_map_one_extent` →
   `ext4_map_blocks(CREATE)` until the accumulated range is fully mapped. Key properties:
   - Allocation may return *less* than asked (credits, fragmentation); the loop
     continues. Forward-progress guarantee: the last touched page is always fully mapped
     before returning (comment at `inode.c:2447`).
   - **Transient-error handling** (`inode.c:2480-2492`): ENOMEM/EAGAIN, or ENOSPC while
     free clusters exist (a commit will release pinned-freed blocks), return the error
     upward for retry — but if partial progress was made inside a folio, first submit the
     already-mapped buffers (`mpage_submit_partial_folio`) so *allocated-but-unwritten
     disk blocks are never left containing stale data* while the page stays dirty.
   - **Hard failure** → `give_up_on_write = true`; the caller discards dirty pages
     ("Data will be lost" — `inode.c:2502`) rather than loop forever redirtying.
5. **`i_disksize` advance** (`inode.c:2520-2545`): after submission, on-disk size is
   raised to the end of submitted data, under `i_data_sem` and rechecked against `i_size`
   to not race truncate. `i_disksize` vs `i_size` is ext4's answer to
   size-vs-data crash ordering with delalloc: `i_size` (user-visible) may run ahead of
   `i_disksize` (journaled) — a crash then shows the shorter size instead of a
   tail of zeros/garbage.
6. **Handle stop timing** (`inode.c:2940-2971`): a *synchronous* handle's
   `ext4_journal_stop` can block on transaction commit, which may itself wait on this
   very writeback's page locks and io_end references → the code carefully submits bios,
   releases page locks, and defers the io_end drop before stopping a sync handle. A
   compact real-world example of a lock-order-through-completion dependency.
7. **ENOSPC retry** (`inode.c:2976`): force a nested commit (frees blocks pinned by the
   committing transaction) and continue; `-EAGAIN` restarts benignly.

Around all of this, `s_writepages_rwsem` is held for read; `super.c`-side operations
that flip an inode's mapping *mode* (migration, journal-data toggle) take it for write.

---

## 4. mballoc: allocation-context lifecycle (`mballoc.c:6230, 3001`)

`ext4_mb_new_blocks` builds an `ext4_allocation_context` (`ac`) holding three extents:
`ac_o_ex` (original request), `ac_g_ex` (goal, post-normalization), `ac_b_ex` (best
found). Flow:

1. Try inode PA / locality-group PA (fastest, no group scan at all).
2. Normalize request (`ext4_mb_normalize_request`) — file-size-tiered rounding.
3. `ext4_mb_regular_allocator` (`mballoc.c:3001`):
   - **Goal attempt** first (`ext4_mb_find_by_goal`) — physical continuity beats
     everything else, so the goal group is scanned before any heuristic.
   - `ac_2order` set only for power-of-two lengths ≥ `mb_order2_reqs` — enables the
     buddy-exact path and the `CR_POWER2_ALIGNED` criterion. Note
     `array_index_nospec` (`mballoc.c:3036`) — Spectre-v1 hygiene on an
     attacker-influenceable array index, deep inside a filesystem allocator.
   - **Stream allocation** (`mballoc.c:3040-3047`): small-file ("stream") allocations
     ignore the per-inode goal and instead use a **global rotating goal** hashed by inode
     number over `s_mb_nr_global_goals` slots (`s_mb_last_groups[hash]`) — successive
     small-file writes from different inodes chain into the same region of the disk.
     This is newer than the classic design comment: it replaced the single
     `s_mb_last_group` to reduce cross-CPU cacheline bouncing while keeping the
     "pack streams together" behavior.
   - **Criteria ladder loop** (`mballoc.c:3061-3069`): `ext4_mb_scan_groups` per
     criterion, escalating CR_POWER2_ALIGNED → GOAL_LEN_FAST → BEST_AVAIL_LEN →
     GOAL_LEN_SLOW → ANY_FREE until `AC_STATUS_FOUND`.
   - **Best-found fallback** (`mballoc.c:3071-3099`): scanning is bounded by
     `mb_max_to_scan`; on exhaustion, try to grab the best extent seen so far. If a racer
     took it (`s_mb_lost_chunks` counter), restart the whole ladder at `CR_ANY_FREE` with
     `EXT4_MB_HINT_FIRST` ("take literally the first free extent"). Statistical
     greed with a hard-bounded retreat to first-fit.
4. On success: mark bits in the on-disk bitmap under group lock, dirty
   bitmap+group-descriptor buffers in the handle, update free counters, possibly create a
   new PA from the surplus.
5. On failure: discard preallocations group-by-group and retry, governed by the percpu
   `discard_pa_seq` protocol (`mballoc.c:436`) — the retry is justified only if some
   CPU's alloc/free/discard activity changed the picture since the attempt began.

Freed-block parking (companion doc §5.4) is implemented as `ext4_free_data` records
sorted per-group in a rbtree on the committing transaction, released to the buddy in the
jbd2 commit callback — and merged aggressively so a rm -rf produces few records.

---

## 5. Truncate & punch-hole protocol (`inode.c:4458`, `extents.c:4551`)

Punch-hole is the fullest expression of the multi-transaction locking contract. Ordering
inside `ext4_punch_hole` (caller holds `i_rwsem` + `invalidate_lock` exclusive):

1. Clamp the range: no punching past `i_size` (extended to block boundary so the tail
   block isn't pointlessly zeroed); indirect-format files can't punch the last block
   (`s_bitmap_maxbytes` quirk, `inode.c:4477`).
2. **`ext4_update_disksize_before_punch`**: if `i_disksize > punch start` but dirty
   delalloc pages before the punch haven't updated it yet, journal the size *first* — a
   crash between page-cache truncation and extent removal must not expose a disksize
   covering now-missing blocks.
3. **Truncate page cache** for the range (`ext4_truncate_page_cache_block_range`) —
   possible only because both big locks exclude faults and reads.
4. **Zero partial edge blocks** (`ext4_zero_partial_blocks`) via the page cache; O_SYNC
   forces them out immediately.
5. Start handle (`EXT4_HT_TRUNCATE`) with per-format credits.
6. Under `i_data_sem` write: discard PAs → `ext4_es_remove_extent` → remove on-disk
   extents (`ext4_ext_remove_space`) → **insert an explicit HOLE ES entry**
   (`inode.c:4549`). The hole entry isn't decoration: it re-establishes the "ES tree is
   authoritative" invariant so subsequent lookups don't fault in stale mappings.
7. FC-track the range, mark inode dirty, sync handle if O_SYNC.

`ext4_ext_remove_space` itself may need many transactions (each extent freed dirties
bitmaps/descriptors); it restarts the handle at leaf boundaries using
`ext4_datasem_ensure_credits` — legal here only because steps 1-4 excluded all observers.
The classic akpm comment above `ext4_truncate` (`inode.c:4597`) states the recovery
principle: *the on-disk tree must be consistent and restartable at every commit
boundary* — truncate proceeds bottom-up so a crash mid-way leaves a valid, shorter tree,
and the **orphan list/file** ensures the truncate is restarted after replay ("journal
replay occurs before the restart of truncate against the orphan list",
`inode.c:4614`).

The same pattern with roles reversed appears in `ext4_ext_truncate` (`extents.c:4551`):
set `i_disksize = i_size` and journal it *before* removing blocks — crash-safe direction
depends on whether you're growing or shrinking visibility.

---

## 6. fsync (`fsync.c:128`)

Short but every line is a policy decision:

```
ext4_sync_file:
    file_write_and_wait_range()              # data via writeback (delalloc allocates here)
    if no journal: sync_inode_metadata + sync parent dirs (!) + flush
    else: ext4_fsync_journal → ext4_fc_commit(journal, commit_tid)
    if needs_barrier: blkdev_issue_flush()
```

- **tid selection** (`fsync.c:101`): `datasync ? i_datasync_tid : i_sync_tid`. Combined
  with §2's rule (unwritten-extent allocation doesn't bump `i_datasync_tid`),
  `fdatasync()` on a preallocated file skips the journal commit entirely if only
  non-visibility-affecting metadata changed.
- **`ext4_fc_commit`** is the *single entry point*: it decides internally whether a fast
  commit suffices or a full commit is needed (ineligibility, non-regular file — note
  directories force a full commit, `fsync.c:107`).
- **Barrier avoidance** (`fsync.c:110`): if the target transaction is going to issue a
  cache flush anyway (`jbd2_trans_will_send_data_barrier`), don't issue a second one.
  Storage flushes are expensive enough to justify this bookkeeping.
- **The no-journal path syncs the parent directory chain** (`ext4_sync_parent`,
  `fsync.c:88`): without a journal, a new file's dirent isn't durable unless the parent
  is written too — POSIX doesn't require it, but losing the file after fsync is
  indefensible. With a journal this is unnecessary (dirent and inode commit atomically).
- Error propagation: `file_check_and_advance_wb_err` at the end picks up async writeback
  errors recorded since the last fsync — the errseq_t mechanism that fixed the
  "fsync error reporting" class of bugs (PostgreSQL fsync-gate).

---

## 7. Direct I/O write (`file.c:575`)

DIO goes through iomap (`iomap_dio_rw` with `ext4_iomap_ops`), but the interesting part
is the **lock-mode decision matrix** in `ext4_dio_write_checks` (`file.c:493`):

| Condition | Lock | Why |
|---|---|---|
| Overwrite of allocated, initialized blocks, aligned | `i_rwsem` **shared** | Pure data overwrite; block mapping is immutable; parallel DIO writers are safe |
| Unaligned write needing partial-block zeroing | exclusive + `inode_dio_wait` | Zeroing the head/tail of a block races with any concurrent DIO into the same block |
| Extending write | exclusive + forced-sync completion | `i_disksize` must be updated after data lands; orphan-list protection spans the I/O |
| `!IS_NOSEC` (setuid/setgid present) | exclusive | Killing privileges modifies the inode |

Mechanics worth noting:

- **Optimistic lock, verify, restart** (`file.c:504,526-536`): take shared based on a
  lockless `i_size` guess, redo the checks under the lock, upgrade by
  full unlock-relock-goto-restart if wrong (never in-place upgrade — that deadlocks).
  `IOCB_NOWAIT` returns `-EAGAIN` at every point where blocking would be needed.
- **Extending writes are orphan-protected** (`file.c:635-646`): the inode goes on the
  orphan list *before* the I/O, so a crash after allocation but before the size update
  leaves an orphan whose recovery truncates the dangling blocks. Same machinery as
  truncate, reused for the mirror-image hazard.
- Extending DIO is forced synchronous (`IOMAP_DIO_FORCE_WAIT`, and
  `WARN_ON_ONCE(ret == -EIOCBQUEUED)` at `file.c:659`).
- **Short-DIO fallback** (`file.c:669-700`): if iomap completed only part of the request
  (e.g. hit a hole where DIO can't proceed), the remainder goes through buffered write,
  then is immediately written back and invalidated from cache — preserving the "DIO
  doesn't leave data in page cache" contract at some cost. Atomic writes must never take
  this path (`WARN_ON_ONCE(IOCB_ATOMIC)`).
- DIO into unwritten (fallocated) extents: mapping returns unwritten; conversion to
  written happens in `end_io` (`ext4_dio_write_ops`) in process/workqueue context with
  its own handle.

---

## 8. Inode allocation — Orlov in detail (`ialloc.c:404`)

`find_group_orlov` policy (comment at `ialloc.c:404-422`):

- **Top-level directories** (parent is root, or `EXT4_INODE_TOPDIR`): *spread* — pick
  among groups with ≥-average free inodes and clusters, the one with fewest directories;
  fallback random. Top-level dirs are assumed to be independent subtrees that will each
  grow.
- **Other directories**: *cluster with parent unless degraded* — parent's group is fine
  unless it exceeds `max_dirs` or falls below `min_inodes`/`min_clusters` thresholds;
  then scan cyclically for a group passing the thresholds; final fallback: any group with
  above-average free inodes.
- **Regular files** (`find_group_other`): parent's group, then quadratic probing
  (parent+1, +2, +4, +8…), then linear scan.
- With **flex_bg**, all of the above operates on flex-group aggregates
  (`ialloc.c:444-448` — group numbers shifted by `s_log_groups_per_flex`; stats summed
  via `get_orlov_stats`).

The bitmap-set itself: group lock only, then journal the bitmap block. Newly added
groups' unused inode-table tail (`s_itb_per_group` tracking via `BG_INODE_UNINIT` /
`bg_itable_unused`) lets mkfs skip zeroing inode tables; `ext4lazyinit` kthread zeroes
them in the background post-mount.

---

## 9. xattr internals (`xattr.c:17-52`)

On-disk layout (both in-inode area and external block): entry descriptors grow *down*
from the header, values grow *up* from the end, four null bytes separate — the same
two-ended arena as a dirent block or a B-tree page. In blocks entries are sorted, in the
inode area unsorted (in-inode sets are small; sort cost > scan cost).

Storage tiers, tried in order:

1. **In-inode** (`i_extra_isize` permitting) — free with the inode read.
2. **One external block** (`i_file_acl`), *shared copy-on-write across inodes*: identical
   attribute blocks are deduplicated via **mbcache** (hash of contents → block).
   Refcounted; any modification to a shared block allocates a private copy first
   (`xattr.c:48-51` — a block is only modified in place if exclusively owned; the
   refcount is the only mutable field of a shared block, serialized by the buffer lock).
   This exists because SELinux/ACL labels are massively duplicated across files.
3. **EA-inode** (`ea_inode` feature): each large value gets its own inode (up to 64K
   values), itself deduplicated by content hash + refcount. The parent references it by
   inode number + hash.

Locking: `xattr_sem` per inode guards `i_file_acl` and the in-inode area. The ugly part
of this file is credit estimation: a "set xattr" may cascade into
unshare-block + allocate-EA-inode + dedup-lookup, each with different journal credit
needs, computed pessimistically up front.

Deletion during inode eviction must walk and unref EA-inodes — one of the reasons
`ext4_evict_inode` (`inode.c:167`) has its own careful transaction management, including
the `EXT4_STATE_MAY_INLINE_DATA` / xattr interaction (inline data *lives in* the xattr
space as the `system.data` attribute).

---

## 10. Cross-cutting mechanics that only show up when reading the code

### `i_disksize` vs `i_size`
`i_size` is what stat shows and is updated eagerly; `i_disksize` is what's journaled and
only ever covers data whose blocks are actually allocated+submitted. Every write path has
an explicit "when do I advance disksize" decision: writeback advances it after bio
submission (§3.5); extending DIO after I/O completion (§7); truncate sets it *before*
removing blocks (§5). All three choices are forced by the same rule: *at any commit
point, [0, i_disksize) must be fully backed by valid data or zeros.*

### Handle-less operation
Many paths call `ext4_map_blocks(NULL, ...)` (readahead, fiemap, DIO overwrite of mapped
blocks). A NULL handle means "guaranteed no allocation" — the journal is only consulted
when metadata can change. Conversely `ext4_journal_current_handle()` assertions
(`fsync.c:138`) document paths that must *never* run under a transaction (fsync would
deadlock waiting for a commit that can't finish while a handle is open).

### Reserved handles (`ext4_journal_start_with_reserve`)
Two-phase credit acquisition: normal credits from the running transaction, plus a
reservation usable *later, from another context* (unwritten conversion at bio
completion). jbd2 guarantees the reserved credits are honored even if the running
transaction has since committed. This decouples "when I know I'll need credits" from
"when I use them" — without it, end-of-I/O conversion could deadlock on transaction
space.

### The `EXT4_STATE_*` per-inode runtime flags
Distinct from on-disk `EXT4_INODE_*` flags. Load-bearing examples seen in these paths:
`EXT4_STATE_MAY_INLINE_DATA` (cleared *before* DIO write checks to close a race window —
`file.c:625`), `EXT4_STATE_EXT_MIGRATE` (cleared when indirect-format allocation happens
mid-migration so the migration aborts instead of corrupting — `inode.c:640`),
`EXT4_STATE_FC_COMMITTING` (fast-commit snapshot fence).

### Trace points everywhere
Nearly every function traced here has a `trace_ext4_*` pair (enter/exit with results).
`/sys/kernel/debug/tracing` + these events reconstruct complete allocation/writeback
decisions in production without a debugger — the observability story is part of the
design, and mballoc additionally exports per-group state via procfs
(`ext4_mb_seq_groups_show`, `mballoc.c:3144`) and success-per-criterion counters
(`s_bal_cX_hits`).

---

## 11. Reading list for a third pass

Paths deliberately not traced here, roughly in order of educational value:

1. `ext4_ext_insert_extent` + `ext4_ext_split`/`ext4_ext_grow_indepth` — B+-tree node
   splitting under journal credits.
2. `extents_status.c` shrinker (`es_reclaim_extents`) and the `s_es_list` LRU — how not
   to let a cache OOM the box while keeping the load-bearing delayed entries.
3. `fast_commit.c` replay side (`ext4_fc_replay_*`) — the TLV state machine enforcing
   the idempotence rules.
4. `resize.c:ext4_flex_group_add` — journaling the creation of new metadata that the
   superblock doesn't reference yet.
5. `mballoc.c:ext4_mb_init_cache` — buddy generation from bitmap + PA overlay, and
   `ext4_mb_scan_groups`' prefetch pipeline (`ac_prefetch_*`).
6. `page-io.c` — io_end reference counting between submit path, bio completion, and
   conversion workqueue (three owners, one refcount).
7. `move_extent.c` — defragmentation by swapping extents between inodes with both
   `i_data_sem`s held in inode-number order.
