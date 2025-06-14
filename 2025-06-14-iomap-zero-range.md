# Understanding IOMAP Zero-Range : Handling Delalloc, Unwritten Extents, and Brian Foster’s Folio Batch Optimization

## Introduction

This post documents the full data path of buffered I/O in Linux’s iomap infrastructure, covering:

- Delayed allocation (delalloc)
- Unwritten extents
- Extent state transitions
- Pagecache dirty folio state
- Writeback
- And finally, Brian Foster’s optimization for `fallocate(FALLOC_FL_ZERO_RANGE)` over unwritten mappings.

This material is inspired by Brian Foster’s RFC patch series:

> iomap: zero range folio batch processing prototype\
> [LKML link](https://lore.kernel.org/linux-fsdevel/20241119154656.774395-1-bfoster@redhat.com/)

---

## Quick Terminology

| Term          | Meaning                                                    |
| ------------- | ---------------------------------------------------------- |
| `iomap->type` | Returned by `iomap_begin()`                                |
| `br_state`    | Filesystem extent state (XFS extent tree)                  |
| DELALLOC      | Delayed allocation: logical reservation, no physical block |
| UNWRITTEN     | Physical blocks allocated, data not yet valid              |
| WRITTEN       | Physical blocks allocated, valid data                      |
| Folio         | Unit of memory inside pagecache                            |

---

## Buffered Write: Two Scenarios

### 1️⃣ Write into hole (Delayed Allocation)

- `iomap_begin()` returns `IOMAP_DELALLOC`
- Filesystem extent tree stores:

```
br_startblock = NULLBLOCK (delalloc)
br_state = N/A
```

- Dirty folios written into pagecache.

- **Extent state change happens at writeback.**

- At writeback:

```
iomap_write_begin() allocates physical blocks
Extent state updated to WRITTEN
```

---

### 2️⃣ Write into preallocated unwritten extent (Data Conversion)

- `fallocate(FALLOC_FL_KEEP_SIZE)` preallocates physical blocks.
- Filesystem extent:

```
br_startblock = physical block
br_state = UNWRITTEN
```

- `iomap_begin()` returns `IOMAP_UNWRITTEN`.

- On buffered write:

```
iomap_write_begin() triggers data-conversion immediately:
Extent state transitions UNWRITTEN → WRITTEN for written subrange.
```

- Dirty folios written into pagecache.
- **Extent state change happens at write time (before writeback is even queued).**

---

## The Problem with Zero-Range on Unwritten Mappings

When issuing:

```bash
fallocate(FALLOC_FL_ZERO_RANGE, offset, length)
```

We may encounter:

- Dirty folios in pagecache overlapping unwritten extents.

- Current kernel forces:

  1. Flush dirty folios (even stale data).
  2. Then apply zero-range.

- This results in:

- 🔴 Unnecessary write I/O

- 🔴 Risk of stale data hitting disk momentarily

- 🔴 Two writes (first flushing stale data, then writing zeros)

---

## Brian Foster’s Optimization

### Key Insight:

> We can avoid flushing dirty folios entirely by locking them directly in pagecache, zeroing them in place, and marking them dirty again.

---

### High-Level Flow:

| Step                 | Description                                                       |
| -------------------- | ----------------------------------------------------------------- |
| `iomap_begin()`      | Filesystem gathers dirty folios using `iomap_fill_dirty_folios()` |
| `iomap->fbatch`      | Stores batched dirty folios                                       |
| `IOMAP_F_HAS_FOLIOS` | Flag to signal batch exists                                       |
| `iomap_zero_iter()`  | Walks folio batch and zeroes folios directly                      |

---

### Call Flow:

```c
iomap_zero_range()  
└── iomap_apply()  
    └── iomap_iter()  
        └── xfs_buffered_write_iomap_begin()  
            └── iomap_fill_dirty_folios() (Brian’s hook)  
    └── iomap_zero_iter()  
        └── folio_batch_next() (Brian’s hook)
```

---

### Benefits:

- ✅ No need to flush dirty folios.
- ✅ Avoids race with writeback and reclaim.
- ✅ Cleaner, efficient zero-range implementation.
- ✅ Limited, targeted patch — only modifies `iomap_begin()` and `iomap_zero_iter()`.

---

## Unified Timeline Example

Let’s walk a concrete example:

| Logical Region | Extent State                    | Pagecache State |
| -------------- | ------------------------------- | --------------- |
| 100–200        | UNWRITTEN                       | No folios       |
| 120–140        | WRITTEN (data-conversion write) | Dirty folios    |
| 140–160        | UNWRITTEN                       | No folios       |

- Zero-range called on 120–160.
- Existing kernel flushes dirty folios on 120–140.
- Brian’s patch locks 120–140 folios, zeroes them directly.

---

## When Does Extent State Change?

| Operation                            | Extent State Transition?                                  |
| ------------------------------------ | --------------------------------------------------------- |
| Hole write                           | At writeback                                              |
| Buffered write into unwritten extent | At write time                                             |
| Writeback                            | ❌ No extent state change                                  |
| Zero-range                           | ❌ No extent state change (unless hole allocation happens) |

---

## Code Paths Summary

| Layer             | Function                                                        |
| ----------------- | --------------------------------------------------------------- |
| Write path        | `iomap_file_buffered_write()`                                   |
| Write begin       | `iomap_write_begin()`                                           |
| Extent conversion | `xfs_bmapi_convert_delalloc()`, `xfs_bmapi_convert_unwritten()` |
| Zero-range        | `iomap_zero_range() → iomap_apply() → iomap_iter()`             |
| Brian’s hook      | `xfs_buffered_write_iomap_begin()` & `iomap_zero_iter()`        |

---

## Closing

This optimization is a great example of how subtle correctness and performance issues arise when pagecache, filesystem extent state, and iomap mapping logic interact — especially when overlapping dirty data, preallocated unwritten extents, and zero-range syscalls collide.

---

## References

- [Brian Foster’s patch series](https://lore.kernel.org/linux-fsdevel/20241119154656.774395-1-bfoster@redhat.com/)
- [iomap documentation in kernel tree](https://github.com/torvalds/linux/tree/master/fs/iomap)

---

*Written after deep-dive analysis and breakdown discussions.*

