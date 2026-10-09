# EXPERIMENT 5 `[CO3]`

**Aim:** Create SQL views for department salary summary and employee hierarchy. Test updatability of views. Implement a recursive CTE to display reporting chains.

---

## 5.1 View — Department Salary Summary

```sql
CREATE VIEW DeptSalarySummary AS
SELECT D.DNumber, D.DName,
       COUNT(E.SSN)  AS EmpCount,
       SUM(E.Salary) AS TotalPayroll,
       AVG(E.Salary) AS AvgSalary
FROM Department D
JOIN Employee E ON D.DNumber = E.DNo
GROUP BY D.DNumber, D.DName;
GO

SELECT * FROM DeptSalarySummary ORDER BY TotalPayroll DESC;
```

**Output:**

| | DNumber | DName | EmpCount | TotalPayroll | AvgSalary |
|---|---|---|---|---|---|
| 1 | 3 | Headquarters | 6 | 492847 | 82141.17 |
| 2 | 5 | IT | 6 | 467794 | 77965.67 |
| 3 | 1 | Research | 6 | 414083 | 69013.83 |
| 4 | 4 | Sales | 6 | 391338 | 65223 |
| 5 | 2 | Administration | 6 | 367226 | 61204.33 |

---

## 5.2 View — Employee Hierarchy

```sql
CREATE VIEW EmployeeHierarchy AS
SELECT E.SSN,
       E.FName + ' ' + E.LName AS EmployeeName,
       S.FName + ' ' + S.LName AS SupervisorName,
       D.DName
FROM Employee E
LEFT JOIN Employee S ON E.SuperSSN = S.SSN
JOIN Department D    ON E.DNo = D.DNumber;
GO

SELECT TOP 10 * FROM EmployeeHierarchy ORDER BY DName, EmployeeName;
```

**Output:**

| | SSN | EmployeeName | SupervisorName | DName |
|---|---|---|---|---|
| 1 | 100000007 | Diya Singh | Isha Verma | Administration |
| 2 | 100000002 | Isha Verma | Rohan Mehta | Administration |
| 3 | 100000012 | Kunal Malhotra | Isha Verma | Administration |
| 4 | 100000017 | Pooja Shah | Isha Verma | Administration |
| 5 | 100000027 | Simran Dutta | Isha Verma | Administration |
| 6 | 100000022 | Yash Chauhan | Isha Verma | Administration |
| 7 | 100000008 | Aditya Rao | Rohan Mehta | Headquarters |
| 8 | 100000028 | Aryan Saxena | Rohan Mehta | Headquarters |
| 9 | 100000023 | Nisha Trivedi | Rohan Mehta | Headquarters |
| 10 | 100000013 | Priya Patel | Rohan Mehta | Headquarters |

---

## 5.3 Testing Updatability of Views

### (a) `DeptSalarySummary` — contains `GROUP BY` and aggregates

```sql
UPDATE DeptSalarySummary SET TotalPayroll = 999999 WHERE DNumber = 1;
```

**Messages:**

```
Msg 4406, Level 16, State 1, Line 1
Update or insert of view or function 'DeptSalarySummary' failed
because it contains a derived or constant field.
```

**Rejected:** a view built with `GROUP BY` / aggregate functions is never updatable in SQL Server because there is no one-to-one mapping back to base-table rows.

### (b) `EmployeeHierarchy` — contains a computed (concatenated) column

```sql
UPDATE EmployeeHierarchy SET EmployeeName = 'Test Name' WHERE SSN = '100000001';
```

**Messages:**

```
Msg 4406, Level 16, State 1, Line 1
Update or insert of view or function 'EmployeeHierarchy' failed
because it contains a derived or constant field.
```

**Rejected:** `EmployeeName` is a derived (concatenated) expression, not a plain base-table column, so SQL Server cannot translate the `UPDATE` unambiguously back onto `Employee`.

```sql
-- A plain, unambiguous base-table column IS updatable through this view, e.g.:
UPDATE EmployeeHierarchy SET DName = 'IT Department' WHERE SSN = '100000001';
-- (would succeed, since DName maps directly and unambiguously to Department.DName)
```

---

## 5.4 Recursive CTE — Reporting Chain

```sql
WITH ReportingChain (SSN, EmployeeName, SuperSSN, Level) AS (
    -- Anchor member: the top-level employee(s) with no supervisor
    SELECT SSN, FName + ' ' + LName, SuperSSN, 0
    FROM Employee
    WHERE SuperSSN IS NULL

    UNION ALL

    -- Recursive member: walk down the supervision chain
    SELECT E.SSN, E.FName + ' ' + E.LName, E.SuperSSN, RC.Level + 1
    FROM Employee E
    JOIN ReportingChain RC ON E.SuperSSN = RC.SSN
)
SELECT * FROM ReportingChain
ORDER BY Level, EmployeeName
OPTION (MAXRECURSION 100);
```

**Output:**

| | SSN | EmployeeName | SuperSSN | Level |
|---|---|---|---|---|
| 1 | 100000003 | Rohan Mehta | NULL | 0 |
| 2 | 100000001 | Aarav Sharma | 100000003 | 1 |
| 3 | 100000008 | Aditya Rao | 100000003 | 1 |
| 4 | 100000004 | Ananya Iyer | 100000003 | 1 |
| 5 | 100000028 | Aryan Saxena | 100000003 | 1 |
| 6 | 100000002 | Isha Verma | 100000003 | 1 |
| 7 | 100000005 | Kabir Nair | 100000003 | 1 |
| 8 | 100000023 | Nisha Trivedi | 100000003 | 1 |
| 9 | 100000013 | Priya Patel | 100000003 | 1 |
| 10 | 100000018 | Siddharth Pillai | 100000003 | 1 |
| 11 | 100000025 | Anjali Bansal | 100000005 | 2 |
| 12 | 100000010 | Arjun Gupta | 100000005 | 2 |
| 13 | 100000026 | Dev Kaur | 100000001 | 2 |

*(showing 13 of 30 rows — Level 0 is the CEO, Level 1 the five department managers, Level 2 their direct reports)*

---

## Conclusion

Two views were created — one aggregate (non-updatable) and one join-based view with a derived column (also non-updatable for that column, but updatable through plain base-table columns). A recursive CTE successfully unrolled the multi-level employee-supervisor reporting chain, from the top-level executive down to individual contributors.
