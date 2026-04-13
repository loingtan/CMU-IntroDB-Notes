# Lecture 19: Timestamp Ordering Concurrency Control

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 10, 2025  
**Readings:** Database System Concepts, Chapter 18.5-18.6  
**Project:** Concurrency Control  
**[📹 Watch Video](https://www.youtube.com/watch?v=risHwKeWbBM&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=19)**

---

## Timestamp-Based Concurrency Control

Instead of locks, use timestamps to order transactions.

### Basic Timestamp Ordering (BTO)

Each transaction gets unique timestamp TS(T) at start.

Each data item X has:
- **W-TS(X)**: Timestamp of last write
- **R-TS(X)**: Timestamp of last read

### Read Rule

```
IF TS(T) < W-TS(X):
    # T is trying to read a value written by a future transaction
    # This would violate serializability
    ABORT and restart T (with new timestamp)
ELSE:
    ALLOW the read
    R-TS(X) = max(R-TS(X), TS(T))
```

### Write Rule

```
IF TS(T) < R-TS(X):
    # T is trying to overwrite a value read by a future transaction
    # This would violate serializability
    ABORT and restart T
ELSE IF TS(T) < W-TS(X):
    # T is trying to overwrite a value written by a future transaction
    # This write is outdated, ignore it (Thomas Write Rule)
    IGNORE the write
ELSE:
    ALLOW the write
    W-TS(X) = TS(T)
```

### Example

```
Initial: W-TS(A) = 0, R-TS(A) = 0

T1 (TS=1): Read(A)   → OK, R-TS(A) = 1
T2 (TS=2): Read(A)   → OK, R-TS(A) = 2
T3 (TS=3): Write(A)  → OK, W-TS(A) = 3
T4 (TS=4): Read(A)   → OK, R-TS(A) = 4

Now if T1 tries to Write(A):
  TS(T1) = 1 < R-TS(A) = 4
  → ABORT T1 (someone read after T1 started)
```

---

## Thomas Write Rule

Skip outdated writes instead of aborting.

```
Normal Write Rule:
  IF TS(T) < W-TS(X):
    ABORT T

Thomas Write Rule:
  IF TS(T) < W-TS(X):
    IGNORE the write (no abort)
```

**Benefit:** Reduces unnecessary aborts.

**Example:**
```
T1 (TS=1): Write(A)  → W-TS(A) = 1
T2 (TS=2): Write(A)  → W-TS(A) = 2

Now T1 tries to Write(A) again:
  TS(T1) = 1 < W-TS(A) = 2
  → IGNORE (outdated, T2 already wrote)
```

---

## Optimistic Concurrency Control (OCC)

### Assumption

Conflicts are rare, so don't check during execution.

### Phases

```
Phase 1: READ
---------
  - Read data into private workspace
  - Write to private workspace
  - No actual database modifications

Phase 2: VALIDATION
------------------
  - Check for conflicts with committed transactions
  - If conflict found: ABORT
  - If no conflict: proceed to write

Phase 3: WRITE
-------------
  - Apply changes to database
  - Commit
```

### Validation

```
Backward Validation:
  Check if any committed transaction T' wrote data T read
  IF yes: ABORT T

Forward Validation:
  Check if any active transaction T' read data T wrote
  IF yes: ABORT T or T'
```

### Example

```
T1 and T2 both read A=100

T1: Read(A), A=A+10, Write(A) [private]
T2: Read(A), A=A+20, Write(A) [private]

T1 validates first:
  - No conflicts (no one committed since T1 started)
  - T1 commits, A=110

T2 validates:
  - T1 wrote A after T2 read it
  - CONFLICT! T2 aborts
```

### Trade-offs

| Aspect | Locking | OCC |
|--------|---------|-----|
| **Overhead during execution** | Lock management | None (private workspace) |
| **Conflict handling** | Wait or deadlock | Abort at validation |
| **Best for** | High contention | Low contention |
| **Read-heavy** | OK | Excellent |
| **Write-heavy** | OK | Many aborts |

---

## Snapshot Isolation

Each transaction sees a snapshot at its start timestamp.

### Properties

```
- Reads never block, never conflict
- Each transaction sees consistent snapshot
- First-Committer-Wins for writes
```

### Write Conflicts

```
T1 starts at TS=1, sees snapshot at 1
T2 starts at TS=2, sees snapshot at 2

Both read X=100
T1 writes X=110
T2 writes X=120

First-Committer-Wins:
  If T1 commits first: T2 aborts (write conflict)
  If T2 commits first: T1 aborts (write conflict)
```

### Write Skew Anomaly

Snapshot isolation allows some non-serializable schedules.

```
Constraint: A + B > 0

Initial: A = 100, B = 100

T1: Read(A)=100, Read(B)=100
    Write(A = -50)  # T1 thinks A+B = 150 > 0

T2: Read(A)=100, Read(B)=100
    Write(B = -50)  # T2 thinks A+B = 150 > 0

Both commit (no write conflict, different items)

Result: A = -50, B = -50, A+B = -100 VIOLATES CONSTRAINT!
```

---

## Serializable Snapshot Isolation (SSI)

Detects potential serialization anomalies.

### Approach

```
Track:
  - rw-conflicts: T1 reads, T2 writes
  - ww-conflicts: T1 writes, T2 writes
  - wr-conflicts: T1 writes, T2 reads

If dangerous structure detected:
  T1 → T2 → T3 (rw and wr edges forming cycle)
  → Abort one transaction
```

### Benefits

- True serializability
- Snapshot isolation performance
- Used in PostgreSQL

---

## Summary

| Technique | Mechanism | Best For |
|-----------|-----------|----------|
| **Basic TO** | Timestamps, abort on conflict | Simple implementation |
| **Thomas Write Rule** | Ignore outdated writes | Fewer aborts |
| **OCC** | Validate at commit | Low contention |
| **Snapshot Isolation** | Consistent snapshot | Read-heavy |
| **SSI** | Detect anomalies | True serializability |

---

## Key Takeaways

1. **Timestamp Ordering**: Alternative to locking
2. **Thomas Write Rule**: Reduces aborts
3. **OCC**: Optimistic, good for low contention
4. **Snapshot Isolation**: Popular, but has anomalies
5. **SSI**: True serializability with SI performance

---

*Next Lecture: Multi-Version Concurrency Control*
