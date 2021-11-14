---
layout: post
title:  "SQL for Period Aggregate Reporting"
date:   2021-11-14 09:00:00 +0100
categories: sql macro aggregate reporting plsql
---

In financial reporting there is often a requirement to report on sales performance across multiple time periods. For example we might want to know total sales of products for a given month, then for the 3 month period, the year to date, and the 12 month period up to that month, etc.. We might also want to break down associated costs of selling across the same periods. There are several ways to approach implementing such reports in SQL, and in real world applications that may have many columns and periods to report on, different approaches will have different levels of complexity and performance characteristics. 

In this article I will describe three approaches using static SQL, starting with perhaps the most obvious one, which is to write a Group By query for each period type and combine them all in a union. This obvious approach involves quite a few repeated scans of the source table, so we will describe two other approaches that are less obvious, but avoid so many scans and should be better performing. We'll use a simple set of generated test data to illustrate the three queries, and will provide the execution plans associated with each one.

After the section on static SQL, we will take the most efficient of the three static queries and implement it in dynamic SQL of two kinds. Dynamic SQL will allow the query to be generated from lists of measures and periods, and unlike the static queries, the metadata-driven code will not increase in length with the product of the list cardinalities. The first kind of dynamic SQL, using a pipelined function, allows for the period list to be parametrized, but not the measure list. The second kind, using a SQL macro, a feature available from Oracle database version 21c, allows both lists to be parametrized.

<img src="/images/2021/11/07/star_trek.jpg">

["It's SQL, Jim, but not as we know it"](https://oupacademic.tumblr.com/post/45375455323/misquotation-its-life-jim-but-not-as-we-know)

There is a recording on this article here: <a href="https://twitter.com/BrenPatF/status/1366062116264955912" target="_blank">Tweet</a>, and also in my GitHub project, <a href="https://github.com/BrenPatF/oracle_sql_projects" target="_blank">Small SQL projects</a>.

The static SQL section of this article first appeared in an article on my Wordpress blog [SQL for Period Aggregate Reporting](http://aprogrammerwrites.eu/?p=3006) in February 2021.

## Contents
[&darr; Report Requirement](#report-requirement)<br />
[&darr; Test Data](#test-data)<br />
[&darr; Static SQL Queries](#static-sql-queries)<br />
[&darr; Dynamic SQL Queries](#dynamic-sql-queries)<br />
[&darr; Conclusion](#conclusion)<br />
[&darr; See Also](#see-also)

## Report Requirement
[&uarr; Contents](#contents)<br />

Given source data comprising a set of measures defined for each month for a set of products, we want a report that lists, for each:
- month
- product
- measure

the aggregate values of the measure for the product over each of a list of periods going back from the month. For example, for a list of measures:
- Value
- Cost

and a list of periods:
- 1 Month
- 3 Months
- YTD  
- 1 Year

the report output for November 2020 and product PROD_ONE, based on test data shown below, might be:

```
Month       Product    Period             Value     Cost
----------- ---------- --------------- -------- --------
01-Nov-2020 PROD_ONE   P1 - 1 Month       4,585      137
                       P2 - 3 Months      7,436    1,016
                       P3 - YTD          56,181    5,407
                       P4 - 1 Year       58,894    6,289
```

The number of measure value cells is the product of the cardinalities of the sets of periods and of measures. In real-world problems there can be quite large numbers of periods and measures, and large data volumes, so that performance can be an important issue.

## Test Data
[&uarr; Contents](#contents)<br />

We take two products and generate three years of monthly sales and cost values randomly between fixed limits in each case. Here is the generation code, for the table and first product (script c_sales_history.sql):

```sql
CREATE TABLE sales_history (
        prod_code               VARCHAR2(30) NOT NULL,
        month_dt                DATE NOT NULL,
        sales_value             NUMBER(10,0),
        sales_cost              NUMBER(10,0),
        CONSTRAINT slh_pk       PRIMARY KEY (prod_code, month_dt)
)
/
PROMPT 3 years random values for PROD_ONE
INSERT INTO sales_history
WITH month_gen AS (
    SELECT LEVEL rn, Add_Months(Trunc(SYSDATE, 'MONTH'), LEVEL - 36) dt
      FROM DUAL
   CONNECT BY LEVEL < 37
)
SELECT 'PROD_ONE', dt,
       DBMS_Random.Value (low => 1000, high => 10000),
       DBMS_Random.Value (low => 100, high => 1000)
  FROM month_gen
/
```

Here is the table data generated, with running sums of the two measures, that can help in testing the queries:

```
Sales History Report with Running Sums

Product    Month          Value Value to Date     Cost Cost to Date
---------- ----------- -------- ------------- -------- ------------
PROD_ONE   01-Dec-2018    4,898         4,898      703          703
           01-Jan-2019    4,023         8,921      669        1,372
           01-Feb-2019    6,558        15,479      115        1,487
           01-Mar-2019    6,328        21,807      604        2,091
           01-Apr-2019    3,542        25,349      797        2,888
           01-May-2019    8,366        33,715      769        3,657
           01-Jun-2019    5,225        38,940      139        3,796
           01-Jul-2019    8,574        47,514      677        4,473
           01-Aug-2019    4,441        51,955      615        5,088
           01-Sep-2019    7,075        59,030      845        5,933
           01-Oct-2019    7,993        67,023      245        6,178
           01-Nov-2019    8,046        75,069      674        6,852
           01-Dec-2019    2,713        77,782      882        7,734
           01-Jan-2020    3,848        81,630      154        7,888
           01-Feb-2020    9,555        91,185      145        8,033
           01-Mar-2020    6,832        98,017      698        8,731
           01-Apr-2020    2,328       100,345      376        9,107
           01-May-2020    5,390       105,735      997       10,104
           01-Jun-2020    3,260       108,995      242       10,346
           01-Jul-2020    7,648       116,643      888       11,234
           01-Aug-2020    9,884       126,527      891       12,125
           01-Sep-2020    1,766       128,293      635       12,760
           01-Oct-2020    1,085       129,378      244       13,004
           01-Nov-2020    4,585       133,963      137       13,141
           01-Dec-2020    3,155       137,118      356       13,497
           01-Jan-2021    5,044       142,162      895       14,392
           01-Feb-2021    1,825       143,987      653       15,045
           01-Mar-2021    8,391       152,378      661       15,706
           01-Apr-2021    5,849       158,227      687       16,393
           01-May-2021    7,226       165,453      611       17,004
           01-Jun-2021    1,596       167,049      316       17,320
           01-Jul-2021    3,027       170,076      487       17,807
           01-Aug-2021    1,739       171,815      376       18,183
           01-Sep-2021    7,660       179,475      748       18,931
           01-Oct-2021    2,801       182,276      622       19,553
           01-Nov-2021    5,652       187,928      433       19,986
PROD_TWO   01-Dec-2018    5,056         5,056      784          784
...

72 rows selected.
```

## Static SQL Queries
[&uarr; Contents](#contents)<br />
[&darr; Group By Union Query](#group-by-union-query)<br />
[&darr; Analytic Functions and Unpivot Query](#analytic-functions-and-unpivot-query)<br />
[&darr; Single Group By with CASE Expressions Query](#single-group-by-with-case-expressions-query)

In this section we provide three static queries for the required report, with execution plans.

### Group By Union Query
[&uarr; Static SQL Queries](#static-sql-queries)<br />
[&darr; Notes on Group By Union Query](#notes-on-group-by-union-query)<br />
[&darr; Notes on Group By Union Query Execution Plan](#notes-on-group-by-union-query-execution-plan)

Here is the first query:

```sql
SELECT /*+ gather_plan_statistics XPLAN_UGB */ 
     month_dt, prod_code, 'P1 - 1 Month' per_tp, sales_value, sales_cost
  FROM sales_history
UNION ALL
SELECT drv.month_dt, drv.prod_code, 'P2 - 3 Months', Sum(msr.sales_value), Sum(msr.sales_cost)
  FROM sales_history drv
  JOIN sales_history msr
  ON msr.prod_code = drv.prod_code
   AND msr.month_dt BETWEEN Add_Months (drv.month_dt, -2) AND drv.month_dt
 GROUP BY drv.prod_code, drv.month_dt
UNION ALL
SELECT drv.month_dt, drv.prod_code, 'P3 - YTD', Sum(msr.sales_value), Sum(msr.sales_cost)
  FROM sales_history drv
  JOIN sales_history msr
  ON msr.prod_code = drv.prod_code
   AND msr.month_dt BETWEEN Trunc(drv.month_dt, 'YEAR') AND drv.month_dt
 GROUP BY drv.prod_code, drv.month_dt
UNION ALL
SELECT drv.month_dt, drv.prod_code, 'P4 - 1 Year', Sum(msr.sales_value), Sum(msr.sales_cost)
  FROM sales_history drv
  JOIN sales_history msr
  ON msr.prod_code = drv.prod_code
   AND msr.month_dt BETWEEN Add_Months (drv.month_dt, -11) AND drv.month_dt
 GROUP BY drv.prod_code, drv.month_dt
 ORDER BY 1, 2, 3
```

#### Notes on Group By Union Query
[&uarr; Group By Union Query](#group-by-union-query)<br />

- First union member subquery does not aggregate, and includes the label for the period type
- The remaining aggregation subqueries drive from one scan of the table and join a second instance
- The second instance has the date range for the period type in its join condition
- The gather_plan_statistics hint allows capture of plan statistics
- The XPLAN_UGB is a string used to identify the SQL id to  pass to the API for displaying the plan


Here are the results for the first four months from the first query (script period_agg_report_queries.sql), which are the same for the other two queries.
```
Periods Report by Union of Group Bys

Month       Product    Period             Value     Cost
----------- ---------- --------------- -------- --------
01-Dec-2018 PROD_ONE   P1 - 1 Month       4,898      703
                       P2 - 3 Months      4,898      703
                       P3 - YTD           4,898      703
                       P4 - 1 Year        4,898      703
            PROD_TWO   P1 - 1 Month       5,056      784
                       P2 - 3 Months      5,056      784
                       P3 - YTD           5,056      784
                       P4 - 1 Year        5,056      784
01-Jan-2019 PROD_ONE   P1 - 1 Month       4,023      669
                       P2 - 3 Months      8,921    1,372
                       P3 - YTD           4,023      669
                       P4 - 1 Year        8,921    1,372
            PROD_TWO   P1 - 1 Month       4,806      808
                       P2 - 3 Months      9,862    1,592
                       P3 - YTD           4,806      808
                       P4 - 1 Year        9,862    1,592
01-Feb-2019 PROD_ONE   P1 - 1 Month       6,558      115
                       P2 - 3 Months     15,479    1,487
                       P3 - YTD          10,581      784
                       P4 - 1 Year       15,479    1,487
            PROD_TWO   P1 - 1 Month       3,891      263
                       P2 - 3 Months     13,753    1,855
                       P3 - YTD           8,697    1,071
                       P4 - 1 Year       13,753    1,855
01-Mar-2019 PROD_ONE   P1 - 1 Month       6,328      604
                       P2 - 3 Months     16,909    1,388
                       P3 - YTD          16,909    1,388
                       P4 - 1 Year       21,807    2,091
            PROD_TWO   P1 - 1 Month       7,114      487
                       P2 - 3 Months     15,811    1,558
                       P3 - YTD          15,811    1,558
                       P4 - 1 Year       20,867    2,342
...
288 rows selected.
```

Here is the execution plan:
```
Plan hash value: 3154844439
----------------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |               |      1 |        |    288 |00:00:00.01 |      27 |       |       |          |
|   1 |  SORT ORDER BY        |               |      1 |    225 |    288 |00:00:00.01 |      27 | 36864 | 36864 |32768  (0)|
|   2 |   UNION-ALL           |               |      1 |        |    288 |00:00:00.01 |      27 |       |       |          |
|   3 |    TABLE ACCESS FULL  | SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       6 |       |       |          |
|   4 |    HASH GROUP BY      |               |      1 |     51 |     72 |00:00:00.01 |       7 |   988K|   988K|          |
|*  5 |     HASH JOIN         |               |      1 |     67 |    210 |00:00:00.01 |       7 |  1506K|  1506K|  920K (0)|
|   6 |      INDEX FULL SCAN  | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
|   7 |      TABLE ACCESS FULL| SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       6 |       |       |          |
|   8 |    HASH GROUP BY      |               |      1 |     51 |     72 |00:00:00.01 |       7 |   988K|   988K|          |
|*  9 |     HASH JOIN         |               |      1 |     67 |    446 |00:00:00.01 |       7 |  1506K|  1506K|  922K (0)|
|  10 |      INDEX FULL SCAN  | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
|  11 |      TABLE ACCESS FULL| SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       6 |       |       |          |
|  12 |    HASH GROUP BY      |               |      1 |     51 |     72 |00:00:00.01 |       7 |   988K|   988K|          |
|* 13 |     HASH JOIN         |               |      1 |     67 |    732 |00:00:00.01 |       7 |  1506K|  1506K|  922K (0)|
|  14 |      INDEX FULL SCAN  | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
|  15 |      TABLE ACCESS FULL| SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       6 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
5 - access("MSR"."PROD_CODE"="DRV"."PROD_CODE")
filter(("MSR"."MONTH_DT"<="DRV"."MONTH_DT" AND "MSR"."MONTH_DT">=ADD_MONTHS(INTERNAL_FUNCTION("DRV"."MONTH_DT
"),-2)))
9 - access("MSR"."PROD_CODE"="DRV"."PROD_CODE")
filter(("MSR"."MONTH_DT"<="DRV"."MONTH_DT" AND "MSR"."MONTH_DT">=TRUNC(INTERNAL_FUNCTION("DRV"."MONTH_DT"),'f
myear')))
13 - access("MSR"."PROD_CODE"="DRV"."PROD_CODE")
filter(("MSR"."MONTH_DT"<="DRV"."MONTH_DT" AND "MSR"."MONTH_DT">=ADD_MONTHS(INTERNAL_FUNCTION("DRV"."MONTH_DT
"),-11)))
```

#### Notes on Group By Union Query Execution Plan
[&uarr; Group By Union Query](#group-by-union-query)

- There are 4 full table scans, and 3 index full scans
- The Buffers value of 27 is a measure of the work done, in logical I/O operations

### Analytic Functions and Unpivot Query
[&uarr; Static SQL Queries](#static-sql-queries)<br />
[&darr; Notes on Analytic Functions and Unpivot Query](#notes-on-analytic-functions-and-unpivot-query)<br />
[&darr; Notes on Analytic Functions and Unpivot Query Execution Plan](#notes-on-analytic-functions-and-unpivot-query-execution-plan)

Here is the second query:
```sql
WITH period_aggs AS (
  SELECT /*+ gather_plan_statistics XPLAN_AAG */ 
       month_dt, prod_code, sales_value, 
       Sum(sales_value) OVER (PARTITION BY prod_code ORDER BY month_dt
                  RANGE BETWEEN INTERVAL '2' MONTH PRECEDING AND CURRENT ROW)     sales_value_3m, 
       Sum(sales_value) OVER (PARTITION BY prod_code, Trunc(month_dt, 'YEAR') ORDER BY month_dt
                  RANGE BETWEEN INTERVAL '11' MONTH PRECEDING AND CURRENT ROW)    sales_value_ytd, 
       Sum(sales_value) OVER (PARTITION BY prod_code ORDER BY month_dt
                  RANGE BETWEEN INTERVAL '11' MONTH PRECEDING AND CURRENT ROW)    sales_value_1y, 
       sales_cost,
       Sum(sales_cost) OVER (PARTITION BY prod_code ORDER BY month_dt
                  RANGE BETWEEN INTERVAL '2' MONTH PRECEDING AND CURRENT ROW)     sales_cost_3m, 
       Sum(sales_cost) OVER (PARTITION BY prod_code, Trunc(month_dt, 'YEAR') ORDER BY month_dt
                  RANGE BETWEEN INTERVAL '11' MONTH PRECEDING AND CURRENT ROW)    sales_cost_ytd, 
       Sum(sales_cost) OVER (PARTITION BY prod_code ORDER BY month_dt
                  RANGE BETWEEN INTERVAL '11' MONTH PRECEDING AND CURRENT ROW)    sales_cost_1y
    FROM sales_history
)
SELECT *
  FROM period_aggs
UNPIVOT (
    (sales_value, sales_cost)
    FOR per_tp IN (
      (sales_value, sales_cost)         AS 'P1 - 1 Month',
      (sales_value_3m, sales_cost_3m)   AS 'P2 - 3 Months',
      (sales_value_ytd, sales_cost_ytd) AS 'P3 - YTD',
      (sales_value_1y, sales_cost_1y)   AS 'P4 - 1 Year'
    )
)
 ORDER BY 1, 2, 3
```

#### Notes on Analytic Functions and Unpivot Query
[&uarr; Analytic Functions and Unpivot Query](#analytic-functions-and-unpivot-query)

- For each measure a column is added for each period type to do the aggregation via analytic functions
- The UNPIVOT clause in the main query converts the period type columns into rows with column pair as specified in the first line
- The column name pair is specified in the first line for the unpivoted row values
- The 'FOR per_tp IN' clauses specifies the name of the period type column with values given in the rows below

Here is the execution plan:

```
Plan hash value: 1117112452
----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                         | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                  |               |      1 |        |    288 |00:00:00.01 |       2 |       |       |          |
|   1 |  SORT ORDER BY                    |               |      1 |    288 |    288 |00:00:00.01 |       2 | 27648 | 27648 |24576  (0)|
|*  2 |   VIEW                            |               |      1 |    288 |    288 |00:00:00.01 |       2 |       |       |          |
|   3 |    UNPIVOT                        |               |      1 |        |    288 |00:00:00.01 |       2 |       |       |          |
|   4 |     VIEW                          |               |      1 |     72 |     72 |00:00:00.01 |       2 |       |       |          |
|   5 |      WINDOW SORT                  |               |      1 |     72 |     72 |00:00:00.01 |       2 | 13312 | 13312 |12288  (0)|
|   6 |       WINDOW BUFFER               |               |      1 |     72 |     72 |00:00:00.01 |       2 |  6144 |  6144 | 6144  (0)|
|   7 |        TABLE ACCESS BY INDEX ROWID| SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       2 |       |       |          |
|   8 |         INDEX FULL SCAN           | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
2 - filter(("unpivot_view_006"."SALES_VALUE" IS NOT NULL OR "unpivot_view_006"."SALES_COST" IS NOT NULL))
```

#### Notes on Analytic Functions and Unpivot Query Execution Plan
[&uarr; Analytic Functions and Unpivot Query](#analytic-functions-and-unpivot-query)

- There is a single index full scan and a single table access by index rowid 
- The number of buffers is only 2
- The plan suggests that this query will be a lot more efficient than the first one

### Single Group By with CASE Expressions Query
[&uarr; Static SQL Queries](#static-sql-queries)<br />
[&darr; Notes on Single Group By with CASE Expressions Query](#notes-on-single-group-by-with-case-expressions-query)<br />
[&darr; Notes on Single Group By with CASE Expressions Query Execution Plan](#notes-on-single-group-by-with-case-expressions-query-execution-plan)

Here is the third query:

```sql
WITH period_list AS (
  SELECT month_dt, prod_code, COLUMN_VALUE per_tp
    FROM TABLE(SYS.ODCIVarchar2List(
          'P1 - 1 Month',
          'P2 - 3 Months',
          'P3 - YTD',
          'P4 - 1 Year')
      )
  CROSS JOIN (SELECT month_dt, prod_code FROM sales_history)
)
SELECT /*+ gather_plan_statistics XPLAN_GBC */
       drv.month_dt, drv.prod_code, drv.per_tp,
       Sum( CASE WHEN ( per_tp = 'P1 - 1 Month'  AND msr.month_dt = drv.month_dt ) OR 
                      ( per_tp = 'P2 - 3 Months' AND msr.month_dt >= Add_Months (drv.month_dt, -2) ) OR 
                      ( per_tp = 'P3 - YTD'      AND Trunc (msr.month_dt, 'YEAR') = Trunc (drv.month_dt, 'YEAR') ) OR 
                      ( per_tp = 'P4 - 1 Year'   AND msr.month_dt >= Add_Months (drv.month_dt, -11) )
                 THEN msr.sales_value END) sales_value,
       Sum( CASE WHEN ( per_tp = 'P1 - 1 Month'  AND msr.month_dt = drv.month_dt ) OR 
                      ( per_tp = 'P2 - 3 Months' AND msr.month_dt >= Add_Months (drv.month_dt, -2) ) OR 
                      ( per_tp = 'P3 - YTD'      AND Trunc (msr.month_dt, 'YEAR') = Trunc (drv.month_dt, 'YEAR') ) OR 
                      ( per_tp = 'P4 - 1 Year'   AND msr.month_dt >= Add_Months (drv.month_dt, -11) )
                 THEN msr.sales_cost END) sales_cost
  FROM period_list drv
  JOIN sales_history msr
  ON msr.prod_code = drv.prod_code
   AND msr.month_dt <= drv.month_dt
 GROUP BY drv.prod_code, drv.month_dt, drv.per_tp
 ORDER BY 1, 2, 3
```

#### Notes on Single Group By with CASE Expressions Query
[&uarr; Single Group By with CASE Expressions Query](#single-group-by-with-case-expressions-query)

- In the first subquery we add in the period type values for each product and month
- The main query then includes the extra column in its grouping fields
- The main query drives from the first subquery, joining the table to aggregate over, and including only records not later than the driving record
- The CASE expressions within the Sums ensure that a measure is counted in the sum only if its date on the joined table falls in the required range for the period type, relative to the date in the driving subquery

```
Plan hash value: 1337298906
------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                 | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                          |               |      1 |        |    288 |00:00:00.01 |       7 |       |       |          |
|   1 |  SORT GROUP BY                            |               |      1 |     51 |    288 |00:00:00.01 |       7 | 40960 | 40960 |36864  (0)|
|*  2 |   HASH JOIN                               |               |      1 |     10M|   5328 |00:00:00.01 |       7 |  1476K|  1476K|  847K (0)|
|   3 |    INDEX FULL SCAN                        | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
|   4 |    MERGE JOIN CARTESIAN                   |               |      1 |    588K|    288 |00:00:00.01 |       6 |       |       |          |
|   5 |     TABLE ACCESS FULL                     | SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       6 |       |       |          |
|   6 |     BUFFER SORT                           |               |     72 |   8168 |    288 |00:00:00.01 |       0 |  2048 |  2048 | 2048  (0)|
|   7 |      COLLECTION ITERATOR CONSTRUCTOR FETCH|               |      1 |   8168 |      4 |00:00:00.01 |       0 |       |       |          |
------------------------------------------------------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
2 - access("MSR"."PROD_CODE"="PROD_CODE")
filter("MSR"."MONTH_DT"<="MONTH_DT")
```

#### Notes on Single Group By with CASE Expressions Query Execution Plan
[&uarr; Single Group By with CASE Expressions Query](#single-group-by-with-case-expressions-query)

- There is a single index full scan and a single full table access
- The number of buffers is 7
- The plan suggests that this query will also be a lot more efficient than the first one
- The data set is too small to be conclusive regarding performance comparison with the second query

## Dynamic SQL Queries
[&uarr; Contents](#contents)<br />
[&darr; Pipelined Function Query](#pipelined-function-query)<br />
[&darr; Macro Query](#macro-query)

In this section we provide two dynamic SQL queries for the required report, with execution plans, using PL/SQL functions of type:
- Pipelined function
- SQL macro

In each case the query produced is the same as the second static query above, as that appears to be the most efficient.

### Pipelined Function Query
[&uarr; Dynamic SQL Queries](#dynamic-sql-queries)<br />
[&darr; Notes on Pipelined Function](#notes-on-pipelined-function)<br />
[&darr; Notes on Pipelined Function Query Execution Plan](#notes-on-pipelined-function-query-execution-plan)

Here is the query:

```sql
SELECT *
  FROM Period_Agg_Dynamic.Period_Aggs(p_period_tps_lis => L1_chr_arr('cur', '3m', 'ytd', '1y'), 
                                      p_period_nos_lis => L1_num_arr( 0,     2,    11,    11))
 ORDER BY 1, 2, 3
```

...and here is the pipelined function:

```sql
FUNCTION Period_Aggs(
            p_period_tps_lis               L1_chr_arr,                 -- period types to include (eg 'ytd', '1y' etc.)
            p_period_nos_lis               L1_num_arr)                 -- # period months to go back per period name
            RETURN                         period_agg_arr PIPELINED IS -- array of report records
  csr_period_aggs     SYS_REFCURSOR;
  l_column_lis        L1_chr_arr := L1_chr_arr('sales_value', 'sales_cost');
  l_query_text          VARCHAR2(4000) := 'WITH period_aggs AS (' ||
                                        'SELECT /*+ gather_plan_statistics XPLAN_PLF */ ' ||
                                        ' month_dt, prod_code';
  l_period_aggs       period_agg_rec;
BEGIN
  FOR i IN 1..l_column_lis.COUNT LOOP
    FOR j IN 1..p_period_tps_lis.COUNT LOOP
      l_query_text := l_query_text || ', ' || 
        CASE p_period_tps_lis(j) 
          WHEN 'cur' THEN l_column_lis(i) 
        ELSE
          'Sum(' || l_column_lis(i) || ') OVER (PARTITION BY prod_code ORDER BY ' ||
            CASE p_period_tps_lis(j) WHEN 'ytd' THEN 'Trunc(month_dt, ''YEAR'')' 
                                                ELSE ' month_dt' END ||
            ' RANGE BETWEEN INTERVAL ''' || p_period_nos_lis(j) || 
            ''' MONTH PRECEDING AND CURRENT ROW) ' ||
            l_column_lis(i) || CASE WHEN p_period_tps_lis(j)  != 'cur' THEN '_' || 
            p_period_tps_lis(j) END
        END;
    END LOOP;
  END LOOP;

  l_query_text := l_query_text || ' FROM sales_history) SELECT * FROM period_aggs UNPIVOT ((';
  FOR i IN 1..l_column_lis.COUNT LOOP
    l_query_text := l_query_text || l_column_lis(i) || ',';
  END LOOP;
  l_query_text := RTrim(l_query_text, ',') || ') FOR per_tp IN (';

  FOR j IN 1..p_period_tps_lis.COUNT LOOP
    l_query_text := l_query_text || '(';
    FOR i IN 1..l_column_lis.COUNT LOOP
      l_query_text := l_query_text || l_column_lis(i) || CASE WHEN p_period_tps_lis(j) != 'cur' THEN '_' ||
                    p_period_tps_lis(j) END || ',';
    END LOOP;
    l_query_text := RTrim(l_query_text, ',') || ') AS ''P' || j || ' - ' || p_period_tps_lis(j) || ''',';
  END LOOP;
  l_query_text := RTrim(l_query_text, ',') || '))';

  OPEN csr_period_aggs FOR l_query_text;
  LOOP
      FETCH csr_period_aggs INTO l_period_aggs;
      EXIT WHEN csr_period_aggs%NOTFOUND;
      PIPE ROW (l_period_aggs);
  END LOOP;
END Period_Aggs;
```

#### Notes on Pipelined Function
[&uarr; Pipelined Function Query](#pipelined-function-query)

- The query text is constructed with nested loops over the column and period lists for the select list of analytic expressions
- The unpivot clause is constructed with nested loops over the period and column lists
- Once constructed, the query is opened via e ref cursor variable, and the rows returned are piped out
- The period lists, of type names and number of aggregation months, are passed as parameters
- The record structure with given column list is specified as a record type in the package spec, so can't be parametrized
- The execution plan marker string, XPLAN_PLF, is specified in the query within the function

Here is the execution plan:

```
Plan hash value: 2203348307
---------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |               |      1 |        |    288 |00:00:00.01 |       2 |       |       |          |
|*  1 |  VIEW                            |               |      1 |    288 |    288 |00:00:00.01 |       2 |       |       |          |
|   2 |   UNPIVOT                        |               |      1 |        |    288 |00:00:00.01 |       2 |       |       |          |
|   3 |    VIEW                          |               |      1 |     72 |     72 |00:00:00.01 |       2 |       |       |          |
|   4 |     WINDOW SORT                  |               |      1 |     72 |     72 |00:00:00.01 |       2 | 13312 | 13312 |12288  (0)|
|   5 |      WINDOW BUFFER               |               |      1 |     72 |     72 |00:00:00.01 |       2 |  6144 |  6144 | 6144  (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID| SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       2 |       |       |          |
|   7 |        INDEX FULL SCAN           | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
1 - filter(("unpivot_view_006"."SALES_VALUE" IS NOT NULL OR "unpivot_view_006"."SALES_COST" IS NOT NULL))
```

#### Notes on Pipelined Function Query Execution Plan
[&uarr; Pipelined Function Query](#pipelined-function-query)

- The execution plan is almost the same as for the static query
- The `SORT ORDER BY` step is missing because the `ORDER BY` clause is in the outer query

### Macro Query
[&uarr; Dynamic SQL Queries](#dynamic-sql-queries)<br />
[&darr; Notes on Macro](#notes-on-macro)<br />
[&darr; Notes on Macro Query Execution Plan](#notes-on-macro-query-execution-plan)

Here is the query:

```sql
SELECT  /*+ gather_plan_statistics XPLAN_MAC */ *
  FROM Period_Agg_Dynamic.Period_Aggs_Macro(p_column_lis     => L1_chr_arr('sales_value', 'sales_cost'), 
                                            p_period_tps_lis => L1_chr_arr('cur', '3m', 'ytd', '1y'),
                                            p_period_nos_lis => L1_num_arr( 0,     2,    11,    11))
 ORDER BY 1, 2, 3
```

...and here is the macro:

```sql
FUNCTION Period_Aggs_Macro(
            p_column_lis                   L1_chr_arr,           -- columns to include
            p_period_tps_lis               L1_chr_arr,           -- period types to include (eg 'ytd', '1y' etc.)
            p_period_nos_lis               L1_num_arr)           -- # period months to go back per period name
            RETURN                         VARCHAR2 SQL_MACRO IS -- SQL macro = text of query (upto 4000ch)
  l_query_text          VARCHAR2(4000) := 'WITH period_aggs AS (' ||
                                        'SELECT month_dt, prod_code';
BEGIN
  FOR i IN 1..p_column_lis.COUNT LOOP
    FOR j IN 1..p_period_tps_lis.COUNT LOOP
      l_query_text := l_query_text || ', ' || 
        CASE p_period_tps_lis(j) 
          WHEN 'cur' THEN p_column_lis(i) 
        ELSE
          'Sum(' || p_column_lis(i) || ') OVER (PARTITION BY prod_code ORDER BY ' ||
            CASE p_period_tps_lis(j) WHEN 'ytd' THEN 'Trunc(month_dt, ''YEAR'')' ELSE ' month_dt' END ||
            ' RANGE BETWEEN INTERVAL ''' || p_period_nos_lis(j) || 
            ''' MONTH PRECEDING AND CURRENT ROW) ' ||
            p_column_lis(i) || CASE WHEN p_period_tps_lis(j)  != 'cur' THEN '_' || 
            p_period_tps_lis(j) END
        END;
    END LOOP;
  END LOOP;

  l_query_text := l_query_text || ' FROM sales_history) SELECT * FROM period_aggs UNPIVOT ((';
  FOR i IN 1..p_column_lis.COUNT LOOP
    l_query_text := l_query_text || p_column_lis(i) || ',';
  END LOOP;

  l_query_text := RTrim(l_query_text, ',') || ') FOR per_tp IN (';
  FOR j IN 1..p_period_tps_lis.COUNT LOOP
    l_query_text := l_query_text || '(';
    FOR i IN 1..p_column_lis.COUNT LOOP
      l_query_text := l_query_text || p_column_lis(i) || CASE WHEN p_period_tps_lis(j) != 'cur' THEN '_' ||
       p_period_tps_lis(j) END || ',';
    END LOOP;
    l_query_text := RTrim(l_query_text, ',') || ') AS ''P' || j || ' - ' || p_period_tps_lis(j) || ''',';
  END LOOP;
  l_query_text := RTrim(l_query_text, ',') || '))';
  RETURN l_query_text;
END Period_Aggs_Macro;
```

#### Notes on Macro
[&uarr; Macro Query](#macro-query)

- The query text is constructed in the same way as in the pipelined function
- Instead of being executed directly, the query text forms the return value
- The period lists, of type names and numbers of aggregation months, are passed as parameters
- The measure column list is also passed as a parameter
- The execution plan marker string, XPLAN_MAC, is specified in the outer query, not within the macro

Here is the execution plan:

```
Plan hash value: 1117112452
----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                         | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                  |               |      1 |        |    288 |00:00:00.01 |       2 |       |       |          |
|   1 |  SORT ORDER BY                    |               |      1 |    288 |    288 |00:00:00.01 |       2 | 33792 | 33792 |30720  (0)|
|*  2 |   VIEW                            |               |      1 |    288 |    288 |00:00:00.01 |       2 |       |       |          |
|   3 |    UNPIVOT                        |               |      1 |        |    288 |00:00:00.01 |       2 |       |       |          |
|   4 |     VIEW                          |               |      1 |     72 |     72 |00:00:00.01 |       2 |       |       |          |
|   5 |      WINDOW SORT                  |               |      1 |     72 |     72 |00:00:00.01 |       2 | 13312 | 13312 |12288  (0)|
|   6 |       WINDOW BUFFER               |               |      1 |     72 |     72 |00:00:00.01 |       2 |  6144 |  6144 | 6144  (0)|
|   7 |        TABLE ACCESS BY INDEX ROWID| SALES_HISTORY |      1 |     72 |     72 |00:00:00.01 |       2 |       |       |          |
|   8 |         INDEX FULL SCAN           | SLH_PK        |      1 |     72 |     72 |00:00:00.01 |       1 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
2 - filter(("unpivot_view_010"."SALES_VALUE" IS NOT NULL OR "unpivot_view_010"."SALES_COST" IS NOT NULL))
```

#### Notes on Macro Query Execution Plan
[&uarr; Macro Query](#macro-query)

- The execution plan is identical to that of the static query

## Conclusion
[&uarr; Contents](#contents)

- We have given three queries in static SQL to provide reports on sales performance across multiple time periods
- The execution plans show varying performance characteristics, with the use of analytic aggregation and unpivoting appearing most efficient
- Two dynamic SQL implementations of the most efficient query have been given, using pipelined functions and SQL macros
- Dynamic SQL solutions offer the possibility to parametrize period lists, and with SQL macros, also measure lists
- The dynamic SQL functions also avoid the explicit growth in complexity of the query code with period and measure cardinalities by generating the query from the lists

## See Also
[&uarr; Contents](#contents)<br />
- [Utils - Oracle PL/SQL general utilities module](https://github.com/BrenPatF/oracle_plsql_utils)
