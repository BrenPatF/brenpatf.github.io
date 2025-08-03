---
layout: post
title: "SQL and Modularity: Patterns, Anti-Patterns and the Kitchen Sink"
date: 2013-09-08
migrated: true
group: design
categories: 
  - "design"
  - "object"
  - "oracle"
  - "performance"
  - "sql"
tags: 
  - "dal"
  - "data-access-layer"
  - "oracle"
  - "performance-2"
  - "sql"
  - "view"
---

There's a lot of importance placed on code re-use in the database development world. In traditional procedural programming languages, such as C or Fortran, the value of modular programming and its application to promoting code re-use is well known and understood. When SQL enters the picture, however, the situation becomes less clear, and there is less consensus on how best to apply the traditional concept of modularity.

This article will consider the concept of modularity, and how it may best be applied to SQL, from the perspective of 'patterns' and 'anti-patterns'. Here is a definition of these terms from Wikipedia, [Anti-pattern](http://en.wikipedia.org/wiki/Anti-pattern):

_An anti-pattern (or antipattern) is a pattern used in social or business operations or software engineering that may be commonly used but is ineffective and/or counterproductive in practice.\[1\]\[2\]_

The term was coined in 1995 by Andrew Koenig,\[3\] inspired by Gang of Four's book Design Patterns, which developed the concept of design patterns in the software field, since now a days software is used for any business from banks to small stores. The term was widely popularized three years later by the book AntiPatterns, which extended the use of the term beyond the field of software design and into general social interaction. According to the authors of the latter, there must be at least two key elements present to formally distinguish an actual anti-pattern from a simple bad habit, bad practice, or bad idea:

- Some repeated pattern of action, process or structure that initially appears to be beneficial, but ultimately produces more bad consequences than beneficial results, and
- An alternative solution exists that is clearly documented, proven in actual practice and repeatable

## Procedural Modularity

Modularity starts from the idea that a complex design can generally be broken down into a set of less complex component modules that is easier to work with. In programming terms, a long main program would be broken down into smaller subroutines, with a much shorter main program that calls the subroutines.

From this starting point emerges the possibility of code re-use, whereby the decomposition into modules aims at identifying common logic that can be placed in generic modules and called in multiple places. A simple example of this would be an error-logging module in PL/SQL that would write any Oracle errors to a table along with call stack information, that could be called wherever such errors need to be trapped. This might be termed an error-logging pattern, and it's clear that this kind of code re-use can lead to simpler and more maintainable systems.

## SQL Modularity: Design Patterns

### Transactional APIs

The concept of transactions is important for modular design within a database application.

The Oracle manual, [Oracle Database Concepts](http://docs.oracle.com/cd/E11882_01/server.112/e40540/toc.htm), defines a transaction thus: _A transaction is a logical unit of work that contains one or more SQL statements. A transaction is an atomic unit. The effects of all the SQL statements in a transaction can be either all committed (applied to the database) or all rolled back (undone from the database)._

When a transaction needs to be performed in more than one place, then the code can be placed in a PL/SQL module, sometimes called a 'transactional API'. This is really just the database-specific version of standard modularity, and is obviously a good design pattern to follow.

### Data Access Layers

Transactional APIs are often used to form a _Data Access Layer (DAL)_ for front end applications written in languages such as Java. Where the front end requires a record set from the database, the APIs may return a reference cursor, which is essentially a pointer to the data, and avoids the overhead of passing the whole data set at once. The data access layer pattern has a number of important advantages:

- performance is enhanced through reduced network traffic between application server and database
- SQL operates in an efficient set-based fashion for retrieving data in batches
- the PL/SQL language, highly integrated with SQL, is specifically designed for database processing
- storing the database processing code in database packages promotes modularity and code re-use

It is considered best practice to use data access layers even for clients, such as Oracle Forms, that have an embedded PL/SQL engine.

Of course, in order to achieve these benefits the data access layer has to be correctly written, avoiding the anti-pattern pitfalls discussed later. In particular, 'kitchen sink' style APIs that return far more data than required, to promote re-use, must be avoided; different client programs requiring different data should have separate APIs.

### Views

Database views have been available in Oracle SQL from the earliest versions and can be used to avoid duplicating a complex SQL query that might be needed in multiple places.

This approach could be seens as a special case of transactional modularity, where a single query is the transaction, and might be regarded as a design pattern for re-use of SQL statements.

Views may form an alternative kind of data access layer, typically used by reporting tools such as Business Objects, and may also be used in conjumction with an API-based layer, which is a common approach in Oracle Forms applications.

### Within-SQL Modularity

SQL is essentially a declarative, rather than procedural language for retrieving (and updating etc.) data from relational databases. The original idea was that the programmer specifies the tables and columns where the data are stored, as well as how the tables are related through key values, but does not specify algorithms for retrieving the data: The retrieval algorithms are performed by the SQL engine 'under the covers'. There might therefore seem to be little scope for modularity within a SQL select statement. However, this is not quite true for a couple of reasons: First, logical paths have to be specified between the tables, and the same path may need to be specified multiple times from different starting points; for example, the path to billing and shipping addresses on a sales order would typically involve the same sequence of steps from different id columns; second, as SQL has evolved, procedural capabilities have been added, such as analytic functions and recursion.

In Oracle in-line views were introduced in v7.2, and are in a sense a first step in modularising an SQL statement, followed in v9.2 by the 'WITH' clause for subquery factoring. Analytic functions were introduced in v8i.

Here is an example based on Oracle's HR demo schema. Suppose we want a list of employees with their current and previous jobs (if any), the same for their manager, and a count of the number of subordinates they have. To get the previous jobs, we need to find the latest records in the job\_history table for the employee and his manager separately. This can be done using subqueries, but that is inefficient and it is normally better to join to aggregation views that use the DENSE\_RANK clause to allow the required previous records to be obtained in a single pass each. Similarly, the subordinate count could be done by a scalar subquery, but again performance would usually dictate the use of another aggregation view.

Here is a query using in-line views to achieve this:

```sql
SELECT emp.last_name || ', ' || emp.first_name name,
       job.job_title,
       job_p.job_title             job_title_prior,
       emp_m.last_name || ', ' || emp_m.first_name name_mgr,
       job_m.job_title             job_title_mgr,
       job_pm.job_title            job_title_mgr_prior,
       sub.n_sub
  FROM hr.employees                emp
  JOIN hr.jobs                     job
    ON job.job_id                  = emp.job_id
  LEFT JOIN (SELECT employee_id,
                    Max (job_id) KEEP (DENSE_RANK LAST ORDER BY end_date) job_id
               FROM hr.job_history
              GROUP BY employee_id
            )                      jhs
    ON jhs.employee_id             = emp.employee_id
  LEFT JOIN hr.jobs                job_p
    ON job_p.job_id                = jhs.job_id
  LEFT JOIN hr.employees           emp_m
    ON emp_m.employee_id           = emp.manager_id
  LEFT JOIN hr.jobs                job_m
    ON job_m.job_id                = emp_m.job_id
  LEFT JOIN (SELECT employee_id,
                    Max (job_id) KEEP (DENSE_RANK LAST ORDER BY end_date) job_id
               FROM hr.job_history
              GROUP BY employee_id
            )                      jhs_m
    ON jhs_m.employee_id           = emp.manager_id
  LEFT JOIN hr.jobs                job_pm
    ON job_pm.job_id               = jhs_m.job_id
  LEFT JOIN (SELECT manager_id,
                    Count(*)       n_sub
               FROM hr.employees
              GROUP BY manager_id
            )                      sub
    ON sub.manager_id              = emp.employee_id
 WHERE emp.department_id           = 30
 ORDER BY 1
```

Here is a query using subquery factors to achieve the same:

```sql
WITH jhs_f AS (
SELECT employee_id,
       Max (job_id) KEEP (DENSE_RANK LAST ORDER BY end_date) job_id
  FROM hr.job_history
 GROUP BY employee_id
), sub_f AS (
SELECT manager_id,
       Count(*)                    n_sub
  FROM hr.employees
 GROUP BY manager_id
)
SELECT emp.last_name || ', ' || emp.first_name name,
       job.job_title,
       job_p.job_title             job_title_prior,
       emp_m.last_name || ', ' || emp_m.first_name name_mgr,
       job_m.job_title             job_title_mgr,
       job_pm.job_title            job_title_mgr_prior,
       sub.n_sub
  FROM hr.employees                emp
  JOIN hr.jobs                     job
    ON job.job_id                  = emp.job_id
  LEFT JOIN jhs_f                  jhs
    ON jhs.employee_id             = emp.employee_id
  LEFT JOIN hr.jobs                job_p
    ON job_p.job_id                = jhs.job_id
  LEFT JOIN hr.employees           emp_m
    ON emp_m.employee_id           = emp.manager_id
  LEFT JOIN hr.jobs                job_m
    ON job_m.job_id                = emp_m.job_id
  LEFT JOIN jhs_f                  jhs_m
    ON jhs_m.employee_id           = emp.employee_id
  LEFT JOIN hr.jobs                job_pm
    ON job_pm.job_id               = jhs_m.job_id
  LEFT JOIN sub_f                  sub
    ON sub.manager_id              = emp.employee_id
 WHERE emp.department_id           = 30
 ORDER BY 1
```

The second query, although only a line shorter, could be said to be more modular in two ways:

1. The more complex processing is placed at the beginning, prior to the main select, which now contains only simple joins. This might be said to parallel the procedural modularity practice of having a simple main program calling subroutines, and may help maintainability
2. A single subquery factor replaces the two inline views for previous jobs, a more modular design, and one that may be more efficient for larger data sets since Oracle generally materialises subquery factors referenced multiple times

This approach might be regarded as a design pattern for modularity within individual SQL statements.

### ANSI Join Syntax

The queries above are written using ANSI join syntax, introduced in v9. Oracle SQL originally used its own proprietary syntax, as shown below for the same query requirement:

```sql
WITH jhs_f AS (
SELECT employee_id,
       Max (job_id) KEEP (DENSE_RANK LAST ORDER BY end_date) job_id
  FROM hr.job_history
 GROUP BY employee_id
), sub_f AS (
SELECT manager_id,
       Count(*)                    n_sub
  FROM hr.employees
 GROUP BY manager_id
)
SELECT emp.last_name || ', ' || emp.first_name name,
       job.job_title,
       job_p.job_title             job_title_prior,
       emp_m.last_name || ', ' || emp_m.first_name name_mgr,
       job_m.job_title             job_title_mgr,
       job_pm.job_title            job_title_mgr_prior,
       sub.n_sub
  FROM hr.employees                emp,
       hr.jobs                     job,
       jhs_f                       jhs,
       hr.jobs                     job_p,
       hr.employees                emp_m,
       hr.jobs                     job_m,
       jhs_f                       jhs_m,
       hr.jobs                     job_pm,
       sub_f                       sub
 WHERE emp.department_id           = 30
   AND job.job_id                  = emp.job_id
   AND jhs.employee_id (+)         = emp.employee_id
   AND job_p.job_id (+)            = jhs.job_id
   AND emp_m.employee_id (+)       = emp.manager_id
   AND job_m.job_id (+)            = emp_m.job_id
   AND jhs_m.employee_id (+)       = emp.employee_id
   AND job_pm.job_id (+)           = jhs_m.job_id
   AND sub.manager_id (+)          = emp.employee_id
 ORDER BY 1
```

The tables are listed together, join clauses are in a single block not separated from constraints, and outer joins are specified using a (+) token against every column in the 'left' table. The outer join syntax leads to widespread bugs when developers miss the (+) from one of the columns, which silently converts the join to an inner join.

Some of the advantages of the newer syntax are:

- Greater functionality is available, including full outer joining
- Outer joining is much harder to get wrong
- The syntax follows an ANSI standard
- Locating the join conditions with the table being joined appears to be more modular and readable

ANSI join syntax might therefore be considered a good design pattern to follow.

## SQL Modularity: Design Anti-patterns

<img src="/migrated_images/2013/09/everything-but-the-kitchen-sink-IDIOM.png" alt="everything-but-the-kitchen-sink-IDIOM" title="everything-but-the-kitchen-sink-IDIOM" />

Here is a thread from Tom Kyte's AskTom forum dealing with an SQL anti-pattern that is unfortunately common, and strongly opposed by Tom Kyte: [Considering SQL as a Service](http://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:672724700346558185). The idea behind the anti-pattern seems to be to avoid repeating even simple table joins in SQL by hiding them in a special type of data access layer designed to be called within individual SQL statements. This results in over-complex 'kitchen-sink' SQL within the layer itself and performance problems in the 'client' SQL. In addition, complex PL/SQL involving object types and arrays tends to be needed to glue it all together; the approach thus achieves the opposite of its intended purpose - simplification - in the manner of a classic anti-pattern.

We'll illustrate the anti-pattern by extending the HR example used above and working through an example. First we'll use the PL/SQL packaged procedure variant, then look at an older form based on views.

### APIs as SQL Building Blocks Anti-pattern

Let's suppose that we start from the idea that we should centralise the SQL for employee information in a re-useable API. We could take the SQL above and add in department name, address and manager information to make it more general. We might think initially of making the API a function taking an employee id as input and returning a record. But what if we needed the information for a list of employees? It would be inefficient to call a function for every record in the list, which might lead us to think of making the function take a list as input and return a list of records. We might therefore define object and array types, and a function with the following signature:

```sql
FUNCTION Emp_Info_List (p_emp_id_list SYS.ODCINumberList) RETURN emp_info_list_type;
```

\[All code and output referenced is available at a link at the bottom of this article.\]

Now let's consider a scenario in which we have to provide an API for a web front end following the design pattern of returning a reference cursor. The data required are the following details for all employees in a given department:

- employee name
- manager name
- list of subordinates

The output for department 30 would be:

```
NAME                      NAME_MGR               NAME_SUB
------------------------- ---------------------- -------------------
Baida, Shelli             Raphaely, Den
Colmenares, Karen         Raphaely, Den
Himuro, Guy               Raphaely, Den
Khoo, Alexander           Raphaely, Den
Raphaely, Den             King, Steven           Baida, Shelli
Raphaely, Den             King, Steven           Colmenares, Karen
Raphaely, Den             King, Steven           Himuro, Guy
Raphaely, Den             King, Steven           Khoo, Alexander
Raphaely, Den             King, Steven           Tobias, Sigal
Tobias, Sigal             Raphaely, Den

10 rows selected.
```

Here is a possible procedure implementation:

```sql
PROCEDURE Get_Mgr_Subs_KS (p_dept_id PLS_INTEGER, x_mgr_sub_cur OUT SYS_REFCURSOR) IS
  l_emp_id_list SYS.ODCINumberList;
BEGIN

  SELECT employee_id
    BULK COLLECT INTO l_emp_id_list
    FROM hr.employees
   WHERE department_id = p_dept_id;

  OPEN x_mgr_sub_cur FOR
  SELECT t.name,
         t.name_mgr,
         e.last_name || ', ' || e.first_name
    FROM TABLE (KSink_Emp.Emp_Info_List (l_emp_id_list)) t
    LEFT JOIN hr.employees e
      ON e.manager_id = t.employee_id
   ORDER BY 1, 2, 3;

END Get_Mgr_Subs_KS;
```

The first step is to get the list of employees for the department, which we then pass into the API, wrapped in the TABLE key word, and join the employees table to get the subordinates. Three SQL select statements are executed. You might argue that I have over-complicated this by having the API take a list of employees rather than the department id, but remember that in this design anti-pattern the API can't be designed for one specific caller, and the list input is more general. It is intended to cater for all calls for employee information so in practice such compromises will happen frequently.

We can compare this to an alternative implementation in which we simply join the tables required:

```sql
PROCEDURE Get_Mgr_Subs_SQL (p_dept_id PLS_INTEGER, x_mgr_sub_cur OUT SYS_REFCURSOR) IS
BEGIN

  OPEN x_mgr_sub_cur FOR
  SELECT e.last_name || ', ' || e.first_name,
         m.last_name || ', ' || m.first_name,
         s.last_name || ', ' || s.first_name
    FROM hr.employees e
    LEFT JOIN hr.employees m
      ON m.employee_id = e.manager_id
    LEFT JOIN hr.employees s
      ON s.manager_id = e.employee_id
   WHERE e.department_id = p_dept_id
   ORDER BY 1, 2, 3;

END Get_Mgr_Subs_SQL;
```

This joins three tables in one statement while the earlier procedure effectively makes a join through PL/SQL using an array, which is arguably slightly more complicated. In any case the real problems become apparent when you compare the execution plans. I have written a test driver program that calls each of the APIs and loops over the returned cursor.

Taking the plan for the straight SQL implementation first:

```
-------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name              | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |                   |      1 |        |     10 |00:00:00.01 |      14 |       |       |          |
|   1 |  SORT ORDER BY                         |                   |      1 |     35 |     10 |00:00:00.01 |      14 |  2048 |  2048 | 2048  (0)|
|   2 |   NESTED LOOPS OUTER                   |                   |      1 |     35 |     10 |00:00:00.01 |      14 |       |       |          |
|*  3 |    HASH JOIN OUTER                     |                   |      1 |      6 |      6 |00:00:00.01 |      10 |  1281K|  1281K|  544K (0)|
|   4 |     TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEES         |      1 |      6 |      6 |00:00:00.01 |       2 |       |       |          |
|*  5 |      INDEX RANGE SCAN                  | EMP_DEPARTMENT_IX |      1 |      6 |      6 |00:00:00.01 |       1 |       |       |          |
|   6 |     VIEW                               | index$_join$_002  |      1 |    107 |    107 |00:00:00.01 |       8 |       |       |          |
|*  7 |      HASH JOIN                         |                   |      1 |        |    107 |00:00:00.01 |       8 |  1245K|  1245K| 1439K (0)|
|   8 |       INDEX FAST FULL SCAN             | EMP_NAME_IX       |      1 |    107 |    107 |00:00:00.01 |       4 |       |       |          |
|   9 |       INDEX FAST FULL SCAN             | EMP_EMP_ID_PK     |      1 |    107 |    107 |00:00:00.01 |       4 |       |       |          |
|  10 |    TABLE ACCESS BY INDEX ROWID BATCHED | EMPLOYEES         |      6 |      6 |      5 |00:00:00.01 |       4 |       |       |          |
|* 11 |     INDEX RANGE SCAN                   | EMP_MANAGER_IX    |      6 |      6 |      5 |00:00:00.01 |       3 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - access("M"."EMPLOYEE_ID"="E"."MANAGER_ID")
   5 - access("E"."DEPARTMENT_ID"=:B1)
   7 - access(ROWID=ROWID)
  11 - access("S"."MANAGER_ID"="E"."EMPLOYEE_ID")
```

This is a relatively simple plan for the single SQL statement, with 14 buffers read.

For the anti-pattern version there are thee SQL select statements, but we'll ignore the plan for initial bulk collect SQL and consider the other two execution plans. First, the client API SQL:

```
---------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name             | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                  |      1 |        |     10 |00:00:00.14 |    1195 |     41 |      1 |       |       |          |
|   1 |  SORT ORDER BY                      |                  |      1 |  48554 |     10 |00:00:00.14 |    1195 |     41 |      1 |  2048 |  2048 | 2048  (0)|
|*  2 |   HASH JOIN RIGHT OUTER             |                  |      1 |  48554 |     10 |00:00:00.14 |    1195 |     41 |      1 |  1368K|  1368K| 1322K (0)|
|   3 |    VIEW                             | index$_join$_002 |      1 |    107 |    106 |00:00:00.02 |       8 |     12 |      0 |       |       |          |
|*  4 |     HASH JOIN                       |                  |      1 |        |    106 |00:00:00.02 |       8 |     12 |      0 |  1519K|  1519K| 1575K (0)|
|   5 |      INDEX FAST FULL SCAN           | EMP_MANAGER_IX   |      1 |    107 |    106 |00:00:00.01 |       4 |      6 |      0 |       |       |          |
|   6 |      INDEX FAST FULL SCAN           | EMP_NAME_IX      |      1 |    107 |    107 |00:00:00.01 |       4 |      6 |      0 |       |       |          |
|   7 |    COLLECTION ITERATOR PICKLER FETCH| EMP_INFO_LIST    |      1 |   8168 |      6 |00:00:00.13 |    1187 |     29 |      1 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("E"."MANAGER_ID"=SYS_OP_ATG(VALUE(KOKBF$),1,2,2))
   4 - access(ROWID=ROWID)
```

Note the extreme inaccuracy of the cardinality estimates at steps 1 and 2, which originate in the step 7 estimate of 8168, which is a database-level default for an array function call. This is exposing the general problem that joining to arrays prevents accurate cardinality estimates. A total of 1195 buffers were read.

Next, the plan for the inner API SQL:

```
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                       | Name                      | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                                |                           |      1 |        |      6 |00:00:00.01 |      94 |      1 |      1 |       |       |          |
|   1 |  TEMP TABLE TRANSFORMATION                      |                           |      1 |        |      6 |00:00:00.01 |      94 |      1 |      1 |       |       |          |
|   2 |   LOAD AS SELECT                                |                           |      1 |        |      0 |00:00:00.01 |       6 |      0 |      1 |  1036K|  1036K|          |
|   3 |    SORT GROUP BY NOSORT                         |                           |      1 |      7 |      7 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|   4 |     TABLE ACCESS BY INDEX ROWID                 | JOB_HISTORY               |      1 |     10 |     10 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|   5 |      INDEX FULL SCAN                            | JHIST_EMP_ID_ST_DATE_PK   |      1 |     10 |     10 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|*  6 |   HASH JOIN OUTER                               |                           |      1 |      6 |      6 |00:00:00.01 |      85 |      1 |      0 |   735K|   735K|  506K (0)|
|*  7 |    HASH JOIN OUTER                              |                           |      1 |      6 |      6 |00:00:00.01 |      78 |      1 |      0 |   736K|   736K|  891K (0)|
|*  8 |     HASH JOIN OUTER                             |                           |      1 |      6 |      6 |00:00:00.01 |      75 |      1 |      0 |   737K|   737K|  550K (0)|
|*  9 |      HASH JOIN OUTER                            |                           |      1 |      6 |      6 |00:00:00.01 |      68 |      1 |      0 |   739K|   739K|  883K (0)|
|* 10 |       HASH JOIN OUTER                           |                           |      1 |      6 |      6 |00:00:00.01 |      62 |      0 |      0 |   740K|   740K|  903K (0)|
|* 11 |        HASH JOIN OUTER                          |                           |      1 |      6 |      6 |00:00:00.01 |      55 |      0 |      0 |   746K|   746K|  541K (0)|
|* 12 |         HASH JOIN OUTER                         |                           |      1 |      6 |      6 |00:00:00.01 |      48 |      0 |      0 |   754K|   754K|  534K (0)|
|* 13 |          HASH JOIN OUTER                        |                           |      1 |      6 |      6 |00:00:00.01 |      41 |      0 |      0 |   766K|   766K|  418K (0)|
|* 14 |           HASH JOIN OUTER                       |                           |      1 |      6 |      6 |00:00:00.01 |      33 |      0 |      0 |   773K|   773K|  414K (0)|
|  15 |            NESTED LOOPS OUTER                   |                           |      1 |      6 |      6 |00:00:00.01 |      26 |      0 |      0 |       |       |          |
|* 16 |             HASH JOIN OUTER                     |                           |      1 |      6 |      6 |00:00:00.01 |      23 |      0 |      0 |   833K|   833K|  414K (0)|
|* 17 |              HASH JOIN OUTER                    |                           |      1 |      6 |      6 |00:00:00.01 |      16 |      0 |      0 |   876K|   876K|  415K (0)|
|* 18 |               HASH JOIN                         |                           |      1 |      6 |      6 |00:00:00.01 |       9 |      0 |      0 |   905K|   905K| 1259K (0)|
|  19 |                MERGE JOIN                       |                           |      1 |    107 |    107 |00:00:00.01 |       9 |      0 |      0 |       |       |          |
|  20 |                 TABLE ACCESS BY INDEX ROWID     | JOBS                      |      1 |     19 |     19 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|  21 |                  INDEX FULL SCAN                | JOB_ID_PK                 |      1 |     19 |     19 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|* 22 |                 SORT JOIN                       |                           |     19 |    107 |    107 |00:00:00.01 |       7 |      0 |      0 | 18432 | 18432 |16384  (0)|
|  23 |                  TABLE ACCESS FULL              | EMPLOYEES                 |      1 |    107 |    107 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  24 |                COLLECTION ITERATOR PICKLER FETCH|                           |      1 |      6 |      6 |00:00:00.01 |       0 |      0 |      0 |       |       |          |
|  25 |               TABLE ACCESS FULL                 | DEPARTMENTS               |      1 |     27 |     27 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  26 |              TABLE ACCESS FULL                  | LOCATIONS                 |      1 |     23 |     23 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|* 27 |             INDEX UNIQUE SCAN                   | COUNTRY_C_ID_PK           |      6 |      1 |      6 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
|  28 |            TABLE ACCESS FULL                    | REGIONS                   |      1 |      4 |      4 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  29 |           VIEW                                  | index$_join$_024          |      1 |    107 |    107 |00:00:00.01 |       8 |      0 |      0 |       |       |          |
|* 30 |            HASH JOIN                            |                           |      1 |        |    107 |00:00:00.01 |       8 |      0 |      0 |  1245K|  1245K| 1550K (0)|
|  31 |             INDEX FAST FULL SCAN                | EMP_NAME_IX               |      1 |    107 |    107 |00:00:00.01 |       4 |      0 |      0 |       |       |          |
|  32 |             INDEX FAST FULL SCAN                | EMP_EMP_ID_PK             |      1 |    107 |    107 |00:00:00.01 |       4 |      0 |      0 |       |       |          |
|  33 |          TABLE ACCESS FULL                      | EMPLOYEES                 |      1 |    107 |    107 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  34 |         TABLE ACCESS FULL                       | JOBS                      |      1 |     19 |     19 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  35 |        VIEW                                     |                           |      1 |     18 |     19 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  36 |         HASH GROUP BY                           |                           |      1 |     18 |     19 |00:00:00.01 |       7 |      0 |      0 |  1558K|  1558K| 1185K (0)|
|  37 |          TABLE ACCESS FULL                      | EMPLOYEES                 |      1 |    107 |    107 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  38 |       VIEW                                      |                           |      1 |      7 |      7 |00:00:00.01 |       6 |      1 |      0 |       |       |          |
|  39 |        TABLE ACCESS FULL                        | SYS_TEMP_0FD9D6604_5813E5 |      1 |      7 |      7 |00:00:00.01 |       6 |      1 |      0 |       |       |          |
|  40 |      TABLE ACCESS FULL                          | JOBS                      |      1 |     19 |     19 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  41 |     VIEW                                        |                           |      1 |      7 |      7 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
|  42 |      TABLE ACCESS FULL                          | SYS_TEMP_0FD9D6604_5813E5 |      1 |      7 |      7 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
|  43 |    TABLE ACCESS FULL                            | JOBS                      |      1 |     19 |     19 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - access("JOB_P"."JOB_ID"="JHS"."JOB_ID")
   7 - access("JHS"."EMPLOYEE_ID"="EMP"."EMPLOYEE_ID")
   8 - access("JOB_MP"."JOB_ID"="JHS_M"."JOB_ID")
   9 - access("JHS_M"."EMPLOYEE_ID"="EMP"."EMPLOYEE_ID")
  10 - access("SUB"."MANAGER_ID"="EMP"."EMPLOYEE_ID")
  11 - access("JOB_M"."JOB_ID"="EMP_M"."JOB_ID")
  12 - access("EMP_M"."EMPLOYEE_ID"="EMP"."MANAGER_ID")
  13 - access("EMP_DM"."EMPLOYEE_ID"="DEP"."MANAGER_ID")
  14 - access("REG"."REGION_ID"="COU"."REGION_ID")
  16 - access("LOC"."LOCATION_ID"="DEP"."LOCATION_ID")
  17 - access("DEP"."DEPARTMENT_ID"="EMP"."DEPARTMENT_ID")
  18 - access("EMP"."EMPLOYEE_ID"=VALUE(KOKBF$))
  22 - access("JOB"."JOB_ID"="EMP"."JOB_ID")
       filter("JOB"."JOB_ID"="EMP"."JOB_ID")
  27 - access("COU"."COUNTRY_ID"="LOC"."COUNTRY_ID")
  30 - access(ROWID=ROWID)
```

This is pretty complex. We can't pretend to do a real performance analysis on such a small database (107 employees), but the potential for performance problems in real cases is clear. It's important to understand that any performance analysis on the client API has to take into account not just the client code, but the full complexity of the centralised SQL, so any apparent simplification from using it is to a great extent illusory.

### Database Inter-Schema Data Access Layer Anti-pattern

There is a variant of the anti-pattern above in which data access layers are used to retrieve data across schema boundaries. This variant suffers from exactly the same problems as the first of course, and should equally be avoided.

### Views as SQL Building Blocks Anti-Pattern

The same ideas as are behind the _APIs as SQL Building Blocks Anti-Pattern_ can also be implemented through views, and in fact this variant form of the anti-pattern has been around longer I think. We can illustrate it on the same example, by creating a view instead of the central API cursor.

```sql
CREATE OR REPLACE VIEW emp_ks_v (
       employee_id,
       name,
       job_title,
       job_title_p,
       name_mgr,
       job_title_mgr,
       job_title_mgr_p,
       n_sub,
       department_id,
       department_name,
       name_d_mgr,
       street_address,
       country_name,
       region_name) AS
WITH jhs_f AS (
SELECT employee_id,
     Max (job_id) KEEP (DENSE_RANK LAST ORDER BY end_date) job_id
  FROM hr.job_history
 GROUP BY employee_id
), sub AS (
SELECT manager_id,
       Count(*)                  n_sub
  FROM hr.employees
 GROUP BY manager_id
)
SELECT emp.employee_id,
       emp.last_name || ', ' || emp.first_name,
       job.job_title,
       job_p.job_title,
       emp_m.last_name || ', ' || emp_m.first_name,
       job_m.job_title,
       job_mp.job_title,
       sub.n_sub,
       dep.department_id,
       dep.department_name,
       emp_dm.last_name || ', ' || emp_dm.first_name,
       loc.street_address,
       cou.country_name,
       reg.region_name
  FROM hr.employees              emp
  JOIN hr.jobs                   job
    ON job.job_id                = emp.job_id
  LEFT JOIN jhs_f                jhs
    ON jhs.employee_id           = emp.employee_id
  LEFT JOIN hr.jobs              job_p
    ON job_p.job_id              = jhs.job_id
  LEFT JOIN hr.employees         emp_m
    ON emp_m.employee_id         = emp.manager_id
  LEFT JOIN hr.jobs              job_m
    ON job_m.job_id              = emp_m.job_id
  LEFT JOIN jhs_f                jhs_m
    ON jhs_m.employee_id         = emp.employee_id
  LEFT JOIN hr.jobs              job_mp
    ON job_mp.job_id             = jhs_m.job_id
  LEFT JOIN sub
    ON sub.manager_id            = emp.employee_id
  LEFT JOIN hr.departments       dep
    ON dep.department_id         = emp.department_id
  LEFT JOIN hr.employees         emp_dm
    ON emp_dm.employee_id        = dep.manager_id
  LEFT JOIN hr.locations         loc
    ON loc.location_id           = dep.location_id
  LEFT JOIN hr.countries         cou
    ON cou.country_id            = loc.country_id
  LEFT JOIN hr.regions           reg
    ON reg.region_id             = cou.region_id
```

The view can then be called to get the employee details for example for a given department, thus:

```sql
SELECT t.name,
       t.name_mgr,
       CASE WHEN e.last_name IS NOT NULL THEN e.last_name || ', ' || e.first_name END name_sub
  FROM emp_ks_v t
  LEFT JOIN hr.employees e
    ON e.manager_id = t.employee_id
 WHERE t.department_id = 30
 ORDER BY 1, 2, 3

```

This is actually quite a lot better than the API-based approach as it's much simpler, avoiding the need for object arrays, and allowing use simply by joining. Let's look at the execution plan though (we'll just run the query rather than put it into a client API returning a reference cursor):

```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name                      | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |                           |      1 |        |     10 |00:00:00.01 |      42 |      1 |      1 |       |       |          |
|   1 |  SORT ORDER BY                               |                           |      1 |     35 |     10 |00:00:00.01 |      42 |      1 |      1 |  2048 |  2048 | 2048  (0)|
|   2 |   NESTED LOOPS OUTER                         |                           |      1 |     35 |     10 |00:00:00.01 |      42 |      1 |      1 |       |       |          |
|   3 |    VIEW                                      | EMP_KS_V                  |      1 |      6 |      6 |00:00:00.01 |      38 |      1 |      1 |       |       |          |
|   4 |     TEMP TABLE TRANSFORMATION                |                           |      1 |        |      6 |00:00:00.01 |      38 |      1 |      1 |       |       |          |
|   5 |      LOAD AS SELECT                          |                           |      1 |        |      0 |00:00:00.01 |       6 |      0 |      1 |  1036K|  1036K|          |
|   6 |       SORT GROUP BY NOSORT                   |                           |      1 |      7 |      7 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|   7 |        TABLE ACCESS BY INDEX ROWID           | JOB_HISTORY               |      1 |     10 |     10 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|   8 |         INDEX FULL SCAN                      | JHIST_EMP_ID_ST_DATE_PK   |      1 |     10 |     10 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|*  9 |      HASH JOIN OUTER                         |                           |      1 |      6 |      6 |00:00:00.01 |      29 |      1 |      0 |   883K|   883K|  517K (0)|
|* 10 |       HASH JOIN OUTER                        |                           |      1 |      6 |      6 |00:00:00.01 |      21 |      1 |      0 |   890K|   890K|  857K (0)|
|* 11 |        HASH JOIN OUTER                       |                           |      1 |      6 |      6 |00:00:00.01 |      14 |      1 |      0 |   895K|   895K|  886K (0)|
|* 12 |         HASH JOIN OUTER                      |                           |      1 |      6 |      6 |00:00:00.01 |      11 |      1 |      0 |   905K|   905K|  893K (0)|
|  13 |          NESTED LOOPS                        |                           |      1 |      6 |      6 |00:00:00.01 |       5 |      0 |      0 |       |       |          |
|  14 |           NESTED LOOPS OUTER                 |                           |      1 |      1 |      1 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
|  15 |            TABLE ACCESS BY INDEX ROWID       | DEPARTMENTS               |      1 |      1 |      1 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|* 16 |             INDEX UNIQUE SCAN                | DEPT_ID_PK                |      1 |      1 |      1 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|* 17 |            INDEX UNIQUE SCAN                 | LOC_ID_PK                 |      1 |      1 |      1 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  18 |           TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEES                 |      1 |      6 |      6 |00:00:00.01 |       2 |      0 |      0 |       |       |          |
|* 19 |            INDEX RANGE SCAN                  | EMP_DEPARTMENT_IX         |      1 |      6 |      6 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  20 |          VIEW                                |                           |      1 |      7 |      7 |00:00:00.01 |       6 |      1 |      0 |       |       |          |
|  21 |           TABLE ACCESS FULL                  | SYS_TEMP_0FD9D6606_5813E5 |      1 |      7 |      7 |00:00:00.01 |       6 |      1 |      0 |       |       |          |
|  22 |         VIEW                                 |                           |      1 |      7 |      7 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
|  23 |          TABLE ACCESS FULL                   | SYS_TEMP_0FD9D6606_5813E5 |      1 |      7 |      7 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
|  24 |        VIEW                                  |                           |      1 |     18 |     19 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  25 |         HASH GROUP BY                        |                           |      1 |     18 |     19 |00:00:00.01 |       7 |      0 |      0 |  1558K|  1558K| 1211K (0)|
|  26 |          TABLE ACCESS FULL                   | EMPLOYEES                 |      1 |    107 |    107 |00:00:00.01 |       7 |      0 |      0 |       |       |          |
|  27 |       VIEW                                   | index$_join$_013          |      1 |    107 |    107 |00:00:00.01 |       8 |      0 |      0 |       |       |          |
|* 28 |        HASH JOIN                             |                           |      1 |        |    107 |00:00:00.01 |       8 |      0 |      0 |  1245K|  1245K| 1410K (0)|
|  29 |         INDEX FAST FULL SCAN                 | EMP_NAME_IX               |      1 |    107 |    107 |00:00:00.01 |       4 |      0 |      0 |       |       |          |
|  30 |         INDEX FAST FULL SCAN                 | EMP_EMP_ID_PK             |      1 |    107 |    107 |00:00:00.01 |       4 |      0 |      0 |       |       |          |
|  31 |    TABLE ACCESS BY INDEX ROWID BATCHED       | EMPLOYEES                 |      6 |      6 |      5 |00:00:00.01 |       4 |      0 |      0 |       |       |          |
|* 32 |     INDEX RANGE SCAN                         | EMP_MANAGER_IX            |      6 |      6 |      5 |00:00:00.01 |       3 |      0 |      0 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   9 - access("EMP_M"."EMPLOYEE_ID"="EMP"."MANAGER_ID")
  10 - access("SUB"."MANAGER_ID"="EMP"."EMPLOYEE_ID")
  11 - access("JHS_M"."EMPLOYEE_ID"="EMP"."EMPLOYEE_ID")
  12 - access("JHS"."EMPLOYEE_ID"="EMP"."EMPLOYEE_ID")
  16 - access("DEP"."DEPARTMENT_ID"=30)
  17 - access("LOC"."LOCATION_ID"="DEP"."LOCATION_ID")
  19 - access("EMP"."DEPARTMENT_ID"=30)
  28 - access(ROWID=ROWID)
  32 - access("E"."MANAGER_ID"="T"."EMPLOYEE_ID")
```

There is only one SQL statement, and the plan is better too, with better cardinality estimates owing to the absence of array processing, and with 42 buffers read. It's still a bad idea though because the client caller would be executing far more complex SQL than is required and as before performance analysis on the client requires the full complexity of the 'centralised' SQL to be included. Using complex views as SQL building blocks is generally considered to have poor performance characteristics.

## SQL Modularity: Other Design Options

### Splitting Up Long SQL Statements

Oracle's Cost Based Optimiser (CBO) has been greatly enhanced since it's introduction in v7, but remains imperfect. The combinatorial nature of the problem that it tries to solve suggests that there will always be larger queries where it makes a bad choice of plan. In some cases splitting a large query into smaller ones and using temporary tables to join them can give better performance. This may arise from new indexing options for the CBO, or by dynamic sampling capabilities on the temporary tables, or just from the CBO algorithms happening to work better on the divided queries.

It's important to understand though that any such splitting should be done purely on performance grounds: the splitting increases the code complexity and breaking a sequence of declarative joins into several subsequences is not comparable with standard modularisation of programs into subprograms.

Long SQL statements are not necessarily problematic, but obviously should only be as long as necessary, and avoiding the design anti-patterns mentioned helps to ensure this.

### Simple Views

Simple views, without joins, are often used in areas such as access control; for example Oracle Applications multi-org features (upto release 11) generally involve transactional tables being referenced by such simple views. These do not cause the performance problems seen with the complex building-block views anti-pattern.

### Data Access Layers for Back-end Programs

It is quite possible to use a similar approach for database programs as for front-end programs in terms of a data access layer. One could adopt a standard that stand-alone database PL/SQL programs should access data through packaged APIs rather than directly. This obviously does not have the performance or language advantages of the design pattern in relation to Java front-ends for example, but it may in some circumstances be a preferred method of code organisation. It is therefore not an anti-pattern of the kind we have considered - as long as it is for stand-alone programs only.

## Conclusion

The main aim of this article has been to distinguish between good approaches to modularity in SQL _(patterns)_ and bad ones _(anti-patterns)_ based on personal experience of seeing both types applied.

- The Data Access Layer design pattern is an excellent approach for client applications developed in Java, .net etc. to access a database
- Using Data Access Layers for internal access within a database is a classic anti-pattern leading to overcomplication and performance problems
- A good design pattern used in an inappropriate context can become an anti-pattern

[SQL Modularity Code, in GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/modularity-in-sql-patterns-anti-patterns-and-the-kitchen-sink)
