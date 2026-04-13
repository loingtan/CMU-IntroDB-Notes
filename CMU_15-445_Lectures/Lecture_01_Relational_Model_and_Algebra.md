# Lecture 01: Relational Model & Algebra

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** August 25, 2025  
**Readings:** Database System Concepts, Chapters 1-2  
**Project:** C++ Primer  
**[📹 Watch Video](https://www.youtube.com/watch?v=7NPIENPr-zk&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=1)**

---

## Introduction to Database Systems

A **Database Management System (DBMS)** is software that allows applications to store and analyze information in a database. The earliest DBMSs were developed in the 1960s to manage complex data and provide **data independence**—protecting applications from changes in how data is stored and managed.

### Key Goals of Database Systems

1. **Data Independence**: Applications shouldn't need to know how data is physically stored
2. **Efficient Access**: Fast retrieval and modification of data
3. **Data Integrity**: Ensure data correctness through constraints
4. **Security**: Control who can access what data
5. **Concurrency**: Allow multiple users to access data simultaneously
6. **Crash Recovery**: Survive system failures without data loss

---

## The Relational Model

Proposed by **Edgar Codd** in 1969, the relational model revolutionized database systems by providing:

- **Simple, uniform data representation**: All data stored in relations (tables)
- **High-level abstraction**: Users describe *what* they want, not *how* to get it
- **Physical data independence**: The underlying storage can change without affecting applications

### Why the Relational Model Won

Before relational databases, systems used hierarchical (IMS) or network (CODASYL) models. These required programmers to navigate complex pointer structures. The relational model's declarative approach (SQL) separated the logical view from physical implementation, making databases more accessible.

---

## Core Concepts

### Relation (Table)

A **relation** is a set of tuples (rows) with the same schema:

| sid | name  | login       | age | gpa |
|-----|-------|-------------|-----|-----|
| 1   | Alice | alice@cmu   | 20  | 3.8 |
| 2   | Bob   | bob@cmu     | 21  | 3.2 |
| 3   | Carol | carol@cmu   | 19  | 3.9 |

**Key Properties:**
- Relations are **unordered sets** (no inherent row order)
- Each tuple represents a single entity or relationship instance
- All tuples in a relation have the same attributes (columns)

### Schema vs. Instance

**Schema**: The logical structure of the database (blueprint)
```
Student(sid: INT, name: VARCHAR(50), login: VARCHAR(50), age: INT, gpa: FLOAT)
```

**Instance**: The actual data at a specific point in time (the table above)

### Keys

**Primary Key**: Uniquely identifies each tuple in a relation
- Must be unique and not null
- Example: `sid` in the Student table

**Foreign Key**: References the primary key of another relation, enabling relationships
```sql
Enrolled(sid: INT REFERENCES Student, cid: INT REFERENCES Course, grade: CHAR)
```

**Candidate Key**: Minimal set of attributes that uniquely identify a tuple
- A relation can have multiple candidate keys
- One is chosen as the primary key

**Super Key**: Any set of attributes that uniquely identifies a tuple
- Contains a candidate key (possibly plus additional attributes)

---

## Relational Algebra

Relational algebra is a **formal language** for manipulating relations. It provides the theoretical foundation for SQL. Every SQL query can be translated into a sequence of relational algebra operations.

### Fundamental Operations

| Operation | Symbol | Description | Example |
|-----------|--------|-------------|---------|
| **Selection** | σ (sigma) | Filters rows based on a predicate | `σ_{gpa>3.5}(Student)` |
| **Projection** | π (pi) | Selects specific columns | `π_{name,gpa}(Student)` |
| **Union** | ∪ | Combines two union-compatible relations | `R ∪ S` |
| **Intersection** | ∩ | Returns common tuples | `R ∩ S` |
| **Difference** | − | Returns tuples in first but not second | `R − S` |
| **Product** | × | Cartesian product of two relations | `R × S` |
| **Join** | ⋈ | Combines relations based on related columns | `R ⋈_{R.id=S.id} S` |
| **Rename** | ρ (rho) | Renames relations or attributes | `ρ_{S(sid,name)}(Student)` |

### Selection (σ)

Selects tuples that satisfy a predicate:

```
σ_{gpa>3.5}(Student)
```

Result:
| sid | name  | login       | age | gpa |
|-----|-------|-------------|-----|-----|
| 1   | Alice | alice@cmu   | 20  | 3.8 |
| 3   | Carol | carol@cmu   | 19  | 3.9 |

**Properties:**
- Commutative: `σ_{p}(σ_{q}(R)) = σ_{q}(σ_{p}(R))`
- Can combine predicates: `σ_{p∧q}(R) = σ_{p}(σ_{q}(R))`

### Projection (π)

Selects specific columns, removing duplicates:

```
π_{name,gpa}(Student)
```

Result:
| name  | gpa |
|-------|-----|
| Alice | 3.8 |
| Bob   | 3.2 |
| Carol | 3.9 |

**Properties:**
- Not commutative with itself
- Eliminates duplicates (sets, not bags)

### Union, Intersection, Difference

These require **union-compatible** relations (same number of attributes with compatible types).

**Union** (`R ∪ S`): All tuples in R or S (or both)

**Intersection** (`R ∩ S`): Tuples in both R and S

**Difference** (`R − S`): Tuples in R but not in S

### Cartesian Product (×)

Combines every tuple from R with every tuple from S:

If R has n tuples and S has m tuples, R × S has n × m tuples.

```
Student × Course
```

Result combines every student with every course (usually not useful alone).

### Join Operations

**Inner Join (⋈)**:
```
Student ⋈_{Student.sid=Enrolled.sid} Enrolled
```

Only includes tuples where the join condition matches.

**Outer Joins**:
- **Left Outer Join (⋊)**: All tuples from left relation, matching from right
- **Right Outer Join (⋉)**: All tuples from right relation, matching from left
- **Full Outer Join (⋈⁺)**: All tuples from both relations

Example Left Outer Join:
```
Student ⋊_{Student.sid=Enrolled.sid} Enrolled
```

Includes all students, even those not enrolled in any course (NULL for course attributes).

---

## Relational Algebra Equivalences

Understanding these equivalences helps in query optimization:

1. **Selection Pushdown**: Apply selections as early as possible
   ```
   σ_{p}(R ⋈ S) = σ_{p}(R) ⋈ S  (if p only involves R)
   ```

2. **Projection Pushdown**: Project early to reduce data size
   ```
   π_{A}(R ⋈ S) = π_{A}(π_{A∪B}(R) ⋈ π_{B∪C}(S))
   ```

3. **Join Associativity**: Order of joins can be changed
   ```
   (R ⋈ S) ⋈ T = R ⋈ (S ⋈ T)
   ```

4. **Join Commutativity**: Order of relations can be swapped
   ```
   R ⋈ S = S ⋈ R
   ```

---

## From Relational Algebra to SQL

Every SQL query has an equivalent relational algebra expression:

```sql
SELECT name, gpa 
FROM Student 
WHERE gpa > 3.5
```

Equivalent to:
```
π_{name,gpa}(σ_{gpa>3.5}(Student))
```

```sql
SELECT S.name, C.title 
FROM Student S 
JOIN Enrolled E ON S.sid = E.sid 
JOIN Course C ON E.cid = C.cid
```

Equivalent to:
```
π_{name,title}(Student ⋈ Enrolled ⋈ Course)
```

---

## Summary

| Concept | Description |
|---------|-------------|
| **Relation** | A table; set of tuples with same schema |
| **Schema** | Structure: attribute names and types |
| **Instance** | Actual data at a point in time |
| **Primary Key** | Uniquely identifies each tuple |
| **Foreign Key** | References another relation's primary key |
| **Selection** | Filter rows (σ) |
| **Projection** | Select columns (π) |
| **Join** | Combine relations based on condition (⋈) |

---

## Key Takeaways

1. **Relational Model**: Simple, powerful, declarative approach to data management
2. **Data Independence**: Physical storage can change without affecting applications
3. **Relational Algebra**: Formal foundation for database query languages
4. **Keys**: Enable relationships and ensure data integrity
5. **Operations**: Selection, projection, join are the building blocks of queries

---

*Next Lecture: Modern SQL - The practical language built on relational algebra foundations*
