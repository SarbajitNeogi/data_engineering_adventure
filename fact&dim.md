# Dimensional Modeling Guide: Facts, Dimensions & Schema Design

A complete reference for **Fact Tables**, **Dimension Tables**, **Factless Fact Tables**, **Snapshot Facts**, and related data warehousing concepts — with examples.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Fact Tables](#2-fact-tables)
   - [2.1 What is a Fact Table](#21-what-is-a-fact-table)
   - [2.2 Types of Measures](#22-types-of-measures)
   - [2.3 Grain](#23-grain)
3. [Types of Fact Tables](#3-types-of-fact-tables)
   - [3.1 Transaction Fact Table](#31-transaction-fact-table)
   - [3.2 Periodic Snapshot Fact Table](#32-periodic-snapshot-fact-table)
   - [3.3 Accumulating Snapshot Fact Table](#33-accumulating-snapshot-fact-table)
   - [3.4 Factless Fact Table](#34-factless-fact-table)
4. [Dimension Tables](#4-dimension-tables)
   - [4.1 What is a Dimension Table](#41-what-is-a-dimension-table)
   - [4.2 Types of Dimensions](#42-types-of-dimensions)
   - [4.3 Slowly Changing Dimensions (SCD)](#43-slowly-changing-dimensions-scd)
5. [Schema Designs](#5-schema-designs)
   - [5.1 Star Schema](#51-star-schema)
   - [5.2 Snowflake Schema](#52-snowflake-schema)
   - [5.3 Galaxy / Fact Constellation Schema](#53-galaxy--fact-constellation-schema)
6. [Bridge Tables](#6-bridge-tables)
7. [Quick Comparison Tables](#7-quick-comparison-tables)
8. [Best Practices](#8-best-practices)

---

## 1. Introduction

Dimensional modeling (popularized by **Ralph Kimball**) organizes data warehouse data into two main table types:

- **Fact tables** — store measurable, quantitative business events (sales amount, quantity, duration).
- **Dimension tables** — store descriptive context (who, what, where, when, how) that gives meaning to facts.

Together they form **schemas** (Star, Snowflake, Galaxy) optimized for analytical queries and reporting (OLAP), as opposed to normalized OLTP schemas used for transactional systems.

---

## 2. Fact Tables

### 2.1 What is a Fact Table

A **fact table** sits at the center of a star schema and stores:

- **Foreign keys** referencing dimension tables (`date_key`, `product_key`, `store_key`, etc.)
- **Measures** — numeric values you aggregate (`SUM`, `AVG`, `COUNT`, etc.)
- Optionally a **degenerate dimension** (e.g., `order_number`) — an attribute stored in the fact table without its own dimension table.

**Example — `fact_sales`**

| date_key | product_key | store_key | customer_key | order_number | quantity_sold | unit_price | sales_amount |
|----------|-------------|-----------|--------------|--------------|----------------|------------|---------------|
| 20260101 | P1001       | S05       | C2345        | ORD-9981     | 3              | 250.00     | 750.00        |
| 20260101 | P1002       | S05       | C2346        | ORD-9982     | 1              | 1200.00    | 1200.00       |

```sql
CREATE TABLE fact_sales (
    date_key       INT REFERENCES dim_date(date_key),
    product_key    INT REFERENCES dim_product(product_key),
    store_key      INT REFERENCES dim_store(store_key),
    customer_key   INT REFERENCES dim_customer(customer_key),
    order_number   VARCHAR(20),       -- degenerate dimension
    quantity_sold  INT,
    unit_price     DECIMAL(10,2),
    sales_amount   DECIMAL(12,2)
);
```

### 2.2 Types of Measures

| Measure Type        | Can Sum Across All Dimensions? | Example                          |
|----------------------|--------------------------------|-----------------------------------|
| **Additive**          | Yes, across every dimension    | `sales_amount`, `quantity_sold`  |
| **Semi-additive**     | Yes, except along time         | `account_balance`, `inventory_on_hand` |
| **Non-additive**      | No (must recalculate)          | `unit_price`, `profit_margin %`, ratios |

> Semi-additive measures are summed across products/stores but **averaged or taken at point-in-time** across dates (e.g., you don't sum a bank balance across 30 days — you take the end-of-month value).

### 2.3 Grain

The **grain** of a fact table defines exactly what one row represents. It must be declared *before* designing dimensions.

Examples of grain statements:
- "One row per line item, per order" (transaction grain)
- "One row per product, per store, per day" (daily snapshot grain)
- "One row per customer account, per month-end" (monthly snapshot grain)

> **Rule of thumb:** Always declare the grain first. Mixing grains in one fact table (e.g., daily totals + monthly totals) breaks aggregations.

---

## 3. Types of Fact Tables

There are four classic fact table types in Kimball's methodology.

### 3.1 Transaction Fact Table

Records one row **per business event/transaction**, at the moment it happens. Most granular and most common type.

**Example — `fact_order_line` (grain: one row per order line item)**

| order_line_key | date_key | product_key | customer_key | quantity | line_amount |
|----------------|----------|-------------|--------------|----------|-------------|
| 1               | 20260615 | P1001       | C2345        | 2        | 500.00      |
| 2               | 20260615 | P1010       | C2345        | 1        | 99.00       |

- Grows continuously as new events occur (insert-only).
- Rows are rarely updated once written.
- Best for detailed, drill-down analysis.

---

### 3.2 Periodic Snapshot Fact Table

Captures the **state of a measure at regular, predictable intervals** (daily, weekly, monthly) — even if no transaction occurred. Used for "as-of" or "balance" reporting.

**Example — `fact_account_balance_daily` (grain: one row per account, per day)**

| date_key | account_key | opening_balance | deposits | withdrawals | closing_balance |
|----------|-------------|------------------|----------|-------------|------------------|
| 20260618 | ACC-001     | 5000.00          | 1000.00  | 200.00      | 5800.00          |
| 20260619 | ACC-001     | 5800.00          | 0.00     | 300.00      | 5500.00          |

**Other real-world examples:**
- Daily inventory levels per warehouse per SKU.
- Monthly snapshot of subscriber counts per plan.
- Weekly snapshot of open support tickets by priority.

> Measures here are typically **semi-additive** (e.g., `closing_balance` can't be summed across dates, only across accounts on the same date).

---

### 3.3 Accumulating Snapshot Fact Table

Tracks an entity as it moves through a **well-defined process/pipeline with a fixed set of milestone steps**. The same row is **updated repeatedly** as milestones complete — unlike transaction or periodic facts, which are insert-only.

**Example — `fact_order_fulfillment` (grain: one row per order, updated as it progresses)**

| order_key | order_date_key | ship_date_key | delivery_date_key | days_to_ship | days_to_deliver | order_status |
|-----------|-----------------|----------------|---------------------|----------------|-------------------|---------------|
| ORD-9981  | 20260601        | 20260603       | 20260607            | 2              | 6                 | Delivered     |
| ORD-9982  | 20260615        | NULL           | NULL                | NULL           | NULL              | Pending Ship  |

- Each milestone (`order_date`, `ship_date`, `delivery_date`, `return_date`...) gets its own date-key column, often pointing to the same `dim_date` table multiple times (a **role-playing dimension**, see §4.2).
- NULLs (or a placeholder "not applicable" key) are used until a milestone is reached.
- Common uses: order-to-cash pipelines, loan/mortgage processing, claims processing, student admissions funnels, support-ticket lifecycle.

```sql
CREATE TABLE fact_order_fulfillment (
    order_key            INT PRIMARY KEY,
    order_date_key        INT REFERENCES dim_date(date_key),
    ship_date_key          INT REFERENCES dim_date(date_key),
    delivery_date_key      INT REFERENCES dim_date(date_key),
    days_to_ship           INT,
    days_to_deliver        INT,
    order_status           VARCHAR(20)
);

-- Later, when the order ships:
UPDATE fact_order_fulfillment
SET ship_date_key = 20260603,
    days_to_ship   = 2,
    order_status   = 'Shipped'
WHERE order_key = 'ORD-9982';
```

---

### 3.4 Factless Fact Table

A fact table with **no numeric measures at all** — only foreign keys to dimensions. It records that an **event happened** or that a **relationship/condition exists**. Counting the rows (`COUNT(*)`) is itself the measure.

There are two common sub-types:

#### a) Event-Tracking Factless Fact

Records that something occurred, with no associated numeric value.

**Example — `fact_student_attendance` (grain: one row per student, per class, per day attended)**

| date_key | student_key | class_key | teacher_key |
|----------|-------------|-----------|-------------|
| 20260619 | STU-101     | CLS-MATH9 | TCH-22      |
| 20260619 | STU-102     | CLS-MATH9 | TCH-22      |

To find attendance count: `SELECT COUNT(*) FROM fact_student_attendance WHERE class_key = 'CLS-MATH9'`.

#### b) Coverage / Condition Factless Fact

Records a relationship or eligibility — useful for **"what didn't happen"** analysis (e.g., which products were *promoted but not sold*).

**Example — `fact_promotion_coverage` (grain: one row per product promoted in a store, in a given week, regardless of whether it sold)**

| week_key | product_key | store_key | promotion_key |
|----------|-------------|-----------|----------------|
| 202625   | P1001       | S05       | PROMO-SUMMER   |
| 202625   | P1099       | S05       | PROMO-SUMMER   |

This lets you answer: *"Which promoted products had zero sales?"* by joining this factless table against `fact_sales` and finding non-matches — something a transaction fact table alone cannot answer.

```sql
CREATE TABLE fact_promotion_coverage (
    week_key       INT REFERENCES dim_date(week_key),
    product_key    INT REFERENCES dim_product(product_key),
    store_key      INT REFERENCES dim_store(store_key),
    promotion_key  INT REFERENCES dim_promotion(promotion_key)
);

-- Products promoted but NOT sold that week:
SELECT p.product_key
FROM fact_promotion_coverage p
LEFT JOIN fact_sales s
  ON p.product_key = s.product_key
 AND p.store_key   = s.store_key
 AND p.week_key    = s.week_key
WHERE s.product_key IS NULL;
```

---

## 4. Dimension Tables

### 4.1 What is a Dimension Table

A **dimension table** stores descriptive, mostly textual attributes that provide context to facts — answering *who, what, where, when, why, how*.

**Example — `dim_product`**

| product_key (PK) | product_id (NK) | product_name   | category   | subcategory | brand     |
|-------------------|------------------|------------------|------------|-------------|------------|
| 1001               | SKU-55821        | Wireless Mouse   | Electronics| Accessories | LogiTech   |
| 1002               | SKU-55822        | Office Chair     | Furniture  | Seating     | ErgoPlus   |

- `product_key` — **surrogate key** (warehouse-generated, meaningless integer), used for joins.
- `product_id` — **natural/business key** from the source system.
- Wide and denormalized by design — optimized for filtering/grouping, not for storage efficiency.

### 4.2 Types of Dimensions

| Dimension Type        | Description                                                                 | Example |
|------------------------|-------------------------------------------------------------------------------|----------|
| **Conformed Dimension** | Shared, identically-defined dimension used by multiple fact tables/data marts | `dim_date` used by both `fact_sales` and `fact_inventory` |
| **Junk Dimension**      | Combines several low-cardinality flags/indicators into one table to avoid clutter in the fact table | `dim_order_flags` (gift-wrapped Y/N, rush-order Y/N, payment-type) |
| **Degenerate Dimension**| A dimension attribute with no separate table — stored directly in the fact table | `order_number`, `invoice_number` |
| **Role-Playing Dimension** | One physical dimension table used multiple times in a fact table, each in a different "role" | `dim_date` used as `order_date`, `ship_date`, and `delivery_date` |
| **Conformed vs. Confirmed** | (Common typo) — always "conformed," meaning consistent meaning/keys across facts | — |

**Junk dimension example — `dim_order_flags`**

| flag_key | gift_wrapped | rush_order | payment_type |
|----------|---------------|-------------|----------------|
| 1         | Y             | N           | Credit Card    |
| 2         | N             | Y           | Cash on Delivery |

**Role-playing dimension example (SQL views aliasing the same table):**

```sql
SELECT
    f.order_key,
    od.full_date AS order_date,
    sd.full_date AS ship_date,
    dd.full_date AS delivery_date
FROM fact_order_fulfillment f
JOIN dim_date od ON f.order_date_key    = od.date_key
JOIN dim_date sd ON f.ship_date_key     = sd.date_key
JOIN dim_date dd ON f.delivery_date_key = dd.date_key;
```

### 4.3 Slowly Changing Dimensions (SCD)

Dimension attributes change over time (a customer moves city, a product is recategorized). SCD techniques handle this.

| Type | Name | Behavior | Example |
|------|------|-----------|----------|
| **Type 0** | Fixed | Attribute never changes, original value retained forever | Date of birth |
| **Type 1** | Overwrite | Old value is overwritten — no history kept | Correcting a misspelled name |
| **Type 2** | Add New Row | A new row is inserted with a new surrogate key; old row is "expired" — full history preserved | Customer address change, tracked with `effective_date`/`end_date`/`is_current` flag |
| **Type 3** | Add New Column | Adds a column to store the previous value alongside the current one — limited history | `previous_category`, `current_category` |
| **Type 4** | History Table | Current values in main dimension; full history in a separate history table | `dim_customer` + `dim_customer_history` |
| **Type 6** | Hybrid (1+2+3) | Combines overwrite, new row, and new column techniques | Common in enterprise retail DWs |

**SCD Type 2 example — `dim_customer`**

| customer_key | customer_id | name        | city      | effective_date | end_date   | is_current |
|---------------|--------------|--------------|-----------|------------------|-------------|-------------|
| 501            | C2345        | Asha Mehta  | Mumbai    | 2024-01-01       | 2026-03-14  | N           |
| 789            | C2345        | Asha Mehta  | Pune      | 2026-03-15       | 9999-12-31  | Y           |

Note `customer_id` (natural key) stays the same, but `customer_key` (surrogate key) changes — so historical facts still correctly point to "Mumbai" sales, while new facts point to "Pune."

```sql
-- Expire the old row
UPDATE dim_customer
SET end_date = '2026-03-14', is_current = 'N'
WHERE customer_id = 'C2345' AND is_current = 'Y';

-- Insert the new current row
INSERT INTO dim_customer (customer_key, customer_id, name, city, effective_date, end_date, is_current)
VALUES (789, 'C2345', 'Asha Mehta', 'Pune', '2026-03-15', '9999-12-31', 'Y');
```

---

## 5. Schema Designs

### 5.1 Star Schema

One central fact table directly connected to fully denormalized dimension tables — looks like a star.

```
             dim_date
                |
dim_customer — fact_sales — dim_product
                |
            dim_store
```

- **Pros:** Fewer joins, faster queries, simpler for BI tools.
- **Cons:** Some data redundancy in dimensions.

### 5.2 Snowflake Schema

Dimensions are further normalized into sub-dimension tables, reducing redundancy.

```
dim_product → dim_category → dim_department
```

- **Pros:** Saves storage, enforces normalization.
- **Cons:** More joins, can slow down query performance.

### 5.3 Galaxy / Fact Constellation Schema

Multiple fact tables share one or more conformed dimensions.

```
fact_sales ───┐
              ├── dim_date (conformed) ── fact_inventory
fact_returns ─┘
```

Used in enterprise data warehouses spanning multiple business processes (sales, inventory, returns) that share common dimensions like `dim_date`, `dim_product`, `dim_store`.

---

## 6. Bridge Tables

Used to resolve **many-to-many relationships** between a fact and a dimension (e.g., a bank account with multiple joint account holders).

**Example — `bridge_account_customer`**

| account_key | customer_key | ownership_pct |
|--------------|---------------|-----------------|
| ACC-001       | C2345         | 60              |
| ACC-001       | C2346         | 40              |

This sits between `fact_account_balance_daily` and `dim_customer`, allowing one account to be linked to multiple customers without duplicating fact rows.

---

## 7. Quick Comparison Tables

### Fact Table Types at a Glance

| Type | Grain | Insert/Update? | Example |
|------|--------|------------------|----------|
| Transaction Fact | One row per event | Insert-only | Order line items |
| Periodic Snapshot | One row per entity per fixed time interval | Insert-only (new row each period) | Daily account balances |
| Accumulating Snapshot | One row per process instance | Row updated repeatedly | Order fulfillment pipeline |
| Factless Fact | One row per event/condition, no measures | Insert-only | Student attendance, promotion coverage |

### Dimension Types at a Glance

| Type | Purpose |
|------|----------|
| Conformed | Shared across fact tables for consistency |
| Junk | Bundles miscellaneous flags/indicators |
| Degenerate | Lives in the fact table itself (no separate table) |
| Role-Playing | Same physical table used in multiple contexts |
| SCD Type 1/2/3/4/6 | Manages attribute changes over time |

---

## 8. Best Practices

1. **Declare grain explicitly** before building any fact table — write it down as a sentence.
2. **Use surrogate keys** for dimensions, never natural/business keys, to support SCD Type 2 history and decouple from source systems.
3. **Keep dimensions wide and denormalized** (favor star over snowflake) unless storage or governance needs require normalization.
4. Use a **conformed `dim_date`** across the entire warehouse — don't let every fact table invent its own date logic.
5. For **semi-additive measures**, document clearly which dimension they can't be summed across (usually time).
6. Use **factless fact tables** whenever you need to analyze absence/coverage, not just occurrence.
7. Prefer **accumulating snapshots** for fixed, milestone-driven pipelines; use **periodic snapshots** for state-over-time reporting; use **transaction facts** for granular drill-down.
8. Avoid mixing grains within a single fact table — if you need both daily and monthly views, build two separate fact tables (or aggregate on the fly).

---

### Summary

| Concept | One-Line Definition |
|----------|------------------------|
| **Fact Table** | Stores numeric measures + foreign keys to dimensions |
| **Dimension Table** | Stores descriptive context attributes |
| **Factless Fact Table** | Fact table with no measures — records events/coverage |
| **Transaction Fact** | One row per business event |
| **Periodic Snapshot Fact** | One row per entity per regular time interval |
| **Accumulating Snapshot Fact** | One row per process, updated as milestones complete |
| **Star Schema** | Fact table + denormalized dimensions |
| **Snowflake Schema** | Fact table + normalized, multi-level dimensions |
| **SCD** | Technique for handling dimension attribute changes over time |
| **Bridge Table** | Resolves many-to-many relationships between fact and dimension |

---
