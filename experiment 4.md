# EXPERIMENT 4 `[CO2]`

**Aim:** Using the Employee schema, write queries with `INNER JOIN`, `LEFT JOIN`, self-join, 3-way join, correlated subqueries, `EXISTS`, and `INTERSECT`/`EXCEPT`. Compare execution plans using the estimated execution plan.

---

## 4.1 INNER JOIN

```sql
SELECT TOP 10 E.FName, E.LName, D.DName
FROM Employee E
INNER JOIN Department D ON E.DNo = D.DNumber
ORDER BY D.DName, E.LName;
```

**Output:**

| | FName | LName | DName |
|---|---|---|---|
| 1 | Yash | Chauhan | Administration |
| 2 | Simran | Dutta | Administration |
| 3 | Kunal | Malhotra | Administration |
| 4 | Pooja | Shah | Administration |
| 5 | Diya | Singh | Administration |
| 6 | Isha | Verma | Administration |
| 7 | Rohan | Mehta | Headquarters |
| 8 | Priya | Patel | Headquarters |
| 9 | Siddharth | Pillai | Headquarters |
| 10 | Aditya | Rao | Headquarters |

---

## 4.2 LEFT JOIN

`LEFT JOIN` is used so that employees would still appear even if they had no project assignment in `WorksOn` (none do in this dataset, confirming full participation).

```sql
SELECT TOP 12 E.FName, E.LName, P.PName, W.Hours
FROM Employee E
LEFT JOIN WorksOn W ON E.SSN = W.ESSN
LEFT JOIN Project P ON W.PNo = P.PNumber
ORDER BY E.LName;
```

**Output:**

| | FName | LName | PName | Hours |
|---|---|---|---|---|
| 1 | Kavya | Agarwal | Database Migration | 24.20 |
| 2 | Kavya | Agarwal | Payroll System | 5.70 |
| 3 | Kavya | Agarwal | Customer CRM Upgrade | 7.90 |
| 4 | Anjali | Bansal | Cloud Infrastructure | 4 |
| 5 | Tara | Bhat | AI Research Lab | 13.70 |
| 6 | Rahul | Bose | Customer CRM Upgrade | 5.50 |
| 7 | Neha | Chatterjee | Cloud Infrastructure | 18 |
| 8 | Yash | Chauhan | Payroll System | 21 |
| 9 | Riya | Desai | Market Expansion | 5.30 |
| 10 | Simran | Dutta | Payroll System | 8.60 |
| 11 | Simran | Dutta | Office Renovation | 5.20 |
| 12 | Simran | Dutta | Market Expansion | 19.50 |

---

## 4.3 Self-Join — Employee and Supervisor

```sql
SELECT TOP 12 E.FName + ' ' + E.LName AS Employee,
       S.FName + ' ' + S.LName AS Supervisor
FROM Employee E
LEFT JOIN Employee S ON E.SuperSSN = S.SSN
ORDER BY Supervisor, Employee;
```

**Output:**

| | Employee | Supervisor |
|---|---|---|
| 1 | Rohan Mehta | NULL |
| 2 | Dev Kaur | Aarav Sharma |
| 3 | Karan Menon | Aarav Sharma |
| 4 | Sneha Joshi | Aarav Sharma |
| 5 | Tara Bhat | Aarav Sharma |
| 6 | Vivaan Kapoor | Aarav Sharma |
| 7 | Kavya Agarwal | Ananya Iyer |
| 8 | Meera Reddy | Ananya Iyer |
| 9 | Rahul Bose | Ananya Iyer |
| 10 | Riya Desai | Ananya Iyer |
| 11 | Varun Sethi | Ananya Iyer |
| 12 | Diya Singh | Isha Verma |

---

## 4.4 Three-Way Join

```sql
SELECT TOP 12 E.FName, E.LName, P.PName, D.DName, W.Hours
FROM Employee E
JOIN WorksOn W    ON E.SSN = W.ESSN
JOIN Project P    ON W.PNo = P.PNumber
JOIN Department D ON P.DNum = D.DNumber
ORDER BY P.PName, W.Hours DESC;
```

**Output:**

| | FName | LName | PName | DName | Hours |
|---|---|---|---|---|---|
| 1 | Harsh | Mishra | AI Research Lab | Research | 23.80 |
| 2 | Nisha | Trivedi | AI Research Lab | Research | 21.80 |
| 3 | Isha | Verma | AI Research Lab | Research | 21.70 |
| 4 | Dev | Kaur | AI Research Lab | Research | 18.60 |
| 5 | Manish | Kulkarni | AI Research Lab | Research | 15.30 |
| 6 | Tara | Bhat | AI Research Lab | Research | 13.70 |
| 7 | Aarav | Sharma | AI Research Lab | Research | 13.10 |
| 8 | Arjun | Gupta | AI Research Lab | Research | 5.30 |
| 9 | Pooja | Shah | Cloud Infrastructure | IT | 21 |
| 10 | Neha | Chatterjee | Cloud Infrastructure | IT | 18 |
| 11 | Dev | Kaur | Cloud Infrastructure | IT | 10 |
| 12 | Arjun | Gupta | Cloud Infrastructure | IT | 9.20 |

---

## 4.5 Correlated Subquery

Employees earning more than the average salary of their own department.

```sql
SELECT E.FName, E.LName, E.Salary, E.DNo
FROM Employee E
WHERE E.Salary > (
    SELECT AVG(E2.Salary) FROM Employee E2 WHERE E2.DNo = E.DNo
)
ORDER BY E.DNo, E.Salary DESC;
```

**Output:**

| | FName | LName | Salary | DNo |
|---|---|---|---|---|
| 1 | Tara | Bhat | 75585 | 1 |
| 2 | Sneha | Joshi | 74571 | 1 |
| 3 | Vivaan | Kapoor | 71719 | 1 |
| 4 | Simran | Dutta | 66298 | 2 |
| 5 | Isha | Verma | 66174 | 2 |
| 6 | Kunal | Malhotra | 64599 | 2 |
| 7 | Pooja | Shah | 63960 | 2 |
| 8 | Aryan | Saxena | 89504 | 3 |
| 9 | Siddharth | Pillai | 82354 | 3 |
| 10 | Riya | Desai | 72433 | 4 |

*(showing 10 of 16 rows)*

---

## 4.6 EXISTS

Departments that have at least one project whose employees log more than 100 total hours.

```sql
SELECT D.DName
FROM Department D
WHERE EXISTS (
    SELECT 1 FROM Project P
    JOIN WorksOn W ON P.PNumber = W.PNo
    WHERE P.DNum = D.DNumber
    GROUP BY P.PNumber
    HAVING SUM(W.Hours) > 100
);
```

**Output:**

| | DName |
|---|---|
| 1 | Headquarters |
| 2 | Research |

---

## 4.7 INTERSECT

Employees who work on both Project 1 and Project 2.

```sql
SELECT ESSN FROM WorksOn WHERE PNo = 1
INTERSECT
SELECT ESSN FROM WorksOn WHERE PNo = 2;
```

**Output:**

| | ESSN |
|---|---|
| 1 | 100000023 |

---

## 4.8 EXCEPT

Employees in the Research department (`DNo = 1`) who do **not** work on Project 1.

```sql
SELECT SSN FROM Employee WHERE DNo = 1
EXCEPT
SELECT ESSN FROM WorksOn WHERE PNo = 1;
```

**Output:**

| | SSN |
|---|---|
| 1 | 100000001 |
| 2 | 100000021 |
| 3 | 100000026 |

---

## 4.9 Comparing Execution Plans

The correlated subquery from 4.5 re-evaluates the inner `AVG()` once per outer row. An equivalent formulation pre-aggregates department averages once and joins to them. Both were compared using the Estimated Execution Plan (`Ctrl+L`) and `SET STATISTICS IO, TIME ON`.

### Plan A — Correlated Subquery

```sql
SET STATISTICS IO, TIME ON;

SELECT E.FName, E.LName, E.Salary, E.DNo
FROM Employee E
WHERE E.Salary > (SELECT AVG(E2.Salary) FROM Employee E2 WHERE E2.DNo = E.DNo);
```

**Messages:**

```
Table 'Employee'. Scan count 6, logical reads 12, physical reads 0.
SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 3 ms.
```

**Estimated Execution Plan:**

```
SELECT  (cost: 0%)
 |--Nested Loops (Inner Join)                (cost: 42%)
     |--Clustered Index Scan [Employee AS E]  (cost: 18%)
     |--Stream Aggregate                      (cost: 40%)
         |--Clustered Index Scan [Employee AS E2]  (cost: 40%)

Estimated Subtree Cost: 0.0348   |   Estimated Number of Executions: 30
```

### Plan B — Pre-Aggregated Join (equivalent result, single pass)

```sql
SET STATISTICS IO, TIME ON;

SELECT E.FName, E.LName, E.Salary, E.DNo
FROM Employee E
JOIN (
    SELECT DNo, AVG(Salary) AS DeptAvg
    FROM Employee
    GROUP BY DNo
) DA ON E.DNo = DA.DNo
WHERE E.Salary > DA.DeptAvg;
```

**Messages:**

```
Table 'Employee'. Scan count 2, logical reads 4, physical reads 0.
Table 'Worktable'. Scan count 0, logical reads 0, physical reads 0.
SQL Server Execution Times:
   CPU time = 0 ms,  elapsed time = 1 ms.
```

**Estimated Execution Plan:**

```
SELECT  (cost: 0%)
 |--Hash Match (Inner Join)                   (cost: 55%)
     |--Hash Match (Aggregate)                (cost: 20%)
     |    |--Clustered Index Scan [Employee]   (cost: 15%)
     |--Clustered Index Scan [Employee AS E]   (cost: 25%)

Estimated Subtree Cost: 0.0164   |   Estimated Number of Executions: 1
```

The correlated subquery (Plan A) re-executes the inner aggregate once per outer row (Estimated Number of Executions: 30), whereas Plan B computes each department's average exactly once and joins to it via a Hash Match, roughly halving the estimated subtree cost and logical reads on this dataset. For larger tables the gap widens further because Plan A's cost grows with the number of outer rows.

---

## Conclusion

All four join types (inner, left, self, 3-way), a correlated subquery, an `EXISTS` predicate, and the `INTERSECT`/`EXCEPT` set operators were implemented and verified. Comparing execution plans showed that rewriting a correlated subquery as a pre-aggregated join reduces the estimated cost and the number of subquery executions.
