# Experiment 10 [CO5]

**Generate at least 10,000 rows. Write five queries, analyze using execution plans, optimize using indexing and query rewriting, and compare execution time before and after optimization.**

## 10.1 Table Creation and Data Generation

```sql
CREATE DATABASE PerfDB;
GO
USE PerfDB;
GO

CREATE TABLE OnlineOrders (
    OrderID       INT PRIMARY KEY,
    CustomerID    INT NOT NULL,
    OrderDate     DATE NOT NULL,
    Amount        DECIMAL(10,2) NOT NULL,
    Category      VARCHAR(20) NOT NULL,
    City          VARCHAR(20) NOT NULL,
    Status        VARCHAR(10) NOT NULL,
    CustomerName  VARCHAR(50) NOT NULL
);
GO

SET NOCOUNT ON;
WITH N AS (
    SELECT TOP (12000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
INSERT INTO OnlineOrders
SELECT n,
       ABS(CHECKSUM(NEWID())) % 1500 + 1,
       DATEADD(DAY, ABS(CHECKSUM(NEWID())) % 1095, '2024-01-01'),
       ABS(CHECKSUM(NEWID())) % 24900 + 100,
       CHOOSE(ABS(CHECKSUM(NEWID())) % 6 + 1, 'Electronics','Books','Clothing','Groceries','Furniture','Toys'),
       CHOOSE(ABS(CHECKSUM(NEWID())) % 7 + 1, 'Pune','Mumbai','Delhi','Chennai','Bengaluru','Kolkata','Hyderabad'),
       CHOOSE(ABS(CHECKSUM(NEWID())) % 4 + 1, 'Delivered','Shipped','Pending','Cancelled'),
       CHOOSE(ABS(CHECKSUM(NEWID())) % 10 + 1, 'Aarav','Isha','Rohan','Ananya','Kabir','Diya','Aditya','Meera','Pooja','Rahul')
       + ' ' + CHOOSE(ABS(CHECKSUM(NEWID())) % 8 + 1, 'Sharma','Verma','Mehta','Iyer','Nair','Singh','Patel','Rao')
FROM N;
SET NOCOUNT OFF;

SELECT COUNT(*) AS TotalOrders FROM OnlineOrders;
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

**Output:**

| TotalOrders |
|---|
| 12000 |

## 10.2 Query 1 — Function on an indexed column

Total orders and amount for the year 2025.

### Before

```sql
SELECT COUNT(*) AS Orders2025, SUM(Amount) AS TotalAmount
FROM OnlineOrders
WHERE YEAR(OrderDate) = 2025;
```

**Output:**

| Orders2025 | TotalAmount |
|---|---|
| 4043 | 51116999 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 124, physical reads 0.
   CPU time = 15 ms,  elapsed time = 19 ms.
Plan:  Stream Aggregate <- Clustered Index Scan [PK_OnlineOrders]  (cost: 100%)
```

Applying YEAR() to the column makes the predicate non-sargable, so no index on OrderDate could be used. The query is rewritten as a date range and a covering index is added.

### After

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Date ON OnlineOrders (OrderDate) INCLUDE (Amount);
GO
SELECT COUNT(*) AS Orders2025, SUM(Amount) AS TotalAmount
FROM OnlineOrders
WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01';
```

**Output:**

| Orders2025 | TotalAmount |
|---|---|
| 4043 | 51116999 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 10, physical reads 0.
   CPU time = 0 ms,  elapsed time = 3 ms.
Plan:  Stream Aggregate <- Index Seek [IX_Orders_Date]  (cost: 100%)
```

## 10.3 Query 2 — Lookup by customer

### Before

```sql
SELECT OrderID, CustomerID, Amount, Status
FROM OnlineOrders
WHERE CustomerID = 742
ORDER BY OrderDate;
```

**Output:**

| OrderID | CustomerID | Amount | Status |
|---|---|---|---|
| 4727 | 742 | 585 | Shipped |
| 6096 | 742 | 22495 | Shipped |
| 6735 | 742 | 24842 | Cancelled |
| 2548 | 742 | 272 | Pending |
| 7774 | 742 | 22939 | Delivered |

*(8 rows returned, first 5 shown)*

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 124, physical reads 0.
   CPU time = 0 ms,  elapsed time = 11 ms.
Plan:  Sort <- Clustered Index Scan [PK_OnlineOrders]  (cost: 100%)
```

### After

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer
    ON OnlineOrders (CustomerID)
    INCLUDE (OrderDate, Amount, Status);
GO
SELECT OrderID, CustomerID, Amount, Status
FROM OnlineOrders
WHERE CustomerID = 742
ORDER BY OrderDate;
```

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 3, physical reads 0.
   CPU time = 0 ms,  elapsed time = 0 ms.
Plan:  Sort <- Index Seek [IX_Orders_Customer]  (cost: 100%)
```

## 10.4 Query 3 — Grouped revenue of delivered orders

### Before

```sql
SELECT TOP 5 City, Category, COUNT(*) AS Orders, SUM(Amount) AS Revenue
FROM OnlineOrders
WHERE Status = 'Delivered'
GROUP BY City, Category
ORDER BY Revenue DESC;
```

**Output:**

| City | Category | Orders | Revenue |
|---|---|---|---|
| Pune | Toys | 88 | 1148580 |
| Bengaluru | Furniture | 82 | 1143344 |
| Pune | Clothing | 87 | 1104046 |
| Kolkata | Clothing | 77 | 1077853 |
| Bengaluru | Electronics | 79 | 1066238 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 124, physical reads 0.
   CPU time = 16 ms,  elapsed time = 24 ms.
Plan:  Top N Sort <- Hash Match (Aggregate) <- Clustered Index Scan  (cost: 100%)
```

### After

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Status
    ON OnlineOrders (Status)
    INCLUDE (City, Category, Amount);
GO
-- same query re-run
```

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 15, physical reads 0.
   CPU time = 0 ms,  elapsed time = 8 ms.
Plan:  Top N Sort <- Hash Match (Aggregate) <- Index Seek [IX_Orders_Status]  (cost: 100%)
```

## 10.5 Query 4 — Orders above the average of their category

### Before (correlated subquery)

```sql
SELECT COUNT(*) AS AboveAvg
FROM OnlineOrders o
WHERE Amount > (SELECT AVG(Amount) FROM OnlineOrders i WHERE i.Category = o.Category);
```

**Output:**

| AboveAvg |
|---|
| 6011 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 2, logical reads 248, physical reads 0.
Table 'Worktable'. Scan count 12000, logical reads 24006, physical reads 0.
   CPU time = 156 ms,  elapsed time = 171 ms.
Plan:  Stream Aggregate <- Nested Loops <- Clustered Index Scan [o]
                                       <- Table Spool <- Stream Aggregate  (re-evaluated per row)
```

### After (rewritten with a window function)

```sql
SELECT COUNT(*) AS AboveAvg
FROM (
    SELECT Amount, AVG(Amount) OVER (PARTITION BY Category) AS CatAvg
    FROM OnlineOrders
) t
WHERE Amount > CatAvg;
```

**Output:**

| AboveAvg |
|---|
| 6011 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 124, physical reads 0.
Table 'Worktable'. Scan count 0, logical reads 0, physical reads 0.
   CPU time = 16 ms,  elapsed time = 21 ms.
Plan:  Stream Aggregate <- Filter <- Window Spool <- Sort <- Clustered Index Scan  (single pass)
```

## 10.6 Query 5 — OR condition on Status and a range on Amount

### Before

```sql
SELECT COUNT(*) AS PendingOrCancelledHighValue
FROM OnlineOrders
WHERE (Status = 'Pending' OR Status = 'Cancelled') AND Amount > 20000;
```

**Output:**

| PendingOrCancelledHighValue |
|---|
| 1277 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 1, logical reads 124, physical reads 0.
   CPU time = 0 ms,  elapsed time = 12 ms.
Plan:  Stream Aggregate <- Clustered Index Scan [PK_OnlineOrders]  (cost: 100%)
```

### After (composite index and UNION ALL rewrite)

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Status_Amount
    ON OnlineOrders (Status, Amount);
GO
SELECT SUM(c) AS PendingOrCancelledHighValue
FROM (
    SELECT COUNT(*) AS c FROM OnlineOrders WHERE Status = 'Pending'   AND Amount > 20000
    UNION ALL
    SELECT COUNT(*)      FROM OnlineOrders WHERE Status = 'Cancelled' AND Amount > 20000
) x;
```

**Output:**

| PendingOrCancelledHighValue |
|---|
| 1277 |

**Messages:**

```text
Table 'OnlineOrders'. Scan count 2, logical reads 8, physical reads 0.
   CPU time = 0 ms,  elapsed time = 2 ms.
Plan:  Stream Aggregate <- Concatenation <- Index Seek [IX_Orders_Status_Amount] x 2  (cost: 100%)
```

## 10.7 Before and After Comparison

**Output:**

| Query | Optimization | Reads Before | Reads After | Time Before (ms) | Time After (ms) |
|---|---|---|---|---|---|
| Q1 | Sargable range + covering index on OrderDate | 124 | 10 | 19 | 3 |
| Q2 | Covering index on CustomerID | 124 | 3 | 11 | 0 |
| Q3 | Index on Status with INCLUDE columns | 124 | 15 | 24 | 8 |
| Q4 | Correlated subquery -> window function | 24254 | 124 | 171 | 21 |
| Q5 | Composite index + UNION ALL rewrite | 124 | 8 | 12 | 2 |

**Conclusion:** On 12000 rows, every query that scanned the whole table was improved. Indexes (covering and composite) removed full scans for Queries 1, 2, 3 and 5, while rewriting the non-sargable YEAR() predicate and the correlated subquery let the optimizer use an index and a single pass respectively. The largest gain came from Query 4, where logical reads dropped from over 24,000 to 124.
