---
layout: post
title: "SQL for Continuum and Contiguity Grouping"
date: 2013-11-16
migrated: true
group: general-sql
categories: 
  - "match_recognize"
  - "oracle"
  - "qsd"
  - "sql"
  - "v12"
tags: 
  - "aggregation"
  - "contiguity"
  - "continuum"
  - "grouping"
  - "match_recognize"
  - "oracle"
  - "plsql"
  - "postgres"
  - "qsd"
  - "range-based"
  - "sql"
---

A question was asked some time ago on Oracle's SQL&PL/SQL forum ([Solution design and performance - SQL or PL/SQL?](https://forums.oracle.com/ords/apexds/post/solution-design-and-performance-sql-or-pl-sql-7171)), the gist of which the poster specified thus:

> I have a set of records which keep track of a status at a given date for a range of points on a given road. Basically, I need the current bits to "shine through".

The thread, despite the open-minded title, gives a nice illustration of the odd preference that people often have for complicated PL/SQL solutions to problems amenable to simpler SQL. I provided an SQL solution in that thread for this problem, and in this article I have taken a simpler, more general form of the problem defined there, and provide SQL solutions with diagrams illustrating how they work.

The class of problem can be characterised as follows:

- Records have range fields, and we want to aggregate separately at each point within the 1-dimensional range domains (or 'continuum'), independently by grouping key
- Within the groups we want to aggregate only the first or last records, ordering by some non-key fields

The first point above can be seen as equivalent to adding in to the main grouping key an implicit field specifying the domain _leg_, i.e. the region between the break points defined by the record ranges, and splitting the records by leg. We might term this form of aggregation _continuum aggregation_, or perhaps _vertical aggregation_ if we view the range as distance, the internal ordering as by time, and visualise it graphically as in my diagrams below. Once this _vertical aggregation_ is established, it is natural to think of adding a second aggregation, that might be termed _contiguity aggregation_, or with the same visualisation, _horizontal aggregation_:

- Contiguous legs having the same values for the aggregated attributes will be grouped together

Here is an extract on grouping from [Oracle® Database SQL Language Reference](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/index.html):

> Oracle applies the aggregate functions to each group of rows and returns a single result row for each group. ... The aggregate functions MIN, MAX, SUM, AVG, COUNT, VARIANCE, and STDDEV, when followed by the KEEP keyword, can be used in conjunction with the FIRST or LAST function to operate on a set of values from a set of rows that rank as the FIRST or LAST with respect to a given sorting specification. Refer to [FIRST](http://docs.oracle.com/cd/E16655_01/server.121/e17209/functions072.htm#sthref1405) for more information.

A simple use-case for the _KEEP_ form of grouping is to list contacts with their most recent telephone number when multiple numbers are stored in a separate table. A few years ago, I realised that in most examples that I see developers write their own code instead of using the built-in constructs, and I wrote an article comparing the performance and complexity of the various alternatives for this requirement (as well as alternatives for SQL pivoting), [SQL Pivot and Prune Queries - Keeping an Eye on Performance](https://www.scribd.com/document/54433084/SQL-Pivot-and-Prune-Queries-Keeping-an-Eye-on-Performance).

**Update, 20 November 2013:** Added a Postgres version of the SQL solution.

## Test Problem


### Data Model

We take for our example problem a very simple model consisting of roads and road events where the events occur over a stretch of road at a given time, and event type forms the horizontal grouping attribute.

<img src="/migrated_images/2013/11/Road-ERD2.jpg" alt="Road - ERD2" title="Road - ERD2" />

### Test Data

```
Input data - Roads

ROAD_ID ROAD_DESC       START_POINT  END_POINT
------- --------------- ----------- ----------
      1 The Strand                0        200
      2 Piccadilly                0        100

Input data - road_events

ROAD_ID     REV_ID E_TYPE START_POINT  END_POINT E_DATE
------- ---------- ------ ----------- ---------- ---------
      1          1 OPEN             0         10 01-JAN-07
                 2 OPEN            20         50 01-JAN-08
                 3 CLOSED         130        160 01-FEB-08
                 4 CLOSED          55         85 05-JUN-08
                 5 OTHER           45        115 01-JAN-09
                 6 OPEN            60        100 12-FEB-11
                 7 CLOSED         115        145 12-FEB-12
      2          8 CLOSED          10         30 01-JAN-10
                 9 OPEN            40         50 01-JAN-11
                10 OPEN            50         70 01-JAN-12

10 rows selected.
```

The following diagrams display the data and solutions for our test problem.

<img src="/migrated_images/2013/11/Road-Road-11.jpg" alt="Road - Road 1" title="Road - Road 1" />

<img src="/migrated_images/2013/11/Road-Road-2.jpg" alt="Road - Road 2" title="Road - Road 2" />

## SQL Solution for 'Point' Problem


To start with, here is a standard SQL solution for the 'zero-dimensional' problem, where the most recent record is required for each road:

```
Zero-Dimensional Solution

ROAD_DESC          R_START      R_END    E_START      E_END E_DATE    E_TYPE
--------------- ---------- ---------- ---------- ---------- --------- ------
Piccadilly               0        100         50         70 01-JAN-12 OPEN
The Strand               0        200        115        145 12-FEB-12 CLOSED

  1  SELECT r.road_desc, r.start_point r_start, r.end_point r_end,
  2         Max (e.start_point) KEEP (DENSE_RANK LAST ORDER BY e.event_date) e_start,
  3         Max (e.end_point) KEEP (DENSE_RANK LAST ORDER BY e.event_date) e_end,
  4         Max (e.event_date) KEEP (DENSE_RANK LAST ORDER BY e.event_date) e_date,
  5         Max (e.event_type) KEEP (DENSE_RANK LAST ORDER BY e.event_date) e_type
  6    FROM road_events e
  7    JOIN roads r
  8      ON r.id = e.road_id
  9   GROUP BY r.road_desc, r.start_point, r.end_point
 10*  ORDER BY 1, 2, 3
```

The SQL is more complicated for the _continuum_ problem, and we provide two versions, that differ in the final stage, of _horizontal_ grouping by _contiguous_ event type. The first uses a differencing method popular in Oracle 11g and earlier versions for this common type of grouping problem; the second uses a new feature from Oracle 12c, row pattern matching. \[I have also used another technique for this type of grouping, in an article on contiguity and other range-based grouping problems in June 2011, [Forming Range-Based Break Groups With Advanced SQL](http://www.scribd.com/doc/57696875/Forming-Range-Based-Break-Groups-With-Advanced-SQL).\]

## SQL Solution for 'Continuum' Problem, Contiguity Grouping by Differences


### SQL (Contiguity by Differences)

```sql
WITH breaks AS  (
        SELECT road_id, start_point bp FROM road_events
         UNION
        SELECT road_id, end_point FROM road_events
         UNION
        SELECT id, start_point FROM roads
         UNION
        SELECT id, end_point FROM roads
), legs AS (
        SELECT road_id, bp leg_start, Lead (bp) OVER (PARTITION BY road_id ORDER BY bp) leg_end
          FROM breaks
), latest_events AS ( 
        SELECT l.road_id, l.leg_start, l.leg_end,
               Max (e.id) KEEP (DENSE_RANK LAST ORDER BY e.event_date) event_id,
               Nvl (Max (e.event_type) KEEP (DENSE_RANK LAST ORDER BY e.event_date), '(none)') event_type
          FROM legs l
          LEFT JOIN road_events e
            ON e.road_id = l.road_id
           AND e.start_point <= l.leg_start
	   AND e.end_point >= l.leg_end
         WHERE l.leg_end IS NOT NULL
         GROUP BY l.road_id, l.leg_start, l.leg_end
), latest_events_group AS ( 
        SELECT road_id,
               leg_start,
               leg_end,
               event_id,
               event_type,
               Dense_Rank () OVER (PARTITION BY road_id ORDER BY leg_start, leg_end) -
               Dense_Rank () OVER (PARTITION BY road_id, event_type ORDER BY leg_start, leg_end) group_no
          FROM latest_events
)
SELECT l.road_id, r.road_desc,
       Min (l.leg_start)        sec_start,
       Max (l.leg_end)          sec_end,
       l.event_type             e_type,
       l.group_no
  FROM latest_events_group l
  JOIN roads r
    ON r.id = l.road_id
 GROUP BY l.road_id,
        r.road_desc, 
        l.event_type,
        l.group_no
ORDER BY 1, 2, 3
```

#### Output
```

ROAD_ID ROAD_DESC        SEC_START    SEC_END E_TYPE   GROUP_NO
------- --------------- ---------- ---------- ------ ----------
      1 The Strand               0         10 OPEN            0
                                10         20 (none)          1
                                20         45 OPEN            1
                                45         60 OTHER           3
                                60        100 OPEN            4
                               100        115 OTHER           5
                               115        160 CLOSED          9
                               160        200 (none)         11
      2 Piccadilly               0         10 (none)          0
                                10         30 CLOSED          1
                                30         40 (none)          1
                                40         70 OPEN            3
                                70        100 (none)          3

13 rows selected.
```

### Query Structure Diagram (Contiguity by Differences)

<img src="/migrated_images/2013/11/Road-V1.5-QSD-Diff.jpg" alt="Road, V1.5 - QSD-Diff" title="Road, V1.5 - QSD-Diff" />

### Execution Plan (Contiguity by Differences)

```
--------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                       | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |               |      1 |        |     13 |00:00:00.01 |      25 |       |       |          |
|   1 |  SORT ORDER BY                  |               |      1 |      2 |     13 |00:00:00.01 |      25 |  2048 |  2048 | 2048  (0)|
|   2 |   HASH GROUP BY                 |               |      1 |      2 |     13 |00:00:00.01 |      25 |   900K|   900K| 1343K (0)|
|   3 |    MERGE JOIN                   |               |      1 |     24 |     19 |00:00:00.01 |      25 |       |       |          |
|   4 |     TABLE ACCESS BY INDEX ROWID | ROADS         |      1 |      2 |      2 |00:00:00.01 |       2 |       |       |          |
|   5 |      INDEX FULL SCAN            | ROAD_PK       |      1 |      2 |      2 |00:00:00.01 |       1 |       |       |          |
|*  6 |     SORT JOIN                   |               |      2 |     24 |     19 |00:00:00.01 |      23 |  2048 |  2048 | 2048  (0)|
|   7 |      VIEW                       |               |      1 |     24 |     19 |00:00:00.01 |      23 |       |       |          |
|   8 |       WINDOW SORT               |               |      1 |     24 |     19 |00:00:00.01 |      23 |  2048 |  2048 | 2048  (0)|
|   9 |        WINDOW NOSORT            |               |      1 |     24 |     19 |00:00:00.01 |      23 | 73728 | 73728 |          |
|  10 |         SORT GROUP BY           |               |      1 |     24 |     19 |00:00:00.01 |      23 |  4096 |  4096 | 4096  (0)|
|* 11 |          HASH JOIN OUTER        |               |      1 |     24 |     25 |00:00:00.01 |      23 |  1696K|  1696K|  540K (0)|
|* 12 |           VIEW                  |               |      1 |     24 |     19 |00:00:00.01 |      16 |       |       |          |
|  13 |            WINDOW SORT          |               |      1 |     24 |     21 |00:00:00.01 |      16 |  2048 |  2048 | 2048  (0)|
|  14 |             VIEW                |               |      1 |     24 |     21 |00:00:00.01 |      16 |       |       |          |
|  15 |              SORT UNIQUE        |               |      1 |     24 |     21 |00:00:00.01 |      16 |  2048 |  2048 | 2048  (0)|
|  16 |               UNION-ALL         |               |      1 |        |     24 |00:00:00.01 |      16 |       |       |          |
|  17 |                INDEX FULL SCAN  | ROAD_EVENT_N1 |      1 |     10 |     10 |00:00:00.01 |       1 |       |       |          |
|  18 |                INDEX FULL SCAN  | ROAD_EVENT_N3 |      1 |     10 |     10 |00:00:00.01 |       1 |       |       |          |
|  19 |                TABLE ACCESS FULL| ROADS         |      1 |      2 |      2 |00:00:00.01 |       7 |       |       |          |
|  20 |                TABLE ACCESS FULL| ROADS         |      1 |      2 |      2 |00:00:00.01 |       7 |       |       |          |
|  21 |           TABLE ACCESS FULL     | ROAD_EVENTS   |      1 |     10 |     10 |00:00:00.01 |       7 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - access("R"."ID"="L"."ROAD_ID")
       filter("R"."ID"="L"."ROAD_ID")
  11 - access("E"."ROAD_ID"="L"."ROAD_ID")
       filter(("E"."START_POINT"<="L"."LEG_START" AND "E"."END_POINT">="L"."LEG_END"))
  12 - filter("L"."LEG_END" IS NOT NULL)
```

## SQL Solution for 'Continuum' Problem, Contiguity Grouping by 12c Row Pattern Matching


### SQL (Contiguity by Pattern Matching)

```sql
WITH breaks AS  (
        SELECT road_id, start_point bp FROM road_events
         UNION
        SELECT road_id, end_point FROM road_events
         UNION
        SELECT id, start_point FROM roads
         UNION
        SELECT id, end_point FROM roads
), legs AS (
        SELECT road_id, bp leg_start, Lead (bp) OVER (PARTITION BY road_id ORDER BY bp) leg_end
          FROM breaks
), latest_events AS ( 
        SELECT l.road_id, r.road_desc, l.leg_start, l.leg_end,
               Max (e.id) KEEP (DENSE_RANK LAST ORDER BY e.event_date) event_id,
               Nvl (Max (e.event_type) KEEP (DENSE_RANK LAST ORDER BY e.event_date), '(none)') event_type
          FROM legs l
          JOIN roads r
            ON r.id = l.road_id
          LEFT JOIN road_events e
            ON e.road_id = l.road_id
           AND e.start_point <= l.leg_start
	   AND e.end_point >= l.leg_end
         WHERE l.leg_end IS NOT NULL
         GROUP BY l.road_id, r.road_desc, l.leg_start, l.leg_end
)
SELECT m.road_id, m.road_desc, m.sec_start, m.sec_end, m.event_type e_type
  FROM latest_events
 MATCH_RECOGNIZE (
   PARTITION BY road_id, road_desc
   ORDER BY leg_start, leg_end
   MEASURES FIRST (leg_start) sec_start,
            LAST (leg_end) sec_end,
            LAST (event_type) event_type
   PATTERN (strt sm*)
   DEFINE sm AS PREV(sm.event_type) = sm.event_type
 ) m
ORDER BY 1, 2, 3
```

#### Output
```
ROAD_ID ROAD_DESC        SEC_START    SEC_END E_TYPE
------- --------------- ---------- ---------- ------
      1 The Strand               0         10 OPEN
                                10         20 (none)
                                20         45 OPEN
                                45         60 OTHER
                                60        100 OPEN
                               100        115 OTHER
                               115        160 CLOSED
                               160        200 (none)
      2 Piccadilly               0         10 (none)
                                10         30 CLOSED
                                30         40 (none)
                                40         70 OPEN
                                70        100 (none)

13 rows selected.
```

### Query Structure Diagram (Contiguity by Pattern Matching)

<img src="/migrated_images/2013/11/Road-V1.5-QSD-MR.jpg" alt="Road, V1.5 - QSD-MR" title="Road, V1.5 - QSD-MR" />

### Execution Plan (Contiguity by Pattern Matching)

```
-------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                        | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                                 |               |      1 |        |     13 |00:00:00.01 |      25 |       |       |          |
|   1 |  SORT ORDER BY                                   |               |      1 |      2 |     13 |00:00:00.01 |      25 |  2048 |  2048 | 2048  (0)|
|   2 |   VIEW                                           |               |      1 |      2 |     13 |00:00:00.01 |      25 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT DETERMINISTIC FINITE AUTO|               |      1 |      2 |     13 |00:00:00.01 |      25 |  2048 |  2048 | 2048  (0)|
|   4 |     VIEW                                         |               |      1 |      2 |     19 |00:00:00.01 |      25 |       |       |          |
|   5 |      SORT GROUP BY                               |               |      1 |      2 |     19 |00:00:00.01 |      25 |  4096 |  4096 | 4096  (0)|
|*  6 |       HASH JOIN OUTER                            |               |      1 |     24 |     25 |00:00:00.01 |      25 |   987K|   987K|  525K (0)|
|   7 |        MERGE JOIN                                |               |      1 |     24 |     19 |00:00:00.01 |      18 |       |       |          |
|   8 |         TABLE ACCESS BY INDEX ROWID              | ROADS         |      1 |      2 |      2 |00:00:00.01 |       2 |       |       |          |
|   9 |          INDEX FULL SCAN                         | ROAD_PK       |      1 |      2 |      2 |00:00:00.01 |       1 |       |       |          |
|* 10 |         SORT JOIN                                |               |      2 |     24 |     19 |00:00:00.01 |      16 |  2048 |  2048 | 2048  (0)|
|* 11 |          VIEW                                    |               |      1 |     24 |     19 |00:00:00.01 |      16 |       |       |          |
|  12 |           WINDOW SORT                            |               |      1 |     24 |     21 |00:00:00.01 |      16 |  2048 |  2048 | 2048  (0)|
|  13 |            VIEW                                  |               |      1 |     24 |     21 |00:00:00.01 |      16 |       |       |          |
|  14 |             SORT UNIQUE                          |               |      1 |     24 |     21 |00:00:00.01 |      16 |  2048 |  2048 | 2048  (0)|
|  15 |              UNION-ALL                           |               |      1 |        |     24 |00:00:00.01 |      16 |       |       |          |
|  16 |               INDEX FULL SCAN                    | ROAD_EVENT_N1 |      1 |     10 |     10 |00:00:00.01 |       1 |       |       |          |
|  17 |               INDEX FULL SCAN                    | ROAD_EVENT_N3 |      1 |     10 |     10 |00:00:00.01 |       1 |       |       |          |
|  18 |               TABLE ACCESS FULL                  | ROADS         |      1 |      2 |      2 |00:00:00.01 |       7 |       |       |          |
|  19 |               TABLE ACCESS FULL                  | ROADS         |      1 |      2 |      2 |00:00:00.01 |       7 |       |       |          |
|  20 |        TABLE ACCESS FULL                         | ROAD_EVENTS   |      1 |     10 |     10 |00:00:00.01 |       7 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - access("E"."ROAD_ID"="L"."ROAD_ID")
       filter(("E"."START_POINT"<="L"."LEG_START" AND "E"."END_POINT">="L"."LEG_END"))
  10 - access("R"."ID"="L"."ROAD_ID")
       filter("R"."ID"="L"."ROAD_ID")
  11 - filter("L"."LEG_END" IS NOT NULL)

Note
-----
   - this is an adaptive plan
```

## SQL Solution for 'Continuum' Problem, Contiguity Grouping - Postgres


The Oracle v11 (and earlier versions) solution, with contiguity by differences, can be converted to work in Postgres, as shown below.

### SQL (Continuum by Row\_Number, Contiguity by Differences - Postgres)

```sql
WITH breaks AS  (
        SELECT road_id, start_point bp FROM road_events
         UNION
        SELECT road_id, end_point FROM road_events
         UNION
        SELECT id, start_point FROM roads
         UNION
        SELECT id, end_point FROM roads
), legs AS (
        SELECT road_id, bp leg_start, Lead (bp) OVER (PARTITION BY road_id ORDER BY bp) leg_end
          FROM breaks
), ranked_events AS ( 
        SELECT l.road_id, l.leg_start, l.leg_end,
               e.id event_id, Coalesce (e.event_type, '(none)') event_type,
               Row_Number() OVER (PARTITION BY l.road_id, l.leg_start ORDER BY e.event_date DESC) rnk
          FROM legs l
          LEFT JOIN road_events e
            ON e.road_id = l.road_id
           AND e.start_point <= l.leg_start            AND e.end_point >= l.leg_end
         WHERE l.leg_end IS NOT NULL
), latest_events_group AS ( 
        SELECT road_id,
               leg_start,
               leg_end,
               event_id,
               event_type,
               Dense_Rank () OVER (PARTITION BY road_id ORDER BY leg_start, leg_end) -
               Dense_Rank () OVER (PARTITION BY road_id, event_type ORDER BY leg_start, leg_end) group_no
          FROM ranked_events
         WHERE rnk = 1
)
SELECT l.road_id, r.road_desc,
       Min (l.leg_start)        sec_start,
       Max (l.leg_end)          sec_end,
       l.event_type             e_type,
       l.group_no
  FROM latest_events_group l
  JOIN roads r
    ON r.id = l.road_id
 GROUP BY l.road_id,
        r.road_desc, 
        l.event_type,
        l.group_no
ORDER BY 1, 2, 3;
```

#### Continuum/contiguity Solution with Row_Number...
```
 road_id | road_desc  | sec_start | sec_end | e_type | group_no 
---------+------------+-----------+---------+--------+----------
       1 | The Strand |         0 |      10 | OPEN   |        0
       1 | The Strand |        10 |      20 | (none) |        1
       1 | The Strand |        20 |      45 | OPEN   |        1
       1 | The Strand |        45 |      60 | OTHER  |        3
       1 | The Strand |        60 |     100 | OPEN   |        4
       1 | The Strand |       100 |     115 | OTHER  |        5
       1 | The Strand |       115 |     160 | CLOSED |        9
       1 | The Strand |       160 |     200 | (none) |       11
       2 | Piccadilly |         0 |      10 | (none) |        0
       2 | Piccadilly |        10 |      30 | CLOSED |        1
       2 | Piccadilly |        30 |      40 | (none) |        1
       2 | Piccadilly |        40 |      70 | OPEN   |        3
       2 | Piccadilly |        70 |     100 | (none) |        3
(13 rows)

SELECT Version();

Postgres version...
                           version                           
-------------------------------------------------------------
 PostgreSQL 9.3.1, compiled by Visual C++ build 1600, 64-bit
(1 row)
```

### Notes on Conversion of SQL to Postgres

- Postgres does not have an exact equivalent of Oracle's KEEP/FIRST grouping functionality, but it can be emulated via the analytic function Row\_Number within a subquery
- Coalesce, which is also available in Oracle, replaces Nvl, which does not exist in Postgres
- Oracle's execution plans were obtained using DBMS\_XPlan after running the queries, while the Postgres version was obtained by running the query prefaced by 'EXPLAIN ANALYZE '
- There is no Postgres equivalent of Oracle v12's row pattern matching

### Query Structure Diagram (_Continuum by Row\_Number, Contiguity by Differences - Postgres_)

<img src="/migrated_images/2013/11/Road-V1.5-QSD-PG.jpg" alt="Road, V1.5 - QSD-PG" title="Road, V1.5 - QSD-PG" />

### Execution Plan (Continuum by Row\_Number, Contiguity by Differences - Postgres)

Prefacing the query with 'EXPLAIN ANALYZE ' gives:

```
Explaining Continuum/contiguity Solution with Row_Number...
                                                                 QUERY PLAN                                                                  
---------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=619.45..619.47 rows=8 width=270) (actual time=0.235..0.235 rows=13 loops=1)
   Sort Key: l.road_id, r.road_desc, (min(l.leg_start))
   Sort Method: quicksort  Memory: 26kB
   CTE breaks
     ->  HashAggregate  (cost=79.50..95.30 rows=1580 width=8) (actual time=0.013..0.020 rows=21 loops=1)
           ->  Append  (cost=0.00..71.60 rows=1580 width=8) (actual time=0.002..0.006 rows=24 loops=1)
                 ->  Seq Scan on road_events  (cost=0.00..14.80 rows=480 width=8) (actual time=0.001..0.003 rows=10 loops=1)
                 ->  Seq Scan on road_events road_events_1  (cost=0.00..14.80 rows=480 width=8) (actual time=0.000..0.002 rows=10 loops=1)
                 ->  Seq Scan on roads  (cost=0.00..13.10 rows=310 width=8) (actual time=0.000..0.001 rows=2 loops=1)
                 ->  Seq Scan on roads roads_1  (cost=0.00..13.10 rows=310 width=8) (actual time=0.000..0.000 rows=2 loops=1)
   CTE legs
     ->  WindowAgg  (cost=115.54..147.14 rows=1580 width=8) (actual time=0.030..0.041 rows=21 loops=1)
           ->  Sort  (cost=115.54..119.49 rows=1580 width=8) (actual time=0.028..0.029 rows=21 loops=1)
                 Sort Key: breaks.road_id, breaks.bp
                 Sort Method: quicksort  Memory: 25kB
                 ->  CTE Scan on breaks  (cost=0.00..31.60 rows=1580 width=8) (actual time=0.014..0.023 rows=21 loops=1)
   CTE ranked_events
     ->  WindowAgg  (cost=290.71..326.08 rows=1572 width=138) (actual time=0.089..0.104 rows=25 loops=1)
           ->  Sort  (cost=290.71..294.64 rows=1572 width=138) (actual time=0.088..0.089 rows=25 loops=1)
                 Sort Key: l_1.road_id, l_1.leg_start, e.event_date
                 Sort Method: quicksort  Memory: 26kB
                 ->  Hash Left Join  (cost=20.80..207.25 rows=1572 width=138) (actual time=0.044..0.079 rows=25 loops=1)
                       Hash Cond: (l_1.road_id = e.road_id)
                       Join Filter: ((e.start_point <= l_1.leg_start) AND (e.end_point >= l_1.leg_end))
                       Rows Removed by Join Filter: 89
                       ->  CTE Scan on legs l_1  (cost=0.00..31.60 rows=1572 width=12) (actual time=0.031..0.048 rows=19 loops=1)
                             Filter: (leg_end IS NOT NULL)
                             Rows Removed by Filter: 2
                       ->  Hash  (cost=14.80..14.80 rows=480 width=138) (actual time=0.005..0.005 rows=10 loops=1)
                             Buckets: 1024  Batches: 1  Memory Usage: 1kB
                             ->  Seq Scan on road_events e  (cost=0.00..14.80 rows=480 width=138) (actual time=0.001..0.002 rows=10 loops=1)
   CTE latest_events_group
     ->  WindowAgg  (cost=35.79..36.01 rows=8 width=48) (actual time=0.165..0.181 rows=19 loops=1)
           ->  Sort  (cost=35.79..35.81 rows=8 width=48) (actual time=0.164..0.165 rows=19 loops=1)
                 Sort Key: ranked_events.road_id, ranked_events.event_type, ranked_events.leg_start, ranked_events.leg_end
                 Sort Method: quicksort  Memory: 26kB
                 ->  WindowAgg  (cost=35.49..35.67 rows=8 width=48) (actual time=0.124..0.138 rows=19 loops=1)
                       ->  Sort  (cost=35.49..35.51 rows=8 width=48) (actual time=0.123..0.124 rows=19 loops=1)
                             Sort Key: ranked_events.road_id, ranked_events.leg_start, ranked_events.leg_end
                             Sort Method: quicksort  Memory: 26kB
                             ->  CTE Scan on ranked_events  (cost=0.00..35.37 rows=8 width=48) (actual time=0.090..0.117 rows=19 loops=1)
                                   Filter: (rnk = 1)
                                   Rows Removed by Filter: 6
   ->  HashAggregate  (cost=14.72..14.80 rows=8 width=270) (actual time=0.218..0.221 rows=13 loops=1)
         ->  Hash Join  (cost=0.26..14.60 rows=8 width=270) (actual time=0.203..0.207 rows=19 loops=1)
               Hash Cond: (r.id = l.road_id)
               ->  Seq Scan on roads r  (cost=0.00..13.10 rows=310 width=222) (actual time=0.002..0.002 rows=2 loops=1)
               ->  Hash  (cost=0.16..0.16 rows=8 width=52) (actual time=0.192..0.192 rows=19 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 2kB
                     ->  CTE Scan on latest_events_group l  (cost=0.00..0.16 rows=8 width=52) (actual time=0.167..0.190 rows=19 loops=1)
 Total runtime: 0.432 ms
(51 rows)
```

## Code

[Code, in GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/sql-for-continuum-and-contiguity-grouping)