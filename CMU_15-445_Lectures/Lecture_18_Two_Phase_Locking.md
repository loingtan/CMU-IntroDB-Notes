# Lecture 18: Two-Phase Locking Concurrency Control

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 5, 2025  
**Readings:** Database System Concepts, Chapter 18.1-18.3, 18.9  
**Homework:** Transactions!  
**Flash Talk:** Benjamin Wagner (Firebolt)  
**[📹 Watch Video](https://www.youtube.com/watch?v=drStlhNbfHI&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=18)**

---

## Lock-Based Concurrency Control

Locks prevent conflicting operations from executing concurrently.

### Lock Types

| Lock | Symbol | Allows |
|------|--------|--------|
| **Shared** | S | Multiple readers |
| **Exclusive** | X | One writer |

### Lock Compatibility

|  | S | X |
|--|---|---|
| **S** | ✓ | ✗ |
| **X** | ✗ | ✗ |

```
Multiple transactions can hold S lock on same item.
Only one transaction can hold X lock.
X lock incompatible with S or X.
```

---

## Two-Phase Locking (2PL)

### Protocol

1. **Growing Phase**: Acquire locks as needed
2. **Shrinking Phase**: Release locks (after reaching "lock point")

```
T1: Lock(A,X), Write(A), Lock(B,X), Write(B), Unlock(A), Unlock(B)
        |<--- Growing Phase --->|<- Shrinking ->|
```

**Rule:** Once a lock is released, no new locks can be acquired.

### Theorem

**2PL ensures conflict serializability.**

```
Proof sketch:
  - If Ti locks before Tj on conflicting item
  - Then Ti's lock point is before Tj's lock point
  - Precedence graph edge: Ti → Tj
  - No cycles possible (would require lock point before itself)
```

### Example

```
T1: Lock(A,X), Read(A), A=A+10, Write(A), Unlock(A)
    Lock(B,X), Read(B), B=B-10, Write(B), Unlock(B)

T2: Lock(A,S), Read(A), Unlock(A)
    Lock(B,S), Read(B), Unlock(B)

Schedule:
  T1: Lock(A,X)
  T1: Read(A), A=A+10, Write(A)
  T2: Lock(A,S) ← waits for T1
  T1: Unlock(A)
  T2: Lock(A,S), Read(A), Unlock(A)
  T1: Lock(B,X)
  T1: Read(B), B=B-10, Write(B)
  T2: Lock(B,S) ← waits for T1
  T1: Unlock(B)
  T2: Lock(B,S), Read(B), Unlock(B)

Result: Serializable (T1 before T2)
```

---

## Strict Two-Phase Locking (Strict 2PL)

Hold all **exclusive locks** until commit/abort.

```
T1: Lock(A,X), Write(A), Lock(B,X), Write(B), Commit, Unlock(A), Unlock(B)
                                                        ↑
                                              All X locks held until here
```

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| **No cascading aborts** | Uncommitted data never visible |
| **Easier recovery** | No undo needed for uncommitted writes |
| **Widely used** | Most commercial systems |

### Cascading Aborts Problem

```
Without Strict 2PL:
  T1: Lock(A,X), Write(A), Unlock(A)
  T2: Lock(A,S), Read(A) ← reads uncommitted value
  T1: ABORT
  T2: must also ABORT (read dirty data!)

With Strict 2PL:
  T1: Lock(A,X), Write(A)
  T2: Lock(A,S) ← waits (T1 still holds X lock)
  T1: ABORT, Unlock(A)
  T2: Lock(A,S), Read(A) ← reads old value
  T2: can commit safely
```

---

## Lock Granularity

### Hierarchy

```
Database
    └── Table
            └── Page
                    └── Row
                            └── Column
```

### Intention Locks

Indicate intent to lock at finer granularity.

| Lock | Symbol | Meaning |
|------|--------|---------|
| **Intent Shared** | IS | Intent to get S lock on child |
| **Intent Exclusive** | IX | Intent to get X lock on child |
| **Shared + Intent Exclusive** | SIX | S on current, IX on children |

### Lock Compatibility (with Intention Locks)

|  | IS | IX | S | SIX | X |
|--|----|----|---|-----|---|
| **IS** | ✓ | ✓ | ✓ | ✓ | ✗ |
| **IX** | ✓ | ✓ | ✗ | ✗ | ✗ |
| **S** | ✓ | ✗ | ✓ | ✗ | ✗ |
| **SIX** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **X** | ✗ | ✗ | ✗ | ✗ | ✗ |

### Lock Acquisition Protocol

```
To get S lock on row:
  1. Get IS lock on database
  2. Get IS lock on table
  3. Get IS lock on page
  4. Get S lock on row

To get X lock on row:
  1. Get IX lock on database
  2. Get IX lock on table
  3. Get IX lock on page
  4. Get X lock on row
```

---

## Deadlock Handling

### Deadlock Detection

Build wait-for graph, look for cycles.

```
T1 waits for T2 (T1 → T2)
T2 waits for T1 (T2 → T1)

Cycle detected! Deadlock.

Choose victim to abort (based on age, progress, etc.)
```

### Deadlock Prevention

#### Wait-Die (Non-preemptive)

```
Older transaction waits for younger.
Younger transaction dies (aborts) if it would wait for older.

T1 (older) wants lock held by T2 (younger):
  T1 waits

T2 (younger) wants lock held by T1 (older):
  T2 dies (aborts and restarts)
```

#### Wound-Wait (Preemptive)

```
Older transaction wounds (aborts) younger.
Younger transaction waits for older.

T1 (older) wants lock held by T2 (younger):
  T2 is wounded (aborts), T1 gets lock

T2 (younger) wants lock held by T1 (older):
  T2 waits
```

---

## Lock Escalation

When too many row locks held:

```
T1 holds 10,000 row locks on table R

Lock escalation:
  - Convert to single table lock on R
  - Reduces lock overhead
  - May reduce concurrency
```

**Thresholds:**
- Number of locks exceeds limit
- Memory for locks exhausted

---

## Summary

| Concept | Description |
|---------|-------------|
| **2PL** | Acquire locks in growing phase, release in shrinking |
| **Strict 2PL** | Hold X locks until commit |
| **Intention Locks** | IS, IX, SIX for hierarchical locking |
| **Deadlock Detection** | Wait-for graph, abort victim |
| **Deadlock Prevention** | Wait-Die or Wound-Wait |

---

## Key Takeaways

1. **2PL Guarantees Serializability**: Simple and effective
2. **Strict 2PL Prevents Cascading Aborts**: Most systems use this
3. **Intention Locks Enable Hierarchy**: Fine-grained locking
4. **Deadlocks are Inevitable**: Detect or prevent
5. **Lock Escalation Balances Overhead**: Trade granularity for efficiency

---

*Next Lecture: Timestamp Ordering Concurrency Control*
