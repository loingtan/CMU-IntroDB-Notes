# Lecture 21: Database Logging

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 17, 2025  
**Readings:** Database System Concepts, Chapter 19.1-19.8  
**[📹 Watch Video](https://www.youtube.com/watch?v=CedEy54pe3g&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=21)**

---

## Write-Ahead Logging (WAL)

The fundamental principle of database recovery: **log before data**.

### WAL Rule

```
Before modifying any data page on disk,
the corresponding log record must be on stable storage.
```

### Why WAL Works

| Property | Explanation |
|----------|-------------|
| **Log is sequential** | Fast to write (append-only) |
| **Log records are idempotent** | Can redo multiple times safely |
| **Enables undo and redo** | Both directions of recovery |

---

## Log Records

### Structure

```
+------------------+
| Log Sequence #   |  ← Unique identifier (LSN)
+------------------+
| Transaction ID   |  ← Which transaction
+------------------+
| Type             |  ← UPDATE, COMMIT, ABORT, etc.
+------------------+
| Page ID          |  ← Which page affected
+------------------+
| Offset           |  ← Location in page
+------------------+
| Before Image     |  ← Old value (for undo)
+------------------+
| After Image      |  ← New value (for redo)
+------------------+
| Checksum         |  ← Data integrity
+------------------+
```

### Log Types

| Type | Purpose |
|------|---------|
| **UPDATE** | Record data modification |
| **COMMIT** | Transaction committed |
| **ABORT** | Transaction aborted |
| **CHECKPOINT** | Recovery point marker |
| **CLR** | Compensation Log Record (for undo) |

### Example

```
Transaction T1 updates A from 100 to 110:

Log Record:
  LSN: 1000
  TID: T1
  Type: UPDATE
  Page: Page_A
  Offset: 0
  Before: 100
  After: 110
```

---

## Logging Schemes

### Physical Logging

Log actual byte changes to pages.

```
Page 5, offset 100:
  Before: [bytes at that location]
  After:  [new bytes]
```

**Pros:** Simple, exact  
**Cons:** Large log records

### Logical Logging

Log the operation.

```
Operation: "INSERT INTO table VALUES (...)"
```

**Pros:** Small log records  
**Cons:** Harder to redo/undo (must re-execute)

### Physiological Logging

Hybrid approach (most common).

```
Page location + slot number + changes

Example:
  Page 5, slot 3:
    Field 1: old=10, new=20
    Field 2: old="Alice", new="Bob"
```

**Pros:** Balance of size and simplicity  
**Cons:** Moderate complexity

---

## Log Buffer Management

### Group Commit

Batch multiple commits together.

```
T1 commits → buffer log record
T2 commits → buffer log record
T3 commits → buffer log record

Flush log buffer (all 3 commits written together!)

Benefit: Fewer I/O operations
```

### Log Flush Policies

| Policy | Description |
|--------|-------------|
| **Force at Commit** | Wait for log to disk (durability) |
| **No-Force** | Allow asynchronous log writes (risky) |

---

## Log Sequence Numbers (LSN)

### LSN Chain

```
Log records form a chain:
  LSN 1 → LSN 2 → LSN 3 → LSN 4 → ...

Each page tracks:
  pageLSN: LSN of most recent update to this page

During recovery:
  - If pageLSN < log LSN: need to redo
  - If pageLSN >= log LSN: already applied
```

### LSN Uses

| Use | Description |
|-----|-------------|
| **Ordering** | Determine log record order |
| **Recovery** | Know which updates to redo |
| **Page tracking** | Link pages to log records |

---

## Summary

| Concept | Description |
|---------|-------------|
| **WAL** | Log before data |
| **Log Record** | Before image, after image, metadata |
| **Physiological Logging** | Page + slot + changes |
| **Group Commit** | Batch commits for efficiency |
| **LSN** | Unique log record identifier |

---

## Key Takeaways

1. **WAL is Fundamental**: Ensures durability and recoverability
2. **Log Records are Idempotent**: Can apply multiple times
3. **Physiological Logging**: Best balance in practice
4. **Group Commit Improves Throughput**: Batch log writes
5. **LSN Enables Recovery**: Track what needs redo

---

*Next Lecture: Database Recovery*
