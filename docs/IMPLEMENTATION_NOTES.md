# Implementation Notes (if anyone builds this)

These notes are not a tutorial and not a checklist of the obvious. They are a
map of the places where a straightforward implementation goes subtly wrong, or
where a part is harder than it looks. Read the relevant spec section first; this
file only flags the traps around it.

Nothing here is binding. The specs define *what* must hold; this file is one
opinion on *where the work actually is*. Pick the subset that matches whatever
you decide to build.

---

## A build order that stays shippable at every step

Each milestone is crash-safe on its own, so you can stop at any of them and have
something correct rather than something half-wired.

- **M1 — read-only core.** STORE + standard DEFLATE providers, EAGER index,
  Page Cache, `mmap`'d vmidx reads. No writes at all. This proves the seek-index
  and decompression path, which is the part most likely to be wrong.
- **M2 — minimal writes.** Tier 1 only (`dirty_limit=UNLIMITED`, no spill),
  `commit()` via the FULL path only (build `archive.new.zip`, `rename()`). This
  is the simplest crash-safe write that exists; no journal yet.
- **M3 — durability.** Tier 2 spill + the vmdirty journal + the recovery
  protocol. This is where the real correctness lives. Do not rush it.
- **M4 — efficiency.** INCREMENTAL commit + Dead Space Freelist.
- **M5+ — optional.** VMM-native DEFLATE, Zstandard, background commit, async
  spill, adaptive checkpointing, `sidecar_dir` registry. All of these are
  performance or convenience; none change what the format can express.

---

## Durability and crash safety — the load-bearing part

This is the section to be paranoid about. Everything else is recoverable from a
bug; this isn't.

- **The `fdatasync` ordering *is* the design.** The "crash before / after step N"
  annotations scattered through the specs are not prose — they are your test
  oracle. If you can't point at which `fdatasync` makes a given claim true, the
  claim isn't true yet.
- **`rename()` is atomic but not durable by itself.** To survive a crash you must
  `fsync` the *containing directory* after the rename, otherwise the rename can be
  lost even though the new file's contents were synced. The same applies to the
  `mkdir` of `archive.zip.vmm/` — sync the parent dir before writing sidecar
  files into it.
- **`fdatasync` *failure* is not retryable.** After an `EIO` the kernel may have
  already dropped the dirty page while your in-memory copy still looks clean
  (this is the 2018 "fsyncgate" lesson). Treat it as loss of the durability
  guarantee and go to ERROR state — do not loop and retry as if it were
  transient.
- **CRC-32C is Castagnoli, not zlib's `crc32()`.** zlib's `crc32()` is the
  ISO-HDLC polynomial; the journal uses CRC-32C (the SSE4.2 / ARM-CRC one). Using
  the wrong polynomial "works" in the sense that it round-trips — and silently
  fails to match the spec and any other CRC-32C reader. Use a real CRC-32C.

---

## DEFLATE random access — the single hardest piece

Restoring a decompressor to the middle of a stream is the part people
underestimate the most.

- **It is bit-level, not byte-level.** DEFLATE blocks do not align to byte
  boundaries, so a checkpoint is a byte offset *plus* a leftover-bits count, and
  restore must re-inject those bits before decoding. With zlib this is
  `inflatePrime()` for the partial byte, `inflateInit2(strm, -15)` for raw
  inflate, and `inflateSetDictionary()` to load the 32 KB window. The canonical
  reference is `examples/zran.c` in the zlib source — read it before writing your
  own.
- **The 32 KB window is the *uncompressed* preceding output**, fed as the
  dictionary, not a slice of the compressed stream. Getting this confused
  produces output that is plausible-looking garbage.
- This bit-level cost is exactly *why* standard-DEFLATE indexes are huge and why
  VMM-native DEFLATE and Zstandard exist. If random access into third-party
  DEFLATE feels too expensive, that's the design telling you to prefer the other
  two for archives you generate.

---

## VMM-native DEFLATE — defer it, and fence the fiddly sub-problem

This whole provider is pure optimization. INCREMENTAL and FULL commits already
work without it. Build it last, behind tests.

- **Block completion journaling must come before any in-place overwrite.** A
  block's *clean* pages exist only inside the block you are about to overwrite;
  if you overwrite first and crash, they're gone. Journal the clean pages of
  every dirty block (with a COMMIT MARKER + `fdatasync`) before touching the
  archive. See CRASH SAFETY OF IN-PLACE COMMIT.
- **Padding tiling is a self-contained nasty problem.** Filling a capacity slot
  uses empty stored blocks (`00 00 FF FF`) plus empty fixed-Huffman blocks for
  bit-phase, giving tileable gap sizes `{0, 5, 6, 7}` and every size `≥ 9`, with
  `1–4` and exactly `8` untileable. This is bit-manipulation with off-by-one
  teeth — get it correct and unit-tested in isolation before it touches the
  commit path, and keep the overflow→INCREMENTAL fallback as the safety net.

---

## Dead Space Freelist — the live CD/EOCD trap

- The Central Directory and EOCD (and the ZIP64 EOCD record + locator, when
  present) are **live allocated regions**, not free space. The gap between the
  last entry and the CD is *not* a hole. Treat it as one, write into it, and you
  have corrupted a valid archive at the one offset readers depend on. CD/EOCD
  regions left by *prior* commits are genuinely dead and reusable — but only the
  ones the current EOCD no longer points at. This is already in the spec; it's
  listed here because it's the kind of thing a fast reader skips.

---

## vmdirty journal and recovery

- **One `write()` per record.** Build the whole record (header through CRC) in a
  buffer and emit it in a single call to shrink the torn-write window; open with
  `O_DSYNC` in sync-spill mode.
- **The recovery walk stops at the first failure** — bad CRC, generation-ID
  mismatch, or truncation — and never trusts anything past that point, even if
  later bytes happen to parse. A gap means the tail is untrustworthy, full stop.
- **Compaction renumbers sequences from 1 only because it mints a new
  generation ID.** That is the whole reason a new generation ID is drawn. If you
  ever reuse a generation ID, you lose the right to renumber and the recovery
  reasoning breaks.

---

## Locking and `fork()`

- **`flock` and `fcntl` locks are not interchangeable.** `flock` is per
  open-file-description (so a `dup`/`fork` shares it); `fcntl` is per process (so
  a second mount *in the same process* cannot be arbitrated by it — the process
  already "owns" the lock). Same-process double-mount needs `flock`. Choose
  deliberately, per the LockProvider section.
- **`O_CLOEXEC` on the lock fd**, or an `exec()` after `fork()` leaks the lock
  into the new image.
- **The `pthread_atfork` prepare handler does no I/O.** It only takes internal
  mutexes. The "obvious" version calls `flush()` there — that runs disk I/O while
  holding locks inside an atfork handler, which deadlocks against other
  libraries' handlers. The spec removed it for exactly this reason.

---

## External-change detection

- **An inode mismatch alone does not prove the archive changed.** Inode
  recycling, a restored backup, and an AOT-distributed index all present a
  different inode over identical bytes. Let `cd_hash` decide, and on a `cd_hash`
  match refresh the stored size/inode/mtime in place.
- **`mtime` is a runtime *trigger*, not an `open()`-time gate.** It is unreliable
  on NFS/FAT, so it never gates validity — but on the read path it cheaply
  catches a same-size in-place edit that size+inode would miss, with any false
  positive resolved by `cd_hash`.

---

## ZIP parsing and untrusted input

- **ZIP64 is signalled by sentinels.** `0xFFFF` (counts/disk numbers) and
  `0xFFFFFFFF` (sizes/offsets) in the regular EOCD mean "the real value is in the
  ZIP64 EOCD." Locate the ZIP64 EOCD *locator* (it sits just before the regular
  EOCD) first; don't trust the 32-bit fields when the sentinel is present.
- **Bounds-check before allocating.** Validate CD entry counts, offsets, and
  sizes against the file length *before* allocating anything proportional to
  them. Overlapping entry regions and a decompressed stream exceeding its
  declared size are your zip-bomb / smuggling guards — enforce the declared size
  during inflate rather than trusting it.
- **Entry names are opaque keys.** The VMM never builds a filesystem path from an
  entry name, so `../` and absolute prefixes are just bytes here. The traversal
  risk lives in whatever *extracts* entries — keep that boundary clear.

---

## vmidx is a cache — lean on it

- **Any validation failure means discard and rebuild.** There is no repair mode
  and no partial-trust mode, by design. This removes a whole class of "recover a
  half-written index" code you might otherwise feel obligated to write — don't
  write it.
- **The advisory region is deliberately unprotected** (no CRC). Miss counters and
  block-state records can be lost or garbage with zero correctness impact; clamp
  implausible values to zero on load instead of trusting them or treating them as
  corruption.

---

## Small but bite-y

- **The last page of an entry is short** (`data_len < page_size`). This is an
  off-by-one farm across read, write, spill, and recompress — centralise the
  "size of page i of entry e" calculation in one function.
- **Implicit extension zero-fills gaps.** A `write()` past `logical_size` extends
  the entry; pages in the gap that were never written must read as zeros and be
  materialised as zero pages at commit. Recovery derives `logical_size` as the
  maximum page extent seen, so there's no separate record for it — make sure the
  final short page's `data_len` is right.
- **`mmap` the vmidx with `madvise(MADV_RANDOM)`** and keep the in-memory
  structures (dirty index, hot-entry tables) bounded *separately* from the
  mapping, so a multi-GB standard-DEFLATE index doesn't drag the resident set
  with it.

---

## Testing strategy

- **Crash-inject at every enumerated barrier.** The specs already list the points
  ("crash before step 4", "crash after rename", …); turn that list directly into
  a fault-injection matrix.
- **One invariant rules them all:** at *every* crash point, `archive.zip` must
  still open cleanly in a stock extractor (Python `zipfile`, `unzip`, 7-Zip).
  Make that a property test driven by the fault injector — if it ever fails, a
  durability-ordering claim is wrong.
- **Fuzz `open()` with malformed ZIPs.** The containment guarantees in UNTRUSTED
  ARCHIVES are only as good as the parser; a fuzzer over truncated, overlapping,
  and oversized-declaration inputs is the cheapest way to find the gaps.
