
# Deep Dive into IOMAP Buffered I/O Path in Linux Filesystem Stack

*Author: Anuj Gupta*

---

## Introduction

In this post, we take a detailed walkthrough of how Linux filesystems implement buffered I/O using the **IOMAP infrastructure**. We'll cover:

- IOMAP architecture
- IOMAP types
- Extent and disk block model
- Full buffered write path
- Folio interaction and page cache behavior
- Failure consistency model
- Dirty tracking with granular large folio support

This blog captures the complete understanding I developed while deep diving into IOMAP internals with lots of questions, examples, and visualizations.

---

## Why IOMAP Exists

Before IOMAP, each filesystem used to implement their own:

- Write paths
- Page cache interactions
- Zeroing logic
- Writeback logic

IOMAP abstracts these complexities and allows filesystems like XFS, ext4, F2FS, BTRFS, etc., to simply provide:

- Their own extent mapping logic via `iomap_begin()` and `iomap_end()`.
- Everything else (buffered write, read, writeback, dirty marking, zeroing etc.) is handled generically via IOMAP core.

---

## Extent vs Disk Block — Mental Model

| Term        | Meaning                                                      |
| ----------- | ------------------------------------------------------------- |
| **Extent**  | Logical mapping: `{ file_offset, length, disk_block_start }`  |
| **Disk Block** | Physical sector number (LBA on disk)                       |

> Extent maps file offset ranges to physical disk blocks.

---

## The IOMAP Types — Logical State Machine

| IOMAP Type     | Extent Allocated? | Disk Blocks Reserved? | Data Valid? |
| -------------- | ----------------- | ---------------------- | ------------ |
| `IOMAP_MAPPED` | ✅ Yes            | ✅ Yes                 | ✅ Yes       |
| `IOMAP_UNWRITTEN` | ✅ Yes        | ✅ Yes                 | ❌ No        |
| `IOMAP_HOLE`   | ❌ No             | ❌ No                  | ❌ No        |
| `IOMAP_DELALLOC` | ❌ No (in-memory) | 🔶 Logical reservation only | ❌ No |
| `IOMAP_INLINE` | ✅ Yes (in inode) | ✅ Yes                 | ✅ Yes       |

---

## Real Examples for IOMAP Types

### Case 1: Normal Written File

```
File: |=======================|
Type:   IOMAP_MAPPED
```

### Case 2: Sparse Hole

```
File: |====|.........|====|
Type:   MAPPED  HOLE   MAPPED
```

### Case 3: Fallocate (preallocation)

```
File: |~~~~~~~~~~~~~|
Type:   IOMAP_UNWRITTEN
```

### Case 4: Buffered Write Delayed Allocation

```
File: |^^^^^^|
Type:   IOMAP_DELALLOC
```

---

## Buffered Write Path Walkthrough

### Scenario

- App issues:

```c
pwrite(fd, user_buf, 128KB, 4MB);
```

- Filesystem: XFS (or ext4 with iomap)
- Assume offset 4MB was previously a hole.

---

### The Full Path:

```
1️⃣ vfs_write()
  └ file_operations->write_iter()
     └ xfs_file_write_iter()
        └ xfs_file_buffered_write()
           └ iomap_file_buffered_write()

2️⃣ iomap_iter()
  └ Calls iomap_begin()
     └ Allocates extents for hole → returns IOMAP_MAPPED

3️⃣ iomap_write_iter()
  └ Calls iomap_write_begin()

4️⃣ iomap_write_begin()
  └ Calls __iomap_get_folio()
     └ Allocates folio from page cache via __filemap_get_folio()

5️⃣ __iomap_write_begin()
  └ Prepares folio:
      - Zeroes unwritten portions
      - Reads existing blocks if necessary
      - For full overwrite → fast path

6️⃣ Copy data:
  └ copy_folio_from_iter_atomic()

7️⃣ iomap_write_end()
  └ Marks folio dirty
  └ Updates inode size if extended

8️⃣ Write returns to user successfully

9️⃣ Background writeback flushes dirty folios via iomap_writepages()
```

---

## Page Cache Layout

```
inode->i_mapping  (address_space --> XArray based pagecache)

Index 0  -->  Folio (offset 0 - 4K or 0 - N KB for large folio)
Index 1  -->  Folio (offset 4K - 8K)
Index 2  -->  Folio (offset 8K - 12K)
...
Index N  -->  Folio (offset N*PAGE_SIZE ...)
```

> XArray only has entries for folios that actually exist in DRAM (sparse structure).

---

## Buffered I/O vs Direct I/O Allocation Differences

| Feature            | Buffered I/O        | Direct I/O      |
| ------------------ | ------------------- | --------------- |
| Allocation Timing  | At `iomap_begin()`  | At `iomap_begin()` (but marked unwritten) |
| Extent Type        | IOMAP_MAPPED        | IOMAP_UNWRITTEN |
| Data Placement     | Page Cache (DRAM)   | Direct to disk  |
| Consistency Model  | Relaxed (metadata safe) | Strict (data safe) |
| Writeback          | Async (delayed flush)| Immediate (blocking) |
| Crash Exposure     | Stale data possible | No garbage exposed |

---

## Why `iomap_write_begin()` Exists

- Allocates folio.
- Validates mapping state.
- Ensures mapping hasn't changed (racing CoW cases).
- Returns locked folio for write.

---

## Why `__iomap_write_begin()` Exists

- Prepares folio memory before copy:

```
- Zero unwritten regions (holes, delalloc)
- Read existing data if partial overwrite
- Skip if full folio overwrite
```

- Enables safe partial writes without data corruption.

---

## Why `iomap_write_end()` Exists

- Marks folio dirty (`filemap_dirty_folio()`)
- Updates inode size (`i_size_write()`)
- For large folios → granular dirty tracking (`iomap_set_range_dirty()`)

---

## Full Buffered Write Call Graph

```
vfs_write()
  └ iomap_file_buffered_write()
      └ iomap_iter()
          └ iomap_begin() → extent allocation
      └ iomap_write_iter()
          └ iomap_write_begin()
              └ __iomap_get_folio() → allocates folio
              └ __iomap_write_begin() → prepares folio memory
          └ copy_folio_from_iter_atomic() → copy user data
          └ iomap_write_end() → mark dirty + size update
```

---

## Joanne's Granular Dirty Tracking (for Large Folios)

- For large folios: not every block inside folio may be dirty.
- Joanne’s patches allow:

```
- Per-block dirty bitmap tracking inside folio.
- Writeback can flush only dirty portions.
- Greatly improves writeback efficiency for huge folios.
```

---

## Failure Consistency Model

|                  | Buffered I/O       | Direct I/O        |
|------------------|--------------------|--------------------|
| Metadata safe?   | ✅ Always         | ✅ Always          |
| Data safe?       | ❌ Dirty folios lost if crash | ✅ Unwritten protects data |
| Allocation model | Immediate (iomap_begin) | Staged (iomap_end does conversion) |

---

## Closing Thoughts

Buffered I/O with IOMAP is elegant because:

- Filesystems focus only on extent mapping.
- All folio management, zeroing, dirty tracking, and writeback is generic.
- Clean layering makes new features like large folios and granular dirty tracking much easier to add.

---

## Quick Terminology Recap

| Term         | Meaning                                    |
| ------------ | ------------------------------------------ |
| Extent       | Logical offset → physical disk mapping    |
| Folio        | Memory unit in page cache (may be large)   |
| iomap_begin()| Filesystem returns mapping                 |
| iomap_write_begin() | Allocates + locks folio             |
| __iomap_write_begin() | Prepares folio (zero/read)        |
| iomap_write_end() | Marks dirty, updates file size        |
| Writeback    | Flushes dirty folios to disk asynchronously|
| XArray       | Backing structure for page cache           |
