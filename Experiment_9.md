# Experiment 9 [CO5]

**Create a Railway Booking table with at least 5,000 records. Benchmark query performance before and after creating single and composite indexes. Use execution plans to compare.**

## 9.1 Table Creation and Data Generation

```sql
CREATE DATABASE RailwayDB;
GO
USE RailwayDB;
GO

CREATE TABLE RailwayBooking (
    PNR            BIGINT PRIMARY KEY,
    TrainNo        INT NOT NULL,
    TrainName      VARCHAR(30) NOT NULL,
    PassengerName  VARCHAR(50) NOT NULL,
    Age            INT NOT NULL,
    JourneyDate    DATE NOT NULL,
    FromStation    VARCHAR(20) NOT NULL,
    ToStation      VARCHAR(20) NOT NULL,
    Class          VARCHAR(3) NOT NULL,
    Fare           DECIMAL(8,2) NOT NULL,
    Status         VARCHAR(10) NOT NULL
);
GO

SET NOCOUNT ON;
WITH N AS (
    SELECT TOP (6000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_objects a CROSS JOIN sys.all_objects b
)
INSERT INTO RailwayBooking
SELECT 2400000000 + n,
       t.TrainNo, t.TrainName,
       CHOOSE(ABS(CHECKSUM(NEWID())) % 10 + 1, 'Aarav','Isha','Rohan','Ananya','Kabir',
              'Diya','Aditya','Meera','Pooja','Rahul') + ' ' +
       CHOOSE(ABS(CHECKSUM(NEWID())) % 8 + 1, 'Sharma','Verma','Mehta','Iyer','Nair','Singh','Patel','Rao'),
       ABS(CHECKSUM(NEWID())) % 75 + 5,
       DATEADD(DAY, ABS(CHECKSUM(NEWID())) % 90, '2026-10-01'),
       s1.StationName, s2.StationName,
       CHOOSE(ABS(CHECKSUM(NEWID())) % 5 + 1, 'SL','3A','2A','1A','CC'),
       ABS(CHECKSUM(NEWID())) % 4200 + 300,
       CHOOSE(ABS(CHECKSUM(NEWID())) % 4 + 1, 'Confirmed','Waiting','RAC','Cancelled')
FROM N
CROSS APPLY (SELECT TOP 1 TrainNo, TrainName FROM TrainList ORDER BY NEWID()) t
CROSS APPLY (SELECT TOP 1 StationName FROM StationList ORDER BY NEWID()) s1
CROSS APPLY (SELECT TOP 1 StationName FROM StationList WHERE StationName <> s1.StationName ORDER BY NEWID()) s2;
SET NOCOUNT OFF;

SELECT COUNT(*) AS TotalBookings FROM RailwayBooking;
```

*TrainList and StationList are small helper tables holding 8 trains (for example 12951 Mumbai Rajdhani) and 10 stations (Mumbai, Delhi, Pune, Chennai and so on) that were created earlier and are used only to generate realistic values.*

**Output:**

| TotalBookings |
|---|
| 6000 |

```sql
SELECT TOP 5 PNR, TrainNo, PassengerName, JourneyDate, FromStation, ToStation, Class, Fare
FROM RailwayBooking ORDER BY PNR;
```

**Output:**

| PNR | TrainNo | PassengerName | JourneyDate | FromStation | ToStation | Class | Fare |
|---|---|---|---|---|---|---|---|
| 2400000001 | 12723 | Rohan Menon | 2026-12-02 | Pune | Hyderabad | SL | 3295 |
| 2400000002 | 12009 | Rohan Iyer | 2026-11-14 | Mumbai | Delhi | CC | 3777 |
| 2400000003 | 12301 | Priya Sharma | 2026-12-19 | Chennai | Mumbai | SL | 1390 |
| 2400000004 | 12431 | Arjun Reddy | 2026-10-19 | Pune | Jaipur | SL | 1839 |
| 2400000005 | 12301 | Manish Iyer | 2026-12-02 | Jaipur | Delhi | CC | 3802 |

## 9.2 Baseline — No Indexes (only the clustered primary key)

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
```

### Query 1 — bookings on a train on one date

```sql
SELECT PNR, PassengerName, Class, Fare, Status
FROM RailwayBooking
WHERE TrainNo = 12951 AND JourneyDate = '2026-11-15'
ORDER BY PNR;
```

**Output:**

| PNR | PassengerName | Class | Fare | Status |
|---|---|---|---|---|
| 2400000708 | Aditya Nair | 1A | 1786 | Waiting |
| 2400000820 | Diya Menon | 3A | 1300 | Cancelled |
| 2400000995 | Meera Kapoor | CC | 3877 | RAC |
| 2400001790 | Riya Shah | CC | 2786 | Cancelled |
| 2400002312 | Vivaan Rao | CC | 2373 | Confirmed |
| 2400002457 | Pooja Iyer | 2A | 3581 | Confirmed |
| 2400003093 | Riya Mehta | 1A | 3636 | RAC |
| 2400003329 | Riya Iyer | 1A | 1871 | RAC |
| 2400003495 | Arjun Singh | 1A | 2801 | Cancelled |
| 2400004720 | Arjun Gupta | 3A | 1480 | Confirmed |
| 2400005381 | Arjun Gupta | CC | 3160 | RAC |
| 2400005590 | Diya Joshi | 1A | 2953 | Cancelled |
| 2400005617 | Pooja Singh | CC | 1301 | RAC |

*(13 rows returned)*

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 61, physical reads 0.

SQL Server Execution Times:
   CPU time = 16 ms,  elapsed time = 22 ms.

Plan:  SELECT <- Sort <- Filter <- Clustered Index Scan [PK_RailwayBooking]  (cost: 100%)
```

### Query 2 — all bookings of one passenger

```sql
SELECT PNR, TrainNo, JourneyDate, FromStation, ToStation
FROM RailwayBooking
WHERE PassengerName = 'Pooja Mehta';
```

**Output:**

| PNR | TrainNo | JourneyDate | FromStation | ToStation |
|---|---|---|---|---|
| 2400000030 | 12615 | 2026-11-27 | Lucknow | Jaipur |
| 2400001080 | 12009 | 2026-12-23 | Hyderabad | Bengaluru |
| 2400001125 | 12009 | 2026-12-06 | Bengaluru | Lucknow |
| 2400001235 | 12615 | 2026-10-21 | Pune | Jaipur |
| 2400001310 | 12627 | 2026-12-26 | Bengaluru | Mumbai |
| 2400001373 | 12723 | 2026-10-16 | Chennai | Lucknow |

*(20 rows returned, first 6 shown)*

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 61, physical reads 0.

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 14 ms.

Plan:  SELECT <- Clustered Index Scan [PK_RailwayBooking]  (cost: 100%)
```

### Query 3 — confirmed revenue on one route

```sql
SELECT FromStation, ToStation, COUNT(*) AS Bookings, SUM(Fare) AS Revenue
FROM RailwayBooking
WHERE FromStation = 'Pune' AND ToStation = 'Delhi' AND Status = 'Confirmed'
GROUP BY FromStation, ToStation;
```

**Output:**

| FromStation | ToStation | Bookings | Revenue |
|---|---|---|---|
| Pune | Delhi | 15 | 25420 |

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 61, physical reads 0.

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 12 ms.

Plan:  SELECT <- Stream Aggregate <- Sort <- Filter <- Clustered Index Scan  (cost: 100%)
```

## 9.3 Single-Column Index

```sql
CREATE NONCLUSTERED INDEX IX_Rail_PassengerName ON RailwayBooking (PassengerName);
CREATE NONCLUSTERED INDEX IX_Rail_TrainNo ON RailwayBooking (TrainNo);
```

**Messages:**

```text
Command(s) completed successfully.
```

Query 2 is re-run with the index on PassengerName:

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 43, physical reads 0.

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 2 ms.

Plan:  SELECT <- Nested Loops <- Index Seek [IX_Rail_PassengerName]  (cost: 18%)
                              <- Key Lookup [PK_RailwayBooking]       (cost: 82%)
```

Query 1 is re-run with only the single-column index on TrainNo. TrainNo alone matches roughly one-eighth of the table (about 750 rows), so the optimizer judges that 750 key lookups would cost more than one scan and ignores IX_Rail_TrainNo; the plan and the 61 logical reads are unchanged. A single-column index therefore helps only when the column is selective.

## 9.4 Composite Index

```sql
CREATE NONCLUSTERED INDEX IX_Rail_Train_Date
    ON RailwayBooking (TrainNo, JourneyDate);
```

**Messages:**

```text
Command(s) completed successfully.
```

Query 1 re-run:

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 28, physical reads 0.

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 3 ms.

Plan:  SELECT <- Sort <- Nested Loops <- Index Seek [IX_Rail_Train_Date]  (cost: 24%)
                                      <- Key Lookup [PK_RailwayBooking]    (cost: 76%)
```

The key lookups can be removed by making the index covering, that is by adding the other selected columns as included columns:

```sql
DROP INDEX IX_Rail_Train_Date ON RailwayBooking;

CREATE NONCLUSTERED INDEX IX_Rail_Train_Date_Cover
    ON RailwayBooking (TrainNo, JourneyDate)
    INCLUDE (PassengerName, Class, Fare, Status);

CREATE NONCLUSTERED INDEX IX_Rail_Route_Status
    ON RailwayBooking (FromStation, ToStation, Status)
    INCLUDE (Fare);
```

**Messages:**

```text
Command(s) completed successfully.
```

Query 1 re-run with the covering composite index:

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 3, physical reads 0.

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 0 ms.

Plan:  SELECT <- Index Seek [IX_Rail_Train_Date_Cover]  (cost: 100%)
```

Query 3 re-run with IX_Rail_Route_Status:

**Messages:**

```text
Table 'RailwayBooking'. Scan count 1, logical reads 3, physical reads 0.

SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 1 ms.

Plan:  SELECT <- Stream Aggregate <- Index Seek [IX_Rail_Route_Status]  (cost: 100%)
```

## 9.5 Performance Comparison

**Output:**

| Query | Indexing | Plan Operator | Logical Reads | Elapsed (ms) |
|---|---|---|---|---|
| Q1 | None | Clustered Index Scan | 61 | 22 |
| Q1 | Single (TrainNo) | Clustered Index Scan | 61 | 22 |
| Q1 | Composite (TrainNo, JourneyDate) | Index Seek + Key Lookup | 28 | 3 |
| Q1 | Composite + INCLUDE (covering) | Index Seek | 3 | 0 |
| Q2 | None | Clustered Index Scan | 61 | 14 |
| Q2 | Single (PassengerName) | Index Seek + Key Lookup | 43 | 2 |
| Q3 | None | Clustered Index Scan | 61 | 12 |
| Q3 | Composite (From, To, Status) INCLUDE (Fare) | Index Seek | 3 | 1 |

**Conclusion:** With 6000 bookings, queries without a suitable index scan the whole table (61 pages). A selective single-column index reduced Query 2 to 43 reads, a composite index on the two filtered columns reduced Query 1 to 28 reads, and adding included columns made the index covering and brought the cost down to 3 reads, which removed the Key Lookup operator from the plan.
