# Lecture 17: Concurrency Control Theory

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 3, 2025  
**Readings:** Database System Concepts, Chapter 18  
**[📹 Watch Video](https://www.youtube.com/watch?v=tMFAgvDViAI&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=17)**

---

## ACID Properties

Transactions must satisfy four key properties.

### Atomicity

All operations complete, or none do.

```
Transfer $100 from A to B:
  1. Debit A by $100
  2. Credit B by $100
  
If step 2 fails, must undo step 1.
```

**Achieved via:** Logging (undo/redo)

### Consistency

Database remains in a valid state.

```
Constraints:
  - Account balance >= 0
  - Total money conserved
  
Transaction must maintain all constraints.
```

**Achieved via:** Atomicity + Isolation + Constraints

### Isolation

Concurrent transactions don't interfere.

```
T1: Read A, A = A + 10, Write A
T2: Read A, A = A + 20, Write A

Without isolation:
  T1 reads A=100
  T2 reads A=100
  T1 writes A=110
  T2 writes A=120  ← Lost T1's update!

With isolation:
  Result is either 110 then 130, or 120 then 130
```

**Achieved via:** Concurrency control

### Durability

Committed changes survive crashes.

```
Transaction commits
  ↓
Power failure
  ↓
After recovery: changes still present
```

**Achieved via:** Logging and checkpoints

---

## Transaction Model

### Transaction States

```
        Begin
          |
          v
    +-----------+
    |  Active   | ← Executing operations
    +-----------+
          |
    +-----+-----+
    |           |
    v           v
Partially   Committed
 Committed      |
    |           |
    v           v
   Failed    Committed ← Changes permanent
    |           |
    +-----+-----+
          |
          v
       Aborted ← Changes undone
```

---

## Schedules and Serializability

### Schedule

An interleaving of operations from multiple transactions.

```
T1: R(A), W(A), R(B), W(B)
T2: R(A), W(A), R(B), W(B)

Serial Schedule 1 (T1 then T2):
  R1(A), W1(A), R1(B), W1(B), R2(A), W2(A), R2(B), W2(B)

Serial Schedule 2 (T2 then T1):
  R2(A), W2(A), R2(B), W2(B), R1(A), W1(A), R1(B), W1(B)

Interleaved Schedule:
  R1(A), R2(A), W1(A), W2(A), R1(B), R2(B), W1(B), W2(B)
```

### Serial Schedule

Transactions execute one after another (no interleaving).

```
T1 completes entirely, then T2 starts.
```

**Guaranteed correct** but poor concurrency.

### Serializable Schedule

Equivalent to some serial schedule.

```
Schedule S is serializable if:
  S produces same result as some serial schedule
```

**Goal:** Allow concurrency while maintaining correctness.

---

## Conflict Operations

Two operations conflict if:
1. They are from different transactions
2. They operate on the same data item
3. At least one is a write

### Conflict Types

| Conflict | Operations | Problem |
|----------|-----------|---------|
| **Read-Write (RW)** | Read after write | Unrepeatable read |
| **Write-Read (WR)** | Write after read | Dirty read |
| **Write-Write (WW)** | Write after write | Lost update |

### Examples

```
Dirty Read (WR conflict):
  T1: W(A), ABORT
  T2: R(A) ← reads uncommitted value!
      
Unrepeatable Read (RW conflict):
  T1: R(A)
  T2: W(A), COMMIT
  T1: R(A) ← different value!
  
Lost Update (WW conflict):
  T1: R(A), W(A)
  T2: R(A), W(A) ← overwrites T1!
```

---

## Conflict Serializability

### Precedence Graph

- **Nodes:** Transactions
- **Edge Ti → Tj:** Ti has conflicting operation before Tj

```
Schedule: R1(A), W2(A), R1(B), W2(B)

Conflicts:
  R1(A) before W2(A): T1 → T2
  R1(B) before W2(B): T1 → T2

Precedence Graph: T1 → T2 (no cycle)

Result: Serializable (equivalent to T1 then T2)
```

```
Schedule: W1(A), W2(A), W2(B), W1(B)

Conflicts:
  W1(A) before W2(A): T1 → T2
  W2(B) before W1(B): T2 → T1

Precedence Graph: T1 ↔ T2 (cycle!)

Result: Not conflict serializable
```

### Theorem

A schedule is **conflict serializable** if and only if its precedence graph has **no cycles**.

---

## Isolation Levels

SQL defines four isolation levels with different trade-offs.

| Level | Dirty Read | Non-Repeatable | Phantom |
|-------|-----------|----------------|---------|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Not Possible | Possible | Possible |
| REPEATABLE READ | Not Possible | Not Possible | Possible |
| SERIALIZABLE | Not Possible | Not Possible | Not Possible |

### Read Uncommitted

```
T1: W(A), (not committed)
T2: R(A) ← reads uncommitted value!

Fastest but least consistent.
Rarely used in practice.
```

### Read Committed

```
T1: W(A), COMMIT
T2: R(A) ← reads committed value

May see different values on re-read.
Default in Oracle, PostgreSQL.
```

### Repeatable Read

```
T1: R(A) ← value = 10
T2: W(A), COMMIT
T1: R(A) ← still sees 10 (snapshot!)

May see phantom rows (new inserts).
Default in MySQL InnoDB.
```

### Serializable

```
Complete isolation, as if serial execution.

May use:
  - Locking
  - Optimistic validation
  - Snapshot isolation with detection

Safest but potentially slowest.
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **ACID** | Atomicity, Consistency, Isolation, Durability |
| **Conflict** | Operations that can't be reordered |
| **Precedence Graph** | Detects serializability |
| **Serializable** | Equivalent to serial schedule |
| **Isolation Levels** | Trade consistency for performance |

---

## Key Takeaways

1. **ACID is Fundamental**: Guarantees transaction correctness
2. **Conflicts Determine Ordering**: RW, WR, WW conflicts
3. **Precedence Graph Detects Serializability**: No cycles = serializable
4. **Isolation Levels Trade Off**: More isolation = less concurrency
5. **Serializable is Safest**: But may impact performance

---

*Next Lecture: Two-Phase Locking Concurrency Control*
