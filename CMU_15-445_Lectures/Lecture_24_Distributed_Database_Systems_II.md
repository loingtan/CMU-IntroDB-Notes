# Lecture 24: Distributed Database Systems II

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** December 1, 2025  
**Readings:** Database System Concepts, Chapter 20.7, 22.9  
**[📹 Watch Video](https://www.youtube.com/watch?v=pQh5fka3FC0&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=24)**

---

## Replication

### Why Replicate?

| Reason | Benefit |
|--------|---------|
| **Availability** | Survive node failures |
| **Read scaling** | Distribute read load |
| **Latency** | Local reads |
| **Durability** | Multiple copies |

### Replication Modes

| Mode | Description |
|------|-------------|
| **Synchronous** | Wait for acknowledgment from replicas |
| **Asynchronous** | Don't wait for replicas |
| **Semi-synchronous** | Wait for at least one replica |

### Synchronous Replication

```
Client → Primary → Replicas → ACK → Client
         ↑            ↑
       Write      All ACKs

Pros: No data loss
Cons: Higher latency
```

### Asynchronous Replication

```
Client → Primary → ACK → Client
              ↓\n           Replicas (async)

Pros: Lower latency
Cons: Potential data loss on failure
```

---

## Replication Topologies

### Single Leader

```
Primary → Replica 1
       → Replica 2
       → Replica 3

Writes: Primary only
Reads: Any node
```

**Pros:** Simple, consistent  
**Cons:** Primary bottleneck

### Multi-Leader

```
Leader 1 ←→ Leader 2 ←→ Leader 3
   ↓           ↓           ↓
Replicas    Replicas    Replicas

Writes: Any leader
Conflict resolution needed
```

**Pros:** Write scaling, geographic distribution  
**Cons:** Conflict resolution complexity

### Leaderless

```
All nodes accept writes
Quorum for consistency

Write: W replicas must acknowledge
Read: R replicas must agree

Consistency when W + R > N
```

**Pros:** High availability  
**Cons:** Complex consistency

**Systems:** Dynamo, Cassandra, Voldemort

---

## CAP Theorem

In a distributed system, you can only guarantee **two of three**:

| Property | Description |
|----------|-------------|
| **Consistency** | All nodes see same data |
| **Availability** | Every request gets a response |
| **Partition Tolerance** | Works despite network partitions |

### CAP Trade-offs

```
Network Partition Occurs:

Option 1: Choose Consistency
  - Refuse writes until partition heals
  - System unavailable during partition
  
Option 2: Choose Availability
  - Accept writes on both sides
  - May have inconsistent data
```

### System Categories

| Category | Systems | Guarantees |
|----------|---------|------------|
| **CA** | Traditional single-node DB | Consistency + Availability |
| **CP** | HBase, MongoDB, etcd | Consistency + Partition Tolerance |
| **AP** | Cassandra, DynamoDB | Availability + Partition Tolerance |

---

## Distributed Query Processing

### Query Shipping

```
Send query to data nodes:

Client → Coordinator
            ↓
        Node 1: Execute partial query
        Node 2: Execute partial query
        Node 3: Execute partial query
            ↓
        Coordinator: Combine results
```

**Best for:** Simple queries, data locality

### Data Shipping

```
Bring data to coordinator:

Client → Coordinator
            ↓
        Fetch data from nodes
            ↓
        Execute centrally
```

**Best for:** Complex queries, small result sets

### Hybrid Approach

```
Push down filters and aggregations:

SELECT dept, SUM(salary) 
FROM employees 
WHERE country = 'USA'
GROUP BY dept

Push down:
  - Filter: country = 'USA'
  - Aggregate: SUM(salary) per dept

Ship intermediate results to coordinator
Final aggregation at coordinator
```

---

## Summary

| Topic | Key Idea |
|-------|----------|
| **Replication** | Multiple copies for availability |
| **Synchronous** | No data loss, higher latency |
| **Asynchronous** | Lower latency, potential loss |
| **CAP** | Can't have all three |
| **Query Processing** | Ship query or data |

---

## Key Takeaways

1. **Replication Improves Availability**: But adds complexity
2. **Synchronous vs. Asynchronous**: Latency vs. durability trade-off
3. **CAP is Fundamental**: Must choose in partitions
4. **Most Systems are CP or AP**: Not truly CA
5. **Query Processing is Distributed**: Ship code or data

---

*Next Lecture: Final Review + Systems Potpourri*
