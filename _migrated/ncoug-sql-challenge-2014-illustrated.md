---
layout: post
title: "NoCOUG SQL Challenge 2014 Illustrated"
date: 2014-08-25
migrated: true
categories: 
  - "oracle"
  - "performance"
  - "qsd"
  - "sql"
  - "subquery-factor"
tags: 
  - "ncoug"
  - "oracle"
  - "qsd"
  - "sql"
  - "subquery"
  - "tuning"
---

An SQL challenge was posted recently on the blog of the Northern California Oracle user group, [SQL Mini-challenge](http://nocoug.wordpress.com/2014/08/12/sql-mini-challenge/). A query was given against Oracle's demo HR schema, with description:

> It lists the locations containing a department that either contains an employee named Steven King or an employee who holds the title of President or an employee who has previously held the title of President.

The challenge was to rewrite the query avoiding the relatively expensive existence subqueries, and minimising the number of consistent gets reported on the small demo data set. Oracle's Cost-Based Optimiser will itself transform queries within its parsing phase, but at a relatively low level; for example, an 'OR' condition might be changed to a union if the CBO thinks that will aid performance. The solutions to the challenge present a nice illustration of how more extensive query transfomations can improve performance and modularity characteristics.

In this article I will list four equivalent queries for the problem, three based on solutions provided on the blog, using Ansi syntax here for consistency. For each query I give the output from DBMS\_XPlan, and include a query structure diagram following my own diagramming notation. The example query provided in the challenge is not in fact the most literal translation of the requirement into SQL, and I think it will be interesting to start with my idea of what that would be.

**Update 28 October 2014:** I noticed that in the 'literal' query I had omitted the location condition on the third subquery. I have fixed this and the execution plan is much worse. The results reported here were from version 11.2; running on v12.1 gives some extremely interesting differences, and I have added the v12.1 results at the end for the 'literal' query. They show a much improved plan, with departments 'factorised' out.

## Query 1: Literal

This is my attempt at the most literal translation of the stated requirement into SQL. The three conditions are all separately expressed as existence subqueries.

### QSD Literal

<img src="/migrated_images/2014/08/NCOUG-Literal.jpg" alt="NCOUG-Literal" title="NCOUG-Literal" />

### Query Literal

Note that in an earlier version of this article, I had omitted the final 'd.location\_id = l.location\_id' condition.

```sql
SELECT l.location_id, l.city
  FROM locations l
 WHERE EXISTS
  (SELECT *
     FROM departments d
     JOIN employees e
       ON e.department_id = d.department_id
    WHERE d.location_id = l.location_id
      AND e.first_name = 'Steven' AND e.last_name = 'King'
  ) OR EXISTS
  (SELECT *
     FROM departments d
     JOIN employees e
       ON e.department_id = d.department_id
     JOIN jobs j
       ON j.job_id = e.job_id
    WHERE d.location_id = l.location_id
      AND j.job_title = 'President'
  ) OR EXISTS
  (SELECT *
     FROM departments d
     JOIN employees e
       ON e.department_id = d.department_id
     JOIN job_history h
       ON h.employee_id = e.employee_id
     JOIN jobs j2
       ON j2.job_id = h.job_id
    WHERE d.location_id = l.location_id
      AND j2.job_title   = 'President'
  )

```

### XPlan Literal

```
-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name              | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                   |      1 |        |      1 |00:00:00.01 |     426 |       |       |          |
|*  1 |  FILTER                          |                   |      1 |        |      1 |00:00:00.01 |     426 |       |       |          |
|   2 |   VIEW                           | index$_join$_001  |      1 |     23 |     23 |00:00:00.01 |       7 |       |       |          |
|*  3 |    HASH JOIN                     |                   |      1 |        |     23 |00:00:00.01 |       7 |  1023K|  1023K| 1150K (0)|
|   4 |     INDEX FAST FULL SCAN         | LOC_CITY_IX       |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
|   5 |     INDEX FAST FULL SCAN         | LOC_ID_PK         |      1 |     23 |     23 |00:00:00.01 |       4 |       |       |          |
|   6 |   NESTED LOOPS                   |                   |     23 |        |      1 |00:00:00.01 |      92 |       |       |          |
|   7 |    NESTED LOOPS                  |                   |     23 |      1 |     23 |00:00:00.01 |      69 |       |       |          |
|   8 |     TABLE ACCESS BY INDEX ROWID  | EMPLOYEES         |     23 |      1 |     23 |00:00:00.01 |      46 |       |       |          |
|*  9 |      INDEX RANGE SCAN            | EMP_NAME_IX       |     23 |      1 |     23 |00:00:00.01 |      23 |       |       |          |
|* 10 |     INDEX UNIQUE SCAN            | DEPT_ID_PK        |     23 |      1 |     23 |00:00:00.01 |      23 |       |       |          |
|* 11 |    TABLE ACCESS BY INDEX ROWID   | DEPARTMENTS       |     23 |      1 |      1 |00:00:00.01 |      23 |       |       |          |
|  12 |   NESTED LOOPS                   |                   |     22 |        |      0 |00:00:00.01 |     173 |       |       |          |
|  13 |    NESTED LOOPS                  |                   |     22 |      1 |     88 |00:00:00.01 |     166 |       |       |          |
|  14 |     NESTED LOOPS                 |                   |     22 |      2 |      6 |00:00:00.01 |     160 |       |       |          |
|* 15 |      TABLE ACCESS FULL           | JOBS              |     22 |      1 |     22 |00:00:00.01 |     132 |       |       |          |
|  16 |      TABLE ACCESS BY INDEX ROWID | DEPARTMENTS       |     22 |      2 |      6 |00:00:00.01 |      28 |       |       |          |
|* 17 |       INDEX RANGE SCAN           | DEPT_LOCATION_IX  |     22 |      2 |      6 |00:00:00.01 |      22 |       |       |          |
|* 18 |     INDEX RANGE SCAN             | EMP_DEPARTMENT_IX |      6 |     10 |     88 |00:00:00.01 |       6 |       |       |          |
|* 19 |    TABLE ACCESS BY INDEX ROWID   | EMPLOYEES         |     88 |      1 |      0 |00:00:00.01 |       7 |       |       |          |
|  20 |   NESTED LOOPS                   |                   |     22 |        |      0 |00:00:00.01 |     154 |       |       |          |
|  21 |    NESTED LOOPS                  |                   |     22 |      1 |      0 |00:00:00.01 |     154 |       |       |          |
|  22 |     NESTED LOOPS                 |                   |     22 |      1 |      0 |00:00:00.01 |     154 |       |       |          |
|  23 |      NESTED LOOPS                |                   |     22 |      1 |      0 |00:00:00.01 |     154 |       |       |          |
|* 24 |       TABLE ACCESS FULL          | JOBS              |     22 |      1 |     22 |00:00:00.01 |     132 |       |       |          |
|  25 |       TABLE ACCESS BY INDEX ROWID| JOB_HISTORY       |     22 |      1 |      0 |00:00:00.01 |      22 |       |       |          |
|* 26 |        INDEX RANGE SCAN          | JHIST_JOB_IX      |     22 |      1 |      0 |00:00:00.01 |      22 |       |       |          |
|  27 |      TABLE ACCESS BY INDEX ROWID | EMPLOYEES         |      0 |      1 |      0 |00:00:00.01 |       0 |       |       |          |
|* 28 |       INDEX UNIQUE SCAN          | EMP_EMP_ID_PK     |      0 |      1 |      0 |00:00:00.01 |       0 |       |       |          |
|* 29 |     INDEX UNIQUE SCAN            | DEPT_ID_PK        |      0 |      1 |      0 |00:00:00.01 |       0 |       |       |          |
|* 30 |    TABLE ACCESS BY INDEX ROWID   | DEPARTMENTS       |      0 |      1 |      0 |00:00:00.01 |       0 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(( IS NOT NULL OR  IS NOT NULL OR  IS NOT NULL))
   3 - access(ROWID=ROWID)
   9 - access("E"."LAST_NAME"='King' AND "E"."FIRST_NAME"='Steven')
  10 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  11 - filter("D"."LOCATION_ID"=:B1)
  15 - filter("J"."JOB_TITLE"='President')
  17 - access("D"."LOCATION_ID"=:B1)
  18 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  19 - filter("J"."JOB_ID"="E"."JOB_ID")
  24 - filter("J2"."JOB_TITLE"='President')
  26 - access("J2"."JOB_ID"="H"."JOB_ID")
  28 - access("H"."EMPLOYEE_ID"="E"."EMPLOYEE_ID")
  29 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  30 - filter("D"."LOCATION_ID"=:B1)

```

## Query 2: NoCOUG Example

This is the example in the original challenge article, translated into Ansi syntax. It nests the job history existence subquery within an outer existence subquery, and references the departments and employees tables only once.

### QSD NoCOUG Example

<img src="/migrated_images/2014/08/NCOUG-Example.jpg" alt="NCOUG-Example" title="NCOUG-Example" />

### Query NoCOUG Example

```
SELECT l.location_id, l.city
  FROM locations l
 WHERE EXISTS
  (SELECT *
     FROM departments d
     JOIN employees e
       ON e.department_id = d.department_id
     JOIN jobs j
       ON j.job_id = e.job_id
    WHERE d.location_id = l.location_id
      AND ( 
            (e.first_name = 'Steven' AND e.last_name = 'King')
         OR j.job_title = 'President'
         OR EXISTS
            (SELECT *
               FROM job_history h
               JOIN jobs j2
                 ON j2.job_id = h.job_id
              WHERE h.employee_id = e.employee_id
                AND j2.job_title   = 'President'
            ) 
          )
  )

```

### XPlan NoCOUG Example

```
-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name              | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                   |      1 |        |      1 |00:00:00.01 |     152 |       |       |          |
|*  1 |  HASH JOIN SEMI                  |                   |      1 |      7 |      1 |00:00:00.01 |     152 |  1156K|  1156K| 1120K (0)|
|   2 |   VIEW                           | index$_join$_001  |      1 |     23 |     23 |00:00:00.01 |       6 |       |       |          |
|*  3 |    HASH JOIN                     |                   |      1 |        |     23 |00:00:00.01 |       6 |  1023K|  1023K| 1443K (0)|
|   4 |     INDEX FAST FULL SCAN         | LOC_CITY_IX       |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
|   5 |     INDEX FAST FULL SCAN         | LOC_ID_PK         |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
|   6 |   VIEW                           | VW_SQ_1           |      1 |     11 |      1 |00:00:00.01 |     146 |       |       |          |
|*  7 |    FILTER                        |                   |      1 |        |      1 |00:00:00.01 |     146 |       |       |          |
|*  8 |     HASH JOIN                    |                   |      1 |    106 |    106 |00:00:00.01 |      15 |   876K|   876K|  895K (0)|
|   9 |      MERGE JOIN                  |                   |      1 |    107 |    107 |00:00:00.01 |       8 |       |       |          |
|  10 |       TABLE ACCESS BY INDEX ROWID| JOBS              |      1 |     19 |     19 |00:00:00.01 |       2 |       |       |          |
|  11 |        INDEX FULL SCAN           | JOB_ID_PK         |      1 |     19 |     19 |00:00:00.01 |       1 |       |       |          |
|* 12 |       SORT JOIN                  |                   |     19 |    107 |    107 |00:00:00.01 |       6 | 15360 | 15360 |14336  (0)|
|  13 |        TABLE ACCESS FULL         | EMPLOYEES         |      1 |    107 |    107 |00:00:00.01 |       6 |       |       |          |
|  14 |      TABLE ACCESS FULL           | DEPARTMENTS       |      1 |     27 |     27 |00:00:00.01 |       7 |       |       |          |
|  15 |     NESTED LOOPS                 |                   |    105 |        |      0 |00:00:00.01 |     131 |       |       |          |
|  16 |      NESTED LOOPS                |                   |    105 |      1 |     10 |00:00:00.01 |     121 |       |       |          |
|  17 |       TABLE ACCESS BY INDEX ROWID| JOB_HISTORY       |    105 |      1 |     10 |00:00:00.01 |     112 |       |       |          |
|* 18 |        INDEX RANGE SCAN          | JHIST_EMPLOYEE_IX |    105 |      1 |     10 |00:00:00.01 |     105 |       |       |          |
|* 19 |       INDEX UNIQUE SCAN          | JOB_ID_PK         |     10 |      1 |     10 |00:00:00.01 |       9 |       |       |          |
|* 20 |      TABLE ACCESS BY INDEX ROWID | JOBS              |     10 |      1 |      0 |00:00:00.01 |      10 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("ITEM_1"="L"."LOCATION_ID")
   3 - access(ROWID=ROWID)
   7 - filter((("E"."FIRST_NAME"='Steven' AND "E"."LAST_NAME"='King') OR "J"."JOB_TITLE"='President' OR  IS NOT NULL))
   8 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  12 - access("J"."JOB_ID"="E"."JOB_ID")
       filter("J"."JOB_ID"="E"."JOB_ID")
  18 - access("H"."EMPLOYEE_ID"=:B1)
  19 - access("J2"."JOB_ID"="H"."JOB_ID")
  20 - filter("J2"."JOB_TITLE"='President')

```

## Query 3: Subquery Factor Union

This converts the 'OR' conditions into a union of three driving subqueries that return the matching department ids from a subquery factor (which could equally be an inline view), and then joins departments and locations.

### QSD Subquery Factor Union

<img src="/migrated_images/2014/08/NCOUG-SQF.jpg" alt="NCOUG-SQF" title="NCOUG-SQF" />

### Query Subquery Factor Union

```
WITH driving_union AS (
SELECT e.department_id
  FROM employees e
 WHERE (e.first_name = 'Steven' AND e.last_name = 'King')
 UNION
SELECT e.department_id
  FROM jobs j
  JOIN employees e
    ON e.job_id        = j.job_id
 WHERE j.job_title     = 'President'
 UNION
SELECT e.department_id
  FROM jobs j
  JOIN job_history h
    ON j.job_id        = h.job_id
  JOIN employees e
    ON e.employee_id    = h.employee_id
 WHERE j.job_title     = 'President'
)
SELECT DISTINCT l.location_id, l.city
  FROM driving_union u
  JOIN departments d
    ON d.department_id 	= u.department_id
  JOIN locations l
    ON l.location_id 	= d.location_id

```

### XPlan Subquery Factor Union

```
----------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name             | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                  |      1 |        |      1 |00:00:00.01 |      29 |       |       |          |
|   1 |  HASH UNIQUE                         |                  |      1 |      8 |      1 |00:00:00.01 |      29 |  1156K|  1156K|  464K (0)|
|*  2 |   HASH JOIN                          |                  |      1 |      8 |      1 |00:00:00.01 |      29 |  1517K|  1517K|  366K (0)|
|*  3 |    HASH JOIN                         |                  |      1 |      8 |      1 |00:00:00.01 |      23 |  1517K|  1517K|  382K (0)|
|   4 |     VIEW                             |                  |      1 |      8 |      1 |00:00:00.01 |      17 |       |       |          |
|   5 |      SORT UNIQUE                     |                  |      1 |      8 |      1 |00:00:00.01 |      17 |  2048 |  2048 | 2048  (0)|
|   6 |       UNION-ALL                      |                  |      1 |        |      2 |00:00:00.01 |      17 |       |       |          |
|   7 |        TABLE ACCESS BY INDEX ROWID   | EMPLOYEES        |      1 |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|*  8 |         INDEX RANGE SCAN             | EMP_NAME_IX      |      1 |      1 |      1 |00:00:00.01 |       1 |       |       |          |
|   9 |        NESTED LOOPS                  |                  |      1 |        |      1 |00:00:00.01 |       8 |       |       |          |
|  10 |         NESTED LOOPS                 |                  |      1 |      6 |      1 |00:00:00.01 |       7 |       |       |          |
|* 11 |          TABLE ACCESS FULL           | JOBS             |      1 |      1 |      1 |00:00:00.01 |       6 |       |       |          |
|* 12 |          INDEX RANGE SCAN            | EMP_JOB_IX       |      1 |      6 |      1 |00:00:00.01 |       1 |       |       |          |
|  13 |         TABLE ACCESS BY INDEX ROWID  | EMPLOYEES        |      1 |      6 |      1 |00:00:00.01 |       1 |       |       |          |
|  14 |        NESTED LOOPS                  |                  |      1 |        |      0 |00:00:00.01 |       7 |       |       |          |
|  15 |         NESTED LOOPS                 |                  |      1 |      1 |      0 |00:00:00.01 |       7 |       |       |          |
|  16 |          NESTED LOOPS                |                  |      1 |      1 |      0 |00:00:00.01 |       7 |       |       |          |
|* 17 |           TABLE ACCESS FULL          | JOBS             |      1 |      1 |      1 |00:00:00.01 |       6 |       |       |          |
|  18 |           TABLE ACCESS BY INDEX ROWID| JOB_HISTORY      |      1 |      1 |      0 |00:00:00.01 |       1 |       |       |          |
|* 19 |            INDEX RANGE SCAN          | JHIST_JOB_IX     |      1 |      1 |      0 |00:00:00.01 |       1 |       |       |          |
|* 20 |          INDEX UNIQUE SCAN           | EMP_EMP_ID_PK    |      0 |      1 |      0 |00:00:00.01 |       0 |       |       |          |
|  21 |         TABLE ACCESS BY INDEX ROWID  | EMPLOYEES        |      0 |      1 |      0 |00:00:00.01 |       0 |       |       |          |
|  22 |     VIEW                             | index$_join$_011 |      1 |     27 |     27 |00:00:00.01 |       6 |       |       |          |
|* 23 |      HASH JOIN                       |                  |      1 |        |     27 |00:00:00.01 |       6 |  1096K|  1096K| 1547K (0)|
|  24 |       INDEX FAST FULL SCAN           | DEPT_ID_PK       |      1 |     27 |     27 |00:00:00.01 |       3 |       |       |          |
|  25 |       INDEX FAST FULL SCAN           | DEPT_LOCATION_IX |      1 |     27 |     27 |00:00:00.01 |       3 |       |       |          |
|  26 |    VIEW                              | index$_join$_013 |      1 |     23 |     23 |00:00:00.01 |       6 |       |       |          |
|* 27 |     HASH JOIN                        |                  |      1 |        |     23 |00:00:00.01 |       6 |  1023K|  1023K| 1426K (0)|
|  28 |      INDEX FAST FULL SCAN            | LOC_CITY_IX      |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
|  29 |      INDEX FAST FULL SCAN            | LOC_ID_PK        |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("L"."LOCATION_ID"="D"."LOCATION_ID")
   3 - access("D"."DEPARTMENT_ID"="U"."DEPARTMENT_ID")
   8 - access("E"."LAST_NAME"='King' AND "E"."FIRST_NAME"='Steven')
  11 - filter("J"."JOB_TITLE"='President')
  12 - access("E"."JOB_ID"="J"."JOB_ID")
  17 - filter("J"."JOB_TITLE"='President')
  19 - access("J"."JOB_ID"="H"."JOB_ID")
  20 - access("E"."EMPLOYEE_ID"="H"."EMPLOYEE_ID")
  23 - access(ROWID=ROWID)
  27 - access(ROWID=ROWID)

```

## Query 4: Outer Joins

This avoids existence subqueries using the idea that an outer join with a constraint that the joined record is not null, with a distinct qualifier to eliminate duplicates, can serve as a logical equivalent.

### QSD Outer Joins

<img src="/migrated_images/2014/08/NCOUG-OJ.jpg" alt="NCOUG-OJ" title="NCOUG-OJ" />

### Query Outer Joins

```
SELECT DISTINCT l.location_id, l.city
  FROM employees e
  LEFT JOIN jobs j
    ON j.job_id        	= e.job_id
   AND j.job_title      = 'President'  
  LEFT JOIN job_history h
    ON h.employee_id    = e.employee_id
  LEFT JOIN jobs j2
    ON j2.job_id        = h.job_id
   AND j2.job_title     = 'President' 
  JOIN departments d
    ON d.department_id 	= e.department_id
  JOIN locations l
    ON l.location_id 	= d.location_id
 WHERE (e.first_name = 'Steven' AND e.last_name = 'King')
    OR j.job_id IS NOT NULL
    OR j2.job_id IS NOT NULL

```

### XPlan Outer Joins

```
----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name              | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |                   |      1 |        |      1 |00:00:00.01 |      36 |       |       |          |
|   1 |  HASH UNIQUE                  |                   |      1 |    106 |      1 |00:00:00.01 |      36 |  1156K|  1156K|  464K (0)|
|*  2 |   FILTER                      |                   |      1 |        |      1 |00:00:00.01 |      36 |       |       |          |
|*  3 |    HASH JOIN RIGHT OUTER      |                   |      1 |    106 |    109 |00:00:00.01 |      36 |  1269K|  1269K|  369K (0)|
|*  4 |     TABLE ACCESS FULL         | JOBS              |      1 |      1 |      1 |00:00:00.01 |       6 |       |       |          |
|*  5 |     HASH JOIN RIGHT OUTER     |                   |      1 |    106 |    109 |00:00:00.01 |      30 |  1134K|  1134K|  751K (0)|
|   6 |      VIEW                     | index$_join$_004  |      1 |     10 |     10 |00:00:00.01 |       6 |       |       |          |
|*  7 |       HASH JOIN               |                   |      1 |        |     10 |00:00:00.01 |       6 |  1096K|  1096K| 1331K (0)|
|   8 |        INDEX FAST FULL SCAN   | JHIST_EMPLOYEE_IX |      1 |     10 |     10 |00:00:00.01 |       3 |       |       |          |
|   9 |        INDEX FAST FULL SCAN   | JHIST_JOB_IX      |      1 |     10 |     10 |00:00:00.01 |       3 |       |       |          |
|* 10 |      HASH JOIN OUTER          |                   |      1 |    106 |    106 |00:00:00.01 |      24 |   858K|   858K| 1270K (0)|
|* 11 |       HASH JOIN               |                   |      1 |    106 |    106 |00:00:00.01 |      18 |  1063K|  1063K| 1252K (0)|
|* 12 |        HASH JOIN              |                   |      1 |     27 |     27 |00:00:00.01 |      12 |  1156K|  1156K| 1133K (0)|
|  13 |         VIEW                  | index$_join$_010  |      1 |     23 |     23 |00:00:00.01 |       6 |       |       |          |
|* 14 |          HASH JOIN            |                   |      1 |        |     23 |00:00:00.01 |       6 |  1023K|  1023K| 1450K (0)|
|  15 |           INDEX FAST FULL SCAN| LOC_CITY_IX       |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
|  16 |           INDEX FAST FULL SCAN| LOC_ID_PK         |      1 |     23 |     23 |00:00:00.01 |       3 |       |       |          |
|  17 |         VIEW                  | index$_join$_008  |      1 |     27 |     27 |00:00:00.01 |       6 |       |       |          |
|* 18 |          HASH JOIN            |                   |      1 |        |     27 |00:00:00.01 |       6 |  1096K|  1096K| 1547K (0)|
|  19 |           INDEX FAST FULL SCAN| DEPT_ID_PK        |      1 |     27 |     27 |00:00:00.01 |       3 |       |       |          |
|  20 |           INDEX FAST FULL SCAN| DEPT_LOCATION_IX  |      1 |     27 |     27 |00:00:00.01 |       3 |       |       |          |
|  21 |        TABLE ACCESS FULL      | EMPLOYEES         |      1 |    107 |    107 |00:00:00.01 |       6 |       |       |          |
|* 22 |       TABLE ACCESS FULL       | JOBS              |      1 |      1 |      1 |00:00:00.01 |       6 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter((("E"."FIRST_NAME"='Steven' AND "E"."LAST_NAME"='King') OR "J"."JOB_ID" IS NOT NULL OR "J2"."JOB_ID" IS NOT NULL))
   3 - access("J2"."JOB_ID"="H"."JOB_ID")
   4 - filter("J2"."JOB_TITLE"='President')
   5 - access("H"."EMPLOYEE_ID"="E"."EMPLOYEE_ID")
   7 - access(ROWID=ROWID)
  10 - access("J"."JOB_ID"="E"."JOB_ID")
  11 - access("D"."DEPARTMENT_ID"="E"."DEPARTMENT_ID")
  12 - access("L"."LOCATION_ID"="D"."LOCATION_ID")
  14 - access(ROWID=ROWID)
  18 - access(ROWID=ROWID)
  22 - filter("J"."JOB_TITLE"='President')

```

## Query Summary Table v11.2

Here is a summary of some statistics on the queries, run on an Oracle 11.2 XE instance. Query lines depends on formatting of course.

| **Query** | **Buffers** | **Table Instances** | **XPlan Steps** | **Query Lines** |
| --- | --- | --- | --- | --- |
| **Literal** | 426 | 10 | 30 | 30 |
| **NoCOUG Example** | 152 | 6 | 20 | 23 |
| **Subquery Factor Union** | 29 | 6 | 29 | 25 |
| **Outer Joins** | 36 | 6 | 22 | 17 |

## Oracle 12c

While the original version of this article, posted 25 August 2014, was based on Oracle 11.2, I later ran my script on Oracle 12.1, and noted that the 'literal' query now had a much-changed execution plan. In particular, the departments table had been 'factorised' out, appearing only once, and giving a much reduced buffer count of 31. It seems that this comes from an improvement in the transformation phase of query execution. The other queries did not show so much difference in plans, see the summary table below.

### Execution Plan for Literal v12.1

```
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name               | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |                    |      1 |        |      1 |00:00:00.05 |      31 |      1 |       |       |          |
|*  1 |  HASH JOIN SEMI                              |                    |      1 |      7 |      1 |00:00:00.05 |      31 |      1 |  1645K|  1645K| 1432K (0)|
|   2 |   VIEW                                       | index$_join$_001   |      1 |     23 |     23 |00:00:00.01 |       8 |      0 |       |       |          |
|*  3 |    HASH JOIN                                 |                    |      1 |        |     23 |00:00:00.01 |       8 |      0 |  1368K|  1368K| 1566K (0)|
|   4 |     INDEX FAST FULL SCAN                     | LOC_CITY_IX        |      1 |     23 |     23 |00:00:00.01 |       4 |      0 |       |       |          |
|   5 |     INDEX FAST FULL SCAN                     | LOC_ID_PK          |      1 |     23 |     23 |00:00:00.01 |       4 |      0 |       |       |          |
|   6 |   VIEW                                       | VW_SQ_1            |      1 |      8 |      1 |00:00:00.05 |      23 |      1 |       |       |          |
|   7 |    MERGE JOIN SEMI                           |                    |      1 |      8 |      1 |00:00:00.05 |      23 |      1 |       |       |          |
|   8 |     TABLE ACCESS BY INDEX ROWID              | DEPARTMENTS        |      1 |     27 |     10 |00:00:00.01 |       4 |      0 |       |       |          |
|   9 |      INDEX FULL SCAN                         | DEPT_ID_PK         |      1 |     27 |     10 |00:00:00.01 |       2 |      0 |       |       |          |
|* 10 |     SORT UNIQUE                              |                    |     10 |      8 |      1 |00:00:00.05 |      19 |      1 |  2048 |  2048 | 2048  (0)|
|  11 |      VIEW                                    | VW_JF_SET$1236063A |      1 |      8 |      2 |00:00:00.05 |      19 |      1 |       |       |          |
|  12 |       UNION-ALL                              |                    |      1 |        |      2 |00:00:00.05 |      19 |      1 |       |       |          |
|  13 |        NESTED LOOPS                          |                    |      1 |        |      1 |00:00:00.05 |       9 |      1 |       |       |          |
|  14 |         NESTED LOOPS                         |                    |      1 |      6 |      1 |00:00:00.05 |       8 |      1 |       |       |          |
|* 15 |          TABLE ACCESS FULL                   | JOBS               |      1 |      1 |      1 |00:00:00.01 |       7 |      0 |       |       |          |
|* 16 |          INDEX RANGE SCAN                    | EMP_JOB_IX         |      1 |      6 |      1 |00:00:00.05 |       1 |      1 |       |       |          |
|  17 |         TABLE ACCESS BY INDEX ROWID          | EMPLOYEES          |      1 |      6 |      1 |00:00:00.01 |       1 |      0 |       |       |          |
|  18 |        TABLE ACCESS BY INDEX ROWID BATCHED   | EMPLOYEES          |      1 |      1 |      1 |00:00:00.01 |       2 |      0 |       |       |          |
|* 19 |         INDEX RANGE SCAN                     | EMP_NAME_IX        |      1 |      1 |      1 |00:00:00.01 |       1 |      0 |       |       |          |
|  20 |        NESTED LOOPS                          |                    |      1 |        |      0 |00:00:00.01 |       8 |      0 |       |       |          |
|  21 |         NESTED LOOPS                         |                    |      1 |      1 |      0 |00:00:00.01 |       8 |      0 |       |       |          |
|  22 |          NESTED LOOPS                        |                    |      1 |      1 |      0 |00:00:00.01 |       8 |      0 |       |       |          |
|* 23 |           TABLE ACCESS FULL                  | JOBS               |      1 |      1 |      1 |00:00:00.01 |       7 |      0 |       |       |          |
|  24 |           TABLE ACCESS BY INDEX ROWID BATCHED| JOB_HISTORY        |      1 |      1 |      0 |00:00:00.01 |       1 |      0 |       |       |          |
|* 25 |            INDEX RANGE SCAN                  | JHIST_JOB_IX       |      1 |      1 |      0 |00:00:00.01 |       1 |      0 |       |       |          |
|* 26 |          INDEX UNIQUE SCAN                   | EMP_EMP_ID_PK      |      0 |      1 |      0 |00:00:00.01 |       0 |      0 |       |       |          |
|  27 |         TABLE ACCESS BY INDEX ROWID          | EMPLOYEES          |      0 |      1 |      0 |00:00:00.01 |       0 |      0 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("VW_COL_1"="L"."LOCATION_ID")
   3 - access(ROWID=ROWID)
  10 - access("ITEM_1"="D"."DEPARTMENT_ID")
       filter("ITEM_1"="D"."DEPARTMENT_ID")
  15 - filter("J"."JOB_TITLE"='President')
  16 - access("J"."JOB_ID"="E"."JOB_ID")
  19 - access("E"."LAST_NAME"='King' AND "E"."FIRST_NAME"='Steven')
  23 - filter("J2"."JOB_TITLE"='President')
  25 - access("J2"."JOB_ID"="H"."JOB_ID")
  26 - access("H"."EMPLOYEE_ID"="E"."EMPLOYEE_ID")

Note
-----
   - this is an adaptive plan

```

### Query Summary Table v12.1

| **Query** | **Buffers** | **Table Instances** | **XPlan Steps** | **Query Lines** |
| --- | --- | --- | --- | --- |
| **Literal** | 31 | 10 | 27 | 31 |
| **NoCOUG Example** | 156 | 6 | 19 | 23 |
| **Subquery Factor Union** | 29 | 6 | 28 | 25 |
| **Outer Joins** | 45 | 6 | 22 | 17 |

[Here](https://github.com/BrenPatF/wp_ghp_migration/tree/master/ncoug-sql-challenge-2014-illustrated) is the v12.1 output, in GitHub container.
