---
layout: post
title: "Dimensional Benchmarking of General SQL Bursting Problems"
date: 2016-11-26
migrated: true
dimben_prev: /dimben/dimben-1/
dimben_next: /dimben/dimben-3/
categories: 
  - "match_recognize"
  - "model"
  - "oracle"
  - "performance"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "testing"
  - "v12"
tags: 
  - "benchmarking"
  - "match_recognize"
  - "model-2"
  - "oracle"
  - "performance-2"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "tuning"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
#### Part 2 in a series on: Dimensional Benchmarking of SQL Performance

<div class="dimben-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 DIMBEN Series:</strong>
  <a href="/migrated/dimben-series-index/">Index</a>
  {% if page.dimben_prev %}
    | <a href="{{ page.dimben_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.dimben_next %}
    | <a href="{{ page.dimben_next }}">Next ▶</a>
  {% endif %}
</div>

I recently posted an article on [Dimensional Benchmarking of Oracle v10-v12 Queries for SQL Bursting Problems](http://aprogrammerwrites.eu/?p=1836). This article added an Oracle v12 SQL solution, involving Match\_Recognize to benchmark against some v10 and v11 solutions that I had posted on Scribd a few years ago. A few days before posting it I noticed an OTN thread with a problem that struck me as being of a similar type, [Amalgamating groups to be beyond a given size threshold](https://community.oracle.com/thread/3987322). Where in my original 'bursting' problem a group is defined by a maximum interval from its starting date, in the OTN problem a group is defined by the cumulative sum of a numeric attribute from the group starting record.

I added a comment on the thread at the time mentioning the results that I had got on the original problem, and adding a model solution for the problem raised on the new thread. I have now taken this second 'bursting'-type problem and have benchmarked both the main two solutions proposed on that thread (by other posters), as well as two versions of my own model solution, and a variant of the recursive subquery factor solution that uses a temporary table to achieve much faster performance.

Also, I noticed a question just yesterday on AskTom that is posing essentially the same problem as in my earlier article (which itself came from AskTom several years ago 🙂 ), [Complex sql](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:9532423000346256435).

The results show that Match\_Recognize, as before, is by far the most efficient solution. They also show that the faster solutions vary linearly with dataset size (within a given partition), while the slow ones vary quadratically. One interesting finding is that the solution by the Model clause can be changed from very slow, and quadratically varying, to linearly varying, and second in performance only to Match\_Recognize, by using a rule ordering clause (which avoids the need for automatic rules ordering).

I obtained these results on my Windows 10 home computer with Oracle 12.1, and used my own benchmarking framework, available on GitHub and described on the [index page](/migrated/dimben-series-index/) for this series of articles.

## Generalised 'Bursting' Problem Definition

The problem is to determine break groups using a running aggregate based on some function of the record attributes, with a defined ordering, starting from the group starting record, and with a group's end record defined by the aggregate reaching (or exceeding) some limit. One may consider the first record reaching (or exceeding) the limit to define the first record in the next group, as in the original bursting problem, or to be the last record in the current group, as in the OTN example.

The data are partitioned by some key in general.

## OTN-like Item Weights 'Bursting' Problem

The data structure used in this article is based on that of the original poster in the OTN thread, but with more generic table and column names.

### ITEMS Table

```
CREATE TABLE items (
    id          NUMBER NOT NULL, 
    cat         VARCHAR2(30) NOT NULL, 
    seq         NUMBER NOT NULL, 
    weight      NUMBER NOT NULL,
    CONSTRAINT itm_pk PRIMARY KEY (id)
)
/
CREATE INDEX items_N1 ON items (cat, seq)
/
DROP  SEQUENCE itm_s
/
CREATE SEQUENCE itm_s
/

```

### Functional Test Data

I created test data with a test weight limit of 10, as follows, with groups shown at detailed level. The first two categories are taken from the OTN problem, while I added a third category to test the case where the limit is not reached.

```
 ID CAT                                   SEQ     WEIGHT SUB_WEIGHT  FINAL_GRP
--- ------------------------------ ---------- ---------- ---------- ----------
  1 Rural                                  10          2          2          4
  2                                         9          3          5          4
  3                                         8          1          6          4
  4                                         7          4         10          4
  5                                         6         11         11          5
  6                                         5          2          2          9
  7                                         4          2          4          9
  8                                         3          4          8          9
  9                                         2         30         38          9
 10                                         1         12         12         10

 11 Urban                                  10          1          1         12
 12                                         9         12         13         12
 13                                         8          2          2         15
 14                                         7          5          7         15
 15                                         6          7         14         15
 16                                         5         15         15         16
 17                                         4         25         25         17
 18                                         3          2          2         20
 19                                         2          1          3         20
 20                                         1          8         11         20

 21 Suburban                               10          2          2         22
 22                                         9          1          3         22

22 rows selected.

```

The queries shown later give the groups at summary level, as follows:

```
CAT                              # Groups
------------------------------ ----------
Rural                                   4
Suburban                                1
Urban                                   5

```

### A Note on Functional Testing and 'Scenario Coverage'

In the Match\_Recognize query proposed in the OTN thread the pattern is defined in terms of two categories, say s and t, where:

- s denotes a record where the running sum < the limit
- t denotes a record where the running sum >= the limit

The pattern to match can be written as (s\* t?) meaning zero or more category s records, followed by zero or one category t records. This immediately suggests that any given match falls into one of the following scenarios for frequencies of (s, t):

1. (0, 0) - this looks like an empty set of records, but could be non-empty if null values were allowed for the weight
2. (1+, 0) - the case where the limit is not reached, which must be the last match if there are no null weights
3. (0, 1) - where the first record in a group reaches the limit by itself
4. (1+, 1) - where one or more records in a group are below the limit, followed by a record that reaches the limit

In the results above, we see that group 22 matches scenario 2, while groups 10, 16 and 17 match scenario 3, and the remainder match scenario 4. We take the weight to be not null so scenario 1 is not possible. This kind of 'scenario coverage' is much more important than the 'code coverage' that is often focussed on in testing, especially by object oriented programmers.

In the following sections for individual queries, the query (and other SQL) is listed first, followed by the execution plan for the largest problem (W40-D8000).

## Model Query 1 - Automatic Order (MOD\_QRY)

```
WITH all_rows AS (  
SELECT  id,
        cat, 
        seq, 
        weight,
        sub_weight,
        final_grp
  FROM items
 MODEL
    PARTITION BY (cat)
    DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn)
    MEASURES (id, weight, weight sub_weight, id final_grp, seq)
    RULES AUTOMATIC ORDER (
       sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END,
       final_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()])
    )
)
SELECT  cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows
  FROM all_rows
GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:21:10.69 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:21:10.69 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|55:38:12.41 |    1074 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |    320K|    320K|55:38:12.36 |    1074 |    38M|  4773K|   32M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.19 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------
```

### Notes on Model Query 1 - Automatic Order (MOD\QRY)_

The default rules order for Model is sequential, but if we specify sequential, or accept the default, in the query above, then we get the error:

```
ORA-32637: Self cyclic rule in sequential order Model
```

[Rule Dependency in AUTOMATIC ORDER Models](https://docs.oracle.com/database/121/DWHSG/sqlmodel.htm#DWHSG8794)

> In some cases, Oracle Database may not be able to ascertain that your model is acyclic even though there is no cyclical dependency among the rules. This can happen if you have complex expressions in your cell references. Oracle Database assumes that the rules are cyclic and employs a CYCLIC algorithm that evaluates the model iteratively based on the rules and data. Iteration stops as soon as convergence is reached and the results are returned. Convergence is defined as the state in which further executions of the model will not change values of any of the cell in the model. Convergence is certain to be reached when there are no cyclical dependencies.

When we specify automatic order, the solution is obtained without error using Oracle's cyclic algorithm (operation SQL MODEL CYCLIC). Unfortunately, in this case there is a large performance impact, and we will see in the results section that execution time varies as the square of the number of records within a partition, i.e. quadratically.

## Model Query 2 - Sequential Order (MOD\_QRY\_D)

```
WITH all_rows AS (  
SELECT  id,
        cat, 
        seq, 
        weight,
        sub_weight,
        final_grp
  FROM items
 MODEL
    PARTITION BY (cat)
    DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn)
    MEASURES (id, weight, weight sub_weight, id final_grp, seq)
    RULES (
       sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END,
       final_grp[ANY] ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()])
    )
)
SELECT 
        cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows
  FROM all_rows
GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:00:02.43 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:00:02.43 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|00:01:32.47 |    1074 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |    320K|    320K|00:01:32.42 |    1074 |    36M|  4844K|   29M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.18 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------
```

### Notes on Model Query 2 - Sequential Order (MOD\_QRY\_D)

In the query above, the rules order clause is omitted, thus defaulting to sequential, while avoiding the ORA-32637 error. This is achieved by specifying ORDER BY rn DESC on the left side of the second rule. The solution, via operation SQL MODEL ORDERED is much faster, and we will see in the results section that execution time now varies linearly with the number of records within a partition.

## Match Recognize Query (MTH\_QRY)

```
SELECT
        cat         cat,
        final_grp   final_grp,
        num_rows    num_rows
  FROM items
 MATCH_RECOGNIZE (
   PARTITION BY cat
   ORDER BY seq DESC
   MEASURES FINAL LAST (id) final_grp,
            COUNT(*) num_rows
   ONE ROW PER MATCH
   PATTERN (s* t?)
   DEFINE s AS Sum (weight) < 5000,
          t AS Sum (weight) >= 5000
 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |   3229 |00:00:00.17 |    1074 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                 |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |    320K|   3229 |00:00:00.17 |    1074 |    15M|  1460K|   13M (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
---------------------------------------------------------------------------------------------------------------------
```

### Notes on Match Recognize Query (MTH\_QRY)

The query above is essentially the same as one of the posters proposed on the OTN thread, with a slight tweak to the pattern that does not alter its meaning, and also changing it to return one row per match. The query performs much more efficiently than any of the other queries, using the Match\_Recognize clause introduced in Oracle 12.1 [SQL for Pattern Matching](https://docs.oracle.com/database/121/DWHSG/pattern.htm#DWHSG8956).

## Recursive Subquery Factors Query 1 - Direct (RSF\_QRY)

```
WITH itm AS (
SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn
  FROM items
), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS (
SELECT id, cat, rn, seq, weight, weight, 1
  FROM itm
 WHERE rn = 1
 UNION ALL
SELECT  itm.id,
        itm.cat,
        itm.rn,
        itm.seq,
        itm.weight,
        itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END,    
        rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END
  FROM rsq
  JOIN itm
    ON itm.rn        = rsq.rn + 1
   AND itm.cat       = rsq.cat
), final_grouping AS (
SELECT  cat             cat,
        First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
FROM rsq
)
SELECT  cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows
FROM final_grouping
GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |   3229 |00:08:50.80 |    9365K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     81 |   3229 |00:08:50.80 |    9365K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |       |      1 |     81 |    320K|00:08:50.80 |    9365K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     81 |    320K|00:08:50.75 |    9365K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |       |      1 |     81 |    320K|01:13:10.25 |    9365K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |    320K|01:13:10.14 |    9365K|  4096 |  4096 |   20M (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.34 |    1074 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |    320K|     40 |00:00:00.34 |    1074 |    23M|  1772K|   20M (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |    320K|    320K|00:00:00.04 |    1074 |       |       |          |
|*  9 |       HASH JOIN                              |       |   8000 |     80 |    319K|00:25:04.02 |    8592K|  1214K|  1214K| 1283K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   8000 |        |    320K|00:00:00.19 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   8000 |    320K|   2560M|00:30:26.18 |    8592K|       |       |          |
|  12 |         WINDOW SORT                          |       |   8000 |    320K|   2560M|00:24:31.69 |    8592K|    15M|  1460K|   13M (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   8000 |    320K|   2560M|00:04:01.43 |    8592K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")
```

### Notes on Recursive Subquery Factors Query 1 - Direct (RSF\_QRY)

This is based on the second of the recursive subquery factor queries in the OTN thread, and we can see the performance issue in the plan above. The recursive branch of the UNION ALL executes once for each record within a partition and performs a full scan on the items table each time. This results in execution time varying as the square of the number of records within a partition, as can be seen in the results section later. The performance can be much improved by using a temporary table, as in the next query.

## Recursive Subquery Factors Query 2 - With Temporary Table (RSF\_TMP)

### Temporary Table Definition

```sql
CREATE GLOBAL TEMPORARY TABLE items_tmp (
    id          NUMBER, 
    cat         VARCHAR2(30), 
    seq         NUMBER, 
    weight      NUMBER,
    itm_rownum  NUMBER
)
ON COMMIT DELETE ROWS
/
CREATE INDEX items_tmp_N1 ON items_tmp (itm_rownum, cat)
/
```

### SQL - Insert to temporary table

```
INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
```

### Query using temporary table

```sql
WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS (
SELECT id, cat, itm_rownum, seq, weight, weight, 1
  FROM items_tmp
 WHERE itm_rownum = 1
 UNION ALL
SELECT  /*+ INDEX (itm items_tmp_n1) */ itm.id, 
        itm.cat,
        itm.itm_rownum,
        itm.seq,
        itm.weight,
        itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END,    
        rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END
  FROM rsq
  JOIN items_tmp itm
    ON itm.itm_rownum   = rsq.rn + 1
   AND itm.cat          = rsq.cat
), final_grouping AS (
SELECT  cat             cat,
        First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
FROM rsq
)
SELECT  cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows
FROM final_grouping
GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |   3229 |00:00:02.69 |    1118K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     91 |   3229 |00:00:02.69 |    1118K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |              |      1 |     91 |    320K|00:00:02.69 |    1118K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     91 |    320K|00:00:02.63 |    1118K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |              |      1 |     91 |    320K|00:00:02.73 |    1118K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |    320K|00:00:02.65 |    1118K|  4096 |  4096 |   20M (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      43 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       3 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   8000 |     51 |    319K|00:00:01.15 |     346K|       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   8000 |        |    320K|00:00:00.07 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |    320K|      1 |    319K|00:00:00.78 |     346K|       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |    320K|   3191 |    319K|00:00:00.31 |   26059 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan
```

### Notes on Recursive Subquery Factors Query 2 - With Temporary Table (RSF\_TMP)

In this solution, the initial subquery from the previous query is written to a temporary table that is indexed on the join column. This means that the join in the recursive branch of the UNION ALL is indexed and much quicker, resulting in linear variation in execution time with the number of records in a partition.

Notice that it was necessary to hint the index usage. It is possible to achieve the indexed join without a hint by including a call to gather statistics in the pre-query SQL. Unfortunately, Oracle's DBMS\_Stats procedure performs a commit - which clears the data from the temporary table. Although we could get around the clearing of the table by making it a normal table and manually truncating it, it is probably better to accept this as a valid use-case for a hint - after all, the whole purpose of the temporary table is to permit index use.

## Performance Testing Results

The 'width' parameter is taken to be the number of cat values partitioning the dataset, while the 'depth' parameter is taken to be the number of records within each category. The weight is assigned a random integer between 1 and 100, and the weight limit is 5,000.

### Record Counts Table

<table border="0" cellpadding="0" cellspacing="0" style="width: 256px;" jive-data-cell="{&quot;color&quot;:&quot;#575757&quot;,&quot;textAlign&quot;:&quot;left&quot;,&quot;padding&quot;:&quot;NaN&quot;,&quot;backgroundColor&quot;:&quot;#00CCFF&quot;,&quot;fontFamily&quot;:&quot;arial,helvetica,sans-serif&quot;,&quot;verticalAlign&quot;:&quot;baseline&quot;}" jive-data-header="{&quot;color&quot;:&quot;#FFFFFF&quot;,&quot;backgroundColor&quot;:&quot;#6690BC&quot;,&quot;textAlign&quot;:&quot;center&quot;,&quot;padding&quot;:&quot;2&quot;}">
<tbody>
<tr>
<td class="xl67" colspan="3" height="20" style="background-color: transparent;" width="192">Input Record Counts</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%;">D1000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">10,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">20,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">40,000</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc;">D2000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right;">20,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right;">40,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right;">80,000</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%;">D4000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">40,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">80,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">160,000</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; background-color: #ccffcc;">D8000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; text-align: right;">80,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; text-align: right;">160,000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; text-align: right;">320,000</td>
</tr>
<tr>
<td class="xl67" colspan="3" height="20" style="background-color: transparent;">Output Record Counts</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; border-width: 0.5pt medium; border-style: solid none; border-color: #4f81bd -moz-use-text-color; background-color: #ccffcc; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%;">D1000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">105</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">209</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">422</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc;">D2000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right;">205</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right;">411</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right;">829</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%;">D4000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">406</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">815</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right;">1,625</td>
</tr>
<tr>
<td class="xl65" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; background-color: #ccffcc;">D8000</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; text-align: right;">808</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; text-align: right;">1,618</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; border-width: medium medium 0.5pt; border-style: none none solid; border-color: -moz-use-text-color -moz-use-text-color #4f81bd; text-align: right;">3,229</td>
</tr>
</tbody>
</table>

### Elapsed Times Table (elapsed seconds)

<table border="0" cellpadding="0" cellspacing="0" style="width: 598px;" jive-data-cell="{&quot;color&quot;:&quot;#366092&quot;,&quot;textAlign&quot;:&quot;right&quot;,&quot;padding&quot;:&quot;NaN&quot;,&quot;backgroundColor&quot;:&quot;#CCFFCC&quot;,&quot;fontFamily&quot;:&quot;Calibri&quot;,&quot;verticalAlign&quot;:&quot;baseline&quot;}" jive-data-header="{&quot;color&quot;:&quot;#FFFFFF&quot;,&quot;backgroundColor&quot;:&quot;#6690BC&quot;,&quot;textAlign&quot;:&quot;center&quot;,&quot;padding&quot;:&quot;2&quot;}">
<tbody>
<tr>
<td height="20" style="background-color: #ff6600; text-align: left; vertical-align: baseline;" width="86">MOD_QRY</td>
<td class="xl69" colspan="3" style="text-align: center; vertical-align: baseline; background-color: #ffff99;" width="192">Elapsed Seconds</td>
<td class="xl69" colspan="3" style="text-align: center; vertical-align: baseline; background-color: #ffcc00;" width="192">Depth Ratios to Prior</td>
<td class="xl69" colspan="2" style="text-align: center; vertical-align: baseline; background-color: #99cc00;" width="128">Width Ratios to Prior</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D1000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">16</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">47</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">99</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.1</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D2000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">62</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">190</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">397</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.1</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D4000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">243</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">762</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1,390</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.5</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.8</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D8000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">962</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2,082</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">5,566</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.7</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.2</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.7</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">Average</td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.6</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.8</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.8</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.2</td>
</tr>
<tr>
<td height="20" style="background-color: #ff6600; text-align: left; vertical-align: baseline;">MOD_QRY_D</td>
<td class="xl69" colspan="3" style="background-color: #ffff99; text-align: center; vertical-align: baseline;">Elapsed Seconds</td>
<td class="xl69" colspan="3" style="background-color: #ffcc00; text-align: center; vertical-align: baseline;">Depth Ratios to Prior</td>
<td class="xl69" colspan="2" style="background-color: #99cc00; text-align: center; vertical-align: baseline;">Width Ratios to Prior</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D1000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.08</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.16</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.31</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.9</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D2000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.16</td>
<td align="right" class="xl67" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.30</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.61</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D4000</td>
<td align="right" class="xl67" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.30</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.59</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.19</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D8000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.59</td>
<td align="right" class="xl67" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.20</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.42</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">Average</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td height="20" style="background-color: #ff6600; text-align: left; vertical-align: baseline;">MTH_QRY</td>
<td class="xl69" colspan="3" style="text-align: center; vertical-align: baseline; background-color: #ffff99;">Elapsed Seconds</td>
<td class="xl69" colspan="3" style="text-align: center; vertical-align: baseline; background-color: #ffcc00;">Depth Ratios to Prior</td>
<td class="xl69" colspan="2" style="text-align: center; vertical-align: baseline; background-color: #99cc00;">Width Ratios to Prior</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D1000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.016</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.016</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="center" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">#DIV/0!</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D2000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.016</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.016</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.047</td>
<td align="center" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">#DIV/0!</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.9</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D4000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.016</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.031</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.078</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.7</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.5</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D8000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.047</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.094</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.172</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.2</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.8</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">Average</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.3</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.6</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.1</td>
</tr>
<tr>
<td height="20" style="background-color: #ff6600; text-align: left; vertical-align: baseline;">RSF_QRY</td>
<td class="xl69" colspan="3" style="background-color: #ffff99; text-align: center; vertical-align: baseline;">Elapsed Seconds</td>
<td class="xl69" colspan="3" style="background-color: #ffcc00; text-align: center; vertical-align: baseline;">Depth Ratios to Prior</td>
<td class="xl69" colspan="2" style="background-color: #99cc00; text-align: center; vertical-align: baseline;">Width Ratios to Prior</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D1000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">6</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">12</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">24</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D2000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">23</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">46</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">94</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.8</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.8</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D4000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">92</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">185</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">377</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D8000</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">369</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">750</td>
<td align="right" class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1,513</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">Average</td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td class="xl65" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">3.9</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">4.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td height="20" style="background-color: #ff6600; text-align: left; vertical-align: baseline;">RSF_TMP</td>
<td class="xl69" colspan="3" style="background-color: #ffff99; text-align: center; vertical-align: baseline;">Elapsed Seconds</td>
<td class="xl69" colspan="3" style="background-color: #ffcc00; text-align: center; vertical-align: baseline;">Depth Ratios to Prior</td>
<td class="xl69" colspan="2" style="background-color: #99cc00; text-align: center; vertical-align: baseline;">Width Ratios to Prior</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">Depth</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffff99; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W10</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #ffcc00; text-align: right; vertical-align: baseline;">W40</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W20</td>
<td class="xl66" style="font-size: 11pt; color: #366092; font-weight: bold; font-family: Calibri; background-color: #99cc00; text-align: right; vertical-align: baseline;">W40</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D1000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.09</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.19</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.36</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.9</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D2000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.19</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.38</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.73</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.9</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ccffcc none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">D4000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.39</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">0.77</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">1.53</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #dce6f1 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background-color: #ccffcc; text-align: right; vertical-align: baseline;">D8000</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">0.77</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">1.55</td>
<td align="right" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">3.49</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.3</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; text-align: right; vertical-align: baseline;">2.3</td>
</tr>
<tr>
<td class="xl66" height="20" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">Average</td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc99 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;"></td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #ffcc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.1</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
<td align="right" class="xl68" style="font-size: 11pt; color: #366092; font-weight: 400; font-family: Calibri; background: #99cc00 none repeat scroll 0% 0%; text-align: right; vertical-align: baseline;">2.0</td>
</tr>
</tbody>
</table>

### Slice Graphs

<img src="/migrated_images/2016/11/Wide.png" alt="Wide" title="Wide" />

<img src="/migrated_images/2016/11/Deep.png" alt="Deep" title="Deep" />

## Performance Discussion

### Variation with Width

The width parameter represents the number of categories here, and category (CAT) is the query partitioning key. We might therefore expect that the execution time would be proportional to the width when the depth parameter is fixed. The width values used were 10, 20 and 40, so we would expect times to double between W10 and W20, and again between W20 and W40.

In fact, we see from the width ratios columns in the tables that this expectation is very closely matched in the cases of MOD\_QRY\_D, RSF\_QRY, AND RSF\_TMP.

For MOD\_QRY, the ratios are quite variable, and mostly above 2, so that the CYCLIC Model algorithm does not meet our expectation.

For MTH\_QRY (Match\_Recogize), the elapsed times are very small, 0.17 for the largest problem (14 times faster than the next best, MOD\_QRY\_D), and that likely explains the variance.

### Variation with Depth

The depth parameter represents the number of of records for each category. The depth ratios show that two of the queries show very close to quadratic variation of time with depth, while three show very close to linear variation, and the linear queries are unsurprisingly much faster.

MOD\_QRY and RSF\_QRY vary quadratically with depth (number of records per partition key).

MOD\_QRY\_D, MTH\_QRY, AND RSF\_TMP vary linearly with depth (number of records per partition key).

## Conclusion

- As in the earlier article, the new v12.1 feature Match\_Recogize proved to be much faster than the other techniques for this problem
- The solution using Model clause with the operation SQL MODEL CYCLIC showed quadratic variation in execution times with size, but a very simple change to allow SQL MODEL ORDERED operation produced linear variation, and was second only to Match\_Recogize in performance
- Recursive subquery factoring had timings that increased quadratically with number of records; this was due to a combination of the number of starts of a subquery, and full scans within it
- This kind of unscaleable quadratic resource usage can often be avoided by the use of a temporary table with appropriate indexes, as demonstrated here. There is another example of its use for performance improvement here, [SQL for Shortest Path Problems 2: A Branch and Bound Approach](http://aprogrammerwrites.eu/?p=1415)
- It's hard to see how you could detect some of the performance characteristics found here without benchmarking across varying problem sizes

## Full Output Log

<div class="scrollbox">
<pre>
SQL> 
SQL> COLUMN "Database"	FORMAT A20
SQL> COLUMN "Time"		FORMAT A20
SQL> COLUMN "Version"	FORMAT A30
SQL> COLUMN "Session"	FORMAT 9999990
SQL> COLUMN "OS User"	FORMAT A10
SQL> COLUMN "Machine"	FORMAT A20
SQL> SET LINES 180
SQL> SET PAGES 1000
SQL> 
SQL> SELECT 'Start: ' || dbs.name "Database", To_Char (SYSDATE,'DD-MON-YYYY HH24:MI:SS') "Time",
  2  	Replace (Substr(ver.banner, 1, Instr(ver.banner, '64')-4), 'Enterprise Edition Release ', '') "Version"
  3    FROM v$database dbs,  v$version ver
  4   WHERE ver.banner LIKE 'Oracle%';

Database             Time                 Version
-------------------- -------------------- ------------------------------
Start: ORCL          22-NOV-2016 20:46:15 Oracle Database 12c 12.1.0.2.0

SQL> 
SQL> DEFINE RUNDESC='Itm-One'
SQL> 
SQL> SET SERVEROUTPUT ON
SQL> SET TIMING ON
SQL> 
SQL> ALTER SESSION SET NLS_DATE_FORMAT = 'DD-MON-YYYY';

Session altered.

Elapsed: 00:00:00.00
SQL> BEGIN
  2  
  3    Utils.Clear_Log;
  4    Bench_Queries.Create_Run (
  5  			p_run_desc		=> '&RUNDESC',
  6  			p_points_wide_list	=> L1_num_arr (10, 20, 40),
  7  			p_points_deep_list	=> L1_num_arr (1000, 2000, 4000, 8000),
  8  /*
  9              p_points_wide_list	=> L1_num_arr (10),
 10  			p_points_deep_list	=> L1_num_arr (500, 1000, 2000),
 11  */
 12  			p_query_group		=> 'WEIGHTS',
 13              p_redo_data_yn      => 'Y');
 14    Bench_Queries.Execute_Run;
 15  
 16  END;
 17  /
old   5: 			p_run_desc		=> '&RUNDESC',
new   5: 			p_run_desc		=> 'Itm-One',

PL/SQL procedure successfully completed.

Elapsed: 04:17:04.13
SQL> PROMPT Default log
Default log
SQL> @../sql/L_Log_Default

TEXT
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Bench run 41 created

Elapsed: 00:00:00.00
SQL> PROMPT Execute_Run log
Execute_Run log
SQL> @../sql/L_Log_Gp

TEXT
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Bench run id = 41

Wide Points
===========
10, 20, 40

Deep Points
===========
1000, 2000, 4000, 8000
Header="cat","final_grp","num_rows","Random"
mid=
        cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows

new='"' ||  cat             || '","' || final_grp       || '","' ||
        COUNT(*) || '","#?"'

/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","#?"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

Header="cat","final_grp","num_rows","Random"
mid=
        cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows

new='"' ||  cat             || '","' || final_grp       || '","' ||
        COUNT(*) || '","#?"'

/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","#?"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

Header="cat","final_grp","num_rows","Random"
mid=
        cat         cat,
        final_grp   final_grp,
        num_rows    num_rows

new='"' ||  cat         || '","' || final_grp   || '","' ||
        num_rows || '","#?"'

/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","#?"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LAST
 (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

Header="cat","final_grp","num_rows","Random"
mid=
        cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows

new='"' ||  cat             || '","' || final_grp       || '","' ||
        COUNT(*) || '","#?"'

/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","#?"' FROM final_grouping GROUP BY cat, final_grp ORDER BY ca
t, final_grp

Header="cat","final_grp","num_rows","Random"
mid=
        cat             cat,
        final_grp       final_grp,
        COUNT(*)        num_rows

new='"' ||  cat             || '","' || final_grp       || '","' ||
        COUNT(*) || '","#?"'

/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","#?"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Items truncated
10000 (1000) records (per category) added, average group size (from) = 116.3 (1000), # of groups = 8.6

Timer Set: Setup, Constructed at 22 Nov 2016 20:46:15, written at 20:46:16
==========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   0.70        0.56             1        0.70300        0.56000
Gather_Table_Stats        0.08        0.08             1        0.07800        0.08000
GRP_CNT                   0.02        0.02             1        0.01600        0.02000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     0.80        0.66             4        0.19925        0.16500
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","9324"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  3f1awu08xfyms, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","9324"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    105 |00:00:16.01 |      38 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  10000 |    105 |00:00:16.01 |      38 |  9216 |  9216 | 8192  (0)|
|   2 |   VIEW                |       |      1 |  10000 |  10000 |00:00:07.47 |      38 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  10000 |  10000 |00:00:07.47 |      38 |  1869K|  1197K| 1695K (0)|
|   4 |     WINDOW SORT       |       |      1 |  10000 |  10000 |00:00:00.01 |      38 |   619K|   472K|  550K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  10000 |  10000 |00:00:00.01 |      38 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:46:16, written at 20:46:32
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             16.00       16.00             1       16.00200       16.00000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.27        0.27             1        0.26500        0.27000
(Other)                  0.03        0.03             1        0.03100        0.03000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   16.30       16.30             8        2.03725        2.03750
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:46:16, written at 20:46:32
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       16.31       16.31             1       16.31400       16.31000
------- ---------- ---------- ------------ ------------- -------------
Total         16.31       16.31             2        8.15700        8.15500
------- ---------- ---------- ------------ ------------- -------------
106 rows written to MOD_QRY.csv
Summary for W/D = 10/1000 , bench_run_statistics_id = 404

Timer Set: Run_One, Constructed at 22 Nov 2016 20:46:16, written at 20:46:32
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           16.33       16.33             1       16.33000       16.33000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         16.33       16.33             2        8.16500        8.16500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","1112"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  8h4usyvxw24zv, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","1112"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    105 |00:00:00.07 |      38 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  10000 |    105 |00:00:00.07 |      38 |  9216 |  9216 | 8192  (0)|
|   2 |   VIEW                |       |      1 |  10000 |  10000 |00:00:00.03 |      38 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  10000 |  10000 |00:00:00.03 |      38 |  1822K|  1215K| 1339K (0)|
|   4 |     WINDOW SORT       |       |      1 |  10000 |  10000 |00:00:00.01 |      38 |   619K|   472K|  550K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  10000 |  10000 |00:00:00.01 |      38 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:46:32, written at 20:46:33
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.08        0.08             1        0.07800        0.08000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.24        0.23             1        0.23500        0.23000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.33        0.32             8        0.04100        0.04000
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:46:32, written at 20:46:33
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.33        0.32             1        0.32800        0.32000
------- ---------- ---------- ------------ ------------- -------------
Total          0.33        0.32             2        0.16400        0.16000
------- ---------- ---------- ------------ ------------- -------------
106 rows written to MOD_QRY_D.csv
Summary for W/D = 10/1000 , bench_run_statistics_id = 405

Timer Set: Run_One, Constructed at 22 Nov 2016 20:46:32, written at 20:46:33
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.34        0.34             1        0.34300        0.34000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.34        0.34             2        0.17150        0.17000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","8765"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  c6gfsqmux9bs5, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","8765"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    105 |00:00:00.01 |      38 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  10000 |    105 |00:00:00.01 |      38 | 13312 | 13312 |12288  (0)|
|   2 |   VIEW                 |       |      1 |  10000 |    105 |00:00:00.01 |      38 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  10000 |    105 |00:00:00.01 |      38 |   619K|   472K|  550K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  10000 |  10000 |00:00:00.01 |      38 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:46:33, written at 20:46:33
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.00        0.00             1        0.00000        0.00000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.25        0.25             8        0.03125        0.03125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:46:33, written at 20:46:33
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.27        0.27             1        0.26600        0.27000
------- ---------- ---------- ------------ ------------- -------------
Total          0.27        0.27             2        0.13300        0.13500
------- ---------- ---------- ------------ ------------- -------------
106 rows written to MTH_QRY.csv
Summary for W/D = 10/1000 , bench_run_statistics_id = 406

Timer Set: Run_One, Constructed at 22 Nov 2016 20:46:33, written at 20:46:33
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.27        0.27             1        0.26600        0.27000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.27        0.27             2        0.13300        0.13500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","6581"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  dd67jjzfxbpr9, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","6581"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    105 |00:00:05.88 |   45370 |       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     11 |    105 |00:00:05.88 |   45370 |  9216 |  9216 | 8192  (0)|
|   2 |   VIEW                                       |       |      1 |     11 |  10000 |00:00:05.88 |   45370 |       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     11 |  10000 |00:00:05.88 |   45370 |   549K|   457K|  487K (0)|
|   4 |     VIEW                                     |       |      1 |     11 |  10000 |00:00:07.19 |   45370 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  10000 |00:00:07.19 |   45370 |  2048 |  2048 |  676K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     10 |00:00:00.01 |      38 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  10000 |     10 |00:00:00.01 |      38 |   761K|   499K|  676K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  10000 |  10000 |00:00:00.01 |      38 |       |       |          |
|*  9 |       HASH JOIN                              |       |   1000 |     10 |   9990 |00:00:05.62 |   38000 |  1214K|  1214K|  892K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   1000 |        |  10000 |00:00:00.01 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   1000 |  10000 |     10M|00:00:06.75 |   38000 |       |       |          |
|  12 |         WINDOW SORT                          |       |   1000 |  10000 |     10M|00:00:05.34 |   38000 |   619K|   472K|  550K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   1000 |  10000 |     10M|00:00:00.95 |   38000 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 20:46:33, written at 20:46:39
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.02             1        0.01500        0.02000
First fetch              5.88        5.87             1        5.87600        5.87000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    6.13        6.13             8        0.76563        0.76625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:46:33, written at 20:46:39
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        6.14        6.13             1        6.14100        6.13000
------- ---------- ---------- ------------ ------------- -------------
Total          6.14        6.13             2        3.07050        3.06500
------- ---------- ---------- ------------ ------------- -------------
106 rows written to RSF_QRY.csv
Summary for W/D = 10/1000 , bench_run_statistics_id = 407

Timer Set: Run_One, Constructed at 22 Nov 2016 20:46:33, written at 20:46:39
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            6.14        6.13             1        6.14100        6.13000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          6.14        6.13             2        3.07050        3.06500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","7865"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  06r0kbvv7kxxx, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","7865"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    105 |00:00:00.07 |   18689 |       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     20 |    105 |00:00:00.07 |   18689 |  9216 |  9216 | 8192  (0)|
|   2 |   VIEW                                       |              |      1 |     20 |  10000 |00:00:00.07 |   18689 |       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     20 |  10000 |00:00:00.07 |   18689 |   549K|   457K|  487K (0)|
|   4 |     VIEW                                     |              |      1 |     20 |  10000 |00:00:00.06 |   18689 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  10000 |00:00:00.06 |   18689 |  2048 |  2048 |  676K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     10 |     10 |00:00:00.01 |      12 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     10 |     10 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   1000 |     10 |   9990 |00:00:00.03 |   11345 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   1000 |        |  10000 |00:00:00.01 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  10000 |      1 |   9990 |00:00:00.02 |   11345 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  10000 |    100 |   9990 |00:00:00.01 |    1355 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 20:46:39, written at 20:46:40
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.02        0.02             1        0.01600        0.02000
Open cursor              0.02        0.01             1        0.01600        0.01000
First fetch              0.06        0.07             1        0.06200        0.07000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.27        0.26             1        0.26600        0.26000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.38        0.37             8        0.04688        0.04625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:46:39, written at 20:46:40
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.38        0.37             1        0.37500        0.37000
------- ---------- ---------- ------------ ------------- -------------
Total          0.38        0.37             2        0.18750        0.18500
------- ---------- ---------- ------------ ------------- -------------
106 rows written to RSF_TMP.csv
Summary for W/D = 10/1000 , bench_run_statistics_id = 408

Timer Set: Run_One, Constructed at 22 Nov 2016 20:46:39, written at 20:46:40
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.39        0.39             1        0.39000        0.39000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.39        0.39             2        0.19500        0.19500
------- ---------- ---------- ------------ ------------- -------------
Items truncated
20000 (2000) records (per category) added, average group size (from) = 111.7 (2000), # of groups = 17.9

Timer Set: Setup, Constructed at 22 Nov 2016 20:46:40, written at 20:46:41
==========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   1.17        1.06             1        1.17200        1.06000
Gather_Table_Stats        0.06        0.06             1        0.06300        0.06000
GRP_CNT                   0.00        0.00             1        0.00000        0.00000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     1.24        1.12             4        0.30875        0.28000
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","5462"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  34cyh2gw9bkva, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","5462"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    205 |00:01:01.50 |      68 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  20000 |    205 |00:01:01.50 |      68 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                |       |      1 |  20000 |  20000 |00:01:38.36 |      68 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  20000 |  20000 |00:01:38.35 |      68 |  2990K|  1197K| 3174K (0)|
|   4 |     WINDOW SORT       |       |      1 |  20000 |  20000 |00:00:00.01 |      68 |  1045K|   546K|  928K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:46:41, written at 20:47:43
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             61.51       61.50             1       61.50600       61.50000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.24        0.24             1        0.23500        0.24000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   61.76       61.75             8        7.71963        7.71875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:46:41, written at 20:47:43
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       61.77       61.76             1       61.77200       61.76000
------- ---------- ---------- ------------ ------------- -------------
Total         61.77       61.76             2       30.88600       30.88000
------- ---------- ---------- ------------ ------------- -------------
206 rows written to MOD_QRY.csv
Summary for W/D = 10/2000 , bench_run_statistics_id = 409

Timer Set: Run_One, Constructed at 22 Nov 2016 20:46:41, written at 20:47:43
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           61.77       61.76             1       61.77200       61.76000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         61.77       61.76             2       30.88600       30.88000
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","4455"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  5bps7t3zrrtpp, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","4455"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    205 |00:00:00.15 |      68 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  20000 |    205 |00:00:00.15 |      68 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                |       |      1 |  20000 |  20000 |00:00:01.68 |      68 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  20000 |  20000 |00:00:01.68 |      68 |  2883K|  1215K| 2462K (0)|
|   4 |     WINDOW SORT       |       |      1 |  20000 |  20000 |00:00:00.01 |      68 |  1045K|   546K|  928K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:47:43, written at 20:47:43
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.01             1        0.01600        0.01000
First fetch              0.14        0.15             1        0.14000        0.15000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.24        0.23             1        0.23500        0.23000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.39        0.39             8        0.04888        0.04875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:47:43, written at 20:47:43
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.41        0.39             1        0.40600        0.39000
------- ---------- ---------- ------------ ------------- -------------
Total          0.41        0.39             2        0.20300        0.19500
------- ---------- ---------- ------------ ------------- -------------
206 rows written to MOD_QRY_D.csv
Summary for W/D = 10/2000 , bench_run_statistics_id = 410

Timer Set: Run_One, Constructed at 22 Nov 2016 20:47:43, written at 20:47:43
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.41        0.39             1        0.40600        0.39000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.41        0.39             2        0.20300        0.19500
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","1173"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  93r2k5tp0npsh, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","1173"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    205 |00:00:00.01 |      68 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  20000 |    205 |00:00:00.01 |      68 | 22528 | 22528 |20480  (0)|
|   2 |   VIEW                 |       |      1 |  20000 |    205 |00:00:00.01 |      68 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  20000 |    205 |00:00:00.01 |      68 |  1045K|   546K|  928K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:47:43, written at 20:47:43
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.02        0.01             1        0.01600        0.01000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.25        0.25             8        0.03125        0.03125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:47:43, written at 20:47:43
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.25        0.25             1        0.25000        0.25000
------- ---------- ---------- ------------ ------------- -------------
Total          0.25        0.25             2        0.12500        0.12500
------- ---------- ---------- ------------ ------------- -------------
206 rows written to MTH_QRY.csv
Summary for W/D = 10/2000 , bench_run_statistics_id = 411

Timer Set: Run_One, Constructed at 22 Nov 2016 20:47:43, written at 20:47:43
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.27        0.26             1        0.26600        0.26000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.27        0.26             2        0.13300        0.13000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","9417"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  6g93u1jpujr2h, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","9417"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    205 |00:00:23.15 |     159K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     21 |    205 |00:00:23.15 |     159K| 15360 | 15360 |14336  (0)|
|   2 |   VIEW                                       |       |      1 |     21 |  20000 |00:00:23.15 |     159K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     21 |  20000 |00:00:23.15 |     159K|   974K|   535K|  865K (0)|
|   4 |     VIEW                                     |       |      1 |     21 |  20000 |00:00:28.88 |     159K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  20000 |00:00:28.88 |     159K|  2048 |  2048 | 1369K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     10 |00:00:00.02 |      68 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  20000 |     10 |00:00:00.02 |      68 |  1541K|   615K| 1369K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
|*  9 |       HASH JOIN                              |       |   2000 |     20 |  19990 |00:00:22.15 |     136K|  1214K|  1214K|  870K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   2000 |        |  20000 |00:00:00.01 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   2000 |  20000 |     40M|00:00:26.76 |     136K|       |       |          |
|  12 |         WINDOW SORT                          |       |   2000 |  20000 |     40M|00:00:21.22 |     136K|  1045K|   546K|  928K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   2000 |  20000 |     40M|00:00:03.56 |     136K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 20:47:43, written at 20:48:07
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             23.16       23.15             1       23.15900       23.15000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   23.41       23.41             8        2.92600        2.92625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:47:43, written at 20:48:07
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       23.42       23.42             1       23.42400       23.42000
------- ---------- ---------- ------------ ------------- -------------
Total         23.42       23.42             2       11.71200       11.71000
------- ---------- ---------- ------------ ------------- -------------
206 rows written to RSF_QRY.csv
Summary for W/D = 10/2000 , bench_run_statistics_id = 412

Timer Set: Run_One, Constructed at 22 Nov 2016 20:47:43, written at 20:48:07
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           23.42       23.42             1       23.42400       23.42000
(Other)        0.02        0.02             1        0.01600        0.02000
------- ---------- ---------- ------------ ------------- -------------
Total         23.44       23.44             2       11.72000       11.72000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","680"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  c28d82nb1rwpg, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","680"' FROM final_grouping GROUP BY cat, final_grp ORDER
BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    205 |00:00:00.14 |   46582 |       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     20 |    205 |00:00:00.14 |   46582 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                                       |              |      1 |     20 |  20000 |00:00:00.14 |   46582 |       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     20 |  20000 |00:00:00.14 |   46582 |   974K|   535K|  865K (0)|
|   4 |     VIEW                                     |              |      1 |     20 |  20000 |00:00:00.13 |   46582 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  20000 |00:00:00.13 |   46582 |  2048 |  2048 | 1369K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     10 |     10 |00:00:00.01 |      12 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     10 |     10 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   2000 |     10 |  19990 |00:00:00.06 |   22725 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   2000 |        |  20000 |00:00:00.01 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  20000 |      1 |  19990 |00:00:00.05 |   22725 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  20000 |    197 |  19990 |00:00:00.02 |    2735 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 20:48:07, written at 20:48:07
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.03        0.04             1        0.03200        0.04000
Open cursor              0.02        0.01             1        0.01500        0.01000
First fetch              0.14        0.14             1        0.14100        0.14000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.45        0.45             8        0.05663        0.05625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:48:07, written at 20:48:07
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.47        0.47             1        0.46900        0.47000
------- ---------- ---------- ------------ ------------- -------------
Total          0.47        0.47             2        0.23450        0.23500
------- ---------- ---------- ------------ ------------- -------------
206 rows written to RSF_TMP.csv
Summary for W/D = 10/2000 , bench_run_statistics_id = 413

Timer Set: Run_One, Constructed at 22 Nov 2016 20:48:07, written at 20:48:07
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.47        0.47             1        0.46900        0.47000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.47        0.47             2        0.23450        0.23500
------- ---------- ---------- ------------ ------------- -------------
Items truncated
40000 (4000) records (per category) added, average group size (from) = 107.8 (4000), # of groups = 37.1

Timer Set: Setup, Constructed at 22 Nov 2016 20:48:07, written at 20:48:10
==========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   2.30        2.11             1        2.29700        2.11000
Gather_Table_Stats        0.16        0.10             1        0.15700        0.10000
GRP_CNT                   0.02        0.01             1        0.01500        0.01000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     2.47        2.22             4        0.61725        0.55500
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","7000"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  6jjb8ybnuwrq4, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","7000"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    406 |00:04:03.06 |     175 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  40000 |    406 |00:04:03.06 |     175 | 31744 | 31744 |28672  (0)|
|   2 |   VIEW                |       |      1 |  40000 |  40000 |00:06:37.77 |     175 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  40000 |  40000 |00:06:37.77 |     175 |  5905K|  2393K| 5593K (0)|
|   4 |     WINDOW SORT       |       |      1 |  40000 |  40000 |00:00:00.02 |     175 |  1966K|   666K| 1747K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     175 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:48:10, written at 20:52:13
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            243.07      243.04             1      243.07000      243.04000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  243.32      243.29             8       30.41500       30.41125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:48:10, written at 20:52:13
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      243.34      243.31             1      243.33500      243.31000
------- ---------- ---------- ------------ ------------- -------------
Total        243.34      243.31             2      121.66750      121.65500
------- ---------- ---------- ------------ ------------- -------------
407 rows written to MOD_QRY.csv
Summary for W/D = 10/4000 , bench_run_statistics_id = 414

Timer Set: Run_One, Constructed at 22 Nov 2016 20:48:10, written at 20:52:13
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          243.34      243.31             1      243.33500      243.31000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        243.34      243.31             2      121.66750      121.65500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","6294"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  0hf4h0kmpj82n, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","6294"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    406 |00:00:00.30 |     175 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  40000 |    406 |00:00:00.30 |     175 | 31744 | 31744 |28672  (0)|
|   2 |   VIEW                |       |      1 |  40000 |  40000 |00:00:06.57 |     175 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  40000 |  40000 |00:00:06.56 |     175 |  5691K|  2430K| 4633K (0)|
|   4 |     WINDOW SORT       |       |      1 |  40000 |  40000 |00:00:00.02 |     175 |  1966K|   666K| 1747K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     175 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:52:13, written at 20:52:14
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.30        0.29             1        0.29700        0.29000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.55        0.55             8        0.06838        0.06875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:52:13, written at 20:52:14
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.56        0.56             1        0.56300        0.56000
------- ---------- ---------- ------------ ------------- -------------
Total          0.56        0.56             2        0.28150        0.28000
------- ---------- ---------- ------------ ------------- -------------
407 rows written to MOD_QRY_D.csv
Summary for W/D = 10/4000 , bench_run_statistics_id = 415

Timer Set: Run_One, Constructed at 22 Nov 2016 20:52:13, written at 20:52:14
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.56        0.56             1        0.56300        0.56000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.56        0.56             2        0.28150        0.28000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","8817"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  9gpxxhf686vmr, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","8817"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    406 |00:00:00.02 |     175 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  40000 |    406 |00:00:00.02 |     175 | 36864 | 36864 |32768  (0)|
|   2 |   VIEW                 |       |      1 |  40000 |    406 |00:00:00.02 |     175 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  40000 |    406 |00:00:00.02 |     175 |  1966K|   666K| 1747K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     175 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:52:14, written at 20:52:14
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.02        0.02             1        0.01600        0.02000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.27        0.26             8        0.03313        0.03250
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:52:14, written at 20:52:14
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.28        0.28             1        0.28100        0.28000
------- ---------- ---------- ------------ ------------- -------------
Total          0.28        0.28             2        0.14050        0.14000
------- ---------- ---------- ------------ ------------- -------------
407 rows written to MTH_QRY.csv
Summary for W/D = 10/4000 , bench_run_statistics_id = 416

Timer Set: Run_One, Constructed at 22 Nov 2016 20:52:14, written at 20:52:14
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.28        0.28             1        0.28100        0.28000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.28        0.28             2        0.14050        0.14000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","6385"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  cy67f0fxy493a, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","6385"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    406 |00:01:32.18 |     761K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     41 |    406 |00:01:32.18 |     761K| 31744 | 31744 |28672  (0)|
|   2 |   VIEW                                       |       |      1 |     41 |  40000 |00:01:32.18 |     761K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     41 |  40000 |00:01:32.17 |     761K|  1824K|   650K| 1621K (0)|
|   4 |     VIEW                                     |       |      1 |     41 |  40000 |00:01:55.25 |     761K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  40000 |00:01:55.24 |     761K|  2048 |  2048 | 2692K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     10 |00:00:00.04 |     175 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  40000 |     10 |00:00:00.04 |     175 |  3029K|   773K| 2692K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     175 |       |       |          |
|*  9 |       HASH JOIN                              |       |   4000 |     40 |  39990 |00:01:28.70 |     700K|  1214K|  1214K|  917K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   4000 |        |  40000 |00:00:00.01 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   4000 |  40000 |    160M|00:01:47.97 |     700K|       |       |          |
|  12 |         WINDOW SORT                          |       |   4000 |  40000 |    160M|00:01:25.76 |     700K|  1966K|   666K| 1747K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   4000 |  40000 |    160M|00:00:14.92 |     700K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 20:52:14, written at 20:53:46
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.02             1        0.01500        0.02000
First fetch             92.17       92.17             1       92.16700       92.17000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   92.43       92.44             8       11.55400       11.55500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:52:14, written at 20:53:46
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       92.45       92.44             1       92.44700       92.44000
------- ---------- ---------- ------------ ------------- -------------
Total         92.45       92.44             2       46.22350       46.22000
------- ---------- ---------- ------------ ------------- -------------
407 rows written to RSF_QRY.csv
Summary for W/D = 10/4000 , bench_run_statistics_id = 417

Timer Set: Run_One, Constructed at 22 Nov 2016 20:52:14, written at 20:53:46
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           92.45       92.44             1       92.44700       92.44000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         92.45       92.44             2       46.22350       46.22000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","8052"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  gvtmd6dudkcgg, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","8052"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    406 |00:00:00.31 |     106K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     19 |    406 |00:00:00.31 |     106K| 31744 | 31744 |28672  (0)|
|   2 |   VIEW                                       |              |      1 |     19 |  40000 |00:00:00.31 |     106K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     19 |  40000 |00:00:00.30 |     106K|  1824K|   650K| 1621K (0)|
|   4 |     VIEW                                     |              |      1 |     19 |  40000 |00:00:00.27 |     106K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  40000 |00:00:00.26 |     106K|  2048 |  2048 | 2692K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     10 |     10 |00:00:00.01 |      12 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     10 |     10 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   4000 |      9 |  39990 |00:00:00.13 |   45459 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   4000 |        |  40000 |00:00:00.01 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  40000 |      1 |  39990 |00:00:00.09 |   45459 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  40000 |    363 |  39990 |00:00:00.04 |    5469 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 20:53:46, written at 20:53:47
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.08        0.08             1        0.07800        0.08000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.31        0.31             1        0.31300        0.31000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.64        0.64             8        0.08013        0.08000
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:53:46, written at 20:53:47
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.66        0.65             1        0.65600        0.65000
------- ---------- ---------- ------------ ------------- -------------
Total          0.66        0.65             2        0.32800        0.32500
------- ---------- ---------- ------------ ------------- -------------
407 rows written to RSF_TMP.csv
Summary for W/D = 10/4000 , bench_run_statistics_id = 418

Timer Set: Run_One, Constructed at 22 Nov 2016 20:53:46, written at 20:53:47
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.67        0.67             1        0.67200        0.67000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.67        0.67             2        0.33600        0.33500
------- ---------- ---------- ------------ ------------- -------------
Items truncated
80000 (8000) records (per category) added, average group size (from) = 105.4 (8000), # of groups = 75.9

Timer Set: Setup, Constructed at 22 Nov 2016 20:53:47, written at 20:53:52
==========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   4.36        4.17             1        4.36000        4.17000
Gather_Table_Stats        0.17        0.17             1        0.17100        0.17000
GRP_CNT                   0.03        0.03             1        0.03200        0.03000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     4.56        4.37             4        1.14075        1.09250
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","4753"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  8q3mrfktdgxrb, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","4753"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    808 |00:16:01.82 |     317 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  80000 |    808 |00:16:01.82 |     317 | 64512 | 64512 |57344  (0)|
|   2 |   VIEW                |       |      1 |  80000 |  80000 |00:17:19.69 |     317 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  80000 |  80000 |00:17:19.68 |     317 |    10M|  2393K| 9427K (0)|
|   4 |     WINDOW SORT       |       |      1 |  80000 |  80000 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 20:53:52, written at 21:09:54
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            961.82      961.72             1      961.81800      961.72000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  962.08      961.99             8      120.26038      120.24875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 20:53:52, written at 21:09:54
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      962.10      962.00             1      962.09900      962.00000
------- ---------- ---------- ------------ ------------- -------------
Total        962.10      962.00             2      481.04950      481.00000
------- ---------- ---------- ------------ ------------- -------------
809 rows written to MOD_QRY.csv
Summary for W/D = 10/8000 , bench_run_statistics_id = 419

Timer Set: Run_One, Constructed at 22 Nov 2016 20:53:52, written at 21:09:54
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          962.11      962.02             1      962.11400      962.02000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        962.11      962.02             2      481.05700      481.01000
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","5063"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  crd5nyctragx8, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","5063"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    808 |00:00:00.60 |     317 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  80000 |    808 |00:00:00.60 |     317 | 64512 | 64512 |57344  (0)|
|   2 |   VIEW                |       |      1 |  80000 |  80000 |00:00:13.23 |     317 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  80000 |  80000 |00:00:13.22 |     317 |     9M|  2430K| 7938K (0)|
|   4 |     WINDOW SORT       |       |      1 |  80000 |  80000 |00:00:00.05 |     317 |  3880K|   845K| 3448K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:09:54, written at 21:09:55
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.59        0.59             1        0.59400        0.59000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.84        0.84             8        0.10550        0.10500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:09:54, written at 21:09:55
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.86        0.86             1        0.86000        0.86000
------- ---------- ---------- ------------ ------------- -------------
Total          0.86        0.86             2        0.43000        0.43000
------- ---------- ---------- ------------ ------------- -------------
809 rows written to MOD_QRY_D.csv
Summary for W/D = 10/8000 , bench_run_statistics_id = 420

Timer Set: Run_One, Constructed at 22 Nov 2016 21:09:54, written at 21:09:55
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.88        0.88             1        0.87500        0.88000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.88        0.88             2        0.43750        0.44000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","4828"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  d29ms2dn9rgtm, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","4828"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    808 |00:00:00.04 |     317 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  80000 |    808 |00:00:00.04 |     317 | 66560 | 66560 |59392  (0)|
|   2 |   VIEW                 |       |      1 |  80000 |    808 |00:00:00.05 |     317 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  80000 |    808 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:09:55, written at 21:09:55
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.05        0.05             1        0.04700        0.05000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.30        0.29             8        0.03713        0.03625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:09:55, written at 21:09:55
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.31        0.29             1        0.31300        0.29000
------- ---------- ---------- ------------ ------------- -------------
Total          0.31        0.29             2        0.15650        0.14500
------- ---------- ---------- ------------ ------------- -------------
809 rows written to MTH_QRY.csv
Summary for W/D = 10/8000 , bench_run_statistics_id = 421

Timer Set: Run_One, Constructed at 22 Nov 2016 21:09:55, written at 21:09:55
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.31        0.29             1        0.31300        0.29000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.31        0.29             2        0.15650        0.14500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","3576"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  d4mcycx55uxzx, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","3576"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    808 |00:06:08.73 |    2675K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     81 |    808 |00:06:08.73 |    2675K| 64512 | 64512 |57344  (0)|
|   2 |   VIEW                                       |       |      1 |     81 |  80000 |00:06:08.73 |    2675K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     81 |  80000 |00:06:08.71 |    2675K|  3596K|   822K| 3196K (0)|
|   4 |     VIEW                                     |       |      1 |     81 |  80000 |00:07:42.33 |    2675K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  80000 |00:07:42.31 |    2675K|  2048 |  2048 | 5275K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     10 |00:00:00.08 |     317 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  80000 |     10 |00:00:00.08 |     317 |  6077K|  1001K| 5401K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
|*  9 |       HASH JOIN                              |       |   8000 |     80 |  79990 |00:05:53.90 |    2536K|  1214K|  1214K|  907K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   8000 |        |  80000 |00:00:00.03 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   8000 |  80000 |    640M|00:07:12.12 |    2536K|       |       |          |
|  12 |         WINDOW SORT                          |       |   8000 |  80000 |    640M|00:05:40.94 |    2536K|  3880K|   845K| 3448K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   8000 |  80000 |    640M|00:01:03.35 |    2536K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 21:09:55, written at 21:16:04
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            368.73      368.70             1      368.73100      368.70000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  368.98      368.95             8       46.12263       46.11875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:09:55, written at 21:16:04
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      369.00      368.97             1      368.99700      368.97000
------- ---------- ---------- ------------ ------------- -------------
Total        369.00      368.97             2      184.49850      184.48500
------- ---------- ---------- ------------ ------------- -------------
809 rows written to RSF_QRY.csv
Summary for W/D = 10/8000 , bench_run_statistics_id = 422

Timer Set: Run_One, Constructed at 22 Nov 2016 21:09:55, written at 21:16:04
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          369.00      368.97             1      368.99700      368.97000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        369.00      368.97             2      184.49850      184.48500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","9822"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  cnnm0wa8kvur0, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","9822"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    808 |00:00:00.64 |     229K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     22 |    808 |00:00:00.64 |     229K| 64512 | 64512 |57344  (0)|
|   2 |   VIEW                                       |              |      1 |     22 |  80000 |00:00:00.63 |     229K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     22 |  80000 |00:00:00.62 |     229K|  3596K|   822K| 3196K (0)|
|   4 |     VIEW                                     |              |      1 |     22 |  80000 |00:00:00.56 |     229K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  80000 |00:00:00.55 |     229K|  2048 |  2048 | 5275K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     10 |     10 |00:00:00.01 |      12 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     10 |     10 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   8000 |     12 |  79990 |00:00:00.26 |   90925 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   8000 |        |  80000 |00:00:00.02 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  80000 |      1 |  79990 |00:00:00.19 |   90925 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  80000 |    948 |  79990 |00:00:00.08 |   10935 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 21:16:04, written at 21:16:05
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.13        0.12             1        0.12500        0.12000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              0.63        0.62             1        0.62500        0.62000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.27        0.27             1        0.26600        0.27000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.05        1.05             8        0.13088        0.13125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:16:04, written at 21:16:05
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        1.09        1.06             1        1.09300        1.06000
------- ---------- ---------- ------------ ------------- -------------
Total          1.09        1.06             2        0.54650        0.53000
------- ---------- ---------- ------------ ------------- -------------
809 rows written to RSF_TMP.csv
Summary for W/D = 10/8000 , bench_run_statistics_id = 423

Timer Set: Run_One, Constructed at 22 Nov 2016 21:16:04, written at 21:16:05
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.09        1.06             1        1.09300        1.06000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.09        1.06             2        0.54650        0.53000
------- ---------- ---------- ------------ ------------- -------------
Items truncated
20000 (1000) records (per category) added, average group size (from) = 131.6 (1000), # of groups = 7.6

Timer Set: Setup, Constructed at 22 Nov 2016 21:16:05, written at 21:16:06
==========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   1.17        1.08             1        1.17200        1.08000
Gather_Table_Stats        0.08        0.06             1        0.07800        0.06000
GRP_CNT                   0.00        0.00             1        0.00000        0.00000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     1.25        1.14             4        0.31250        0.28500
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","8607"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  4bxcp23jrtkwp, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","8607"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    209 |00:00:47.25 |      68 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  20000 |    209 |00:00:47.25 |      68 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                |       |      1 |  20000 |  20000 |00:00:15.01 |      68 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  20000 |  20000 |00:00:15.01 |      68 |  2990K|  1197K| 3622K (0)|
|   4 |     WINDOW SORT       |       |      1 |  20000 |  20000 |00:00:00.01 |      68 |  1045K|   546K|  928K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:16:06, written at 21:16:54
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             47.26       47.25             1       47.25700       47.25000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.00             1        0.01500        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   47.51       47.48             8        5.93825        5.93500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:16:06, written at 21:16:54
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       47.52       47.48             1       47.52200       47.48000
------- ---------- ---------- ------------ ------------- -------------
Total         47.52       47.48             2       23.76100       23.74000
------- ---------- ---------- ------------ ------------- -------------
210 rows written to MOD_QRY.csv
Summary for W/D = 20/1000 , bench_run_statistics_id = 424

Timer Set: Run_One, Constructed at 22 Nov 2016 21:16:06, written at 21:16:54
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           47.52       47.48             1       47.52200       47.48000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         47.52       47.48             2       23.76100       23.74000
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","6518"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  b95sanccf7mtt, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","6518"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    209 |00:00:00.15 |      68 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  20000 |    209 |00:00:00.15 |      68 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                |       |      1 |  20000 |  20000 |00:00:00.06 |      68 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  20000 |  20000 |00:00:00.06 |      68 |  2883K|  1215K| 2472K (0)|
|   4 |     WINDOW SORT       |       |      1 |  20000 |  20000 |00:00:00.01 |      68 |  1045K|   546K|  928K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:16:54, written at 21:16:54
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.16        0.15             1        0.15700        0.15000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.39        0.39             8        0.04888        0.04875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:16:54, written at 21:16:54
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.41        0.39             1        0.40700        0.39000
------- ---------- ---------- ------------ ------------- -------------
Total          0.41        0.39             2        0.20350        0.19500
------- ---------- ---------- ------------ ------------- -------------
210 rows written to MOD_QRY_D.csv
Summary for W/D = 20/1000 , bench_run_statistics_id = 425

Timer Set: Run_One, Constructed at 22 Nov 2016 21:16:54, written at 21:16:54
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.42        0.40             1        0.42200        0.40000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.42        0.40             2        0.21100        0.20000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","6028"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  4kcs7ncpmdy5g, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","6028"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    209 |00:00:00.01 |      68 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  20000 |    209 |00:00:00.01 |      68 | 29696 | 29696 |26624  (0)|
|   2 |   VIEW                 |       |      1 |  20000 |    209 |00:00:00.01 |      68 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  20000 |    209 |00:00:00.01 |      68 |  1045K|   546K|  928K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:16:54, written at 21:16:54
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.02        0.01             1        0.01600        0.01000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.27        0.27             8        0.03325        0.03375
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:16:54, written at 21:16:55
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.28        0.27             1        0.28200        0.27000
------- ---------- ---------- ------------ ------------- -------------
Total          0.28        0.27             2        0.14100        0.13500
------- ---------- ---------- ------------ ------------- -------------
210 rows written to MTH_QRY.csv
Summary for W/D = 20/1000 , bench_run_statistics_id = 426

Timer Set: Run_One, Constructed at 22 Nov 2016 21:16:54, written at 21:16:55
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.28        0.27             1        0.28200        0.27000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.28        0.27             2        0.14100        0.13500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","4605"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  a9kr3zh8uynnp, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","4605"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    209 |00:00:11.62 |   91913 |       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     11 |    209 |00:00:11.62 |   91913 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                                       |       |      1 |     11 |  20000 |00:00:11.62 |   91913 |       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     11 |  20000 |00:00:11.62 |   91913 |   974K|   535K|  865K (0)|
|   4 |     VIEW                                     |       |      1 |     11 |  20000 |00:00:20.48 |   91913 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  20000 |00:00:20.48 |   91913 |  2048 |  2048 | 1369K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     20 |00:00:00.02 |      68 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  20000 |     20 |00:00:00.02 |      68 |  1541K|   615K| 1369K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  20000 |  20000 |00:00:00.01 |      68 |       |       |          |
|*  9 |       HASH JOIN                              |       |   1000 |     10 |  19980 |00:00:11.33 |   68000 |  1214K|  1214K| 1112K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   1000 |        |  20000 |00:00:00.01 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   1000 |  20000 |     20M|00:00:13.45 |   68000 |       |       |          |
|  12 |         WINDOW SORT                          |       |   1000 |  20000 |     20M|00:00:10.68 |   68000 |  1045K|   546K|  928K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   1000 |  20000 |     20M|00:00:01.88 |   68000 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 21:16:55, written at 21:17:06
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             11.63       11.63             1       11.62600       11.63000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.24        0.23             1        0.23500        0.23000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   11.86       11.86             8        1.48263        1.48250
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:16:55, written at 21:17:06
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       11.88       11.88             1       11.87600       11.88000
------- ---------- ---------- ------------ ------------- -------------
Total         11.88       11.88             2        5.93800        5.94000
------- ---------- ---------- ------------ ------------- -------------
210 rows written to RSF_QRY.csv
Summary for W/D = 20/1000 , bench_run_statistics_id = 427

Timer Set: Run_One, Constructed at 22 Nov 2016 21:16:55, written at 21:17:06
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           11.88       11.88             1       11.87600       11.88000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         11.88       11.88             2        5.93800        5.94000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","9441"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  5ry01680bjj8n, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","9441"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    209 |00:00:00.14 |   45554 |       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     40 |    209 |00:00:00.14 |   45554 | 15360 | 15360 |14336  (0)|
|   2 |   VIEW                                       |              |      1 |     40 |  20000 |00:00:00.14 |   45554 |       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     40 |  20000 |00:00:00.13 |   45554 |   974K|   535K|  865K (0)|
|   4 |     VIEW                                     |              |      1 |     40 |  20000 |00:00:00.13 |   45554 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  20000 |00:00:00.13 |   45554 |  2048 |  2048 | 1369K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     20 |     20 |00:00:00.01 |      22 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     20 |     20 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   1000 |     20 |  19980 |00:00:00.06 |   21687 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   1000 |        |  20000 |00:00:00.01 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  20000 |      1 |  19980 |00:00:00.05 |   21687 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  20000 |    200 |  19980 |00:00:00.02 |    1707 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 21:17:06, written at 21:17:07
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.05        0.05             1        0.04700        0.05000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.14        0.14             1        0.14100        0.14000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.44        0.44             8        0.05475        0.05500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:17:06, written at 21:17:07
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.50        0.46             1        0.50000        0.46000
------- ---------- ---------- ------------ ------------- -------------
Total          0.50        0.46             2        0.25000        0.23000
------- ---------- ---------- ------------ ------------- -------------
210 rows written to RSF_TMP.csv
Summary for W/D = 20/1000 , bench_run_statistics_id = 428

Timer Set: Run_One, Constructed at 22 Nov 2016 21:17:06, written at 21:17:07
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.50        0.46             1        0.50000        0.46000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.50        0.46             2        0.25000        0.23000
------- ---------- ---------- ------------ ------------- -------------
Items truncated
40000 (2000) records (per category) added, average group size (from) = 121.2 (2000), # of groups = 16.5

Timer Set: Setup, Constructed at 22 Nov 2016 21:17:07, written at 21:17:09
==========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   2.14        2.13             1        2.14000        2.13000
Gather_Table_Stats        0.09        0.09             1        0.09400        0.09000
GRP_CNT                   0.02        0.02             1        0.01600        0.02000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     2.25        2.24             4        0.56250        0.56000
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","1323"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  16wxjdc8g657n, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","1323"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    411 |00:00:43.83 |     191 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  40000 |    411 |00:00:43.83 |     191 | 31744 | 31744 |28672  (0)|
|   2 |   VIEW                |       |      1 |  40000 |  40000 |00:02:09.97 |     191 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  40000 |  40000 |00:02:09.96 |     191 |  5905K|  2393K| 5813K (0)|
|   4 |     WINDOW SORT       |       |      1 |  40000 |  40000 |00:00:00.02 |     191 |  2037K|   674K| 1810K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:17:09, written at 21:20:19
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            189.50      189.50             1      189.50400      189.50000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.24        0.24             1        0.23500        0.24000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  189.74      189.74             8       23.71738       23.71750
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:17:09, written at 21:20:19
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      189.75      189.75             1      189.75400      189.75000
------- ---------- ---------- ------------ ------------- -------------
Total        189.75      189.75             2       94.87700       94.87500
------- ---------- ---------- ------------ ------------- -------------
412 rows written to MOD_QRY.csv
Summary for W/D = 20/2000 , bench_run_statistics_id = 429

Timer Set: Run_One, Constructed at 22 Nov 2016 21:17:09, written at 21:20:19
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          189.75      189.75             1      189.75400      189.75000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        189.75      189.75             2       94.87700       94.87500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","2053"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  0t5a2v1yy5v0q, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","2053"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    411 |00:00:00.30 |     191 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  40000 |    411 |00:00:00.30 |     191 | 31744 | 31744 |28672  (0)|
|   2 |   VIEW                |       |      1 |  40000 |  40000 |00:00:01.72 |     191 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  40000 |  40000 |00:00:01.71 |     191 |  5691K|  2430K| 4326K (0)|
|   4 |     WINDOW SORT       |       |      1 |  40000 |  40000 |00:00:00.02 |     191 |  2037K|   674K| 1810K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:20:19, written at 21:20:19
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.30        0.30             1        0.29800        0.30000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.22        0.22             1        0.21800        0.22000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.53        0.53             8        0.06638        0.06625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:20:19, written at 21:20:20
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.55        0.55             1        0.54700        0.55000
------- ---------- ---------- ------------ ------------- -------------
Total          0.55        0.55             2        0.27350        0.27500
------- ---------- ---------- ------------ ------------- -------------
412 rows written to MOD_QRY_D.csv
Summary for W/D = 20/2000 , bench_run_statistics_id = 430

Timer Set: Run_One, Constructed at 22 Nov 2016 21:20:19, written at 21:20:20
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.56        0.56             1        0.56300        0.56000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.56        0.56             2        0.28150        0.28000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","6319"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  1fqt0thpqm2n3, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","6319"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    411 |00:00:00.02 |     191 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  40000 |    411 |00:00:00.02 |     191 | 40960 | 40960 |36864  (0)|
|   2 |   VIEW                 |       |      1 |  40000 |    411 |00:00:00.02 |     191 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  40000 |    411 |00:00:00.02 |     191 |  2037K|   674K| 1810K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:20:20, written at 21:20:20
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.02        0.01             1        0.01600        0.01000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.27        0.27             8        0.03313        0.03375
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:20:20, written at 21:20:20
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.28        0.27             1        0.28100        0.27000
------- ---------- ---------- ------------ ------------- -------------
Total          0.28        0.27             2        0.14050        0.13500
------- ---------- ---------- ------------ ------------- -------------
412 rows written to MTH_QRY.csv
Summary for W/D = 20/2000 , bench_run_statistics_id = 431

Timer Set: Run_One, Constructed at 22 Nov 2016 21:20:20, written at 21:20:20
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.28        0.27             1        0.28100        0.27000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.28        0.27             2        0.14050        0.13500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","817"' FROM final_grouping GROUP BY cat, final_grp ORDER BY c
at, final_grp

SQL_ID  g15k7qzf897dz, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","817"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    411 |00:00:45.97 |     443K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     21 |    411 |00:00:45.97 |     443K| 31744 | 31744 |28672  (0)|
|   2 |   VIEW                                       |       |      1 |     21 |  40000 |00:00:45.97 |     443K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     21 |  40000 |00:00:45.96 |     443K|  1895K|   658K| 1684K (0)|
|   4 |     VIEW                                     |       |      1 |     21 |  40000 |00:01:21.19 |     443K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  40000 |00:01:21.18 |     443K|  2048 |  2048 | 2692K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     20 |00:00:00.04 |     191 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  40000 |     20 |00:00:00.04 |     191 |  3029K|   773K| 2692K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
|*  9 |       HASH JOIN                              |       |   2000 |     20 |  39980 |00:00:45.00 |     382K|  1214K|  1214K| 1149K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   2000 |        |  40000 |00:00:00.01 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   2000 |  40000 |     80M|00:00:53.80 |     382K|       |       |          |
|  12 |         WINDOW SORT                          |       |   2000 |  40000 |     80M|00:00:42.76 |     382K|  2037K|   674K| 1810K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   2000 |  40000 |     80M|00:00:07.33 |     382K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 21:20:20, written at 21:21:06
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             45.97       45.99             1       45.97300       45.99000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   46.22       46.24             8        5.77788        5.78000
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:20:20, written at 21:21:06
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       46.22       46.24             1       46.22300       46.24000
------- ---------- ---------- ------------ ------------- -------------
Total         46.22       46.24             2       23.11150       23.12000
------- ---------- ---------- ------------ ------------- -------------
412 rows written to RSF_QRY.csv
Summary for W/D = 20/2000 , bench_run_statistics_id = 432

Timer Set: Run_One, Constructed at 22 Nov 2016 21:20:20, written at 21:21:06
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           46.24       46.25             1       46.23900       46.25000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         46.24       46.25             2       23.11950       23.12500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","678"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  5r497hrv1bnmk, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","678"' FROM final_grouping GROUP BY cat, final_grp ORDER
BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    411 |00:00:00.29 |     104K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     38 |    411 |00:00:00.29 |     104K| 31744 | 31744 |28672  (0)|
|   2 |   VIEW                                       |              |      1 |     38 |  40000 |00:00:00.29 |     104K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     38 |  40000 |00:00:00.29 |     104K|  1895K|   658K| 1684K (0)|
|   4 |     VIEW                                     |              |      1 |     38 |  40000 |00:00:00.28 |     104K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  40000 |00:00:00.27 |     104K|  2048 |  2048 | 2692K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     20 |     20 |00:00:00.01 |      22 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     20 |     20 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   2000 |     18 |  39980 |00:00:00.13 |   43419 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   2000 |        |  40000 |00:00:00.01 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  40000 |      1 |  39980 |00:00:00.09 |   43419 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  40000 |    351 |  39980 |00:00:00.04 |    3439 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 21:21:06, written at 21:21:07
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.06        0.06             1        0.06200        0.06000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              0.30        0.29             1        0.29700        0.29000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.64        0.64             8        0.08013        0.08000
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:21:06, written at 21:21:07
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.64        0.64             1        0.64100        0.64000
------- ---------- ---------- ------------ ------------- -------------
Total          0.64        0.64             2        0.32050        0.32000
------- ---------- ---------- ------------ ------------- -------------
412 rows written to RSF_TMP.csv
Summary for W/D = 20/2000 , bench_run_statistics_id = 433

Timer Set: Run_One, Constructed at 22 Nov 2016 21:21:06, written at 21:21:07
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.66        0.66             1        0.65600        0.66000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.66        0.66             2        0.32800        0.33000
------- ---------- ---------- ------------ ------------- -------------
Items truncated
80000 (4000) records (per category) added, average group size (from) = 113.2 (4000), # of groups = 35.35

Timer Set: Setup, Constructed at 22 Nov 2016 21:21:07, written at 21:21:13
==========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   5.80        4.25             1        5.79800        4.25000
Gather_Table_Stats        0.16        0.16             1        0.15600        0.16000
GRP_CNT                   0.03        0.03             1        0.03100        0.03000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     5.99        4.44             4        1.49625        1.11000
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","5061"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  0hx4f962n9g70, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","5061"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    815 |00:12:41.97 |     317 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  80000 |    815 |00:12:41.97 |     317 | 64512 | 64512 |57344  (0)|
|   2 |   VIEW                |       |      1 |  80000 |  80000 |00:06:25.46 |     317 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  80000 |  80000 |00:06:25.45 |     317 |    10M|  2393K| 9929K (0)|
|   4 |     WINDOW SORT       |       |      1 |  80000 |  80000 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:21:13, written at 21:33:55
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch            761.97      761.89             1      761.96700      761.89000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.24        0.23             1        0.23500        0.23000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  762.22      762.14             8       95.27725       95.26750
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:21:13, written at 21:33:55
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      762.23      762.16             1      762.23300      762.16000
------- ---------- ---------- ------------ ------------- -------------
Total        762.23      762.16             2      381.11650      381.08000
------- ---------- ---------- ------------ ------------- -------------
816 rows written to MOD_QRY.csv
Summary for W/D = 20/4000 , bench_run_statistics_id = 434

Timer Set: Run_One, Constructed at 22 Nov 2016 21:21:13, written at 21:33:55
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          762.25      762.17             1      762.24900      762.17000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        762.25      762.17             2      381.12450      381.08500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","3377"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  0hmy4jzyus8am, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","3377"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    815 |00:00:00.60 |     317 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  80000 |    815 |00:00:00.60 |     317 | 64512 | 64512 |57344  (0)|
|   2 |   VIEW                |       |      1 |  80000 |  80000 |00:00:03.49 |     317 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  80000 |  80000 |00:00:03.48 |     317 |     9M|  2430K| 7934K (0)|
|   4 |     WINDOW SORT       |       |      1 |  80000 |  80000 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:33:55, written at 21:33:56
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.59        0.59             1        0.59400        0.59000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.24             1        0.23400        0.24000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.84        0.85             8        0.10550        0.10625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:33:55, written at 21:33:56
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.86        0.86             1        0.85900        0.86000
------- ---------- ---------- ------------ ------------- -------------
Total          0.86        0.86             2        0.42950        0.43000
------- ---------- ---------- ------------ ------------- -------------
816 rows written to MOD_QRY_D.csv
Summary for W/D = 20/4000 , bench_run_statistics_id = 435

Timer Set: Run_One, Constructed at 22 Nov 2016 21:33:55, written at 21:33:56
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.86        0.86             1        0.85900        0.86000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.86        0.86             2        0.42950        0.43000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","160"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LAS
T (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  0zbz2m6f8nw1j, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","160"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    815 |00:00:00.04 |     317 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  80000 |    815 |00:00:00.04 |     317 | 66560 | 66560 |59392  (0)|
|   2 |   VIEW                 |       |      1 |  80000 |    815 |00:00:00.04 |     317 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  80000 |    815 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:33:56, written at 21:33:56
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.03        0.04             1        0.03100        0.04000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.28        0.28             8        0.03513        0.03500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:33:56, written at 21:33:56
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.33        0.30             1        0.32800        0.30000
------- ---------- ---------- ------------ ------------- -------------
Total          0.33        0.30             2        0.16400        0.15000
------- ---------- ---------- ------------ ------------- -------------
816 rows written to MTH_QRY.csv
Summary for W/D = 20/4000 , bench_run_statistics_id = 436

Timer Set: Run_One, Constructed at 22 Nov 2016 21:33:56, written at 21:33:56
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.33        0.30             1        0.32800        0.30000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.33        0.30             2        0.16400        0.15000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","8422"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  6fvtbjwzfpj1p, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","8422"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    815 |00:03:04.83 |    1407K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     41 |    815 |00:03:04.83 |    1407K| 64512 | 64512 |57344  (0)|
|   2 |   VIEW                                       |       |      1 |     41 |  80000 |00:03:04.83 |    1407K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     41 |  80000 |00:03:04.82 |    1407K|  3667K|   828K| 3259K (0)|
|   4 |     VIEW                                     |       |      1 |     41 |  80000 |00:05:26.75 |    1407K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  80000 |00:05:26.73 |    1407K|  2048 |  2048 | 5338K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     20 |00:00:00.08 |     317 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  80000 |     20 |00:00:00.08 |     317 |  6077K|  1001K| 5401K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
|*  9 |       HASH JOIN                              |       |   4000 |     40 |  79980 |00:03:01.04 |    1268K|  1214K|  1214K| 1150K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   4000 |        |  80000 |00:00:00.02 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   4000 |  80000 |    320M|00:03:36.41 |    1268K|       |       |          |
|  12 |         WINDOW SORT                          |       |   4000 |  80000 |    320M|00:02:52.25 |    1268K|  3880K|   845K| 3448K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   4000 |  80000 |    320M|00:00:28.25 |    1268K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 21:33:56, written at 21:37:01
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            184.83      184.83             1      184.83200      184.83000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.23        0.23             1        0.23400        0.23000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  185.08      185.08             8       23.13525       23.13500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:33:56, written at 21:37:01
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      185.10      185.10             1      185.09800      185.10000
------- ---------- ---------- ------------ ------------- -------------
Total        185.10      185.10             2       92.54900       92.55000
------- ---------- ---------- ------------ ------------- -------------
816 rows written to RSF_QRY.csv
Summary for W/D = 20/4000 , bench_run_statistics_id = 437

Timer Set: Run_One, Constructed at 22 Nov 2016 21:33:56, written at 21:37:01
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          185.11      185.11             1      185.11300      185.11000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        185.11      185.11             2       92.55650       92.55500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","5419"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  5mjfc53d2x2xv, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","5419"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    815 |00:00:00.62 |     225K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     40 |    815 |00:00:00.62 |     225K| 64512 | 64512 |57344  (0)|
|   2 |   VIEW                                       |              |      1 |     40 |  80000 |00:00:00.62 |     225K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     40 |  80000 |00:00:00.60 |     225K|  3667K|   828K| 3259K (0)|
|   4 |     VIEW                                     |              |      1 |     40 |  80000 |00:00:00.58 |     225K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  80000 |00:00:00.56 |     225K|  2048 |  2048 | 5338K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     20 |     20 |00:00:00.01 |      22 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     20 |     20 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   4000 |     20 |  79980 |00:00:00.26 |   86855 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   4000 |        |  80000 |00:00:00.02 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  80000 |      1 |  79980 |00:00:00.19 |   86855 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  80000 |    729 |  79980 |00:00:00.08 |    6875 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 21:37:01, written at 21:37:02
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.13        0.12             1        0.12500        0.12000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              0.63        0.62             1        0.62500        0.62000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.25        0.25             1        0.25000        0.25000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.03        1.03             8        0.12900        0.12875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:37:01, written at 21:37:02
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        1.05        1.03             1        1.04700        1.03000
------- ---------- ---------- ------------ ------------- -------------
Total          1.05        1.03             2        0.52350        0.51500
------- ---------- ---------- ------------ ------------- -------------
816 rows written to RSF_TMP.csv
Summary for W/D = 20/4000 , bench_run_statistics_id = 438

Timer Set: Run_One, Constructed at 22 Nov 2016 21:37:01, written at 21:37:02
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.05        1.03             1        1.04700        1.03000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.05        1.03             2        0.52350        0.51500
------- ---------- ---------- ------------ ------------- -------------
Items truncated
160000 (8000) records (per category) added, average group size (from) = 111.8 (8000), # of groups = 71.55

Timer Set: Setup, Constructed at 22 Nov 2016 21:37:02, written at 21:37:13
==========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                  10.10        8.42             1       10.09500        8.42000
Gather_Table_Stats        0.36        0.33             1        0.35900        0.33000
GRP_CNT                   0.06        0.06             1        0.06300        0.06000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                    10.52        8.81             4        2.62925        2.20250
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","2205"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  47v5h9n80mrbv, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","2205"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   1618 |00:34:42.19 |     569 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    160K|   1618 |00:34:42.19 |     569 |   124K|   124K|  110K (0)|
|   2 |   VIEW                |       |      1 |    160K|    160K|00:14:06.82 |     569 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |    160K|    160K|00:14:06.79 |     569 |    20M|  4787K|   18M (0)|
|   4 |     WINDOW SORT       |       |      1 |    160K|    160K|00:00:00.09 |     569 |  7636K|  1095K| 6787K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    160K|    160K|00:00:00.01 |     569 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 21:37:13, written at 22:11:55
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch          2,082.19    2,082.06             1    2,082.19200    2,082.06000
Write to file            0.02        0.01             3        0.00533        0.00333
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.28        0.29             1        0.28100        0.29000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                2,082.50    2,082.38            10      208.25040      208.23800
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 21:37:13, written at 22:11:55
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.02        0.01             2        0.00800        0.00500
(Other)    2,082.50    2,082.38             1    2,082.50400    2,082.38000
------- ---------- ---------- ------------ ------------- -------------
Total      2,082.52    2,082.39             3      694.17333      694.13000
------- ---------- ---------- ------------ ------------- -------------
1619 rows written to MOD_QRY.csv
Summary for W/D = 20/8000 , bench_run_statistics_id = 439

Timer Set: Run_One, Constructed at 22 Nov 2016 21:37:13, written at 22:11:55
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run        2,082.54    2,082.41             1    2,082.53500    2,082.41000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total      2,082.54    2,082.41             2    1,041.26750    1,041.20500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","728"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  0523ckj4rxzfz, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","728"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   1618 |00:00:01.21 |     569 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    160K|   1618 |00:00:01.21 |     569 |   124K|   124K|  110K (0)|
|   2 |   VIEW                |       |      1 |    160K|    160K|00:00:19.82 |     569 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |    160K|    160K|00:00:19.80 |     569 |    18M|  2430K|   15M (0)|
|   4 |     WINDOW SORT       |       |      1 |    160K|    160K|00:00:00.09 |     569 |  7636K|  1095K| 6787K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    160K|    160K|00:00:00.01 |     569 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:11:55, written at 22:11:57
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              1.20        1.21             1        1.20300        1.21000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.28        0.28             1        0.28200        0.28000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.50        1.50            10        0.15010        0.15000
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:11:55, written at 22:11:57
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        1.52        1.50             1        1.51600        1.50000
------- ---------- ---------- ------------ ------------- -------------
Total          1.52        1.50             3        0.50533        0.50000
------- ---------- ---------- ------------ ------------- -------------
1619 rows written to MOD_QRY_D.csv
Summary for W/D = 20/8000 , bench_run_statistics_id = 440

Timer Set: Run_One, Constructed at 22 Nov 2016 22:11:55, written at 22:11:57
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.52        1.50             1        1.51600        1.50000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.52        1.50             2        0.75800        0.75000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","1576"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  bsf5kqrw5gf4t, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","1576"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |   1618 |00:00:00.09 |     569 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |    160K|   1618 |00:00:00.09 |     569 |   133K|   133K|  118K (0)|
|   2 |   VIEW                 |       |      1 |    160K|   1618 |00:00:00.08 |     569 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |    160K|   1618 |00:00:00.08 |     569 |  7636K|  1095K| 6787K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |    160K|    160K|00:00:00.01 |     569 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:11:57, written at 22:11:57
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.09        0.10             1        0.09400        0.10000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.30        0.26             1        0.29600        0.26000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.39        0.36            10        0.03900        0.03600
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:11:57, written at 22:11:57
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        0.39        0.36             1        0.39000        0.36000
------- ---------- ---------- ------------ ------------- -------------
Total          0.39        0.36             3        0.13000        0.12000
------- ---------- ---------- ------------ ------------- -------------
1619 rows written to MTH_QRY.csv
Summary for W/D = 20/8000 , bench_run_statistics_id = 441

Timer Set: Run_One, Constructed at 22 Nov 2016 22:11:57, written at 22:11:57
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.41        0.38             1        0.40600        0.38000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.41        0.38             2        0.20300        0.19000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","407"' FROM final_grouping GROUP BY cat, final_grp ORDER BY c
at, final_grp

SQL_ID  50705jnaz6wnw, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","407"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |   1618 |00:12:29.80 |    4851K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     81 |   1618 |00:12:29.80 |    4851K|   124K|   124K|  110K (0)|
|   2 |   VIEW                                       |       |      1 |     81 |    160K|00:12:29.79 |    4851K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     81 |    160K|00:12:29.77 |    4851K|  7211K|  1071K| 6409K (0)|
|   4 |     VIEW                                     |       |      1 |     81 |    160K|00:22:12.43 |    4851K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |    160K|00:22:12.39 |    4851K|  2048 |  2048 |   10M (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     20 |00:00:00.16 |     569 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |    160K|     20 |00:00:00.16 |     569 |    11M|  1321K|   10M (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |    160K|    160K|00:00:00.02 |     569 |       |       |          |
|*  9 |       HASH JOIN                              |       |   8000 |     80 |    159K|00:12:14.36 |    4552K|  1214K|  1214K| 1133K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   8000 |        |    160K|00:00:00.06 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   8000 |    160K|   1280M|00:14:47.56 |    4552K|       |       |          |
|  12 |         WINDOW SORT                          |       |   8000 |    160K|   1280M|00:11:49.39 |    4552K|  7636K|  1095K| 6787K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   8000 |    160K|   1280M|00:02:02.85 |    4552K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 22:11:57, written at 22:24:27
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            749.81      749.77             1      749.80600      749.77000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.28        0.28             1        0.28100        0.28000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  750.10      750.06            10       75.01030       75.00600
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:11:57, written at 22:24:27
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)      750.12      750.08             1      750.11900      750.08000
------- ---------- ---------- ------------ ------------- -------------
Total        750.12      750.08             3      250.03967      250.02667
------- ---------- ---------- ------------ ------------- -------------
1619 rows written to RSF_QRY.csv
Summary for W/D = 20/8000 , bench_run_statistics_id = 442

Timer Set: Run_One, Constructed at 22 Nov 2016 22:11:57, written at 22:24:27
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          750.14      750.09             1      750.13500      750.09000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        750.14      750.09             2      375.06750      375.04500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","207"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  cjawkzdjn6r07, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","207"' FROM final_grouping GROUP BY cat, final_grp ORDER
BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |   1618 |00:00:01.27 |     475K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     40 |   1618 |00:00:01.27 |     475K|   124K|   124K|  110K (0)|
|   2 |   VIEW                                       |              |      1 |     40 |    160K|00:00:01.27 |     475K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     40 |    160K|00:00:01.24 |     475K|  7211K|  1071K| 6409K (0)|
|   4 |     VIEW                                     |              |      1 |     40 |    160K|00:00:01.24 |     475K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |    160K|00:00:01.21 |     475K|  2048 |  2048 |   10M (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     20 |     20 |00:00:00.01 |      23 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     20 |     20 |00:00:00.01 |       3 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   8000 |     20 |    159K|00:00:00.55 |     177K|       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   8000 |        |    160K|00:00:00.03 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |    160K|      1 |    159K|00:00:00.38 |     177K|       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |    160K|   1592 |    159K|00:00:00.16 |   17111 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 22:24:27, written at 22:24:29
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.27        0.26             1        0.26600        0.26000
Open cursor              0.02        0.02             1        0.01500        0.02000
First fetch              1.27        1.26             1        1.26600        1.26000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.30        0.30             1        0.29700        0.30000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.86        1.86            10        0.18590        0.18600
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:24:27, written at 22:24:29
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        1.88        1.88             1        1.87500        1.88000
------- ---------- ---------- ------------ ------------- -------------
Total          1.88        1.88             3        0.62500        0.62667
------- ---------- ---------- ------------ ------------- -------------
1619 rows written to RSF_TMP.csv
Summary for W/D = 20/8000 , bench_run_statistics_id = 443

Timer Set: Run_One, Constructed at 22 Nov 2016 22:24:27, written at 22:24:29
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.88        1.88             1        1.87500        1.88000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.88        1.88             2        0.93750        0.94000
------- ---------- ---------- ------------ ------------- -------------
Items truncated
40000 (1000) records (per category) added, average group size (from) = 155.6 (1000), # of groups = 6.425

Timer Set: Setup, Constructed at 22 Nov 2016 22:24:29, written at 22:24:32
==========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   2.36        2.14             1        2.36000        2.14000
Gather_Table_Stats        0.14        0.14             1        0.14100        0.14000
GRP_CNT                   0.02        0.02             1        0.01500        0.02000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     2.52        2.30             4        0.62900        0.57500
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","2654"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  chcj3ma3q1kz9, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","2654"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    422 |00:01:39.13 |     191 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  40000 |    422 |00:01:39.13 |     191 | 31744 | 31744 |28672  (0)|
|   2 |   VIEW                |       |      1 |  40000 |  40000 |00:10:50.23 |     191 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  40000 |  40000 |00:10:50.22 |     191 |  5905K|  2393K| 5190K (0)|
|   4 |     WINDOW SORT       |       |      1 |  40000 |  40000 |00:00:00.02 |     191 |  2037K|   674K| 1810K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:24:32, written at 22:26:11
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch             99.12       99.12             1       99.12400       99.12000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.30        0.30             1        0.29700        0.30000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   99.44       99.44             8       12.42963       12.43000
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:24:32, written at 22:26:11
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       99.45       99.46             1       99.45200       99.46000
------- ---------- ---------- ------------ ------------- -------------
Total         99.45       99.46             2       49.72600       49.73000
------- ---------- ---------- ------------ ------------- -------------
423 rows written to MOD_QRY.csv
Summary for W/D = 40/1000 , bench_run_statistics_id = 444

Timer Set: Run_One, Constructed at 22 Nov 2016 22:24:32, written at 22:26:11
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           99.45       99.46             1       99.45200       99.46000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         99.45       99.46             2       49.72600       49.73000
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","5021"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  bpgtpaaj7amwr, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","5021"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    422 |00:00:00.30 |     191 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  40000 |    422 |00:00:00.30 |     191 | 31744 | 31744 |28672  (0)|
|   2 |   VIEW                |       |      1 |  40000 |  40000 |00:00:01.71 |     191 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  40000 |  40000 |00:00:01.70 |     191 |  5767K|  2430K| 4466K (0)|
|   4 |     WINDOW SORT       |       |      1 |  40000 |  40000 |00:00:00.02 |     191 |  2037K|   674K| 1810K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:26:11, written at 22:26:12
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              0.30        0.30             1        0.29600        0.30000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.27        0.26             1        0.26600        0.26000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.58        0.58             8        0.07225        0.07250
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:26:11, written at 22:26:12
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.59        0.60             1        0.59400        0.60000
------- ---------- ---------- ------------ ------------- -------------
Total          0.59        0.60             2        0.29700        0.30000
------- ---------- ---------- ------------ ------------- -------------
423 rows written to MOD_QRY_D.csv
Summary for W/D = 40/1000 , bench_run_statistics_id = 445

Timer Set: Run_One, Constructed at 22 Nov 2016 22:26:11, written at 22:26:12
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.61        0.61             1        0.60900        0.61000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.61        0.61             2        0.30450        0.30500
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","7555"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  g1axa6y27m3y3, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","7555"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    422 |00:00:00.02 |     191 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  40000 |    422 |00:00:00.02 |     191 | 48128 | 48128 |43008  (0)|
|   2 |   VIEW                 |       |      1 |  40000 |    422 |00:00:00.03 |     191 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  40000 |    422 |00:00:00.03 |     191 |  2037K|   674K| 1810K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:26:12, written at 22:26:12
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.02        0.01             1        0.01600        0.01000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.28        0.28             1        0.28100        0.28000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.31        0.31             8        0.03913        0.03875
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:26:12, written at 22:26:12
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.33        0.31             1        0.32900        0.31000
------- ---------- ---------- ------------ ------------- -------------
Total          0.33        0.31             2        0.16450        0.15500
------- ---------- ---------- ------------ ------------- -------------
423 rows written to MTH_QRY.csv
Summary for W/D = 40/1000 , bench_run_statistics_id = 446

Timer Set: Run_One, Constructed at 22 Nov 2016 22:26:12, written at 22:26:12
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.33        0.31             1        0.32900        0.31000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.33        0.31             2        0.16450        0.15500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","9634"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  3zm1q81tvxpyr, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","9634"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    422 |00:00:23.53 |     252K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     11 |    422 |00:00:23.53 |     252K| 31744 | 31744 |28672  (0)|
|   2 |   VIEW                                       |       |      1 |     11 |  40000 |00:00:23.53 |     252K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     11 |  40000 |00:00:23.52 |     252K|  1895K|   658K| 1684K (0)|
|   4 |     VIEW                                     |       |      1 |     11 |  40000 |00:01:07.27 |     252K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  40000 |00:01:07.26 |     252K|  4096 |  4096 | 2692K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.04 |     191 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  40000 |     40 |00:00:00.04 |     191 |  3029K|   773K| 2692K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  40000 |  40000 |00:00:00.01 |     191 |       |       |          |
|*  9 |       HASH JOIN                              |       |   1000 |     10 |  39960 |00:00:23.13 |     191K|  1214K|  1214K| 1300K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   1000 |        |  40000 |00:00:00.02 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   1000 |  40000 |     40M|00:00:27.67 |     191K|       |       |          |
|  12 |         WINDOW SORT                          |       |   1000 |  40000 |     40M|00:00:22.11 |     191K|  2037K|   674K| 1810K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   1000 |  40000 |     40M|00:00:03.78 |     191K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 22:26:12, written at 22:26:36
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             23.53       23.53             1       23.53300       23.53000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.28        0.28             1        0.28200        0.28000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   23.82       23.81             8        2.97688        2.97625
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:26:12, written at 22:26:36
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       23.83       23.83             1       23.83000       23.83000
------- ---------- ---------- ------------ ------------- -------------
Total         23.83       23.83             2       11.91500       11.91500
------- ---------- ---------- ------------ ------------- -------------
423 rows written to RSF_QRY.csv
Summary for W/D = 40/1000 , bench_run_statistics_id = 447

Timer Set: Run_One, Constructed at 22 Nov 2016 22:26:12, written at 22:26:36
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           23.85       23.84             1       23.84600       23.84000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         23.85       23.84             2       11.92300       11.92000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","9740"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  dqdk5s88swg96, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","9740"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    422 |00:00:00.29 |     103K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     76 |    422 |00:00:00.29 |     103K| 31744 | 31744 |28672  (0)|
|   2 |   VIEW                                       |              |      1 |     76 |  40000 |00:00:00.29 |     103K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     76 |  40000 |00:00:00.28 |     103K|  1895K|   658K| 1684K (0)|
|   4 |     VIEW                                     |              |      1 |     76 |  40000 |00:00:00.27 |     103K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  40000 |00:00:00.27 |     103K|  4096 |  4096 | 2692K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      42 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   1000 |     36 |  39960 |00:00:00.14 |   42363 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   1000 |        |  40000 |00:00:00.01 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  40000 |      1 |  39960 |00:00:00.09 |   42363 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  40000 |    311 |  39960 |00:00:00.04 |    2403 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 22:26:36, written at 22:26:37
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.06        0.06             1        0.06300        0.06000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              0.28        0.28             1        0.28100        0.28000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.30        0.30             1        0.29700        0.30000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.67        0.68             8        0.08400        0.08500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:26:36, written at 22:26:37
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.69        0.69             1        0.68700        0.69000
------- ---------- ---------- ------------ ------------- -------------
Total          0.69        0.69             2        0.34350        0.34500
------- ---------- ---------- ------------ ------------- -------------
423 rows written to RSF_TMP.csv
Summary for W/D = 40/1000 , bench_run_statistics_id = 448

Timer Set: Run_One, Constructed at 22 Nov 2016 22:26:36, written at 22:26:37
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.70        0.71             1        0.70300        0.71000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.70        0.71             2        0.35150        0.35500
------- ---------- ---------- ------------ ------------- -------------
Items truncated
80000 (2000) records (per category) added, average group size (from) = 146.8 (2000), # of groups = 13.625

Timer Set: Setup, Constructed at 22 Nov 2016 22:26:37, written at 22:26:44
==========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                   7.00        4.22             1        7.00000        4.22000
Gather_Table_Stats        0.17        0.17             1        0.17200        0.17000
GRP_CNT                   0.03        0.03             1        0.03200        0.03000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                     7.20        4.42             4        1.80100        1.10500
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","4025"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  9xrr798ddt4tz, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","4025"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    829 |00:02:10.81 |     317 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  80000 |    829 |00:02:10.81 |     317 | 64512 | 64512 |57344  (0)|
|   2 |   VIEW                |       |      1 |  80000 |  80000 |02:29:27.12 |     317 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |  80000 |  80000 |02:29:27.10 |     317 |    10M|  2393K| 9686K (0)|
|   4 |     WINDOW SORT       |       |      1 |  80000 |  80000 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:26:44, written at 22:33:21
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch            396.54      396.54             1      396.53800      396.54000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.28        0.28             1        0.28100        0.28000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  396.82      396.82             8       49.60238       49.60250
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:26:44, written at 22:33:21
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)      396.84      396.83             1      396.83500      396.83000
------- ---------- ---------- ------------ ------------- -------------
Total        396.84      396.83             2      198.41750      198.41500
------- ---------- ---------- ------------ ------------- -------------
830 rows written to MOD_QRY.csv
Summary for W/D = 40/2000 , bench_run_statistics_id = 449

Timer Set: Run_One, Constructed at 22 Nov 2016 22:26:44, written at 22:33:21
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          396.85      396.85             1      396.85100      396.85000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        396.85      396.85             2      198.42550      198.42500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","5614"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  9fr7qrvx1kwkk, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","5614"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |    829 |00:00:00.60 |     317 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |  80000 |    829 |00:00:00.60 |     317 | 64512 | 64512 |57344  (0)|
|   2 |   VIEW                |       |      1 |  80000 |  80000 |00:00:11.63 |     317 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |  80000 |  80000 |00:00:11.61 |     317 |     9M|  2430K| 8432K (0)|
|   4 |     WINDOW SORT       |       |      1 |  80000 |  80000 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:33:21, written at 22:33:22
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.61        0.60             1        0.61000        0.60000
Write to file            0.00        0.01             2        0.00000        0.00500
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.27        0.27             1        0.26500        0.27000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.89        0.89             8        0.11125        0.11125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:33:21, written at 22:33:22
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.91        0.90             1        0.90600        0.90000
------- ---------- ---------- ------------ ------------- -------------
Total          0.91        0.90             2        0.45300        0.45000
------- ---------- ---------- ------------ ------------- -------------
830 rows written to MOD_QRY_D.csv
Summary for W/D = 40/2000 , bench_run_statistics_id = 450

Timer Set: Run_One, Constructed at 22 Nov 2016 22:33:21, written at 22:33:22
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.92        0.92             1        0.92200        0.92000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.92        0.92             2        0.46100        0.46000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","580"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LAS
T (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  g430ufhtu0np6, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","580"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |    829 |00:00:00.04 |     317 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |  80000 |    829 |00:00:00.04 |     317 | 70656 | 70656 |63488  (0)|
|   2 |   VIEW                 |       |      1 |  80000 |    829 |00:00:00.04 |     317 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |  80000 |    829 |00:00:00.04 |     317 |  3880K|   845K| 3448K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:33:22, written at 22:33:22
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.05        0.04             1        0.04700        0.04000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.27        0.27             1        0.26600        0.27000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.33        0.33             8        0.04100        0.04125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:33:22, written at 22:33:22
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        0.34        0.34             1        0.34300        0.34000
------- ---------- ---------- ------------ ------------- -------------
Total          0.34        0.34             2        0.17150        0.17000
------- ---------- ---------- ------------ ------------- -------------
830 rows written to MTH_QRY.csv
Summary for W/D = 40/2000 , bench_run_statistics_id = 451

Timer Set: Run_One, Constructed at 22 Nov 2016 22:33:22, written at 22:33:22
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.36        0.36             1        0.35900        0.36000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.36        0.36             2        0.17950        0.18000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","8497"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  bdd3hwu6wc893, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","8497"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |    829 |00:01:33.66 |     773K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     21 |    829 |00:01:33.66 |     773K| 61440 | 61440 |55296  (0)|
|   2 |   VIEW                                       |       |      1 |     21 |  80000 |00:01:33.66 |     773K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     21 |  80000 |00:01:33.64 |     773K|  3667K|   828K| 3259K (0)|
|   4 |     VIEW                                     |       |      1 |     21 |  80000 |00:04:29.06 |     773K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |  80000 |00:04:29.04 |     773K|  4096 |  4096 | 5338K (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.07 |     317 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |  80000 |     40 |00:00:00.07 |     317 |  6077K|  1001K| 5401K (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |  80000 |  80000 |00:00:00.01 |     317 |       |       |          |
|*  9 |       HASH JOIN                              |       |   2000 |     20 |  79960 |00:01:32.67 |     634K|  1214K|  1214K| 1296K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   2000 |        |  80000 |00:00:00.04 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   2000 |  80000 |    160M|00:01:50.32 |     634K|       |       |          |
|  12 |         WINDOW SORT                          |       |   2000 |  80000 |    160M|00:01:28.25 |     634K|  3880K|   845K| 3448K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   2000 |  80000 |    160M|00:00:15.26 |     634K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 22:33:22, written at 22:34:56
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch             93.65       93.64             1       93.65100       93.64000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.30        0.29             1        0.29700        0.29000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                   93.96       93.95             8       11.74550       11.74375
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:33:22, written at 22:34:56
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)       93.98       93.97             1       93.98000       93.97000
------- ---------- ---------- ------------ ------------- -------------
Total         93.98       93.97             2       46.99000       46.98500
------- ---------- ---------- ------------ ------------- -------------
830 rows written to RSF_QRY.csv
Summary for W/D = 40/2000 , bench_run_statistics_id = 452

Timer Set: Run_One, Constructed at 22 Nov 2016 22:33:22, written at 22:34:56
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run           93.98       93.97             1       93.98000       93.97000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total         93.98       93.97             2       46.99000       46.98500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","6813"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  0pt42v6jyt84g, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","6813"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |    829 |00:00:00.60 |     223K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     92 |    829 |00:00:00.60 |     223K| 61440 | 61440 |55296  (0)|
|   2 |   VIEW                                       |              |      1 |     92 |  80000 |00:00:00.59 |     223K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     92 |  80000 |00:00:00.58 |     223K|  3667K|   828K| 3259K (0)|
|   4 |     VIEW                                     |              |      1 |     92 |  80000 |00:00:00.57 |     223K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |  80000 |00:00:00.56 |     223K|  4096 |  4096 | 5338K (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      42 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       2 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   2000 |     52 |  79960 |00:00:00.28 |   84799 |       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   2000 |        |  80000 |00:00:00.02 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |  80000 |      1 |  79960 |00:00:00.19 |   84799 |       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |  80000 |    968 |  79960 |00:00:00.08 |    4839 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 22:34:56, written at 22:34:57
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.13        0.12             1        0.12500        0.12000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              0.59        0.59             1        0.59300        0.59000
Write to file            0.00        0.00             2        0.00000        0.00000
Remaining fetches        0.00        0.00             1        0.00000        0.00000
Write plan               0.30        0.30             1        0.29700        0.30000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.05        1.05             8        0.13088        0.13125
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:34:56, written at 22:34:57
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             1        0.00000        0.00000
(Other)        1.06        1.07             1        1.06300        1.07000
------- ---------- ---------- ------------ ------------- -------------
Total          1.06        1.07             2        0.53150        0.53500
------- ---------- ---------- ------------ ------------- -------------
830 rows written to RSF_TMP.csv
Summary for W/D = 40/2000 , bench_run_statistics_id = 453

Timer Set: Run_One, Constructed at 22 Nov 2016 22:34:56, written at 22:34:57
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.08        1.08             1        1.07900        1.08000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.08        1.08             2        0.53950        0.54000
------- ---------- ---------- ------------ ------------- -------------
Items truncated
160000 (4000) records (per category) added, average group size (from) = 129.2 (4000), # of groups = 30.95

Timer Set: Setup, Constructed at 22 Nov 2016 22:34:57, written at 22:35:09
==========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                  11.50        8.49             1       11.50100        8.49000
Gather_Table_Stats        0.31        0.31             1        0.31200        0.31000
GRP_CNT                   0.06        0.06             1        0.06300        0.06000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                    11.88        8.86             4        2.96900        2.21500
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","6404"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  1csk09zpvjfkm, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","6404"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   1625 |00:23:09.92 |     569 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    160K|   1625 |00:23:09.92 |     569 |   124K|   124K|  110K (0)|
|   2 |   VIEW                |       |      1 |    160K|    160K|04:20:03.92 |     569 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |    160K|    160K|04:20:03.90 |     569 |    20M|  4787K|   17M (0)|
|   4 |     WINDOW SORT       |       |      1 |    160K|    160K|00:00:00.09 |     569 |  7707K|  1099K| 6850K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    160K|    160K|00:00:00.01 |     569 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:35:09, written at 22:58:19
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch          1,389.92    1,389.86             1    1,389.92100    1,389.86000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.30        0.29             1        0.29700        0.29000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                1,390.22    1,390.15            10      139.02180      139.01500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:35:09, written at 22:58:19
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)    1,390.23    1,390.17             1    1,390.23300    1,390.17000
------- ---------- ---------- ------------ ------------- -------------
Total      1,390.23    1,390.17             3      463.41100      463.39000
------- ---------- ---------- ------------ ------------- -------------
1626 rows written to MOD_QRY.csv
Summary for W/D = 40/4000 , bench_run_statistics_id = 454

Timer Set: Run_One, Constructed at 22 Nov 2016 22:35:09, written at 22:58:19
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run        1,390.23    1,390.17             1    1,390.23300    1,390.17000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total      1,390.23    1,390.17             2      695.11650      695.08500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","8480"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  1cccz41scjnmm, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","8480"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   1625 |00:00:01.20 |     569 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    160K|   1625 |00:00:01.20 |     569 |   124K|   124K|  110K (0)|
|   2 |   VIEW                |       |      1 |    160K|    160K|00:00:16.50 |     569 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |    160K|    160K|00:00:16.47 |     569 |    18M|  2430K|   15M (0)|
|   4 |     WINDOW SORT       |       |      1 |    160K|    160K|00:00:00.09 |     569 |  7707K|  1099K| 6850K (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    160K|    160K|00:00:00.01 |     569 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:58:19, written at 22:58:21
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              1.19        1.19             1        1.18800        1.19000
Write to file            0.02        0.01             3        0.00533        0.00333
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.27        0.27             1        0.26500        0.27000
(Other)                  0.02        0.02             1        0.01500        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.48        1.49            10        0.14840        0.14900
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:58:19, written at 22:58:21
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.02        0.01             2        0.00800        0.00500
(Other)        1.48        1.49             1        1.48400        1.49000
------- ---------- ---------- ------------ ------------- -------------
Total          1.50        1.50             3        0.50000        0.50000
------- ---------- ---------- ------------ ------------- -------------
1626 rows written to MOD_QRY_D.csv
Summary for W/D = 40/4000 , bench_run_statistics_id = 455

Timer Set: Run_One, Constructed at 22 Nov 2016 22:58:19, written at 22:58:21
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.50        1.50             1        1.50000        1.50000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.50        1.50             2        0.75000        0.75000
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","8182"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  258a02qk58aty, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","8182"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |   1625 |00:00:00.09 |     569 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |    160K|   1625 |00:00:00.09 |     569 |   124K|   124K|  110K (0)|
|   2 |   VIEW                 |       |      1 |    160K|   1625 |00:00:00.09 |     569 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |    160K|   1625 |00:00:00.09 |     569 |  7707K|  1099K| 6850K (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |    160K|    160K|00:00:00.01 |     569 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 22:58:21, written at 22:58:21
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.08        0.07             1        0.07800        0.07000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.28        0.29             1        0.28200        0.29000
(Other)                  0.02        0.02             1        0.01600        0.02000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.38        0.38            10        0.03760        0.03800
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:58:21, written at 22:58:21
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        0.39        0.38             1        0.39100        0.38000
------- ---------- ---------- ------------ ------------- -------------
Total          0.39        0.38             3        0.13033        0.12667
------- ---------- ---------- ------------ ------------- -------------
1626 rows written to MTH_QRY.csv
Summary for W/D = 40/4000 , bench_run_statistics_id = 456

Timer Set: Run_One, Constructed at 22 Nov 2016 22:58:21, written at 22:58:21
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.39        0.38             1        0.39100        0.38000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.39        0.38             2        0.19550        0.19000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","9263"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  a9s9a98bvj9uj, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","9263"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |   1625 |00:06:17.27 |    2575K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     41 |   1625 |00:06:17.27 |    2575K|   124K|   124K|  110K (0)|
|   2 |   VIEW                                       |       |      1 |     41 |    160K|00:06:17.27 |    2575K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     41 |    160K|00:06:17.25 |    2575K|  7211K|  1071K| 6409K (0)|
|   4 |     VIEW                                     |       |      1 |     41 |    160K|00:18:11.40 |    2575K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |    160K|00:18:11.35 |    2575K|  4096 |  4096 |   10M (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.16 |     569 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |    160K|     40 |00:00:00.16 |     569 |    11M|  1321K|   10M (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |    160K|    160K|00:00:00.02 |     569 |       |       |          |
|*  9 |       HASH JOIN                              |       |   4000 |     40 |    159K|00:06:14.75 |    2276K|  1214K|  1214K| 1284K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   4000 |        |    160K|00:00:00.11 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   4000 |    160K|    640M|00:07:28.41 |    2276K|       |       |          |
|  12 |         WINDOW SORT                          |       |   4000 |    160K|    640M|00:05:59.99 |    2276K|  7707K|  1099K| 6850K (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   4000 |    160K|    640M|00:00:59.66 |    2276K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 22 Nov 2016 22:58:21, written at 23:04:39
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch            377.26      377.23             1      377.26200      377.23000
Write to file            0.02        0.02             3        0.00500        0.00667
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.28        0.28             1        0.28200        0.28000
(Other)                  0.00        0.00             1        0.00000        0.00000
----------------- ---------- ---------- ------------ ------------- -------------
Total                  377.58      377.55            10       37.75750       37.75500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 22:58:21, written at 23:04:39
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.02        0.02             2        0.00750        0.01000
(Other)      377.58      377.54             1      377.57500      377.54000
------- ---------- ---------- ------------ ------------- -------------
Total        377.59      377.56             3      125.86333      125.85333
------- ---------- ---------- ------------ ------------- -------------
1626 rows written to RSF_QRY.csv
Summary for W/D = 40/4000 , bench_run_statistics_id = 457

Timer Set: Run_One, Constructed at 22 Nov 2016 22:58:21, written at 23:04:39
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run          377.59      377.56             1      377.59000      377.56000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total        377.59      377.56             2      188.79500      188.78000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","1971"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  6w60jnuvzn484, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","1971"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |   1625 |00:00:01.24 |     471K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     94 |   1625 |00:00:01.24 |     471K|   124K|   124K|  110K (0)|
|   2 |   VIEW                                       |              |      1 |     94 |    160K|00:00:01.24 |     471K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     94 |    160K|00:00:01.21 |     471K|  7211K|  1071K| 6409K (0)|
|   4 |     VIEW                                     |              |      1 |     94 |    160K|00:00:01.23 |     471K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |    160K|00:00:01.20 |     471K|  4096 |  4096 |   10M (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      43 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       3 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   4000 |     54 |    159K|00:00:00.57 |     172K|       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   4000 |        |    160K|00:00:00.03 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |    160K|      1 |    159K|00:00:00.38 |     172K|       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |    160K|   1819 |    159K|00:00:00.16 |   13031 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 22 Nov 2016 23:04:39, written at 23:04:41
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.27        0.27             1        0.26500        0.27000
Open cursor              0.02        0.02             1        0.01600        0.02000
First fetch              1.25        1.25             1        1.25000        1.25000
Write to file            0.00        0.00             3        0.00000        0.00000
Remaining fetches        0.00        0.00             2        0.00000        0.00000
Write plan               0.30        0.29             1        0.29700        0.29000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    1.84        1.84            10        0.18440        0.18400
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 23:04:39, written at 23:04:41
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        1.88        1.86             1        1.87500        1.86000
------- ---------- ---------- ------------ ------------- -------------
Total          1.88        1.86             3        0.62500        0.62000
------- ---------- ---------- ------------ ------------- -------------
1626 rows written to RSF_TMP.csv
Summary for W/D = 40/4000 , bench_run_statistics_id = 458

Timer Set: Run_One, Constructed at 22 Nov 2016 23:04:39, written at 23:04:41
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            1.89        1.87             1        1.89100        1.87000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          1.89        1.87             2        0.94550        0.93500
------- ---------- ---------- ------------ ------------- -------------
Items truncated
320000 (8000) records (per category) added, average group size (from) = 124.2 (8000), # of groups = 64.425

Timer Set: Setup, Constructed at 22 Nov 2016 23:04:41, written at 23:05:08
==========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed         CPU         Calls       Ela/Call       CPU/Call
------------------ ---------- ---------- ------------ ------------- -------------
Add_Itm                  26.77       16.96             1       26.76900       16.96000
Gather_Table_Stats        0.63        0.62             1        0.62500        0.62000
GRP_CNT                   0.13        0.13             1        0.12500        0.13000
(Other)                   0.00        0.00             1        0.00000        0.00000
------------------ ---------- ---------- ------------ ------------- -------------
Total                    27.52       17.71             4        6.87975        4.42750
------------------ ---------- ---------- ------------ ------------- -------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURE
S (id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, fin
al_grp[ANY] = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_
grp || '","' || COUNT(*) || '","817"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  2ytj4xgt4v9s4, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","817"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:21:10.69 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:21:10.69 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|55:38:12.41 |    1074 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |    320K|    320K|55:38:12.36 |    1074 |    38M|  4773K|   32M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.19 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 22 Nov 2016 23:05:08, written at 00:37:54
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch          5,565.65    5,565.41             1    5,565.65400    5,565.41000
Write to file            0.00        0.00             5        0.00000        0.00000
Remaining fetches        0.00        0.00             4        0.00000        0.00000
Write plan               0.30        0.30             1        0.29700        0.30000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                5,565.97    5,565.72            14      397.56900      397.55143
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 22 Nov 2016 23:05:08, written at 00:37:54
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             3        0.00000        0.00000
(Other)    5,565.98    5,565.73             1    5,565.98100    5,565.73000
------- ---------- ---------- ------------ ------------- -------------
Total      5,565.98    5,565.73             4    1,391.49525    1,391.43250
------- ---------- ---------- ------------ ------------- -------------
3230 rows written to MOD_QRY.csv
Summary for W/D = 40/8000 , bench_run_statistics_id = 459

Timer Set: Run_One, Constructed at 22 Nov 2016 23:05:08, written at 00:37:54
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run        5,566.00    5,565.75             1    5,565.99700    5,565.75000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total      5,566.00    5,565.75             2    2,782.99850    2,782.87500
------- ---------- ---------- ------------ ------------- -------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight, sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY (Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASU
RES (id, weight, weight sub_weight, id final_grp, seq) RULES ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY] OR
DER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || fin
al_grp || '","' || COUNT(*) || '","9807"' FROM all_rows GROUP BY cat, final_grp ORDER BY cat, final_grp

SQL_ID  2ws6cb6spn1tm, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","9807"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:00:02.43 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:00:02.43 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|00:01:32.47 |    1074 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |    320K|    320K|00:01:32.42 |    1074 |    36M|  4844K|   29M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.18 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 23 Nov 2016 00:37:54, written at 00:37:57
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              2.42        2.42             1        2.42200        2.42000
Write to file            0.02        0.02             5        0.00300        0.00400
Remaining fetches        0.00        0.00             4        0.00000        0.00000
Write plan               0.28        0.28             1        0.28200        0.28000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    2.74        2.73            14        0.19536        0.19500
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 23 Nov 2016 00:37:54, written at 00:37:57
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             4        0.00000        0.00000
(Other)        2.77        2.73             1        2.76600        2.73000
------- ---------- ---------- ------------ ------------- -------------
Total          2.77        2.73             5        0.55320        0.54600
------- ---------- ---------- ------------ ------------- -------------
3230 rows written to MOD_QRY_D.csv
Summary for W/D = 40/8000 , bench_run_statistics_id = 460

Timer Set: Run_One, Constructed at 23 Nov 2016 00:37:54, written at 00:37:57
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            2.78        2.75             1        2.78200        2.75000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          2.78        2.75             2        1.39100        1.37500
------- ---------- ---------- ------------ ------------- -------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || num_rows || '","7787"' FROM items MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL LA
ST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s* t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m ORDER BY cat, final_grp

SQL_ID  3h9qsj5tyf2ah, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","7787"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |   3229 |00:00:00.17 |    1074 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                 |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |    320K|   3229 |00:00:00.17 |    1074 |    15M|  1460K|   13M (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

Timer Set: Cursor, Constructed at 23 Nov 2016 00:37:57, written at 00:37:58
===========================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch              0.17        0.17             1        0.17200        0.17000
Write to file            0.02        0.02             5        0.00320        0.00400
Remaining fetches        0.00        0.00             4        0.00000        0.00000
Write plan               0.27        0.27             1        0.26500        0.27000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    0.47        0.47            14        0.03343        0.03357
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 23 Nov 2016 00:37:57, written at 00:37:58
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.02        0.02             4        0.00400        0.00500
(Other)        0.47        0.46             1        0.46800        0.46000
------- ---------- ---------- ------------ ------------- -------------
Total          0.48        0.48             5        0.09680        0.09600
------- ---------- ---------- ------------ ------------- -------------
3230 rows written to MTH_QRY.csv
Summary for W/D = 40/8000 , bench_run_statistics_id = 461

Timer Set: Run_One, Constructed at 23 Nov 2016 00:37:57, written at 00:37:58
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            0.50        0.50             1        0.50000        0.50000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          0.50        0.50             2        0.25000        0.25000
------- ---------- ---------- ------------ ------------- -------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat
, rn, seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id, itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_
num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, g
rp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","5736"' FROM final_grouping GROUP BY cat, final_grp ORDER BY
cat, final_grp

SQL_ID  94y37zwgjs9ug, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","5736"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |   3229 |00:08:50.80 |    9365K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     81 |   3229 |00:08:50.80 |    9365K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |       |      1 |     81 |    320K|00:08:50.80 |    9365K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     81 |    320K|00:08:50.75 |    9365K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |       |      1 |     81 |    320K|01:13:10.25 |    9365K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |    320K|01:13:10.14 |    9365K|  4096 |  4096 |   20M (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.34 |    1074 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |    320K|     40 |00:00:00.34 |    1074 |    23M|  1772K|   20M (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |    320K|    320K|00:00:00.04 |    1074 |       |       |          |
|*  9 |       HASH JOIN                              |       |   8000 |     80 |    319K|00:25:04.02 |    8592K|  1214K|  1214K| 1283K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   8000 |        |    320K|00:00:00.19 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   8000 |    320K|   2560M|00:30:26.18 |    8592K|       |       |          |
|  12 |         WINDOW SORT                          |       |   8000 |    320K|   2560M|00:24:31.69 |    8592K|    15M|  1460K|   13M (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   8000 |    320K|   2560M|00:04:01.43 |    8592K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Timer Set: Cursor, Constructed at 23 Nov 2016 00:37:58, written at 01:03:11
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.00        0.00             1        0.00000        0.00000
Open cursor              0.00        0.00             1        0.00000        0.00000
First fetch          1,512.88    1,512.85             1    1,512.88000    1,512.85000
Write to file            0.02        0.01             5        0.00320        0.00200
Remaining fetches        0.00        0.00             4        0.00000        0.00000
Write plan               0.30        0.30             1        0.29700        0.30000
(Other)                  0.02        0.01             1        0.01500        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                1,513.21    1,513.17            14      108.08629      108.08357
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 23 Nov 2016 00:37:58, written at 01:03:11
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.02        0.01             4        0.00400        0.00250
(Other)    1,513.21    1,513.16             1    1,513.20700    1,513.16000
------- ---------- ---------- ------------ ------------- -------------
Total      1,513.22    1,513.17             5      302.64460      302.63400
------- ---------- ---------- ------------ ------------- -------------
3230 rows written to RSF_QRY.csv
Summary for W/D = 40/8000 , bench_run_statistics_id = 462

Timer Set: Run_One, Constructed at 23 Nov 2016 00:37:58, written at 01:03:11
============================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run        1,513.22    1,513.17             1    1,513.22300    1,513.17000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total      1,513.22    1,513.17             2      756.61150      756.58500
------- ---------- ---------- ------------ ------------- -------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items
_tmp_n1) */ itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num + CASE WHEN rsq.sub_weight >= 5000 THEN 1 EL
SE 0 END FROM rsq JOIN items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq) final_grp
 FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' || COUNT(*) || '","8326"' FROM final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

INSERT INTO items_tmp SELECT id, cat, seq, weight, Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) FROM items
SQL_ID  3u062brw9anr9, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","8326"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |   3229 |00:00:02.69 |    1118K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     91 |   3229 |00:00:02.69 |    1118K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |              |      1 |     91 |    320K|00:00:02.69 |    1118K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     91 |    320K|00:00:02.63 |    1118K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |              |      1 |     91 |    320K|00:00:02.73 |    1118K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |    320K|00:00:02.65 |    1118K|  4096 |  4096 |   20M (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      43 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       3 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   8000 |     51 |    319K|00:00:01.15 |     346K|       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   8000 |        |    320K|00:00:00.07 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |    320K|      1 |    319K|00:00:00.78 |     346K|       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |    320K|   3191 |    319K|00:00:00.31 |   26059 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Timer Set: Cursor, Constructed at 23 Nov 2016 01:03:11, written at 01:03:15
===========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer                 Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Pre SQL                  0.78        0.57             1        0.78100        0.57000
Open cursor              0.02        0.01             1        0.01600        0.01000
First fetch              2.69        2.69             1        2.68800        2.69000
Write to file            0.00        0.00             5        0.00000        0.00000
Remaining fetches        0.00        0.00             4        0.00000        0.00000
Write plan               0.31        0.31             1        0.31200        0.31000
(Other)                  0.02        0.01             1        0.01600        0.01000
----------------- ---------- ---------- ------------ ------------- -------------
Total                    3.81        3.59            14        0.27236        0.25643
----------------- ---------- ---------- ------------ ------------- -------------

Timer Set: File Writer, Constructed at 23 Nov 2016 01:03:11, written at 01:03:15
================================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             4        0.00000        0.00000
(Other)        3.83        3.59             1        3.82900        3.59000
------- ---------- ---------- ------------ ------------- -------------
Total          3.83        3.59             5        0.76580        0.71800
------- ---------- ---------- ------------ ------------- -------------
3230 rows written to RSF_TMP.csv
Summary for W/D = 40/8000 , bench_run_statistics_id = 463

Timer Set: Run_One, Constructed at 23 Nov 2016 01:03:11, written at 01:03:15
============================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Run            3.83        3.59             1        3.82900        3.59000
(Other)        0.00        0.00             1        0.00000        0.00000
------- ---------- ---------- ------------ ------------- -------------
Total          3.83        3.59             2        1.91450        1.79500
------- ---------- ---------- ------------ ------------- -------------

Distinct Plans
==============
MOD_QRY: 3/4 (1 of 1)
SQL_ID  2ytj4xgt4v9s4, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","817"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:21:10.69 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:21:10.69 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|55:38:12.41 |    1074 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |    320K|    320K|55:38:12.36 |    1074 |    38M|  4773K|   32M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.19 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

MOD_QRY_D: 3/4 (1 of 1)
SQL_ID  2ws6cb6spn1tm, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","9807"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:00:02.43 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:00:02.43 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|00:01:32.47 |    1074 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |    320K|    320K|00:01:32.42 |    1074 |    36M|  4844K|   29M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.18 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

MTH_QRY: 3/4 (1 of 1)
SQL_ID  3h9qsj5tyf2ah, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","7787"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |   3229 |00:00:00.17 |    1074 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                 |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |    320K|   3229 |00:00:00.17 |    1074 |    15M|  1460K|   13M (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

RSF_QRY: 3/4 (1 of 1)
SQL_ID  94y37zwgjs9ug, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","5736"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |   3229 |00:08:50.80 |    9365K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     81 |   3229 |00:08:50.80 |    9365K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |       |      1 |     81 |    320K|00:08:50.80 |    9365K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     81 |    320K|00:08:50.75 |    9365K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |       |      1 |     81 |    320K|01:13:10.25 |    9365K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |    320K|01:13:10.14 |    9365K|  4096 |  4096 |   20M (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.34 |    1074 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |    320K|     40 |00:00:00.34 |    1074 |    23M|  1772K|   20M (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |    320K|    320K|00:00:00.04 |    1074 |       |       |          |
|*  9 |       HASH JOIN                              |       |   8000 |     80 |    319K|00:25:04.02 |    8592K|  1214K|  1214K| 1283K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   8000 |        |    320K|00:00:00.19 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   8000 |    320K|   2560M|00:30:26.18 |    8592K|       |       |          |
|  12 |         WINDOW SORT                          |       |   8000 |    320K|   2560M|00:24:31.69 |    8592K|    15M|  1460K|   13M (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   8000 |    320K|   2560M|00:04:01.43 |    8592K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

RSF_TMP: 3/4 (1 of 1)
SQL_ID  3u062brw9anr9, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","8326"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |   3229 |00:00:02.69 |    1118K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     91 |   3229 |00:00:02.69 |    1118K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |              |      1 |     91 |    320K|00:00:02.69 |    1118K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     91 |    320K|00:00:02.63 |    1118K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |              |      1 |     91 |    320K|00:00:02.73 |    1118K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |    320K|00:00:02.65 |    1118K|  4096 |  4096 |   20M (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      43 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       3 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   8000 |     51 |    319K|00:00:01.15 |     346K|       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   8000 |        |    320K|00:00:00.07 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |    320K|      1 |    319K|00:00:00.78 |     346K|       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |    320K|   3191 |    319K|00:00:00.31 |   26059 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Data Points
===========
Data Point:               size_wide      size_deep       cpu_time        elapsed       num_recs       per_part     group_size
Data Point                       10           1000            .66           .797          10000           1000            116
Data Point                       10           2000           1.14           1.25          20000           2000            112
Data Point                       10           4000           2.24          2.485          40000           4000            108
Data Point                       10           8000           4.37          4.563          80000           8000            105
Data Point                       20           1000           1.16          1.266          20000           1000            132
Data Point                       20           2000           2.25          2.265          40000           2000            121
Data Point                       20           4000           4.45          6.001          80000           4000            113
Data Point                       20           8000           8.81         10.517         160000           8000            112
Data Point                       40           1000           2.31          2.532          40000           1000            156
Data Point                       40           2000           4.43          7.219          80000           2000            147
Data Point                       40           4000           8.88         11.892         160000           4000            129
Data Point                       40           8000          17.71         27.519         320000           8000            124

num_records_out
===============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10            105            205            406            808
MOD_QRY                         W20            209            411            815           1618
MOD_QRY                         W40            422            829           1625           3229
MOD_QRY_D                       W10            105            205            406            808
MOD_QRY_D                       W20            209            411            815           1618
MOD_QRY_D                       W40            422            829           1625           3229
MTH_QRY                         W10            105            205            406            808
MTH_QRY                         W20            209            411            815           1618
MTH_QRY                         W40            422            829           1625           3229
RSF_QRY                         W10            105            205            406            808
RSF_QRY                         W20            209            411            815           1618
RSF_QRY                         W40            422            829           1625           3229
RSF_TMP                         W10            105            205            406            808
RSF_TMP                         W20            209            411            815           1618
RSF_TMP                         W40            422            829           1625           3229

num_records_out_SLICE
=====================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                         422            829           1625           3229
MOD_QRY_D                       422            829           1625           3229
MTH_QRY                         422            829           1625           3229
RSF_QRY                         422            829           1625           3229
RSF_TMP                         422            829           1625           3229

num_records_out_RATIO
=====================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              1              1              1              1
MOD_QRY                         W20              1              1              1              1
MOD_QRY                         W40              1              1              1              1
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1              1
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10              1              1              1              1
RSF_QRY                         W20              1              1              1              1
RSF_QRY                         W40              1              1              1              1
RSF_TMP                         W10              1              1              1              1
RSF_TMP                         W20              1              1              1              1
RSF_TMP                         W40              1              1              1              1

num_records_out_SLICE_RATIO
===========================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1              1              1
MOD_QRY_D                         1              1              1              1
MTH_QRY                           1              1              1              1
RSF_QRY                           1              1              1              1
RSF_TMP                           1              1              1              1

cpu_time
========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10             16           61.5         243.04         961.72
MOD_QRY                         W20          47.25          189.5         761.91        2082.06
MOD_QRY                         W40          99.14         396.54        1389.86        5565.41
MOD_QRY_D                       W10            .08            .16            .29            .59
MOD_QRY_D                       W20            .15             .3            .59           1.21
MOD_QRY_D                       W40            .32             .6           1.19           2.42
MTH_QRY                         W10              0            .01            .02            .05
MTH_QRY                         W20            .01            .01            .04             .1
MTH_QRY                         W40            .01            .04            .07            .17
RSF_QRY                         W10           5.89          23.15          92.19          368.7
RSF_QRY                         W20          11.63          45.99         184.83         749.77
RSF_QRY                         W40          23.53          93.64         377.25        1512.85
RSF_TMP                         W10             .1            .19            .39            .76
RSF_TMP                         W20            .19            .37            .76           1.54
RSF_TMP                         W40            .36            .73           1.54           3.27

cpu_time_SLICE
==============
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       99.14         396.54        1389.86        5565.41
MOD_QRY_D                       .32             .6           1.19           2.42
MTH_QRY                         .01            .04            .07            .17
RSF_QRY                       23.53          93.64         377.25        1512.85
RSF_TMP                         .36            .73           1.54           3.27

cpu_time_RATIO
==============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10       16000000           6150          12152        19234.4
MOD_QRY                         W20           4725          18950       19047.75        20820.6
MOD_QRY                         W40           9914         9913.5       19855.14       32737.71
MOD_QRY_D                       W10          80000             16           14.5           11.8
MOD_QRY_D                       W20             15             30          14.75           12.1
MOD_QRY_D                       W40             32             15             17          14.24
MTH_QRY                         W10              0              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10        5890000           2315         4609.5           7374
RSF_QRY                         W20           1163           4599        4620.75         7497.7
RSF_QRY                         W40           2353           2341        5389.29        8899.12
RSF_TMP                         W10         100000             19           19.5           15.2
RSF_TMP                         W20             19             37             19           15.4
RSF_TMP                         W40             36          18.25             22          19.24

cpu_time_SLICE_RATIO
====================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                        9914         9913.5       19855.14       32737.71
MOD_QRY_D                        32             15             17          14.24
MTH_QRY                           1              1              1              1
RSF_QRY                        2353           2341        5389.29        8899.12
RSF_TMP                          36          18.25             22          19.24

elapsed_time
============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10         16.002         61.506         243.07        961.818
MOD_QRY                         W20         47.257        189.504        761.983       2082.192
MOD_QRY                         W40          99.14        396.538       1389.921       5565.654
MOD_QRY_D                       W10           .078           .156           .297           .594
MOD_QRY_D                       W20           .157           .298           .594          1.203
MOD_QRY_D                       W40           .312            .61          1.188          2.422
MTH_QRY                         W10              0           .016           .016           .047
MTH_QRY                         W20           .016           .016           .031           .094
MTH_QRY                         W40           .016           .047           .078           .172
RSF_QRY                         W10          5.891         23.159         92.182        368.731
RSF_QRY                         W20         11.626         45.973        184.832        749.806
RSF_QRY                         W40         23.533         93.651        377.278        1512.88
RSF_TMP                         W10           .094           .188           .391           .766
RSF_TMP                         W20           .188           .375           .766          1.547
RSF_TMP                         W40            .36           .734          1.531          3.485

elapsed_time_SLICE
==================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       99.14        396.538       1389.921       5565.654
MOD_QRY_D                      .312            .61          1.188          2.422
MTH_QRY                        .016           .047           .078           .172
RSF_QRY                      23.533         93.651        377.278        1512.88
RSF_TMP                         .36           .734          1.531          3.485

elapsed_time_RATIO
==================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10       16002000        3844.13       15191.88       20464.21
MOD_QRY                         W20        2953.56          11844        24580.1       22150.98
MOD_QRY                         W40        6196.25        8436.98        17819.5       32358.45
MOD_QRY_D                       W10          78000           9.75          18.56          12.64
MOD_QRY_D                       W20           9.81          18.63          19.16           12.8
MOD_QRY_D                       W40           19.5          12.98          15.23          14.08
MTH_QRY                         W10              0              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10        5891000        1447.44        5761.38        7845.34
RSF_QRY                         W20         726.63        2873.31        5962.32        7976.66
RSF_QRY                         W40        1470.81        1992.57         4836.9        8795.81
RSF_TMP                         W10          94000          11.75          24.44           16.3
RSF_TMP                         W20          11.75          23.44          24.71          16.46
RSF_TMP                         W40           22.5          15.62          19.63          20.26

elapsed_time_SLICE_RATIO
========================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                     6196.25        8436.98        17819.5       32358.45
MOD_QRY_D                      19.5          12.98          15.23          14.08
MTH_QRY                           1              1              1              1
RSF_QRY                     1470.81        1992.57         4836.9        8795.81
RSF_TMP                        22.5          15.62          19.63          20.26

memory_used
===========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10        1735680        3250176        5727232        9653248
MOD_QRY                         W20        3708928        5952512       10167296       19100672
MOD_QRY                         W40        5314560        9918464       18241536       34169856
MOD_QRY_D                       W10        1371136        2521088        4744192        8128512
MOD_QRY_D                       W20        2531328        4429824        8124416       16745472
MOD_QRY_D                       W40        4573184        8634368       16484352       30871552
MTH_QRY                         W10         563200         950272        1788928        3530752
MTH_QRY                         W20         950272        1853440        3530752        6949888
MTH_QRY                         W40        1853440        3530752        7014400       13981696
RSF_QRY                         W10         913408        1401856        2756608        5530624
RSF_QRY                         W20        1401856        2756608        5530624       11014144
RSF_QRY                         W40        2756608        5530624       11014144       21916672
RSF_TMP                         W10         692224        1401856        2756608        5401600
RSF_TMP                         W20        1401856        2756608        5466112       10885120
RSF_TMP                         W40        2756608        5466112       10949632       21852160

memory_used_SLICE
=================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                     5314560        9918464       18241536       34169856
MOD_QRY_D                   4573184        8634368       16484352       30871552
MTH_QRY                     1853440        3530752        7014400       13981696
RSF_QRY                     2756608        5530624       11014144       21916672
RSF_TMP                     2756608        5466112       10949632       21852160

memory_used_RATIO
=================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           3.08           3.42            3.2           2.73
MOD_QRY                         W20            3.9           3.21           2.88           2.75
MOD_QRY                         W40           2.87           2.81            2.6           2.44
MOD_QRY_D                       W10           2.43           2.65           2.65            2.3
MOD_QRY_D                       W20           2.66           2.39            2.3           2.41
MOD_QRY_D                       W40           2.47           2.45           2.35           2.21
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10           1.62           1.48           1.54           1.57
RSF_QRY                         W20           1.48           1.49           1.57           1.58
RSF_QRY                         W40           1.49           1.57           1.57           1.57
RSF_TMP                         W10           1.23           1.48           1.54           1.53
RSF_TMP                         W20           1.48           1.49           1.55           1.57
RSF_TMP                         W40           1.49           1.55           1.56           1.56

memory_used_SLICE_RATIO
=======================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                        2.87           2.81            2.6           2.44
MOD_QRY_D                      2.47           2.45           2.35           2.21
MTH_QRY                           1              1              1              1
RSF_QRY                        1.49           1.57           1.57           1.57
RSF_TMP                        1.49           1.55           1.56           1.56

buffers
=======
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10             38             68            175            317
MOD_QRY                         W20             68            191            317            569
MOD_QRY                         W40            191            317            569           1074
MOD_QRY_D                       W10             38             68            175            317
MOD_QRY_D                       W20             68            191            317            569
MOD_QRY_D                       W40            191            317            569           1074
MTH_QRY                         W10             38             68            175            317
MTH_QRY                         W20             68            191            317            569
MTH_QRY                         W40            191            317            569           1074
RSF_QRY                         W10          45370         159913         761195        2675218
RSF_QRY                         W20          91913         443211        1407218        4851116
RSF_QRY                         W40         252211         773218        2575116        9365407
RSF_TMP                         W10          18689          46582         106491         229838
RSF_TMP                         W20          45554         104461         225778         475661
RSF_TMP                         W40         103425         223742         471581        1118395

buffers_SLICE
=============
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                         191            317            569           1074
MOD_QRY_D                       191            317            569           1074
MTH_QRY                         191            317            569           1074
RSF_QRY                      252211         773218        2575116        9365407
RSF_TMP                      103425         223742         471581        1118395

buffers_RATIO
=============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              1              1              1              1
MOD_QRY                         W20              1              1              1              1
MOD_QRY                         W40              1              1              1              1
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1              1
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10        1193.95        2351.66        4349.69        8439.17
RSF_QRY                         W20        1351.66        2320.48        4439.17        8525.69
RSF_QRY                         W40        1320.48        2439.17        4525.69        8720.12
RSF_TMP                         W10         491.82         685.03         608.52         725.04
RSF_TMP                         W20         669.91         546.92         712.23         835.96
RSF_TMP                         W40         541.49         705.81         828.79        1041.34

buffers_SLICE_RATIO
===================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1              1              1
MOD_QRY_D                         1              1              1              1
MTH_QRY                           1              1              1              1
RSF_QRY                     1320.48        2439.17        4525.69        8720.12
RSF_TMP                      541.49         705.81         828.79        1041.34

disk_reads
==========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              0              0              0              0
MOD_QRY                         W20              0              0              0              0
MOD_QRY                         W40              0              0              0              0
MOD_QRY_D                       W10              0              0              0              0
MOD_QRY_D                       W20              0              0              0              0
MOD_QRY_D                       W40              0              0              0              0
MTH_QRY                         W10              0              0              0              0
MTH_QRY                         W20              0              0              0              0
MTH_QRY                         W40              0              0              0              0
RSF_QRY                         W10              0              0              0              0
RSF_QRY                         W20              0              0              0              0
RSF_QRY                         W40              0              0              0              0
RSF_TMP                         W10              0              0              0              0
RSF_TMP                         W20              0              0              0              0
RSF_TMP                         W40              0              0              0              0

disk_reads_SLICE
================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           0              0              0              0
MOD_QRY_D                         0              0              0              0
MTH_QRY                           0              0              0              0
RSF_QRY                           0              0              0              0
RSF_TMP                           0              0              0              0

disk_reads_RATIO
================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              0              0              0              0
MOD_QRY                         W20              0              0              0              0
MOD_QRY                         W40              0              0              0              0
MOD_QRY_D                       W10              0              0              0              0
MOD_QRY_D                       W20              0              0              0              0
MOD_QRY_D                       W40              0              0              0              0
MTH_QRY                         W10              0              0              0              0
MTH_QRY                         W20              0              0              0              0
MTH_QRY                         W40              0              0              0              0
RSF_QRY                         W10              0              0              0              0
RSF_QRY                         W20              0              0              0              0
RSF_QRY                         W40              0              0              0              0
RSF_TMP                         W10              0              0              0              0
RSF_TMP                         W20              0              0              0              0
RSF_TMP                         W40              0              0              0              0

disk_reads_SLICE_RATIO
======================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           0              0              0              0
MOD_QRY_D                         0              0              0              0
MTH_QRY                           0              0              0              0
RSF_QRY                           0              0              0              0
RSF_TMP                           0              0              0              0

disk_writes
===========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              0              0              0              0
MOD_QRY                         W20              0              0              0              0
MOD_QRY                         W40              0              0              0              0
MOD_QRY_D                       W10              0              0              0              0
MOD_QRY_D                       W20              0              0              0              0
MOD_QRY_D                       W40              0              0              0              0
MTH_QRY                         W10              0              0              0              0
MTH_QRY                         W20              0              0              0              0
MTH_QRY                         W40              0              0              0              0
RSF_QRY                         W10              0              0              0              0
RSF_QRY                         W20              0              0              0              0
RSF_QRY                         W40              0              0              0              0
RSF_TMP                         W10              0              0              0              0
RSF_TMP                         W20              0              0              0              0
RSF_TMP                         W40              0              0              0              0

disk_writes_SLICE
=================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           0              0              0              0
MOD_QRY_D                         0              0              0              0
MTH_QRY                           0              0              0              0
RSF_QRY                           0              0              0              0
RSF_TMP                           0              0              0              0

disk_writes_RATIO
=================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              0              0              0              0
MOD_QRY                         W20              0              0              0              0
MOD_QRY                         W40              0              0              0              0
MOD_QRY_D                       W10              0              0              0              0
MOD_QRY_D                       W20              0              0              0              0
MOD_QRY_D                       W40              0              0              0              0
MTH_QRY                         W10              0              0              0              0
MTH_QRY                         W20              0              0              0              0
MTH_QRY                         W40              0              0              0              0
RSF_QRY                         W10              0              0              0              0
RSF_QRY                         W20              0              0              0              0
RSF_QRY                         W40              0              0              0              0
RSF_TMP                         W10              0              0              0              0
RSF_TMP                         W20              0              0              0              0
RSF_TMP                         W40              0              0              0              0

disk_writes_SLICE_RATIO
=======================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           0              0              0              0
MOD_QRY_D                         0              0              0              0
MTH_QRY                           0              0              0              0
RSF_QRY                           0              0              0              0
RSF_TMP                           0              0              0              0

tempseg_size
============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10
MOD_QRY                         W20
MOD_QRY                         W40
MOD_QRY_D                       W10
MOD_QRY_D                       W20
MOD_QRY_D                       W40
MTH_QRY                         W10
MTH_QRY                         W20
MTH_QRY                         W40
RSF_QRY                         W10
RSF_QRY                         W20
RSF_QRY                         W40
RSF_TMP                         W10
RSF_TMP                         W20
RSF_TMP                         W40

tempseg_size_SLICE
==================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY
MOD_QRY_D
MTH_QRY
RSF_QRY
RSF_TMP

tempseg_size_RATIO
==================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10
MOD_QRY                         W20
MOD_QRY                         W40
MOD_QRY_D                       W10
MOD_QRY_D                       W20
MOD_QRY_D                       W40
MTH_QRY                         W10
MTH_QRY                         W20
MTH_QRY                         W40
RSF_QRY                         W10
RSF_QRY                         W20
RSF_QRY                         W40
RSF_TMP                         W10
RSF_TMP                         W20
RSF_TMP                         W40

tempseg_size_SLICE_RATIO
========================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY
MOD_QRY_D
MTH_QRY
RSF_QRY
RSF_TMP

cardinality
===========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10          10000          20000          40000          80000
MOD_QRY                         W20          20000          40000          80000         160000
MOD_QRY                         W40          40000          80000         160000         320000
MOD_QRY_D                       W10          10000          20000          40000          80000
MOD_QRY_D                       W20          20000          40000          80000         160000
MOD_QRY_D                       W40          40000          80000         160000         320000
MTH_QRY                         W10          10000          20000          40000          80000
MTH_QRY                         W20          20000          40000          80000         160000
MTH_QRY                         W40          40000          80000         160000         320000
RSF_QRY                         W10          10000          20000          40000          80000
RSF_QRY                         W20          20000          40000          80000         160000
RSF_QRY                         W40          40000          80000         160000         320000
RSF_TMP                         W10            100            197            363            948
RSF_TMP                         W20            200            351            729           1592
RSF_TMP                         W40            311            968           1819           3191

cardinality_SLICE
=================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       40000          80000         160000         320000
MOD_QRY_D                     40000          80000         160000         320000
MTH_QRY                       40000          80000         160000         320000
RSF_QRY                       40000          80000         160000         320000
RSF_TMP                         311            968           1819           3191

cardinality_RATIO
=================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10            100         101.52         110.19          84.39
MOD_QRY                         W20            100         113.96         109.74          100.5
MOD_QRY                         W40         128.62          82.64          87.96         100.28
MOD_QRY_D                       W10            100         101.52         110.19          84.39
MOD_QRY_D                       W20            100         113.96         109.74          100.5
MOD_QRY_D                       W40         128.62          82.64          87.96         100.28
MTH_QRY                         W10            100         101.52         110.19          84.39
MTH_QRY                         W20            100         113.96         109.74          100.5
MTH_QRY                         W40         128.62          82.64          87.96         100.28
RSF_QRY                         W10            100         101.52         110.19          84.39
RSF_QRY                         W20            100         113.96         109.74          100.5
RSF_QRY                         W40         128.62          82.64          87.96         100.28
RSF_TMP                         W10              1              1              1              1
RSF_TMP                         W20              1              1              1              1
RSF_TMP                         W40              1              1              1              1

cardinality_SLICE_RATIO
=======================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                      128.62          82.64          87.96         100.28
MOD_QRY_D                    128.62          82.64          87.96         100.28
MTH_QRY                      128.62          82.64          87.96         100.28
RSF_QRY                      128.62          82.64          87.96         100.28
RSF_TMP                           1              1              1              1

output_rows
===========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10          10000          20000          40000          80000
MOD_QRY                         W20          20000          40000          80000         160000
MOD_QRY                         W40          40000          80000         160000         320000
MOD_QRY_D                       W10          10000          20000          40000          80000
MOD_QRY_D                       W20          20000          40000          80000         160000
MOD_QRY_D                       W40          40000          80000         160000         320000
MTH_QRY                         W10          10000          20000          40000          80000
MTH_QRY                         W20          20000          40000          80000         160000
MTH_QRY                         W40          40000          80000         160000         320000
RSF_QRY                         W10       10000000       40000000      160000000      640000000
RSF_QRY                         W20       20000000       80000000      320000000     1280000000
RSF_QRY                         W40       40000000      160000000      640000000     2560000000
RSF_TMP                         W10          10000          20000          40000          80000
RSF_TMP                         W20          20000          40000          80000         160000
RSF_TMP                         W40          40000          80000         160000         320000

output_rows_SLICE
=================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       40000          80000         160000         320000
MOD_QRY_D                     40000          80000         160000         320000
MTH_QRY                       40000          80000         160000         320000
RSF_QRY                    40000000      160000000      640000000     2560000000
RSF_TMP                       40000          80000         160000         320000

output_rows_RATIO
=================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              1              1              1              1
MOD_QRY                         W20              1              1              1              1
MOD_QRY                         W40              1              1              1              1
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1              1
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10           1000           2000           4000           8000
RSF_QRY                         W20           1000           2000           4000           8000
RSF_QRY                         W40           1000           2000           4000           8000
RSF_TMP                         W10              1              1              1              1
RSF_TMP                         W20              1              1              1              1
RSF_TMP                         W40              1              1              1              1

output_rows_SLICE_RATIO
=======================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1              1              1
MOD_QRY_D                         1              1              1              1
MTH_QRY                           1              1              1              1
RSF_QRY                        1000           2000           4000           8000
RSF_TMP                           1              1              1              1

cardinality_error
=================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           9895          19795          39594          79192
MOD_QRY                         W20          19791          39589          79185         158382
MOD_QRY                         W40          39578          79171         158375         316771
MOD_QRY_D                       W10           9895          19795          39594          79192
MOD_QRY_D                       W20          19791          39589          79185         158382
MOD_QRY_D                       W40          39578          79171         158375         316771
MTH_QRY                         W10           9895          19795          39594          79192
MTH_QRY                         W20          19791          39589          79185         158382
MTH_QRY                         W40          39578          79171         158375         316771
RSF_QRY                         W10           9990          20010         120010         560010
RSF_QRY                         W20          19989          39980          80020         480020
RSF_QRY                         W40          39989          79979         159960         320040
RSF_TMP                         W10         990010        3920010       14480010       75760010
RSF_TMP                         W20        3980020       14000020       58240020      254560020
RSF_TMP                         W40       12400040       77360040      290880040     1020800040

cardinality_error_SLICE
=======================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       39578          79171         158375         316771
MOD_QRY_D                     39578          79171         158375         316771
MTH_QRY                       39578          79171         158375         316771
RSF_QRY                       39989          79979         159960         320040
RSF_TMP                    12400040       77360040      290880040     1020800040

cardinality_error_RATIO
=======================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              1              1              1              1
MOD_QRY                         W20              1              1              1              1
MOD_QRY                         W40              1              1              1              1
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1              1
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10           1.01           1.01           3.03           7.07
RSF_QRY                         W20           1.01           1.01           1.01           3.03
RSF_QRY                         W40           1.01           1.01           1.01           1.01
RSF_TMP                         W10         100.05         198.03         365.71         956.66
RSF_TMP                         W20          201.1         353.63         735.49        1607.25
RSF_TMP                         W40         313.31         977.13        1836.65        3222.52

cardinality_error_SLICE_RATIO
=============================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1              1              1
MOD_QRY_D                         1              1              1              1
MTH_QRY                           1              1              1              1
RSF_QRY                        1.01           1.01           1.01           1.01
RSF_TMP                      313.31         977.13        1836.65        3222.52
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'sorts (rows)' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' || Max (CASE point_deep WH
EN 1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 THEN f_real END) || '"'
text FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

sorts (rows)
============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10          20000          40000          80000         160000
MOD_QRY                         W20          40000          80000         160000         320000
MOD_QRY                         W40          80000         160000         320000         640000
MOD_QRY_D                       W10          30000          60000         120000         240000
MOD_QRY_D                       W20          60000         120000         240000         480000
MOD_QRY_D                       W40         120000         240000         480000         960000
MTH_QRY                         W10          10105          20205          40406          80808
MTH_QRY                         W20          20209          40411          80815         161618
MTH_QRY                         W40          40422          80829         161625         323229
RSF_QRY                         W10       10040000       40080000      160160000      640320000
RSF_QRY                         W20       20080000       80160000      320320000     1280640000
RSF_QRY                         W40       40160000      160320000      640640000     2561280000
RSF_TMP                         W10          60000         113046         190102         359798
RSF_TMP                         W20         111472         189128         350112         672820
RSF_TMP                         W40         185792         359330         677204        1312532

sorts (rows)_SLICE
==================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       80000         160000         320000         640000
MOD_QRY_D                    120000         240000         480000         960000
MTH_QRY                       40422          80829         161625         323229
RSF_QRY                    40160000      160320000      640640000     2561280000
RSF_TMP                      185792         359330         677204        1312532

sorts (rows)_RATIO
==================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           1.98           1.98           1.98           1.98
MOD_QRY                         W20           1.98           1.98           1.98           1.98
MOD_QRY                         W40           1.98           1.98           1.98           1.98
MOD_QRY_D                       W10           2.97           2.97           2.97           2.97
MOD_QRY_D                       W20           2.97           2.97           2.97           2.97
MOD_QRY_D                       W40           2.97           2.97           2.97           2.97
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10         993.57        1983.67        3963.77        7923.97
RSF_QRY                         W20         993.62        1983.62        3963.62        7923.87
RSF_QRY                         W40         993.52        1983.45        3963.74        7924.04
RSF_TMP                         W10           5.94           5.59            4.7           4.45
RSF_TMP                         W20           5.52           4.68           4.33           4.16
RSF_TMP                         W40            4.6           4.45           4.19           4.06

sorts (rows)_SLICE_RATIO
========================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                        1.98           1.98           1.98           1.98
MOD_QRY_D                      2.97           2.97           2.97           2.97
MTH_QRY                           1              1              1              1
RSF_QRY                      993.52        1983.45        3963.74        7924.04
RSF_TMP                         4.6           4.45           4.19           4.06

Top Stats
=========
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'temp space allocated (bytes)' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' || Max (CA
SE point_deep WHEN 1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 THEN f_r
eal END) || '"' text FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

temp space allocated (bytes)
============================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              0              0              0              0
MOD_QRY                         W20              0              0              0              0
MOD_QRY                         W40              0              0              0              0
MOD_QRY_D                       W10              0              0              0              0
MOD_QRY_D                       W20              0              0              0              0
MOD_QRY_D                       W40              0              0              0              0
MTH_QRY                         W10              0              0              0              0
MTH_QRY                         W20              0              0              0              0
MTH_QRY                         W40              0              0              0              0
RSF_QRY                         W10              0              0              0              0
RSF_QRY                         W20              0              0              0              0
RSF_QRY                         W40              0              0              0              0
RSF_TMP                         W10        2097152        2097152        4194304        6291456
RSF_TMP                         W20        2097152        4194304        6291456       11534336
RSF_TMP                         W40        4194304        6291456       11534336       22020096

temp space allocated (bytes)_SLICE
==================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           0              0              0              0
MOD_QRY_D                         0              0              0              0
MTH_QRY                           0              0              0              0
RSF_QRY                           0              0              0              0
RSF_TMP                     4194304        6291456       11534336       22020096

temp space allocated (bytes)_RATIO
==================================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              0              0              0              0
MOD_QRY                         W20              0              0              0              0
MOD_QRY                         W40              0              0              0              0
MOD_QRY_D                       W10              0              0              0              0
MOD_QRY_D                       W20              0              0              0              0
MOD_QRY_D                       W40              0              0              0              0
MTH_QRY                         W10              0              0              0              0
MTH_QRY                         W20              0              0              0              0
MTH_QRY                         W40              0              0              0              0
RSF_QRY                         W10              0              0              0              0
RSF_QRY                         W20              0              0              0              0
RSF_QRY                         W40              0              0              0              0
RSF_TMP                         W10  2097152000000  2097152000000  4194304000000  6291456000000
RSF_TMP                         W20  2097152000000  4194304000000  6291456000000 11534336000000
RSF_TMP                         W40  4194304000000  6291456000000 11534336000000 22020096000000

temp space allocated (bytes)_SLICE_RATIO
========================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           0              0              0              0
MOD_QRY_D                         0              0              0              0
MTH_QRY                           0              0              0              0
RSF_QRY                           0              0              0              0
RSF_TMP               4194304000000  6291456000000 11534336000000 22020096000000
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'process queue reference' AND stat_type = 'LATCH') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' || Max (CASE p
oint_deep WHEN 1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 THEN f_real
END) || '"' text FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

process queue reference
=======================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           1042          27017           9012          67269
MOD_QRY                         W20           2104           6858          70058         128878
MOD_QRY                         W40           3442          14426          92323         384220
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1           1046
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10              1           1039           3385          26168
RSF_QRY                         W20              1           2093           6487          68317
RSF_QRY                         W40           1044           4443          33129          98338
RSF_TMP                         W10              1              1              1              1
RSF_TMP                         W20              1              1              1              1
RSF_TMP                         W40              1              1              1              1

process queue reference_SLICE
=============================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                        3442          14426          92323         384220
MOD_QRY_D                         1              1              1           1046
MTH_QRY                           1              1              1              1
RSF_QRY                        1044           4443          33129          98338
RSF_TMP                           1              1              1              1

process queue reference_RATIO
=============================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           1042          27017           9012          67269
MOD_QRY                         W20           2104           6858          70058         128878
MOD_QRY                         W40           3442          14426          92323         384220
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1           1046
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10              1           1039           3385          26168
RSF_QRY                         W20              1           2093           6487          68317
RSF_QRY                         W40           1044           4443          33129          98338
RSF_TMP                         W10              1              1              1              1
RSF_TMP                         W20              1              1              1              1
RSF_TMP                         W40              1              1              1              1

process queue reference_SLICE_RATIO
===================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                        3442          14426          92323         384220
MOD_QRY_D                         1              1              1           1046
MTH_QRY                           1              1              1              1
RSF_QRY                        1044           4443          33129          98338
RSF_TMP                           1              1              1              1
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'table scan disk non-IMC rows gotten' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' ||
Max (CASE point_deep WHEN 1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 T
HEN f_real END) || '"' text FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

table scan disk non-IMC rows gotten
===================================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10          10000          20000          40000          80000
MOD_QRY                         W20          20000          40000          80000         160000
MOD_QRY                         W40          40000          80000         160000         320000
MOD_QRY_D                       W10          10000          20000          40000          80000
MOD_QRY_D                       W20          20000          40000          80000         160000
MOD_QRY_D                       W40          40000          80000         160000         320000
MTH_QRY                         W10          10000          20000          40000          80000
MTH_QRY                         W20          20000          40000          80000         160000
MTH_QRY                         W40          40000          80000         160000         320000
RSF_QRY                         W10       10010000       40020000      160040000      640080000
RSF_QRY                         W20       20020000       80040000      320080000     1280160000
RSF_QRY                         W40       40040000      160080000      640160000     2560320000
RSF_TMP                         W10          30000          20000          40000          80000
RSF_TMP                         W20          20000          40000          80000         160000
RSF_TMP                         W40          40000          80000         160000         320000

table scan disk non-IMC rows gotten_SLICE
=========================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       40000          80000         160000         320000
MOD_QRY_D                     40000          80000         160000         320000
MTH_QRY                       40000          80000         160000         320000
RSF_QRY                    40040000      160080000      640160000     2560320000
RSF_TMP                       40000          80000         160000         320000

table scan disk non-IMC rows gotten_RATIO
=========================================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10              1              1              1              1
MOD_QRY                         W20              1              1              1              1
MOD_QRY                         W40              1              1              1              1
MOD_QRY_D                       W10              1              1              1              1
MOD_QRY_D                       W20              1              1              1              1
MOD_QRY_D                       W40              1              1              1              1
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10           1001           2001           4001           8001
RSF_QRY                         W20           1001           2001           4001           8001
RSF_QRY                         W40           1001           2001           4001           8001
RSF_TMP                         W10              3              1              1              1
RSF_TMP                         W20              1              1              1              1
RSF_TMP                         W40              1              1              1              1

table scan disk non-IMC rows gotten_SLICE_RATIO
===============================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1              1              1
MOD_QRY_D                         1              1              1              1
MTH_QRY                           1              1              1              1
RSF_QRY                        1001           2001           4001           8001
RSF_TMP                           1              1              1              1
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'sorts (rows)' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' || Max (CASE point_deep WH
EN 1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 THEN f_real END) || '"'
text FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

sorts (rows)
============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10          20000          40000          80000         160000
MOD_QRY                         W20          40000          80000         160000         320000
MOD_QRY                         W40          80000         160000         320000         640000
MOD_QRY_D                       W10          30000          60000         120000         240000
MOD_QRY_D                       W20          60000         120000         240000         480000
MOD_QRY_D                       W40         120000         240000         480000         960000
MTH_QRY                         W10          10105          20205          40406          80808
MTH_QRY                         W20          20209          40411          80815         161618
MTH_QRY                         W40          40422          80829         161625         323229
RSF_QRY                         W10       10040000       40080000      160160000      640320000
RSF_QRY                         W20       20080000       80160000      320320000     1280640000
RSF_QRY                         W40       40160000      160320000      640640000     2561280000
RSF_TMP                         W10          60000         113046         190102         359798
RSF_TMP                         W20         111472         189128         350112         672820
RSF_TMP                         W40         185792         359330         677204        1312532

sorts (rows)_SLICE
==================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                       80000         160000         320000         640000
MOD_QRY_D                    120000         240000         480000         960000
MTH_QRY                       40422          80829         161625         323229
RSF_QRY                    40160000      160320000      640640000     2561280000
RSF_TMP                      185792         359330         677204        1312532

sorts (rows)_RATIO
==================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           1.98           1.98           1.98           1.98
MOD_QRY                         W20           1.98           1.98           1.98           1.98
MOD_QRY                         W40           1.98           1.98           1.98           1.98
MOD_QRY_D                       W10           2.97           2.97           2.97           2.97
MOD_QRY_D                       W20           2.97           2.97           2.97           2.97
MOD_QRY_D                       W40           2.97           2.97           2.97           2.97
MTH_QRY                         W10              1              1              1              1
MTH_QRY                         W20              1              1              1              1
MTH_QRY                         W40              1              1              1              1
RSF_QRY                         W10         993.57        1983.67        3963.77        7923.97
RSF_QRY                         W20         993.62        1983.62        3963.62        7923.87
RSF_QRY                         W40         993.52        1983.45        3963.74        7924.04
RSF_TMP                         W10           5.94           5.59            4.7           4.45
RSF_TMP                         W20           5.52           4.68           4.33           4.16
RSF_TMP                         W40            4.6           4.45           4.19           4.06

sorts (rows)_SLICE_RATIO
========================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                        1.98           1.98           1.98           1.98
MOD_QRY_D                      2.97           2.97           2.97           2.97
MTH_QRY                           1              1              1              1
RSF_QRY                      993.52        1983.45        3963.74        7924.04
RSF_TMP                         4.6           4.45           4.19           4.06
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'logical read bytes from cache' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' || Max (C
ASE point_deep WHEN 1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 THEN f_
real END) || '"' text FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

logical read bytes from cache
=============================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10       16826368       15958016       17334272       19701760
MOD_QRY                         W20       17080320       17186816       18071552        8036352
MOD_QRY                         W40        4792320        5472256        8077312       12050432
MOD_QRY_D                       W10       16089088       17178624       17670144       18513920
MOD_QRY_D                       W20       16539648       17154048       18694144        8011776
MOD_QRY_D                       W40        4792320        5873664        8282112       12124160
MTH_QRY                         W10       16670720       17211392       17063936       18399232
MTH_QRY                         W20       16777216       17596416       18145280        7299072
MTH_QRY                         W40        5431296        5898240        7847936       12722176
RSF_QRY                         W10      387252224     1325637632     6252453888    21930770432
RSF_QRY                         W20      768524288     3647430656    11544403968    39743684608
RSF_QRY                         W40     2070929408     6337478656    21098643456    76724371456
RSF_TMP                         W10      185991168      431144960      955179008     2030608384
RSF_TMP                         W20      430202880      952655872     2024398848     4234272768
RSF_TMP                         W40      956489728     2049122304     4332290048    10213171200

logical read bytes from cache_SLICE
===================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                     4792320        5472256        8077312       12050432
MOD_QRY_D                   4792320        5873664        8282112       12124160
MTH_QRY                     5431296        5898240        7847936       12722176
RSF_QRY                  2070929408     6337478656    21098643456    76724371456
RSF_TMP                   956489728     2049122304     4332290048    10213171200

logical read bytes from cache_RATIO
===================================
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           1.05              1           1.02           1.07
MOD_QRY                         W20           1.03              1              1            1.1
MOD_QRY                         W40              1              1           1.03              1
MOD_QRY_D                       W10              1           1.08           1.04           1.01
MOD_QRY_D                       W20              1              1           1.03            1.1
MOD_QRY_D                       W40              1           1.07           1.06           1.01
MTH_QRY                         W10           1.04           1.08              1              1
MTH_QRY                         W20           1.01           1.03              1              1
MTH_QRY                         W40           1.13           1.08              1           1.06
RSF_QRY                         W10          24.07          83.07         366.41        1191.94
RSF_QRY                         W20          46.47         212.63         638.82        5445.03
RSF_QRY                         W40         432.14        1158.11        2688.43        6366.94
RSF_TMP                         W10          11.56          27.02          55.98         110.36
RSF_TMP                         W20          26.01          55.54         112.02         580.11
RSF_TMP                         W40         199.59         374.46         552.03         847.54

logical read bytes from cache_SLICE_RATIO
=========================================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1           1.03              1
MOD_QRY_D                         1           1.07           1.06           1.01
MTH_QRY                        1.13           1.08              1           1.06
RSF_QRY                      432.14        1158.11        2688.43        6366.94
RSF_TMP                      199.59         374.46         552.03         847.54
WITH wit AS (SELECT query_name, point_wide, size_wide, point_deep, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'redo size' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_wide, '"' || query_name || '","W' || size_wide || '","' || Max (CASE point_deep WHEN
1 THEN f_real END) || '","' || Max (CASE point_deep WHEN 2 THEN f_real END) || '","' || Max (CASE point_deep WHEN 3 THEN f_real END) || '","' || Max (CASE point_deep WHEN 4 THEN f_real END) || '"' tex
t FROM wit GROUP BY query_name, point_wide, size_wide) ORDER BY query_name, point_wide

redo size
=========
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10         385788         325816         346144         390064
MOD_QRY                         W20         386152         327056         325916         346140
MOD_QRY                         W40         344756         344380         366016         380912
MOD_QRY_D                       W10         345476         353168         382596         363040
MOD_QRY_D                       W20         347244         343584         381004         364440
MOD_QRY_D                       W40         379896         380304         333460         347928
MTH_QRY                         W10         383860         386196         343428         361868
MTH_QRY                         W20         381604         380396         342680         324824
MTH_QRY                         W40         369580         348136         346828         369596
RSF_QRY                         W10         343244         343888         384680         324972
RSF_QRY                         W20         343360         351672         351804         348752
RSF_QRY                         W40         392176         380396         380492         343920
RSF_TMP                         W10        1011996        1709376        3027732        5711344
RSF_TMP                         W20        1761236        3217776        6021064       11683492
RSF_TMP                         W40        3389236        6522596       12715036       25113080

redo size_SLICE
===============
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                      344756         344380         366016         380912
MOD_QRY_D                    379896         380304         333460         347928
MTH_QRY                      369580         348136         346828         369596
RSF_QRY                      392176         380396         380492         343920
RSF_TMP                     3389236        6522596       12715036       25113080

redo size_RATIO
===============
Run Type                      Width          D1000          D2000          D4000          D8000
MOD_QRY                         W10           1.12              1           1.01            1.2
MOD_QRY                         W20           1.12              1              1           1.07
MOD_QRY                         W40              1              1            1.1           1.11
MOD_QRY_D                       W10           1.01           1.08           1.11           1.12
MOD_QRY_D                       W20           1.01           1.05           1.17           1.12
MOD_QRY_D                       W40            1.1            1.1              1           1.01
MTH_QRY                         W10           1.12           1.19              1           1.11
MTH_QRY                         W20           1.11           1.16           1.05              1
MTH_QRY                         W40           1.07           1.01           1.04           1.07
RSF_QRY                         W10              1           1.06           1.12              1
RSF_QRY                         W20              1           1.08           1.08           1.07
RSF_QRY                         W40           1.14            1.1           1.14              1
RSF_TMP                         W10           2.95           5.25           8.82          17.57
RSF_TMP                         W20           5.13           9.84          18.47          35.97
RSF_TMP                         W40           9.83          18.94          38.13          73.02

redo size_SLICE_RATIO
=====================
Run Type                      D1000          D2000          D4000          D8000
MOD_QRY                           1              1            1.1           1.11
MOD_QRY_D                       1.1            1.1              1           1.01
MTH_QRY                        1.07           1.01           1.04           1.07
RSF_QRY                        1.14            1.1           1.14              1
RSF_TMP                        9.83          18.94          38.13          73.02

Timer Set: File Writer, Constructed at 23 Nov 2016 01:03:15, written at 01:03:17
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000016), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        2.03        1.94             1        2.03100        1.94000
------- ---------- ---------- ------------ ------------- -------------
Total          2.03        1.94             3        0.67700        0.64667
------- ---------- ---------- ------------ ------------- -------------
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, FACT f_real, Round (FACT / Greatest (Min (FACT) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM bench_v$sql_pl
an_stats_all_v) SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (CASE point_wide WHEN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN
 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point_deep, size_deep) ORDER BY query_name, point_deep

Distinct Plans
==============
MOD_QRY: 3/4 (1 of 1)
SQL_ID  2ytj4xgt4v9s4, child number 0
-------------------------------------
/* MOD_QRY */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES AUTOMATIC
ORDER ( sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
= PRESENTV (final_grp[cv()+1], CASE WHEN sub_weight[cv()] >= 5000 THEN
id[cv()] ELSE final_grp[cv()+1] END, id[cv()]) ) ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","817"' FROM all_rows GROUP BY cat, final_grp ORDER BY
cat, final_grp

Plan hash value: 1769595206

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:21:10.69 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:21:10.69 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|55:38:12.41 |    1074 |       |       |          |
|   3 |    SQL MODEL CYCLIC   |       |      1 |    320K|    320K|55:38:12.36 |    1074 |    38M|  4773K|   32M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.19 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

MOD_QRY_D: 3/4 (1 of 1)
SQL_ID  2ws6cb6spn1tm, child number 0
-------------------------------------
/* MOD_QRY_D */ WITH all_rows AS ( SELECT id, cat, seq, weight,
sub_weight, final_grp FROM items MODEL PARTITION BY (cat) DIMENSION BY
(Row_Number() OVER (PARTITION BY cat ORDER BY seq DESC) rn) MEASURES
(id, weight, weight sub_weight, id final_grp, seq) RULES (
sub_weight[rn > 1] = CASE WHEN sub_weight[cv()-1] >= 5000 THEN
weight[cv()] ELSE sub_weight[cv()-1] + weight[cv()] END, final_grp[ANY]
ORDER BY rn DESC = PRESENTV (final_grp[cv()+1], CASE WHEN
sub_weight[cv()] >= 5000 THEN id[cv()] ELSE final_grp[cv()+1] END,
id[cv()]) ) ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || COUNT(*) || '","9807"' FROM all_rows GROUP BY
cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3654604359

--------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |       |      1 |        |   3229 |00:00:02.43 |    1074 |       |       |          |
|   1 |  SORT GROUP BY        |       |      1 |    320K|   3229 |00:00:02.43 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                |       |      1 |    320K|    320K|00:01:32.47 |    1074 |       |       |          |
|   3 |    SQL MODEL ORDERED  |       |      1 |    320K|    320K|00:01:32.42 |    1074 |    36M|  4844K|   29M (0)|
|   4 |     WINDOW SORT       |       |      1 |    320K|    320K|00:00:00.18 |    1074 |    15M|  1460K|   13M (0)|
|   5 |      TABLE ACCESS FULL| ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
--------------------------------------------------------------------------------------------------------------------

MTH_QRY: 3/4 (1 of 1)
SQL_ID  3h9qsj5tyf2ah, child number 0
-------------------------------------
/* MTH_QRY */ SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat || '","'
|| final_grp || '","' || num_rows || '","7787"' FROM items
MATCH_RECOGNIZE ( PARTITION BY cat ORDER BY seq DESC MEASURES FINAL
LAST (id) final_grp, COUNT(*) num_rows ONE ROW PER MATCH PATTERN (s*
t?) DEFINE s AS Sum (weight) < 5000, t AS Sum (weight) >= 5000 ) m
ORDER BY cat, final_grp

Plan hash value: 1353329411

---------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |       |      1 |        |   3229 |00:00:00.17 |    1074 |       |       |          |
|   1 |  SORT ORDER BY         |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |   267K|   267K|  237K (0)|
|   2 |   VIEW                 |       |      1 |    320K|   3229 |00:00:00.17 |    1074 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT|       |      1 |    320K|   3229 |00:00:00.17 |    1074 |    15M|  1460K|   13M (0)|
|   4 |     TABLE ACCESS FULL  | ITEMS |      1 |    320K|    320K|00:00:00.03 |    1074 |       |       |          |
---------------------------------------------------------------------------------------------------------------------

RSF_QRY: 3/4 (1 of 1)
SQL_ID  94y37zwgjs9ug, child number 0
-------------------------------------
/* RSF_QRY */ WITH itm AS ( SELECT id, cat, seq, weight, Row_Number()
OVER (PARTITION BY cat ORDER BY seq DESC) rn FROM items ), rsq (id,
cat, rn, seq, weight, sub_weight, grp_num) AS ( SELECT id, cat, rn,
seq, weight, weight, 1 FROM itm WHERE rn = 1 UNION ALL SELECT itm.id,
itm.cat, itm.rn, itm.seq, itm.weight, itm.weight + CASE WHEN
rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num +
CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN itm ON
itm.rn = rsq.rn + 1 AND itm.cat = rsq.cat ), final_grouping AS ( SELECT
cat cat, First_Value(id) OVER (PARTITION BY cat, grp_num ORDER BY seq)
final_grp FROM rsq ) SELECT /*+ GATHER_PLAN_STATISTICS */ '"' || cat ||
'","' || final_grp || '","' || COUNT(*) || '","5736"' FROM
final_grouping GROUP BY cat, final_grp ORDER BY cat, final_grp

Plan hash value: 3844655400

-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |       |      1 |        |   3229 |00:08:50.80 |    9365K|       |       |          |
|   1 |  SORT GROUP BY                               |       |      1 |     81 |   3229 |00:08:50.80 |    9365K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |       |      1 |     81 |    320K|00:08:50.80 |    9365K|       |       |          |
|   3 |    WINDOW SORT                               |       |      1 |     81 |    320K|00:08:50.75 |    9365K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |       |      1 |     81 |    320K|01:13:10.25 |    9365K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|       |      1 |        |    320K|01:13:10.14 |    9365K|  4096 |  4096 |   20M (0)|
|*  6 |       VIEW                                   |       |      1 |      1 |     40 |00:00:00.34 |    1074 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |       |      1 |    320K|     40 |00:00:00.34 |    1074 |    23M|  1772K|   20M (0)|
|   8 |         TABLE ACCESS FULL                    | ITEMS |      1 |    320K|    320K|00:00:00.04 |    1074 |       |       |          |
|*  9 |       HASH JOIN                              |       |   8000 |     80 |    319K|00:25:04.02 |    8592K|  1214K|  1214K| 1283K (0)|
|  10 |        RECURSIVE WITH PUMP                   |       |   8000 |        |    320K|00:00:00.19 |       0 |       |       |          |
|  11 |        VIEW                                  |       |   8000 |    320K|   2560M|00:30:26.18 |    8592K|       |       |          |
|  12 |         WINDOW SORT                          |       |   8000 |    320K|   2560M|00:24:31.69 |    8592K|    15M|  1460K|   13M (0)|
|  13 |          TABLE ACCESS FULL                   | ITEMS |   8000 |    320K|   2560M|00:04:01.43 |    8592K|       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "CAT" ORDER BY INTERNAL_FUNCTION("SEQ") DESC )<=1)
   9 - access("ITM"."RN"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

RSF_TMP: 3/4 (1 of 1)
SQL_ID  3u062brw9anr9, child number 0
-------------------------------------
/* RSF_TMP */ WITH rsq (id, cat, rn, seq, weight, sub_weight, grp_num)
AS ( SELECT id, cat, itm_rownum, seq, weight, weight, 1 FROM items_tmp
WHERE itm_rownum = 1 UNION ALL SELECT /*+ INDEX (itm items_tmp_n1) */
itm.id, itm.cat, itm.itm_rownum, itm.seq, itm.weight, itm.weight + CASE
WHEN rsq.sub_weight >= 5000 THEN 0 ELSE rsq.sub_weight END, rsq.grp_num
+ CASE WHEN rsq.sub_weight >= 5000 THEN 1 ELSE 0 END FROM rsq JOIN
items_tmp itm ON itm.itm_rownum = rsq.rn + 1 AND itm.cat = rsq.cat ),
final_grouping AS ( SELECT cat cat, First_Value(id) OVER (PARTITION BY
cat, grp_num ORDER BY seq) final_grp FROM rsq ) SELECT /*+
GATHER_PLAN_STATISTICS */ '"' || cat || '","' || final_grp || '","' ||
COUNT(*) || '","8326"' FROM final_grouping GROUP BY cat, final_grp
ORDER BY cat, final_grp

Plan hash value: 3755682712

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name         | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |              |      1 |        |   3229 |00:00:02.69 |    1118K|       |       |          |
|   1 |  SORT GROUP BY                               |              |      1 |     91 |   3229 |00:00:02.69 |    1118K|   267K|   267K|  237K (0)|
|   2 |   VIEW                                       |              |      1 |     91 |    320K|00:00:02.69 |    1118K|       |       |          |
|   3 |    WINDOW SORT                               |              |      1 |     91 |    320K|00:00:02.63 |    1118K|    13M|  1417K|   12M (0)|
|   4 |     VIEW                                     |              |      1 |     91 |    320K|00:00:02.73 |    1118K|       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|              |      1 |        |    320K|00:00:02.65 |    1118K|  4096 |  4096 |   20M (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED    | ITEMS_TMP    |      1 |     40 |     40 |00:00:00.01 |      43 |       |       |          |
|*  7 |        INDEX RANGE SCAN                      | ITEMS_TMP_N1 |      1 |     40 |     40 |00:00:00.01 |       3 |       |       |          |
|   8 |       NESTED LOOPS                           |              |   8000 |     51 |    319K|00:00:01.15 |     346K|       |       |          |
|   9 |        RECURSIVE WITH PUMP                   |              |   8000 |        |    320K|00:00:00.07 |       0 |       |       |          |
|  10 |        TABLE ACCESS BY INDEX ROWID BATCHED   | ITEMS_TMP    |    320K|      1 |    319K|00:00:00.78 |     346K|       |       |          |
|* 11 |         INDEX RANGE SCAN                     | ITEMS_TMP_N1 |    320K|   3191 |    319K|00:00:00.31 |   26059 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   7 - access("ITM_ROWNUM"=1)
  11 - access("ITM"."ITM_ROWNUM"="RSQ"."RN"+1 AND "ITM"."CAT"="RSQ"."CAT")

Note
-----
   - dynamic statistics used: dynamic sampling (level=2)
   - this is an adaptive plan

Data Points
===========
Data Point:               size_wide      size_deep       cpu_time        elapsed       num_recs       per_part     group_size
Data Point                       10           1000            .66           .797          10000           1000            116
Data Point                       10           2000           1.14           1.25          20000           2000            112
Data Point                       10           4000           2.24          2.485          40000           4000            108
Data Point                       10           8000           4.37          4.563          80000           8000            105
Data Point                       20           1000           1.16          1.266          20000           1000            132
Data Point                       20           2000           2.25          2.265          40000           2000            121
Data Point                       20           4000           4.45          6.001          80000           4000            113
Data Point                       20           8000           8.81         10.517         160000           8000            112
Data Point                       40           1000           2.31          2.532          40000           1000            156
Data Point                       40           2000           4.43          7.219          80000           2000            147
Data Point                       40           4000           8.88         11.892         160000           4000            129
Data Point                       40           8000          17.71         27.519         320000           8000            124

num_records_out
===============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000            105            209            422
MOD_QRY                       D2000            205            411            829
MOD_QRY                       D4000            406            815           1625
MOD_QRY                       D8000            808           1618           3229
MOD_QRY_D                     D1000            105            209            422
MOD_QRY_D                     D2000            205            411            829
MOD_QRY_D                     D4000            406            815           1625
MOD_QRY_D                     D8000            808           1618           3229
MTH_QRY                       D1000            105            209            422
MTH_QRY                       D2000            205            411            829
MTH_QRY                       D4000            406            815           1625
MTH_QRY                       D8000            808           1618           3229
RSF_QRY                       D1000            105            209            422
RSF_QRY                       D2000            205            411            829
RSF_QRY                       D4000            406            815           1625
RSF_QRY                       D8000            808           1618           3229
RSF_TMP                       D1000            105            209            422
RSF_TMP                       D2000            205            411            829
RSF_TMP                       D4000            406            815           1625
RSF_TMP                       D8000            808           1618           3229

num_records_out_SLICE
=====================
Run Type                        W10            W20            W40
MOD_QRY                         808           1618           3229
MOD_QRY_D                       808           1618           3229
MTH_QRY                         808           1618           3229
RSF_QRY                         808           1618           3229
RSF_TMP                         808           1618           3229

num_records_out_RATIO
=====================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              1              1              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000              1              1              1
MOD_QRY                       D8000              1              1              1
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1              1
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000              1              1              1
RSF_QRY                       D2000              1              1              1
RSF_QRY                       D4000              1              1              1
RSF_QRY                       D8000              1              1              1
RSF_TMP                       D1000              1              1              1
RSF_TMP                       D2000              1              1              1
RSF_TMP                       D4000              1              1              1
RSF_TMP                       D8000              1              1              1

num_records_out_SLICE_RATIO
===========================
Run Type                        W10            W20            W40
MOD_QRY                           1              1              1
MOD_QRY_D                         1              1              1
MTH_QRY                           1              1              1
RSF_QRY                           1              1              1
RSF_TMP                           1              1              1

cpu_time
========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000             16          47.25          99.14
MOD_QRY                       D2000           61.5          189.5         396.54
MOD_QRY                       D4000         243.04         761.91        1389.86
MOD_QRY                       D8000         961.72        2082.06        5565.41
MOD_QRY_D                     D1000            .08            .15            .32
MOD_QRY_D                     D2000            .16             .3             .6
MOD_QRY_D                     D4000            .29            .59           1.19
MOD_QRY_D                     D8000            .59           1.21           2.42
MTH_QRY                       D1000              0            .01            .01
MTH_QRY                       D2000            .01            .01            .04
MTH_QRY                       D4000            .02            .04            .07
MTH_QRY                       D8000            .05             .1            .17
RSF_QRY                       D1000           5.89          11.63          23.53
RSF_QRY                       D2000          23.15          45.99          93.64
RSF_QRY                       D4000          92.19         184.83         377.25
RSF_QRY                       D8000          368.7         749.77        1512.85
RSF_TMP                       D1000             .1            .19            .36
RSF_TMP                       D2000            .19            .37            .73
RSF_TMP                       D4000            .39            .76           1.54
RSF_TMP                       D8000            .76           1.54           3.27

cpu_time_SLICE
==============
Run Type                        W10            W20            W40
MOD_QRY                      961.72        2082.06        5565.41
MOD_QRY_D                       .59           1.21           2.42
MTH_QRY                         .05             .1            .17
RSF_QRY                       368.7         749.77        1512.85
RSF_TMP                         .76           1.54           3.27

cpu_time_RATIO
==============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000       16000000           4725           9914
MOD_QRY                       D2000           6150          18950         9913.5
MOD_QRY                       D4000          12152       19047.75       19855.14
MOD_QRY                       D8000        19234.4        20820.6       32737.71
MOD_QRY_D                     D1000          80000             15             32
MOD_QRY_D                     D2000             16             30             15
MOD_QRY_D                     D4000           14.5          14.75             17
MOD_QRY_D                     D8000           11.8           12.1          14.24
MTH_QRY                       D1000              0              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000        5890000           1163           2353
RSF_QRY                       D2000           2315           4599           2341
RSF_QRY                       D4000         4609.5        4620.75        5389.29
RSF_QRY                       D8000           7374         7497.7        8899.12
RSF_TMP                       D1000         100000             19             36
RSF_TMP                       D2000             19             37          18.25
RSF_TMP                       D4000           19.5             19             22
RSF_TMP                       D8000           15.2           15.4          19.24

cpu_time_SLICE_RATIO
====================
Run Type                        W10            W20            W40
MOD_QRY                     19234.4        20820.6       32737.71
MOD_QRY_D                      11.8           12.1          14.24
MTH_QRY                           1              1              1
RSF_QRY                        7374         7497.7        8899.12
RSF_TMP                        15.2           15.4          19.24

elapsed_time
============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000         16.002         47.257          99.14
MOD_QRY                       D2000         61.506        189.504        396.538
MOD_QRY                       D4000         243.07        761.983       1389.921
MOD_QRY                       D8000        961.818       2082.192       5565.654
MOD_QRY_D                     D1000           .078           .157           .312
MOD_QRY_D                     D2000           .156           .298            .61
MOD_QRY_D                     D4000           .297           .594          1.188
MOD_QRY_D                     D8000           .594          1.203          2.422
MTH_QRY                       D1000              0           .016           .016
MTH_QRY                       D2000           .016           .016           .047
MTH_QRY                       D4000           .016           .031           .078
MTH_QRY                       D8000           .047           .094           .172
RSF_QRY                       D1000          5.891         11.626         23.533
RSF_QRY                       D2000         23.159         45.973         93.651
RSF_QRY                       D4000         92.182        184.832        377.278
RSF_QRY                       D8000        368.731        749.806        1512.88
RSF_TMP                       D1000           .094           .188            .36
RSF_TMP                       D2000           .188           .375           .734
RSF_TMP                       D4000           .391           .766          1.531
RSF_TMP                       D8000           .766          1.547          3.485

elapsed_time_SLICE
==================
Run Type                        W10            W20            W40
MOD_QRY                     961.818       2082.192       5565.654
MOD_QRY_D                      .594          1.203          2.422
MTH_QRY                        .047           .094           .172
RSF_QRY                     368.731        749.806        1512.88
RSF_TMP                        .766          1.547          3.485

elapsed_time_RATIO
==================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000       16002000        2953.56        6196.25
MOD_QRY                       D2000        3844.13          11844        8436.98
MOD_QRY                       D4000       15191.88        24580.1        17819.5
MOD_QRY                       D8000       20464.21       22150.98       32358.45
MOD_QRY_D                     D1000          78000           9.81           19.5
MOD_QRY_D                     D2000           9.75          18.63          12.98
MOD_QRY_D                     D4000          18.56          19.16          15.23
MOD_QRY_D                     D8000          12.64           12.8          14.08
MTH_QRY                       D1000              0              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000        5891000         726.63        1470.81
RSF_QRY                       D2000        1447.44        2873.31        1992.57
RSF_QRY                       D4000        5761.38        5962.32         4836.9
RSF_QRY                       D8000        7845.34        7976.66        8795.81
RSF_TMP                       D1000          94000          11.75           22.5
RSF_TMP                       D2000          11.75          23.44          15.62
RSF_TMP                       D4000          24.44          24.71          19.63
RSF_TMP                       D8000           16.3          16.46          20.26

elapsed_time_SLICE_RATIO
========================
Run Type                        W10            W20            W40
MOD_QRY                    20464.21       22150.98       32358.45
MOD_QRY_D                     12.64           12.8          14.08
MTH_QRY                           1              1              1
RSF_QRY                     7845.34        7976.66        8795.81
RSF_TMP                        16.3          16.46          20.26

memory_used
===========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000        1735680        3708928        5314560
MOD_QRY                       D2000        3250176        5952512        9918464
MOD_QRY                       D4000        5727232       10167296       18241536
MOD_QRY                       D8000        9653248       19100672       34169856
MOD_QRY_D                     D1000        1371136        2531328        4573184
MOD_QRY_D                     D2000        2521088        4429824        8634368
MOD_QRY_D                     D4000        4744192        8124416       16484352
MOD_QRY_D                     D8000        8128512       16745472       30871552
MTH_QRY                       D1000         563200         950272        1853440
MTH_QRY                       D2000         950272        1853440        3530752
MTH_QRY                       D4000        1788928        3530752        7014400
MTH_QRY                       D8000        3530752        6949888       13981696
RSF_QRY                       D1000         913408        1401856        2756608
RSF_QRY                       D2000        1401856        2756608        5530624
RSF_QRY                       D4000        2756608        5530624       11014144
RSF_QRY                       D8000        5530624       11014144       21916672
RSF_TMP                       D1000         692224        1401856        2756608
RSF_TMP                       D2000        1401856        2756608        5466112
RSF_TMP                       D4000        2756608        5466112       10949632
RSF_TMP                       D8000        5401600       10885120       21852160

memory_used_SLICE
=================
Run Type                        W10            W20            W40
MOD_QRY                     9653248       19100672       34169856
MOD_QRY_D                   8128512       16745472       30871552
MTH_QRY                     3530752        6949888       13981696
RSF_QRY                     5530624       11014144       21916672
RSF_TMP                     5401600       10885120       21852160

memory_used_RATIO
=================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           3.08            3.9           2.87
MOD_QRY                       D2000           3.42           3.21           2.81
MOD_QRY                       D4000            3.2           2.88            2.6
MOD_QRY                       D8000           2.73           2.75           2.44
MOD_QRY_D                     D1000           2.43           2.66           2.47
MOD_QRY_D                     D2000           2.65           2.39           2.45
MOD_QRY_D                     D4000           2.65            2.3           2.35
MOD_QRY_D                     D8000            2.3           2.41           2.21
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000           1.62           1.48           1.49
RSF_QRY                       D2000           1.48           1.49           1.57
RSF_QRY                       D4000           1.54           1.57           1.57
RSF_QRY                       D8000           1.57           1.58           1.57
RSF_TMP                       D1000           1.23           1.48           1.49
RSF_TMP                       D2000           1.48           1.49           1.55
RSF_TMP                       D4000           1.54           1.55           1.56
RSF_TMP                       D8000           1.53           1.57           1.56

memory_used_SLICE_RATIO
=======================
Run Type                        W10            W20            W40
MOD_QRY                        2.73           2.75           2.44
MOD_QRY_D                       2.3           2.41           2.21
MTH_QRY                           1              1              1
RSF_QRY                        1.57           1.58           1.57
RSF_TMP                        1.53           1.57           1.56

buffers
=======
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000             38             68            191
MOD_QRY                       D2000             68            191            317
MOD_QRY                       D4000            175            317            569
MOD_QRY                       D8000            317            569           1074
MOD_QRY_D                     D1000             38             68            191
MOD_QRY_D                     D2000             68            191            317
MOD_QRY_D                     D4000            175            317            569
MOD_QRY_D                     D8000            317            569           1074
MTH_QRY                       D1000             38             68            191
MTH_QRY                       D2000             68            191            317
MTH_QRY                       D4000            175            317            569
MTH_QRY                       D8000            317            569           1074
RSF_QRY                       D1000          45370          91913         252211
RSF_QRY                       D2000         159913         443211         773218
RSF_QRY                       D4000         761195        1407218        2575116
RSF_QRY                       D8000        2675218        4851116        9365407
RSF_TMP                       D1000          18689          45554         103425
RSF_TMP                       D2000          46582         104461         223742
RSF_TMP                       D4000         106491         225778         471581
RSF_TMP                       D8000         229838         475661        1118395

buffers_SLICE
=============
Run Type                        W10            W20            W40
MOD_QRY                         317            569           1074
MOD_QRY_D                       317            569           1074
MTH_QRY                         317            569           1074
RSF_QRY                     2675218        4851116        9365407
RSF_TMP                      229838         475661        1118395

buffers_RATIO
=============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              1              1              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000              1              1              1
MOD_QRY                       D8000              1              1              1
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1              1
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000        1193.95        1351.66        1320.48
RSF_QRY                       D2000        2351.66        2320.48        2439.17
RSF_QRY                       D4000        4349.69        4439.17        4525.69
RSF_QRY                       D8000        8439.17        8525.69        8720.12
RSF_TMP                       D1000         491.82         669.91         541.49
RSF_TMP                       D2000         685.03         546.92         705.81
RSF_TMP                       D4000         608.52         712.23         828.79
RSF_TMP                       D8000         725.04         835.96        1041.34

buffers_SLICE_RATIO
===================
Run Type                        W10            W20            W40
MOD_QRY                           1              1              1
MOD_QRY_D                         1              1              1
MTH_QRY                           1              1              1
RSF_QRY                     8439.17        8525.69        8720.12
RSF_TMP                      725.04         835.96        1041.34

disk_reads
==========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              0              0              0
MOD_QRY                       D2000              0              0              0
MOD_QRY                       D4000              0              0              0
MOD_QRY                       D8000              0              0              0
MOD_QRY_D                     D1000              0              0              0
MOD_QRY_D                     D2000              0              0              0
MOD_QRY_D                     D4000              0              0              0
MOD_QRY_D                     D8000              0              0              0
MTH_QRY                       D1000              0              0              0
MTH_QRY                       D2000              0              0              0
MTH_QRY                       D4000              0              0              0
MTH_QRY                       D8000              0              0              0
RSF_QRY                       D1000              0              0              0
RSF_QRY                       D2000              0              0              0
RSF_QRY                       D4000              0              0              0
RSF_QRY                       D8000              0              0              0
RSF_TMP                       D1000              0              0              0
RSF_TMP                       D2000              0              0              0
RSF_TMP                       D4000              0              0              0
RSF_TMP                       D8000              0              0              0

disk_reads_SLICE
================
Run Type                        W10            W20            W40
MOD_QRY                           0              0              0
MOD_QRY_D                         0              0              0
MTH_QRY                           0              0              0
RSF_QRY                           0              0              0
RSF_TMP                           0              0              0

disk_reads_RATIO
================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              0              0              0
MOD_QRY                       D2000              0              0              0
MOD_QRY                       D4000              0              0              0
MOD_QRY                       D8000              0              0              0
MOD_QRY_D                     D1000              0              0              0
MOD_QRY_D                     D2000              0              0              0
MOD_QRY_D                     D4000              0              0              0
MOD_QRY_D                     D8000              0              0              0
MTH_QRY                       D1000              0              0              0
MTH_QRY                       D2000              0              0              0
MTH_QRY                       D4000              0              0              0
MTH_QRY                       D8000              0              0              0
RSF_QRY                       D1000              0              0              0
RSF_QRY                       D2000              0              0              0
RSF_QRY                       D4000              0              0              0
RSF_QRY                       D8000              0              0              0
RSF_TMP                       D1000              0              0              0
RSF_TMP                       D2000              0              0              0
RSF_TMP                       D4000              0              0              0
RSF_TMP                       D8000              0              0              0

disk_reads_SLICE_RATIO
======================
Run Type                        W10            W20            W40
MOD_QRY                           0              0              0
MOD_QRY_D                         0              0              0
MTH_QRY                           0              0              0
RSF_QRY                           0              0              0
RSF_TMP                           0              0              0

disk_writes
===========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              0              0              0
MOD_QRY                       D2000              0              0              0
MOD_QRY                       D4000              0              0              0
MOD_QRY                       D8000              0              0              0
MOD_QRY_D                     D1000              0              0              0
MOD_QRY_D                     D2000              0              0              0
MOD_QRY_D                     D4000              0              0              0
MOD_QRY_D                     D8000              0              0              0
MTH_QRY                       D1000              0              0              0
MTH_QRY                       D2000              0              0              0
MTH_QRY                       D4000              0              0              0
MTH_QRY                       D8000              0              0              0
RSF_QRY                       D1000              0              0              0
RSF_QRY                       D2000              0              0              0
RSF_QRY                       D4000              0              0              0
RSF_QRY                       D8000              0              0              0
RSF_TMP                       D1000              0              0              0
RSF_TMP                       D2000              0              0              0
RSF_TMP                       D4000              0              0              0
RSF_TMP                       D8000              0              0              0

disk_writes_SLICE
=================
Run Type                        W10            W20            W40
MOD_QRY                           0              0              0
MOD_QRY_D                         0              0              0
MTH_QRY                           0              0              0
RSF_QRY                           0              0              0
RSF_TMP                           0              0              0

disk_writes_RATIO
=================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              0              0              0
MOD_QRY                       D2000              0              0              0
MOD_QRY                       D4000              0              0              0
MOD_QRY                       D8000              0              0              0
MOD_QRY_D                     D1000              0              0              0
MOD_QRY_D                     D2000              0              0              0
MOD_QRY_D                     D4000              0              0              0
MOD_QRY_D                     D8000              0              0              0
MTH_QRY                       D1000              0              0              0
MTH_QRY                       D2000              0              0              0
MTH_QRY                       D4000              0              0              0
MTH_QRY                       D8000              0              0              0
RSF_QRY                       D1000              0              0              0
RSF_QRY                       D2000              0              0              0
RSF_QRY                       D4000              0              0              0
RSF_QRY                       D8000              0              0              0
RSF_TMP                       D1000              0              0              0
RSF_TMP                       D2000              0              0              0
RSF_TMP                       D4000              0              0              0
RSF_TMP                       D8000              0              0              0

disk_writes_SLICE_RATIO
=======================
Run Type                        W10            W20            W40
MOD_QRY                           0              0              0
MOD_QRY_D                         0              0              0
MTH_QRY                           0              0              0
RSF_QRY                           0              0              0
RSF_TMP                           0              0              0

tempseg_size
============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000
MOD_QRY                       D2000
MOD_QRY                       D4000
MOD_QRY                       D8000
MOD_QRY_D                     D1000
MOD_QRY_D                     D2000
MOD_QRY_D                     D4000
MOD_QRY_D                     D8000
MTH_QRY                       D1000
MTH_QRY                       D2000
MTH_QRY                       D4000
MTH_QRY                       D8000
RSF_QRY                       D1000
RSF_QRY                       D2000
RSF_QRY                       D4000
RSF_QRY                       D8000
RSF_TMP                       D1000
RSF_TMP                       D2000
RSF_TMP                       D4000
RSF_TMP                       D8000

tempseg_size_SLICE
==================
Run Type                        W10            W20            W40
MOD_QRY
MOD_QRY_D
MTH_QRY
RSF_QRY
RSF_TMP

tempseg_size_RATIO
==================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000
MOD_QRY                       D2000
MOD_QRY                       D4000
MOD_QRY                       D8000
MOD_QRY_D                     D1000
MOD_QRY_D                     D2000
MOD_QRY_D                     D4000
MOD_QRY_D                     D8000
MTH_QRY                       D1000
MTH_QRY                       D2000
MTH_QRY                       D4000
MTH_QRY                       D8000
RSF_QRY                       D1000
RSF_QRY                       D2000
RSF_QRY                       D4000
RSF_QRY                       D8000
RSF_TMP                       D1000
RSF_TMP                       D2000
RSF_TMP                       D4000
RSF_TMP                       D8000

tempseg_size_SLICE_RATIO
========================
Run Type                        W10            W20            W40
MOD_QRY
MOD_QRY_D
MTH_QRY
RSF_QRY
RSF_TMP

cardinality
===========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000          10000          20000          40000
MOD_QRY                       D2000          20000          40000          80000
MOD_QRY                       D4000          40000          80000         160000
MOD_QRY                       D8000          80000         160000         320000
MOD_QRY_D                     D1000          10000          20000          40000
MOD_QRY_D                     D2000          20000          40000          80000
MOD_QRY_D                     D4000          40000          80000         160000
MOD_QRY_D                     D8000          80000         160000         320000
MTH_QRY                       D1000          10000          20000          40000
MTH_QRY                       D2000          20000          40000          80000
MTH_QRY                       D4000          40000          80000         160000
MTH_QRY                       D8000          80000         160000         320000
RSF_QRY                       D1000          10000          20000          40000
RSF_QRY                       D2000          20000          40000          80000
RSF_QRY                       D4000          40000          80000         160000
RSF_QRY                       D8000          80000         160000         320000
RSF_TMP                       D1000            100            200            311
RSF_TMP                       D2000            197            351            968
RSF_TMP                       D4000            363            729           1819
RSF_TMP                       D8000            948           1592           3191

cardinality_SLICE
=================
Run Type                        W10            W20            W40
MOD_QRY                       80000         160000         320000
MOD_QRY_D                     80000         160000         320000
MTH_QRY                       80000         160000         320000
RSF_QRY                       80000         160000         320000
RSF_TMP                         948           1592           3191

cardinality_RATIO
=================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000            100            100         128.62
MOD_QRY                       D2000         101.52         113.96          82.64
MOD_QRY                       D4000         110.19         109.74          87.96
MOD_QRY                       D8000          84.39          100.5         100.28
MOD_QRY_D                     D1000            100            100         128.62
MOD_QRY_D                     D2000         101.52         113.96          82.64
MOD_QRY_D                     D4000         110.19         109.74          87.96
MOD_QRY_D                     D8000          84.39          100.5         100.28
MTH_QRY                       D1000            100            100         128.62
MTH_QRY                       D2000         101.52         113.96          82.64
MTH_QRY                       D4000         110.19         109.74          87.96
MTH_QRY                       D8000          84.39          100.5         100.28
RSF_QRY                       D1000            100            100         128.62
RSF_QRY                       D2000         101.52         113.96          82.64
RSF_QRY                       D4000         110.19         109.74          87.96
RSF_QRY                       D8000          84.39          100.5         100.28
RSF_TMP                       D1000              1              1              1
RSF_TMP                       D2000              1              1              1
RSF_TMP                       D4000              1              1              1
RSF_TMP                       D8000              1              1              1

cardinality_SLICE_RATIO
=======================
Run Type                        W10            W20            W40
MOD_QRY                       84.39          100.5         100.28
MOD_QRY_D                     84.39          100.5         100.28
MTH_QRY                       84.39          100.5         100.28
RSF_QRY                       84.39          100.5         100.28
RSF_TMP                           1              1              1

output_rows
===========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000          10000          20000          40000
MOD_QRY                       D2000          20000          40000          80000
MOD_QRY                       D4000          40000          80000         160000
MOD_QRY                       D8000          80000         160000         320000
MOD_QRY_D                     D1000          10000          20000          40000
MOD_QRY_D                     D2000          20000          40000          80000
MOD_QRY_D                     D4000          40000          80000         160000
MOD_QRY_D                     D8000          80000         160000         320000
MTH_QRY                       D1000          10000          20000          40000
MTH_QRY                       D2000          20000          40000          80000
MTH_QRY                       D4000          40000          80000         160000
MTH_QRY                       D8000          80000         160000         320000
RSF_QRY                       D1000       10000000       20000000       40000000
RSF_QRY                       D2000       40000000       80000000      160000000
RSF_QRY                       D4000      160000000      320000000      640000000
RSF_QRY                       D8000      640000000     1280000000     2560000000
RSF_TMP                       D1000          10000          20000          40000
RSF_TMP                       D2000          20000          40000          80000
RSF_TMP                       D4000          40000          80000         160000
RSF_TMP                       D8000          80000         160000         320000

output_rows_SLICE
=================
Run Type                        W10            W20            W40
MOD_QRY                       80000         160000         320000
MOD_QRY_D                     80000         160000         320000
MTH_QRY                       80000         160000         320000
RSF_QRY                   640000000     1280000000     2560000000
RSF_TMP                       80000         160000         320000

output_rows_RATIO
=================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              1              1              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000              1              1              1
MOD_QRY                       D8000              1              1              1
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1              1
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000           1000           1000           1000
RSF_QRY                       D2000           2000           2000           2000
RSF_QRY                       D4000           4000           4000           4000
RSF_QRY                       D8000           8000           8000           8000
RSF_TMP                       D1000              1              1              1
RSF_TMP                       D2000              1              1              1
RSF_TMP                       D4000              1              1              1
RSF_TMP                       D8000              1              1              1

output_rows_SLICE_RATIO
=======================
Run Type                        W10            W20            W40
MOD_QRY                           1              1              1
MOD_QRY_D                         1              1              1
MTH_QRY                           1              1              1
RSF_QRY                        8000           8000           8000
RSF_TMP                           1              1              1

cardinality_error
=================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           9895          19791          39578
MOD_QRY                       D2000          19795          39589          79171
MOD_QRY                       D4000          39594          79185         158375
MOD_QRY                       D8000          79192         158382         316771
MOD_QRY_D                     D1000           9895          19791          39578
MOD_QRY_D                     D2000          19795          39589          79171
MOD_QRY_D                     D4000          39594          79185         158375
MOD_QRY_D                     D8000          79192         158382         316771
MTH_QRY                       D1000           9895          19791          39578
MTH_QRY                       D2000          19795          39589          79171
MTH_QRY                       D4000          39594          79185         158375
MTH_QRY                       D8000          79192         158382         316771
RSF_QRY                       D1000           9990          19989          39989
RSF_QRY                       D2000          20010          39980          79979
RSF_QRY                       D4000         120010          80020         159960
RSF_QRY                       D8000         560010         480020         320040
RSF_TMP                       D1000         990010        3980020       12400040
RSF_TMP                       D2000        3920010       14000020       77360040
RSF_TMP                       D4000       14480010       58240020      290880040
RSF_TMP                       D8000       75760010      254560020     1020800040

cardinality_error_SLICE
=======================
Run Type                        W10            W20            W40
MOD_QRY                       79192         158382         316771
MOD_QRY_D                     79192         158382         316771
MTH_QRY                       79192         158382         316771
RSF_QRY                      560010         480020         320040
RSF_TMP                    75760010      254560020     1020800040

cardinality_error_RATIO
=======================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              1              1              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000              1              1              1
MOD_QRY                       D8000              1              1              1
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1              1
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000           1.01           1.01           1.01
RSF_QRY                       D2000           1.01           1.01           1.01
RSF_QRY                       D4000           3.03           1.01           1.01
RSF_QRY                       D8000           7.07           3.03           1.01
RSF_TMP                       D1000         100.05          201.1         313.31
RSF_TMP                       D2000         198.03         353.63         977.13
RSF_TMP                       D4000         365.71         735.49        1836.65
RSF_TMP                       D8000         956.66        1607.25        3222.52

cardinality_error_SLICE_RATIO
=============================
Run Type                        W10            W20            W40
MOD_QRY                           1              1              1
MOD_QRY_D                         1              1              1
MTH_QRY                           1              1              1
RSF_QRY                        7.07           3.03           1.01
RSF_TMP                      956.66        1607.25        3222.52
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'sorts (rows)' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (CASE point_wide WH
EN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point_deep, size_deep)
 ORDER BY query_name, point_deep

sorts (rows)
============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000          20000          40000          80000
MOD_QRY                       D2000          40000          80000         160000
MOD_QRY                       D4000          80000         160000         320000
MOD_QRY                       D8000         160000         320000         640000
MOD_QRY_D                     D1000          30000          60000         120000
MOD_QRY_D                     D2000          60000         120000         240000
MOD_QRY_D                     D4000         120000         240000         480000
MOD_QRY_D                     D8000         240000         480000         960000
MTH_QRY                       D1000          10105          20209          40422
MTH_QRY                       D2000          20205          40411          80829
MTH_QRY                       D4000          40406          80815         161625
MTH_QRY                       D8000          80808         161618         323229
RSF_QRY                       D1000       10040000       20080000       40160000
RSF_QRY                       D2000       40080000       80160000      160320000
RSF_QRY                       D4000      160160000      320320000      640640000
RSF_QRY                       D8000      640320000     1280640000     2561280000
RSF_TMP                       D1000          60000         111472         185792
RSF_TMP                       D2000         113046         189128         359330
RSF_TMP                       D4000         190102         350112         677204
RSF_TMP                       D8000         359798         672820        1312532

sorts (rows)_SLICE
==================
Run Type                        W10            W20            W40
MOD_QRY                      160000         320000         640000
MOD_QRY_D                    240000         480000         960000
MTH_QRY                       80808         161618         323229
RSF_QRY                   640320000     1280640000     2561280000
RSF_TMP                      359798         672820        1312532

sorts (rows)_RATIO
==================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           1.98           1.98           1.98
MOD_QRY                       D2000           1.98           1.98           1.98
MOD_QRY                       D4000           1.98           1.98           1.98
MOD_QRY                       D8000           1.98           1.98           1.98
MOD_QRY_D                     D1000           2.97           2.97           2.97
MOD_QRY_D                     D2000           2.97           2.97           2.97
MOD_QRY_D                     D4000           2.97           2.97           2.97
MOD_QRY_D                     D8000           2.97           2.97           2.97
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000         993.57         993.62         993.52
RSF_QRY                       D2000        1983.67        1983.62        1983.45
RSF_QRY                       D4000        3963.77        3963.62        3963.74
RSF_QRY                       D8000        7923.97        7923.87        7924.04
RSF_TMP                       D1000           5.94           5.52            4.6
RSF_TMP                       D2000           5.59           4.68           4.45
RSF_TMP                       D4000            4.7           4.33           4.19
RSF_TMP                       D8000           4.45           4.16           4.06

sorts (rows)_SLICE_RATIO
========================
Run Type                        W10            W20            W40
MOD_QRY                        1.98           1.98           1.98
MOD_QRY_D                      2.97           2.97           2.97
MTH_QRY                           1              1              1
RSF_QRY                     7923.97        7923.87        7924.04
RSF_TMP                        4.45           4.16           4.06

Top Stats
=========
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'temp space allocated (bytes)' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (CA
SE point_wide WHEN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point_
deep, size_deep) ORDER BY query_name, point_deep

temp space allocated (bytes)
============================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              0              0              0
MOD_QRY                       D2000              0              0              0
MOD_QRY                       D4000              0              0              0
MOD_QRY                       D8000              0              0              0
MOD_QRY_D                     D1000              0              0              0
MOD_QRY_D                     D2000              0              0              0
MOD_QRY_D                     D4000              0              0              0
MOD_QRY_D                     D8000              0              0              0
MTH_QRY                       D1000              0              0              0
MTH_QRY                       D2000              0              0              0
MTH_QRY                       D4000              0              0              0
MTH_QRY                       D8000              0              0              0
RSF_QRY                       D1000              0              0              0
RSF_QRY                       D2000              0              0              0
RSF_QRY                       D4000              0              0              0
RSF_QRY                       D8000              0              0              0
RSF_TMP                       D1000        2097152        2097152        4194304
RSF_TMP                       D2000        2097152        4194304        6291456
RSF_TMP                       D4000        4194304        6291456       11534336
RSF_TMP                       D8000        6291456       11534336       22020096

temp space allocated (bytes)_SLICE
==================================
Run Type                        W10            W20            W40
MOD_QRY                           0              0              0
MOD_QRY_D                         0              0              0
MTH_QRY                           0              0              0
RSF_QRY                           0              0              0
RSF_TMP                     6291456       11534336       22020096

temp space allocated (bytes)_RATIO
==================================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              0              0              0
MOD_QRY                       D2000              0              0              0
MOD_QRY                       D4000              0              0              0
MOD_QRY                       D8000              0              0              0
MOD_QRY_D                     D1000              0              0              0
MOD_QRY_D                     D2000              0              0              0
MOD_QRY_D                     D4000              0              0              0
MOD_QRY_D                     D8000              0              0              0
MTH_QRY                       D1000              0              0              0
MTH_QRY                       D2000              0              0              0
MTH_QRY                       D4000              0              0              0
MTH_QRY                       D8000              0              0              0
RSF_QRY                       D1000              0              0              0
RSF_QRY                       D2000              0              0              0
RSF_QRY                       D4000              0              0              0
RSF_QRY                       D8000              0              0              0
RSF_TMP                       D1000  2097152000000  2097152000000  4194304000000
RSF_TMP                       D2000  2097152000000  4194304000000  6291456000000
RSF_TMP                       D4000  4194304000000  6291456000000 11534336000000
RSF_TMP                       D8000  6291456000000 11534336000000 22020096000000

temp space allocated (bytes)_SLICE_RATIO
========================================
Run Type                        W10            W20            W40
MOD_QRY                           0              0              0
MOD_QRY_D                         0              0              0
MTH_QRY                           0              0              0
RSF_QRY                           0              0              0
RSF_TMP               6291456000000 11534336000000 22020096000000
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'process queue reference' AND stat_type = 'LATCH') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (CASE p
oint_wide WHEN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point_deep
, size_deep) ORDER BY query_name, point_deep

process queue reference
=======================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           1042           2104           3442
MOD_QRY                       D2000          27017           6858          14426
MOD_QRY                       D4000           9012          70058          92323
MOD_QRY                       D8000          67269         128878         384220
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1           1046
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000              1              1           1044
RSF_QRY                       D2000           1039           2093           4443
RSF_QRY                       D4000           3385           6487          33129
RSF_QRY                       D8000          26168          68317          98338
RSF_TMP                       D1000              1              1              1
RSF_TMP                       D2000              1              1              1
RSF_TMP                       D4000              1              1              1
RSF_TMP                       D8000              1              1              1

process queue reference_SLICE
=============================
Run Type                        W10            W20            W40
MOD_QRY                       67269         128878         384220
MOD_QRY_D                         1              1           1046
MTH_QRY                           1              1              1
RSF_QRY                       26168          68317          98338
RSF_TMP                           1              1              1

process queue reference_RATIO
=============================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           1042           2104           3442
MOD_QRY                       D2000          27017           6858          14426
MOD_QRY                       D4000           9012          70058          92323
MOD_QRY                       D8000          67269         128878         384220
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1           1046
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000              1              1           1044
RSF_QRY                       D2000           1039           2093           4443
RSF_QRY                       D4000           3385           6487          33129
RSF_QRY                       D8000          26168          68317          98338
RSF_TMP                       D1000              1              1              1
RSF_TMP                       D2000              1              1              1
RSF_TMP                       D4000              1              1              1
RSF_TMP                       D8000              1              1              1

process queue reference_SLICE_RATIO
===================================
Run Type                        W10            W20            W40
MOD_QRY                       67269         128878         384220
MOD_QRY_D                         1              1           1046
MTH_QRY                           1              1              1
RSF_QRY                       26168          68317          98338
RSF_TMP                           1              1              1
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'table scan disk non-IMC rows gotten' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' ||
Max (CASE point_wide WHEN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name,
 point_deep, size_deep) ORDER BY query_name, point_deep

table scan disk non-IMC rows gotten
===================================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000          10000          20000          40000
MOD_QRY                       D2000          20000          40000          80000
MOD_QRY                       D4000          40000          80000         160000
MOD_QRY                       D8000          80000         160000         320000
MOD_QRY_D                     D1000          10000          20000          40000
MOD_QRY_D                     D2000          20000          40000          80000
MOD_QRY_D                     D4000          40000          80000         160000
MOD_QRY_D                     D8000          80000         160000         320000
MTH_QRY                       D1000          10000          20000          40000
MTH_QRY                       D2000          20000          40000          80000
MTH_QRY                       D4000          40000          80000         160000
MTH_QRY                       D8000          80000         160000         320000
RSF_QRY                       D1000       10010000       20020000       40040000
RSF_QRY                       D2000       40020000       80040000      160080000
RSF_QRY                       D4000      160040000      320080000      640160000
RSF_QRY                       D8000      640080000     1280160000     2560320000
RSF_TMP                       D1000          30000          20000          40000
RSF_TMP                       D2000          20000          40000          80000
RSF_TMP                       D4000          40000          80000         160000
RSF_TMP                       D8000          80000         160000         320000

table scan disk non-IMC rows gotten_SLICE
=========================================
Run Type                        W10            W20            W40
MOD_QRY                       80000         160000         320000
MOD_QRY_D                     80000         160000         320000
MTH_QRY                       80000         160000         320000
RSF_QRY                   640080000     1280160000     2560320000
RSF_TMP                       80000         160000         320000

table scan disk non-IMC rows gotten_RATIO
=========================================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000              1              1              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000              1              1              1
MOD_QRY                       D8000              1              1              1
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000              1              1              1
MOD_QRY_D                     D4000              1              1              1
MOD_QRY_D                     D8000              1              1              1
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000           1001           1001           1001
RSF_QRY                       D2000           2001           2001           2001
RSF_QRY                       D4000           4001           4001           4001
RSF_QRY                       D8000           8001           8001           8001
RSF_TMP                       D1000              3              1              1
RSF_TMP                       D2000              1              1              1
RSF_TMP                       D4000              1              1              1
RSF_TMP                       D8000              1              1              1

table scan disk non-IMC rows gotten_SLICE_RATIO
===============================================
Run Type                        W10            W20            W40
MOD_QRY                           1              1              1
MOD_QRY_D                         1              1              1
MTH_QRY                           1              1              1
RSF_QRY                        8001           8001           8001
RSF_TMP                           1              1              1
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'sorts (rows)' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (CASE point_wide WH
EN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point_deep, size_deep)
 ORDER BY query_name, point_deep

sorts (rows)
============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000          20000          40000          80000
MOD_QRY                       D2000          40000          80000         160000
MOD_QRY                       D4000          80000         160000         320000
MOD_QRY                       D8000         160000         320000         640000
MOD_QRY_D                     D1000          30000          60000         120000
MOD_QRY_D                     D2000          60000         120000         240000
MOD_QRY_D                     D4000         120000         240000         480000
MOD_QRY_D                     D8000         240000         480000         960000
MTH_QRY                       D1000          10105          20209          40422
MTH_QRY                       D2000          20205          40411          80829
MTH_QRY                       D4000          40406          80815         161625
MTH_QRY                       D8000          80808         161618         323229
RSF_QRY                       D1000       10040000       20080000       40160000
RSF_QRY                       D2000       40080000       80160000      160320000
RSF_QRY                       D4000      160160000      320320000      640640000
RSF_QRY                       D8000      640320000     1280640000     2561280000
RSF_TMP                       D1000          60000         111472         185792
RSF_TMP                       D2000         113046         189128         359330
RSF_TMP                       D4000         190102         350112         677204
RSF_TMP                       D8000         359798         672820        1312532

sorts (rows)_SLICE
==================
Run Type                        W10            W20            W40
MOD_QRY                      160000         320000         640000
MOD_QRY_D                    240000         480000         960000
MTH_QRY                       80808         161618         323229
RSF_QRY                   640320000     1280640000     2561280000
RSF_TMP                      359798         672820        1312532

sorts (rows)_RATIO
==================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           1.98           1.98           1.98
MOD_QRY                       D2000           1.98           1.98           1.98
MOD_QRY                       D4000           1.98           1.98           1.98
MOD_QRY                       D8000           1.98           1.98           1.98
MOD_QRY_D                     D1000           2.97           2.97           2.97
MOD_QRY_D                     D2000           2.97           2.97           2.97
MOD_QRY_D                     D4000           2.97           2.97           2.97
MOD_QRY_D                     D8000           2.97           2.97           2.97
MTH_QRY                       D1000              1              1              1
MTH_QRY                       D2000              1              1              1
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1              1
RSF_QRY                       D1000         993.57         993.62         993.52
RSF_QRY                       D2000        1983.67        1983.62        1983.45
RSF_QRY                       D4000        3963.77        3963.62        3963.74
RSF_QRY                       D8000        7923.97        7923.87        7924.04
RSF_TMP                       D1000           5.94           5.52            4.6
RSF_TMP                       D2000           5.59           4.68           4.45
RSF_TMP                       D4000            4.7           4.33           4.19
RSF_TMP                       D8000           4.45           4.16           4.06

sorts (rows)_SLICE_RATIO
========================
Run Type                        W10            W20            W40
MOD_QRY                        1.98           1.98           1.98
MOD_QRY_D                      2.97           2.97           2.97
MTH_QRY                           1              1              1
RSF_QRY                     7923.97        7923.87        7924.04
RSF_TMP                        4.45           4.16           4.06
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'logical read bytes from cache' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (C
ASE point_wide WHEN 1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point
_deep, size_deep) ORDER BY query_name, point_deep

logical read bytes from cache
=============================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000       16826368       17080320        4792320
MOD_QRY                       D2000       15958016       17186816        5472256
MOD_QRY                       D4000       17334272       18071552        8077312
MOD_QRY                       D8000       19701760        8036352       12050432
MOD_QRY_D                     D1000       16089088       16539648        4792320
MOD_QRY_D                     D2000       17178624       17154048        5873664
MOD_QRY_D                     D4000       17670144       18694144        8282112
MOD_QRY_D                     D8000       18513920        8011776       12124160
MTH_QRY                       D1000       16670720       16777216        5431296
MTH_QRY                       D2000       17211392       17596416        5898240
MTH_QRY                       D4000       17063936       18145280        7847936
MTH_QRY                       D8000       18399232        7299072       12722176
RSF_QRY                       D1000      387252224      768524288     2070929408
RSF_QRY                       D2000     1325637632     3647430656     6337478656
RSF_QRY                       D4000     6252453888    11544403968    21098643456
RSF_QRY                       D8000    21930770432    39743684608    76724371456
RSF_TMP                       D1000      185991168      430202880      956489728
RSF_TMP                       D2000      431144960      952655872     2049122304
RSF_TMP                       D4000      955179008     2024398848     4332290048
RSF_TMP                       D8000     2030608384     4234272768    10213171200

logical read bytes from cache_SLICE
===================================
Run Type                        W10            W20            W40
MOD_QRY                    19701760        8036352       12050432
MOD_QRY_D                  18513920        8011776       12124160
MTH_QRY                    18399232        7299072       12722176
RSF_QRY                 21930770432    39743684608    76724371456
RSF_TMP                  2030608384     4234272768    10213171200

logical read bytes from cache_RATIO
===================================
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           1.05           1.03              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000           1.02              1           1.03
MOD_QRY                       D8000           1.07            1.1              1
MOD_QRY_D                     D1000              1              1              1
MOD_QRY_D                     D2000           1.08              1           1.07
MOD_QRY_D                     D4000           1.04           1.03           1.06
MOD_QRY_D                     D8000           1.01            1.1           1.01
MTH_QRY                       D1000           1.04           1.01           1.13
MTH_QRY                       D2000           1.08           1.03           1.08
MTH_QRY                       D4000              1              1              1
MTH_QRY                       D8000              1              1           1.06
RSF_QRY                       D1000          24.07          46.47         432.14
RSF_QRY                       D2000          83.07         212.63        1158.11
RSF_QRY                       D4000         366.41         638.82        2688.43
RSF_QRY                       D8000        1191.94        5445.03        6366.94
RSF_TMP                       D1000          11.56          26.01         199.59
RSF_TMP                       D2000          27.02          55.54         374.46
RSF_TMP                       D4000          55.98         112.02         552.03
RSF_TMP                       D8000         110.36         580.11         847.54

logical read bytes from cache_SLICE_RATIO
=========================================
Run Type                        W10            W20            W40
MOD_QRY                        1.07            1.1              1
MOD_QRY_D                      1.01            1.1           1.01
MTH_QRY                           1              1           1.06
RSF_QRY                     1191.94        5445.03        6366.94
RSF_TMP                      110.36         580.11         847.54
WITH wit AS (SELECT query_name, point_deep, size_deep, point_wide, stat_val f_real, Round (stat_val / Greatest (Min (stat_val) OVER (PARTITION BY point_deep, point_wide), 0.000001), 2) f_ratio FROM be
nch_run_v$stats_v WHERE stat_name = 'redo size' AND stat_type = 'STAT') SELECT text FROM (SELECT query_name, point_deep, '"' || query_name || '","D' || size_deep || '","' || Max (CASE point_wide WHEN
1 THEN f_real END) || '","' || Max (CASE point_wide WHEN 2 THEN f_real END) || '","' || Max (CASE point_wide WHEN 3 THEN f_real END) || '"' text FROM wit GROUP BY query_name, point_deep, size_deep) OR
DER BY query_name, point_deep

redo size
=========
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000         385788         386152         344756
MOD_QRY                       D2000         325816         327056         344380
MOD_QRY                       D4000         346144         325916         366016
MOD_QRY                       D8000         390064         346140         380912
MOD_QRY_D                     D1000         345476         347244         379896
MOD_QRY_D                     D2000         353168         343584         380304
MOD_QRY_D                     D4000         382596         381004         333460
MOD_QRY_D                     D8000         363040         364440         347928
MTH_QRY                       D1000         383860         381604         369580
MTH_QRY                       D2000         386196         380396         348136
MTH_QRY                       D4000         343428         342680         346828
MTH_QRY                       D8000         361868         324824         369596
RSF_QRY                       D1000         343244         343360         392176
RSF_QRY                       D2000         343888         351672         380396
RSF_QRY                       D4000         384680         351804         380492
RSF_QRY                       D8000         324972         348752         343920
RSF_TMP                       D1000        1011996        1761236        3389236
RSF_TMP                       D2000        1709376        3217776        6522596
RSF_TMP                       D4000        3027732        6021064       12715036
RSF_TMP                       D8000        5711344       11683492       25113080

redo size_SLICE
===============
Run Type                        W10            W20            W40
MOD_QRY                      390064         346140         380912
MOD_QRY_D                    363040         364440         347928
MTH_QRY                      361868         324824         369596
RSF_QRY                      324972         348752         343920
RSF_TMP                     5711344       11683492       25113080

redo size_RATIO
===============
Run Type                      Depth            W10            W20            W40
MOD_QRY                       D1000           1.12           1.12              1
MOD_QRY                       D2000              1              1              1
MOD_QRY                       D4000           1.01              1            1.1
MOD_QRY                       D8000            1.2           1.07           1.11
MOD_QRY_D                     D1000           1.01           1.01            1.1
MOD_QRY_D                     D2000           1.08           1.05            1.1
MOD_QRY_D                     D4000           1.11           1.17              1
MOD_QRY_D                     D8000           1.12           1.12           1.01
MTH_QRY                       D1000           1.12           1.11           1.07
MTH_QRY                       D2000           1.19           1.16           1.01
MTH_QRY                       D4000              1           1.05           1.04
MTH_QRY                       D8000           1.11              1           1.07
RSF_QRY                       D1000              1              1           1.14
RSF_QRY                       D2000           1.06           1.08            1.1
RSF_QRY                       D4000           1.12           1.08           1.14
RSF_QRY                       D8000              1           1.07              1
RSF_TMP                       D1000           2.95           5.13           9.83
RSF_TMP                       D2000           5.25           9.84          18.94
RSF_TMP                       D4000           8.82          18.47          38.13
RSF_TMP                       D8000          17.57          35.97          73.02

redo size_SLICE_RATIO
=====================
Run Type                        W10            W20            W40
MOD_QRY                         1.2           1.07           1.11
MOD_QRY_D                      1.12           1.12           1.01
MTH_QRY                        1.11              1           1.07
RSF_QRY                           1           1.07              1
RSF_TMP                       17.57          35.97          73.02

Timer Set: File Writer, Constructed at 23 Nov 2016 01:03:17, written at 01:03:19
================================================================================
[Timer timed: Elapsed (per call): 0.02 (0.000015), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Lines          0.00        0.00             2        0.00000        0.00000
(Other)        2.55        2.53             1        2.54700        2.53000
------- ---------- ---------- ------------ ------------- -------------
Total          2.55        2.53             3        0.84900        0.84333
------- ---------- ---------- ------------ ------------- -------------

Timer Set: Top, Constructed at 22 Nov 2016 20:46:15, written at 01:03:19
========================================================================
[Timer timed: Elapsed (per call): 0.00 (0.000000), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer          Elapsed         CPU         Calls       Ela/Call       CPU/Call
---------- ---------- ---------- ------------ ------------- -------------
Setup Data       78.31       58.41            12        6.52550        4.86750
Querying     15,341.19   15,339.75            12    1,278.43267    1,278.31250
(Other)           4.63        4.51             1        4.62500        4.51000
---------- ---------- ---------- ------------ ------------- -------------
Total        15,424.12   15,402.67            25      616.96492      616.10680
---------- ---------- ---------- ------------ ------------- -------------
Successfully completed

7458 rows selected.

Elapsed: 00:00:00.53
SQL> 
SQL> SELECT 'End: ' || To_Char(SYSDATE,'DD-MON-YYYY HH24:MI:SS') FROM DUAL
  2  /

'END:'||TO_CHAR(SYSDATE,'
-------------------------
End: 23-NOV-2016 01:03:20

Elapsed: 00:00:00.00
SQL> SPOOL OFF
</pre>
</div>

<div class="dimben-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 DIMBEN Series:</strong>
  <a href="/migrated/dimben-series-index/">Index</a>
  {% if page.dimben_prev %}
    | <a href="{{ page.dimben_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.dimben_next %}
    | <a href="{{ page.dimben_next }}">Next ▶</a>
  {% endif %}
</div>
