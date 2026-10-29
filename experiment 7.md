# Experiment 7 [CO4]

**Aim:** Given a denormalized Sales table, identify functional dependencies and normalize step-by-step from UNF to 3NF and BCNF. Verify lossless join and dependency preservation using SQL queries.

## 7.1 Denormalized Sales Table

```sql
CREATE DATABASE NormalizationDB;
GO
USE NormalizationDB;
GO

CREATE TABLE Sales (
    OrderID       INT,
    OrderDate     DATE,
    CustomerID    INT,
    CustomerName  VARCHAR(50),
    CustomerCity  VARCHAR(30),
    ProductID     INT,
    ProductName   VARCHAR(50),
    Category      VARCHAR(30),
    UnitPrice     DECIMAL(10,2),
    Quantity      INT,
    SalesRepID    INT,
    SalesRepName  VARCHAR(50),
    Region        VARCHAR(20),
    CONSTRAINT PK_Sales PRIMARY KEY (OrderID, ProductID)
);
GO

INSERT INTO Sales VALUES
 (1001,'2026-08-01',1,'Aarav Sharma','Pune',101,'Wireless Mouse','Electronics',799.00,1,11,'Karan Singh','West'),
 (1001,'2026-08-01',1,'Aarav Sharma','Pune',103,'DBMS Textbook','Books',650.00,2,11,'Karan Singh','West'),
 (1002,'2026-08-03',2,'Isha Verma','Mumbai',102,'Keyboard','Electronics',3499.00,1,11,'Karan Singh','West'),
 (1003,'2026-08-10',3,'Rohan Mehta','Delhi',104,'Mixer Grinder','Appliances',2999.00,1,13,'Yash Patel','North'),
 (1003,'2026-08-10',3,'Rohan Mehta','Delhi',101,'Wireless Mouse','Electronics',799.00,2,13,'Yash Patel','North'),
 (1004,'2026-08-15',4,'Ananya Iyer','Chennai',103,'DBMS Textbook','Books',650.00,3,12,'Neha Rao','South'),
 (1004,'2026-08-15',4,'Ananya Iyer','Chennai',102,'Keyboard','Electronics',3499.00,1,12,'Neha Rao','South'),
 (1005,'2026-08-18',1,'Aarav Sharma','Pune',104,'Mixer Grinder','Appliances',2999.00,2,11,'Karan Singh','West');
```
**Messages:**

```text
Command(s) completed successfully.

(8 rows affected)
```

```sql
SELECT OrderID, CustomerName, ProductName, UnitPrice, Quantity, SalesRepName, Region
FROM Sales
ORDER BY OrderID, ProductID;
```

**Output:**

| OrderID | CustomerName | ProductName | UnitPrice | Quantity | SalesRepName | Region |
|---|---|---|---|---|---|---|
| 1001 | Aarav Sharma | Wireless Mouse | 799.0 | 1 | Karan Singh | West |
| 1001 | Aarav Sharma | DBMS Textbook | 650.0 | 2 | Karan Singh | West |
| 1002 | Isha Verma | Keyboard | 3499.0 | 1 | Karan Singh | West |
| 1003 | Rohan Mehta | Wireless Mouse | 799.0 | 2 | Yash Patel | North |
| 1003 | Rohan Mehta | Mixer Grinder | 2999.0 | 1 | Yash Patel | North |
| 1004 | Ananya Iyer | Keyboard | 3499.0 | 1 | Neha Rao | South |
| 1004 | Ananya Iyer | DBMS Textbook | 650.0 | 3 | Neha Rao | South |
| 1005 | Aarav Sharma | Mixer Grinder | 2999.0 | 2 | Karan Singh | West |

The table repeats customer, product and sales-representative details on every row, which causes update, insertion and deletion anomalies. For example, changing Aarav Sharma's city would need three rows to be updated.

## 7.2 Functional Dependencies

```text
FD1:  OrderID  ->  OrderDate, CustomerID, SalesRepID
FD2:  CustomerID  ->  CustomerName, CustomerCity
FD3:  ProductID  ->  ProductName, Category, UnitPrice
FD4:  SalesRepID  ->  SalesRepName, Region
FD5:  (OrderID, ProductID)  ->  Quantity
```

Candidate key of Sales: `(OrderID, ProductID)`. Every other attribute depends on this key, but FD1 to FD4 show that many attributes depend on only part of the key or on non-key attributes.

## 7.3 Normalization Steps

### UNF (Unnormalized Form)

In the raw order sheet one row held a whole order with a repeating group of products:

| OrderID | Customer | Products (Qty) |
|---|---|---|
| 1001 | Aarav Sharma | Wireless Mouse (1), DBMS Textbook (2) |
| 1002 | Isha Verma | Keyboard (1) |
| 1003 | Rohan Mehta | Mixer Grinder (1), Wireless Mouse (2) |
| 1004 | Ananya Iyer | DBMS Textbook (3), Keyboard (1) |
| 1005 | Aarav Sharma | Mixer Grinder (2) |

### 1NF: remove repeating groups

Each product on an order is placed in its own row so that every column holds a single atomic value. This is the Sales table created in 7.1 with the composite key (OrderID, ProductID).

### 2NF: remove partial dependencies

Customer, order and sales-rep attributes depend only on OrderID, and product attributes depend only on ProductID. These are separated from Quantity, which depends on the whole key.

```text
Orders    (OrderID, OrderDate, CustomerID, CustomerName, CustomerCity, SalesRepID, SalesRepName, Region)
Product   (ProductID, ProductName, Category, UnitPrice)
OrderItem (OrderID, ProductID, Quantity)
```

### 3NF: remove transitive dependencies

In Orders, OrderID -> CustomerID -> CustomerName and OrderID -> SalesRepID -> SalesRepName are transitive dependencies, so Customer and SalesRep are split off.

```text
Customer  (CustomerID, CustomerName, CustomerCity)
SalesRep  (SalesRepID, SalesRepName, Region)
Orders    (OrderID, OrderDate, CustomerID, SalesRepID)
Product   (ProductID, ProductName, Category, UnitPrice)
OrderItem (OrderID, ProductID, Quantity)
```

### BCNF

A relation is in BCNF when every determinant is a candidate key. In the five relations above the only determinants are CustomerID, SalesRepID, OrderID, ProductID and (OrderID, ProductID), and each one is the primary key of its own table, so the schema is already in BCNF.

## 7.4 Creating and Populating the BCNF Tables

```sql
CREATE TABLE Customer (
    CustomerID    INT PRIMARY KEY,
    CustomerName  VARCHAR(50) NOT NULL,
    CustomerCity  VARCHAR(30) NOT NULL
);
CREATE TABLE SalesRep (
    SalesRepID    INT PRIMARY KEY,
    SalesRepName  VARCHAR(50) NOT NULL,
    Region        VARCHAR(20) NOT NULL
);
CREATE TABLE Product (
    ProductID    INT PRIMARY KEY,
    ProductName  VARCHAR(50) NOT NULL,
    Category     VARCHAR(30) NOT NULL,
    UnitPrice    DECIMAL(10,2) NOT NULL
);
CREATE TABLE Orders (
    OrderID     INT PRIMARY KEY,
    OrderDate   DATE NOT NULL,
    CustomerID  INT NOT NULL REFERENCES Customer(CustomerID),
    SalesRepID  INT NOT NULL REFERENCES SalesRep(SalesRepID)
);
CREATE TABLE OrderItem (
    OrderID    INT NOT NULL REFERENCES Orders(OrderID),
    ProductID  INT NOT NULL REFERENCES Product(ProductID),
    Quantity   INT NOT NULL,
    PRIMARY KEY (OrderID, ProductID)
);
GO

INSERT INTO Customer  SELECT DISTINCT CustomerID, CustomerName, CustomerCity FROM Sales;
INSERT INTO SalesRep  SELECT DISTINCT SalesRepID, SalesRepName, Region FROM Sales;
INSERT INTO Product   SELECT DISTINCT ProductID, ProductName, Category, UnitPrice FROM Sales;
INSERT INTO Orders    SELECT DISTINCT OrderID, OrderDate, CustomerID, SalesRepID FROM Sales;
INSERT INTO OrderItem SELECT OrderID, ProductID, Quantity FROM Sales;
```

**Messages:**

```text
Command(s) completed successfully.

(4 rows affected)

(3 rows affected)

(4 rows affected)

(5 rows affected)

(8 rows affected)
```

**`SELECT * FROM Customer;`**

| CustomerID | CustomerName | CustomerCity |
|---|---|---|
| 1 | Aarav Sharma | Pune |
| 2 | Isha Verma | Mumbai |
| 3 | Rohan Mehta | Delhi |
| 4 | Ananya Iyer | Chennai |

**`SELECT * FROM SalesRep;`**

| SalesRepID | SalesRepName | Region |
|---|---|---|
| 11 | Karan Singh | West |
| 12 | Neha Rao | South |
| 13 | Yash Patel | North |

**`SELECT * FROM Product;`**

| ProductID | ProductName | Category | UnitPrice |
|---|---|---|---|
| 101 | Wireless Mouse | Electronics | 799.0 |
| 102 | Keyboard | Electronics | 3499.0 |
| 103 | DBMS Textbook | Books | 650.0 |
| 104 | Mixer Grinder | Appliances | 2999.0 |

**`SELECT * FROM Orders;`**

| OrderID | OrderDate | CustomerID | SalesRepID |
|---|---|---|---|
| 1001 | 2026-08-01 | 1 | 11 |
| 1002 | 2026-08-03 | 2 | 11 |
| 1003 | 2026-08-10 | 3 | 13 |
| 1004 | 2026-08-15 | 4 | 12 |
| 1005 | 2026-08-18 | 1 | 11 |

**`SELECT * FROM OrderItem;`**

| OrderID | ProductID | Quantity |
|---|---|---|
| 1001 | 101 | 1 |
| 1001 | 103 | 2 |
| 1002 | 102 | 1 |
| 1003 | 101 | 2 |
| 1003 | 104 | 1 |
| 1004 | 102 | 1 |
| 1004 | 103 | 3 |
| 1005 | 104 | 2 |

## 7.5 Verifying Lossless Join

The five tables are joined back together and compared with the original Sales table in both directions using `EXCEPT`. A lossless decomposition gives no extra rows and no missing rows.

```sql
WITH Rejoined AS (
    SELECT o.OrderID, o.OrderDate, o.CustomerID, cu.CustomerName, cu.CustomerCity,
           oi.ProductID, p.ProductName, p.Category, p.UnitPrice, oi.Quantity,
           o.SalesRepID, r.SalesRepName, r.Region
    FROM Orders o
    JOIN OrderItem oi ON o.OrderID = oi.OrderID
    JOIN Customer cu  ON o.CustomerID = cu.CustomerID
    JOIN Product p    ON oi.ProductID = p.ProductID
    JOIN SalesRep r   ON o.SalesRepID = r.SalesRepID
)
SELECT
  (SELECT COUNT(*) FROM Sales)    AS OriginalRows,
  (SELECT COUNT(*) FROM Rejoined) AS RejoinedRows,
  (SELECT COUNT(*) FROM (SELECT * FROM Rejoined EXCEPT SELECT * FROM Sales) x) AS ExtraRows,
  (SELECT COUNT(*) FROM (SELECT * FROM Sales EXCEPT SELECT * FROM Rejoined) y) AS MissingRows;
```

**Output:**

| OriginalRows | RejoinedRows | ExtraRows | MissingRows |
|---|---|---|---|
| 8 | 8 | 0 | 0 |

*RejoinedRows equals OriginalRows and both ExtraRows and MissingRows are 0, so no spurious tuples were created and no information was lost: the decomposition is lossless.*

## 7.6 Verifying Dependency Preservation

Each functional dependency must be checkable inside a single decomposed table. First each FD is tested on the original Sales data by looking for any determinant value that maps to more than one dependent value; zero violations means the FD holds.

```sql
SELECT 'FD1: OrderID -> OrderDate, CustomerID, SalesRepID' AS FD, COUNT(*) AS Violations
FROM (SELECT OrderID FROM Sales GROUP BY OrderID
      HAVING COUNT(DISTINCT CONCAT(OrderDate,'|',CustomerID,'|',SalesRepID)) > 1) a
UNION ALL
SELECT 'FD2: CustomerID -> CustomerName, CustomerCity', COUNT(*)
FROM (SELECT CustomerID FROM Sales GROUP BY CustomerID
      HAVING COUNT(DISTINCT CONCAT(CustomerName,'|',CustomerCity)) > 1) b
UNION ALL
SELECT 'FD3: ProductID -> ProductName, Category, UnitPrice', COUNT(*)
FROM (SELECT ProductID FROM Sales GROUP BY ProductID
      HAVING COUNT(DISTINCT CONCAT(ProductName,'|',Category,'|',UnitPrice)) > 1) c
UNION ALL
SELECT 'FD4: SalesRepID -> SalesRepName, Region', COUNT(*)
FROM (SELECT SalesRepID FROM Sales GROUP BY SalesRepID
      HAVING COUNT(DISTINCT CONCAT(SalesRepName,'|',Region)) > 1) d;
```

**Output:**

| FD | Violations |
|---|---|
| OrderID -> OrderDate, CustomerID, SalesRepID | 0 |
| CustomerID -> CustomerName, CustomerCity | 0 |
| ProductID -> ProductName, Category, UnitPrice | 0 |
| SalesRepID -> SalesRepName, Region | 0 |

Since FD1 to FD4 hold, they are enforced by the primary keys of Orders, Customer, Product and SalesRep respectively, and FD5 is enforced by the primary key of OrderItem. Every dependency is therefore preserved inside one table and no join is needed to enforce any of them.

### Business report from the normalized tables

```sql
SELECT TOP 6 o.OrderID, cu.CustomerName, p.ProductName, oi.Quantity,
       p.UnitPrice * oi.Quantity AS LineTotal
FROM Orders o
JOIN Customer cu  ON o.CustomerID = cu.CustomerID
JOIN OrderItem oi ON o.OrderID = oi.OrderID
JOIN Product p    ON oi.ProductID = p.ProductID
ORDER BY o.OrderID, p.ProductID;
```

**Output:**

| OrderID | CustomerName | ProductName | Quantity | LineTotal |
|---|---|---|---|---|
| 1001 | Aarav Sharma | Wireless Mouse | 1 | 799.0 |
| 1001 | Aarav Sharma | DBMS Textbook | 2 | 1300.0 |
| 1002 | Isha Verma | Keyboard | 1 | 3499.0 |
| 1003 | Rohan Mehta | Wireless Mouse | 2 | 1598.0 |
| 1003 | Rohan Mehta | Mixer Grinder | 1 | 2999.0 |
| 1004 | Ananya Iyer | Keyboard | 1 | 3499.0 |

**Conclusion:** The Sales table was normalized step by step from UNF to 1NF, 2NF, 3NF and BCNF. The decomposition is lossless (verified with EXCEPT in both directions) and dependency preserving (each FD is enforced by a key of a single table).
