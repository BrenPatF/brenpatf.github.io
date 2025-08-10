---
layout: post
title: "Extracting Pure Functionality from SQL Queries"
date: 2018-10-29
migrated: true
group: general-sql
categories: 
  - "oracle"
  - "plsql"
  - "sql"
  - "subquery-factor"
  - "testing"
tags: 
  - "oracle"
  - "plsql"
  - "sql"
  - "subquery-factor"
  - "unit-test"
---

In my last Oracle User Group presentation, [Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing), I discussed how the concept of the pure function can be extremely useful in the context of automated testing of database APIs.

In this article I show how the concept can also be useful in testing, and writing, SQL queries regardless of whether or not automated testing is in use. The idea is that queries often contain complex logic involving CASE, Nvl and other logical constructs, as well as retrieval of database data. If we could somehow separate out the pure logical part from the impure database accesses, we may be able to do more effective testing, since pure functions are inherently easier to test than impure ones. We will show this by means of a simple example against the Oracle HR demo schema.

Suppose we want to calculate an employee bonus using the following logic:

- Use a 10% multiplier applied to one of two salaries...
- ...for department managers, use the departmental average salary; for others, use their own salary
- For employees who have been previously employed, i.e. who have a job history record, add a further 10%
- For employees whose job is 'IT\_PROG', add a (well deserved 🙂) further 50%

Here is a query to calculate this:

```sql
WITH depsals AS (
  SELECT dep.department_id, dep.manager_id, Avg(emp.salary) avgsal
    FROM departments dep
    JOIN employees emp ON emp.department_id = dep.department_id
    GROUP BY Dep.department_id, dep.manager_id
)
SELECT emp.employee_id, emp.salary, dsl.avgsal,
       Round(Nvl(dsl.avgsal, emp.salary) * 0.1 *
       Nvl2(jhs.employee_id, 1.1, 1) *
       CASE job.job_id WHEN 'IT_PROG' THEN 1.5 ELSE 1 END) bonus
  FROM employees emp
  JOIN jobs job
    ON emp.job_id = job.job_id
  LEFT JOIN depsals dsl
    ON dsl.manager_id = emp.employee_id
  LEFT JOIN (SELECT employee_id FROM job_history GROUP BY employee_id) jhs
    ON jhs.employee_id = emp.employee_id
 ORDER BY 1
```
with results:
<div class="scrollbox">
<pre>
EMPLOYEE_ID     SALARY     AVGSAL      BONUS
----------- ---------- ---------- ----------
        100      24000 19333.3333       1933
        101      17000                  1870
        102      17000                  1870
        103       9000       5760        864
        104       6000                   900
        105       4800                   720
        106       4800                   720
        107       4200                   630
        108      12008 8601.33333        860
        109       9000                   900
        110       8200                   820
        111       7700                   770
        112       7800                   780
        113       6900                   690
        114      11000       4150        457
        115       3100                   310
        116       2900                   290
        117       2800                   280
        118       2600                   260
        119       2500                   250
        120       8000                   800
        121       8200 3475.55556        348
        122       7900                   869
        123       6500                   650
        124       5800                   580
        125       3200                   320
        126       2700                   270
        127       2400                   240
        128       2200                   220
        129       3300                   330
        130       2800                   280
        131       2500                   250
        132       2100                   210
        133       3300                   330
        134       2900                   290
        135       2400                   240
        136       2200                   220
        137       3600                   360
        138       3200                   320
        139       2700                   270
        140       2500                   250
        141       3500                   350
        142       3100                   310
        143       2600                   260
        144       2500                   250
        145      14000 8955.88235        896
        146      13500                  1350
        147      12000                  1200
        148      11000                  1100
        149      10500                  1050
        150      10000                  1000
        151       9500                   950
        152       9000                   900
        153       8000                   800
        154       7500                   750
        155       7000                   700
        156      10000                  1000
        157       9500                   950
        158       9000                   900
        159       8000                   800
        160       7500                   750
        161       7000                   700
        162      10500                  1050
        163       9500                   950
        164       7200                   720
        165       6800                   680
        166       6400                   640
        167       6200                   620
        168      11500                  1150
        169      10000                  1000
        170       9600                   960
        171       7400                   740
        172       7300                   730
        173       6100                   610
        174      11000                  1100
        175       8800                   880
        176       8600                   946
        177       8400                   840
        178       7000                   700
        179       6200                   620
        180       3200                   320
        181       3100                   310
        182       2500                   250
        183       2800                   280
        184       4200                   420
        185       4100                   410
        186       3400                   340
        187       3000                   300
        188       3800                   380
        189       3600                   360
        190       2900                   290
        191       2500                   250
        192       4000                   400
        193       3900                   390
        194       3200                   320
        195       2800                   280
        196       3100                   310
        197       3000                   300
        198       2600                   260
        199       2600                   260
        200       4400       4400        484
        201      13000       9500       1045
        202       6000                   600
        203       6500       6500        650
        204      10000      10000       1000
        205      12008      10154       1015
        206       8300                   830

107 rows selected.
</pre>
</div>

We see the bonus calculation in the select list with fields embedded from tables and a subquery. Setting up test data in multiple tables, and filtering out database noise can be a difficult task, so it would be nice if we could bypass that to test the calculation logic independently. If we are on version 12.1 or higher we can facilitate this by making the calculation into a WITH function, like this:

```sql
WITH FUNCTION calc_bonus(p_jhs_emp_id NUMBER, p_job_id VARCHAR2, p_salary NUMBER, p_avgsal NUMBER) RETURN NUMBER IS
BEGIN
  RETURN Round(0.1 *
    Nvl(p_avgsal, p_salary) * 
    CASE WHEN p_jhs_emp_id IS NULL THEN 1 ELSE 1.1 END *
    CASE p_job_id WHEN 'IT_PROG' THEN 1.5 ELSE 1 END);
END;
depsals AS (
  SELECT dep.department_id, dep.manager_id, Avg(emp.salary) avgsal
    FROM departments dep
    JOIN employees emp ON emp.department_id = dep.department_id
    GROUP BY Dep.department_id, dep.manager_id
)
SELECT emp.employee_id, emp.salary, dsl.avgsal,
       calc_bonus(jhs.employee_id, job.job_id, emp.salary, dsl.avgsal) bonus
  FROM employees emp
  JOIN jobs job
    ON emp.job_id = job.job_id
  LEFT JOIN depsals dsl
    ON dsl.manager_id = emp.employee_id
  LEFT JOIN (SELECT employee_id FROM job_history GROUP BY employee_id) jhs
    ON jhs.employee_id = emp.employee_id
 ORDER BY 1
```

Now the declared function, which is 'pure', separates out the calculation logic from the impure parts of the query that reference database fields. We can now test this function by replacing the rest of the query with a test data generator designed to cover all scenarios.

In the presentation referenced above I discussed how to assess test coverage properly, in terms of behavioural, or scenario, coverage, rather than the popular but spurious 'code coverage' metrics. I explained the value of thinking in terms of domain and subdomain partitioning to maximise true test coverage. If the subdomains are orthogonal (or independent) we can test behaviour across their partitions in parallel. What about the current case? We can see that we have three subdomains, each having two partitions, and in fact they are interdependent (because they multiply together an error in one factor could neutralise an error in another): that means we need 2x2x2 = 8 test records. There is no need to vary the base salary, so we will use a bind variable:

```sql
VAR SALARY NUMBER
EXEC :SALARY := 20000
```

The query with test data generator is then:

```sql
WITH FUNCTION calc_bonus(p_jhs_emp_id NUMBER, p_job_id VARCHAR2, p_salary NUMBER, p_avgsal NUMBER) RETURN NUMBER IS
BEGIN
  RETURN Round(0.1 *
    Nvl(p_avgsal, p_salary) * 
    CASE WHEN p_jhs_emp_id IS NULL THEN 1 ELSE 1.1 END *
    CASE p_job_id WHEN 'IT_PROG' THEN 1.5 ELSE 1 END);
END;
test_data AS (
  SELECT NULL jhs_emp_id, 'OTHER'   job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'OTHER'   job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT NULL jhs_emp_id, 'IT_PROG' job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'IT_PROG' job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT NULL jhs_emp_id, 'OTHER'   job_id, 10000 avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'OTHER'   job_id, 10000 avgsal FROM DUAL UNION ALL
  SELECT NULL jhs_emp_id, 'IT_PROG' job_id, 10000 avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'IT_PROG' job_id, 10000 avgsal FROM DUAL
)
SELECT dat.jhs_emp_id, dat.job_id,  dat.avgsal,
       calc_bonus(dat.jhs_emp_id, dat.job_id, :SALARY, dat.avgsal) bonus
  FROM test_data dat
 ORDER BY 1, 2, 3
```

<img src="/migrated_images/2018/10/Test-Combis.png" alt="Test data combinations" title="Test data combinations" />

Test results:

```
JHS_EMP_ID JOB_ID      AVGSAL      BONUS
---------- ------- ---------- ----------
         1 IT_PROG      10000       1650
         1 IT_PROG                  3300
         1 OTHER        10000       1100
         1 OTHER                    2200
           IT_PROG      10000       1500
           IT_PROG                  3000
           OTHER        10000       1000
           OTHER                    2000
```

The results can be checked manually, and there is probably little value in automating this beyond scripting.

Ok, but what if we are on a database version prior to 12.1, or for some reason we don't want to use a WITH function? In that case, we can do something similar, but not quite as cleanly because we will need to modify the code under test slightly, to reference the test data subquery:

```sql
WITH test_data AS (
  SELECT NULL jhs_emp_id, 'OTHER'   job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'OTHER'   job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT NULL jhs_emp_id, 'IT_PROG' job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'IT_PROG' job_id, NULL  avgsal FROM DUAL UNION ALL
  SELECT NULL jhs_emp_id, 'OTHER'   job_id, 10000 avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'OTHER'   job_id, 10000 avgsal FROM DUAL UNION ALL
  SELECT NULL jhs_emp_id, 'IT_PROG' job_id, 10000 avgsal FROM DUAL UNION ALL
  SELECT 1    jhs_emp_id, 'IT_PROG' job_id, 10000 avgsal FROM DUAL
)
SELECT dat.jhs_emp_id, dat.job_id,  dat.avgsal,
       Round(Nvl(dat.avgsal, :SALARY) * 0.1 *
       Nvl2(dat.jhs_emp_id, 1.1, 1) *
       CASE dat.job_id WHEN 'IT_PROG' THEN 1.5 ELSE 1 END) bonus
  FROM test_data dat
 ORDER BY 1, 2, 3
```

## Conclusion

We have shown how extracting pure functionality from a query can help in making testing more rigorous and modular.

We have also shown how the WITH function feature, new in v12.1, can be used to extract pure functions from the main SQL and so enhance modularity and testability. This is a usage for the feature that is not commonly noted, the advantage usually cited being replacement of database functions to avoid context switches.
