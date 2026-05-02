# 📚 Complete Guide to Indexing in SSMS (SQL Server)

---

## 📌 Table of Contents
1. [What is an Index?](#what-is-an-index)
2. [Why Use Indexes?](#why-use-indexes)
3. [Types of Indexes](#types-of-indexes)
4. [Clustered Index](#1-clustered-index)
5. [Non-Clustered Index](#2-non-clustered-index)
6. [Unique Index](#3-unique-index)
7. [Composite Index](#4-composite-index)
8. [Full Text Index](#5-full-text-index)
9. [Filtered Index](#6-filtered-index)
10. [How to Create Index via GUI](#how-to-create-index-via-gui-ssms)
11. [How to View Existing Indexes](#how-to-view-existing-indexes)
12. [How to Drop an Index](#how-to-drop-an-index)
13. [Index Performance Check](#index-performance-check)
14. [When to Use Which Index](#when-to-use-which-index)
15. [Enterprise Best Practices](#enterprise-best-practices)

---

## What is an Index?

An index in SQL Server is like the **index at the back of a book**.

Without index:
```
SQL Server reads EVERY row to find your data
→ Called "Full Table Scan"
→ Very SLOW for large tables
```

With index:
```
SQL Server jumps directly to the right row
→ Called "Index Seek"
→ Very FAST!
```

### Real Life Example
```
Table: orders (10 million rows)
Query: Find all orders for customer_id = 1001

Without Index:
→ Scans all 10 million rows ❌ SLOW

With Index on customer_id:
→ Directly jumps to customer 1001 rows ✅ FAST
```

---

## Why Use Indexes?

| Without Index | With Index |
|---|---|
| Full table scan | Direct row lookup |
| Slow queries | Fast queries |
| High CPU usage | Low CPU usage |
| Poor performance | Great performance |

---

## Types of Indexes

```
SQL Server Indexes
│
├── Clustered Index
├── Non-Clustered Index
├── Unique Index
├── Composite Index
├── Full Text Index
└── Filtered Index
```

---

## 1. Clustered Index

### What is it?
- **Physically sorts** the table data
- Only **ONE per table** allowed
- By default created on **Primary Key**
- Data is stored **in order** of this index

### Example

```sql
-- Create table
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,  -- Clustered index auto created here!
    customer_id INT,
    order_date  DATE,
    amount      DECIMAL(10,2)
);
```

#### Explicitly creating clustered index:
```sql
-- Drop existing if needed
DROP INDEX IF EXISTS IX_orders_clustered ON orders;

-- Create clustered index
CREATE CLUSTERED INDEX IX_orders_clustered
ON orders(order_id);
```

### Output when you query:
```
-- Query
SELECT * FROM orders WHERE order_id = 1001;

-- Execution Plan shows:
Clustered Index Seek
→ order_id = 1001
→ 1 row returned
→ Logical reads: 2  ✅ (very fast!)
```

### Think of it as:
```
Phone book sorted by last name
→ Data physically arranged by order_id
→ Finding order 1001 is instant!
```

---

## 2. Non-Clustered Index

### What is it?
- **Does NOT** physically sort the table
- Can have **multiple** per table (up to 999)
- Creates a **separate structure** pointing to data
- Like an index at back of book

### Example

```sql
-- Create non-clustered index on customer_id
CREATE NONCLUSTERED INDEX IX_orders_customer
ON orders(customer_id);
```

### Multiple non-clustered indexes:
```sql
-- Index on order_date
CREATE NONCLUSTERED INDEX IX_orders_date
ON orders(order_date);

-- Index on amount
CREATE NONCLUSTERED INDEX IX_orders_amount
ON orders(amount);
```

### Output comparison:

```
-- WITHOUT index
SELECT * FROM orders WHERE customer_id = 5001;

Execution Plan:
Table Scan
→ Reads: 50,000 logical reads ❌ SLOW
→ Time: 2.5 seconds

-- WITH non-clustered index
SELECT * FROM orders WHERE customer_id = 5001;

Execution Plan:
Index Seek (IX_orders_customer)
→ Reads: 5 logical reads ✅ FAST
→ Time: 0.001 seconds
```

---

## 3. Unique Index

### What is it?
- Ensures **no duplicate values** in a column
- Can be clustered or non-clustered
- Automatically created with UNIQUE constraint

### Example

```sql
-- Create unique index on email
CREATE UNIQUE NONCLUSTERED INDEX IX_customers_email
ON customers(email);
```

### What happens with duplicates:

```sql
-- First insert - works fine
INSERT INTO customers(email) VALUES ('test@gmail.com');
-- Output: (1 row affected) ✅

-- Second insert - same email
INSERT INTO customers(email) VALUES ('test@gmail.com');
-- Output:
-- Msg 2601, Level 14
-- Cannot insert duplicate key row in object 'customers'
-- with unique index 'IX_customers_email' ❌
```

---

## 4. Composite Index

### What is it?
- Index on **multiple columns** together
- Order of columns **matters a lot!**
- Used when queries filter on multiple columns

### Example

```sql
-- Composite index on customer_id AND order_date
CREATE NONCLUSTERED INDEX IX_orders_customer_date
ON orders(customer_id, order_date);
```

### When it helps:
```sql
-- This query USES the index ✅
SELECT * FROM orders
WHERE customer_id = 1001
AND order_date = '2026-01-01';

-- This also USES the index ✅ (leading column)
SELECT * FROM orders
WHERE customer_id = 1001;

-- This does NOT use the index efficiently ❌
-- (order_date is not the leading column)
SELECT * FROM orders
WHERE order_date = '2026-01-01';
```

### Output:
```
Query: customer_id = 1001 AND order_date = '2026-01-01'

With Composite Index:
→ Index Seek on IX_orders_customer_date
→ Logical reads: 3 ✅
→ Time: 0.0005 seconds

Without Index:
→ Table Scan
→ Logical reads: 45,000 ❌
→ Time: 3.2 seconds
```

---

## 5. Full Text Index

### What is it?
- Used for **searching text** inside large text columns
- Like searching inside paragraphs, descriptions
- Used with CONTAINS or FREETEXT

### Example

```sql
-- First enable full text on database
-- Then create full text catalog
CREATE FULLTEXT CATALOG ft_catalog AS DEFAULT;

-- Create full text index
CREATE FULLTEXT INDEX ON products(description)
KEY INDEX PK_products
ON ft_catalog;
```

### Query using full text:
```sql
-- Search products containing word "wireless"
SELECT product_name, description
FROM products
WHERE CONTAINS(description, 'wireless');

-- Output:
-- product_name          description
-- Wireless Headphones   Premium wireless audio device...
-- Wireless Mouse        Ergonomic wireless mouse...
-- Wireless Keyboard     Compact wireless keyboard...
```

---

## 6. Filtered Index

### What is it?
- Index on a **subset of rows**
- Has a WHERE condition
- More efficient than full index
- Great for columns with many NULLs

### Example

```sql
-- Index only on active orders (not completed ones)
CREATE NONCLUSTERED INDEX IX_orders_active
ON orders(customer_id, order_date)
WHERE status = 'Active';
```

### Output:
```
-- Query for active orders
SELECT * FROM orders
WHERE customer_id = 1001
AND status = 'Active';

With Filtered Index:
→ Index Seek (IX_orders_active)
→ Reads only ACTIVE orders
→ Logical reads: 2 ✅ (very efficient!)

Without Filtered Index:
→ Reads ALL orders including completed
→ Logical reads: 50,000 ❌
```

---

## How to Create Index via GUI (SSMS)

```
Object Explorer
      ↓
Your Database (ecomdatawarehouse)
      ↓
Tables
      ↓
Your Table (orders)
      ↓
Indexes
      ↓
Right Click
      ↓
New Index
      ↓
Choose type (Clustered/Non-Clustered)
      ↓
Add columns
      ↓
Click OK ✅
```

---

## How to View Existing Indexes

### Method 1: GUI
```
Object Explorer → Tables → Your Table → Indexes
→ All indexes listed here
```

### Method 2: SQL Query
```sql
-- View all indexes on a table
SELECT
    i.name          AS index_name,
    i.type_desc     AS index_type,
    c.name          AS column_name,
    i.is_unique,
    i.is_primary_key
FROM sys.indexes i
JOIN sys.index_columns ic
    ON i.object_id = ic.object_id
    AND i.index_id = ic.index_id
JOIN sys.columns c
    ON ic.object_id = c.object_id
    AND ic.column_id = c.column_id
WHERE i.object_id = OBJECT_ID('orders');
```

### Output:
```
index_name                  index_type        column_name   is_unique   is_primary_key
PK_orders                   CLUSTERED         order_id      1           1
IX_orders_customer          NONCLUSTERED      customer_id   0           0
IX_orders_date              NONCLUSTERED      order_date    0           0
IX_orders_customer_date     NONCLUSTERED      customer_id   0           0
IX_orders_customer_date     NONCLUSTERED      order_date    0           0
```

---

## How to Drop an Index

```sql
-- Drop a specific index
DROP INDEX IX_orders_customer ON orders;

-- Output:
-- Commands completed successfully ✅
```

---

## Index Performance Check

### Check if index is being used:
```sql
-- Enable statistics
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Run your query
SELECT * FROM orders
WHERE customer_id = 1001;

-- Output:
-- Table 'orders'. Scan count 1,
-- logical reads 3,        ← low = good ✅
-- physical reads 0

-- SQL Server Execution Times:
-- CPU time = 0 ms
-- elapsed time = 1 ms    ← fast = good ✅
```

### Check index usage stats:
```sql
SELECT
    i.name              AS index_name,
    s.user_seeks,       -- how many times index was searched
    s.user_scans,       -- how many times full scan happened
    s.user_lookups,
    s.user_updates      -- how many times index was updated
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i
    ON i.object_id = s.object_id
    AND i.index_id = s.index_id
WHERE s.object_id = OBJECT_ID('orders');
```

### Output:
```
index_name               user_seeks   user_scans   user_updates
PK_orders                15420        0            8930
IX_orders_customer       9870         12           8930
IX_orders_date           0            450          8930   ← barely used!
```

---

## When to Use Which Index

| Situation | Index Type |
|---|---|
| Primary Key column | Clustered (auto) |
| Frequently searched column | Non-Clustered |
| No duplicate values needed | Unique |
| Filter on multiple columns | Composite |
| Search inside text/paragraphs | Full Text |
| Index on specific rows only | Filtered |

---

## Enterprise Best Practices

### ✅ DO
```
1. Always index Primary Keys
2. Index Foreign Keys
3. Index columns used in WHERE clause
4. Index columns used in JOIN conditions
5. Use composite index for multi-column filters
6. Monitor unused indexes and drop them
7. Rebuild indexes regularly (fragmentation)
```

### ❌ DON'T
```
1. Don't index every column
2. Don't over-index (slows down INSERT/UPDATE)
3. Don't index columns with few unique values
   Example: gender column (M/F) → bad index
4. Don't forget to maintain indexes
```

### Index Maintenance (Enterprise Standard):
```sql
-- Check index fragmentation
SELECT
    i.name,
    ps.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i
    ON ps.object_id = i.object_id
    AND ps.index_id = i.index_id
WHERE ps.avg_fragmentation_in_percent > 10;

-- Output:
-- index_name              avg_fragmentation
-- IX_orders_customer      45.6   ← needs rebuild!
-- IX_orders_date          8.2    ← okay

-- Rebuild fragmented index
ALTER INDEX IX_orders_customer ON orders REBUILD;
-- Output: Commands completed successfully ✅

-- Reorganize slightly fragmented index (< 30%)
ALTER INDEX IX_orders_date ON orders REORGANIZE;
-- Output: Commands completed successfully ✅
```

---

## Quick Reference Card

```
Create Clustered:
CREATE CLUSTERED INDEX IX_name ON table(column);

Create Non-Clustered:
CREATE NONCLUSTERED INDEX IX_name ON table(column);

Create Unique:
CREATE UNIQUE NONCLUSTERED INDEX IX_name ON table(column);

Create Composite:
CREATE NONCLUSTERED INDEX IX_name ON table(col1, col2);

Create Filtered:
CREATE NONCLUSTERED INDEX IX_name ON table(column)
WHERE condition;

Drop Index:
DROP INDEX IX_name ON table;

View Indexes:
SELECT * FROM sys.indexes WHERE object_id = OBJECT_ID('table');
```

---
