# Lecture 02: Modern SQL

**CMU 15-445/645: Introduction to Database Systems (Fall 2025)**  
**Date:** August 27, 2025  
**Readings:** Database System Concepts, Chapters 3-5  
**Homework:** SQL  
**Flash Talk:** Drew Banin (dbt Labs)  
**[📹 Watch Video](https://www.youtube.com/watch?v=O5gU9NQjCAs&list=PLSE8ODhjZXjYMAgsGH-GtY5rJYZ6zjsd5&index=2)**

---

## SQL: Structured Query Language

SQL is the **standard language** for interacting with relational databases. Unlike relational algebra which is procedural, SQL is **declarative**—you specify what you want, and the DBMS figures out how to get it.

### SQL Standards

- **SQL-86**: First standard
- **SQL-92**: Major revision, widely supported
- **SQL:1999**: Added recursive queries, triggers, object features
- **SQL:2003**: Added window functions, XML support
- **SQL:2011**: Temporal databases
- **SQL:2016**: JSON support, polymorphic table functions

---

## Data Definition Language (DDL)

DDL statements define and modify database schemas.

### Creating Tables

```sql
CREATE TABLE student (
    sid INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    login VARCHAR(50) UNIQUE,
    age INT CHECK (age >= 0 AND age <= 120),
    gpa FLOAT DEFAULT 0.0,
    dept_id INT REFERENCES department(did)
);
```

**Constraints:**
- `PRIMARY KEY`: Unique identifier, not null
- `NOT NULL`: Column must have a value
- `UNIQUE`: No duplicate values allowed
- `DEFAULT`: Default value if not specified
- `CHECK`: Domain constraint
- `REFERENCES`: Foreign key constraint

### Modifying Schema

```sql
-- Add a column
ALTER TABLE student ADD COLUMN email VARCHAR(100);

-- Modify a column
ALTER TABLE student ALTER COLUMN gpa TYPE DECIMAL(3,2);

-- Add a constraint
ALTER TABLE student ADD CONSTRAINT chk_gpa CHECK (gpa >= 0 AND gpa <= 4.0);

-- Drop a column
ALTER TABLE student DROP COLUMN age;

-- Drop a table
DROP TABLE student;

-- Rename a table
ALTER TABLE student RENAME TO undergrad;
```

---

## Data Manipulation Language (DML)

DML statements query and modify data.

### Inserting Data

```sql
-- Insert single row
INSERT INTO student VALUES (1, 'Alice', 'alice@cmu', 20, 3.8, 1);

-- Insert with explicit columns
INSERT INTO student (sid, name, login) VALUES (2, 'Bob', 'bob@cmu');

-- Insert multiple rows
INSERT INTO student VALUES 
    (3, 'Carol', 'carol@cmu', 19, 3.9, 2),
    (4, 'Dave', 'dave@cmu', 21, 3.2, 1);

-- Insert from query
INSERT INTO honor_students (sid, name)
SELECT sid, name FROM student WHERE gpa >= 3.5;
```

### Updating Data

```sql
-- Update single row
UPDATE student SET gpa = 3.9 WHERE sid = 1;

-- Update multiple columns
UPDATE student 
SET gpa = gpa + 0.1, age = age + 1 
WHERE sid = 2;

-- Update based on another table
UPDATE student 
SET gpa = gpa * 1.05 
WHERE dept_id IN (SELECT did FROM department WHERE name = 'CS');
```

### Deleting Data

```sql
-- Delete specific rows
DELETE FROM student WHERE gpa < 2.0;

-- Delete all rows (but keep table)
DELETE FROM student;

-- Delete with subquery
DELETE FROM student 
WHERE sid NOT IN (SELECT sid FROM enrolled);
```

---

## Basic Querying

### SELECT Statement Structure

```sql
SELECT [DISTINCT] column_list
FROM table_list
[WHERE condition]
[GROUP BY grouping_columns]
[HAVING group_condition]
[ORDER BY sort_columns]
[LIMIT count];
```

### Simple Queries

```sql
-- Select all columns
SELECT * FROM student;

-- Select specific columns
SELECT name, gpa FROM student;

-- Eliminate duplicates
SELECT DISTINCT dept_id FROM student;

-- Rename columns
SELECT name AS student_name, gpa AS grade_point_avg FROM student;
```

### Filtering with WHERE

```sql
-- Comparison operators
SELECT * FROM student WHERE gpa > 3.5;
SELECT * FROM student WHERE age BETWEEN 18 AND 25;
SELECT * FROM student WHERE name LIKE 'A%';  -- Starts with A

-- Logical operators
SELECT * FROM student WHERE gpa > 3.5 AND age < 22;
SELECT * FROM student WHERE dept_id = 1 OR dept_id = 2;
SELECT * FROM student WHERE gpa IS NOT NULL;

-- Set membership
SELECT * FROM student WHERE dept_id IN (1, 2, 3);
SELECT * FROM student WHERE dept_id NOT IN (SELECT did FROM closed_departments);
```

### Sorting with ORDER BY

```sql
-- Ascending order (default)
SELECT * FROM student ORDER BY gpa;

-- Descending order
SELECT * FROM student ORDER BY gpa DESC;

-- Multiple sort keys
SELECT * FROM student ORDER BY dept_id ASC, gpa DESC;

-- Sort by column position
SELECT name, gpa FROM student ORDER BY 2 DESC;
```

---

## Joins

### Inner Join

```sql
-- Explicit join
SELECT s.name, c.title 
FROM student s 
JOIN enrolled e ON s.sid = e.sid 
JOIN course c ON e.cid = c.cid;

-- Old syntax (avoid)
SELECT s.name, c.title 
FROM student s, enrolled e, course c
WHERE s.sid = e.sid AND e.cid = c.cid;
```

### Outer Joins

```sql
-- Left outer join: All students, even if not enrolled
SELECT s.name, c.title 
FROM student s 
LEFT JOIN enrolled e ON s.sid = e.sid 
LEFT JOIN course c ON e.cid = c.cid;

-- Right outer join
SELECT s.name, c.title 
FROM student s 
RIGHT JOIN department d ON s.dept_id = d.did;

-- Full outer join
SELECT * FROM student s 
FULL OUTER JOIN department d ON s.dept_id = d.did;
```

### Self Join

```sql
-- Find students in the same department
SELECT s1.name, s2.name 
FROM student s1 
JOIN student s2 ON s1.dept_id = s2.dept_id 
WHERE s1.sid < s2.sid;
```

---

## Aggregation

### Aggregate Functions

| Function | Description |
|----------|-------------|
| `COUNT(*)` | Count all rows |
| `COUNT(column)` | Count non-null values |
| `COUNT(DISTINCT column)` | Count unique values |
| `SUM(column)` | Sum of values |
| `AVG(column)` | Average of values |
| `MAX(column)` | Maximum value |
| `MIN(column)` | Minimum value |

```sql
-- Basic aggregation
SELECT COUNT(*) FROM student;
SELECT AVG(gpa) FROM student;
SELECT MAX(gpa), MIN(gpa) FROM student;

-- With filtering
SELECT AVG(gpa) FROM student WHERE dept_id = 1;
```

### GROUP BY

```sql
-- Average GPA by department
SELECT dept_id, AVG(gpa) AS avg_gpa
FROM student
GROUP BY dept_id;

-- Multiple group by columns
SELECT dept_id, YEAR(enrollment_date), COUNT(*)
FROM student
GROUP BY dept_id, YEAR(enrollment_date);
```

### HAVING

```sql
-- Departments with average GPA above 3.5
SELECT dept_id, AVG(gpa) AS avg_gpa
FROM student
GROUP BY dept_id
HAVING AVG(gpa) > 3.5;

-- Note: WHERE filters rows, HAVING filters groups
SELECT dept_id, AVG(gpa) AS avg_gpa
FROM student
WHERE age > 18
GROUP BY dept_id
HAVING COUNT(*) > 10;
```

---

## Advanced Query Features

### Subqueries

```sql
-- Scalar subquery (returns single value)
SELECT name FROM student 
WHERE gpa > (SELECT AVG(gpa) FROM student);

-- Correlated subquery
SELECT s.name FROM student s
WHERE s.gpa > (SELECT AVG(gpa) FROM student WHERE dept_id = s.dept_id);

-- IN subquery
SELECT name FROM student 
WHERE dept_id IN (SELECT did FROM department WHERE building = 'Wean');

-- EXISTS subquery
SELECT d.name FROM department d
WHERE EXISTS (SELECT 1 FROM student s WHERE s.dept_id = d.did);

-- ALL/ANY subqueries
SELECT name FROM student 
WHERE gpa > ALL (SELECT gpa FROM student WHERE dept_id = 2);
```

### Window Functions

Window functions perform calculations across a set of rows related to the current row.

```sql
-- Row number
SELECT name, gpa, 
       ROW_NUMBER() OVER (ORDER BY gpa DESC) AS rank
FROM student;

-- Rank (handles ties)
SELECT name, gpa, 
       RANK() OVER (ORDER BY gpa DESC) AS rank
FROM student;

-- Dense rank (no gaps)
SELECT name, gpa, 
       DENSE_RANK() OVER (ORDER BY gpa DESC) AS rank
FROM student;

-- Partition by
SELECT dept_id, name, gpa,
       RANK() OVER (PARTITION BY dept_id ORDER BY gpa DESC) AS dept_rank
FROM student;

-- Running totals
SELECT name, gpa,
       SUM(gpa) OVER (ORDER BY sid) AS running_total
FROM student;

-- Moving average
SELECT date, value,
       AVG(value) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING) AS moving_avg
FROM sales;

-- First/last value in group
SELECT dept_id, name, gpa,
       FIRST_VALUE(name) OVER (PARTITION BY dept_id ORDER BY gpa DESC) AS top_student
FROM student;
```

### Common Table Expressions (CTEs)

```sql
-- Simple CTE
WITH high_achievers AS (
    SELECT * FROM student WHERE gpa >= 3.5
)
SELECT * FROM high_achievers WHERE age < 22;

-- Multiple CTEs
WITH 
    dept_stats AS (
        SELECT dept_id, AVG(gpa) AS avg_gpa, COUNT(*) AS num_students
        FROM student
        GROUP BY dept_id
    ),
    top_depts AS (
        SELECT dept_id FROM dept_stats WHERE avg_gpa > 3.5
    )
SELECT s.name, s.gpa 
FROM student s 
JOIN top_depts t ON s.dept_id = t.dept_id;

-- Recursive CTE
WITH RECURSIVE ancestors AS (
    -- Base case
    SELECT parent FROM family WHERE child = 'Alice'
    UNION ALL
    -- Recursive case
    SELECT f.parent 
    FROM family f 
    JOIN ancestors a ON f.child = a.parent
)
SELECT * FROM ancestors;
```

---

## Modern SQL Features

### JSON Support

```sql
-- Store JSON
CREATE TABLE users (
    id INT PRIMARY KEY,
    data JSON
);

-- Insert JSON
INSERT INTO users VALUES (1, '{"name": "Alice", "age": 25}');

-- Query JSON
SELECT data->>'name' FROM users;  -- Extract as text
SELECT data->'age' FROM users;     -- Extract as JSON

-- JSON predicates
SELECT * FROM users WHERE data @> '{"active": true}';

-- Aggregate to JSON
SELECT json_object_agg(name, gpa) FROM student;
```

### Array Operations

```sql
-- Array type
CREATE TABLE courses (
    cid INT PRIMARY KEY,
    prerequisites INT[]
);

-- Array operations
SELECT * FROM courses WHERE 101 = ANY(prerequisites);
SELECT * FROM courses WHERE prerequisites @> ARRAY[101, 102];
```

### Full-Text Search

```sql
-- Create text search index
CREATE INDEX idx_fts ON documents USING GIN (to_tsvector('english', content));

-- Full-text query
SELECT * FROM documents 
WHERE to_tsvector('english', content) @@ to_tsquery('database & systems');
```

---

## Query Optimization Hints

### Best Practices

1. **Use appropriate indexes** for frequently queried columns
2. **Avoid `SELECT *`** in production queries
3. **Use `EXPLAIN`** to analyze query execution plans
4. **Consider query rewriting** for complex joins
5. **Batch operations** when possible

### EXPLAIN

```sql
-- View query plan
EXPLAIN SELECT * FROM student WHERE gpa > 3.5;

-- With execution statistics
EXPLAIN ANALYZE SELECT * FROM student WHERE gpa > 3.5;
```

---

## Summary

| Feature | Purpose |
|---------|---------|
| **DDL** | Define and modify schema |
| **DML** | Query and modify data |
| **JOINs** | Combine related tables |
| **Aggregation** | Summarize data |
| **Window Functions** | Row-by-row calculations |
| **CTEs** | Modular, readable queries |
| **Subqueries** | Nested queries |

---

## Key Takeaways

1. **SQL is Declarative**: Say what you want, not how to get it
2. **Joins are Fundamental**: Understanding join types is crucial
3. **Aggregation with GROUP BY**: Summarize data by groups
4. **Window Functions**: Powerful for analytical queries
5. **CTEs Improve Readability**: Break complex queries into parts
6. **Modern SQL is Powerful**: JSON, arrays, full-text search

---

*Next Lecture: Database Storage I - How data is physically stored on disk*
