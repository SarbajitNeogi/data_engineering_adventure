# 🗄️ SQL Server Collation — Complete Guide

> A comprehensive reference for understanding SQL Server collations, with practical examples for developers and DBAs.

---

## 📚 Table of Contents

- [What is Collation?](#what-is-collation)
- [What Does Collation Affect?](#what-does-collation-affect)
- [Types of Collation](#types-of-collation)
- [Sensitivity Flags](#sensitivity-flags)
- [Collation Scope](#collation-scope)
- [Common Collations](#common-collations)
- [Practical Examples](#practical-examples)
- [How to Check Collation](#how-to-check-collation)
- [How to Change Collation](#how-to-change-collation)
- [Best Practices](#best-practices)
- [FAQ](#faq)

---

## What is Collation?

**Collation** is a set of rules that determines how SQL Server **sorts** and **compares** text data.

It acts as a *rulebook* that SQL Server consults whenever it asks:
- *"Are these two strings equal?"*
- *"Which string comes first alphabetically?"*

```sql
-- The default collation set during database creation
-- ⚠️ Cannot be changed after database creation!
SQL_Latin1_General_CP1_CI_AS
```

### Breaking Down `SQL_Latin1_General_CP1_CI_AS`

| Part | Meaning |
|------|---------|
| `SQL_` | Legacy SQL Server collation |
| `Latin1_General` | Latin1 character set (Western European languages) |
| `CP1` | Code Page 1252 (Windows Western European encoding) |
| `CI` | **Case Insensitive** — `'A'` = `'a'` |
| `AS` | **Accent Sensitive** — `'e'` ≠ `'é'` |

---

## What Does Collation Affect?

Collation **only applies to text columns**: `CHAR`, `VARCHAR`, `NCHAR`, `NVARCHAR`

It does **NOT** affect: numbers (`INT`, `FLOAT`), dates (`DATETIME`), or binary data.

### 1. 🔍 WHERE Clauses (Filtering)

```sql
-- With CI (Case Insensitive) collation:
SELECT * FROM Users WHERE name = 'john'

-- ✅ Matches: 'john', 'John', 'JOHN', 'jOhN'
-- ❌ With CS (Case Sensitive): only matches 'john'
```

### 2. 📋 ORDER BY (Sorting)

```sql
-- Collation determines where accented chars like é, ñ, ü appear
SELECT name FROM Employees ORDER BY name

-- CI_AS result:     CI_AI result:
-- apple             apple
-- Apple     vs.     Apple
-- épée              epee
-- résumé            resume
```

### 3. 🔗 JOINs

```sql
-- Whether 'Apple' in TableA matches 'apple' in TableB
-- depends entirely on collation
SELECT *
FROM TableA a
JOIN TableB b ON a.name = b.name  -- collation rules apply here
```

---

## Types of Collation

### 1. 🪟 Windows Collations *(Recommended)*

Based on Windows OS linguistic rules. Best Unicode support.

```sql
Latin1_General_CI_AS
French_CI_AS
Japanese_CI_AS
Arabic_CI_AS
```

### 2. 🗄️ SQL Server Collations *(Legacy)*

Older collations kept for backward compatibility. Always prefixed with `SQL_`.

```sql
SQL_Latin1_General_CP1_CI_AS    -- Most common default (legacy)
SQL_Latin1_General_CP1_CS_AS    -- Case sensitive variant
SQL_Latin1_General_CP850_BIN    -- Binary variant
```

> ⚠️ Not recommended for new projects. Use Windows collations instead.

### 3. ⚡ Binary Collations

Compares raw byte values. No linguistic rules. Fastest performance.

```sql
Latin1_General_BIN     -- Older binary comparison
Latin1_General_BIN2    -- Strict byte-level comparison (recommended over BIN)
```

| Feature | BIN | BIN2 |
|---------|-----|------|
| Comparison method | Character code | Raw bytes |
| Performance | Fast | Faster |
| Case sensitivity | Always CS | Always CS |
| Recommended | No | Yes |

### 4. 🌐 UTF-8 Collations *(SQL Server 2019+)*

Modern collations with full Unicode/UTF-8 support. Best for multilingual apps.

```sql
Latin1_General_100_CI_AS_SC_UTF8
Latin1_General_100_CS_AS_SC_UTF8
Japanese_Bushu_Kakusu_100_CI_AS_SC_UTF8
```

---

## Sensitivity Flags

Every collation is a combination of these flags:

| Flag | Full Name | Options | Meaning |
|------|-----------|---------|---------|
| `CI` / `CS` | Case | Insensitive / Sensitive | `'A'` = `'a'` or `'A'` ≠ `'a'` |
| `AI` / `AS` | Accent | Insensitive / Sensitive | `'e'` = `'é'` or `'e'` ≠ `'é'` |
| `KI` / `KS` | Kana | Insensitive / Sensitive | Japanese Hiragana vs Katakana |
| `WI` / `WS` | Width | Insensitive / Sensitive | Full-width vs half-width characters |
| `VI` / `VS` | Variation Selector | Insensitive / Sensitive | Unicode variation sequences |
| `SC` | Supplementary Characters | Enabled | Support for emoji & rare Unicode |
| `UTF8` | UTF-8 Encoding | Enabled | UTF-8 storage for VARCHAR |

### Examples

```sql
Latin1_General_CI_AI   -- Case Insensitive, Accent Insensitive
Latin1_General_CI_AS   -- Case Insensitive, Accent Sensitive   ← most common
Latin1_General_CS_AS   -- Case Sensitive,   Accent Sensitive
Latin1_General_CS_AI   -- Case Sensitive,   Accent Insensitive (rare)
```

---

## Collation Scope

Collation can be set at 4 different levels. Each level can override the one above it.

```
┌─────────────────────────────────────────┐
│           SERVER LEVEL                  │
│   Default for all new databases         │
│                                         │
│   ┌─────────────────────────────────┐   │
│   │       DATABASE LEVEL            │   │
│   │  Default for all new columns    │   │
│   │                                 │   │
│   │   ┌─────────────────────────┐   │   │
│   │   │     COLUMN LEVEL        │   │   │
│   │   │  Overrides DB default   │   │   │
│   │   │                         │   │   │
│   │   │  ┌───────────────────┐  │   │   │
│   │   │  │   QUERY LEVEL     │  │   │   │
│   │   │  │  COLLATE keyword  │  │   │   │
│   │   │  └───────────────────┘  │   │   │
│   │   └─────────────────────────┘   │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### Query Level (Override on the fly)

```sql
-- Force case-sensitive comparison for just this query
SELECT * FROM Users
WHERE name = 'John' COLLATE Latin1_General_CS_AS

-- Compare columns with different collations
SELECT *
FROM TableA a
JOIN TableB b
  ON a.name = b.name COLLATE Latin1_General_CI_AS
```

### Column Level

```sql
CREATE TABLE Products (
    id          INT PRIMARY KEY,
    name        NVARCHAR(100),                              -- inherits DB collation
    code        VARCHAR(50) COLLATE Latin1_General_CS_AS,  -- always case sensitive
    description NVARCHAR(500) COLLATE Latin1_General_CI_AI -- case & accent insensitive
)
```

---

## Common Collations

| Collation | Use Case |
|-----------|----------|
| `SQL_Latin1_General_CP1_CI_AS` | Legacy default, most existing SQL Server DBs |
| `Latin1_General_CI_AS` | Modern Windows equivalent of above |
| `Latin1_General_100_CI_AS_SC_UTF8` | New projects needing UTF-8 support |
| `Latin1_General_BIN2` | Case-sensitive exact matching, max performance |
| `French_CI_AS` | French language applications |
| `Arabic_CI_AS` | Arabic language applications |
| `Japanese_CI_AS` | Japanese language applications |
| `Chinese_PRC_CI_AS` | Simplified Chinese applications |

---

## Practical Examples

### Example 1: Login System (Case Insensitive username, Case Sensitive password)

```sql
CREATE TABLE Users (
    id           INT IDENTITY PRIMARY KEY,
    username     VARCHAR(50)  COLLATE Latin1_General_CI_AS,  -- 'John' = 'john'
    password     VARCHAR(255) COLLATE Latin1_General_CS_AS,  -- 'Pass123' ≠ 'pass123'
    email        VARCHAR(100) COLLATE Latin1_General_CI_AS   -- emails are case insensitive
)

-- Login check
SELECT * FROM Users
WHERE username = 'john'          -- matches 'John', 'JOHN'
  AND password = 'MyP@ssw0rd'    -- must match exactly
```

### Example 2: Product Search (Accent Insensitive)

```sql
-- Users searching 'cafe' should find 'café'
SELECT * FROM Products
WHERE name = 'cafe' COLLATE Latin1_General_CI_AI

-- ✅ Returns: 'cafe', 'Cafe', 'café', 'Café'
```

### Example 3: Multilingual App (UTF-8)

```sql
CREATE DATABASE MyGlobalApp
COLLATE Latin1_General_100_CI_AS_SC_UTF8

-- Supports: English, Arabic, Chinese, Japanese, Emoji etc.
-- Stores VARCHAR as UTF-8 (more space efficient than NVARCHAR for ASCII)
```

### Example 4: Collation Conflict in JOIN

```sql
-- ❌ This will ERROR if tables have different collations
SELECT * FROM DB1.dbo.Users u
JOIN DB2.dbo.Orders o ON u.name = o.customer_name
-- Error: Cannot resolve collation conflict

-- ✅ Fix with COLLATE
SELECT * FROM DB1.dbo.Users u
JOIN DB2.dbo.Orders o
  ON u.name = o.customer_name COLLATE Latin1_General_CI_AS
```

---

## How to Check Collation

```sql
-- Check SERVER collation
SELECT SERVERPROPERTY('Collation') AS ServerCollation

-- Check DATABASE collation
SELECT name, collation_name
FROM sys.databases
WHERE name = DB_NAME()

-- Check COLUMN collation
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'YourTableName'
  AND COLLATION_NAME IS NOT NULL

-- List ALL available collations (4000+)
SELECT name, description
FROM sys.fn_helpcollations()
ORDER BY name

-- Filter collations
SELECT name, description
FROM sys.fn_helpcollations()
WHERE name LIKE '%Latin1%CI_AS%'
```

---

## How to Change Collation

> ⚠️ **WARNING**: Changing collation on an existing database is risky and complex. Plan carefully.

### Server Level (requires restart)

```sql
-- Check current server collation
SELECT SERVERPROPERTY('Collation')

-- Change via SQL Server Configuration or setup
-- sqlservr.exe -q "Latin1_General_CI_AS"
```

### Database Level (new databases only — cannot change after creation via Portal)

```sql
-- Set collation at creation time
CREATE DATABASE MyDatabase
COLLATE Latin1_General_100_CI_AS_SC_UTF8

-- Change existing DB collation (use with extreme caution!)
ALTER DATABASE MyDatabase
COLLATE Latin1_General_CI_AS
-- Note: This changes the default for NEW columns only, not existing ones
```

### Column Level

```sql
-- Change collation on existing column
ALTER TABLE Users
ALTER COLUMN name NVARCHAR(100)
COLLATE Latin1_General_CS_AS NOT NULL
-- Note: Drops and recreates any indexes on this column
```

---

## Best Practices

- ✅ **Use Windows collations** (`Latin1_General_CI_AS`) over legacy SQL collations (`SQL_Latin1_General_CP1_CI_AS`) for new projects
- ✅ **Use UTF-8 collations** (`_UTF8`) on SQL Server 2019+ for multilingual/global applications
- ✅ **Be consistent** — use the same collation across all databases that JOIN each other
- ✅ **Set collation at database creation** — changing it later is painful
- ✅ **Use `_SC`** (Supplementary Characters) flag if you need emoji or rare Unicode support
- ✅ **Use `BIN2`** when you need exact byte-level matching and maximum performance
- ❌ **Avoid mixing collations** in the same database unless absolutely necessary
- ❌ **Don't use `SQL_` prefixed collations** for new databases
- ❌ **Never ignore collation conflicts** — fix them at the column or query level

---

## FAQ

**Q: Can I change collation after database creation in Azure Portal?**
> No. The Azure Portal warning is correct — database-level collation cannot be changed after creation. You would need to create a new database with the correct collation and migrate data.

**Q: What's the difference between `CI` and `CS`?**
> `CI` (Case Insensitive) treats `'A'` and `'a'` as equal. `CS` (Case Sensitive) treats them as different. Most applications use `CI` for usernames and searches.

**Q: Why does my JOIN fail with "collation conflict" error?**
> Two columns being compared have different collations. Fix with `COLLATE` keyword: `ON a.col = b.col COLLATE Latin1_General_CI_AS`

**Q: Which collation should I use for a new project?**
> SQL Server 2019+: `Latin1_General_100_CI_AS_SC_UTF8`
> Older versions: `Latin1_General_CI_AS`

**Q: Does collation affect performance?**
> Slightly. Binary collations (`BIN2`) are fastest. Linguistic collations have minimal overhead for most workloads. More importantly, collation mismatches can **prevent index usage**, which is a major performance issue.

**Q: How many collations does SQL Server support?**
> 4,000+ collations covering hundreds of languages. Run `SELECT * FROM sys.fn_helpcollations()` to see all.

---

## 📖 Further Reading

- [Microsoft Docs — Collation and Unicode Support](https://docs.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support)
- [sys.fn_helpcollations (Transact-SQL)](https://docs.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-helpcollations-transact-sql)
- [Windows vs SQL Collations](https://docs.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support#SQL-collations)
- [UTF-8 Support in SQL Server 2019](https://docs.microsoft.com/en-us/sql/relational-databases/collations/collation-and-unicode-support#utf8)

---

*Generated with ❤️ — Feel free to contribute or raise issues!*
