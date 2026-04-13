# Lecture 23: Distributed Database Systems I

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** November 24, 2025  
**Readings:** Database System Concepts, Chapter 20.4-20.5, 21, 23.1-23.4  
**Homework:** Recovery  
**[📹 Watch Video](https://www.youtube.com/watch?v=IFLQBWY6dlE&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=23)**

---

## Distributed Database Concepts

### Why Distributed?

| Reason | Explanation |
|--------|-------------|
| **Scalability** | Handle more data and traffic |
| **Availability** | Survive node failures |
| **Latency** | Place data near users |
| **Disaster Recovery** | Geographic distribution |

### Challenges

| Challenge | Description |
|-----------|-------------|
| **Network latency** | Slower than local access |
| **Network partitions** | Nodes can't communicate |
| **Distributed transactions** | ACID across nodes |
| **Consistency trade-offs** | CAP theorem |

---

## Data Partitioning (Sharding)

### Horizontal Partitioning

Split rows across nodes.

```
Before:
  All data on single node

After:
  Node 1: Rows 1-1000
  Node 2: Rows 1001-2000
  Node 3: Rows 2001-3000
```

### Partitioning Strategies

| Strategy | How It Works |
|----------|--------------|
| **Hash** | hash(key) % N |
| **Range** | Key ranges per node |
| **List** | Specific values per node |
| **Composite** | Combine strategies |

### Hash Partitioning

```
Partition = hash(customer_id) % 4

customer_id=1 → hash(1) % 4 = 1 → Node 1
customer_id=2 → hash(2) % 4 = 2 → Node 2
customer_id=3 → hash(3) % 4 = 3 → Node 3
customer_id=4 → hash(4) % 4 = 0 → Node 0
```

**Pros:** Even distribution  
**Cons:** Range queries hit all nodes

### Range Partitioning

```
Partition by date:
  Node 1: 2020-01-01 to 2020-12-31
  Node 2: 2021-01-01 to 2021-12-31
  Node 3: 2022-01-01 to 2022-12-31
```

**Pros:** Efficient range queries  
**Cons:** Hot spots (recent data)

### Consistent Hashing

Map nodes and keys to a ring.

```
Ring: 0 ----------------------------------------> 2^32-1
         
      Node A          Node B          Node C
        ↓               ↓               ↓
      [key] → next clockwise node

Adding Node D:
  - Only affects keys between C and D
  - Minimal data movement
```

**Benefit:** Adding/removing nodes affects minimal keys.

---

## Distributed Transactions

### Two-Phase Commit (2PC)

```
Coordinator                    Participants
     |                              |
     |------- PREPARE ------------>|
     |<------ YES/NO --------------|
     |                              |
     | (if all YES)                 |
     |------- COMMIT -------------->|
     |<------ ACK -----------------|
     |                              |
     | (if any NO)                  |
     |------- ABORT --------------->|
     |<------ ACK -----------------|
```

**Phase 1 (Voting):**
- Coordinator asks participants to prepare
- Participants vote YES (can commit) or NO (must abort)

**Phase 2 (Decision):**
- If all YES, coordinator sends COMMIT
- If any NO, coordinator sends ABORT
- Participants acknowledge

### 2PC Problems

| Problem | Description |
|---------|-------------|
| **Blocking** | Coordinator failure leaves participants waiting |
| **Slow** | Multiple network round-trips |

### Three-Phase Commit (3PC)

Adds pre-commit phase to reduce blocking.

```
Phase 1: CanCommit?
Phase 2: PreCommit
Phase 3: DoCommit

With timeouts, can make progress without coordinator.
```

**Pros:** Non-blocking  
**Cons:** More complex, more messages

---

## Consensus Protocols

### Paxos

Family of protocols for distributed consensus.

```
Roles:
  - Proposers: Suggest values
  - Acceptors: Vote on values
  - Learners: Learn chosen value

Process:
  1. Proposer sends prepare
  2. Acceptors respond with promise
  3. Proposer sends accept
  4. Acceptors vote
  5. Value chosen when majority accepts
```

**Pros:** Proven correct  
**Cons:** Complex to implement

### Raft

Simpler alternative to Paxos.

```
Roles:
  - Leader: Handles all requests
  - Followers: Replicate leader's log
  - Candidates: Run for leader election

Process:
  1. Leader election (if no leader)
  2. Leader accepts requests
  3. Leader replicates to followers
  4. Commit when majority replicated
```

**Pros:** Easier to understand  
**Cons:** Leader bottleneck

**Systems:** etcd, Consul, TiKV

---

## Summary

| Concept | Description |
|---------|-------------|
| **Partitioning** | Split data across nodes |
| **2PC** | Two-phase commit for distributed transactions |
| **Consensus** | Agree on value across nodes |
| **Raft** | Simpler consensus protocol |

---

## Key Takeaways

1. **Partitioning Enables Scale**: Distribute data and load
2. **Consistent Hashing**: Minimal data movement on changes
3. **2PC is Standard**: But has blocking issues
4. **Consensus is Hard**: Paxos/Raft provide solutions
5. **Trade-offs Everywhere**: Consistency vs. availability

---

*Next Lecture: Distributed Database Systems II*
