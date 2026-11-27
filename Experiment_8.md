# Experiment 8 [CO4]

**Implement banking-style transactions using BEGIN, COMMIT, ROLLBACK, and SAVEPOINT. Simulate partial failures. Demonstrate isolation levels and concurrency anomalies using two sessions.**

## 8.1 Setup

```sql
CREATE DATABASE BankDB;
GO
USE BankDB;
GO

CREATE TABLE Account (
    AccNo       INT PRIMARY KEY,
    HolderName  VARCHAR(50) NOT NULL,
    Balance     DECIMAL(12,2) NOT NULL,
    CONSTRAINT CK_Account_Balance CHECK (Balance >= 0)
);
GO

INSERT INTO Account VALUES
    (101, 'Aarav Sharma', 50000),
    (102, 'Isha Verma',   30000),
    (103, 'Rohan Mehta',  20000),
    (104, 'Ananya Iyer',  15000);

SELECT * FROM Account ORDER BY AccNo;
```

**Messages:**

```text
Command(s) completed successfully.

(4 rows affected)
```

**Output:**

| AccNo | HolderName | Balance |
|---|---|---|
| 101 | Aarav Sharma | 50000 |
| 102 | Isha Verma | 30000 |
| 103 | Rohan Mehta | 20000 |
| 104 | Ananya Iyer | 15000 |

## 8.2 Successful Transaction — BEGIN TRANSACTION and COMMIT

Transfer of Rs. 5000 from account 101 to account 102.

```sql
BEGIN TRANSACTION;
    UPDATE Account SET Balance = Balance - 5000 WHERE AccNo = 101;
    UPDATE Account SET Balance = Balance + 5000 WHERE AccNo = 102;
COMMIT TRANSACTION;

SELECT * FROM Account WHERE AccNo IN (101, 102);
```

**Messages:**

```text
(1 row affected)

(1 row affected)
```

**Output:**

| AccNo | HolderName | Balance |
|---|---|---|
| 101 | Aarav Sharma | 45000 |
| 102 | Isha Verma | 35000 |

## 8.3 Partial Failure — Automatic ROLLBACK

A transfer of Rs. 25000 from account 103 (balance 20000) to account 104. The credit is executed first and the debit fails the CHECK constraint, so the whole transaction must be undone.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
        UPDATE Account SET Balance = Balance + 25000 WHERE AccNo = 104;  -- succeeds
        UPDATE Account SET Balance = Balance - 25000 WHERE AccNo = 103;  -- violates CHECK
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    PRINT 'Transaction rolled back: ' + ERROR_MESSAGE();
END CATCH;

SELECT * FROM Account WHERE AccNo IN (103, 104);
```

**Messages:**

```text
(1 row affected)
Transaction rolled back: The UPDATE statement conflicted with the CHECK constraint
"CK_Account_Balance". The conflict occurred in database "BankDB", table "dbo.Account",
column 'Balance'.
```

**Output:**

| AccNo | HolderName | Balance |
|---|---|---|
| 103 | Rohan Mehta | 20000 |
| 104 | Ananya Iyer | 15000 |

*Account 104 still has 15000 even though its credit statement ran successfully, because ROLLBACK undid the whole transaction. This is atomicity.*

## 8.4 SAVEPOINT — Partial Rollback

In SQL Server a savepoint is created with SAVE TRANSACTION name and undone with ROLLBACK TRANSACTION name.

```sql
BEGIN TRANSACTION;
    UPDATE Account SET Balance = Balance - 2000 WHERE AccNo = 104;
    SELECT AccNo, Balance FROM Account WHERE AccNo = 104;     -- after step 1

    SAVE TRANSACTION AfterFirstWithdrawal;

    UPDATE Account SET Balance = Balance - 3000 WHERE AccNo = 104;
    SELECT AccNo, Balance FROM Account WHERE AccNo = 104;     -- after step 2

    ROLLBACK TRANSACTION AfterFirstWithdrawal;                -- undo step 2
    SELECT AccNo, Balance FROM Account WHERE AccNo = 104;     -- after rollback
COMMIT TRANSACTION;
```

**Output:**

After step 1:

| AccNo | Balance |
|---|---|
| 104 | 13000 |

After step 2:

| AccNo | Balance |
|---|---|
| 104 | 10000 |

After ROLLBACK TRANSACTION AfterFirstWithdrawal:

| AccNo | Balance |
|---|---|
| 104 | 13000 |

Final state of the table after the COMMIT:

```sql
SELECT * FROM Account ORDER BY AccNo;
```

**Output:**

| AccNo | HolderName | Balance |
|---|---|---|
| 101 | Aarav Sharma | 45000 |
| 102 | Isha Verma | 35000 |
| 103 | Rohan Mehta | 20000 |
| 104 | Ananya Iyer | 13000 |

*Only the first withdrawal of 2000 was committed; the second withdrawal of 3000 was rolled back to the savepoint.*

## 8.5 Isolation Levels and Concurrency Anomalies

Two query windows in SSMS (Session 1 and Session 2) were connected to BankDB. The statements below were executed in the order of the step numbers.

### (a) Dirty read — READ UNCOMMITTED

### Step 1 — Session 1

```sql
BEGIN TRANSACTION;
UPDATE Account SET Balance = Balance - 10000 WHERE AccNo = 101;
-- transaction left open
```

### Step 2 — Session 2

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT AccNo, Balance FROM Account WHERE AccNo = 101;
```

**Output:**

| AccNo | Balance |
|---|---|
| 101 | 35000 |

*Session 2 reads 35000, a value that has not been committed (dirty read).*

### Step 3 — Session 1

```sql
ROLLBACK TRANSACTION;
```

### Step 4 — Session 2

```sql
SELECT AccNo, Balance FROM Account WHERE AccNo = 101;
```

**Output:**

| AccNo | Balance |
|---|---|
| 101 | 45000 |

*The balance is 45000 again, so Session 2 had made a decision on data that never existed. With the default READ COMMITTED level, Step 2 simply waits until Session 1 finishes.*

### (b) Non-repeatable read — READ COMMITTED

### Step 1 — Session 1

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;
SELECT AccNo, Balance FROM Account WHERE AccNo = 102;
```

**Output:**

| AccNo | Balance |
|---|---|
| 102 | 35000 |

### Step 2 — Session 2

```sql
UPDATE Account SET Balance = Balance + 5000 WHERE AccNo = 102;   -- auto-commit
```

**Messages:**

```text
(1 row affected)
```

### Step 3 — Session 1

```sql
SELECT AccNo, Balance FROM Account WHERE AccNo = 102;
COMMIT TRANSACTION;
```

**Output:**

| AccNo | Balance |
|---|---|
| 102 | 40000 |

*The same query returned 35000 and then 40000 inside one transaction (non-repeatable read). Using SET TRANSACTION ISOLATION LEVEL REPEATABLE READ in Session 1 makes the UPDATE in Step 2 wait until Session 1 commits, so both reads would return the same value.*

### (c) Phantom read — REPEATABLE READ versus SERIALIZABLE

### Step 1 — Session 1

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
SELECT COUNT(*) AS HighBalanceAccounts FROM Account WHERE Balance >= 20000;
```

**Output:**

| HighBalanceAccounts |
|---|
| 3 |

### Step 2 — Session 2

```sql
INSERT INTO Account VALUES (105, 'Kabir Nair', 60000);   -- auto-commit
```

**Messages:**

```text
(1 row affected)
```

### Step 3 — Session 1

```sql
SELECT COUNT(*) AS HighBalanceAccounts FROM Account WHERE Balance >= 20000;
COMMIT TRANSACTION;
```

**Output:**

| HighBalanceAccounts |
|---|
| 4 |

*A new row (a phantom) appeared in the second read. If Session 1 had used SET TRANSACTION ISOLATION LEVEL SERIALIZABLE, the INSERT in Step 2 would have been blocked until Session 1 committed.*

### Summary of isolation levels

**Output:**

| Isolation Level | Dirty Read | Non-Repeatable | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SNAPSHOT | Prevented | Prevented | Prevented |
| SERIALIZABLE | Prevented | Prevented | Prevented |

**Conclusion:** Funds transfers were executed atomically with BEGIN, COMMIT and ROLLBACK, a failed transfer left no partial changes, and a savepoint allowed part of a transaction to be undone. Two-session tests demonstrated dirty, non-repeatable and phantom reads and which isolation level prevents each of them.
