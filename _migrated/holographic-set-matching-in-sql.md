---
layout: post
title: "Holographic Set Matching in SQL (MDTM2)"
date: 2012-10-12
migrated: true
group: general-sql
categories: 
  - "oracle"
  - "performance"
  - "sql"
  - "subquery-factor"
tags: 
  - "hologram"
  - "holographic"
  - "listagg"
  - "matching"
  - "oracle"
  - "performance-2"
  - "sql"
  - "subquery-factor"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

This article is the second in a sequence of three dealing with a very general class of problems in SQL, and exploring various techniques to find efficient solutions. In the first article, [Master-Detail Transaction Matching in SQL (MDTM1)](https://brenpatf.github.io/migrated/master-detail-transaction-matching-in-sql/ "Master-Detail Transaction Matching in SQL (MDTM1)"), the problem was outlined and divided into two subproblems, of which the first was solved in several variant SQL statements with performance analysis. This second article, takes the most efficient method and applies two new techniques to further improve performance. The third article, [Master-Detail Transaction Reconciliation in SQL (MDTM3)](https://brenpatf.github.io/migrated/master-detail-transaction-reconciliation-in-sql-mdtm3/ "Master-Detail Transaction Reconciliation in SQL (MDTM3)"), adds a sequence of subquery factors to the best solution for the first subproblem to achieve an efficient solution to the overall problem within a single SQL statement.

_The holographic principle is a mathematical principle that the total information contained in a volume of space corresponds to an equal amount of information contained on the boundary of that space._ - [Holographic Principle](http://physics.about.com/od/physicsetoh/g/holoprinciple.htm).

In my last article, I made large performance gains in SQL queries matching sets of detail records by obtaining aggregates of the sets in a subquery factor and matching those at master level before matching the detail sets directly. The performance gain came from the fact that the aggregation is cheap compared to matching sets of records and allows many matching pair candidates to be discarded before doing the expensive direct set matching. However, all of the actually matching transactions would have been directly matched and probably more, and it occurred to me to wonder whether it might not be possible to use aggregate matching to replace detail set matching altogether.

This article develops the previous one by taking the same sample transaction matching problem and adding queries that use the Oracle 11.2 function Listagg to allow just this replacement. This is possible so long as the list-aggregated detail matching fields do not exceed 4,000 characters. If that were to happen then some other aggregation technique would be needed, perhaps a user-defined CLOB version of Listagg. However, it's possible to extend the range of applicability by aggregating identifiers smaller than the actual fields, as I'll discuss at the end.

We'll keep the fastest query from the previous article, and add three new queries:

## Query Variations

- MIN\_NE - Detail grouping in temporary table, with set matching (GTT\_NE previously)
- LAG\_SQF - Detail grouping by Listagg only in subquery factor
- LAG\_NE - Detail grouping by Listagg in subquery factor, with set matching
- LAG\_GTT - Detail grouping by Listagg only in temporary table

#### MIN_NE - Detail grouping in temporary table, with set matching (GTT\_NE previously)
```sql
  WITH tab AS (
SELECT t.owner,
       t.table_name,
       t.ROWID                   row_id,
       Sum (c.n_con)             n_det,
       Min (c.r_constraint_name) min_det,
       Max (c.r_constraint_name) max_det
  FROM tab_cp                    t
  JOIN grp_gtt                   c
    ON c.owner                   = t.owner
   AND c.table_name              = t.table_name
 GROUP BY
        t.owner,
        t.table_name,
        t.ROWID
)
SELECT
       t1.owner                  owner_1,
       t1.table_name             table_name_1,
       t2.owner                  owner_2,
       t2.table_name             table_name_2,
       t1.n_det                  n_det
  FROM tab                       t1
  JOIN tab                       t2
    ON t2.n_det                  = t1.n_det
   AND t2.min_det                = t1.min_det
   AND t2.max_det                = t1.max_det
   AND t2.row_id                 > t1.row_id
 WHERE NOT EXISTS (
SELECT 1
  FROM grp_gtt                   d1
 WHERE d1.owner                  = t1.owner
   AND d1.table_name             = t1.table_name
   AND (
   NOT EXISTS (
SELECT 1
  FROM grp_gtt                   d2
 WHERE d2.owner                  = t2.owner
   AND d2.table_name             = t2.table_name
   AND d2.r_owner                = d1.r_owner
   AND d2.r_constraint_name      = d1.r_constraint_name
   AND d2.n_con                  = d1.n_con
)))
 ORDER BY t1.owner,
       t1.table_name,
       t2.owner,
       t2.table_name
```
#### LAG\_SQF - Detail grouping by Listagg only in subquery factor
```sql
  WITH tab AS (
SELECT t.owner,
       t.table_name,
       t.ROWID                   row_id,
       Count(c.ROWID)            n_det,
       Listagg (c.r_owner||c.r_constraint_name, '') WITHIN GROUP (ORDER BY c.r_owner||c.r_constraint_name) lagg
  FROM tab_cp                    t
  JOIN con_cp                    c
    ON c.owner                   = t.owner
   AND c.table_name              = t.table_name
   AND c.constraint_type         = 'R'
 GROUP BY
        t.owner,
        t.table_name,
        t.ROWID
)
SELECT
       t1.owner                  owner_1,
       t1.table_name             table_name_1,
       t2.owner                  owner_2,
       t2.table_name             table_name_2,
       t1.n_det                  n_det
  FROM tab                       t1
  JOIN tab                       t2
    ON t2.n_det                  = t1.n_det
   AND t2.lagg                   = t1.lagg
   AND t2.row_id                 > t1.row_id
 ORDER BY t1.owner,
       t1.table_name,
       t2.owner,
       t2.table_name
```
#### LAG\_NE - Detail grouping by Listagg in subquery factor, with set matching
```sql
  WITH tab AS (
SELECT t.owner,
       t.table_name,
       t.ROWID                   row_id,
       Sum (c.n_con)             n_det,
       Listagg (c.r_owner||c.r_constraint_name, '') WITHIN GROUP (ORDER BY c.r_owner||c.r_constraint_name) lagg
  FROM tab_cp                    t
  JOIN grp_gtt                   c
    ON c.owner                   = t.owner
   AND c.table_name              = t.table_name
 GROUP BY
        t.owner,
        t.table_name,
        t.ROWID
)
SELECT
       t1.owner                  owner_1,
       t1.table_name             table_name_1,
       t2.owner                  owner_2,
       t2.table_name             table_name_2,
       t1.n_det                  n_det
  FROM tab                       t1
  JOIN tab                       t2
    ON t2.n_det                  = t1.n_det
   AND t2.lagg                   = t1.lagg
   AND t2.row_id                 > t1.row_id
 WHERE NOT EXISTS (
SELECT 1
  FROM grp_gtt                   d1
 WHERE d1.owner                  = t1.owner
   AND d1.table_name             = t1.table_name
   AND (
   NOT EXISTS (
SELECT 1
  FROM grp_gtt                   d2
 WHERE d2.owner                  = t2.owner
   AND d2.table_name             = t2.table_name
   AND d2.r_owner                = d1.r_owner
   AND d2.r_constraint_name      = d1.r_constraint_name
   AND d2.n_con                  = d1.n_con
)))
 ORDER BY t1.owner,
       t1.table_name,
       t2.owner,
       t2.table_name
```
#### LAG\_GTT - Detail grouping by Listagg only in temporary table
```sql
SELECT
       t1.owner                  owner_1,
       t1.table_name             table_name_1,
       t2.owner                  owner_2,
       t2.table_name             table_name_2,
       t1.n_det                  n_det
  FROM tab_gtt                   t1
  JOIN tab_gtt                   t2
    ON t2.n_det                  = t1.n_det
   AND t2.lagg                   = t1.lagg
   AND t2.row_id                 > t1.row_id
 ORDER BY t1.owner,
       t1.table_name,
       t2.owner,
       t2.table_name
```

The two temporary tables are as follows, with \* marking unique keys: 

### tab\_gtt

- owner\*
- table\_name\*
- row\_id
- lagg
- n\_det

A non-unique index is defined on tab\_gtt: 

#### tab\_gtt\_N1

- lagg

### grp\_gtt

- owner\*
- constraint\_name\*
- r\_owner\*
- r\_constraint\_name\*
- n\_con

## Performance Analysis

We presented four query variations above, and in this section give the results of benchmarking these queries across a 1-dimensional data domain obtained by copying the system views once, then by successive powers of two up to 32 times into my test tables described in the previous article. The problem sizes are as follows: 

### Record Counts

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21155&amp;authkey=ADdKXqkPQo8ZoEU&amp;em=2" width="600" height="200" frameborder="0" scrolling="no"></iframe>

### Timings and Statistics
 Click on the query name in the file below to jump to the execution plan for the largest data point. 

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21156&amp;authkey=AFF9iePcp3m95vk&amp;em=2" width="800" height="650" frameborder="0" scrolling="no"></iframe>

### Comparison
 
<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21154&amp;authkey=AA18moA8kKwrUtY&amp;em=2" width="600" height="300" frameborder="0" scrolling="no"></iframe>

The figures above are for the largest data pont, W32. The following points can be made:

- The query that performed best in the earlier article is now worst compared with the new queries
- The best query uses Listagg within a subquery factor and is more than 200 times faster than the worst at the largest data point
- Moving the new listagg-based subquery factor into a temporary table worsens performance because the index is not used
- The statistics tab shows that performance variation is now linear with problem size for the new Listagg queries, which is why they inevitably outperform the old one at large enough problem size

### Subquery Factors and Temporary Tables

Replacing a subquery factor by a temporary table can only help if indexed accesses are beneficial.

### Execution Plan Hash Values
 
In my last article, I noted that the plan hash values differed for all queries between data points, although the plans were essentially the same, and surmised that this was due to the subquery factor internal naming system. LAG\_GTT is the only query here that makes no use of subquery factors, and this is the only one that retains the same plan hash value, thus bearing out the surmise.

## Extending Listagg Applicability

If there are a large number of matching fields then the Listagg limit of 4,000 characters could be hit in quite a small number of details for a master. It's not difficult to write a CLOB version of Listagg, but one way of mitigating the restriction would be to aggregate not the actual matching fields, but the ranking of the set within all distinct detail sets. A further reduction in the size of the aggregated values can be obtained by storing the ranking in a high number-base, rather than base 10, as a zero-left-padded string. If the database character set is UTF8 (as is my 11.2 XE database), base 128 is possible, while extended Ascii character sets should allow base 256. The number of characters assigned to the ranking value determines how many distinct sets and how many detail records per master record are allowed with the standard Listagg function, according to the table below (for UTF8):

```
Chars  Distinct Sets  Details/Master
=====  =============  ==============
    1            128           4,000
    2         16,384           2,000
    3      2,097,152           1,333
    4    268,435,456           1,000
```

### New Query L2\_SQF

```sql
WITH rns AS (
SELECT r_owner,
       r_constraint_name,
       Row_Number () OVER (ORDER BY r_owner, r_constraint_name) - 1 rn
  FROM con_cp
 WHERE constraint_type	    = 'R'
 GROUP BY 
       r_owner,
       r_constraint_name
), rch AS ( 
SELECT r_owner,
       r_constraint_name,
       Chr (Floor (rn / 128)) ||
       Chr ((rn - 128 * Floor (rn / 128))) chr_rank
  FROM rns
), tab AS (
SELECT t.owner,
       t.table_name,
       t.ROWID                   row_id,
       Count(c.ROWID)            n_det,
       Listagg (r.chr_rank, '') WITHIN GROUP (ORDER BY r.chr_rank) lagg
  FROM tab_cp                    t
  JOIN con_cp                    c
    ON c.owner                   = t.owner
   AND c.table_name              = t.table_name
   AND c.constraint_type         = 'R'
  JOIN rch                       r
    ON r.r_owner                 = c.r_owner
   AND r.r_constraint_name       = c.r_constraint_name
 GROUP BY
        t.owner,
        t.table_name,
        t.ROWID
)
SELECT t1.owner                  owner_1,
       t1.table_name             table_name_1,
       t2.owner                  owner_2,
       t2.table_name             table_name_2,
       t1.n_det                  n_det
  FROM tab                       t1
  JOIN tab                       t2
    ON t2.lagg                   = t1.lagg
   AND t2.row_id                 > t1.row_id
 ORDER BY t1.owner,
       t1.table_name,
       t2.owner,
       t2.table_name
```

### QSDs
 
<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21160&amp;authkey=AOrjwkdObU8T85U&amp;em=2" width="402" height="700" frameborder="0" scrolling="no"></iframe>

### Record Counts
 
The same set of data points has been used for the new query (L2\_SQF) with the best of the earlier ones for comparison (LAG\_SQF). The record counts have slightly increased. Note that the constraints per table is for all constraints, not just foreign keys. 

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21158&amp;authkey=AKoka8CG0UjeaNQ&amp;em=2" width="600" height="200" frameborder="0" scrolling="no"></iframe>

### Timings and Statistics

Click on the query name in the file below to jump to the execution plan for the largest data point. 

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21159&amp;authkey=AMkSKkaFKE3ihlc&amp;em=2" width="800" height="650" frameborder="0" scrolling="no"></iframe>

### Comparison
 
<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21157&amp;authkey=ANV9OwJVp2blr8c&amp;em=2" width="600" height="170" frameborder="0" scrolling="no"></iframe>

The figures above are for the largest data pont, W32. The following points can be made:

- The new query is about 20% slower than the old one
- Performance variation remains linear with problem size for the new query
- The test problem has very few details per master, but the relative performances may change for real problems

## Conclusions

This article has used a new idea that I termed _holographic set matching_ to improve performance relative to the queries in my previous article on [Master-Detail Transaction Matching in SQL (MDTM1)](https://brenpatf.github.io/migrated/master-detail-transaction-matching-in-sql/ "Master-Detail Transaction Matching in SQL"), achieving linear time variation with size, compared with the earlier quadratic time. Although the new Oracle 11.2 function Listagg has been used, the method can by applied in earlier versions by adding a user-defined list aggregation function, which is easy to do. The third article in this sequence, [Master-Detail Transaction Reconciliation in SQL (MDTM3)](https://brenpatf.github.io/master-detail-transaction-reconciliation-in-sql-mdtm3/ "Master-Detail Transaction Reconciliation in SQL (MDTM3)"), solves the overall problem described in the first article by adding a sequence of subquery factors to the query developed above.
