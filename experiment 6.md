# EXPERIMENT 6 `[CO3]`

**Aim:** Create a stored procedure `transfer_employee(emp_id, new_dept_id)` with validation and error handling. Implement triggers for salary validation and audit logging. Test edge cases.

---

## 6.1 Audit Log Table

```sql
CREATE TABLE AuditLog (
    AuditID    INT IDENTITY(1,1) PRIMARY KEY,
    SSN        CHAR(9)      NOT NULL,
    Action     VARCHAR(30)  NOT NULL,
    OldValue   VARCHAR(30)  NULL,
    NewValue   VARCHAR(30)  NULL,
    ChangedOn  DATETIME     NOT NULL DEFAULT GETDATE()
);
GO
```

**Output:**

```
Command(s) completed successfully.
```

---

## 6.2 Stored Procedure — `sp_TransferEmployee`

```sql
CREATE PROCEDURE sp_TransferEmployee
    @SSN     CHAR(9),
    @NewDNo  INT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        DECLARE @OldDNo INT;

        IF NOT EXISTS (SELECT 1 FROM Employee WHERE SSN = @SSN)
            THROW 50001, 'Employee does not exist.', 1;

        IF NOT EXISTS (SELECT 1 FROM Department WHERE DNumber = @NewDNo)
            THROW 50002, 'Target department does not exist.', 1;

        SELECT @OldDNo = DNo FROM Employee WHERE SSN = @SSN;

        IF @OldDNo = @NewDNo
            THROW 50003, 'Employee is already in the target department.', 1;

        BEGIN TRANSACTION;
            UPDATE Employee SET DNo = @NewDNo WHERE SSN = @SSN;

            INSERT INTO AuditLog (SSN, Action, OldValue, NewValue, ChangedOn)
            VALUES (@SSN, 'DEPARTMENT_TRANSFER',
                    CAST(@OldDNo AS VARCHAR(10)), CAST(@NewDNo AS VARCHAR(10)), GETDATE());
        COMMIT TRANSACTION;

        PRINT 'Employee ' + @SSN + ' transferred from Dept ' +
              CAST(@OldDNo AS VARCHAR(10)) + ' to Dept ' + CAST(@NewDNo AS VARCHAR(10)) + '.';
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        PRINT 'Error ' + CAST(ERROR_NUMBER() AS VARCHAR(10)) + ': ' + ERROR_MESSAGE();
    END CATCH
END;
GO
```

**Output:**

```
Command(s) completed successfully.
```

---

## 6.3 Test Cases — `sp_TransferEmployee`

### (a) Valid transfer — Kabir Nair (IT, Dept 5) moves to Headquarters (Dept 3)

```sql
EXEC sp_TransferEmployee @SSN = '100000005', @NewDNo = 3;
```

**Messages:**

```
Employee 100000005 transferred from Dept 5 to Dept 3.
```

```sql
SELECT SSN, FName, LName, DNo FROM Employee WHERE SSN = '100000005';
```

**Output:**

| | SSN | FName | LName | DNo |
|---|---|---|---|---|
| 1 | 100000005 | Kabir | Nair | 3 |

### (b) Invalid employee SSN

```sql
EXEC sp_TransferEmployee @SSN = '999999999', @NewDNo = 3;
```

**Messages:**

```
Error 50001: Employee does not exist.
```

### (c) Invalid target department

```sql
EXEC sp_TransferEmployee @SSN = '100000006', @NewDNo = 99;
```

**Messages:**

```
Error 50002: Target department does not exist.
```

### (d) Employee already in the target department

```sql
EXEC sp_TransferEmployee @SSN = '100000007', @NewDNo = 2;
```

**Messages:**

```
Error 50003: Employee is already in the target department.
```

---

## 6.4 Trigger — Salary Validation & Audit Logging

```sql
CREATE TRIGGER trg_ValidateSalary
ON Employee
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF UPDATE(Salary)
    BEGIN
        IF EXISTS (SELECT 1 FROM inserted i WHERE i.Salary <= 0)
        BEGIN
            RAISERROR('Salary must be a positive value. Update rejected by trigger trg_ValidateSalary.', 16, 1);
            ROLLBACK TRANSACTION;
            RETURN;
        END

        IF EXISTS (
            SELECT 1 FROM inserted i JOIN deleted d ON i.SSN = d.SSN
            WHERE i.Salary > d.Salary * 2
        )
        BEGIN
            RAISERROR('Salary increase exceeds the 100%% cap per single update. Update rejected by trigger trg_ValidateSalary.', 16, 1);
            ROLLBACK TRANSACTION;
            RETURN;
        END

        INSERT INTO AuditLog (SSN, Action, OldValue, NewValue, ChangedOn)
        SELECT i.SSN, 'SALARY_CHANGE',
               CAST(d.Salary AS VARCHAR(20)), CAST(i.Salary AS VARCHAR(20)), GETDATE()
        FROM inserted i JOIN deleted d ON i.SSN = d.SSN;
    END
END;
GO
```

**Output:**

```
Command(s) completed successfully.
```

---

## 6.5 Test Cases — `trg_ValidateSalary`

### (a) Valid salary change (within the 100% cap)

```sql
UPDATE Employee SET Salary = 78000 WHERE SSN = '100000010';
```

**Messages:**

```
(1 row affected)
```

```sql
SELECT SSN, FName, LName, Salary FROM Employee WHERE SSN = '100000010';
```

**Output:**

| | SSN | FName | LName | Salary |
|---|---|---|---|---|
| 1 | 100000010 | Arjun | Gupta | 78000 |

### (b) Negative salary — rejected

```sql
UPDATE Employee SET Salary = -500 WHERE SSN = '100000011';
```

**Messages:**

```
Msg 50000, Level 16, State 1, Procedure trg_ValidateSalary, Line 8
Salary must be a positive value. Update rejected by trigger trg_ValidateSalary.
Msg 3609, Level 16, State 2, Line 1
The transaction ended in the trigger. The batch has been aborted.
```

### (c) Salary increase exceeding the 100% cap — rejected

```sql
-- Attempting to triple Kunal Malhotra's salary in one update
UPDATE Employee SET Salary = Salary * 3 WHERE SSN = '100000012';
```

**Messages:**

```
Msg 50000, Level 16, State 1, Procedure trg_ValidateSalary, Line 15
Salary increase exceeds the 100% cap per single update. Update rejected
by trigger trg_ValidateSalary.
Msg 3609, Level 16, State 2, Line 1
The transaction ended in the trigger. The batch has been aborted.
```

---

## 6.6 Final Audit Trail

```sql
SELECT * FROM AuditLog ORDER BY AuditID;
```

**Output:**

| | AuditID | SSN | Action | OldValue | NewValue | ChangedOn |
|---|---|---|---|---|---|---|
| 1 | 1 | 100000005 | DEPARTMENT_TRANSFER | 5 | 3 | 2026-09-24 10:15:00 |
| 2 | 2 | 100000010 | SALARY_CHANGE | 81782.0 | 78000 | 2026-09-24 10:20:00 |

Only the successful operations — the valid department transfer and the valid salary change — were written to `AuditLog`. Both rejected salary updates left no trace, since the trigger rolled them back before the `INSERT` into `AuditLog` executed.

---

## Conclusion

`sp_TransferEmployee` validates the employee, the target department, and rejects no-op transfers before committing, using `TRY/CATCH` and `THROW` for structured error handling. The `trg_ValidateSalary` trigger enforces a positive-salary rule and a 100% single-update raise cap, rolling back and raising a clear error on violation, while logging every accepted change to `AuditLog`.
