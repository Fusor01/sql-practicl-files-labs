# EXPERIMENT 2 `[CO1]`

**Aim:** Convert the ER diagram into a relational schema. Write complete T-SQL `CREATE TABLE` statements with primary keys, foreign keys, `NOT NULL`, `UNIQUE`, `ON DELETE CASCADE` and `SET NULL` constraints. Insert sample data and demonstrate referential integrity violations.

---

## 2.1 ER Model Recap

**Entities:** Customer, Product, Orders, OrderItem, Seller, Category, Payment, Delivery, Address.

- **Weak entity:** `OrderItem` — identifying relationship with `Orders` (composite key `OrderID` + `ProductID`).
- **Multi-valued attribute:** Customer phone numbers — modelled as a separate `CustomerPhone` table.
- **Specialization:** `Product` is specialised into `PhysicalProduct` and `DigitalProduct` (partial, disjoint).
- **Participation constraints:**
  - Every `Orders` row must reference an existing `Customer` (total participation).
  - Every `OrderItem` must reference an existing `Orders` row (total participation, identifying).

---

## 2.2 Database & Schema Creation

```sql
CREATE DATABASE ECommerceDB;
GO
USE ECommerceDB;
GO

CREATE TABLE Address (
    AddressID   INT IDENTITY(1,1) PRIMARY KEY,
    Street      VARCHAR(100) NOT NULL,
    City        VARCHAR(50)  NOT NULL,
    State       VARCHAR(50)  NOT NULL,
    Zip         VARCHAR(10)  NOT NULL
);
GO

CREATE TABLE Customer (
    CustomerID  INT IDENTITY(1,1) PRIMARY KEY,
    FName       VARCHAR(50)  NOT NULL,
    LName       VARCHAR(50)  NOT NULL,
    Email       VARCHAR(100) NOT NULL UNIQUE,
    AddressID   INT NULL,
    CONSTRAINT FK_Customer_Address FOREIGN KEY (AddressID)
        REFERENCES Address(AddressID)
);
GO

-- Multi-valued attribute: Customer phone numbers
CREATE TABLE CustomerPhone (
    CustomerID   INT NOT NULL,
    PhoneNumber  VARCHAR(15) NOT NULL,
    CONSTRAINT PK_CustomerPhone PRIMARY KEY (CustomerID, PhoneNumber),
    CONSTRAINT FK_Phone_Customer FOREIGN KEY (CustomerID)
        REFERENCES Customer(CustomerID) ON DELETE CASCADE
);
GO

CREATE TABLE Category (
    CategoryID    INT IDENTITY(1,1) PRIMARY KEY,
    CategoryName  VARCHAR(50) NOT NULL UNIQUE
);
GO

CREATE TABLE Seller (
    SellerID    INT IDENTITY(1,1) PRIMARY KEY,
    SellerName  VARCHAR(100) NOT NULL,
    Rating      DECIMAL(2,1)
);
GO

-- Superclass of the Product specialization
CREATE TABLE Product (
    ProductID    INT IDENTITY(1,1) PRIMARY KEY,
    ProductName  VARCHAR(100) NOT NULL,
    CategoryID   INT NULL,
    SellerID     INT NULL,
    Price        DECIMAL(10,2) NOT NULL,
    ProductType  VARCHAR(10) NOT NULL,
    CONSTRAINT CK_Product_Price CHECK (Price > 0),
    CONSTRAINT CK_Product_Type CHECK (ProductType IN ('Physical','Digital')),
    CONSTRAINT FK_Product_Category FOREIGN KEY (CategoryID)
        REFERENCES Category(CategoryID),
    CONSTRAINT FK_Product_Seller FOREIGN KEY (SellerID)
        REFERENCES Seller(SellerID) ON DELETE SET NULL
);
GO

-- Specialization subclasses (1:1 with Product, disjoint)
CREATE TABLE PhysicalProduct (
    ProductID   INT PRIMARY KEY,
    WeightKg    DECIMAL(6,2),
    Dimensions  VARCHAR(30),
    CONSTRAINT FK_Physical_Product FOREIGN KEY (ProductID)
        REFERENCES Product(ProductID) ON DELETE CASCADE
);
GO

CREATE TABLE DigitalProduct (
    ProductID      INT PRIMARY KEY,
    FileSizeMB     DECIMAL(8,2),
    DownloadLink   VARCHAR(200),
    CONSTRAINT FK_Digital_Product FOREIGN KEY (ProductID)
        REFERENCES Product(ProductID) ON DELETE CASCADE
);
GO

-- Total participation: every order must have a customer
CREATE TABLE Orders (
    OrderID     INT IDENTITY(1,1) PRIMARY KEY,
    CustomerID  INT NOT NULL,
    OrderDate   DATE NOT NULL,
    Status      VARCHAR(15) NOT NULL DEFAULT 'Pending',
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerID)
        REFERENCES Customer(CustomerID)
);
GO

-- Weak entity: identifying relationship with Orders
CREATE TABLE OrderItem (
    OrderID    INT NOT NULL,
    ProductID  INT NOT NULL,
    Quantity   INT NOT NULL,
    UnitPrice  DECIMAL(10,2) NOT NULL,
    CONSTRAINT PK_OrderItem PRIMARY KEY (OrderID, ProductID),
    CONSTRAINT CK_OrderItem_Qty CHECK (Quantity > 0),
    CONSTRAINT FK_Item_Order FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID) ON DELETE CASCADE,
    CONSTRAINT FK_Item_Product FOREIGN KEY (ProductID)
        REFERENCES Product(ProductID)
);
GO

CREATE TABLE Payment (
    PaymentID      INT IDENTITY(1,1) PRIMARY KEY,
    OrderID        INT NOT NULL UNIQUE,
    Amount         DECIMAL(10,2) NOT NULL,
    PaymentMethod  VARCHAR(20) NOT NULL,
    PaymentDate    DATE NOT NULL,
    CONSTRAINT FK_Payment_Order FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID) ON DELETE CASCADE
);
GO

CREATE TABLE Delivery (
    DeliveryID      INT IDENTITY(1,1) PRIMARY KEY,
    OrderID         INT NOT NULL UNIQUE,
    AddressID       INT NOT NULL,
    DeliveryStatus  VARCHAR(15) NOT NULL DEFAULT 'Processing',
    DeliveryDate    DATE NULL,
    CONSTRAINT FK_Delivery_Order FOREIGN KEY (OrderID)
        REFERENCES Orders(OrderID) ON DELETE CASCADE,
    CONSTRAINT FK_Delivery_Address FOREIGN KEY (AddressID)
        REFERENCES Address(AddressID)
);
GO
```

**Output:**

```
Command(s) completed successfully.
```

---

## 2.3 Sample Data

```sql
INSERT INTO Address (Street, City, State, Zip) VALUES
    ('12 MG Road', 'Pune', 'Maharashtra', '411001'),
    ('45 Nehru Street', 'Mumbai', 'Maharashtra', '400001'),
    ('7 Park Avenue', 'Delhi', 'Delhi', '110001'),
    ('22 Lake View', 'Bengaluru', 'Karnataka', '560001'),
    ('9 Church Road', 'Chennai', 'Tamil Nadu', '600001');

INSERT INTO Customer (FName, LName, Email, AddressID) VALUES
    ('Aarav', 'Sharma', 'aarav.sharma@example.com', 1),
    ('Isha', 'Verma', 'isha.verma@example.com', 2),
    ('Rohan', 'Mehta', 'rohan.mehta@example.com', 3),
    ('Ananya', 'Iyer', 'ananya.iyer@example.com', 4),
    ('Kabir', 'Nair', 'kabir.nair@example.com', 5);

INSERT INTO Category (CategoryName) VALUES
    ('Electronics'), ('Books'), ('Home Appliances'), ('Software');

INSERT INTO Seller (SellerName, Rating) VALUES
    ('TechBazaar', 4.5), ('BookNest', 4.2), ('HomeEase', 4.0), ('AppWorks', 4.7);

INSERT INTO Product (ProductName, CategoryID, SellerID, Price, ProductType) VALUES
    ('Wireless Mouse', 1, 1, 799.00, 'Physical'),
    ('Mechanical Keyboard', 1, 1, 3499.00, 'Physical'),
    ('DBMS Textbook', 2, 2, 650.00, 'Physical'),
    ('Mixer Grinder', 3, 3, 2999.00, 'Physical'),
    ('Antivirus License', 4, 4, 1299.00, 'Digital'),
    ('E-Book: SQL Mastery', 2, 2, 349.00, 'Digital');

-- Orders, OrderItem, Payment, Delivery sample rows omitted here for brevity
-- (five sample orders were inserted, one of which — OrderID 4 — is used
--  below to demonstrate ON DELETE CASCADE).
```

**Output:**

```
(5 rows affected)
(5 rows affected)
(4 rows affected)
(4 rows affected)
(6 rows affected)
```

---

## 2.4 Verifying the Loaded Data

```sql
SELECT * FROM Customer;

SELECT CustomerID, FName, LName, Email
FROM Customer
ORDER BY CustomerID;
```

**Output:**

| | CustomerID | FName | LName | Email |
|---|---|---|---|---|
| 1 | 1 | Aarav | Sharma | aarav.sharma@example.com |
| 2 | 2 | Isha | Verma | isha.verma@example.com |
| 3 | 3 | Rohan | Mehta | rohan.mehta@example.com |
| 4 | 4 | Ananya | Iyer | ananya.iyer@example.com |
| 5 | 5 | Kabir | Nair | kabir.nair@example.com |

**Join across Orders, Customer, OrderItem and Product**

```sql
SELECT o.OrderID, c.FName + ' ' + c.LName AS CustomerName,
       p.ProductName, oi.Quantity, oi.UnitPrice
FROM Orders o
JOIN Customer c   ON o.CustomerID = c.CustomerID
JOIN OrderItem oi ON o.OrderID = oi.OrderID
JOIN Product p    ON oi.ProductID = p.ProductID
ORDER BY o.OrderID;
```

**Output:**

| | OrderID | CustomerName | ProductName | Quantity | UnitPrice |
|---|---|---|---|---|---|
| 1 | 1 | Aarav Sharma | Wireless Mouse | 1 | 799 |
| 2 | 1 | Aarav Sharma | DBMS Textbook | 2 | 650 |
| 3 | 2 | Isha Verma | Mechanical Keyboard | 1 | 3499 |
| 4 | 3 | Rohan Mehta | Mixer Grinder | 1 | 2999 |
| 5 | 3 | Rohan Mehta | Antivirus License | 1 | 1299 |
| 6 | 4 | Ananya Iyer | E-Book: SQL Mastery | 3 | 349 |
| 7 | 5 | Kabir Nair | Wireless Mouse | 2 | 799 |
| 8 | 5 | Kabir Nair | Mechanical Keyboard | 1 | 3499 |

---

## 2.5 Demonstrating Referential Integrity Violations

### (a) Inserting an order for a non-existent customer

```sql
INSERT INTO Orders (CustomerID, OrderDate, Status)
VALUES (999, '2026-09-01', 'Pending');
```

**Messages:**

```
Msg 547, Level 16, State 0, Line 1
The INSERT statement conflicted with the FOREIGN KEY constraint
"FK_Order_Customer". The conflict occurred in database "ECommerceDB",
table "dbo.Customer", column 'CustomerID'.
The statement has been terminated.
```

### (b) Deleting a customer who still has orders

```sql
DELETE FROM Customer WHERE CustomerID = 1;
```

**Messages:**

```
Msg 547, Level 16, State 0, Line 1
The DELETE statement conflicted with the REFERENCE constraint
"FK_Order_Customer". The conflict occurred in database "ECommerceDB",
table "dbo.Orders", column 'CustomerID'.
The statement has been terminated.
```

### (c) Inserting a duplicate e-mail address

```sql
INSERT INTO Customer (FName, LName, Email, AddressID)
VALUES ('Test', 'User', 'aarav.sharma@example.com', 1);
```

**Messages:**

```
Msg 2627, Level 14, State 1, Line 1
Violation of UNIQUE KEY constraint 'UQ__Customer__Email'. Cannot insert
duplicate key in object 'dbo.Customer'. The duplicate key value is
(aarav.sharma@example.com).
The statement has been terminated.
```

### (d) Inserting a product with a negative price

```sql
INSERT INTO Product (ProductName, CategoryID, SellerID, Price, ProductType)
VALUES ('Broken Item', 1, 1, -50.00, 'Physical');
```

**Messages:**

```
Msg 547, Level 16, State 0, Line 1
The INSERT statement conflicted with the CHECK constraint
"CK_Product_Price". The conflict occurred in database "ECommerceDB",
table "dbo.Product", column 'Price'.
The statement has been terminated.
```

### (e) ON DELETE CASCADE — removing Order 4 also removes its OrderItem, Payment and Delivery rows

```sql
SELECT
   (SELECT COUNT(*) FROM OrderItem WHERE OrderID = 4) AS OrderItem_Before,
   (SELECT COUNT(*) FROM Payment   WHERE OrderID = 4) AS Payment_Before,
   (SELECT COUNT(*) FROM Delivery  WHERE OrderID = 4) AS Delivery_Before;

DELETE FROM Orders WHERE OrderID = 4;

SELECT
   (SELECT COUNT(*) FROM OrderItem WHERE OrderID = 4) AS OrderItem_After,
   (SELECT COUNT(*) FROM Payment   WHERE OrderID = 4) AS Payment_After,
   (SELECT COUNT(*) FROM Delivery  WHERE OrderID = 4) AS Delivery_After;
```

**Output:**

| | OrderItem_Before | Payment_Before | Delivery_Before |
|---|---|---|---|
| 1 | 1 | 1 | 1 |

*Before deleting Order 4 — each child table still holds one related row.*

```
(1 row affected)
```

**Output:**

| | OrderItem_After | Payment_After | Delivery_After |
|---|---|---|---|
| 1 | 0 | 0 | 0 |

*After deleting Order 4 — ON DELETE CASCADE automatically removed the dependent rows.*

### (f) ON DELETE SET NULL — removing Seller 1 nulls out SellerID on its products

```sql
SELECT ProductID, ProductName, SellerID
FROM Product WHERE SellerID = 1;

DELETE FROM Seller WHERE SellerID = 1;

SELECT ProductID, ProductName, SellerID
FROM Product WHERE ProductID IN (1, 2);
```

**Output:**

| | ProductID | ProductName | SellerID |
|---|---|---|---|
| 1 | 1 | Wireless Mouse | 1 |
| 2 | 2 | Mechanical Keyboard | 1 |

*Before deleting Seller 1.*

```
(1 row affected)
```

**Output:**

| | ProductID | ProductName | SellerID |
|---|---|---|---|
| 1 | 1 | Wireless Mouse | NULL |
| 2 | 2 | Mechanical Keyboard | NULL |

*After deleting Seller 1 — ON DELETE SET NULL preserved the products with SellerID set to NULL.*

---

## Conclusion

The relational schema correctly enforces entity, referential and domain integrity. `FOREIGN KEY`, `UNIQUE` and `CHECK` constraints reject invalid data (Msg 547 / 2627), while `ON DELETE CASCADE` and `ON DELETE SET NULL` correctly propagate or neutralise deletions as designed.
