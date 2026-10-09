# EXPERIMENT 3 `[CO2]`

**Aim:** Create an Employee–Department–Project schema. Insert at least 30 employees across 5 departments and 8 projects. Write SQL queries demonstrating selection, projection, aggregates, `GROUP BY`, `HAVING`, `CASE` expressions, and `ORDER BY`.

---

## 3.1 Database & Schema Creation

The classic COMPANY schema is used. `Department` and `Employee` reference each other (a department has a manager who is an employee; an employee belongs to a department), so the two mutual foreign keys are added with `ALTER TABLE` after both tables exist.

```sql
CREATE DATABASE CompanyDB;
GO
USE CompanyDB;
GO

CREATE TABLE Department (
    DNumber       INT PRIMARY KEY,
    DName         VARCHAR(30) NOT NULL UNIQUE,
    MgrSSN        CHAR(9) NULL,
    MgrStartDate  DATE NULL
);
GO

CREATE TABLE Employee (
    SSN       CHAR(9) PRIMARY KEY,
    FName     VARCHAR(30) NOT NULL,
    LName     VARCHAR(30) NOT NULL,
    Bdate     DATE,
    Sex       CHAR(1) CHECK (Sex IN ('M','F')),
    Salary    DECIMAL(10,2) NOT NULL CHECK (Salary > 0),
    SuperSSN  CHAR(9) NULL,
    DNo       INT NOT NULL,
    CONSTRAINT FK_Emp_Dept FOREIGN KEY (DNo) REFERENCES Department(DNumber),
    CONSTRAINT FK_Emp_Super FOREIGN KEY (SuperSSN) REFERENCES Employee(SSN)
);
GO

ALTER TABLE Department
    ADD CONSTRAINT FK_Dept_Manager FOREIGN KEY (MgrSSN) REFERENCES Employee(SSN);
GO

CREATE TABLE Project (
    PNumber  INT PRIMARY KEY,
    PName    VARCHAR(30) NOT NULL UNIQUE,
    DNum     INT NOT NULL,
    CONSTRAINT FK_Proj_Dept FOREIGN KEY (DNum) REFERENCES Department(DNumber)
);
GO

CREATE TABLE WorksOn (
    ESSN   CHAR(9) NOT NULL,
    PNo    INT NOT NULL,
    Hours  DECIMAL(4,1) NOT NULL,
    CONSTRAINT PK_WorksOn PRIMARY KEY (ESSN, PNo),
    CONSTRAINT FK_Works_Emp  FOREIGN KEY (ESSN) REFERENCES Employee(SSN),
    CONSTRAINT FK_Works_Proj FOREIGN KEY (PNo)  REFERENCES Project(PNumber)
);
GO
```

**Output:**

```
Command(s) completed successfully.
```

---

## 3.2 Sample Data — 5 Departments, 30 Employees, 8 Projects, 53 Assignments

```sql
INSERT INTO Department (DNumber, DName) VALUES
    (1, 'Research'), (2, 'Administration'), (3, 'Headquarters'),
    (4, 'Sales'), (5, 'IT');

-- 30 employees inserted (6 per department); abbreviated here — see 3.3 for full listing
INSERT INTO Employee (SSN, FName, LName, Bdate, Sex, Salary, SuperSSN, DNo) VALUES
    ('100000001', 'Aarav', 'Sharma', '1988-04-12', 'M', 65024.00, '100000003', 1),
    ('100000002', 'Isha',  'Verma',  '1990-07-03', 'F', 66174.00, '100000003', 2),
    ('100000003', 'Rohan', 'Mehta',  '1979-11-21', 'M', 78041.00, NULL,        3),
    -- ... 27 further rows ...
    ('100000030','Harsh','Mishra','1985-02-18','M', 84338.00, '100000005', 5);

UPDATE Department SET MgrSSN = '100000001', MgrStartDate = '2022-01-01' WHERE DNumber = 1;
UPDATE Department SET MgrSSN = '100000002', MgrStartDate = '2022-01-01' WHERE DNumber = 2;
UPDATE Department SET MgrSSN = '100000003', MgrStartDate = '2022-01-01' WHERE DNumber = 3;
UPDATE Department SET MgrSSN = '100000004', MgrStartDate = '2022-01-01' WHERE DNumber = 4;
UPDATE Department SET MgrSSN = '100000005', MgrStartDate = '2022-01-01' WHERE DNumber = 5;

INSERT INTO Project (PNumber, PName, DNum) VALUES
    (1, 'Database Migration', 1), (2, 'AI Research Lab', 1),
    (3, 'Payroll System', 2),     (4, 'Office Renovation', 3),
    (5, 'Market Expansion', 4),   (6, 'Customer CRM Upgrade', 4),
    (7, 'Network Security', 5),   (8, 'Cloud Infrastructure', 5);

-- 53 WorksOn rows assigning each employee to 1-3 projects (abbreviated)
INSERT INTO WorksOn (ESSN, PNo, Hours) VALUES
    ('100000001', 1, 18.5), ('100000001', 2, 9.0),
    ('100000002', 3, 22.0), ('100000003', 4, 15.5);
    -- ... remaining rows ...
```

**Output:**

```
(5 rows affected)
(30 rows affected)
(1 row affected)
(1 row affected)
(1 row affected)
(1 row affected)
(1 row affected)
(8 rows affected)
(53 rows affected)
```

**Department table after load**

```sql
SELECT DNumber, DName, MgrSSN, MgrStartDate FROM Department ORDER BY DNumber;
```

**Output:**

| | DNumber | DName | MgrSSN | MgrStartDate |
|---|---|---|---|---|
| 1 | 1 | Research | 100000001 | 2022-01-01 |
| 2 | 2 | Administration | 100000002 | 2022-01-01 |
| 3 | 3 | Headquarters | 100000003 | 2022-01-01 |
| 4 | 4 | Sales | 100000004 | 2022-01-01 |
| 5 | 5 | IT | 100000005 | 2022-01-01 |

**Project table after load**

```sql
SELECT PNumber, PName, DNum FROM Project ORDER BY PNumber;
```

**Output:**

| | PNumber | PName | DNum |
|---|---|---|---|
| 1 | 1 | Database Migration | 1 |
| 2 | 2 | AI Research Lab | 1 |
| 3 | 3 | Payroll System | 2 |
| 4 | 4 | Office Renovation | 3 |
| 5 | 5 | Market Expansion | 4 |
| 6 | 6 | Customer CRM Upgrade | 4 |
| 7 | 7 | Network Security | 5 |
| 8 | 8 | Cloud Infrastructure | 5 |

---

## 3.3 Selection

```sql
SELECT FName, LName, Salary, DNo
FROM Employee
WHERE Salary > 70000
ORDER BY Salary DESC;
```

**Output:**

| | FName | LName | Salary | DNo |
|---|---|---|---|---|
| 1 | Aryan | Saxena | 89504 | 3 |
| 2 | Harsh | Mishra | 84338 | 5 |
| 3 | Kabir | Nair | 83295 | 5 |
| 4 | Siddharth | Pillai | 82354 | 3 |
| 5 | Aditya | Rao | 82094 | 3 |
| 6 | Arjun | Gupta | 81782 | 5 |
| 7 | Nisha | Trivedi | 81575 | 3 |
| 8 | Priya | Patel | 79279 | 3 |
| 9 | Rohan | Mehta | 78041 | 3 |
| 10 | Neha | Chatterjee | 76857 | 5 |
| 11 | Tara | Bhat | 75585 | 1 |
| 12 | Sneha | Joshi | 74571 | 1 |
| 13 | Riya | Desai | 72433 | 4 |
| 14 | Manish | Kulkarni | 72336 | 5 |
| 15 | Vivaan | Kapoor | 71719 | 1 |

*(15 rows returned)*

---

## 3.4 Projection

```sql
SELECT FName, LName, DNo
FROM Employee
ORDER BY FName;
```

**Output:**

| | FName | LName | DNo |
|---|---|---|---|
| 1 | Aarav | Sharma | 1 |
| 2 | Aditya | Rao | 3 |
| 3 | Ananya | Iyer | 4 |
| 4 | Anjali | Bansal | 5 |
| 5 | Arjun | Gupta | 5 |
| 6 | Aryan | Saxena | 3 |
| 7 | Dev | Kaur | 1 |
| 8 | Diya | Singh | 2 |
| 9 | Harsh | Mishra | 5 |
| 10 | Isha | Verma | 2 |

*(showing 10 of 30 rows)*

---

## 3.5 Aggregates with GROUP BY

```sql
SELECT D.DName,
       COUNT(E.SSN)      AS EmpCount,
       AVG(E.Salary)     AS AvgSalary,
       MIN(E.Salary)     AS MinSalary,
       MAX(E.Salary)     AS MaxSalary,
       SUM(E.Salary)     AS TotalPayroll
FROM Employee E
JOIN Department D ON E.DNo = D.DNumber
GROUP BY D.DName
ORDER BY AvgSalary DESC;
```

**Output:**

| | DName | EmpCount | AvgSalary | MinSalary | MaxSalary | TotalPayroll |
|---|---|---|---|---|---|---|
| 1 | Headquarters | 6 | 82141.17 | 78041 | 89504 | 492847 |
| 2 | IT | 6 | 77965.67 | 69186 | 84338 | 467794 |
| 3 | Research | 6 | 69013.83 | 58543 | 75585 | 414083 |
| 4 | Sales | 6 | 65223 | 52614 | 72433 | 391338 |
| 5 | Administration | 6 | 61204.33 | 49231 | 66298 | 367226 |

---

## 3.6 GROUP BY with HAVING

```sql
SELECT D.DName, COUNT(*) AS EmpCount, ROUND(AVG(E.Salary),2) AS AvgSalary
FROM Employee E
JOIN Department D ON E.DNo = D.DNumber
GROUP BY D.DName
HAVING AVG(E.Salary) > 65500
ORDER BY AvgSalary DESC;
```

**Output:**

| | DName | EmpCount | AvgSalary |
|---|---|---|---|
| 1 | Headquarters | 6 | 82141.17 |
| 2 | IT | 6 | 77965.67 |
| 3 | Research | 6 | 69013.83 |

*Administration and Sales are excluded — their average salary does not exceed 65,500.*

---

## 3.7 CASE Expression

```sql
SELECT FName, LName, Salary,
    CASE
        WHEN Salary >= 80000 THEN 'Senior'
        WHEN Salary >= 60000 THEN 'Mid-Level'
        ELSE 'Junior'
    END AS SalaryBand
FROM Employee
ORDER BY Salary DESC;
```

**Output:**

| | FName | LName | Salary | SalaryBand |
|---|---|---|---|---|
| 1 | Aryan | Saxena | 89504 | Senior |
| 2 | Harsh | Mishra | 84338 | Senior |
| 3 | Kabir | Nair | 83295 | Senior |
| 4 | Siddharth | Pillai | 82354 | Senior |
| 5 | Aditya | Rao | 82094 | Senior |
| 6 | Arjun | Gupta | 81782 | Senior |
| 7 | Nisha | Trivedi | 81575 | Senior |
| 8 | Priya | Patel | 79279 | Mid-Level |
| 9 | Rohan | Mehta | 78041 | Mid-Level |
| 10 | Neha | Chatterjee | 76857 | Mid-Level |

*(showing 10 of 30 rows)*

---

## 3.8 ORDER BY — Multiple Columns

```sql
SELECT DNo, LName, FName, Salary
FROM Employee
ORDER BY DNo ASC, Salary DESC;
```

**Output:**

| | DNo | LName | FName | Salary |
|---|---|---|---|---|
| 1 | 1 | Bhat | Tara | 75585 |
| 2 | 1 | Joshi | Sneha | 74571 |
| 3 | 1 | Kapoor | Vivaan | 71719 |
| 4 | 1 | Menon | Karan | 68641 |
| 5 | 1 | Sharma | Aarav | 65024 |
| 6 | 1 | Kaur | Dev | 58543 |
| 7 | 2 | Dutta | Simran | 66298 |
| 8 | 2 | Verma | Isha | 66174 |

*(showing 8 of 30 rows — sorted first by department, then salary descending within each department)*

---

## Conclusion

The COMPANY schema (`Department`, `Employee`, `Project`, `WorksOn`) was populated with 30 employees across 5 departments and 8 projects. `SELECT` with `WHERE`, projection, `GROUP BY` aggregates, `HAVING` filters, `CASE`-based derived columns, and multi-column `ORDER BY` were all demonstrated successfully.
