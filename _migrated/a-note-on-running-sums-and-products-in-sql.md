---
layout: post
title: "A Note on Running Sums and Products in SQL"
date: 2020-01-19
migrated: true
group: general-sql
categories: 
  - "model"
  - "oracle"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "oracle"
  - "recursive"
  - "sql"
  - "subquery-factor"
---

\[There is a recording on this article here: [Tweet](https://twitter.com/BrenPatF/status/1219149845505683459), and also in my GitHub project, [Small SQL projects](https://github.com/BrenPatF/oracle_sql_projects).\]

It's a common requirement to compute running sums over time in SQL; for example, to find sales volumes to date. This is easy to do using analytic functions, like this, schematically:

```sql
SELECT SUM(sales) OVER (PARTITION By partition_key ORDER BY date)
  FROM ...
```

The ORDER BY clause implicitly adds a window range of UNBOUNDED PRECEDING, and if you omit it you get the overall totals by partition\_key.

Recently I needed to compute the products of some numeric factors across records over time. This kind of requirement often arises in financial calculations, and is slightly more tricky since Oracle doesn't have an analytic function for products as it does for sums. However, we can achieve the same functionality using the well known mathematical equivalence:

```
	log(xy) = log(x) + log(y)
```

and therefore:

```
	xy = exp(log(x) + log(y))
```

More generally, if we have a set of N records <img src="/migrated_images/2020/01/Set-N.gif" alt="" title="" />, and a function, f, defined on a record, we can write the running sum, for function f, and n in (1..N) as:

<img src="/migrated_images/2020/01/Product.gif" alt="Running sum" title="Running sum" />

and the running product as:

<img src="/migrated_images/2020/01/Sum.gif" alt="Running product" title="Running product" />

Then from the above equivalence, we can write:

<img src="/migrated_images/2020/01/Equation.gif" alt="Running product with exp and log" title="Running product with exp and log" />

This means we can get the running products in SQL using the analytic SUM combined with the (natural) log (LN in Oracle SQL) and exp functions. Let's see how it works using Oracle's HR demo schema. We'll take the employees table and use:

- department\_id as the partition key
- employee\_id as the dimension to order by
- salary as the measure

To start with, let's get the running and total sums, showing results for department\_id = 60:

```sql
SELECT department_id, employee_id, salary,
       SUM(salary) OVER (PARTITION BY department_id ORDER BY employee_id) running_sum,
       SUM(salary) OVER (PARTITION BY department_id) total_sum
  FROM employees
 ORDER BY department_id, employee_id
/
Sums

DEPARTMENT_ID EMPLOYEE_ID     SALARY RUNNING_SUM  TOTAL_SUM
------------- ----------- ---------- ----------- ----------
           60         103       9000        9000      28800
           60         104       6000       15000      28800
           60         105       4800       19800      28800
           60         106       4800       24600      28800
           60         107       4200       28800      28800
```

Next, let's use the above equivalence to get the running and total products of the expression (1 + salary/10000), which we'll call mult:

```sql
SELECT department_id, employee_id, salary, (1 + salary/10000) mult,
       EXP(SUM(LN((1 + salary/10000))) OVER (PARTITION BY department_id ORDER BY employee_id)) running_prod,
       EXP(SUM(LN((1 + salary/10000))) OVER (PARTITION BY department_id)) total_prod
  FROM employees
 ORDER BY department_id, employee_id
/
Products

DEPARTMENT_ID EMPLOYEE_ID     SALARY       MULT RUNNING_PROD TOTAL_PROD
------------- ----------- ---------- ---------- ------------ ----------
           60         103       9000        1.9          1.9 9.45551872
           60         104       6000        1.6         3.04 9.45551872
           60         105       4800       1.48       4.4992 9.45551872
           60         106       4800       1.48     6.658816 9.45551872
           60         107       4200       1.42   9.45551872 9.45551872
```

If we didn't have this technique we could compute the results using explicit recursion, either by MODEL clause, or by recursive subquery factors. Let's do it those ways out of interest. First here's a MODEL clause solution:

```sql
WITH multipliers AS (
SELECT department_id, employee_id, salary, (1 + salary/10000) mult, 
       COUNT(*) OVER (PARTITION BY department_id) n_emps
  FROM employees
)
SELECT department_id, employee_id, salary, mult, running_prod, total_prod
  FROM multipliers
 MODEL
 	PARTITION BY (department_id)
 	DIMENSION BY (Row_Number() OVER (PARTITION BY department_id ORDER BY employee_id) rn)
 	MEASURES (employee_id, salary, mult, mult running_prod, mult total_prod, n_emps)
 	RULES (
 		running_prod[rn > 1] = mult[CV()] * running_prod[CV() - 1],
 		total_prod[any] = running_prod[n_emps[CV()]]
 	)
 ORDER BY department_id, employee_id
/
```

Finally, here's a solution using recursive subquery factors:

```sql
WITH multipliers AS (
SELECT department_id, employee_id, salary, (1 + salary/10000) mult, 
       Row_Number() OVER (PARTITION BY department_id ORDER BY employee_id) rn,
       COUNT(*) OVER (PARTITION BY department_id) n_emps
  FROM employees
 WHERE department_id = 60
), rsf (department_id, employee_id, rn, salary, mult, running_prod) AS (
	SELECT department_id, employee_id, rn, salary, mult, mult running_prod
	  FROM multipliers
	 WHERE rn = 1
	UNION ALL
	SELECT m.department_id, m.employee_id, m.rn, m.salary, m.mult, r.running_prod * m.mult
	  FROM rsf r
	  JOIN multipliers m
	    ON m.rn = r.rn + 1
	   AND m.department_id = r.department_id
)
SELECT department_id, employee_id, salary, mult, running_prod, 
       Last_Value(running_prod) OVER (PARTITION BY department_id) total_prod
  FROM rsf
 ORDER BY department_id, employee_id
/
```

You can see the scripts and full output on my new GitHub project, [Small SQL projects](https://github.com/BrenPatF/oracle_sql_projects), in the sums\_products folder.

You can get the full detail on using analytic functions from the Oracle doc: [SQL for Analysis and Reporting](https://docs.oracle.com/en/database/oracle/oracle-database/19/dwhsg/sql-analysis-reporting-data-warehouses.html#GUID-20EFBF1E-F79D-4E4A-906C-6E496EECA684)

See also: [Analytic and Recursive SQL by Example](https://brenpatf.github.io/migrated/analytic-and-recursive-sql-by-example/)

