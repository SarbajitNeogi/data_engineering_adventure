# 🗄️ Azure SQL — Complete Setup Guide

> Step-by-step guide to create an Azure SQL Server, configure firewall, create a database, build tables, manage indexes, write stored procedures, handle backups, monitor performance, and secure your database.

---

## 📚 Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step 1 — Create a SQL Server in Azure](#step-1--create-a-sql-server-in-azure)
- [Step 2 — Configure Firewall Rules](#step-2--configure-firewall-rules)
- [Step 3 — Create a Database](#step-3--create-a-database)
- [Step 4 — Connect to the Database](#step-4--connect-to-the-database)
- [Step 5 — Create Tables](#step-5--create-tables)
- [Step 6 — Insert Data](#step-6--insert-data)
- [Step 7 — Query the Data](#step-7--query-the-data)
- [Step 8 — Update and Delete Data](#step-8--update-and-delete-data)
- [Step 9 — Indexes](#step-9--indexes)
- [Step 10 — Stored Procedures](#step-10--stored-procedures)
- [Step 11 — Views](#step-11--views)
- [Step 12 — Triggers](#step-12--triggers)
- [Step 13 — Transactions](#step-13--transactions)
- [Step 14 — Backup and Restore](#step-14--backup-and-restore)
- [Step 15 — Security and Users](#step-15--security-and-users)
- [Step 16 — Monitor and Performance](#step-16--monitor-and-performance)
- [Step 17 — Alter and Maintain Tables](#step-17--alter-and-maintain-tables)
- [Full Example Project](#full-example-project)
- [Common Errors](#common-errors)
- [Best Practices](#best-practices)
- [FAQ](#faq)

---

## Prerequisites

Before you begin, make sure you have:

| Requirement | Details |
|-------------|---------|
| ✅ Azure Account | [Create free account](https://azure.microsoft.com/free/) |
| ✅ Azure Subscription | Free tier works fine for learning |
| ✅ Azure Portal Access | [portal.azure.com](https://portal.azure.com) |
| ✅ SQL Client Tool | SSMS, Azure Data Studio, or VS Code |

### Recommended Tools to Connect

| Tool | Platform | Download |
|------|----------|---------|
| **SSMS** (SQL Server Management Studio) | Windows | [Download](https://aka.ms/ssmsfullsetup) |
| **Azure Data Studio** | Windows / Mac / Linux | [Download](https://aka.ms/azuredatastudio) |
| **VS Code + SQL Extension** | All platforms | [Extension](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql) |

---

## Architecture Overview

Understanding the hierarchy before you start:

```
┌──────────────────────────────────────────────────────────────┐
│                    AZURE SUBSCRIPTION                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │               RESOURCE GROUP (MyProject-RG)            │  │
│  │                                                        │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │         SQL SERVER (logical server)              │  │  │
│  │  │   my-sql-server-prod.database.windows.net        │  │  │
│  │  │                                                  │  │  │
│  │  │   ┌──────────────┐   ┌──────────────┐           │  │  │
│  │  │   │  Database 1  │   │  Database 2  │   ...     │  │  │
│  │  │   │  MyAppDB     │   │  AnalyticsDB │           │  │  │
│  │  │   │              │   │              │           │  │  │
│  │  │   │  ┌────────┐  │   └──────────────┘           │  │  │
│  │  │   │  │Tables  │  │                              │  │  │
│  │  │   │  │Views   │  │                              │  │  │
│  │  │   │  │Procs   │  │                              │  │  │
│  │  │   │  └────────┘  │                              │  │  │
│  │  │   └──────────────┘                              │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

> 💡 The **SQL Server** is just a host — it holds no data itself. All data lives inside **Databases**.

---

## Step 1 — Create a SQL Server in Azure

> ⚠️ **Note:** In Azure, you first create a **SQL Server** (the container/host), then create **databases** inside it. The server itself holds no data.

### Via Azure Portal

```
1. Go to portal.azure.com
2. Click "+ Create a resource"
3. Search "SQL Server" → Select "SQL server (logical server)"
4. Click "Create"
```

### Fill in the Basics Tab

```
┌─────────────────────────────────────────────────────┐
│                   BASICS TAB                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Subscription     : Your Azure Subscription         │
│  Resource Group   : Create new → "MyProject-RG"     │
│  Server Name      : my-sql-server-prod              │
│                     (must be globally unique)        │
│  Region           : (Asia Pacific) Central India     │
│                     or closest to your users         │
│  Authentication   : Use SQL Authentication           │
│  Admin Login      : sqladmin                        │
│  Password         : MyStr0ng!Pass#2024               │
│                     (min 8 chars, upper+lower+num)   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

> 💡 **Server name** becomes part of your connection string:
> `my-sql-server-prod.database.windows.net`

### Authentication Options

| Method | Best For | Security Level |
|--------|----------|---------------|
| **SQL Authentication** | Simple setups, learning | 🔒 Medium |
| **Azure AD Only** | Enterprise, teams | 🔒🔒 High |
| **Both** | Transition period | 🔒🔒 High |

### Networking Tab

```
┌─────────────────────────────────────────────────────┐
│                 NETWORKING TAB                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Connectivity Method : Public endpoint              │
│                                                     │
│  Firewall Rules:                                    │
│  ☑ Allow Azure services to access this server       │
│  ☑ Add current client IP address   ← IMPORTANT!    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Review + Create

```
Click "Review + Create"
     │
     ▼
Validation Passed ✅
     │
     ▼
Click "Create"
     │
     ▼
⏳ Deployment in progress... (~1-2 minutes)
     │
     ▼
✅ "Your deployment is complete"
     │
     ▼
Click "Go to resource"
```

### Via Azure CLI

```bash
# Login to Azure
az login

# Create Resource Group
az group create \
    --name MyProject-RG \
    --location centralindia

# Create SQL Server
az sql server create \
    --name my-sql-server-prod \
    --resource-group MyProject-RG \
    --location centralindia \
    --admin-user sqladmin \
    --admin-password "MyStr0ng!Pass#2024"

# Output:
# {
#   "fullyQualifiedDomainName": "my-sql-server-prod.database.windows.net",
#   "state": "Ready",
#   ...
# }
```

---

## Step 2 — Configure Firewall Rules

After server creation, make sure your IP is allowed:

### Via Azure Portal

```
Go to your SQL Server resource
→ Left menu: "Networking" (under Security)
→ "Public access" tab
→ Firewall rules section
→ Click "+ Add your client IPv4 address"
→ Click "Save"
```

### Via T-SQL

```sql
-- Run on master database
EXECUTE sp_set_firewall_rule
    @name = N'MyHomePC',
    @start_ip_address = '203.0.113.45',
    @end_ip_address = '203.0.113.45'
```

### Via Azure CLI

```bash
az sql server firewall-rule create \
    --resource-group MyProject-RG \
    --server my-sql-server-prod \
    --name MyHomePC \
    --start-ip-address 203.0.113.45 \
    --end-ip-address 203.0.113.45
```

---

## Step 3 — Create a Database

Now that the server exists, create a database inside it.

### Via Azure Portal

```
Go to your SQL Server resource
→ Left menu: "Databases"
→ Click "+ Create database"
```

### Fill in the Create Database Form

```
┌─────────────────────────────────────────────────────┐
│               CREATE DATABASE FORM                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  BASICS TAB                                         │
│  ─────────────────────────────                      │
│  Subscription   : Your subscription                 │
│  Resource Group : MyProject-RG                      │
│  Database Name  : MyAppDB                           │
│  Server         : my-sql-server-prod  (existing)    │
│  Elastic Pool   : No (for now)                      │
│  Workload Env   : Development  ← cheaper!           │
│                                                     │
│  COMPUTE + STORAGE                                  │
│  ─────────────────────────────                      │
│  Service Tier   : General Purpose                   │
│  Compute Tier   : Serverless  ← auto-pauses, cheap  │
│  vCores         : 1 (min) - 4 (max)                 │
│  Storage        : 32 GB                             │
│                                                     │
│  COLLATION                                          │
│  ─────────────────────────────                      │
│  Collation      : SQL_Latin1_General_CP1_CI_AS      │
│                   ⚠️ Cannot change after creation!  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Service Tier Comparison

| Tier | Best For | Cost |
|------|----------|------|
| **Serverless** | Dev/Test, sporadic use | 💰 Cheapest (auto-pause) |
| **Basic** | Small apps, low traffic | 💰 Low |
| **Standard** | Medium apps | 💰💰 Medium |
| **Premium** | High performance, OLTP | 💰💰💰 High |
| **Hyperscale** | Very large databases | 💰💰💰💰 Highest |

> 💡 For learning/dev: Use **Serverless** with auto-pause — it stops charging when idle!

### Via Azure CLI

```bash
az sql db create \
    --resource-group MyProject-RG \
    --server my-sql-server-prod \
    --name MyAppDB \
    --service-objective S0 \
    --collation SQL_Latin1_General_CP1_CI_AS

# For serverless:
az sql db create \
    --resource-group MyProject-RG \
    --server my-sql-server-prod \
    --name MyAppDB \
    --edition GeneralPurpose \
    --compute-model Serverless \
    --family Gen5 \
    --min-capacity 0.5 \
    --capacity 2
```

### Via T-SQL (on master database)

```sql
CREATE DATABASE MyAppDB
COLLATE SQL_Latin1_General_CP1_CI_AS
```

---

## Step 4 — Connect to the Database

### Connection String Details

```
Server   : my-sql-server-prod.database.windows.net
Database : MyAppDB
Username : sqladmin
Password : MyStr0ng!Pass#2024
Port     : 1433
```

### Connect Using SSMS

```
1. Open SSMS
2. Server type     : Database Engine
3. Server name     : my-sql-server-prod.database.windows.net
4. Authentication  : SQL Server Authentication
5. Login           : sqladmin
6. Password        : MyStr0ng!Pass#2024
7. Click Connect ✅
```

### Connect Using Azure Data Studio

```
1. Open Azure Data Studio
2. Click "New Connection"
3. Server         : my-sql-server-prod.database.windows.net
4. Authentication : SQL Login
5. Username       : sqladmin
6. Password       : MyStr0ng!Pass#2024
7. Database       : MyAppDB
8. Click Connect ✅
```

### Connection Strings by Language

```csharp
// C# / .NET
"Server=my-sql-server-prod.database.windows.net;
 Database=MyAppDB;
 User Id=sqladmin;
 Password=MyStr0ng!Pass#2024;
 Encrypt=True;"
```

```python
# Python (pyodbc)
conn_str = (
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=my-sql-server-prod.database.windows.net;"
    "DATABASE=MyAppDB;"
    "UID=sqladmin;"
    "PWD=MyStr0ng!Pass#2024;"
    "Encrypt=yes;"
)
```

```javascript
// Node.js (mssql)
const config = {
    server: 'my-sql-server-prod.database.windows.net',
    database: 'MyAppDB',
    user: 'sqladmin',
    password: 'MyStr0ng!Pass#2024',
    options: {
        encrypt: true,
        trustServerCertificate: false
    }
}
```

```java
// Java (JDBC)
String url = "jdbc:sqlserver://my-sql-server-prod.database.windows.net:1433;"
           + "database=MyAppDB;"
           + "user=sqladmin@my-sql-server-prod;"
           + "password=MyStr0ng!Pass#2024;"
           + "encrypt=true;";
```

---

## Step 5 — Create Tables

Now connect to `MyAppDB` and start creating tables.

### Basic Table Syntax

```sql
CREATE TABLE TableName (
    column_name  DataType  Constraints,
    column_name  DataType  Constraints,
    ...
)
```

### Example 1 — Simple Users Table

```sql
USE MyAppDB
GO

CREATE TABLE Users (
    UserID       INT           IDENTITY(1,1) PRIMARY KEY,
    FirstName    NVARCHAR(50)  NOT NULL,
    LastName     NVARCHAR(50)  NOT NULL,
    Email        NVARCHAR(100) NOT NULL UNIQUE,
    Phone        VARCHAR(15)   NULL,
    IsActive     BIT           NOT NULL DEFAULT 1,
    CreatedAt    DATETIME      NOT NULL DEFAULT GETDATE(),
    UpdatedAt    DATETIME      NULL
)
```

### Example 2 — Products Table

```sql
CREATE TABLE Products (
    ProductID    INT             IDENTITY(1,1) PRIMARY KEY,
    ProductName  NVARCHAR(100)   NOT NULL,
    Description  NVARCHAR(500)   NULL,
    Price        DECIMAL(10,2)   NOT NULL CHECK (Price >= 0),
    Stock        INT             NOT NULL DEFAULT 0,
    CategoryID   INT             NULL,
    CreatedAt    DATETIME        NOT NULL DEFAULT GETDATE()
)
```

### Example 3 — Orders Table (with Foreign Keys)

```sql
CREATE TABLE Orders (
    OrderID      INT           IDENTITY(1,1) PRIMARY KEY,
    UserID       INT           NOT NULL,
    OrderDate    DATETIME      NOT NULL DEFAULT GETDATE(),
    TotalAmount  DECIMAL(10,2) NOT NULL,
    Status       VARCHAR(20)   NOT NULL DEFAULT 'Pending',
    ShippedDate  DATETIME      NULL,

    CONSTRAINT FK_Orders_Users
        FOREIGN KEY (UserID)
        REFERENCES Users(UserID)
        ON DELETE CASCADE
        ON UPDATE CASCADE
)
```

### Example 4 — Order Items Table

```sql
CREATE TABLE OrderItems (
    OrderItemID  INT            IDENTITY(1,1) PRIMARY KEY,
    OrderID      INT            NOT NULL,
    ProductID    INT            NOT NULL,
    Quantity     INT            NOT NULL CHECK (Quantity > 0),
    UnitPrice    DECIMAL(10,2)  NOT NULL,

    CONSTRAINT FK_OrderItems_Orders
        FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID)
        ON DELETE CASCADE,

    CONSTRAINT FK_OrderItems_Products
        FOREIGN KEY (ProductID)
        REFERENCES Products(ProductID)
)
```

### Common Data Types Reference

| Data Type | Use For | Example |
|-----------|---------|---------|
| `INT` | Whole numbers | `42`, `1000` |
| `BIGINT` | Large whole numbers | `9999999999` |
| `DECIMAL(p,s)` | Exact decimals (money) | `99.99` |
| `FLOAT` | Approximate decimals | `3.14159` |
| `VARCHAR(n)` | Variable text (ASCII) | `'Hello'` |
| `NVARCHAR(n)` | Variable text (Unicode) | `'नमस्ते'` |
| `CHAR(n)` | Fixed-length text | `'M'` or `'F'` |
| `BIT` | Boolean (0 or 1) | `1` = true |
| `DATETIME` | Date and time | `2024-01-15 10:30:00` |
| `DATE` | Date only | `2024-01-15` |
| `TIME` | Time only | `10:30:00` |
| `UNIQUEIDENTIFIER` | GUID | `'6F9619FF-...'` |

### Common Constraints Reference

| Constraint | Meaning | Example |
|------------|---------|---------|
| `PRIMARY KEY` | Unique identifier for each row | `UserID INT PRIMARY KEY` |
| `NOT NULL` | Value is required | `Name NVARCHAR(50) NOT NULL` |
| `NULL` | Value is optional | `Phone VARCHAR(15) NULL` |
| `UNIQUE` | No duplicate values allowed | `Email NVARCHAR(100) UNIQUE` |
| `DEFAULT` | Auto value if none provided | `DEFAULT GETDATE()` |
| `IDENTITY(1,1)` | Auto-increment starting at 1 | `ID INT IDENTITY(1,1)` |
| `FOREIGN KEY` | Links to another table | See Orders example above |
| `CHECK` | Validates a condition | `CHECK (Price >= 0)` |

---

## Step 6 — Insert Data

```sql
-- Insert single row
INSERT INTO Users (FirstName, LastName, Email, Phone)
VALUES ('Raj', 'Sharma', 'raj@example.com', '+91-9876543210')

-- Insert multiple rows at once
INSERT INTO Users (FirstName, LastName, Email)
VALUES
    ('Priya', 'Patel',  'priya@example.com'),
    ('Amit',  'Gupta',  'amit@example.com'),
    ('Sneha', 'Verma',  'sneha@example.com')

-- Insert into Products
INSERT INTO Products (ProductName, Description, Price, Stock)
VALUES
    ('Laptop Pro',    '15-inch i7 laptop',    75000.00, 50),
    ('Wireless Mouse','Ergonomic USB mouse',     999.00, 200),
    ('USB-C Hub',     '7-in-1 USB-C adapter',  2499.00, 100)

-- Insert an Order
INSERT INTO Orders (UserID, TotalAmount, Status)
VALUES (1, 75999.00, 'Confirmed')

-- Insert Order Items
INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
VALUES
    (1, 1, 1, 75000.00),
    (1, 2, 1,   999.00)
```

---

## Step 7 — Query the Data

### Basic SELECT

```sql
-- Get all users
SELECT * FROM Users

-- Get specific columns
SELECT FirstName, LastName, Email FROM Users

-- Filter with WHERE
SELECT * FROM Users WHERE IsActive = 1

-- Sort results
SELECT * FROM Products ORDER BY Price DESC

-- Limit rows (TOP)
SELECT TOP 10 * FROM Orders ORDER BY OrderDate DESC

-- Search with LIKE
SELECT * FROM Products WHERE ProductName LIKE '%Laptop%'

-- Range filter
SELECT * FROM Products WHERE Price BETWEEN 500 AND 5000

-- Multiple conditions
SELECT * FROM Users
WHERE IsActive = 1 AND CreatedAt >= '2024-01-01'
```

### JOIN Examples

```sql
-- INNER JOIN — Orders with User Details
SELECT
    o.OrderID,
    u.FirstName + ' ' + u.LastName  AS CustomerName,
    u.Email,
    o.OrderDate,
    o.TotalAmount,
    o.Status
FROM Orders o
INNER JOIN Users u ON o.UserID = u.UserID
ORDER BY o.OrderDate DESC

-- Full Order Details (multiple JOINs)
SELECT
    o.OrderID,
    u.FirstName + ' ' + u.LastName  AS CustomerName,
    p.ProductName,
    oi.Quantity,
    oi.UnitPrice,
    (oi.Quantity * oi.UnitPrice)    AS LineTotal
FROM Orders o
INNER JOIN Users       u  ON o.UserID     = u.UserID
INNER JOIN OrderItems  oi ON o.OrderID    = oi.OrderID
INNER JOIN Products    p  ON oi.ProductID = p.ProductID
ORDER BY o.OrderID, p.ProductName

-- LEFT JOIN — All users even if no orders
SELECT
    u.FirstName,
    u.LastName,
    COUNT(o.OrderID) AS TotalOrders
FROM Users u
LEFT JOIN Orders o ON u.UserID = o.UserID
GROUP BY u.UserID, u.FirstName, u.LastName
```

### Aggregation

```sql
-- Total revenue
SELECT SUM(TotalAmount) AS TotalRevenue FROM Orders

-- Count, Min, Max, Avg
SELECT
    COUNT(*)        AS TotalProducts,
    MIN(Price)      AS CheapestPrice,
    MAX(Price)      AS MostExpensive,
    AVG(Price)      AS AveragePrice
FROM Products

-- Orders per user with HAVING filter
SELECT
    u.FirstName + ' ' + u.LastName AS CustomerName,
    COUNT(o.OrderID)               AS TotalOrders,
    SUM(o.TotalAmount)             AS TotalSpent
FROM Users u
LEFT JOIN Orders o ON u.UserID = o.UserID
GROUP BY u.UserID, u.FirstName, u.LastName
HAVING SUM(o.TotalAmount) > 1000
ORDER BY TotalSpent DESC
```

---

## Step 8 — Update and Delete Data

### UPDATE

```sql
-- Update a single row
UPDATE Users
SET Phone = '+91-9999999999'
WHERE UserID = 1

-- Update multiple columns
UPDATE Users
SET
    Phone     = '+91-9999999999',
    UpdatedAt = GETDATE()
WHERE Email = 'raj@example.com'

-- Update based on another table
UPDATE p
SET p.Stock = p.Stock - oi.Quantity
FROM Products p
INNER JOIN OrderItems oi ON p.ProductID = oi.ProductID
WHERE oi.OrderID = 1

-- ⚠️ Always use WHERE — without it ALL rows get updated!
```

### DELETE

```sql
-- Delete a specific row
DELETE FROM Users WHERE UserID = 5

-- Delete with condition
DELETE FROM Orders WHERE Status = 'Cancelled'

-- Delete all rows but keep table structure
TRUNCATE TABLE OrderItems   -- faster, resets IDENTITY counter

-- Safe pattern — verify before deleting
SELECT * FROM Users WHERE UserID = 5    -- check first
DELETE FROM Users WHERE UserID = 5      -- then delete

-- ⚠️ Always use WHERE — without it ALL rows are deleted!
```

---

## Step 9 — Indexes

Indexes speed up data retrieval. Without them, SQL Server scans every row (full table scan).

### Types of Indexes

| Type | Description | Use Case |
|------|-------------|----------|
| **Clustered** | Sorts table physically by this column | Primary Key (auto-created) |
| **Non-Clustered** | Separate lookup structure | Columns used in WHERE/JOIN |
| **Unique** | Enforces uniqueness + speeds lookup | Email, Username |
| **Composite** | Index on multiple columns | Multi-column WHERE clauses |
| **Full-Text** | Search within large text content | Search features |

### Create Indexes

```sql
-- Non-clustered index on Email (speeds up login queries)
CREATE INDEX IX_Users_Email
ON Users (Email)

-- Index on foreign key column (speeds up JOINs)
CREATE INDEX IX_Orders_UserID
ON Orders (UserID)

-- Composite index (for queries filtering by both columns)
CREATE INDEX IX_Orders_UserID_Status
ON Orders (UserID, Status)

-- Unique index
CREATE UNIQUE INDEX UX_Users_Email
ON Users (Email)

-- Index with included columns (covers query entirely — no table lookup needed)
CREATE INDEX IX_Orders_Date_Covering
ON Orders (OrderDate)
INCLUDE (UserID, TotalAmount, Status)
```

### View and Drop Indexes

```sql
-- View all indexes on a table
SELECT
    i.name       AS IndexName,
    i.type_desc  AS IndexType,
    c.name       AS ColumnName
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
JOIN sys.columns c        ON ic.object_id = c.object_id AND ic.column_id = c.column_id
WHERE i.object_id = OBJECT_ID('Orders')

-- Drop an index
DROP INDEX IX_Orders_UserID ON Orders
```

> ⚠️ Too many indexes slow down INSERT/UPDATE/DELETE. Index columns you frequently filter or join on — not every column.

---

## Step 10 — Stored Procedures

Stored procedures are saved SQL blocks you execute by name — like reusable functions for your database.

### Why Use Stored Procedures?

- ✅ Reusable — write once, call anywhere
- ✅ Better performance — execution plan is cached
- ✅ Security — grant EXECUTE without exposing tables directly
- ✅ Reduced network traffic — one call instead of many queries

### Basic Stored Procedure

```sql
-- Get all active users
CREATE PROCEDURE sp_GetActiveUsers
AS
BEGIN
    SELECT UserID, FirstName, LastName, Email
    FROM Users
    WHERE IsActive = 1
    ORDER BY FirstName
END
GO

-- Execute it
EXEC sp_GetActiveUsers
```

### Procedure with Parameters

```sql
CREATE PROCEDURE sp_GetOrdersByUser
    @UserID  INT,
    @Status  VARCHAR(20) = NULL    -- optional, defaults to NULL
AS
BEGIN
    SELECT
        o.OrderID,
        o.OrderDate,
        o.TotalAmount,
        o.Status
    FROM Orders o
    WHERE o.UserID = @UserID
      AND (@Status IS NULL OR o.Status = @Status)
    ORDER BY o.OrderDate DESC
END
GO

-- Usage
EXEC sp_GetOrdersByUser @UserID = 1
EXEC sp_GetOrdersByUser @UserID = 1, @Status = 'Confirmed'
```

### Procedure with OUTPUT Parameter

```sql
CREATE PROCEDURE sp_CreateOrder
    @UserID      INT,
    @TotalAmount DECIMAL(10,2),
    @NewOrderID  INT OUTPUT
AS
BEGIN
    INSERT INTO Orders (UserID, TotalAmount, Status)
    VALUES (@UserID, @TotalAmount, 'Pending')

    SET @NewOrderID = SCOPE_IDENTITY()
END
GO

-- Execute and capture the new ID
DECLARE @CreatedOrderID INT
EXEC sp_CreateOrder
    @UserID = 1,
    @TotalAmount = 5000.00,
    @NewOrderID = @CreatedOrderID OUTPUT

SELECT @CreatedOrderID AS NewOrderID
```

### Modify and Drop

```sql
-- Modify existing procedure
ALTER PROCEDURE sp_GetActiveUsers
AS
BEGIN
    SELECT UserID, FirstName, LastName, Email, CreatedAt
    FROM Users
    WHERE IsActive = 1
    ORDER BY CreatedAt DESC
END
GO

-- Drop procedure
DROP PROCEDURE sp_GetActiveUsers
```

---

## Step 11 — Views

A **View** is a saved SELECT query you can query like a table. It runs the query fresh each time — it doesn't store data.

### Why Use Views?

- ✅ Simplify complex JOINs — query a view instead of rewriting joins every time
- ✅ Security — expose only specific columns to specific users
- ✅ Consistency — everyone uses the same query logic

```sql
-- Create a view for order summaries
CREATE VIEW vw_OrderSummary AS
SELECT
    o.OrderID,
    u.FirstName + ' ' + u.LastName  AS CustomerName,
    u.Email,
    o.OrderDate,
    o.TotalAmount,
    o.Status,
    COUNT(oi.OrderItemID)           AS ItemCount
FROM Orders o
INNER JOIN Users       u  ON o.UserID  = u.UserID
INNER JOIN OrderItems  oi ON o.OrderID = oi.OrderID
GROUP BY
    o.OrderID, u.FirstName, u.LastName,
    u.Email, o.OrderDate, o.TotalAmount, o.Status
GO

-- Use the view like a regular table
SELECT * FROM vw_OrderSummary WHERE Status = 'Confirmed'
SELECT * FROM vw_OrderSummary ORDER BY TotalAmount DESC

-- Drop a view
DROP VIEW vw_OrderSummary
```

---

## Step 12 — Triggers

A **Trigger** automatically runs SQL code when an INSERT, UPDATE, or DELETE happens on a table — without the caller doing anything extra.

### Common Use Cases

- Audit trail — log every change to a history table
- Auto-update — set `UpdatedAt` timestamp on every UPDATE
- Business rules — enforce complex validations

```sql
-- Trigger: auto-set UpdatedAt when a User row changes
CREATE TRIGGER trg_Users_UpdatedAt
ON Users
AFTER UPDATE
AS
BEGIN
    UPDATE Users
    SET UpdatedAt = GETDATE()
    WHERE UserID IN (SELECT UserID FROM inserted)
END
GO

-- Audit table to log deleted orders
CREATE TABLE OrdersAudit (
    AuditID   INT       IDENTITY(1,1) PRIMARY KEY,
    OrderID   INT,
    UserID    INT,
    DeletedAt DATETIME  DEFAULT GETDATE()
)
GO

-- Trigger: log every deleted order to audit table
CREATE TRIGGER trg_Orders_DeleteAudit
ON Orders
AFTER DELETE
AS
BEGIN
    INSERT INTO OrdersAudit (OrderID, UserID, DeletedAt)
    SELECT OrderID, UserID, GETDATE()
    FROM deleted
END
GO

-- Drop a trigger
DROP TRIGGER trg_Users_UpdatedAt
```

---

## Step 13 — Transactions

A **Transaction** groups multiple SQL statements into one atomic operation — either ALL succeed or ALL are rolled back (undone).

### Why Use Transactions?

> Example: Placing an order requires (1) creating the order, (2) adding order items, (3) reducing stock. If step 3 fails, steps 1 and 2 must be undone.

```sql
-- Basic transaction
BEGIN TRANSACTION

    INSERT INTO Orders (UserID, TotalAmount, Status)
    VALUES (1, 5000.00, 'Pending')

    UPDATE Products
    SET Stock = Stock - 1
    WHERE ProductID = 1

COMMIT TRANSACTION    -- saves all changes
-- OR
ROLLBACK TRANSACTION  -- undoes all changes if something failed


-- Transaction with TRY/CATCH error handling
BEGIN TRANSACTION

BEGIN TRY

    -- Step 1: Create order
    INSERT INTO Orders (UserID, TotalAmount, Status)
    VALUES (1, 75000.00, 'Confirmed')

    DECLARE @NewOrderID INT = SCOPE_IDENTITY()

    -- Step 2: Add order items
    INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
    VALUES (@NewOrderID, 1, 1, 75000.00)

    -- Step 3: Reduce stock
    UPDATE Products
    SET Stock = Stock - 1
    WHERE ProductID = 1

    COMMIT TRANSACTION
    PRINT 'Order placed successfully!'

END TRY
BEGIN CATCH

    ROLLBACK TRANSACTION
    PRINT 'Error: ' + ERROR_MESSAGE()

END CATCH
```

---

## Step 14 — Backup and Restore

Azure SQL manages backups **automatically** — but understanding how it works is essential for production systems.

### Automatic Backup Schedule

| Backup Type | Frequency | Retention |
|-------------|-----------|-----------|
| **Full backup** | Once per week | 7–35 days |
| **Differential backup** | Every 12–24 hours | Same as full |
| **Transaction log backup** | Every 5–10 minutes | Same as full |

> 💡 You can restore to **any point in time** within the retention window — down to the minute!

### Point-in-Time Restore (Portal)

```
Go to your SQL Database in Azure Portal
→ Left menu: "Restore"
→ Select: "Point-in-time"
→ Choose date and time to restore to
→ Enter new database name (restores as a NEW database)
→ Click "Review + create" → "Create"
```

### Point-in-Time Restore (Azure CLI)

```bash
az sql db restore \
    --resource-group MyProject-RG \
    --server my-sql-server-prod \
    --name MyAppDB-Restored \
    --source-database MyAppDB \
    --time "2024-06-15T10:30:00"
```

### Manual Export (BACPAC — schema + data)

Useful for migration, archiving, or moving to another server.

```
Go to your SQL Database in Azure Portal
→ Left menu: "Export"
→ Storage account: select or create one
→ File name: MyAppDB-backup-2024.bacpac
→ Authentication: enter your admin password
→ Click "OK"
```

### Import a BACPAC

```
Go to your SQL Server in Azure Portal
→ Left menu: "Import database"
→ Select .bacpac file from Storage
→ Enter database name and credentials
→ Click "OK"
```

### Change Backup Retention Period

```bash
# Extend retention to 30 days (default is 7)
az sql db str-policy set \
    --resource-group MyProject-RG \
    --server my-sql-server-prod \
    --database MyAppDB \
    --retention-days 30
```

---

## Step 15 — Security and Users

### Create Database Users

```sql
-- Step 1: Create a LOGIN at SERVER level
-- Run on MASTER database
CREATE LOGIN AppUser
WITH PASSWORD = 'AppUs3r!Pass#99'
GO

-- Step 2: Create a USER at DATABASE level
-- Run on MyAppDB
USE MyAppDB
GO
CREATE USER AppUser FOR LOGIN AppUser
GO
```

### Assign Roles

```sql
-- Built-in roles:
-- db_owner       → full control
-- db_datareader  → SELECT only (read-only)
-- db_datawriter  → INSERT, UPDATE, DELETE (no SELECT)
-- db_ddladmin    → create/alter/drop tables

-- Grant read-only
ALTER ROLE db_datareader ADD MEMBER AppUser

-- Grant read + write
ALTER ROLE db_datareader ADD MEMBER AppUser
ALTER ROLE db_datawriter ADD MEMBER AppUser

-- Grant full access
ALTER ROLE db_owner ADD MEMBER AppUser
```

### Grant Specific Permissions

```sql
-- Grant SELECT on one table only
GRANT SELECT ON Users TO AppUser

-- Grant EXECUTE on stored procedure only
GRANT EXECUTE ON sp_GetActiveUsers TO AppUser

-- Revoke a permission
REVOKE SELECT ON Users FROM AppUser

-- Deny a permission (overrides even role grants)
DENY DELETE ON Orders FROM AppUser
```

### View All Users and Roles

```sql
-- List all users in current database
SELECT name, type_desc, create_date
FROM sys.database_principals
WHERE type IN ('S', 'U', 'G')
ORDER BY name

-- List role memberships
SELECT
    r.name  AS RoleName,
    m.name  AS MemberName
FROM sys.database_role_members rm
JOIN sys.database_principals r ON rm.role_principal_id   = r.principal_id
JOIN sys.database_principals m ON rm.member_principal_id = m.principal_id
ORDER BY r.name
```

---

## Step 16 — Monitor and Performance

### Check Active Connections

```sql
SELECT
    session_id,
    login_name,
    host_name,
    program_name,
    status,
    cpu_time,
    memory_usage
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
ORDER BY cpu_time DESC
```

### Find Slow / Running Queries

```sql
-- Currently running queries
SELECT
    r.session_id,
    r.status,
    r.cpu_time,
    r.total_elapsed_time / 1000  AS elapsed_seconds,
    r.logical_reads,
    t.text                       AS query_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50
ORDER BY r.total_elapsed_time DESC

-- Kill a blocking session (use carefully!)
KILL 57   -- replace 57 with actual session_id
```

### Check Table Sizes

```sql
SELECT
    t.name                           AS TableName,
    p.rows                           AS RowCount,
    SUM(a.total_pages) * 8 / 1024.0 AS TotalSizeMB,
    SUM(a.used_pages)  * 8 / 1024.0 AS UsedSizeMB
FROM sys.tables t
JOIN sys.indexes        i  ON t.object_id = i.object_id
JOIN sys.partitions     p  ON i.object_id = p.object_id AND i.index_id = p.index_id
JOIN sys.allocation_units a ON p.partition_id = a.container_id
WHERE t.is_ms_shipped = 0
GROUP BY t.name, p.rows
ORDER BY TotalSizeMB DESC
```

### Measure Query Performance

```sql
-- Show I/O and time stats for a query
SET STATISTICS IO ON
SET STATISTICS TIME ON

SELECT * FROM Orders WHERE UserID = 1

SET STATISTICS IO OFF
SET STATISTICS TIME OFF
-- Look for "Table Scan" in results → signals a missing index
```

### Azure Portal Monitoring

```
Go to your SQL Database in Azure Portal:
→ "Query Performance Insight"    — top slow queries ranked by CPU/duration
→ "Metrics"                      — CPU%, DTU usage, storage graphs
→ "Intelligent Performance"      — automatic index recommendations
→ "Alerts"                       — set up email/SMS alerts on thresholds
→ "Diagnostic settings"          — send logs to Log Analytics / Storage
```

---

## Step 17 — Alter and Maintain Tables

### Add / Modify / Drop Columns

```sql
-- Add a new nullable column
ALTER TABLE Users
ADD DateOfBirth DATE NULL

-- Add column with default
ALTER TABLE Users
ADD LastLoginAt DATETIME NULL DEFAULT GETDATE()

-- Change column data type
ALTER TABLE Users
ALTER COLUMN Phone NVARCHAR(20) NULL

-- Rename a column
EXEC sp_rename 'Users.Phone', 'PhoneNumber', 'COLUMN'

-- Drop a column
ALTER TABLE Users
DROP COLUMN DateOfBirth

-- Rename a table
EXEC sp_rename 'Users', 'AppUsers'
```

### Add / Drop Constraints

```sql
-- Add CHECK constraint
ALTER TABLE Products
ADD CONSTRAINT CHK_Products_Price CHECK (Price >= 0)

-- Add DEFAULT constraint
ALTER TABLE Orders
ADD CONSTRAINT DF_Orders_Status DEFAULT 'Pending' FOR Status

-- Drop a constraint
ALTER TABLE Products
DROP CONSTRAINT CHK_Products_Price

-- View all constraints on a table
SELECT
    name       AS ConstraintName,
    type_desc  AS ConstraintType
FROM sys.objects
WHERE parent_object_id = OBJECT_ID('Products')
  AND type IN ('C', 'D', 'F', 'PK', 'UQ')
```

### Useful Utility Queries

```sql
-- List all tables
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_NAME

-- List all columns in a table
SELECT
    COLUMN_NAME,
    DATA_TYPE,
    CHARACTER_MAXIMUM_LENGTH,
    IS_NULLABLE,
    COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Users'
ORDER BY ORDINAL_POSITION

-- List all stored procedures
SELECT name, create_date, modify_date
FROM sys.procedures
ORDER BY name

-- List all views
SELECT name, create_date
FROM sys.views
ORDER BY name

-- Check total database size
SELECT
    DB_NAME()                   AS DatabaseName,
    SUM(size * 8 / 1024.0)     AS TotalSizeMB
FROM sys.database_files
```

---

## Full Example Project

Complete copy-paste script — sets up schema, indexes, a view, a stored procedure with transaction, and sample data:

```sql
-- ============================================
-- FULL SETUP SCRIPT
-- Run on: MyAppDB
-- ============================================

USE MyAppDB
GO

-- 1. USERS TABLE
CREATE TABLE Users (
    UserID     INT           IDENTITY(1,1) PRIMARY KEY,
    FirstName  NVARCHAR(50)  NOT NULL,
    LastName   NVARCHAR(50)  NOT NULL,
    Email      NVARCHAR(100) NOT NULL UNIQUE,
    Phone      VARCHAR(15)   NULL,
    IsActive   BIT           NOT NULL DEFAULT 1,
    CreatedAt  DATETIME      NOT NULL DEFAULT GETDATE(),
    UpdatedAt  DATETIME      NULL
)
GO

-- 2. PRODUCTS TABLE
CREATE TABLE Products (
    ProductID    INT            IDENTITY(1,1) PRIMARY KEY,
    ProductName  NVARCHAR(100)  NOT NULL,
    Description  NVARCHAR(500)  NULL,
    Price        DECIMAL(10,2)  NOT NULL CHECK (Price >= 0),
    Stock        INT            NOT NULL DEFAULT 0,
    CreatedAt    DATETIME       NOT NULL DEFAULT GETDATE()
)
GO

-- 3. ORDERS TABLE
CREATE TABLE Orders (
    OrderID      INT            IDENTITY(1,1) PRIMARY KEY,
    UserID       INT            NOT NULL,
    OrderDate    DATETIME       NOT NULL DEFAULT GETDATE(),
    TotalAmount  DECIMAL(10,2)  NOT NULL,
    Status       VARCHAR(20)    NOT NULL DEFAULT 'Pending',

    CONSTRAINT FK_Orders_Users
        FOREIGN KEY (UserID) REFERENCES Users(UserID)
        ON DELETE CASCADE ON UPDATE CASCADE
)
GO

-- 4. ORDER ITEMS TABLE
CREATE TABLE OrderItems (
    OrderItemID  INT            IDENTITY(1,1) PRIMARY KEY,
    OrderID      INT            NOT NULL,
    ProductID    INT            NOT NULL,
    Quantity     INT            NOT NULL CHECK (Quantity > 0),
    UnitPrice    DECIMAL(10,2)  NOT NULL,

    CONSTRAINT FK_OrderItems_Orders
        FOREIGN KEY (OrderID) REFERENCES Orders(OrderID)
        ON DELETE CASCADE,

    CONSTRAINT FK_OrderItems_Products
        FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
)
GO

-- 5. INDEXES
CREATE INDEX IX_Orders_UserID        ON Orders (UserID)
CREATE INDEX IX_Orders_Status        ON Orders (Status)
CREATE INDEX IX_OrderItems_OrderID   ON OrderItems (OrderID)
CREATE INDEX IX_OrderItems_ProductID ON OrderItems (ProductID)
GO

-- 6. VIEW
CREATE VIEW vw_OrderSummary AS
SELECT
    o.OrderID,
    u.FirstName + ' ' + u.LastName AS CustomerName,
    u.Email,
    o.OrderDate,
    o.TotalAmount,
    o.Status
FROM Orders o
INNER JOIN Users u ON o.UserID = u.UserID
GO

-- 7. STORED PROCEDURE WITH TRANSACTION
CREATE PROCEDURE sp_PlaceOrder
    @UserID      INT,
    @ProductID   INT,
    @Quantity    INT,
    @NewOrderID  INT OUTPUT
AS
BEGIN
    BEGIN TRANSACTION
    BEGIN TRY
        DECLARE @Price DECIMAL(10,2)
        SELECT @Price = Price FROM Products WHERE ProductID = @ProductID

        INSERT INTO Orders (UserID, TotalAmount, Status)
        VALUES (@UserID, @Price * @Quantity, 'Pending')

        SET @NewOrderID = SCOPE_IDENTITY()

        INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
        VALUES (@NewOrderID, @ProductID, @Quantity, @Price)

        UPDATE Products SET Stock = Stock - @Quantity
        WHERE ProductID = @ProductID

        COMMIT TRANSACTION
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION
        PRINT 'Error: ' + ERROR_MESSAGE()
        SET @NewOrderID = -1
    END CATCH
END
GO

-- 8. INSERT SAMPLE DATA
INSERT INTO Users (FirstName, LastName, Email, Phone)
VALUES
    ('Raj',   'Sharma',  'raj@example.com',   '+91-9876543210'),
    ('Priya', 'Patel',   'priya@example.com', '+91-9123456789'),
    ('Amit',  'Gupta',   'amit@example.com',  '+91-9988776655')
GO

INSERT INTO Products (ProductName, Description, Price, Stock)
VALUES
    ('Laptop Pro',     '15-inch i7 Laptop',      75000.00, 50),
    ('Wireless Mouse', 'Ergonomic USB Mouse',       999.00, 200),
    ('USB-C Hub',      '7-in-1 USB-C Adapter',    2499.00, 100)
GO

-- 9. PLACE AN ORDER USING STORED PROCEDURE
DECLARE @OrderID INT
EXEC sp_PlaceOrder
    @UserID     = 1,
    @ProductID  = 1,
    @Quantity   = 1,
    @NewOrderID = @OrderID OUTPUT
SELECT @OrderID AS NewOrderID
GO

-- 10. VERIFY
SELECT 'Users'      AS TableName, COUNT(*) AS Rows FROM Users      UNION ALL
SELECT 'Products'   AS TableName, COUNT(*) AS Rows FROM Products    UNION ALL
SELECT 'Orders'     AS TableName, COUNT(*) AS Rows FROM Orders      UNION ALL
SELECT 'OrderItems' AS TableName, COUNT(*) AS Rows FROM OrderItems
GO
```

---

## Common Errors

### ❌ Cannot connect — firewall block
```
Client with IP '203.x.x.x' is not allowed to access the server
```
**Fix:** Add your IP in Portal → SQL Server → Networking → Firewall Rules

---

### ❌ Database already exists
```
Database 'MyAppDB' already exists. Choose a different database name.
```
**Fix:** Use a different name or drop the existing DB first.

---

### ❌ Invalid object name
```
Invalid object name 'Users'
```
**Fix:** Make sure you're connected to `MyAppDB`, not `master`. Run `USE MyAppDB` first.

---

### ❌ Foreign key constraint violation on INSERT
```
The INSERT statement conflicted with the FOREIGN KEY constraint
```
**Fix:** Insert the parent record first (e.g., insert into `Users` before `Orders`).

---

### ❌ Cannot drop — FK constraint exists
```
Could not drop object 'Users' because it is referenced by a FOREIGN KEY constraint
```
**Fix:** Drop the FK or the child table first.
```sql
ALTER TABLE Orders DROP CONSTRAINT FK_Orders_Users
DROP TABLE Users
```

---

### ❌ Cannot insert NULL
```
Cannot insert the value NULL into column 'Email'
```
**Fix:** Provide a value for all `NOT NULL` columns.

---

### ❌ Arithmetic overflow
```
Arithmetic overflow error converting numeric to data type numeric
```
**Fix:** Check your `DECIMAL(p,s)` — `DECIMAL(10,2)` only stores up to `99999999.99`.

---

### ❌ Deadlock
```
Transaction (Process ID 55) was deadlocked on lock resources
```
**Fix:** Use `TRY/CATCH` with retry logic. Access tables in the same order in all transactions.

---

## Best Practices

### Schema Design
- ✅ Use `NVARCHAR` for any text that may contain non-English characters
- ✅ Add `CreatedAt DATETIME DEFAULT GETDATE()` to every table
- ✅ Add `UpdatedAt DATETIME NULL` and update it via trigger
- ✅ Use `DECIMAL(10,2)` for money — never `FLOAT`
- ✅ Name all constraints explicitly (`FK_Orders_Users`, `CHK_Products_Price`)
- ✅ Always add `CHECK` constraints on numeric columns (`Price >= 0`, `Quantity > 0`)

### Performance
- ✅ Index every foreign key column
- ✅ Use `TOP n` when you only need a few rows
- ✅ Use stored procedures for frequently executed queries
- ✅ Use covering indexes (`INCLUDE`) for read-heavy queries
- ❌ Don't over-index — each index slows down writes
- ❌ Avoid `SELECT *` in production — always name your columns

### Security
- ✅ Use least-privilege roles — `db_datareader` for read-only apps
- ✅ Store connection strings in Azure Key Vault — never in source code
- ✅ Create separate logins per application
- ✅ Enable Azure Defender for SQL in production
- ❌ Never use the admin login in your application connection string
- ❌ Never store plain-text passwords in any table

### Azure
- ✅ Use Serverless tier for dev/test (auto-pauses, saves cost)
- ✅ Choose a region close to your users for lower latency
- ✅ Set backup retention to at least 14 days for production
- ✅ Set up monitoring alerts for CPU%, DTU usage, and storage
- ❌ Never open firewall to `0.0.0.0–255.255.255.255` in production

---

## FAQ

**Q: Can I create multiple databases on one server?**
> Yes! One Azure SQL Server can host many databases. Each is billed separately.

**Q: What's the difference between SQL Server and SQL Database in Azure?**
> "SQL Server" is the logical server (host). "SQL Database" is the actual database inside it. You need both.

**Q: Can I use the free tier?**
> Azure offers a free SQL Database (32 GB) on a free account for 12 months. After that, Serverless is the cheapest option.

**Q: What's the difference between DELETE and TRUNCATE?**
> `DELETE` removes rows one by one, fires triggers, and can be rolled back. `TRUNCATE` removes all rows at once, doesn't fire triggers, resets `IDENTITY` counter, and is much faster for clearing a table.

**Q: How do I copy a table's data to another table?**
```sql
-- Into existing table
INSERT INTO TableB SELECT * FROM TableA

-- Create new table (structure + data)
SELECT * INTO TableB FROM TableA

-- Create new table (structure only)
SELECT * INTO TableB FROM TableA WHERE 1 = 2
```

**Q: What is a schema in SQL Server?**
> A schema is a namespace for organizing objects. The default is `dbo` (e.g., `dbo.Users`). You can group related tables: `CREATE SCHEMA sales` then `CREATE TABLE sales.Orders (...)`.

**Q: How do I see all tables in my database?**
```sql
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE' ORDER BY TABLE_NAME
```

**Q: How do I rename a table or column?**
```sql
EXEC sp_rename 'Users', 'AppUsers'                -- rename table
EXEC sp_rename 'Users.Phone', 'PhoneNumber', 'COLUMN'  -- rename column
```

**Q: How do I add a column to an existing table?**
```sql
ALTER TABLE Users ADD DateOfBirth DATE NULL
```

---

## 📖 Further Reading

- [Microsoft Docs — Quickstart: Create Azure SQL Database](https://docs.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart)
- [Azure SQL Pricing](https://azure.microsoft.com/en-us/pricing/details/azure-sql-database/)
- [T-SQL CREATE TABLE Reference](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-table-transact-sql)
- [Azure SQL Connection Libraries](https://docs.microsoft.com/en-us/azure/azure-sql/database/connect-query-content-reference-guide)
- [SQL Server Data Types](https://docs.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql)
- [Stored Procedures (T-SQL)](https://docs.microsoft.com/en-us/sql/relational-databases/stored-procedures/stored-procedures-database-engine)
- [Indexes — Architecture Guide](https://docs.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- [Azure SQL Security Overview](https://docs.microsoft.com/en-us/azure/azure-sql/database/security-overview)
- [Automated Backups — Azure SQL](https://docs.microsoft.com/en-us/azure/azure-sql/database/automated-backups-overview)
- [Performance Monitoring](https://docs.microsoft.com/en-us/azure/azure-sql/database/monitor-tune-overview)

---

*Generated with ❤️ — Feel free to contribute or raise issues!*
