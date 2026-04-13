# Lecture 20: Multi-Version Concurrency Control

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 12, 2025  
**Readings:** Database System Concepts, Chapter 18.7-18.8  
**[📹 Watch Video](https://www.youtube.com/watch?v=tUFha9-DuSk&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=20)**

---

## Multi-Version Concurrency Control (MVCC)

Instead of overwriting data, create new versions. Readers see consistent snapshots without blocking.

### Version Chain

```
Version 3 (TS=30): value = C ← Most recent
Version 2 (TS=20): value = B
Version 1 (TS=10): value = A  ← Oldest
```

### Key Idea

```
Writers create new versions
Readers see appropriate version based on timestamp
No blocking between readers and writers!
```

---

## Version Storage

### Approach 1: Append-Only Storage

All versions in same table space.

```
Table Page:
+----------------------------------+
| [Header]                         |
+----------------------------------+
| [Version 3: TS=30, value=C]      |
+----------------------------------+
| [Version 2: TS=20, value=B]      |
+----------------------------------+
| [Version 1: TS=10, value=A]      |
+----------------------------------+

Index points to version chain head
```

**Pros:** Simple, all data together  
**Cons:** Table grows unbounded

### Approach 2: Time-Travel Storage

Old versions in separate table.

```
Main Table:          Time-Travel Table:
+--------+           +--------+--------+
| ID | Value |       | ID | TS | Value |
+--------+           +--------+--------+
| 1  | C     |       | 1  | 20 | B     |
+--------+           | 1  | 10 | A     |
                     +--------+--------+

Main table has current version
Time-travel has old versions
```

**Pros:** Fast access to current data  
**Cons:** Extra indirection for old versions

### Approach 3: Delta Storage

Store only changes (deltas) for old versions.

```
Version 1: [A, B, C, D]  ← Full version
Version 2: [Δ: C→X]      ← Only changed field
Version 3: [Δ: A→Y]      ← Only changed field

To reconstruct version 3:
  Start with version 1
  Apply delta from version 2 (if needed)
  Apply delta from version 3
```

**Pros:** Space efficient  
**Cons:** Slower to read old versions

---

## Garbage Collection

Old versions must eventually be removed.

### Tuple-Level GC

```
Background vacuum process:
  1. Scan for old versions
  2. Check if visible to any active transaction
  3. If not visible: remove
```

### Transaction-Level GC

```
Track which transactions created versions:
  - When transaction commits, note its timestamp
  - Remove all versions when no transaction can see them
```

### Vacuum Strategies

| Strategy | How It Works |
|----------|--------------|
| **Background Vacuum** | Periodic scan and cleanup |
| **Cooperative Cleaning** | Transactions clean as they go |
| **Index-Guided** | Use index to find dead tuples |

---

## Index Management with MVCC

### Primary Key Indexes

Point to version chain head.

```
PK Index: [key=1] → Version Chain Head

Easy to find latest version
```

### Secondary Indexes

| Approach | How It Works |
|----------|--------------|
| **Logical Pointers** | Store primary key, lookup in PK index |
| **Physical Pointers** | Point to specific version (needs updating) |
| **Indirect Pointers** | Point to mapping entry (indirection layer) |

### Index Entry Example

```
Secondary index on "name":

Logical: ["Alice"] → PK=1 → lookup PK index
Physical: ["Alice"] → (Page 5, Slot 3) → specific version
```

---

## MVCC Implementation Examples

### PostgreSQL

```
Tuple format:
  xmin: Transaction ID that created version
  xmax: Transaction ID that deleted version (0 = alive)
  data: Actual tuple data

Visibility check:
  - xmin committed and < snapshot.xmin
  - xmax is 0 or not visible to snapshot

Vacuum: Background process removes dead tuples
```

### MySQL InnoDB

```
Undo log stores old versions:
  - Roll segments manage version chains
  - Purge thread removes old versions
  - Index records point to latest version

Visibility: Use undo log to reconstruct old versions
```

### SQL Server

```
Version store in tempdb:
  - Snapshot isolation uses row versioning
  - Background cleanup process
  - Version store size must be managed
```

---

## MVCC Trade-offs

| Aspect | Advantage | Disadvantage |
|--------|-----------|--------------|
| **Reads** | Never block | May need version chain traversal |
| **Writes** | Don't block readers | Create versions (overhead) |
| **Space** | - | Versions consume space |
| **GC** | - | Cleanup overhead |

---

## Summary

| Concept | Description |
|---------|-------------|
| **MVCC** | Multiple versions, readers see snapshots |
| **Version Storage** | Append-only, time-travel, or delta |
| **Garbage Collection** | Remove invisible versions |
| **Index Management** | Logical or physical pointers |

---

## Key Takeaways

1. **MVCC Enables Non-Blocking Reads**: Readers don't wait for writers
2. **Version Storage Trade-offs**: Space vs. access speed
3. **Garbage Collection is Essential**: Prevent unbounded growth
4. **Indexes Need Special Handling**: Point to appropriate versions
5. **Widely Used**: PostgreSQL, MySQL, SQL Server, Oracle

---

*Next Lecture: Database Logging*
