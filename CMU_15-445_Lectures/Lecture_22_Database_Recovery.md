# Lecture 22: Database Recovery

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 19, 2025  
**Readings:** Database System Concepts, Chapter 19.1-19.9  
**Flash Talk:** Robert Schulze (ClickHouse)  
**[📹 Watch Video](https://www.youtube.com/watch?v=X2jc4qalNy0&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=22)**

---

## Failure Types

### Transaction Failure

```
Caused by:
  - Logical error (constraint violation)
  - System error (deadlock detection)

Handling:
  - Undo transaction's changes
  - Release locks
  - Notify application
```

### System Crash

```
Caused by:
  - Power failure
  - Software bug
  - Hardware failure

Characteristics:
  - Memory lost
  - Disk intact (assumed)

Handling:
  - Redo committed transactions
  - Undo active transactions
```

### Disk Failure

```
Caused by:
  - Head crash
  - Media degradation

Handling:
  - Restore from backup
  - Apply log to catch up
```

---

## ARIES Recovery Algorithm

**ARIES** = Algorithm for Recovery and Isolation Exploiting Semantics

Most widely used recovery algorithm.

### Key Principles

| Principle | Description |
|-----------|-------------|
| **Write-Ahead Logging** | Log before data |
| **Redo History** | Repeat all actions |
| **Undo Losers** | Remove uncommitted changes |
| **LSN Chaining** | Link log records for each page |

### Data Structures

```
Page LSN: Last log record affecting this page
RecLSN:   Oldest log record needing redo for this page

Transaction Table:
  - Transaction ID
  - Status (active, committed, aborted)
  - Last LSN

Dirty Page Table:
  - Page ID
  - RecLSN (when page became dirty)
```

---

## Recovery Phases

### Phase 1: Analysis

```
Scan log forward from last checkpoint:
  1. Rebuild transaction table
  2. Rebuild dirty page table
  3. Determine which transactions were active at crash

Output:
  - Active transactions (need undo)
  - Committed transactions (need redo)
  - Dirty pages (need redo if not on disk)
```

### Phase 2: Redo

```
Scan log forward from minimum RecLSN:
  For each log record:
    IF page is in dirty page table AND
       pageLSN < log LSN:
      - Redo the operation
      - Update pageLSN

Idempotent: Can redo multiple times safely
```

### Phase 3: Undo

```
Undo uncommitted transactions in reverse order:
  For each active transaction (from Analysis):
    Follow LSN chain backwards
    For each update:
      - Undo the operation
      - Write CLR (Compensation Log Record)
    Until transaction start

CLR: Records that undo was performed (prevents duplicate undo)
```

---

## Example Recovery

```
Log:
  LSN 1: T1 starts
  LSN 2: T1 updates Page A
  LSN 3: T2 starts
  LSN 4: T2 updates Page B
  LSN 5: T1 commits
  LSN 6: T2 updates Page C
  LSN 7: CHECKPOINT
  LSN 8: T3 starts
  LSN 9: T3 updates Page A
  LSN 10: T3 updates Page D
  CRASH!

Analysis:
  - T1: committed (needs redo)
  - T2: active (needs undo)
  - T3: active (needs undo)
  - Dirty pages: A, B, C, D

Redo (from LSN 7):
  - LSN 9: Redo T3 update to Page A
  - LSN 10: Redo T3 update to Page D

Undo (reverse order):
  - Undo T3: LSN 10, then LSN 9
  - Undo T2: LSN 6, then LSN 4
```

---

## Checkpoints

### Fuzzy Checkpoint

```
Don't stop processing during checkpoint:
  1. Write checkpoint begin record
  2. Continue logging
  3. Write checkpoint end record with current state

More complex but no pause
```

### Sharp Checkpoint

```
Pause all updates:
  1. Stop accepting new transactions
  2. Wait for active transactions to complete
  3. Flush all dirty pages
  4. Write checkpoint record
  5. Resume processing

Simpler but causes delays
```

---

## Summary

| Phase | Purpose |
|-------|---------|
| **Analysis** | Determine what needs redo/undo |
| **Redo** | Reapply committed changes |
| **Undo** | Remove uncommitted changes |

---

## Key Takeaways

1. **ARIES is Widely Used**: Industry standard recovery
2. **Redo History**: Apply all committed changes
3. **Undo Losers**: Remove uncommitted changes
4. **Checkpoints Reduce Recovery Time**: Don't scan entire log
5. **CLRs Prevent Duplicate Undo**: Idempotent recovery

---

*Next Lecture: Distributed Database Systems I*
