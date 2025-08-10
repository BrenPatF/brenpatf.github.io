---
layout: post
title: "SQL for the Travelling Salesman Problem"
date: 2013-07-14
migrated: true
group: recursive-sql
categories: 
  - "analytics"
  - "oracle"
  - "performance"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "combinatorial"
  - "performance-2"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "tsp"
---

_'The travelling salesman problem (TSP) or travelling salesperson problem asks the following question: Given a list of cities and the distances between each pair of cities, what is the shortest possible route that visits each city exactly once and returns to the origin city? It is an NP-hard problem in combinatorial optimization, important in operations research and theoretical computer science.'_ - [Travelling salesman problem](http://en.wikipedia.org/wiki/Travelling_salesman_problem).

I posted a couple of articles recently on approximate solution methods for _'hard'_ combinatorial problems using SQL, [SQL for the Balanced Number Partitioning Problem](https://brenpatf.github.io/migrated/sql-for-the-balanced-number-partitioning-problem/) and [SQL for the Fantasy Football Knapsack Problem](http://aprogrammerwrites.eu/?p=878), and I wondered whether similar techniques could be applied to TSP. This article provides my answer.

**Updated, 20 February 2016**: Added attachment with the input data files, DDL and raw results at the end.

**Update, 12 November 2017:** I have added all the scripts necessary to run the SQL against the two example datasets to my GitHub repo: [Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos). I also add a link to a recent useful primer on TSP: [The Trials And Tribulations Of The Traveling Salesman](https://dev.to/vaidehijoshi/the-trials-and-tribulations-of-the-traveling-salesman-30c).

## Test Problems

I used two test problems, the first being a simple constructed example of five towns arranged in an M-shape. The second is a more realistic problem based on 312 towns in USA and Canada, the data for which I found on a university web site.

### Test Problem 1: Emland

The first problem is small enough to verify the solutions manually.

#### The Short Route 
<img src="/migrated_images/2013/07/TSP-v1.0-Short.jpg" alt="TSP, v1.0 - Short" title="TSP, v1.0 - Short" />

#### The Long Route 
<img src="/migrated_images/2013/07/TSP-v1.0-Long.jpg" alt="TSP, v1.0 - Long" title="TSP, v1.0 - Long" />

### Test Problem 2: USCA312

I got the second problem here, [City Distance Datasets](http://people.sc.fsu.edu/~jburkardt/datasets/cities/cities.html), described thus: _'USCA312 describes 312 cities in the US and Canada. Distances between the city are computed from latitude and longitude, not from road mileage'_

I took the distances based on the x-y values given, using the simple Euclidean distance formula. I am aware that this will not give accurate distances, but it seemed the simplest approach as the provided distances were in a format not so easy to load into a database, and I am interested only in the technical aspects.

## SQL Solution with Recursive Subquery Factoring

### SQL

```sql
WITH count_towns AS (
SELECT Count(*) n_towns FROM towns
), dist_from_root AS (/* XTSP  */
SELECT a, b, dst, Row_Number () OVER (ORDER BY :SIGN * dst) rnk_by_dst, Count(DISTINCT a) OVER () + 1 n_towns
  FROM distances
 WHERE b > a
),  rsf (root, root_leg, path_rnk, nxt_id, lev, tot_price, path, n_towns) AS (
SELECT a, a || ' -> ' || b, 0, d.b, 2, d.dst, 
       CAST ('|' || LPad (d.a, 3, '0') || '|' || LPad (d.b, 3, '0') AS VARCHAR2(4000)) path,
       d.n_towns
  FROM dist_from_root d
 WHERE d.rnk_by_dst <= :KEEP_NUM_ROOT
 UNION ALL
SELECT r.root,
       r.root_leg,
       Row_Number() OVER (PARTITION BY r.root_leg ORDER BY :SIGN * (r.tot_price + d.dst)),
       d.b,
       r.lev + 1,
       r.tot_price + d.dst,
       r.path || '|' || LPad (d.b, 3, '0'),
       r.n_towns
  FROM rsf r
  JOIN distances d
    ON d.a = r.nxt_id
   AND r.path NOT LIKE '%' || '|' || LPad (d.b, 3, '0') || '%'
 WHERE r.path_rnk <= :KEEP_NUM
), circuits AS (
SELECT r.root_leg,
       Row_Number() OVER (PARTITION BY r.root_leg ORDER BY :SIGN * (r.tot_price + d.dst)) path_rnk,
       r.tot_price + d.dst tot_price,
       r.path || '|' || LPad (d.b, 3, '0') path
  FROM rsf r
  JOIN distances d
    ON d.a = r.nxt_id
   AND d.b = r.root
 WHERE r.lev = r.n_towns
   AND r.path_rnk <= :KEEP_NUM
), top_n_paths AS (
SELECT root_leg,
       tot_price,
       path,
       path_rnk,
       town_index
  FROM circuits
  CROSS JOIN (SELECT LEVEL town_index FROM count_towns c CONNECT BY LEVEL <= c.n_towns + 1)
 WHERE path_rnk <= :KEEP_NUM
), top_n_sets AS (
SELECT root_leg,
       tot_price,
       path,
       path_rnk,
       town_index,
       To_Number (Substr (path, (town_index - 1) * 4 + 2, 3)) town_id,
       Lag (To_Number (Substr (path, (town_index - 1) * 4 + 2, 3))) OVER (PARTITION BY root_leg, path_rnk ORDER BY town_index) town_id_prior
  FROM top_n_paths
)
SELECT /*+ GATHER_PLAN_STATISTICS */
       top.root_leg,
       top.path_rnk,
       Round (top.tot_price, 2) tot_dist,
       top.town_id,
       twn.name,
       Round (dst.dst, 2) leg_dist,
       Round (Sum (dst.dst) OVER (PARTITION BY root_leg, path_rnk ORDER BY town_index), 2) cum_dist
  FROM top_n_sets top
  JOIN towns twn
    ON twn.id = top.town_id
  LEFT JOIN distances dst
    ON dst.a = top.town_id_prior
   AND dst.b = top.town_id
ORDER BY top.root_leg, top.path_rnk, top.town_index
```

### How It Works

#### Bind Variables 

- SIGN: 1 means minimise distance; -1, maximise it
- KEEP\_NUM\_ROOT: number of best anchor records (root legs) to retain (throughout)
- KEEP\_NUM: number of best records from previous iteration to retain, partitioning by root leg

The solution approach is based on the idea I had in my last article whereby recursive subquery factoring is at the heart of the method, and analytic row numbering is used to reduce searching to manageable proportions at each iteration. The bind variables control the level of searching desired.

- count\_towns subquery - simply counts the total number of towns
- dist\_from\_root subquery - gets the distance between each pair of towns, without duplication, and ranks the pairs by distance
- rsf subquery, anchor branch - selects the KEEP\_NUM\_ROOT best legs as roots for the recursion
- rsf subquery, recursive branch - joins up to KEEP\_NUM best records from last iteration all possible town via distances table, using pattern matching to exclude towns already visited
- The recursive branch stores the towns visited in a delimited string
- Analytic funtion Row\_Number is used to rank the recursive branch result set at each iteration, partitioning by root leg for use in the next iteration
- circuits subquery - completes the circuit by joining the root leg via the distances table, doing the same constraint by rank and analytic ranking as the recursive branch
- top\_n\_paths subquery - uses a standard row generator subquery to generate indexed rows for the number of towns in the final path (my where clause looks redundant as I write)
- top\_n\_sets subquery - extracts the town id from the path using the index derived in the previous subquery, and gets the previous town id using analytic Lag
- Main query - joins towns table to get the town name and distances table for leg distances

## Results

### Test Problem 1: Emland

I used keep values of 5 and 5 and solved the shortest routes problem in 0.14 seconds, and the longest in 0.16 seconds.

#### Shortest Routes
<div class="scrollbox">

<pre>
ROOT_LEG       PATH_RNK   TOT_DIST    TOWN_ID NAME                             LEG_DIST   CUM_DIST
------------ ---------- ---------- ---------- ------------------------------ ---------- ----------
1 -> 2                1      10.94          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            4 Right Peak                              2       4.24
                                            5 Right Floor                          2.24       6.47
                                            3 Midfield                             2.24       8.71
                                            1 Left Floor                           2.24      10.94
                      2       11.3          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            3 Midfield                             1.41       3.65
                                            4 Right Peak                           1.41       5.06
                                            5 Right Floor                          2.24        7.3
                                            1 Left Floor                              4       11.3
                      3      11.73          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            5 Right Floor                          3.61       5.84
                                            4 Right Peak                           2.24       8.08
                                            3 Midfield                             1.41       9.49
                                            1 Left Floor                           2.24      11.73
                      4      11.73          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            3 Midfield                             1.41       3.65
                                            5 Right Floor                          2.24       5.89
                                            4 Right Peak                           2.24       8.12
                                            1 Left Floor                           3.61      11.73
                      5      11.89          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            4 Right Peak                              2       4.24
                                            3 Midfield                             1.41       5.65
                                            5 Right Floor                          2.24       7.89
                                            1 Left Floor                              4      11.89
2 -> 3                1       11.3          2 Left Peak
                                            3 Midfield                             1.41       1.41
                                            4 Right Peak                           1.41       2.83
                                            5 Right Floor                          2.24       5.06
                                            1 Left Floor                              4       9.06
                                            2 Left Peak                            2.24       11.3
                      2      11.73          2 Left Peak
                                            3 Midfield                             1.41       1.41
                                            5 Right Floor                          2.24       3.65
                                            4 Right Peak                           2.24       5.89
                                            1 Left Floor                           3.61       9.49
                                            2 Left Peak                            2.24      11.73
                      3       13.1          2 Left Peak
                                            3 Midfield                             1.41       1.41
                                            1 Left Floor                           2.24       3.65
                                            4 Right Peak                           3.61       7.26
                                            5 Right Floor                          2.24       9.49
                                            2 Left Peak                            3.61       13.1
                      4      13.26          2 Left Peak
                                            3 Midfield                             1.41       1.41
                                            5 Right Floor                          2.24       3.65
                                            1 Left Floor                              4       7.65
                                            4 Right Peak                           3.61      11.26
                                            2 Left Peak                               2      13.26
                      5      14.04          2 Left Peak
                                            3 Midfield                             1.41       1.41
                                            4 Right Peak                           1.41       2.83
                                            1 Left Floor                           3.61       6.43
                                            5 Right Floor                             4      10.43
                                            2 Left Peak                            3.61      14.04
2 -> 4                1      10.94          2 Left Peak
                                            4 Right Peak                              2          2
                                            5 Right Floor                          2.24       4.24
                                            3 Midfield                             2.24       6.47
                                            1 Left Floor                           2.24       8.71
                                            2 Left Peak                            2.24      10.94
                      2      11.89          2 Left Peak
                                            4 Right Peak                              2          2
                                            3 Midfield                             1.41       3.41
                                            5 Right Floor                          2.24       5.65
                                            1 Left Floor                              4       9.65
                                            2 Left Peak                            2.24      11.89
                      3      11.89          2 Left Peak
                                            4 Right Peak                              2          2
                                            5 Right Floor                          2.24       4.24
                                            1 Left Floor                              4       8.24
                                            3 Midfield                             2.24      10.47
                                            2 Left Peak                            1.41      11.89
                      4      13.26          2 Left Peak
                                            4 Right Peak                              2          2
                                            3 Midfield                             1.41       3.41
                                            1 Left Floor                           2.24       5.65
                                            5 Right Floor                             4       9.65
                                            2 Left Peak                            3.61      13.26
                      5      13.68          2 Left Peak
                                            4 Right Peak                              2          2
                                            1 Left Floor                           3.61       5.61
                                            3 Midfield                             2.24       7.84
                                            5 Right Floor                          2.24      10.08
                                            2 Left Peak                            3.61      13.68
3 -> 4                1       11.3          3 Midfield
                                            4 Right Peak                           1.41       1.41
                                            5 Right Floor                          2.24       3.65
                                            1 Left Floor                              4       7.65
                                            2 Left Peak                            2.24       9.89
                                            3 Midfield                             1.41       11.3
                      2      11.73          3 Midfield
                                            4 Right Peak                           1.41       1.41
                                            5 Right Floor                          2.24       3.65
                                            2 Left Peak                            3.61       7.26
                                            1 Left Floor                           2.24       9.49
                                            3 Midfield                             2.24      11.73
                      3      11.89          3 Midfield
                                            4 Right Peak                           1.41       1.41
                                            2 Left Peak                               2       3.41
                                            1 Left Floor                           2.24       5.65
                                            5 Right Floor                             4       9.65
                                            3 Midfield                             2.24      11.89
                      4       13.1          3 Midfield
                                            4 Right Peak                           1.41       1.41
                                            1 Left Floor                           3.61       5.02
                                            2 Left Peak                            2.24       7.26
                                            5 Right Floor                          3.61      10.86
                                            3 Midfield                             2.24       13.1
                      5      13.26          3 Midfield
                                            4 Right Peak                           1.41       1.41
                                            2 Left Peak                               2       3.41
                                            5 Right Floor                          3.61       7.02
                                            1 Left Floor                              4      11.02
                                            3 Midfield                             2.24      13.26
3 -> 5                1      10.94          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            4 Right Peak                           2.24       4.47
                                            2 Left Peak                               2       6.47
                                            1 Left Floor                           2.24       8.71
                                            3 Midfield                             2.24      10.94
                      2      11.73          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            4 Right Peak                           2.24       4.47
                                            1 Left Floor                           3.61       8.08
                                            2 Left Peak                            2.24      10.31
                                            3 Midfield                             1.41      11.73
                      3      11.89          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            1 Left Floor                              4       6.24
                                            2 Left Peak                            2.24       8.47
                                            4 Right Peak                              2      10.47
                                            3 Midfield                             1.41      11.89
                      4       13.1          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            2 Left Peak                            3.61       5.84
                                            1 Left Floor                           2.24       8.08
                                            4 Right Peak                           3.61      11.68
                                            3 Midfield                             1.41       13.1
                      5      13.68          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            2 Left Peak                            3.61       5.84
                                            4 Right Peak                              2       7.84
                                            1 Left Floor                           3.61      11.45
                                            3 Midfield                             2.24      13.68

150 rows selected.

Elapsed: 00:00:00.14
</pre>
</div>

#### Longest Routes
<div class="scrollbox">

<pre>
ROOT_LEG       PATH_RNK   TOT_DIST    TOWN_ID NAME                             LEG_DIST   CUM_DIST
------------ ---------- ---------- ---------- ------------------------------ ---------- ----------
1 -> 2                1       13.1          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            5 Right Floor                          3.61       5.84
                                            3 Midfield                             2.24       8.08
                                            4 Right Peak                           1.41       9.49
                                            1 Left Floor                           3.61       13.1
                      2      11.89          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            4 Right Peak                              2       4.24
                                            3 Midfield                             1.41       5.65
                                            5 Right Floor                          2.24       7.89
                                            1 Left Floor                              4      11.89
                      3      11.73          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            5 Right Floor                          3.61       5.84
                                            4 Right Peak                           2.24       8.08
                                            3 Midfield                             1.41       9.49
                                            1 Left Floor                           2.24      11.73
                      4      11.73          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            3 Midfield                             1.41       3.65
                                            5 Right Floor                          2.24       5.89
                                            4 Right Peak                           2.24       8.12
                                            1 Left Floor                           3.61      11.73
                      5      10.94          1 Left Floor
                                            2 Left Peak                            2.24       2.24
                                            4 Right Peak                              2       4.24
                                            5 Right Floor                          2.24       6.47
                                            3 Midfield                             2.24       8.71
                                            1 Left Floor                           2.24      10.94
1 -> 4                1      13.68          1 Left Floor
                                            4 Right Peak                           3.61       3.61
                                            2 Left Peak                               2       5.61
                                            5 Right Floor                          3.61       9.21
                                            3 Midfield                             2.24      11.45
                                            1 Left Floor                           2.24      13.68
                      2      13.26          1 Left Floor
                                            4 Right Peak                           3.61       3.61
                                            2 Left Peak                               2       5.61
                                            3 Midfield                             1.41       7.02
                                            5 Right Floor                          2.24       9.26
                                            1 Left Floor                              4      13.26
                      3       13.1          1 Left Floor
                                            4 Right Peak                           3.61       3.61
                                            5 Right Floor                          2.24       5.84
                                            2 Left Peak                            3.61       9.45
                                            3 Midfield                             1.41      10.86
                                            1 Left Floor                           2.24       13.1
                      4       13.1          1 Left Floor
                                            4 Right Peak                           3.61       3.61
                                            3 Midfield                             1.41       5.02
                                            5 Right Floor                          2.24       7.26
                                            2 Left Peak                            3.61      10.86
                                            1 Left Floor                           2.24       13.1
                      5      11.73          1 Left Floor
                                            4 Right Peak                           3.61       3.61
                                            5 Right Floor                          2.24       5.84
                                            3 Midfield                             2.24       8.08
                                            2 Left Peak                            1.41       9.49
                                            1 Left Floor                           2.24      11.73
1 -> 5                1      14.04          1 Left Floor
                                            5 Right Floor                             4          4
                                            2 Left Peak                            3.61       7.61
                                            3 Midfield                             1.41       9.02
                                            4 Right Peak                           1.41      10.43
                                            1 Left Floor                           3.61      14.04
                      2      13.26          1 Left Floor
                                            5 Right Floor                             4          4
                                            2 Left Peak                            3.61       7.61
                                            4 Right Peak                              2       9.61
                                            3 Midfield                             1.41      11.02
                                            1 Left Floor                           2.24      13.26
                      3      13.26          1 Left Floor
                                            5 Right Floor                             4          4
                                            3 Midfield                             2.24       6.24
                                            2 Left Peak                            1.41       7.65
                                            4 Right Peak                              2       9.65
                                            1 Left Floor                           3.61      13.26
                      4      11.89          1 Left Floor
                                            5 Right Floor                             4          4
                                            4 Right Peak                           2.24       6.24
                                            2 Left Peak                               2       8.24
                                            3 Midfield                             1.41       9.65
                                            1 Left Floor                           2.24      11.89
                      5      11.89          1 Left Floor
                                            5 Right Floor                             4          4
                                            3 Midfield                             2.24       6.24
                                            4 Right Peak                           1.41       7.65
                                            2 Left Peak                               2       9.65
                                            1 Left Floor                           2.24      11.89
2 -> 5                1      14.04          2 Left Peak
                                            5 Right Floor                          3.61       3.61
                                            1 Left Floor                              4       7.61
                                            4 Right Peak                           3.61      11.21
                                            3 Midfield                             1.41      12.63
                                            2 Left Peak                            1.41      14.04
                      2      13.68          2 Left Peak
                                            5 Right Floor                          3.61       3.61
                                            3 Midfield                             2.24       5.84
                                            1 Left Floor                           2.24       8.08
                                            4 Right Peak                           3.61      11.68
                                            2 Left Peak                               2      13.68
                      3      13.26          2 Left Peak
                                            5 Right Floor                          3.61       3.61
                                            1 Left Floor                              4       7.61
                                            3 Midfield                             2.24       9.84
                                            4 Right Peak                           1.41      11.26
                                            2 Left Peak                               2      13.26
                      4       13.1          2 Left Peak
                                            5 Right Floor                          3.61       3.61
                                            4 Right Peak                           2.24       5.84
                                            1 Left Floor                           3.61       9.45
                                            3 Midfield                             2.24      11.68
                                            2 Left Peak                            1.41       13.1
                      5      11.73          2 Left Peak
                                            5 Right Floor                          3.61       3.61
                                            4 Right Peak                           2.24       5.84
                                            3 Midfield                             1.41       7.26
                                            1 Left Floor                           2.24       9.49
                                            2 Left Peak                            2.24      11.73
3 -> 5                1      13.68          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            2 Left Peak                            3.61       5.84
                                            4 Right Peak                              2       7.84
                                            1 Left Floor                           3.61      11.45
                                            3 Midfield                             2.24      13.68
                      2      13.26          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            1 Left Floor                              4       6.24
                                            4 Right Peak                           3.61       9.84
                                            2 Left Peak                               2      11.84
                                            3 Midfield                             1.41      13.26
                      3       13.1          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            2 Left Peak                            3.61       5.84
                                            1 Left Floor                           2.24       8.08
                                            4 Right Peak                           3.61      11.68
                                            3 Midfield                             1.41       13.1
                      4      11.89          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            1 Left Floor                              4       6.24
                                            2 Left Peak                            2.24       8.47
                                            4 Right Peak                              2      10.47
                                            3 Midfield                             1.41      11.89
                      5      11.73          3 Midfield
                                            5 Right Floor                          2.24       2.24
                                            4 Right Peak                           2.24       4.47
                                            1 Left Floor                           3.61       8.08
                                            2 Left Peak                            2.24      10.31
                                            3 Midfield                             1.41      11.73

150 rows selected.

Elapsed: 00:00:00.16
</pre>
</div>

### Summary of Results

Here we list the best solutions found for each root leg. \* denotes optimum.

#### Shortest Routes
```
ROOT_LEG       PATH_RNK   TOT_DIST
------------ ---------- ----------
1 -> 2                1      10.94*
2 -> 3                1       11.3
2 -> 4                1      10.94*
3 -> 4                1       11.3
3 -> 5                1      10.94*
```

#### Longest Routes
```
ROOT_LEG       PATH_RNK   TOT_DIST
------------ ---------- ----------
1 -> 2                1       13.1
1 -> 4                1      13.68
1 -> 5                1      14.04*
2 -> 5                1      14.04*
3 -> 5                1      13.68
```

### Test Problem 2: USCA312

I used keep values of 2 and 2 and solved the problems in around 85 seconds.

#### Shortest Routes
<div class="scrollbox">

<pre>
ROOT_LEG       PATH_RNK   TOT_DIST    TOWN_ID NAME                             LEG_DIST   CUM_DIST
------------ ---------- ---------- ---------- ------------------------------ ---------- ----------
184 -> 208            1   55744.86        184 Norfolk, VA
                                          208 Portsmouth, VA                       1.19       1.19
                                          294 Washington, DC                     151.18     152.37
                                           18 Baltimore, MD                       40.06     192.43
                                          141 Lancaster, PA                       55.83     248.26
                                          129 Johnstown, PA                       16.64      264.9
                                          217 Reading, PA                          27.2      292.1
                                          302 Wilmington, DE                      48.48     340.58
                                          198 Philadelphia, PA                    30.05     370.63
                                          283 Trenton, NJ                         34.35     404.99
                                           83 Elizabeth, NJ                       48.02     453.01
                                          177 Newark, NJ                           5.61     458.62
                                          128 Jersey City, NJ                      6.57     465.19
                                          181 New York, NY                         5.04     470.23
                                          194 Paterson, NJ                        18.08     488.32
                                          299 White Plains, NY                    29.39     517.71
                                          266 Stamford, CT                        15.36     533.07
                                           39 Bridgeport, CT                      24.55     557.61
                                          179 New Haven, CT                       21.46     579.08
                                          163 Meriden, CT                         17.96     597.04
                                          178 New Britain,CT                       8.71     605.75
                                          117 Hartford, CT                         9.62     615.37
                                          263 Springfield, MA                     24.25     639.62
                                          202 Pittsfield, MA                      51.31     690.93
                                          285 Troy, NY                            36.34     727.27
                                            3 Albany, NY                           6.88     734.16
                                          251 Schenectady, NY                     16.88     751.04
                                          289 Utica, NY                           91.52     842.56
                                          272 Syracuse, NY                        63.31     905.86
                                           29 Binghamtom, NY                      67.49     973.35
                                          301 Wilkes-Barre, PA                    58.97    1032.33
                                            6 Allentown, PA                       51.68       1084
                                          116 Harrisburg, PA                      99.07    1183.07
                                           13 Atlantic City, NJ                  181.31    1364.38
                                           50 Central Islip, NY                  129.74    1494.12
                                          210 Providence, RI                     142.75    1636.87
                                           91 Fall River, MA                      19.72    1656.59
                                           40 Brockton, MA                        28.03    1684.61
                                           34 Boston, MA                          19.21    1703.83
                                           46 Cambridge, MA                        3.36    1707.19
                                          155 Lowell, MA                          23.03    1730.22
                                          145 Lawrence, MA                        11.73    1741.95
                                          159 Manchester, NH                      28.36     1770.3
                                           67 Concord, NH                         15.76    1786.06
                                          307 Worcester, MA                       67.84     1853.9
                                           38 Brattleboro, VT                     66.17    1920.07
                                          171 Montpelier, VT                      97.37    2017.43
                                           43 Burlington, VT                      46.45    2063.89
                                          172 Montreal, QC                        97.19    2161.08
                                          284 Trois-Rivieres, QC                  98.03     2259.1
                                          254 Sherbrooke, QC                      93.08    2352.18
                                          205 Portland, ME                       157.79    2509.97
                                          207 Portsmouth, NH                      53.74    2563.71
                                           15 Augusta, ME                        109.27    2672.99
                                           19 Bangor, ME                          77.07    2750.06
                                           98 Fredericton, NB                    171.89    2921.95
                                          229 Saint John, NB                      58.53    2980.48
                                          169 Moncton, NB                         99.94    3080.42
                                          113 Halifax, NS                        117.55    3197.97
                                           55 Charlottetown, PE                  100.98    3298.95
                                          271 Syndey, NS                         212.55     3511.5
                                          230 Saint John's, NF                   514.07    4025.57
                                          213 Quebec City, QC                   1290.47    5316.03
                                          191 Ottawa, ON                         400.01    5716.04
                                          137 Kingston, ON                        85.39    5801.42
                                           24 Belleville, ON                      49.28    5850.71
                                          197 Peterborough, ON                    72.87    5923.57
                                          282 Toronto, ON                         82.62    6006.19
                                           42 Burlington, ONT                     33.92    6040.11
                                          115 Hamilton, ON                        19.51    6059.62
                                           37 Brantford, ON                       28.55    6088.16
                                          111 Guelph, ON                           29.4    6117.56
                                          138 Kitchener, ON                       10.29    6127.86
                                          152 London, ON                          67.63    6195.49
                                           86 Erie, PA                            97.44    6292.93
                                          310 Youngstown, OH                      81.12    6374.05
                                          267 Steubenville, OH                    50.45     6424.5
                                          297 Wheeling, WV                        21.97    6446.47
                                          201 Pittsburgh, PA                      56.46    6502.93
                                           47 Canton, OH                          98.69    6601.61
                                            2 Akron, OH                            21.8    6623.41
                                           61 Cleveland, OH                       31.35    6654.76
                                          304 Windsor, ON                        102.38    6757.14
                                           76 Detroit, MI                          7.21    6764.35
                                            9 Ann Arbor, MI                       47.21    6811.56
                                          125 Jackson, MI                         46.67    6858.23
                                          142 Lansing, MI                         35.28    6893.51
                                           21 Battle Creek, MI                    51.65    6945.16
                                          132 Kalamazoo, MI                       28.23    6973.39
                                          259 South Bend, IN                      62.16    7035.55
                                          104 Gary, IN                            76.01    7111.56
                                           58 Chicago, IL                         27.47    7139.03
                                          135 Kenosha, WI                         52.13    7191.16
                                          214 Racine, WI                          10.12    7201.28
                                          165 Milwaukee, WI                       23.24    7224.52
                                          224 Rockford, IL                        97.71    7322.22
                                          158 Madison, WI                         59.34    7381.56
                                           78 Dubuque, IA                         95.84     7477.4
                                           49 Cedar Rapids, IA                    75.74    7553.14
                                          124 Iowa City, IA                       25.25    7578.39
                                          295 Waterloo, IA                        80.35    7658.74
                                          222 Rochester, MN                         106    7764.75
                                          233 Saint Paul, MN                      76.94    7841.69
                                          166 Minneapolis, MN                     12.03    7853.72
                                          228 Saint Cloud, MN                     73.94    7927.66
                                           79 Duluth, MN                         165.27    8092.93
                                          270 Superior, WI                         4.32    8097.25
                                           81 Eau Claire, WI                     138.42    8235.67
                                          108 Green Bay, WI                      241.21    8476.87
                                          253 Sheboygan, WI                       57.13       8534
                                          106 Grand Rapids, MI                   151.51     8685.5
                                          226 Saginaw, MI                        122.77    8808.27
                                           22 Bay City, MI                        12.83     8821.1
                                           94 Flint, MI                           42.55    8863.65
                                          280 Toledo, OH                          93.64    8957.28
                                          149 Lima, OH                            74.14    9031.42
                                          265 Springfield, OH                     60.13    9091.56
                                           71 Dayton, OH                          28.81    9120.37
                                          114 Hamilton, OH                        35.63       9156
                                           60 Cincinnati, OH                      17.93    9173.93
                                          147 Lexington, KY                       81.08    9255.01
                                          154 Louisville, KY                      90.44    9345.45
                                           35 Bowling Green, KY                    99.3    9444.76
                                          175 Nashville, TN                       61.65     9506.4
                                          122 Huntsville, AL                     100.13    9606.53
                                          100 Gadsden, AL                         63.64    9670.18
                                           30 Birmingham, AL                       64.7    9734.88
                                          170 Montgomery, AL                      86.96    9821.85
                                           65 Columbus, GA                         90.9    9912.75
                                           12 Atlanta, GA                         98.17   10010.92
                                          157 Macon, GA                           81.64   10092.56
                                           14 Augusta, GA                        122.52   10215.08
                                          110 Greenville, SC                      99.77   10314.85
                                          260 Spartanburg, NC                     32.62   10347.46
                                           54 Charlotte, NC                       77.64   10425.11
                                          306 Winston-Salem, NC                   73.13   10498.24
                                          109 Greensboro, NC                       31.3   10529.54
                                          215 Raleigh, NC                         82.35   10611.89
                                           80 Durham, NC                          23.61    10635.5
                                          221 Roanoke, VA                        113.91   10749.41
                                           53 Charleston, WV                     138.61   10888.02
                                           11 Ashland, KY                         70.03   10958.04
                                          312 Zanesville, OH                     109.84   11067.89
                                           66 Columbus, OH                        68.11   11135.99
                                          174 Muncie, IN                         165.75   11301.74
                                          123 Indianapolis, IN                    60.87   11362.61
                                          276 Terre Haute, IN                     89.24   11451.85
                                          288 Urbana, IL                           70.6   11522.45
                                           51 Champaign, IL                        2.52   11524.98
                                           73 Decatur, IL                         52.73   11577.71
                                           32 Bloomington, IL                     44.57   11622.28
                                          196 Peoria, IL                           43.6   11665.88
                                          262 Springfield, IL                     61.75   11727.63
                                          232 Saint Louis, MO                     89.73   11817.36
                                           63 Columbia, MO                       149.28   11966.64
                                          264 Springfield, MO                    137.23   12103.88
                                          130 Joplin, MO                          84.44   12188.32
                                          287 Tulsa, OK                          120.76   12309.07
                                          188 Oklahoma City, OK                  115.46   12424.54
                                           85 Enid, OK                            68.83   12493.37
                                          300 Wichita, KS                         97.07   12590.44
                                          236 Salina, KS                          81.55   12671.99
                                          281 Topeka, KS                         134.35   12806.34
                                          133 Kansas City, KS                     72.73   12879.07
                                          134 Kansas City, MO                      3.52    12882.6
                                          231 Saint Joseph, MO                    49.79   12932.39
                                          150 Lincoln, NE                        144.56   13076.95
                                          189 Omaha, NE                           59.52   13136.47
                                          257 Sioux City, IA                      91.54   13228.01
                                          258 Sioux Falls, SD                     75.45   13303.46
                                           92 Fargo, ND                          229.97   13533.43
                                          305 Winnipeg, MB                       211.94   13745.38
                                           36 Brandon, MB                        187.08   13932.46
                                          167 Minot, ND                          148.55      14081
                                           31 Bismarck, ND                       104.59   14185.59
                                          200 Pierre, SD                         171.22   14356.81
                                          216 Rapid City, SD                     199.98   14556.79
                                           57 Cheyenne, WY                       230.96   14787.75
                                           74 Denver, CO                          97.46   14885.21
                                           62 Colorado Springs, CO                63.56   14948.77
                                          212 Pueblo, CO                          42.63    14991.4
                                          246 Santa Fe, NM                       199.75   15191.16
                                            4 Albuquerque, NM                     64.52   15255.68
                                          102 Gallup, NM                         147.72    15403.4
                                           93 Flagstaff, AZ                      202.26   15605.66
                                          199 Phoenix, AZ                        124.38   15730.04
                                          286 Tucson, AZ                         116.06    15846.1
                                          311 Yuma, AZ                           257.86   16103.95
                                          240 San Diego, CA                      175.01   16278.97
                                          239 San Bernardino, CA                  96.69   16375.66
                                          193 Pasadena, CA                        59.12   16434.78
                                          153 Los Angeles, CA                      9.51   16444.29
                                          244 Santa Barbara, CA                  103.68   16547.97
                                           17 Bakersfield, CA                     80.84   16628.81
                                           99 Fresno, CA                         108.31   16737.12
                                          242 San Jose, CA                       152.25   16889.37
                                          245 Santa Cruz, CA                      26.93    16916.3
                                          186 Oakland, CA                         59.72   16976.02
                                          241 San Francisco, CA                   10.47   16986.49
                                           26 Berkeley, CA                        12.14   16998.63
                                          268 Stockton, CA                        68.11   17066.74
                                          225 Sacramento, CA                      45.35   17112.09
                                           48 Carson City, NV                    125.92      17238
                                          219 Reno, NV                            25.48   17263.49
                                           88 Eureka, CA                         313.15   17576.64
                                           87 Eugene, OR                         236.57    17813.2
                                          235 Salem, OR                           61.63   17874.84
                                          206 Portland, OR                        47.17   17922.01
                                          273 Tacoma, WA                         120.57   18042.58
                                          252 Seattle, WA                         25.62    18068.2
                                           25 Bellingham, WA                      80.42   18148.62
                                          290 Vancouver, BC                       56.66   18205.28
                                          291 Victoria, BC                        41.45   18246.73
                                          308 Yakima, WA                         246.37    18493.1
                                          293 Walla Walla, WA                    153.98   18647.07
                                          261 Spokane, WA                        127.07   18774.15
                                           33 Boise, ID                          291.99   19066.14
                                          203 Pocatello, ID                      264.67   19330.81
                                          187 Ogden, UT                          118.47   19449.28
                                          237 Salt Lake City, UT                  32.44   19481.72
                                          211 Provo, UT                           39.79   19521.51
                                          105 Grand Junction, CO                 229.45   19750.96
                                          255 Sheridan, WY                       411.17   20162.14
                                           27 Billings, MT                       126.61   20288.75
                                          107 Great Falls, MT                    226.94   20515.69
                                          118 Helena, MT                           80.7   20596.39
                                           44 Butte, MT                           53.31    20649.7
                                          146 Lethbridge, AB                     251.24   20900.94
                                           45 Calgary, AB                        132.75   21033.69
                                           82 Edmonton, AB                       173.35   21207.04
                                          161 Medicine Hat, AB                   312.59   21519.63
                                          248 Saskatoon, SK                      315.13   21834.76
                                          173 Moose Jaw, SK                      149.68   21984.44
                                          218 Regina, SK                          62.04   22046.48
                                           77 Dodge City, KS                     933.12    22979.6
                                            7 Amarillo, TX                       215.16   23194.75
                                          156 Lubbock, TX                        113.61   23308.37
                                            1 Abilene, TX                        166.25   23474.62
                                           97 Ft Worth, TX                       167.57   23642.19
                                           69 Dallas, TX                          36.19   23678.39
                                          292 Waco, TX                            88.57   23766.96
                                           16 Austin, TX                          97.72   23864.68
                                          238 San Antonio, TX                     77.99   23942.67
                                          143 Laredo, TX                         149.89   24092.56
                                           68 Corpus Christi, TX                 147.28   24239.84
                                          121 Houston, TX                        195.26    24435.1
                                          103 Galveston, TX                       50.45   24485.55
                                          204 Port Arthur, TX                     72.86   24558.41
                                           23 Beaumont, TX                        17.62   24576.03
                                          256 Shreveport, LA                     170.28   24746.32
                                          160 Marshall, TX                        42.67   24788.99
                                          277 Texarkana, TX                       64.71    24853.7
                                           95 Ft Smith, AR                       137.64   24991.34
                                          151 Little Rock, AR                    152.27   25143.61
                                          162 Memphis, TN                         157.3   25300.91
                                          192 Paducah, KY                        166.97   25467.88
                                           89 Evansville, IN                      94.86   25562.74
                                          140 Lafayette, IN                      175.16    25737.9
                                           96 Ft Wayne, IN                       130.36   25868.26
                                          139 Knoxville, TN                      366.84    26235.1
                                           10 Asheville, NC                       97.65   26332.75
                                           64 Columbia, SC                       152.46   26485.21
                                           52 Charleston, SC                      113.9   26599.11
                                          250 Savannah, GA                         93.9   26693.01
                                          127 Jacksonville, FL                   126.96   26819.97
                                          101 Gainesville, FL                     65.95   26885.92
                                           72 Daytona Beach, FL                   94.98    26980.9
                                          190 Orlando, FL                         52.58   27033.48
                                          275 Tampa, FL                           85.01   27118.49
                                          234 Saint Petersburg, FL                19.54   27138.03
                                          247 Sarasota, FL                        31.73   27169.75
                                          296 West Palm Beach, FL                176.46   27346.21
                                          164 Miami, FL                           65.75   27411.96
                                          136 Key West, FL                       138.36   27550.32
                                          274 Tallahassee, FL                    441.61   27991.93
                                          195 Pensacola, FL                      202.87    28194.8
                                          168 Mobile, AL                          60.12   28254.92
                                           28 Biloxi, MS                          61.74   28316.66
                                          112 Gulfport, MS                        14.47   28331.13
                                          180 New Orleans, LA                     73.62   28404.74
                                           20 Baton Rouge, LA                     82.09   28486.83
                                          176 Natchez, MS                         78.58   28565.41
                                          126 Jackson, MS                         98.43   28663.84
                                           56 Chattanooga, TN                    386.64   29050.48
                                          303 Wilmington, NC                     512.01   29562.49
                                          220 Richmond, VA                       232.37   29794.87
                                           41 Buffalo, NY                        381.27   30176.14
                                          182 Niagara Falls, ON                   20.54   30196.68
                                          227 Saint Catherines, ON                 8.92   30205.61
                                          223 Rochester, NY                      107.53   30313.14
                                          185 North Bay, ON                      260.14   30573.28
                                          269 Sudbury, ON                            93   30666.28
                                          279 Timmins, ON                        140.17   30806.45
                                          249 Sault Ste Marie, ON                260.02   31066.47
                                          278 Thunder Bay, ON                    357.29   31423.76
                                           75 Des Moines, IA                     553.68   31977.44
                                           84 El Paso, TX                       1119.89   33097.34
                                          144 Las Vegas, NV                      671.08   33768.41
                                          209 Prince Rupert, BC                 1638.53   35406.94
                                          131 Juneau, AK                         390.43   35797.37
                                          298 Whitehorse, YK                     172.32   35969.69
                                           70 Dawson, YT                         362.92   36332.61
                                           90 Fairbanks, AK                         596   36928.62
                                            8 Anchorage, AK                      292.11   37220.72
                                          183 Nome, AK                          1095.15   38315.87
                                          148 Lihue, HI                          2967.4   41283.27
                                          120 Honolulu, HI                        114.5   41397.77
                                          119 Hilo, HI                              176   41573.77
                                          309 Yellowknife, NT                   4111.93    45685.7
                                           59 Churchill, MB                     1431.71   47117.42
                                            5 Alert, NT                         2742.38    49859.8
                                          243 San Juan, PR                      4433.43   54293.23
                                          184 Norfolk, VA                       1451.64   55744.86
                      2   55746.24        184 Norfolk, VA
                                          208 Portsmouth, VA                       1.19       1.19
                                          294 Washington, DC                     151.18     152.37
                                           18 Baltimore, MD                       40.06     192.43
                                          141 Lancaster, PA                       55.83     248.26
                                          129 Johnstown, PA                       16.64      264.9
                                          217 Reading, PA                          27.2      292.1
                                          302 Wilmington, DE                      48.48     340.58
                                          198 Philadelphia, PA                    30.05     370.63
                                          283 Trenton, NJ                         34.35     404.99
                                           83 Elizabeth, NJ                       48.02     453.01
                                          177 Newark, NJ                           5.61     458.62
                                          128 Jersey City, NJ                      6.57     465.19
                                          181 New York, NY                         5.04     470.23
                                          194 Paterson, NJ                        18.08     488.32
                                          299 White Plains, NY                    29.39     517.71
                                          266 Stamford, CT                        15.36     533.07
                                           39 Bridgeport, CT                      24.55     557.61
                                          179 New Haven, CT                       21.46     579.08
                                          163 Meriden, CT                         17.96     597.04
                                          178 New Britain,CT                       8.71     605.75
                                          117 Hartford, CT                         9.62     615.37
                                          263 Springfield, MA                     24.25     639.62
                                          202 Pittsfield, MA                      51.31     690.93
                                          285 Troy, NY                            36.34     727.27
                                            3 Albany, NY                           6.88     734.16
                                          251 Schenectady, NY                     16.88     751.04
                                          289 Utica, NY                           91.52     842.56
                                          272 Syracuse, NY                        63.31     905.86
                                           29 Binghamtom, NY                      67.49     973.35
                                          301 Wilkes-Barre, PA                    58.97    1032.33
                                            6 Allentown, PA                       51.68       1084
                                          116 Harrisburg, PA                      99.07    1183.07
                                           13 Atlantic City, NJ                  181.31    1364.38
                                           50 Central Islip, NY                  129.74    1494.12
                                          210 Providence, RI                     142.75    1636.87
                                           91 Fall River, MA                      19.72    1656.59
                                           40 Brockton, MA                        28.03    1684.61
                                           34 Boston, MA                          19.21    1703.83
                                           46 Cambridge, MA                        3.36    1707.19
                                          155 Lowell, MA                          23.03    1730.22
                                          145 Lawrence, MA                        11.73    1741.95
                                          159 Manchester, NH                      28.36     1770.3
                                           67 Concord, NH                         15.76    1786.06
                                          307 Worcester, MA                       67.84     1853.9
                                           38 Brattleboro, VT                     66.17    1920.07
                                          171 Montpelier, VT                      97.37    2017.43
                                           43 Burlington, VT                      46.45    2063.89
                                          172 Montreal, QC                        97.19    2161.08
                                          284 Trois-Rivieres, QC                  98.03     2259.1
                                          254 Sherbrooke, QC                      93.08    2352.18
                                          205 Portland, ME                       157.79    2509.97
                                          207 Portsmouth, NH                      53.74    2563.71
                                           15 Augusta, ME                        109.27    2672.99
                                           19 Bangor, ME                          77.07    2750.06
                                           98 Fredericton, NB                    171.89    2921.95
                                          229 Saint John, NB                      58.53    2980.48
                                          169 Moncton, NB                         99.94    3080.42
                                          113 Halifax, NS                        117.55    3197.97
                                           55 Charlottetown, PE                  100.98    3298.95
                                          271 Syndey, NS                         212.55     3511.5
                                          230 Saint John's, NF                   514.07    4025.57
                                          213 Quebec City, QC                   1290.47    5316.03
                                          191 Ottawa, ON                         400.01    5716.04
                                          137 Kingston, ON                        85.39    5801.42
                                           24 Belleville, ON                      49.28    5850.71
                                          197 Peterborough, ON                    72.87    5923.57
                                          282 Toronto, ON                         82.62    6006.19
                                           42 Burlington, ONT                     33.92    6040.11
                                          115 Hamilton, ON                        19.51    6059.62
                                           37 Brantford, ON                       28.55    6088.16
                                          111 Guelph, ON                           29.4    6117.56
                                          138 Kitchener, ON                       10.29    6127.86
                                          152 London, ON                          67.63    6195.49
                                           86 Erie, PA                            97.44    6292.93
                                          310 Youngstown, OH                      81.12    6374.05
                                          267 Steubenville, OH                    50.45     6424.5
                                          297 Wheeling, WV                        21.97    6446.47
                                          201 Pittsburgh, PA                      56.46    6502.93
                                           47 Canton, OH                          98.69    6601.61
                                            2 Akron, OH                            21.8    6623.41
                                           61 Cleveland, OH                       31.35    6654.76
                                          304 Windsor, ON                        102.38    6757.14
                                           76 Detroit, MI                          7.21    6764.35
                                            9 Ann Arbor, MI                       47.21    6811.56
                                          125 Jackson, MI                         46.67    6858.23
                                          142 Lansing, MI                         35.28    6893.51
                                           21 Battle Creek, MI                    51.65    6945.16
                                          132 Kalamazoo, MI                       28.23    6973.39
                                          259 South Bend, IN                      62.16    7035.55
                                          104 Gary, IN                            76.01    7111.56
                                           58 Chicago, IL                         27.47    7139.03
                                          135 Kenosha, WI                         52.13    7191.16
                                          214 Racine, WI                          10.12    7201.28
                                          165 Milwaukee, WI                       23.24    7224.52
                                          224 Rockford, IL                        97.71    7322.22
                                          158 Madison, WI                         59.34    7381.56
                                           78 Dubuque, IA                         95.84     7477.4
                                           49 Cedar Rapids, IA                    75.74    7553.14
                                          124 Iowa City, IA                       25.25    7578.39
                                          295 Waterloo, IA                        80.35    7658.74
                                          222 Rochester, MN                         106    7764.75
                                          233 Saint Paul, MN                      76.94    7841.69
                                          166 Minneapolis, MN                     12.03    7853.72
                                          228 Saint Cloud, MN                     73.94    7927.66
                                           79 Duluth, MN                         165.27    8092.93
                                          270 Superior, WI                         4.32    8097.25
                                           81 Eau Claire, WI                     138.42    8235.67
                                          108 Green Bay, WI                      241.21    8476.87
                                          253 Sheboygan, WI                       57.13       8534
                                          106 Grand Rapids, MI                   151.51     8685.5
                                          226 Saginaw, MI                        122.77    8808.27
                                           22 Bay City, MI                        12.83     8821.1
                                           94 Flint, MI                           42.55    8863.65
                                          280 Toledo, OH                          93.64    8957.28
                                          149 Lima, OH                            74.14    9031.42
                                          265 Springfield, OH                     60.13    9091.56
                                           71 Dayton, OH                          28.81    9120.37
                                          114 Hamilton, OH                        35.63       9156
                                           60 Cincinnati, OH                      17.93    9173.93
                                          147 Lexington, KY                       81.08    9255.01
                                          154 Louisville, KY                      90.44    9345.45
                                           35 Bowling Green, KY                    99.3    9444.76
                                          175 Nashville, TN                       61.65     9506.4
                                          122 Huntsville, AL                     100.13    9606.53
                                          100 Gadsden, AL                         63.64    9670.18
                                           30 Birmingham, AL                       64.7    9734.88
                                          170 Montgomery, AL                      86.96    9821.85
                                           65 Columbus, GA                         90.9    9912.75
                                           12 Atlanta, GA                         98.17   10010.92
                                          157 Macon, GA                           81.64   10092.56
                                           14 Augusta, GA                        122.52   10215.08
                                          110 Greenville, SC                      99.77   10314.85
                                          260 Spartanburg, NC                     32.62   10347.46
                                           54 Charlotte, NC                       77.64   10425.11
                                          306 Winston-Salem, NC                   73.13   10498.24
                                          109 Greensboro, NC                       31.3   10529.54
                                          215 Raleigh, NC                         82.35   10611.89
                                           80 Durham, NC                          23.61    10635.5
                                          221 Roanoke, VA                        113.91   10749.41
                                           53 Charleston, WV                     138.61   10888.02
                                           11 Ashland, KY                         70.03   10958.04
                                          312 Zanesville, OH                     109.84   11067.89
                                           66 Columbus, OH                        68.11   11135.99
                                          174 Muncie, IN                         165.75   11301.74
                                          123 Indianapolis, IN                    60.87   11362.61
                                          276 Terre Haute, IN                     89.24   11451.85
                                          288 Urbana, IL                           70.6   11522.45
                                           51 Champaign, IL                        2.52   11524.98
                                           73 Decatur, IL                         52.73   11577.71
                                           32 Bloomington, IL                     44.57   11622.28
                                          196 Peoria, IL                           43.6   11665.88
                                          262 Springfield, IL                     61.75   11727.63
                                          232 Saint Louis, MO                     89.73   11817.36
                                           63 Columbia, MO                       149.28   11966.64
                                          264 Springfield, MO                    137.23   12103.88
                                          130 Joplin, MO                          84.44   12188.32
                                          287 Tulsa, OK                          120.76   12309.07
                                          188 Oklahoma City, OK                  115.46   12424.54
                                           85 Enid, OK                            68.83   12493.37
                                          300 Wichita, KS                         97.07   12590.44
                                          236 Salina, KS                          81.55   12671.99
                                          281 Topeka, KS                         134.35   12806.34
                                          133 Kansas City, KS                     72.73   12879.07
                                          134 Kansas City, MO                      3.52    12882.6
                                          231 Saint Joseph, MO                    49.79   12932.39
                                          150 Lincoln, NE                        144.56   13076.95
                                          189 Omaha, NE                           59.52   13136.47
                                          257 Sioux City, IA                      91.54   13228.01
                                          258 Sioux Falls, SD                     75.45   13303.46
                                           92 Fargo, ND                          229.97   13533.43
                                          305 Winnipeg, MB                       211.94   13745.38
                                           36 Brandon, MB                        187.08   13932.46
                                          167 Minot, ND                          148.55      14081
                                           31 Bismarck, ND                       104.59   14185.59
                                          200 Pierre, SD                         171.22   14356.81
                                          216 Rapid City, SD                     199.98   14556.79
                                           57 Cheyenne, WY                       230.96   14787.75
                                           74 Denver, CO                          97.46   14885.21
                                           62 Colorado Springs, CO                63.56   14948.77
                                          212 Pueblo, CO                          42.63    14991.4
                                          246 Santa Fe, NM                       199.75   15191.16
                                            4 Albuquerque, NM                     64.52   15255.68
                                          102 Gallup, NM                         147.72    15403.4
                                           93 Flagstaff, AZ                      202.26   15605.66
                                          199 Phoenix, AZ                        124.38   15730.04
                                          286 Tucson, AZ                         116.06    15846.1
                                          311 Yuma, AZ                           257.86   16103.95
                                          240 San Diego, CA                      175.01   16278.97
                                          239 San Bernardino, CA                  96.69   16375.66
                                          193 Pasadena, CA                        59.12   16434.78
                                          153 Los Angeles, CA                      9.51   16444.29
                                          244 Santa Barbara, CA                  103.68   16547.97
                                           17 Bakersfield, CA                     80.84   16628.81
                                           99 Fresno, CA                         108.31   16737.12
                                          242 San Jose, CA                       152.25   16889.37
                                          245 Santa Cruz, CA                      26.93    16916.3
                                          186 Oakland, CA                         59.72   16976.02
                                          241 San Francisco, CA                   10.47   16986.49
                                           26 Berkeley, CA                        12.14   16998.63
                                          268 Stockton, CA                        68.11   17066.74
                                          225 Sacramento, CA                      45.35   17112.09
                                           48 Carson City, NV                    125.92      17238
                                          219 Reno, NV                            25.48   17263.49
                                           88 Eureka, CA                         313.15   17576.64
                                           87 Eugene, OR                         236.57    17813.2
                                          235 Salem, OR                           61.63   17874.84
                                          206 Portland, OR                        47.17   17922.01
                                          273 Tacoma, WA                         120.57   18042.58
                                          252 Seattle, WA                         25.62    18068.2
                                           25 Bellingham, WA                      80.42   18148.62
                                          290 Vancouver, BC                       56.66   18205.28
                                          291 Victoria, BC                        41.45   18246.73
                                          308 Yakima, WA                         246.37    18493.1
                                          293 Walla Walla, WA                    153.98   18647.07
                                          261 Spokane, WA                        127.07   18774.15
                                           33 Boise, ID                          291.99   19066.14
                                          203 Pocatello, ID                      264.67   19330.81
                                          187 Ogden, UT                          118.47   19449.28
                                          237 Salt Lake City, UT                  32.44   19481.72
                                          211 Provo, UT                           39.79   19521.51
                                          105 Grand Junction, CO                 229.45   19750.96
                                          255 Sheridan, WY                       411.17   20162.14
                                           27 Billings, MT                       126.61   20288.75
                                          107 Great Falls, MT                    226.94   20515.69
                                          118 Helena, MT                           80.7   20596.39
                                           44 Butte, MT                           53.31    20649.7
                                          146 Lethbridge, AB                     251.24   20900.94
                                           45 Calgary, AB                        132.75   21033.69
                                           82 Edmonton, AB                       173.35   21207.04
                                          161 Medicine Hat, AB                   312.59   21519.63
                                          248 Saskatoon, SK                      315.13   21834.76
                                          173 Moose Jaw, SK                      149.68   21984.44
                                          218 Regina, SK                          62.04   22046.48
                                           77 Dodge City, KS                     933.12    22979.6
                                            7 Amarillo, TX                       215.16   23194.75
                                          156 Lubbock, TX                        113.61   23308.37
                                            1 Abilene, TX                        166.25   23474.62
                                           97 Ft Worth, TX                       167.57   23642.19
                                           69 Dallas, TX                          36.19   23678.39
                                          292 Waco, TX                            88.57   23766.96
                                           16 Austin, TX                          97.72   23864.68
                                          238 San Antonio, TX                     77.99   23942.67
                                          143 Laredo, TX                         149.89   24092.56
                                           68 Corpus Christi, TX                 147.28   24239.84
                                          121 Houston, TX                        195.26    24435.1
                                          103 Galveston, TX                       50.45   24485.55
                                          204 Port Arthur, TX                     72.86   24558.41
                                           23 Beaumont, TX                        17.62   24576.03
                                          160 Marshall, TX                       170.89   24746.92
                                          256 Shreveport, LA                      42.67   24789.59
                                          277 Texarkana, TX                       65.49   24855.08
                                           95 Ft Smith, AR                       137.64   24992.72
                                          151 Little Rock, AR                    152.27      25145
                                          162 Memphis, TN                         157.3   25302.29
                                          192 Paducah, KY                        166.97   25469.26
                                           89 Evansville, IN                      94.86   25564.12
                                          140 Lafayette, IN                      175.16   25739.28
                                           96 Ft Wayne, IN                       130.36   25869.64
                                          139 Knoxville, TN                      366.84   26236.49
                                           10 Asheville, NC                       97.65   26334.13
                                           64 Columbia, SC                       152.46    26486.6
                                           52 Charleston, SC                      113.9   26600.49
                                          250 Savannah, GA                         93.9   26694.39
                                          127 Jacksonville, FL                   126.96   26821.35
                                          101 Gainesville, FL                     65.95    26887.3
                                           72 Daytona Beach, FL                   94.98   26982.28
                                          190 Orlando, FL                         52.58   27034.86
                                          275 Tampa, FL                           85.01   27119.87
                                          234 Saint Petersburg, FL                19.54   27139.41
                                          247 Sarasota, FL                        31.73   27171.14
                                          296 West Palm Beach, FL                176.46    27347.6
                                          164 Miami, FL                           65.75   27413.34
                                          136 Key West, FL                       138.36    27551.7
                                          274 Tallahassee, FL                    441.61   27993.31
                                          195 Pensacola, FL                      202.87   28196.18
                                          168 Mobile, AL                          60.12    28256.3
                                           28 Biloxi, MS                          61.74   28318.04
                                          112 Gulfport, MS                        14.47   28332.51
                                          180 New Orleans, LA                     73.62   28406.13
                                           20 Baton Rouge, LA                     82.09   28488.22
                                          176 Natchez, MS                         78.58    28566.8
                                          126 Jackson, MS                         98.43   28665.22
                                           56 Chattanooga, TN                    386.64   29051.86
                                          303 Wilmington, NC                     512.01   29563.88
                                          220 Richmond, VA                       232.37   29796.25
                                           41 Buffalo, NY                        381.27   30177.52
                                          182 Niagara Falls, ON                   20.54   30198.06
                                          227 Saint Catherines, ON                 8.92   30206.99
                                          223 Rochester, NY                      107.53   30314.52
                                          185 North Bay, ON                      260.14   30574.66
                                          269 Sudbury, ON                            93   30667.66
                                          279 Timmins, ON                        140.17   30807.83
                                          249 Sault Ste Marie, ON                260.02   31067.85
                                          278 Thunder Bay, ON                    357.29   31425.14
                                           75 Des Moines, IA                     553.68   31978.82
                                           84 El Paso, TX                       1119.89   33098.72
                                          144 Las Vegas, NV                      671.08    33769.8
                                          209 Prince Rupert, BC                 1638.53   35408.32
                                          131 Juneau, AK                         390.43   35798.75
                                          298 Whitehorse, YK                     172.32   35971.07
                                           70 Dawson, YT                         362.92   36333.99
                                           90 Fairbanks, AK                         596      36930
                                            8 Anchorage, AK                      292.11    37222.1
                                          183 Nome, AK                          1095.15   38317.26
                                          148 Lihue, HI                          2967.4   41284.65
                                          120 Honolulu, HI                        114.5   41399.15
                                          119 Hilo, HI                              176   41575.15
                                          309 Yellowknife, NT                   4111.93   45687.09
                                           59 Churchill, MB                     1431.71    47118.8
                                            5 Alert, NT                         2742.38   49861.18
                                          243 San Juan, PR                      4433.43   54294.61
                                          184 Norfolk, VA                       1451.64   55746.24
51 -> 288             1   56794.75         51 Champaign, IL
                                          288 Urbana, IL                           2.52       2.52
                                           73 Decatur, IL                         54.92      57.45
                                          262 Springfield, IL                     47.66     105.11
                                          196 Peoria, IL                          61.75     166.86
                                           32 Bloomington, IL                      43.6     210.46
                                          224 Rockford, IL                       123.66     334.12
                                          135 Kenosha, WI                         90.57     424.69
                                          165 Milwaukee, WI                       31.93     456.62
                                          214 Racine, WI                          23.24     479.86
                                           58 Chicago, IL                         61.23     541.09
                                          104 Gary, IN                            27.47     568.56
                                          259 South Bend, IN                      76.01     644.57
                                          132 Kalamazoo, MI                       62.16     706.73
                                           21 Battle Creek, MI                    28.23     734.96
                                          142 Lansing, MI                         51.65     786.61
                                          125 Jackson, MI                         35.28     821.89
                                            9 Ann Arbor, MI                       46.67     868.56
                                          280 Toledo, OH                          43.57     912.13
                                           76 Detroit, MI                         58.03     970.16
                                          304 Windsor, ON                          7.21     977.37
                                           94 Flint, MI                           71.67    1049.04
                                          226 Saginaw, MI                          33.5    1082.54
                                           22 Bay City, MI                        12.83    1095.36
                                          106 Grand Rapids, MI                   130.44     1225.8
                                           96 Ft Wayne, IN                          132     1357.8
                                          174 Muncie, IN                          67.15    1424.95
                                          123 Indianapolis, IN                    60.87    1485.83
                                          140 Lafayette, IN                       66.81    1552.63
                                          276 Terre Haute, IN                     75.45    1628.09
                                           89 Evansville, IN                     103.56    1731.64
                                          192 Paducah, KY                         94.86    1826.51
                                          175 Nashville, TN                      140.56    1967.07
                                           35 Bowling Green, KY                   61.65    2028.71
                                          154 Louisville, KY                       99.3    2128.02
                                          147 Lexington, KY                       90.44    2218.46
                                           60 Cincinnati, OH                      81.08    2299.54
                                          114 Hamilton, OH                        17.93    2317.47
                                           71 Dayton, OH                          35.63     2353.1
                                          265 Springfield, OH                     28.81    2381.91
                                           66 Columbus, OH                        56.03    2437.94
                                          312 Zanesville, OH                      68.11    2506.04
                                           47 Canton, OH                          73.78    2579.83
                                            2 Akron, OH                            21.8    2601.63
                                           61 Cleveland, OH                       31.35    2632.98
                                          310 Youngstown, OH                      77.36    2710.33
                                          267 Steubenville, OH                    50.45    2760.79
                                          297 Wheeling, WV                        21.97    2782.75
                                          201 Pittsburgh, PA                      56.46    2839.21
                                           86 Erie, PA                           116.83    2956.05
                                           37 Brantford, ON                       71.49    3027.54
                                          138 Kitchener, ON                       22.82    3050.35
                                          111 Guelph, ON                          10.29    3060.65
                                           42 Burlington, ONT                     29.68    3090.32
                                          115 Hamilton, ON                        19.51    3109.83
                                          227 Saint Catherines, ON                52.51    3162.34
                                           41 Buffalo, NY                         29.25    3191.58
                                          182 Niagara Falls, ON                   20.54    3212.12
                                          282 Toronto, ON                         42.01    3254.13
                                          197 Peterborough, ON                    82.62    3336.75
                                           24 Belleville, ON                      72.87    3409.61
                                          137 Kingston, ON                        49.28     3458.9
                                          191 Ottawa, ON                          85.39    3544.28
                                          172 Montreal, QC                       146.68    3690.97
                                           43 Burlington, VT                      97.19    3788.16
                                          171 Montpelier, VT                      46.45    3834.61
                                           38 Brattleboro, VT                     97.37    3931.97
                                          263 Springfield, MA                     51.84    3983.81
                                          117 Hartford, CT                        24.25    4008.06
                                          178 New Britain,CT                       9.62    4017.69
                                          163 Meriden, CT                          8.71     4026.4
                                          179 New Haven, CT                       17.96    4044.36
                                           39 Bridgeport, CT                      21.46    4065.82
                                          266 Stamford, CT                        24.55    4090.37
                                          299 White Plains, NY                    15.36    4105.73
                                          181 New York, NY                        27.75    4133.48
                                          128 Jersey City, NJ                      5.04    4138.52
                                          177 Newark, NJ                           6.57    4145.09
                                           83 Elizabeth, NJ                        5.61     4150.7
                                          194 Paterson, NJ                        17.67    4168.37
                                          283 Trenton, NJ                          62.4    4230.77
                                          198 Philadelphia, PA                    34.35    4265.12
                                          302 Wilmington, DE                      30.05    4295.18
                                          217 Reading, PA                         48.48    4343.66
                                          129 Johnstown, PA                        27.2    4370.86
                                          141 Lancaster, PA                       16.64     4387.5
                                          116 Harrisburg, PA                      43.19    4430.69
                                           18 Baltimore, MD                        70.5    4501.18
                                          294 Washington, DC                      40.06    4541.24
                                          220 Richmond, VA                        97.21    4638.45
                                          208 Portsmouth, VA                      94.39    4732.84
                                          184 Norfolk, VA                          1.19    4734.03
                                          215 Raleigh, NC                        178.76    4912.79
                                           80 Durham, NC                          23.61    4936.41
                                          109 Greensboro, NC                      61.97    4998.37
                                          306 Winston-Salem, NC                    31.3    5029.67
                                           54 Charlotte, NC                       73.13     5102.8
                                          260 Spartanburg, NC                     77.64    5180.45
                                          110 Greenville, SC                      32.62    5213.06
                                           10 Asheville, NC                       52.87    5265.93
                                          139 Knoxville, TN                       97.65    5363.58
                                           56 Chattanooga, TN                    114.91    5478.49
                                          100 Gadsden, AL                         86.02    5564.51
                                          122 Huntsville, AL                      63.64    5628.15
                                           30 Birmingham, AL                      84.91    5713.06
                                          170 Montgomery, AL                      86.96    5800.02
                                           65 Columbus, GA                         90.9    5890.93
                                          157 Macon, GA                           97.26    5988.18
                                           12 Atlanta, GA                         81.64    6069.82
                                           14 Augusta, GA                        167.83    6237.65
                                           64 Columbia, SC                        74.55    6312.21
                                           52 Charleston, SC                      113.9     6426.1
                                          250 Savannah, GA                         93.9       6520
                                          127 Jacksonville, FL                   126.96    6646.96
                                          101 Gainesville, FL                     65.95    6712.91
                                           72 Daytona Beach, FL                   94.98    6807.89
                                          190 Orlando, FL                         52.58    6860.48
                                          275 Tampa, FL                           85.01    6945.48
                                          234 Saint Petersburg, FL                19.54    6965.02
                                          247 Sarasota, FL                        31.73    6996.75
                                          296 West Palm Beach, FL                176.46    7173.21
                                          164 Miami, FL                           65.75    7238.96
                                          136 Key West, FL                       138.36    7377.32
                                          274 Tallahassee, FL                    441.61    7818.92
                                          195 Pensacola, FL                      202.87     8021.8
                                          168 Mobile, AL                          60.12    8081.91
                                           28 Biloxi, MS                          61.74    8143.65
                                          112 Gulfport, MS                        14.47    8158.12
                                          180 New Orleans, LA                     73.62    8231.74
                                           20 Baton Rouge, LA                     82.09    8313.83
                                          176 Natchez, MS                         78.58    8392.41
                                          126 Jackson, MS                         98.43    8490.84
                                          162 Memphis, TN                         197.2    8688.04
                                          151 Little Rock, AR                     157.3    8845.33
                                          277 Texarkana, TX                      151.96    8997.29
                                          160 Marshall, TX                        64.71       9062
                                          256 Shreveport, LA                      42.67    9104.67
                                           23 Beaumont, TX                       170.28    9274.96
                                          204 Port Arthur, TX                     17.62    9292.58
                                          103 Galveston, TX                       72.86    9365.44
                                          121 Houston, TX                         50.45    9415.89
                                           16 Austin, TX                         168.07    9583.96
                                          238 San Antonio, TX                     77.99    9661.95
                                           68 Corpus Christi, TX                 135.39    9797.35
                                          143 Laredo, TX                         147.28    9944.63
                                          292 Waco, TX                            323.5   10268.13
                                           97 Ft Worth, TX                        82.15   10350.28
                                           69 Dallas, TX                          36.19   10386.47
                                          188 Oklahoma City, OK                  191.95   10578.42
                                           85 Enid, OK                            68.83   10647.25
                                          300 Wichita, KS                         97.07   10744.32
                                          236 Salina, KS                          81.55   10825.87
                                          281 Topeka, KS                         134.35   10960.22
                                          133 Kansas City, KS                     72.73   11032.95
                                          134 Kansas City, MO                      3.52   11036.48
                                          231 Saint Joseph, MO                    49.79   11086.27
                                          189 Omaha, NE                          127.61   11213.88
                                          150 Lincoln, NE                         59.52    11273.4
                                          257 Sioux City, IA                     118.91    11392.3
                                          258 Sioux Falls, SD                     75.45   11467.76
                                          228 Saint Cloud, MN                    223.72   11691.48
                                          166 Minneapolis, MN                     73.94   11765.42
                                          233 Saint Paul, MN                      12.03   11777.45
                                          222 Rochester, MN                       76.94   11854.39
                                           81 Eau Claire, WI                       86.5   11940.89
                                          270 Superior, WI                       138.42    12079.3
                                           79 Duluth, MN                           4.32   12083.63
                                          278 Thunder Bay, ON                    221.38   12305.01
                                          108 Green Bay, WI                      281.05   12586.06
                                          253 Sheboygan, WI                       57.13   12643.19
                                          158 Madison, WI                         125.6   12768.78
                                           78 Dubuque, IA                         95.84   12864.62
                                           49 Cedar Rapids, IA                    75.74   12940.36
                                          124 Iowa City, IA                       25.25   12965.61
                                          295 Waterloo, IA                        80.35   13045.96
                                           75 Des Moines, IA                     107.02   13152.99
                                           63 Columbia, MO                       203.13   13356.12
                                          264 Springfield, MO                    137.23   13493.35
                                          130 Joplin, MO                          84.44   13577.79
                                           95 Ft Smith, AR                       117.62    13695.4
                                          287 Tulsa, OK                          122.27   13817.67
                                           77 Dodge City, KS                     299.19   14116.86
                                            7 Amarillo, TX                       215.16   14332.02
                                          156 Lubbock, TX                        113.61   14445.63
                                            1 Abilene, TX                        166.25   14611.88
                                           84 El Paso, TX                        469.27   15081.15
                                            4 Albuquerque, NM                    230.08   15311.23
                                          246 Santa Fe, NM                        64.52   15375.75
                                          102 Gallup, NM                         194.11   15569.86
                                           93 Flagstaff, AZ                      202.26   15772.12
                                          199 Phoenix, AZ                        124.38    15896.5
                                          286 Tucson, AZ                         116.06   16012.56
                                          311 Yuma, AZ                           257.86   16270.41
                                          240 San Diego, CA                      175.01   16445.43
                                          239 San Bernardino, CA                  96.69   16542.12
                                          193 Pasadena, CA                        59.12   16601.24
                                          153 Los Angeles, CA                      9.51   16610.75
                                          244 Santa Barbara, CA                  103.68   16714.43
                                           17 Bakersfield, CA                     80.84   16795.27
                                           99 Fresno, CA                         108.31   16903.58
                                          268 Stockton, CA                       134.15   17037.73
                                          225 Sacramento, CA                      45.35   17083.08
                                           26 Berkeley, CA                         72.8   17155.88
                                          186 Oakland, CA                          4.65   17160.53
                                          241 San Francisco, CA                   10.47      17171
                                          242 San Jose, CA                        47.11   17218.11
                                          245 Santa Cruz, CA                      26.93   17245.03
                                           48 Carson City, NV                    217.59   17462.63
                                          219 Reno, NV                            25.48   17488.11
                                           88 Eureka, CA                         313.15   17801.26
                                           87 Eugene, OR                         236.57   18037.82
                                          235 Salem, OR                           61.63   18099.46
                                          206 Portland, OR                        47.17   18146.63
                                          273 Tacoma, WA                         120.57    18267.2
                                          252 Seattle, WA                         25.62   18292.82
                                           25 Bellingham, WA                      80.42   18373.24
                                          290 Vancouver, BC                       56.66    18429.9
                                          291 Victoria, BC                        41.45   18471.35
                                          308 Yakima, WA                         246.37   18717.72
                                          293 Walla Walla, WA                    153.98    18871.7
                                          261 Spokane, WA                        127.07   18998.77
                                           33 Boise, ID                          291.99   19290.76
                                          203 Pocatello, ID                      264.67   19555.43
                                          187 Ogden, UT                          118.47    19673.9
                                          237 Salt Lake City, UT                  32.44   19706.34
                                          211 Provo, UT                           39.79   19746.13
                                          105 Grand Junction, CO                 229.45   19975.58
                                           74 Denver, CO                         250.76   20226.34
                                           62 Colorado Springs, CO                63.56    20289.9
                                          212 Pueblo, CO                          42.63   20332.53
                                           57 Cheyenne, WY                       199.91   20532.45
                                          216 Rapid City, SD                     230.96    20763.4
                                          200 Pierre, SD                         199.98   20963.38
                                           31 Bismarck, ND                       171.22    21134.6
                                          167 Minot, ND                          104.59   21239.19
                                           36 Brandon, MB                        148.55   21387.74
                                          305 Winnipeg, MB                       187.08   21574.82
                                           92 Fargo, ND                          211.94   21786.76
                                          218 Regina, SK                         597.09   22383.85
                                          173 Moose Jaw, SK                       62.04   22445.89
                                          248 Saskatoon, SK                      149.68   22595.57
                                          161 Medicine Hat, AB                   315.13    22910.7
                                          146 Lethbridge, AB                     146.13   23056.82
                                           45 Calgary, AB                        132.75   23189.57
                                           82 Edmonton, AB                       173.35   23362.92
                                          107 Great Falls, MT                    446.74   23809.67
                                          118 Helena, MT                           80.7   23890.36
                                           44 Butte, MT                           53.31   23943.68
                                           27 Billings, MT                       279.14   24222.81
                                          255 Sheridan, WY                       126.61   24349.42
                                          144 Las Vegas, NV                      821.24   25170.67
                                          209 Prince Rupert, BC                 1638.53   26809.19
                                          131 Juneau, AK                         390.43   27199.62
                                          298 Whitehorse, YK                     172.32   27371.94
                                           70 Dawson, YT                         362.92   27734.86
                                           90 Fairbanks, AK                         596   28330.87
                                            8 Anchorage, AK                      292.11   28622.97
                                          183 Nome, AK                          1095.15   29718.13
                                          148 Lihue, HI                          2967.4   32685.52
                                          120 Honolulu, HI                        114.5   32800.02
                                          119 Hilo, HI                              176   32976.02
                                          309 Yellowknife, NT                   4111.93   37087.96
                                           59 Churchill, MB                     1431.71   38519.67
                                          249 Sault Ste Marie, ON               1073.27   39592.94
                                          269 Sudbury, ON                        256.57   39849.51
                                          185 North Bay, ON                          93   39942.51
                                          279 Timmins, ON                        198.01   40140.52
                                          152 London, ON                         387.85   40528.37
                                          223 Rochester, NY                       249.8   40778.17
                                          272 Syracuse, NY                       101.71   40879.88
                                          289 Utica, NY                           63.31   40943.18
                                           29 Binghamtom, NY                      83.89   41027.07
                                          301 Wilkes-Barre, PA                    58.97   41086.05
                                            6 Allentown, PA                       51.68   41137.72
                                           13 Atlantic City, NJ                  113.26   41250.99
                                           50 Central Islip, NY                  129.74   41380.73
                                          202 Pittsfield, MA                      114.7   41495.43
                                          285 Troy, NY                            36.34   41531.77
                                            3 Albany, NY                           6.88   41538.65
                                          251 Schenectady, NY                     16.88   41555.54
                                          307 Worcester, MA                      152.51   41708.04
                                          210 Providence, RI                      40.53   41748.58
                                           91 Fall River, MA                      19.72   41768.29
                                           40 Brockton, MA                        28.03   41796.32
                                           34 Boston, MA                          19.21   41815.53
                                           46 Cambridge, MA                        3.36    41818.9
                                          155 Lowell, MA                          23.03   41841.93
                                          145 Lawrence, MA                        11.73   41853.65
                                          159 Manchester, NH                      28.36   41882.01
                                           67 Concord, NH                         15.76   41897.76
                                          207 Portsmouth, NH                      54.37   41952.14
                                          205 Portland, ME                        53.74   42005.88
                                           15 Augusta, ME                         55.61   42061.49
                                           19 Bangor, ME                          77.07   42138.56
                                           98 Fredericton, NB                    171.89   42310.45
                                          229 Saint John, NB                      58.53   42368.98
                                          169 Moncton, NB                         99.94   42468.93
                                           55 Charlottetown, PE                  108.58    42577.5
                                          113 Halifax, NS                        100.98   42678.49
                                          271 Syndey, NS                         254.97   42933.46
                                          230 Saint John's, NF                   514.07   43447.53
                                          213 Quebec City, QC                   1290.47   44737.99
                                          284 Trois-Rivieres, QC                 190.67   44928.66
                                          254 Sherbrooke, QC                      93.08   45021.74
                                          221 Roanoke, VA                        802.17   45823.91
                                           53 Charleston, WV                     138.61   45962.52
                                           11 Ashland, KY                         70.03   46032.55
                                          149 Lima, OH                           186.42   46218.96
                                          232 Saint Louis, MO                    445.62   46664.59
                                          303 Wilmington, NC                     899.59   47564.17
                                          243 San Juan, PR                      1361.81   48925.99
                                            5 Alert, NT                         4433.43   53359.42
                                           51 Champaign, IL                     3435.34   56794.75
                      2   56794.89         51 Champaign, IL
                                          288 Urbana, IL                           2.52       2.52
                                           73 Decatur, IL                         54.92      57.45
                                          262 Springfield, IL                     47.66     105.11
                                          196 Peoria, IL                          61.75     166.86
                                           32 Bloomington, IL                      43.6     210.46
                                          224 Rockford, IL                       123.66     334.12
                                          135 Kenosha, WI                         90.57     424.69
                                          165 Milwaukee, WI                       31.93     456.62
                                          214 Racine, WI                          23.24     479.86
                                           58 Chicago, IL                         61.23     541.09
                                          104 Gary, IN                            27.47     568.56
                                          259 South Bend, IN                      76.01     644.57
                                          132 Kalamazoo, MI                       62.16     706.73
                                           21 Battle Creek, MI                    28.23     734.96
                                          142 Lansing, MI                         51.65     786.61
                                          125 Jackson, MI                         35.28     821.89
                                            9 Ann Arbor, MI                       46.67     868.56
                                          280 Toledo, OH                          43.57     912.13
                                           76 Detroit, MI                         58.03     970.16
                                          304 Windsor, ON                          7.21     977.37
                                           94 Flint, MI                           71.67    1049.04
                                          226 Saginaw, MI                          33.5    1082.54
                                           22 Bay City, MI                        12.83    1095.36
                                          106 Grand Rapids, MI                   130.44     1225.8
                                           96 Ft Wayne, IN                          132     1357.8
                                          174 Muncie, IN                          67.15    1424.95
                                          123 Indianapolis, IN                    60.87    1485.83
                                          140 Lafayette, IN                       66.81    1552.63
                                          276 Terre Haute, IN                     75.45    1628.09
                                           89 Evansville, IN                     103.56    1731.64
                                          192 Paducah, KY                         94.86    1826.51
                                          175 Nashville, TN                      140.56    1967.07
                                           35 Bowling Green, KY                   61.65    2028.71
                                          154 Louisville, KY                       99.3    2128.02
                                          147 Lexington, KY                       90.44    2218.46
                                           60 Cincinnati, OH                      81.08    2299.54
                                          114 Hamilton, OH                        17.93    2317.47
                                           71 Dayton, OH                          35.63     2353.1
                                          265 Springfield, OH                     28.81    2381.91
                                           66 Columbus, OH                        56.03    2437.94
                                          312 Zanesville, OH                      68.11    2506.04
                                           47 Canton, OH                          73.78    2579.83
                                            2 Akron, OH                            21.8    2601.63
                                           61 Cleveland, OH                       31.35    2632.98
                                          310 Youngstown, OH                      77.36    2710.33
                                          267 Steubenville, OH                    50.45    2760.79
                                          297 Wheeling, WV                        21.97    2782.75
                                          201 Pittsburgh, PA                      56.46    2839.21
                                           86 Erie, PA                           116.83    2956.05
                                           37 Brantford, ON                       71.49    3027.54
                                          138 Kitchener, ON                       22.82    3050.35
                                          111 Guelph, ON                          10.29    3060.65
                                           42 Burlington, ONT                     29.68    3090.32
                                          115 Hamilton, ON                        19.51    3109.83
                                          227 Saint Catherines, ON                52.51    3162.34
                                          182 Niagara Falls, ON                    8.92    3171.26
                                           41 Buffalo, NY                         20.54     3191.8
                                          282 Toronto, ON                         62.46    3254.26
                                          197 Peterborough, ON                    82.62    3336.88
                                           24 Belleville, ON                      72.87    3409.75
                                          137 Kingston, ON                        49.28    3459.03
                                          191 Ottawa, ON                          85.39    3544.41
                                          172 Montreal, QC                       146.68     3691.1
                                           43 Burlington, VT                      97.19    3788.29
                                          171 Montpelier, VT                      46.45    3834.74
                                           38 Brattleboro, VT                     97.37    3932.11
                                          263 Springfield, MA                     51.84    3983.94
                                          117 Hartford, CT                        24.25     4008.2
                                          178 New Britain,CT                       9.62    4017.82
                                          163 Meriden, CT                          8.71    4026.53
                                          179 New Haven, CT                       17.96    4044.49
                                           39 Bridgeport, CT                      21.46    4065.95
                                          266 Stamford, CT                        24.55     4090.5
                                          299 White Plains, NY                    15.36    4105.86
                                          181 New York, NY                        27.75    4133.61
                                          128 Jersey City, NJ                      5.04    4138.65
                                          177 Newark, NJ                           6.57    4145.22
                                           83 Elizabeth, NJ                        5.61    4150.84
                                          194 Paterson, NJ                        17.67     4168.5
                                          283 Trenton, NJ                          62.4     4230.9
                                          198 Philadelphia, PA                    34.35    4265.26
                                          302 Wilmington, DE                      30.05    4295.31
                                          217 Reading, PA                         48.48    4343.79
                                          129 Johnstown, PA                        27.2    4370.99
                                          141 Lancaster, PA                       16.64    4387.63
                                          116 Harrisburg, PA                      43.19    4430.82
                                           18 Baltimore, MD                        70.5    4501.31
                                          294 Washington, DC                      40.06    4541.38
                                          220 Richmond, VA                        97.21    4638.58
                                          208 Portsmouth, VA                      94.39    4732.97
                                          184 Norfolk, VA                          1.19    4734.16
                                          215 Raleigh, NC                        178.76    4912.92
                                           80 Durham, NC                          23.61    4936.54
                                          109 Greensboro, NC                      61.97    4998.51
                                          306 Winston-Salem, NC                    31.3     5029.8
                                           54 Charlotte, NC                       73.13    5102.94
                                          260 Spartanburg, NC                     77.64    5180.58
                                          110 Greenville, SC                      32.62    5213.19
                                           10 Asheville, NC                       52.87    5266.06
                                          139 Knoxville, TN                       97.65    5363.71
                                           56 Chattanooga, TN                    114.91    5478.63
                                          100 Gadsden, AL                         86.02    5564.64
                                          122 Huntsville, AL                      63.64    5628.29
                                           30 Birmingham, AL                      84.91    5713.19
                                          170 Montgomery, AL                      86.96    5800.16
                                           65 Columbus, GA                         90.9    5891.06
                                          157 Macon, GA                           97.26    5988.32
                                           12 Atlanta, GA                         81.64    6069.95
                                           14 Augusta, GA                        167.83    6237.79
                                           64 Columbia, SC                        74.55    6312.34
                                           52 Charleston, SC                      113.9    6426.24
                                          250 Savannah, GA                         93.9    6520.14
                                          127 Jacksonville, FL                   126.96    6647.09
                                          101 Gainesville, FL                     65.95    6713.05
                                           72 Daytona Beach, FL                   94.98    6808.02
                                          190 Orlando, FL                         52.58    6860.61
                                          275 Tampa, FL                           85.01    6945.62
                                          234 Saint Petersburg, FL                19.54    6965.15
                                          247 Sarasota, FL                        31.73    6996.88
                                          296 West Palm Beach, FL                176.46    7173.34
                                          164 Miami, FL                           65.75    7239.09
                                          136 Key West, FL                       138.36    7377.45
                                          274 Tallahassee, FL                    441.61    7819.06
                                          195 Pensacola, FL                      202.87    8021.93
                                          168 Mobile, AL                          60.12    8082.04
                                           28 Biloxi, MS                          61.74    8143.79
                                          112 Gulfport, MS                        14.47    8158.25
                                          180 New Orleans, LA                     73.62    8231.87
                                           20 Baton Rouge, LA                     82.09    8313.96
                                          176 Natchez, MS                         78.58    8392.54
                                          126 Jackson, MS                         98.43    8490.97
                                          162 Memphis, TN                         197.2    8688.17
                                          151 Little Rock, AR                     157.3    8845.47
                                          277 Texarkana, TX                      151.96    8997.42
                                          160 Marshall, TX                        64.71    9062.13
                                          256 Shreveport, LA                      42.67    9104.81
                                           23 Beaumont, TX                       170.28    9275.09
                                          204 Port Arthur, TX                     17.62    9292.71
                                          103 Galveston, TX                       72.86    9365.57
                                          121 Houston, TX                         50.45    9416.02
                                           16 Austin, TX                         168.07     9584.1
                                          238 San Antonio, TX                     77.99    9662.09
                                           68 Corpus Christi, TX                 135.39    9797.48
                                          143 Laredo, TX                         147.28    9944.76
                                          292 Waco, TX                            323.5   10268.26
                                           97 Ft Worth, TX                        82.15   10350.41
                                           69 Dallas, TX                          36.19    10386.6
                                          188 Oklahoma City, OK                  191.95   10578.55
                                           85 Enid, OK                            68.83   10647.38
                                          300 Wichita, KS                         97.07   10744.46
                                          236 Salina, KS                          81.55      10826
                                          281 Topeka, KS                         134.35   10960.35
                                          133 Kansas City, KS                     72.73   11033.09
                                          134 Kansas City, MO                      3.52   11036.61
                                          231 Saint Joseph, MO                    49.79    11086.4
                                          189 Omaha, NE                          127.61   11214.01
                                          150 Lincoln, NE                         59.52   11273.53
                                          257 Sioux City, IA                     118.91   11392.44
                                          258 Sioux Falls, SD                     75.45   11467.89
                                          228 Saint Cloud, MN                    223.72   11691.61
                                          166 Minneapolis, MN                     73.94   11765.55
                                          233 Saint Paul, MN                      12.03   11777.58
                                          222 Rochester, MN                       76.94   11854.52
                                           81 Eau Claire, WI                       86.5   11941.02
                                          270 Superior, WI                       138.42   12079.44
                                           79 Duluth, MN                           4.32   12083.76
                                          278 Thunder Bay, ON                    221.38   12305.14
                                          108 Green Bay, WI                      281.05   12586.19
                                          253 Sheboygan, WI                       57.13   12643.32
                                          158 Madison, WI                         125.6   12768.91
                                           78 Dubuque, IA                         95.84   12864.75
                                           49 Cedar Rapids, IA                    75.74    12940.5
                                          124 Iowa City, IA                       25.25   12965.74
                                          295 Waterloo, IA                        80.35    13046.1
                                           75 Des Moines, IA                     107.02   13153.12
                                           63 Columbia, MO                       203.13   13356.25
                                          264 Springfield, MO                    137.23   13493.48
                                          130 Joplin, MO                          84.44   13577.92
                                           95 Ft Smith, AR                       117.62   13695.54
                                          287 Tulsa, OK                          122.27    13817.8
                                           77 Dodge City, KS                     299.19      14117
                                            7 Amarillo, TX                       215.16   14332.15
                                          156 Lubbock, TX                        113.61   14445.76
                                            1 Abilene, TX                        166.25   14612.02
                                           84 El Paso, TX                        469.27   15081.29
                                            4 Albuquerque, NM                    230.08   15311.37
                                          246 Santa Fe, NM                        64.52   15375.88
                                          102 Gallup, NM                         194.11   15569.99
                                           93 Flagstaff, AZ                      202.26   15772.25
                                          199 Phoenix, AZ                        124.38   15896.63
                                          286 Tucson, AZ                         116.06   16012.69
                                          311 Yuma, AZ                           257.86   16270.55
                                          240 San Diego, CA                      175.01   16445.56
                                          239 San Bernardino, CA                  96.69   16542.25
                                          193 Pasadena, CA                        59.12   16601.38
                                          153 Los Angeles, CA                      9.51   16610.89
                                          244 Santa Barbara, CA                  103.68   16714.56
                                           17 Bakersfield, CA                     80.84   16795.41
                                           99 Fresno, CA                         108.31   16903.71
                                          268 Stockton, CA                       134.15   17037.87
                                          225 Sacramento, CA                      45.35   17083.21
                                           26 Berkeley, CA                         72.8   17156.01
                                          186 Oakland, CA                          4.65   17160.66
                                          241 San Francisco, CA                   10.47   17171.13
                                          242 San Jose, CA                        47.11   17218.24
                                          245 Santa Cruz, CA                      26.93   17245.17
                                           48 Carson City, NV                    217.59   17462.76
                                          219 Reno, NV                            25.48   17488.24
                                           88 Eureka, CA                         313.15   17801.39
                                           87 Eugene, OR                         236.57   18037.96
                                          235 Salem, OR                           61.63   18099.59
                                          206 Portland, OR                        47.17   18146.76
                                          273 Tacoma, WA                         120.57   18267.33
                                          252 Seattle, WA                         25.62   18292.95
                                           25 Bellingham, WA                      80.42   18373.37
                                          290 Vancouver, BC                       56.66   18430.03
                                          291 Victoria, BC                        41.45   18471.48
                                          308 Yakima, WA                         246.37   18717.85
                                          293 Walla Walla, WA                    153.98   18871.83
                                          261 Spokane, WA                        127.07    18998.9
                                           33 Boise, ID                          291.99    19290.9
                                          203 Pocatello, ID                      264.67   19555.56
                                          187 Ogden, UT                          118.47   19674.03
                                          237 Salt Lake City, UT                  32.44   19706.47
                                          211 Provo, UT                           39.79   19746.26
                                          105 Grand Junction, CO                 229.45   19975.72
                                           74 Denver, CO                         250.76   20226.47
                                           62 Colorado Springs, CO                63.56   20290.03
                                          212 Pueblo, CO                          42.63   20332.67
                                           57 Cheyenne, WY                       199.91   20532.58
                                          216 Rapid City, SD                     230.96   20763.54
                                          200 Pierre, SD                         199.98   20963.52
                                           31 Bismarck, ND                       171.22   21134.74
                                          167 Minot, ND                          104.59   21239.32
                                           36 Brandon, MB                        148.55   21387.87
                                          305 Winnipeg, MB                       187.08   21574.95
                                           92 Fargo, ND                          211.94   21786.89
                                          218 Regina, SK                         597.09   22383.98
                                          173 Moose Jaw, SK                       62.04   22446.03
                                          248 Saskatoon, SK                      149.68    22595.7
                                          161 Medicine Hat, AB                   315.13   22910.83
                                          146 Lethbridge, AB                     146.13   23056.96
                                           45 Calgary, AB                        132.75    23189.7
                                           82 Edmonton, AB                       173.35   23363.06
                                          107 Great Falls, MT                    446.74    23809.8
                                          118 Helena, MT                           80.7    23890.5
                                           44 Butte, MT                           53.31   23943.81
                                           27 Billings, MT                       279.14   24222.95
                                          255 Sheridan, WY                       126.61   24349.56
                                          144 Las Vegas, NV                      821.24    25170.8
                                          209 Prince Rupert, BC                 1638.53   26809.33
                                          131 Juneau, AK                         390.43   27199.75
                                          298 Whitehorse, YK                     172.32   27372.07
                                           70 Dawson, YT                         362.92      27735
                                           90 Fairbanks, AK                         596      28331
                                            8 Anchorage, AK                      292.11   28623.11
                                          183 Nome, AK                          1095.15   29718.26
                                          148 Lihue, HI                          2967.4   32685.65
                                          120 Honolulu, HI                        114.5   32800.16
                                          119 Hilo, HI                              176   32976.16
                                          309 Yellowknife, NT                   4111.93   37088.09
                                           59 Churchill, MB                     1431.71    38519.8
                                          249 Sault Ste Marie, ON               1073.27   39593.07
                                          269 Sudbury, ON                        256.57   39849.65
                                          185 North Bay, ON                          93   39942.65
                                          279 Timmins, ON                        198.01   40140.66
                                          152 London, ON                         387.85   40528.51
                                          223 Rochester, NY                       249.8    40778.3
                                          272 Syracuse, NY                       101.71   40880.01
                                          289 Utica, NY                           63.31   40943.32
                                           29 Binghamtom, NY                      83.89   41027.21
                                          301 Wilkes-Barre, PA                    58.97   41086.18
                                            6 Allentown, PA                       51.68   41137.86
                                           13 Atlantic City, NJ                  113.26   41251.12
                                           50 Central Islip, NY                  129.74   41380.86
                                          202 Pittsfield, MA                      114.7   41495.56
                                          285 Troy, NY                            36.34    41531.9
                                            3 Albany, NY                           6.88   41538.78
                                          251 Schenectady, NY                     16.88   41555.67
                                          307 Worcester, MA                      152.51   41708.18
                                          210 Providence, RI                      40.53   41748.71
                                           91 Fall River, MA                      19.72   41768.43
                                           40 Brockton, MA                        28.03   41796.45
                                           34 Boston, MA                          19.21   41815.67
                                           46 Cambridge, MA                        3.36   41819.03
                                          155 Lowell, MA                          23.03   41842.06
                                          145 Lawrence, MA                        11.73   41853.79
                                          159 Manchester, NH                      28.36   41882.14
                                           67 Concord, NH                         15.76    41897.9
                                          207 Portsmouth, NH                      54.37   41952.27
                                          205 Portland, ME                        53.74   42006.01
                                           15 Augusta, ME                         55.61   42061.62
                                           19 Bangor, ME                          77.07    42138.7
                                           98 Fredericton, NB                    171.89   42310.58
                                          229 Saint John, NB                      58.53   42369.11
                                          169 Moncton, NB                         99.94   42469.06
                                           55 Charlottetown, PE                  108.58   42577.64
                                          113 Halifax, NS                        100.98   42678.62
                                          271 Syndey, NS                         254.97   42933.59
                                          230 Saint John's, NF                   514.07   43447.66
                                          213 Quebec City, QC                   1290.47   44738.13
                                          284 Trois-Rivieres, QC                 190.67    44928.8
                                          254 Sherbrooke, QC                      93.08   45021.87
                                          221 Roanoke, VA                        802.17   45824.05
                                           53 Charleston, WV                     138.61   45962.65
                                           11 Ashland, KY                         70.03   46032.68
                                          149 Lima, OH                           186.42    46219.1
                                          232 Saint Louis, MO                    445.62   46664.72
                                          303 Wilmington, NC                     899.59   47564.31
                                          243 San Juan, PR                      1361.81   48926.12
                                            5 Alert, NT                         4433.43   53359.55
                                           51 Champaign, IL                     3435.34   56794.89

1252 rows selected.

Elapsed: 00:01:25.25
</pre>
</div>

#### Longest Routes
<div class="scrollbox">

<pre>
ROOT_LEG       PATH_RNK   TOT_DIST    TOWN_ID NAME                             LEG_DIST   CUM_DIST
------------ ---------- ---------- ---------- ------------------------------ ---------- ----------
183 -> 230            1  649684.75        183 Nome, AK
                                          230 Saint John's, NF                  7870.89    7870.89
                                          148 Lihue, HI                         7576.95   15447.85
                                            5 Alert, NT                         7905.76   23353.61
                                          119 Hilo, HI                          7786.39   31139.99
                                          271 Syndey, NS                        6867.36   38007.35
                                          120 Honolulu, HI                      6973.04   44980.39
                                           55 Charlottetown, PE                 6769.69   51750.08
                                            8 Anchorage, AK                     6084.17   57834.25
                                          243 San Juan, PR                       6499.8   64334.05
                                           90 Fairbanks, AK                     6485.59   70819.64
                                          113 Halifax, NS                        5979.6   76799.24
                                           70 Dawson, YT                        5390.11   82189.35
                                          169 Moncton, NB                       5290.95    87480.3
                                          298 Whitehorse, YK                    4966.85   92447.16
                                          229 Saint John, NB                    4896.82   97343.98
                                          131 Juneau, AK                        4819.35  102163.34
                                           98 Fredericton, NB                   4768.78  106932.11
                                          209 Prince Rupert, BC                 4454.24  111386.35
                                           19 Bangor, ME                        4310.99  115697.34
                                           88 Eureka, CA                        3836.77  119534.11
                                           15 Augusta, ME                        3765.4  123299.51
                                          291 Victoria, BC                      3718.78  127018.29
                                          205 Portland, ME                      3689.93  130708.23
                                          290 Vancouver, BC                      3676.9  134385.13
                                          207 Portsmouth, NH                    3646.55  138031.68
                                           87 Eugene, OR                        3615.89  141647.57
                                           40 Brockton, MA                      3600.15  145247.72
                                          235 Salem, OR                         3599.44  148847.16
                                           34 Boston, MA                        3595.59  152442.75
                                           25 Bellingham, WA                    3580.78  156023.53
                                           91 Fall River, MA                    3580.15  159603.68
                                          206 Portland, OR                      3569.55  163173.24
                                           46 Cambridge, MA                     3569.83  166743.06
                                          273 Tacoma, WA                        3563.14   170306.2
                                          145 Lawrence, MA                      3557.08  173863.27
                                          241 San Francisco, CA                 3557.84  177421.11
                                          213 Quebec City, QC                      3607  181028.11
                                          186 Oakland, CA                       3596.53  184624.65
                                          155 Lowell, MA                        3536.42  188161.06
                                          252 Seattle, WA                       3541.57  191702.63
                                          210 Providence, RI                     3540.8  195243.43
                                           26 Berkeley, CA                      3524.68  198768.11
                                          254 Sherbrooke, QC                    3533.82  202301.93
                                          245 Santa Cruz, CA                    3527.03  205828.96
                                          159 Manchester, NH                    3519.15  209348.11
                                          242 San Jose, CA                      3506.93  212855.04
                                           67 Concord, NH                       3502.92  216357.96
                                          225 Sacramento, CA                    3466.46  219824.42
                                          307 Worcester, MA                     3442.81  223267.23
                                          268 Stockton, CA                      3432.24  226699.47
                                          284 Trois-Rivieres, QC                3408.48  230107.95
                                          244 Santa Barbara, CA                 3351.71  233459.66
                                          171 Montpelier, VT                     3326.1  236785.76
                                          308 Yakima, WA                        3315.62  240101.39
                                          263 Springfield, MA                   3325.25  243426.64
                                           99 Fresno, CA                        3280.93  246707.57
                                           38 Brattleboro, VT                   3289.35  249996.92
                                          219 Reno, NV                          3273.12  253270.05
                                          117 Hartford, CT                      3259.94  256529.98
                                           48 Carson City, NV                   3258.04  259788.02
                                          178 New Britain,CT                    3251.14  263039.16
                                           17 Bakersfield, CA                   3224.24   266263.4
                                           43 Burlington, VT                    3226.83  269490.23
                                          153 Los Angeles, CA                   3193.66  272683.89
                                          163 Meriden, CT                        3181.7  275865.59
                                          309 Yellowknife, NT                   3219.55  279085.14
                                          164 Miami, FL                         3467.82  282552.96
                                           82 Edmonton, AB                      2998.65  285551.61
                                          136 Key West, FL                      2971.34  288522.96
                                          261 Spokane, WA                       2934.85  291457.81
                                          179 New Haven, CT                     3105.67  294563.47
                                          193 Pasadena, CA                      3163.09  297726.57
                                          172 Montreal, QC                      3157.19  300883.76
                                          240 San Diego, CA                     3118.23  304001.99
                                          202 Pittsfield, MA                     3107.7  307109.68
                                          293 Walla Walla, WA                   3125.94  310235.63
                                           50 Central Islip, NY                 3140.18   313375.8
                                          239 San Bernardino, CA                3080.98  316456.79
                                           39 Bridgeport, CT                    3084.78  319541.56
                                           33 Boise, ID                         2975.73  322517.29
                                          266 Stamford, CT                      2952.96  325470.25
                                          311 Yuma, AZ                           2896.3  328366.55
                                          285 Troy, NY                          2911.41  331277.97
                                          144 Las Vegas, NV                      2899.2  334177.16
                                            3 Albany, NY                        2893.98  337071.14
                                           45 Calgary, AB                       2842.81  339913.95
                                          296 West Palm Beach, FL               2889.69  342803.64
                                          146 Lethbridge, AB                    2761.59  345565.23
                                          299 White Plains, NY                  2761.87   348327.1
                                           44 Butte, MT                         2700.79  351027.89
                                          181 New York, NY                      2687.05  353714.95
                                          199 Phoenix, AZ                       2677.74  356392.69
                                          251 Schenectady, NY                   2713.16  359105.85
                                          203 Pocatello, ID                     2660.51  361766.36
                                          128 Jersey City, NJ                    2655.1  364421.46
                                          118 Helena, MT                         2653.8  367075.27
                                          177 Newark, NJ                        2647.25  369722.51
                                           93 Flagstaff, AZ                     2617.66  372340.18
                                          194 Paterson, NJ                      2619.56  374959.74
                                          187 Ogden, UT                         2611.96   377571.7
                                           83 Elizabeth, NJ                     2609.47  380181.16
                                          107 Great Falls, MT                   2605.85  382787.01
                                           13 Atlantic City, NJ                  2609.3  385396.31
                                          161 Medicine Hat, AB                  2613.82  388010.13
                                          283 Trenton, NJ                        2576.5  390586.63
                                          237 Salt Lake City, UT                2566.97  393153.59
                                          289 Utica, NY                         2538.01   395691.6
                                          286 Tucson, AZ                        2578.23  398269.82
                                          191 Ottawa, ON                        2575.89  400845.71
                                          211 Provo, UT                          2487.2  403332.91
                                          198 Philadelphia, PA                  2521.63  405854.54
                                           27 Billings, MT                      2338.33  408192.87
                                          302 Wilmington, DE                    2314.81  410507.68
                                          248 Saskatoon, SK                     2317.41  412825.09
                                          247 Sarasota, FL                      2394.44  415219.54
                                           59 Churchill, MB                     2311.24  417530.77
                                          234 Saint Petersburg, FL              2279.52  419810.29
                                          173 Moose Jaw, SK                     2220.06  422030.35
                                          190 Orlando, FL                       2249.04  424279.39
                                          218 Regina, SK                        2208.47  426487.86
                                           72 Daytona Beach, FL                 2195.17  428683.03
                                          255 Sheridan, WY                      2090.58  430773.61
                                            6 Allentown, PA                     2193.28  432966.89
                                          102 Gallup, NM                        2324.19  435291.08
                                           29 Binghamtom, NY                    2312.96  437604.04
                                          105 Grand Junction, CO                2264.43  439868.47
                                          301 Wilkes-Barre, PA                  2262.27  442130.74
                                           84 El Paso, TX                       2213.93  444344.68
                                          137 Kingston, ON                      2237.67  446582.35
                                            4 Albuquerque, NM                    2170.5  448752.84
                                          272 Syracuse, NY                      2178.26   450931.1
                                          246 Santa Fe, NM                      2120.23  453051.33
                                          217 Reading, PA                       2098.29  455149.62
                                           74 Denver, CO                        2008.13  457157.75
                                          208 Portsmouth, VA                    1992.18  459149.93
                                           57 Cheyenne, WY                      1993.01  461142.94
                                          184 Norfolk, VA                       1993.78  463136.72
                                           62 Colorado Springs, CO              1976.45  465113.17
                                          129 Johnstown, PA                     1972.05  467085.22
                                          212 Pueblo, CO                        1959.85  469045.06
                                          141 Lancaster, PA                     1959.48  471004.54
                                          216 Rapid City, SD                    1881.23  472885.77
                                          303 Wilmington, NC                    1875.13   474760.9
                                          167 Minot, ND                         1881.45  476642.36
                                          275 Tampa, FL                         1912.76  478555.12
                                           36 Brandon, MB                       1940.02  480495.14
                                          101 Gainesville, FL                   1855.41  482350.55
                                           31 Bismarck, ND                      1741.25   484091.8
                                           18 Baltimore, MD                     1749.02  485840.82
                                          143 Laredo, TX                        1779.18  487620.01
                                           24 Belleville, ON                    1918.78  489538.79
                                          156 Lubbock, TX                       1847.76  491386.54
                                          116 Harrisburg, PA                    1786.27  493172.82
                                            7 Amarillo, TX                      1758.66  494931.47
                                          223 Rochester, NY                     1760.64  496692.12
                                           68 Corpus Christi, TX                1730.17  498422.29
                                          279 Timmins, ON                       1812.36  500234.65
                                          238 San Antonio, TX                   1774.49  502009.14
                                          185 North Bay, ON                     1763.41  503772.55
                                            1 Abilene, TX                       1701.18  505473.73
                                          197 Peterborough, ON                  1685.89  507159.62
                                           16 Austin, TX                         1650.4  508810.02
                                          269 Sudbury, ON                       1627.33  510437.35
                                          121 Houston, TX                       1539.52  511976.87
                                           41 Buffalo, NY                       1455.86  513432.73
                                           77 Dodge City, KS                    1503.01  514935.73
                                          294 Washington, DC                    1589.78  516525.51
                                          200 Pierre, SD                        1654.69   518180.2
                                           52 Charleston, SC                    1622.39  519802.59
                                          305 Winnipeg, MB                      1683.01  521485.61
                                          127 Jacksonville, FL                  1729.19   523214.8
                                           92 Fargo, ND                          1549.3   524764.1
                                          220 Richmond, VA                       1482.8  526246.89
                                          292 Waco, TX                          1422.07  527668.96
                                          282 Toronto, ON                       1483.47  529152.43
                                          103 Galveston, TX                     1453.39  530605.82
                                          182 Niagara Falls, ON                 1446.61  532052.43
                                           97 Ft Worth, TX                      1451.32  533503.75
                                          227 Saint Catherines, ON               1447.6  534951.35
                                           69 Dallas, TX                        1414.51  536365.85
                                           42 Burlington, ONT                   1383.45   537749.3
                                          204 Port Arthur, TX                   1349.79   539099.1
                                          111 Guelph, ON                        1334.48  540433.58
                                           23 Beaumont, TX                      1333.91  541767.49
                                          115 Hamilton, ON                      1332.75  543100.24
                                          188 Oklahoma City, OK                 1326.53  544426.77
                                          138 Kitchener, ON                     1306.17  545732.94
                                           85 Enid, OK                          1303.54  547036.48
                                          215 Raleigh, NC                       1330.04  548366.51
                                          258 Sioux Falls, SD                   1358.74  549725.26
                                          250 Savannah, GA                      1337.75  551063.01
                                          228 Saint Cloud, MN                   1296.84  552359.85
                                          274 Tallahassee, FL                   1248.21  553608.06
                                          278 Thunder Bay, ON                   1287.21  554895.27
                                          180 New Orleans, LA                   1273.63  556168.89
                                          249 Sault Ste Marie, ON               1205.19  557374.08
                                           20 Baton Rouge, LA                   1199.28  558573.36
                                           37 Brantford, ON                     1151.44   559724.8
                                          300 Wichita, KS                       1232.93  560957.73
                                           80 Durham, NC                         1279.4  562237.14
                                          236 Salina, KS                         1307.8  563544.94
                                          109 Greensboro, NC                    1245.96   564790.9
                                          257 Sioux City, IA                    1230.47  566021.37
                                           64 Columbia, SC                      1213.26  567234.63
                                          150 Lincoln, NE                       1177.83  568412.46
                                          221 Roanoke, VA                       1181.08  569593.54
                                          189 Omaha, NE                         1139.08  570732.62
                                          306 Winston-Salem, NC                 1141.42  571874.04
                                           79 Duluth, MN                        1103.04  572977.08
                                          195 Pensacola, FL                     1179.96  574157.04
                                          270 Superior, WI                      1175.77  575332.81
                                           14 Augusta, GA                       1152.38  576485.19
                                          166 Minneapolis, MN                    1113.9  577599.09
                                           54 Charlotte, NC                     1091.15  578690.24
                                          233 Saint Paul, MN                    1080.39  579770.63
                                          157 Macon, GA                         1061.49  580832.12
                                           81 Eau Claire, WI                     989.72  581821.84
                                           28 Biloxi, MS                        1012.28  582834.12
                                          152 London, ON                        1016.65  583850.76
                                          160 Marshall, TX                      1158.56  585009.32
                                           86 Erie, PA                          1188.43  586197.75
                                          287 Tulsa, OK                          1174.1  587371.86
                                          201 Pittsburgh, PA                    1144.29  588516.15
                                          256 Shreveport, LA                    1096.48  589612.63
                                          310 Youngstown, OH                    1081.83  590694.46
                                          277 Texarkana, TX                     1066.85  591761.31
                                          267 Steubenville, OH                  1043.65  592804.96
                                          281 Topeka, KS                        1043.45  593848.41
                                          297 Wheeling, WV                      1035.82  594884.23
                                           95 Ft Smith, AR                       998.79  595883.02
                                           47 Canton, OH                         974.26  596857.28
                                          130 Joplin, MO                         943.13  597800.41
                                            2 Akron, OH                          939.34  598739.75
                                          176 Natchez, MS                        948.25  599688.01
                                           22 Bay City, MI                       980.29   600668.3
                                          112 Gulfport, MS                       982.13  601650.43
                                          108 Green Bay, WI                      980.65  602631.08
                                          168 Mobile, AL                         955.25  603586.33
                                          222 Rochester, MN                      970.35  604556.68
                                          260 Spartanburg, NC                    960.77  605517.44
                                          231 Saint Joseph, MO                   952.41  606469.86
                                          110 Greenville, SC                     925.02  607394.87
                                           75 Des Moines, IA                     904.35  608299.23
                                           10 Asheville, NC                      869.08   609168.3
                                          133 Kansas City, KS                     868.8   610037.1
                                           61 Cleveland, OH                      908.59  610945.69
                                          134 Kansas City, MO                    905.45  611851.14
                                           53 Charleston, WV                     895.98  612747.12
                                          264 Springfield, MO                    809.82  613556.95
                                          312 Zanesville, OH                     802.13  614359.08
                                          151 Little Rock, AR                    795.57  615154.65
                                          226 Saginaw, MI                        831.32  615985.97
                                          126 Jackson, MS                         880.9  616866.87
                                           94 Flint, MI                          865.78  617732.65
                                          170 Montgomery, AL                     757.41  618490.06
                                          295 Waterloo, IA                       814.78  619304.84
                                           65 Columbus, GA                       859.51  620164.34
                                           49 Cedar Rapids, IA                   804.18  620968.53
                                           12 Atlanta, GA                        759.63  621728.16
                                           78 Dubuque, IA                        744.13  622472.29
                                           30 Birmingham, AL                     675.42   623147.7
                                          253 Sheboygan, WI                      709.66  623857.37
                                          100 Gadsden, AL                        683.03   624540.4
                                          158 Madison, WI                        668.43  625208.83
                                           56 Chattanooga, TN                    622.55  625831.37
                                          124 Iowa City, IA                      627.43   626458.8
                                          139 Knoxville, TN                      656.94  627115.74
                                           63 Columbia, MO                       616.95  627732.69
                                          304 Windsor, ON                         687.3  628419.98
                                          162 Memphis, TN                        694.56  629114.55
                                           76 Detroit, MI                        693.11  629807.66
                                          122 Huntsville, AL                     579.38  630387.04
                                          165 Milwaukee, WI                      581.29  630968.33
                                           11 Ashland, KY                        481.46  631449.79
                                          232 Saint Louis, MO                    522.45  631972.25
                                            9 Ann Arbor, MI                      513.15   632485.4
                                          192 Paducah, KY                        491.81   632977.2
                                          142 Lansing, MI                        480.06  633457.26
                                          175 Nashville, TN                      479.16  633936.42
                                          106 Grand Rapids, MI                   475.97  634412.39
                                           35 Bowling Green, KY                  416.17  634828.56
                                          224 Rockford, IL                       408.25  635236.82
                                           66 Columbus, OH                       450.37  635687.19
                                          262 Springfield, IL                    459.25  636146.44
                                          280 Toledo, OH                         439.91  636586.35
                                          196 Peoria, IL                         422.25  637008.59
                                          265 Springfield, OH                    402.89  637411.49
                                           32 Bloomington, IL                    360.32  637771.81
                                          147 Lexington, KY                      356.49  638128.31
                                          214 Racine, WI                         399.12  638527.43
                                          154 Louisville, KY                     339.14  638866.57
                                          135 Kenosha, WI                         331.4  639197.97
                                           60 Cincinnati, OH                     331.61  639529.58
                                           73 Decatur, IL                        314.29  639843.88
                                          125 Jackson, MI                        355.83   640199.7
                                           89 Evansville, IN                     366.87  640566.58
                                           21 Battle Creek, MI                   342.27  640908.84
                                           51 Champaign, IL                       260.8  641169.64
                                          149 Lima, OH                           289.17  641458.82
                                          288 Urbana, IL                         286.77  641745.59
                                           71 Dayton, OH                         278.52  642024.11
                                           58 Chicago, IL                        279.24  642303.35
                                          114 Hamilton, OH                       272.42  642575.77
                                          104 Gary, IN                           244.97  642820.74
                                          174 Muncie, IN                         166.43  642987.17
                                          276 Terre Haute, IN                    148.82  643135.98
                                          132 Kalamazoo, MI                      232.44  643368.42
                                          123 Indianapolis, IN                   178.76  643547.18
                                          259 South Bend, IN                     132.47  643679.65
                                          140 Lafayette, IN                       97.61  643777.26
                                           96 Ft Wayne, IN                       130.36  643907.62
                                          183 Nome, AK                          5777.13  649684.75
                      2  649684.62        183 Nome, AK
                                          230 Saint John's, NF                  7870.89    7870.89
                                          148 Lihue, HI                         7576.95   15447.85
                                            5 Alert, NT                         7905.76   23353.61
                                          119 Hilo, HI                          7786.39   31139.99
                                          271 Syndey, NS                        6867.36   38007.35
                                          120 Honolulu, HI                      6973.04   44980.39
                                           55 Charlottetown, PE                 6769.69   51750.08
                                            8 Anchorage, AK                     6084.17   57834.25
                                          243 San Juan, PR                       6499.8   64334.05
                                           90 Fairbanks, AK                     6485.59   70819.64
                                          113 Halifax, NS                        5979.6   76799.24
                                           70 Dawson, YT                        5390.11   82189.35
                                          169 Moncton, NB                       5290.95    87480.3
                                          298 Whitehorse, YK                    4966.85   92447.16
                                          229 Saint John, NB                    4896.82   97343.98
                                          131 Juneau, AK                        4819.35  102163.34
                                           98 Fredericton, NB                   4768.78  106932.11
                                          209 Prince Rupert, BC                 4454.24  111386.35
                                           19 Bangor, ME                        4310.99  115697.34
                                           88 Eureka, CA                        3836.77  119534.11
                                           15 Augusta, ME                        3765.4  123299.51
                                          291 Victoria, BC                      3718.78  127018.29
                                          205 Portland, ME                      3689.93  130708.23
                                          290 Vancouver, BC                      3676.9  134385.13
                                          207 Portsmouth, NH                    3646.55  138031.68
                                           87 Eugene, OR                        3615.89  141647.57
                                           40 Brockton, MA                      3600.15  145247.72
                                          235 Salem, OR                         3599.44  148847.16
                                           34 Boston, MA                        3595.59  152442.75
                                           25 Bellingham, WA                    3580.78  156023.53
                                           91 Fall River, MA                    3580.15  159603.68
                                          206 Portland, OR                      3569.55  163173.24
                                           46 Cambridge, MA                     3569.83  166743.06
                                          273 Tacoma, WA                        3563.14   170306.2
                                          145 Lawrence, MA                      3557.08  173863.27
                                          241 San Francisco, CA                 3557.84  177421.11
                                          213 Quebec City, QC                      3607  181028.11
                                          186 Oakland, CA                       3596.53  184624.65
                                          155 Lowell, MA                        3536.42  188161.06
                                          252 Seattle, WA                       3541.57  191702.63
                                          210 Providence, RI                     3540.8  195243.43
                                           26 Berkeley, CA                      3524.68  198768.11
                                          254 Sherbrooke, QC                    3533.82  202301.93
                                          245 Santa Cruz, CA                    3527.03  205828.96
                                          159 Manchester, NH                    3519.15  209348.11
                                          242 San Jose, CA                      3506.93  212855.04
                                           67 Concord, NH                       3502.92  216357.96
                                          225 Sacramento, CA                    3466.46  219824.42
                                          307 Worcester, MA                     3442.81  223267.23
                                          268 Stockton, CA                      3432.24  226699.47
                                          284 Trois-Rivieres, QC                3408.48  230107.95
                                          244 Santa Barbara, CA                 3351.71  233459.66
                                          171 Montpelier, VT                     3326.1  236785.76
                                          308 Yakima, WA                        3315.62  240101.39
                                          263 Springfield, MA                   3325.25  243426.64
                                           99 Fresno, CA                        3280.93  246707.57
                                           38 Brattleboro, VT                   3289.35  249996.92
                                          219 Reno, NV                          3273.12  253270.05
                                          117 Hartford, CT                      3259.94  256529.98
                                           48 Carson City, NV                   3258.04  259788.02
                                          178 New Britain,CT                    3251.14  263039.16
                                           17 Bakersfield, CA                   3224.24   266263.4
                                           43 Burlington, VT                    3226.83  269490.23
                                          153 Los Angeles, CA                   3193.66  272683.89
                                          163 Meriden, CT                        3181.7  275865.59
                                          309 Yellowknife, NT                   3219.55  279085.14
                                          164 Miami, FL                         3467.82  282552.96
                                           82 Edmonton, AB                      2998.65  285551.61
                                          136 Key West, FL                      2971.34  288522.96
                                          261 Spokane, WA                       2934.85  291457.81
                                          179 New Haven, CT                     3105.67  294563.47
                                          193 Pasadena, CA                      3163.09  297726.57
                                          172 Montreal, QC                      3157.19  300883.76
                                          240 San Diego, CA                     3118.23  304001.99
                                          202 Pittsfield, MA                     3107.7  307109.68
                                          293 Walla Walla, WA                   3125.94  310235.63
                                           50 Central Islip, NY                 3140.18   313375.8
                                          239 San Bernardino, CA                3080.98  316456.79
                                           39 Bridgeport, CT                    3084.78  319541.56
                                           33 Boise, ID                         2975.73  322517.29
                                          266 Stamford, CT                      2952.96  325470.25
                                          311 Yuma, AZ                           2896.3  328366.55
                                          285 Troy, NY                          2911.41  331277.97
                                          144 Las Vegas, NV                      2899.2  334177.16
                                            3 Albany, NY                        2893.98  337071.14
                                           45 Calgary, AB                       2842.81  339913.95
                                          296 West Palm Beach, FL               2889.69  342803.64
                                          146 Lethbridge, AB                    2761.59  345565.23
                                          299 White Plains, NY                  2761.87   348327.1
                                           44 Butte, MT                         2700.79  351027.89
                                          181 New York, NY                      2687.05  353714.95
                                          199 Phoenix, AZ                       2677.74  356392.69
                                          251 Schenectady, NY                   2713.16  359105.85
                                          203 Pocatello, ID                     2660.51  361766.36
                                          128 Jersey City, NJ                    2655.1  364421.46
                                          118 Helena, MT                         2653.8  367075.27
                                          177 Newark, NJ                        2647.25  369722.51
                                           93 Flagstaff, AZ                     2617.66  372340.18
                                          194 Paterson, NJ                      2619.56  374959.74
                                          187 Ogden, UT                         2611.96   377571.7
                                           83 Elizabeth, NJ                     2609.47  380181.16
                                          107 Great Falls, MT                   2605.85  382787.01
                                           13 Atlantic City, NJ                  2609.3  385396.31
                                          161 Medicine Hat, AB                  2613.82  388010.13
                                          283 Trenton, NJ                        2576.5  390586.63
                                          237 Salt Lake City, UT                2566.97  393153.59
                                          289 Utica, NY                         2538.01   395691.6
                                          286 Tucson, AZ                        2578.23  398269.82
                                          191 Ottawa, ON                        2575.89  400845.71
                                          211 Provo, UT                          2487.2  403332.91
                                          198 Philadelphia, PA                  2521.63  405854.54
                                           27 Billings, MT                      2338.33  408192.87
                                          302 Wilmington, DE                    2314.81  410507.68
                                          248 Saskatoon, SK                     2317.41  412825.09
                                          247 Sarasota, FL                      2394.44  415219.54
                                           59 Churchill, MB                     2311.24  417530.77
                                          234 Saint Petersburg, FL              2279.52  419810.29
                                          173 Moose Jaw, SK                     2220.06  422030.35
                                          190 Orlando, FL                       2249.04  424279.39
                                          218 Regina, SK                        2208.47  426487.86
                                           72 Daytona Beach, FL                 2195.17  428683.03
                                          255 Sheridan, WY                      2090.58  430773.61
                                            6 Allentown, PA                     2193.28  432966.89
                                          102 Gallup, NM                        2324.19  435291.08
                                           29 Binghamtom, NY                    2312.96  437604.04
                                          105 Grand Junction, CO                2264.43  439868.47
                                          301 Wilkes-Barre, PA                  2262.27  442130.74
                                           84 El Paso, TX                       2213.93  444344.68
                                          137 Kingston, ON                      2237.67  446582.35
                                            4 Albuquerque, NM                    2170.5  448752.84
                                          272 Syracuse, NY                      2178.26   450931.1
                                          246 Santa Fe, NM                      2120.23  453051.33
                                          217 Reading, PA                       2098.29  455149.62
                                           74 Denver, CO                        2008.13  457157.75
                                          208 Portsmouth, VA                    1992.18  459149.93
                                           57 Cheyenne, WY                      1993.01  461142.94
                                          184 Norfolk, VA                       1993.78  463136.72
                                           62 Colorado Springs, CO              1976.45  465113.17
                                          129 Johnstown, PA                     1972.05  467085.22
                                          212 Pueblo, CO                        1959.85  469045.06
                                          141 Lancaster, PA                     1959.48  471004.54
                                          216 Rapid City, SD                    1881.23  472885.77
                                          303 Wilmington, NC                    1875.13   474760.9
                                          167 Minot, ND                         1881.45  476642.36
                                          275 Tampa, FL                         1912.76  478555.12
                                           36 Brandon, MB                       1940.02  480495.14
                                          101 Gainesville, FL                   1855.41  482350.55
                                           31 Bismarck, ND                      1741.25   484091.8
                                           18 Baltimore, MD                     1749.02  485840.82
                                          143 Laredo, TX                        1779.18  487620.01
                                           24 Belleville, ON                    1918.78  489538.79
                                          156 Lubbock, TX                       1847.76  491386.54
                                          116 Harrisburg, PA                    1786.27  493172.82
                                            7 Amarillo, TX                      1758.66  494931.47
                                          223 Rochester, NY                     1760.64  496692.12
                                           68 Corpus Christi, TX                1730.17  498422.29
                                          279 Timmins, ON                       1812.36  500234.65
                                          238 San Antonio, TX                   1774.49  502009.14
                                          185 North Bay, ON                     1763.41  503772.55
                                            1 Abilene, TX                       1701.18  505473.73
                                          197 Peterborough, ON                  1685.89  507159.62
                                           16 Austin, TX                         1650.4  508810.02
                                          269 Sudbury, ON                       1627.33  510437.35
                                          121 Houston, TX                       1539.52  511976.87
                                           41 Buffalo, NY                       1455.86  513432.73
                                           77 Dodge City, KS                    1503.01  514935.73
                                          294 Washington, DC                    1589.78  516525.51
                                          200 Pierre, SD                        1654.69   518180.2
                                           52 Charleston, SC                    1622.39  519802.59
                                          305 Winnipeg, MB                      1683.01  521485.61
                                          127 Jacksonville, FL                  1729.19   523214.8
                                           92 Fargo, ND                          1549.3   524764.1
                                          220 Richmond, VA                       1482.8  526246.89
                                          292 Waco, TX                          1422.07  527668.96
                                          282 Toronto, ON                       1483.47  529152.43
                                          103 Galveston, TX                     1453.39  530605.82
                                          182 Niagara Falls, ON                 1446.61  532052.43
                                           97 Ft Worth, TX                      1451.32  533503.75
                                          227 Saint Catherines, ON               1447.6  534951.35
                                           69 Dallas, TX                        1414.51  536365.85
                                           42 Burlington, ONT                   1383.45   537749.3
                                           23 Beaumont, TX                      1349.56  539098.86
                                          111 Guelph, ON                        1333.91  540432.77
                                          204 Port Arthur, TX                   1334.48  541767.25
                                          115 Hamilton, ON                      1332.86  543100.11
                                          188 Oklahoma City, OK                 1326.53  544426.64
                                          138 Kitchener, ON                     1306.17  545732.81
                                           85 Enid, OK                          1303.54  547036.35
                                          215 Raleigh, NC                       1330.04  548366.39
                                          258 Sioux Falls, SD                   1358.74  549725.13
                                          250 Savannah, GA                      1337.75  551062.88
                                          228 Saint Cloud, MN                   1296.84  552359.72
                                          274 Tallahassee, FL                   1248.21  553607.93
                                          278 Thunder Bay, ON                   1287.21  554895.14
                                          180 New Orleans, LA                   1273.63  556168.77
                                          249 Sault Ste Marie, ON               1205.19  557373.95
                                           20 Baton Rouge, LA                   1199.28  558573.23
                                           37 Brantford, ON                     1151.44  559724.67
                                          300 Wichita, KS                       1232.93   560957.6
                                           80 Durham, NC                         1279.4  562237.01
                                          236 Salina, KS                         1307.8  563544.81
                                          109 Greensboro, NC                    1245.96  564790.77
                                          257 Sioux City, IA                    1230.47  566021.24
                                           64 Columbia, SC                      1213.26   567234.5
                                          150 Lincoln, NE                       1177.83  568412.33
                                          221 Roanoke, VA                       1181.08  569593.41
                                          189 Omaha, NE                         1139.08  570732.49
                                          306 Winston-Salem, NC                 1141.42  571873.91
                                           79 Duluth, MN                        1103.04  572976.95
                                          195 Pensacola, FL                     1179.96  574156.91
                                          270 Superior, WI                      1175.77  575332.68
                                           14 Augusta, GA                       1152.38  576485.06
                                          166 Minneapolis, MN                    1113.9  577598.96
                                           54 Charlotte, NC                     1091.15  578690.11
                                          233 Saint Paul, MN                    1080.39   579770.5
                                          157 Macon, GA                         1061.49  580831.99
                                           81 Eau Claire, WI                     989.72  581821.71
                                           28 Biloxi, MS                        1012.28  582833.99
                                          152 London, ON                        1016.65  583850.64
                                          160 Marshall, TX                      1158.56  585009.19
                                           86 Erie, PA                          1188.43  586197.63
                                          287 Tulsa, OK                          1174.1  587371.73
                                          201 Pittsburgh, PA                    1144.29  588516.02
                                          256 Shreveport, LA                    1096.48   589612.5
                                          310 Youngstown, OH                    1081.83  590694.33
                                          277 Texarkana, TX                     1066.85  591761.18
                                          267 Steubenville, OH                  1043.65  592804.83
                                          281 Topeka, KS                        1043.45  593848.28
                                          297 Wheeling, WV                      1035.82   594884.1
                                           95 Ft Smith, AR                       998.79  595882.89
                                           47 Canton, OH                         974.26  596857.16
                                          130 Joplin, MO                         943.13  597800.28
                                            2 Akron, OH                          939.34  598739.63
                                          176 Natchez, MS                        948.25  599687.88
                                           22 Bay City, MI                       980.29  600668.17
                                          112 Gulfport, MS                       982.13   601650.3
                                          108 Green Bay, WI                      980.65  602630.95
                                          168 Mobile, AL                         955.25   603586.2
                                          222 Rochester, MN                      970.35  604556.55
                                          260 Spartanburg, NC                    960.77  605517.32
                                          231 Saint Joseph, MO                   952.41  606469.73
                                          110 Greenville, SC                     925.02  607394.74
                                           75 Des Moines, IA                     904.35   608299.1
                                           10 Asheville, NC                      869.08  609168.17
                                          133 Kansas City, KS                     868.8  610036.97
                                           61 Cleveland, OH                      908.59  610945.56
                                          134 Kansas City, MO                    905.45  611851.01
                                           53 Charleston, WV                     895.98  612746.99
                                          264 Springfield, MO                    809.82  613556.82
                                          312 Zanesville, OH                     802.13  614358.95
                                          151 Little Rock, AR                    795.57  615154.52
                                          226 Saginaw, MI                        831.32  615985.85
                                          126 Jackson, MS                         880.9  616866.74
                                           94 Flint, MI                          865.78  617732.52
                                          170 Montgomery, AL                     757.41  618489.93
                                          295 Waterloo, IA                       814.78  619304.71
                                           65 Columbus, GA                       859.51  620164.22
                                           49 Cedar Rapids, IA                   804.18   620968.4
                                           12 Atlanta, GA                        759.63  621728.03
                                           78 Dubuque, IA                        744.13  622472.16
                                           30 Birmingham, AL                     675.42  623147.57
                                          253 Sheboygan, WI                      709.66  623857.24
                                          100 Gadsden, AL                        683.03  624540.27
                                          158 Madison, WI                        668.43   625208.7
                                           56 Chattanooga, TN                    622.55  625831.25
                                          124 Iowa City, IA                      627.43  626458.68
                                          139 Knoxville, TN                      656.94  627115.61
                                           63 Columbia, MO                       616.95  627732.56
                                          304 Windsor, ON                         687.3  628419.86
                                          162 Memphis, TN                        694.56  629114.42
                                           76 Detroit, MI                        693.11  629807.53
                                          122 Huntsville, AL                     579.38  630386.91
                                          165 Milwaukee, WI                      581.29   630968.2
                                           11 Ashland, KY                        481.46  631449.67
                                          232 Saint Louis, MO                    522.45  631972.12
                                            9 Ann Arbor, MI                      513.15  632485.27
                                          192 Paducah, KY                        491.81  632977.08
                                          142 Lansing, MI                        480.06  633457.13
                                          175 Nashville, TN                      479.16  633936.29
                                          106 Grand Rapids, MI                   475.97  634412.26
                                           35 Bowling Green, KY                  416.17  634828.44
                                          224 Rockford, IL                       408.25  635236.69
                                           66 Columbus, OH                       450.37  635687.06
                                          262 Springfield, IL                    459.25  636146.31
                                          280 Toledo, OH                         439.91  636586.22
                                          196 Peoria, IL                         422.25  637008.47
                                          265 Springfield, OH                    402.89  637411.36
                                           32 Bloomington, IL                    360.32  637771.68
                                          147 Lexington, KY                      356.49  638128.18
                                          214 Racine, WI                         399.12   638527.3
                                          154 Louisville, KY                     339.14  638866.44
                                          135 Kenosha, WI                         331.4  639197.84
                                           60 Cincinnati, OH                     331.61  639529.45
                                           73 Decatur, IL                        314.29  639843.75
                                          125 Jackson, MI                        355.83  640199.57
                                           89 Evansville, IN                     366.87  640566.45
                                           21 Battle Creek, MI                   342.27  640908.71
                                           51 Champaign, IL                       260.8  641169.51
                                          149 Lima, OH                           289.17  641458.69
                                          288 Urbana, IL                         286.77  641745.46
                                           71 Dayton, OH                         278.52  642023.98
                                           58 Chicago, IL                        279.24  642303.22
                                          114 Hamilton, OH                       272.42  642575.65
                                          104 Gary, IN                           244.97  642820.61
                                          174 Muncie, IN                         166.43  642987.04
                                          276 Terre Haute, IN                    148.82  643135.86
                                          132 Kalamazoo, MI                      232.44  643368.29
                                          123 Indianapolis, IN                   178.76  643547.05
                                          259 South Bend, IN                     132.47  643679.52
                                          140 Lafayette, IN                       97.61  643777.13
                                           96 Ft Wayne, IN                       130.36  643907.49
                                          183 Nome, AK                          5777.13  649684.62
5 -> 148              1  650614.77          5 Alert, NT
                                          148 Lihue, HI                         7905.76    7905.76
                                          230 Saint John's, NF                  7576.95   15482.71
                                          183 Nome, AK                          7870.89   23353.61
                                          271 Syndey, NS                        7389.33   30742.93
                                          120 Honolulu, HI                      6973.04   37715.97
                                           55 Charlottetown, PE                 6769.69   44485.66
                                          119 Hilo, HI                          6665.33   51150.99
                                          113 Halifax, NS                       6613.34   57764.33
                                            8 Anchorage, AK                     6075.13   63839.46
                                          243 San Juan, PR                       6499.8   70339.26
                                           90 Fairbanks, AK                     6485.59   76824.85
                                          169 Moncton, NB                        5881.4   82706.26
                                           70 Dawson, YT                        5290.95   87997.21
                                          229 Saint John, NB                    5223.21   93220.42
                                          298 Whitehorse, YK                    4896.82   98117.25
                                           98 Fredericton, NB                   4845.34  102962.58
                                          131 Juneau, AK                        4768.78  107731.36
                                           19 Bangor, ME                        4630.48  112361.84
                                          209 Prince Rupert, BC                 4310.99  116672.83
                                           15 Augusta, ME                       4247.95  120920.78
                                           88 Eureka, CA                         3765.4  124686.18
                                          205 Portland, ME                      3729.96  128416.14
                                          291 Victoria, BC                      3689.93  132106.07
                                          207 Portsmouth, NH                     3659.1  135765.18
                                          290 Vancouver, BC                     3646.55  139411.72
                                           40 Brockton, MA                      3637.58  143049.31
                                          235 Salem, OR                         3599.44  146648.74
                                           34 Boston, MA                        3595.59  150244.33
                                           87 Eugene, OR                        3596.62  153840.95
                                           46 Cambridge, MA                     3593.43  157434.38
                                           25 Bellingham, WA                    3577.51  161011.89
                                           91 Fall River, MA                    3580.15  164592.04
                                          206 Portland, OR                      3569.55  168161.59
                                          145 Lawrence, MA                      3564.53  171726.12
                                          241 San Francisco, CA                 3557.84  175283.96
                                          213 Quebec City, QC                      3607  178890.96
                                          186 Oakland, CA                       3596.53  182487.49
                                          155 Lowell, MA                        3536.42  186023.91
                                          273 Tacoma, WA                           3547  189570.91
                                          210 Providence, RI                    3545.83  193116.74
                                          252 Seattle, WA                        3540.8  196657.53
                                          159 Manchester, NH                    3529.69  200187.22
                                           26 Berkeley, CA                      3528.99  203716.21
                                          254 Sherbrooke, QC                    3533.82  207250.03
                                          245 Santa Cruz, CA                    3527.03  210777.06
                                           67 Concord, NH                       3515.24   214292.3
                                          242 San Jose, CA                      3502.92  217795.22
                                          307 Worcester, MA                     3477.76  221272.98
                                          225 Sacramento, CA                    3442.81  224715.79
                                          284 Trois-Rivieres, QC                 3415.3  228131.09
                                          268 Stockton, CA                      3408.48  231539.57
                                          171 Montpelier, VT                    3393.96  234933.53
                                          244 Santa Barbara, CA                  3326.1  238259.63
                                           38 Brattleboro, VT                   3308.76  241568.39
                                          308 Yakima, WA                           3323   244891.4
                                          263 Springfield, MA                   3325.25  248216.65
                                           99 Fresno, CA                        3280.93  251497.58
                                          117 Hartford, CT                      3271.83  254769.41
                                          219 Reno, NV                          3259.94  258029.35
                                          178 New Britain,CT                     3253.1  261282.44
                                           48 Carson City, NV                   3251.14  264533.58
                                          163 Meriden, CT                        3248.8  267782.39
                                           17 Bakersfield, CA                   3221.22  271003.61
                                           43 Burlington, VT                    3226.83  274230.43
                                          153 Los Angeles, CA                   3193.66   277424.1
                                          179 New Haven, CT                      3170.9  280594.99
                                          309 Yellowknife, NT                   3219.25  283814.25
                                          164 Miami, FL                         3467.82  287282.06
                                           82 Edmonton, AB                      2998.65  290280.72
                                          136 Key West, FL                      2971.34  293252.06
                                          261 Spokane, WA                       2934.85  296186.91
                                           50 Central Islip, NY                 3092.23  299279.15
                                          293 Walla Walla, WA                   3140.18  302419.32
                                           39 Bridgeport, CT                    3137.06  305556.38
                                          193 Pasadena, CA                      3142.69  308699.07
                                          172 Montreal, QC                      3157.19  311856.26
                                          240 San Diego, CA                     3118.23  314974.49
                                          202 Pittsfield, MA                     3107.7  318082.19
                                          239 San Bernardino, CA                3097.28  321179.47
                                          285 Troy, NY                          3070.66  324250.12
                                           33 Boise, ID                         2937.92  327188.04
                                          266 Stamford, CT                      2952.96     330141
                                          311 Yuma, AZ                           2896.3   333037.3
                                            3 Albany, NY                        2905.84  335943.14
                                          144 Las Vegas, NV                     2893.98  338837.12
                                          251 Schenectady, NY                   2883.23  341720.35
                                           45 Calgary, AB                       2828.13  344548.48
                                          296 West Palm Beach, FL               2889.69  347438.17
                                          146 Lethbridge, AB                    2761.59  350199.76
                                          299 White Plains, NY                  2761.87  352961.63
                                           44 Butte, MT                         2700.79  355662.42
                                          181 New York, NY                      2687.05  358349.48
                                          199 Phoenix, AZ                       2677.74  361027.22
                                          128 Jersey City, NJ                   2673.06  363700.28
                                          203 Pocatello, ID                      2655.1  366355.39
                                          177 Newark, NJ                        2648.53  369003.92
                                          118 Helena, MT                        2647.25  371651.17
                                           13 Atlantic City, NJ                 2646.38  374297.55
                                          161 Medicine Hat, AB                  2613.82  376911.37
                                           83 Elizabeth, NJ                     2604.16  379515.53
                                           93 Flagstaff, AZ                     2614.32  382129.85
                                          194 Paterson, NJ                      2619.56  384749.41
                                          187 Ogden, UT                         2611.96  387361.37
                                          283 Trenton, NJ                       2573.35  389934.72
                                          107 Great Falls, MT                   2575.55  392510.27
                                          198 Philadelphia, PA                  2550.72  395060.99
                                          237 Salt Lake City, UT                2538.22  397599.21
                                          289 Utica, NY                         2538.01  400137.22
                                          286 Tucson, AZ                        2578.23  402715.44
                                          191 Ottawa, ON                        2575.89  405291.33
                                          211 Provo, UT                          2487.2  407778.53
                                            6 Allentown, PA                     2499.13  410277.67
                                          102 Gallup, NM                        2324.19  412601.86
                                           29 Binghamtom, NY                    2312.96  414914.82
                                           27 Billings, MT                       2265.6  417180.42
                                          302 Wilmington, DE                    2314.81  419495.23
                                          248 Saskatoon, SK                     2317.41  421812.65
                                          247 Sarasota, FL                      2394.44  424207.09
                                           59 Churchill, MB                     2311.24  426518.32
                                          234 Saint Petersburg, FL              2279.52  428797.84
                                          173 Moose Jaw, SK                     2220.06   431017.9
                                          190 Orlando, FL                       2249.04  433266.94
                                          218 Regina, SK                        2208.47  435475.41
                                           72 Daytona Beach, FL                 2195.17  437670.58
                                          255 Sheridan, WY                      2090.58  439761.16
                                          184 Norfolk, VA                       2189.22  441950.37
                                          105 Grand Junction, CO                2234.59  444184.96
                                          301 Wilkes-Barre, PA                  2262.27  446447.23
                                           84 El Paso, TX                       2213.93  448661.17
                                          137 Kingston, ON                      2237.67  450898.84
                                            4 Albuquerque, NM                    2170.5  453069.33
                                          272 Syracuse, NY                      2178.26  455247.59
                                          246 Santa Fe, NM                      2120.23  457367.82
                                          217 Reading, PA                       2098.29  459466.11
                                           74 Denver, CO                        2008.13  461474.24
                                          208 Portsmouth, VA                    1992.18  463466.42
                                           57 Cheyenne, WY                      1993.01  465459.43
                                          141 Lancaster, PA                     1971.66  467431.09
                                           62 Colorado Springs, CO              1972.03  469403.12
                                          129 Johnstown, PA                     1972.05  471375.16
                                          212 Pueblo, CO                        1959.85  473335.01
                                           18 Baltimore, MD                     1935.73  475270.74
                                          216 Rapid City, SD                    1868.74  477139.48
                                          303 Wilmington, NC                    1875.13  479014.61
                                          167 Minot, ND                         1881.45  480896.06
                                          275 Tampa, FL                         1912.76  482808.82
                                           36 Brandon, MB                       1940.02  484748.84
                                          101 Gainesville, FL                   1855.41  486604.25
                                           31 Bismarck, ND                      1741.25  488345.51
                                          127 Jacksonville, FL                  1744.35  490089.86
                                          305 Winnipeg, MB                      1729.19  491819.04
                                           52 Charleston, SC                    1683.01  493502.06
                                          200 Pierre, SD                        1622.39  495124.45
                                          294 Washington, DC                    1654.69  496779.14
                                          156 Lubbock, TX                       1753.74  498532.87
                                           24 Belleville, ON                    1847.76  500380.63
                                          143 Laredo, TX                        1918.78  502299.41
                                          279 Timmins, ON                       1920.31  504219.72
                                           68 Corpus Christi, TX                1812.36  506032.08
                                          185 North Bay, ON                      1786.7  507818.78
                                          238 San Antonio, TX                   1763.41  509582.19
                                          223 Rochester, NY                     1726.57  511308.76
                                            7 Amarillo, TX                      1760.64   513069.4
                                          116 Harrisburg, PA                    1758.66  514828.06
                                            1 Abilene, TX                       1668.54   516496.6
                                          197 Peterborough, ON                  1685.89  518182.49
                                           16 Austin, TX                         1650.4  519832.89
                                          269 Sudbury, ON                       1627.33  521460.22
                                          121 Houston, TX                       1539.52  522999.73
                                           41 Buffalo, NY                       1455.86   524455.6
                                           77 Dodge City, KS                    1503.01   525958.6
                                          220 Richmond, VA                      1558.59  527517.19
                                           92 Fargo, ND                          1482.8  528999.99
                                          250 Savannah, GA                      1489.99  530489.98
                                          258 Sioux Falls, SD                   1337.75  531827.74
                                          215 Raleigh, NC                       1358.74  533186.48
                                           85 Enid, OK                          1330.04  534516.52
                                          182 Niagara Falls, ON                 1380.26  535896.78
                                          292 Waco, TX                          1483.03  537379.81
                                          282 Toronto, ON                       1483.47  538863.29
                                           97 Ft Worth, TX                       1449.6  540312.89
                                          227 Saint Catherines, ON               1447.6  541760.49
                                          103 Galveston, TX                     1444.61  543205.09
                                           42 Burlington, ONT                   1421.73  544626.82
                                           69 Dallas, TX                        1383.45  546010.27
                                          115 Hamilton, ON                      1368.85  547379.12
                                           23 Beaumont, TX                      1332.75  548711.87
                                          111 Guelph, ON                        1333.91  550045.78
                                          204 Port Arthur, TX                   1334.48  551380.26
                                          138 Kitchener, ON                     1324.27  552704.53
                                          188 Oklahoma City, OK                 1306.17   554010.7
                                           37 Brantford, ON                     1299.42  555310.11
                                          300 Wichita, KS                       1232.93  556543.05
                                           80 Durham, NC                         1279.4  557822.45
                                          236 Salina, KS                         1307.8  559130.25
                                          109 Greensboro, NC                    1245.96  560376.22
                                          257 Sioux City, IA                    1230.47  561606.68
                                           64 Columbia, SC                      1213.26  562819.94
                                          228 Saint Cloud, MN                   1208.61  564028.55
                                          274 Tallahassee, FL                   1248.21  565276.76
                                          278 Thunder Bay, ON                   1287.21  566563.97
                                          180 New Orleans, LA                   1273.63   567837.6
                                          249 Sault Ste Marie, ON               1205.19  569042.78
                                           20 Baton Rouge, LA                   1199.28  570242.06
                                           79 Duluth, MN                        1130.44  571372.51
                                          195 Pensacola, FL                     1179.96  572552.47
                                          270 Superior, WI                      1175.77  573728.24
                                           14 Augusta, GA                       1152.38  574880.62
                                          150 Lincoln, NE                       1134.43  576015.05
                                          221 Roanoke, VA                       1181.08  577196.13
                                          189 Omaha, NE                         1139.08  578335.21
                                          306 Winston-Salem, NC                 1141.42  579476.63
                                          166 Minneapolis, MN                   1088.91  580565.53
                                           54 Charlotte, NC                     1091.15  581656.69
                                          233 Saint Paul, MN                    1080.39  582737.08
                                          157 Macon, GA                         1061.49  583798.57
                                           81 Eau Claire, WI                     989.72  584788.28
                                           28 Biloxi, MS                        1012.28  585800.56
                                          152 London, ON                        1016.65  586817.21
                                          160 Marshall, TX                      1158.56  587975.77
                                           86 Erie, PA                          1188.43   589164.2
                                          287 Tulsa, OK                          1174.1   590338.3
                                          201 Pittsburgh, PA                    1144.29  591482.59
                                          256 Shreveport, LA                    1096.48  592579.08
                                          310 Youngstown, OH                    1081.83   593660.9
                                          277 Texarkana, TX                     1066.85  594727.75
                                          267 Steubenville, OH                  1043.65  595771.41
                                          281 Topeka, KS                        1043.45  596814.86
                                          297 Wheeling, WV                      1035.82  597850.68
                                           95 Ft Smith, AR                       998.79  598849.47
                                           61 Cleveland, OH                      974.07  599823.54
                                          176 Natchez, MS                        959.96  600783.49
                                           22 Bay City, MI                       980.29  601763.78
                                          112 Gulfport, MS                       982.13  602745.92
                                          108 Green Bay, WI                      980.65  603726.56
                                          168 Mobile, AL                         955.25  604681.82
                                          222 Rochester, MN                      970.35  605652.16
                                          260 Spartanburg, NC                    960.77  606612.93
                                          231 Saint Joseph, MO                   952.41  607565.34
                                           47 Canton, OH                         933.28  608498.62
                                          130 Joplin, MO                         943.13  609441.75
                                            2 Akron, OH                          939.34  610381.09
                                          133 Kansas City, KS                    915.85  611296.95
                                           53 Charleston, WV                     899.41  612196.36
                                          134 Kansas City, MO                    895.98  613092.34
                                          110 Greenville, SC                     891.55  613983.89
                                           75 Des Moines, IA                     904.35  614888.24
                                           10 Asheville, NC                      869.08  615757.32
                                          295 Waterloo, IA                       827.18   616584.5
                                           65 Columbus, GA                       859.51  617444.01
                                          253 Sheboygan, WI                      802.52  618246.52
                                          126 Jackson, MS                         809.5  619056.02
                                          226 Saginaw, MI                         880.9  619936.92
                                          151 Little Rock, AR                    831.32  620768.24
                                          304 Windsor, ON                        828.42  621596.66
                                          264 Springfield, MO                    795.38  622392.04
                                          312 Zanesville, OH                     802.13  623194.17
                                           63 Columbia, MO                       716.36  623910.53
                                           76 Detroit, MI                        682.93  624593.46
                                          170 Montgomery, AL                      724.3  625317.77
                                          158 Madison, WI                        770.17  626087.93
                                           12 Atlanta, GA                        731.47  626819.41
                                           49 Cedar Rapids, IA                   759.63  627579.04
                                          100 Gadsden, AL                        675.88  628254.92
                                           78 Dubuque, IA                        668.88   628923.8
                                           30 Birmingham, AL                     675.42  629599.22
                                           94 Flint, MI                          690.27  630289.49
                                          162 Memphis, TN                        698.85  630988.33
                                          142 Lansing, MI                           647  631635.33
                                          122 Huntsville, AL                     570.44  632205.78
                                          124 Iowa City, IA                      588.25  632794.02
                                          139 Knoxville, TN                      656.94  633450.96
                                          224 Rockford, IL                       563.81  634014.76
                                           56 Chattanooga, TN                    563.58  634578.34
                                          165 Milwaukee, WI                      580.71  635159.05
                                           11 Ashland, KY                        481.46  635640.51
                                          232 Saint Louis, MO                    522.45  636162.96
                                            9 Ann Arbor, MI                      513.15  636676.12
                                          192 Paducah, KY                        491.81  637167.92
                                          125 Jackson, MI                        459.78   637627.7
                                          175 Nashville, TN                      451.22  638078.92
                                          106 Grand Rapids, MI                   475.97  638554.89
                                           35 Bowling Green, KY                  416.17  638971.07
                                          214 Racine, WI                         406.98  639378.05
                                          147 Lexington, KY                      399.12  639777.17
                                          196 Peoria, IL                         399.56  640176.73
                                           66 Columbus, OH                       458.14  640634.87
                                          262 Springfield, IL                    459.25  641094.13
                                          280 Toledo, OH                         439.91  641534.03
                                           73 Decatur, IL                        393.78  641927.82
                                          265 Springfield, OH                    355.61  642283.42
                                           32 Bloomington, IL                    360.32  642643.75
                                          149 Lima, OH                           338.23  642981.98
                                           89 Evansville, IN                     305.64  643287.62
                                           21 Battle Creek, MI                   342.27  643629.89
                                          154 Louisville, KY                     283.85  643913.74
                                          135 Kenosha, WI                         331.4  644245.14
                                           60 Cincinnati, OH                     331.61  644576.75
                                           58 Chicago, IL                         288.4  644865.14
                                           71 Dayton, OH                         279.24  645144.39
                                           51 Champaign, IL                      281.04  645425.42
                                          114 Hamilton, OH                       259.18   645684.6
                                          288 Urbana, IL                         256.66  645941.26
                                          132 Kalamazoo, MI                      235.55  646176.81
                                          276 Terre Haute, IN                    232.44  646409.25
                                           96 Ft Wayne, IN                        195.3  646604.55
                                          104 Gary, IN                           156.52  646761.07
                                          174 Muncie, IN                         166.43   646927.5
                                          259 South Bend, IN                     118.99  647046.49
                                          123 Indianapolis, IN                   132.47  647178.96
                                          140 Lafayette, IN                       66.81  647245.77
                                            5 Alert, NT                         3369.01  650614.77
                      2  650613.92          5 Alert, NT
                                          148 Lihue, HI                         7905.76    7905.76
                                          230 Saint John's, NF                  7576.95   15482.71
                                          183 Nome, AK                          7870.89   23353.61
                                          271 Syndey, NS                        7389.33   30742.93
                                          120 Honolulu, HI                      6973.04   37715.97
                                           55 Charlottetown, PE                 6769.69   44485.66
                                          119 Hilo, HI                          6665.33   51150.99
                                          113 Halifax, NS                       6613.34   57764.33
                                            8 Anchorage, AK                     6075.13   63839.46
                                          243 San Juan, PR                       6499.8   70339.26
                                           90 Fairbanks, AK                     6485.59   76824.85
                                          169 Moncton, NB                        5881.4   82706.26
                                           70 Dawson, YT                        5290.95   87997.21
                                          229 Saint John, NB                    5223.21   93220.42
                                          298 Whitehorse, YK                    4896.82   98117.25
                                           98 Fredericton, NB                   4845.34  102962.58
                                          131 Juneau, AK                        4768.78  107731.36
                                           19 Bangor, ME                        4630.48  112361.84
                                          209 Prince Rupert, BC                 4310.99  116672.83
                                           15 Augusta, ME                       4247.95  120920.78
                                           88 Eureka, CA                         3765.4  124686.18
                                          205 Portland, ME                      3729.96  128416.14
                                          291 Victoria, BC                      3689.93  132106.07
                                          207 Portsmouth, NH                     3659.1  135765.18
                                          290 Vancouver, BC                     3646.55  139411.72
                                           40 Brockton, MA                      3637.58  143049.31
                                          235 Salem, OR                         3599.44  146648.74
                                           34 Boston, MA                        3595.59  150244.33
                                           87 Eugene, OR                        3596.62  153840.95
                                           46 Cambridge, MA                     3593.43  157434.38
                                           25 Bellingham, WA                    3577.51  161011.89
                                           91 Fall River, MA                    3580.15  164592.04
                                          206 Portland, OR                      3569.55  168161.59
                                          145 Lawrence, MA                      3564.53  171726.12
                                          241 San Francisco, CA                 3557.84  175283.96
                                          213 Quebec City, QC                      3607  178890.96
                                          186 Oakland, CA                       3596.53  182487.49
                                          155 Lowell, MA                        3536.42  186023.91
                                          273 Tacoma, WA                           3547  189570.91
                                          210 Providence, RI                    3545.83  193116.74
                                          252 Seattle, WA                        3540.8  196657.53
                                          159 Manchester, NH                    3529.69  200187.22
                                           26 Berkeley, CA                      3528.99  203716.21
                                          254 Sherbrooke, QC                    3533.82  207250.03
                                          245 Santa Cruz, CA                    3527.03  210777.06
                                           67 Concord, NH                       3515.24   214292.3
                                          242 San Jose, CA                      3502.92  217795.22
                                          307 Worcester, MA                     3477.76  221272.98
                                          225 Sacramento, CA                    3442.81  224715.79
                                          284 Trois-Rivieres, QC                 3415.3  228131.09
                                          268 Stockton, CA                      3408.48  231539.57
                                          171 Montpelier, VT                    3393.96  234933.53
                                          244 Santa Barbara, CA                  3326.1  238259.63
                                           38 Brattleboro, VT                   3308.76  241568.39
                                          308 Yakima, WA                           3323   244891.4
                                          263 Springfield, MA                   3325.25  248216.65
                                           99 Fresno, CA                        3280.93  251497.58
                                          117 Hartford, CT                      3271.83  254769.41
                                          219 Reno, NV                          3259.94  258029.35
                                          178 New Britain,CT                     3253.1  261282.44
                                           48 Carson City, NV                   3251.14  264533.58
                                          163 Meriden, CT                        3248.8  267782.39
                                           17 Bakersfield, CA                   3221.22  271003.61
                                           43 Burlington, VT                    3226.83  274230.43
                                          153 Los Angeles, CA                   3193.66   277424.1
                                          179 New Haven, CT                      3170.9  280594.99
                                          309 Yellowknife, NT                   3219.25  283814.25
                                          164 Miami, FL                         3467.82  287282.06
                                           82 Edmonton, AB                      2998.65  290280.72
                                          136 Key West, FL                      2971.34  293252.06
                                          261 Spokane, WA                       2934.85  296186.91
                                           50 Central Islip, NY                 3092.23  299279.15
                                          293 Walla Walla, WA                   3140.18  302419.32
                                           39 Bridgeport, CT                    3137.06  305556.38
                                          193 Pasadena, CA                      3142.69  308699.07
                                          172 Montreal, QC                      3157.19  311856.26
                                          240 San Diego, CA                     3118.23  314974.49
                                          202 Pittsfield, MA                     3107.7  318082.19
                                          239 San Bernardino, CA                3097.28  321179.47
                                          285 Troy, NY                          3070.66  324250.12
                                           33 Boise, ID                         2937.92  327188.04
                                          266 Stamford, CT                      2952.96     330141
                                          311 Yuma, AZ                           2896.3   333037.3
                                            3 Albany, NY                        2905.84  335943.14
                                          144 Las Vegas, NV                     2893.98  338837.12
                                          251 Schenectady, NY                   2883.23  341720.35
                                           45 Calgary, AB                       2828.13  344548.48
                                          296 West Palm Beach, FL               2889.69  347438.17
                                          146 Lethbridge, AB                    2761.59  350199.76
                                          299 White Plains, NY                  2761.87  352961.63
                                           44 Butte, MT                         2700.79  355662.42
                                          181 New York, NY                      2687.05  358349.48
                                          199 Phoenix, AZ                       2677.74  361027.22
                                          128 Jersey City, NJ                   2673.06  363700.28
                                          203 Pocatello, ID                      2655.1  366355.39
                                          177 Newark, NJ                        2648.53  369003.92
                                          118 Helena, MT                        2647.25  371651.17
                                           13 Atlantic City, NJ                 2646.38  374297.55
                                          161 Medicine Hat, AB                  2613.82  376911.37
                                           83 Elizabeth, NJ                     2604.16  379515.53
                                           93 Flagstaff, AZ                     2614.32  382129.85
                                          194 Paterson, NJ                      2619.56  384749.41
                                          187 Ogden, UT                         2611.96  387361.37
                                          283 Trenton, NJ                       2573.35  389934.72
                                          107 Great Falls, MT                   2575.55  392510.27
                                          198 Philadelphia, PA                  2550.72  395060.99
                                          237 Salt Lake City, UT                2538.22  397599.21
                                          289 Utica, NY                         2538.01  400137.22
                                          286 Tucson, AZ                        2578.23  402715.44
                                          191 Ottawa, ON                        2575.89  405291.33
                                          211 Provo, UT                          2487.2  407778.53
                                            6 Allentown, PA                     2499.13  410277.67
                                          102 Gallup, NM                        2324.19  412601.86
                                           29 Binghamtom, NY                    2312.96  414914.82
                                           27 Billings, MT                       2265.6  417180.42
                                          302 Wilmington, DE                    2314.81  419495.23
                                          248 Saskatoon, SK                     2317.41  421812.65
                                          247 Sarasota, FL                      2394.44  424207.09
                                           59 Churchill, MB                     2311.24  426518.32
                                          234 Saint Petersburg, FL              2279.52  428797.84
                                          173 Moose Jaw, SK                     2220.06   431017.9
                                          190 Orlando, FL                       2249.04  433266.94
                                          218 Regina, SK                        2208.47  435475.41
                                           72 Daytona Beach, FL                 2195.17  437670.58
                                          255 Sheridan, WY                      2090.58  439761.16
                                          184 Norfolk, VA                       2189.22  441950.37
                                          105 Grand Junction, CO                2234.59  444184.96
                                          301 Wilkes-Barre, PA                  2262.27  446447.23
                                           84 El Paso, TX                       2213.93  448661.17
                                          137 Kingston, ON                      2237.67  450898.84
                                            4 Albuquerque, NM                    2170.5  453069.33
                                          272 Syracuse, NY                      2178.26  455247.59
                                          246 Santa Fe, NM                      2120.23  457367.82
                                          217 Reading, PA                       2098.29  459466.11
                                           74 Denver, CO                        2008.13  461474.24
                                          208 Portsmouth, VA                    1992.18  463466.42
                                           57 Cheyenne, WY                      1993.01  465459.43
                                          141 Lancaster, PA                     1971.66  467431.09
                                           62 Colorado Springs, CO              1972.03  469403.12
                                          129 Johnstown, PA                     1972.05  471375.16
                                          212 Pueblo, CO                        1959.85  473335.01
                                           18 Baltimore, MD                     1935.73  475270.74
                                          216 Rapid City, SD                    1868.74  477139.48
                                          303 Wilmington, NC                    1875.13  479014.61
                                          167 Minot, ND                         1881.45  480896.06
                                          275 Tampa, FL                         1912.76  482808.82
                                           36 Brandon, MB                       1940.02  484748.84
                                          101 Gainesville, FL                   1855.41  486604.25
                                           31 Bismarck, ND                      1741.25  488345.51
                                          127 Jacksonville, FL                  1744.35  490089.86
                                          305 Winnipeg, MB                      1729.19  491819.04
                                           52 Charleston, SC                    1683.01  493502.06
                                          200 Pierre, SD                        1622.39  495124.45
                                          294 Washington, DC                    1654.69  496779.14
                                          156 Lubbock, TX                       1753.74  498532.87
                                           24 Belleville, ON                    1847.76  500380.63
                                          143 Laredo, TX                        1918.78  502299.41
                                          279 Timmins, ON                       1920.31  504219.72
                                           68 Corpus Christi, TX                1812.36  506032.08
                                          185 North Bay, ON                      1786.7  507818.78
                                          238 San Antonio, TX                   1763.41  509582.19
                                          223 Rochester, NY                     1726.57  511308.76
                                            7 Amarillo, TX                      1760.64   513069.4
                                          116 Harrisburg, PA                    1758.66  514828.06
                                            1 Abilene, TX                       1668.54   516496.6
                                          197 Peterborough, ON                  1685.89  518182.49
                                           16 Austin, TX                         1650.4  519832.89
                                          269 Sudbury, ON                       1627.33  521460.22
                                          121 Houston, TX                       1539.52  522999.73
                                           41 Buffalo, NY                       1455.86   524455.6
                                           77 Dodge City, KS                    1503.01   525958.6
                                          220 Richmond, VA                      1558.59  527517.19
                                           92 Fargo, ND                          1482.8  528999.99
                                          250 Savannah, GA                      1489.99  530489.98
                                          258 Sioux Falls, SD                   1337.75  531827.74
                                          215 Raleigh, NC                       1358.74  533186.48
                                           85 Enid, OK                          1330.04  534516.52
                                          182 Niagara Falls, ON                 1380.26  535896.78
                                          292 Waco, TX                          1483.03  537379.81
                                          282 Toronto, ON                       1483.47  538863.29
                                           97 Ft Worth, TX                       1449.6  540312.89
                                          227 Saint Catherines, ON               1447.6  541760.49
                                          103 Galveston, TX                     1444.61  543205.09
                                           42 Burlington, ONT                   1421.73  544626.82
                                           69 Dallas, TX                        1383.45  546010.27
                                          115 Hamilton, ON                      1368.85  547379.12
                                           23 Beaumont, TX                      1332.75  548711.87
                                          111 Guelph, ON                        1333.91  550045.78
                                          204 Port Arthur, TX                   1334.48  551380.26
                                          138 Kitchener, ON                     1324.27  552704.53
                                          188 Oklahoma City, OK                 1306.17   554010.7
                                           37 Brantford, ON                     1299.42  555310.11
                                          300 Wichita, KS                       1232.93  556543.05
                                           80 Durham, NC                         1279.4  557822.45
                                          236 Salina, KS                         1307.8  559130.25
                                          109 Greensboro, NC                    1245.96  560376.22
                                          257 Sioux City, IA                    1230.47  561606.68
                                           64 Columbia, SC                      1213.26  562819.94
                                          228 Saint Cloud, MN                   1208.61  564028.55
                                          274 Tallahassee, FL                   1248.21  565276.76
                                          278 Thunder Bay, ON                   1287.21  566563.97
                                          180 New Orleans, LA                   1273.63   567837.6
                                          249 Sault Ste Marie, ON               1205.19  569042.78
                                           20 Baton Rouge, LA                   1199.28  570242.06
                                           79 Duluth, MN                        1130.44  571372.51
                                          195 Pensacola, FL                     1179.96  572552.47
                                          270 Superior, WI                      1175.77  573728.24
                                           14 Augusta, GA                       1152.38  574880.62
                                          150 Lincoln, NE                       1134.43  576015.05
                                          221 Roanoke, VA                       1181.08  577196.13
                                          189 Omaha, NE                         1139.08  578335.21
                                          306 Winston-Salem, NC                 1141.42  579476.63
                                          166 Minneapolis, MN                   1088.91  580565.53
                                           54 Charlotte, NC                     1091.15  581656.69
                                          233 Saint Paul, MN                    1080.39  582737.08
                                          157 Macon, GA                         1061.49  583798.57
                                           81 Eau Claire, WI                     989.72  584788.28
                                           28 Biloxi, MS                        1012.28  585800.56
                                          152 London, ON                        1016.65  586817.21
                                          160 Marshall, TX                      1158.56  587975.77
                                           86 Erie, PA                          1188.43   589164.2
                                          287 Tulsa, OK                          1174.1   590338.3
                                          201 Pittsburgh, PA                    1144.29  591482.59
                                          256 Shreveport, LA                    1096.48  592579.08
                                          310 Youngstown, OH                    1081.83   593660.9
                                          277 Texarkana, TX                     1066.85  594727.75
                                          267 Steubenville, OH                  1043.65  595771.41
                                          281 Topeka, KS                        1043.45  596814.86
                                          297 Wheeling, WV                      1035.82  597850.68
                                           95 Ft Smith, AR                       998.79  598849.47
                                           61 Cleveland, OH                      974.07  599823.54
                                          176 Natchez, MS                        959.96  600783.49
                                           22 Bay City, MI                       980.29  601763.78
                                          112 Gulfport, MS                       982.13  602745.92
                                          108 Green Bay, WI                      980.65  603726.56
                                          168 Mobile, AL                         955.25  604681.82
                                          222 Rochester, MN                      970.35  605652.16
                                          260 Spartanburg, NC                    960.77  606612.93
                                          231 Saint Joseph, MO                   952.41  607565.34
                                           47 Canton, OH                         933.28  608498.62
                                          130 Joplin, MO                         943.13  609441.75
                                            2 Akron, OH                          939.34  610381.09
                                          133 Kansas City, KS                    915.85  611296.95
                                           53 Charleston, WV                     899.41  612196.36
                                          134 Kansas City, MO                    895.98  613092.34
                                          110 Greenville, SC                     891.55  613983.89
                                           75 Des Moines, IA                     904.35  614888.24
                                           10 Asheville, NC                      869.08  615757.32
                                          295 Waterloo, IA                       827.18   616584.5
                                           65 Columbus, GA                       859.51  617444.01
                                          253 Sheboygan, WI                      802.52  618246.52
                                          126 Jackson, MS                         809.5  619056.02
                                          226 Saginaw, MI                         880.9  619936.92
                                          151 Little Rock, AR                    831.32  620768.24
                                          304 Windsor, ON                        828.42  621596.66
                                          264 Springfield, MO                    795.38  622392.04
                                          312 Zanesville, OH                     802.13  623194.17
                                           63 Columbia, MO                       716.36  623910.53
                                           76 Detroit, MI                        682.93  624593.46
                                          170 Montgomery, AL                      724.3  625317.77
                                          158 Madison, WI                        770.17  626087.93
                                           12 Atlanta, GA                        731.47  626819.41
                                           49 Cedar Rapids, IA                   759.63  627579.04
                                          100 Gadsden, AL                        675.88  628254.92
                                           78 Dubuque, IA                        668.88   628923.8
                                           30 Birmingham, AL                     675.42  629599.22
                                           94 Flint, MI                          690.27  630289.49
                                          162 Memphis, TN                        698.85  630988.33
                                          142 Lansing, MI                           647  631635.33
                                          122 Huntsville, AL                     570.44  632205.78
                                          124 Iowa City, IA                      588.25  632794.02
                                          139 Knoxville, TN                      656.94  633450.96
                                          224 Rockford, IL                       563.81  634014.76
                                           56 Chattanooga, TN                    563.58  634578.34
                                          165 Milwaukee, WI                      580.71  635159.05
                                           11 Ashland, KY                        481.46  635640.51
                                          232 Saint Louis, MO                    522.45  636162.96
                                            9 Ann Arbor, MI                      513.15  636676.12
                                          192 Paducah, KY                        491.81  637167.92
                                          125 Jackson, MI                        459.78   637627.7
                                          175 Nashville, TN                      451.22  638078.92
                                          106 Grand Rapids, MI                   475.97  638554.89
                                           35 Bowling Green, KY                  416.17  638971.07
                                          214 Racine, WI                         406.98  639378.05
                                          147 Lexington, KY                      399.12  639777.17
                                          196 Peoria, IL                         399.56  640176.73
                                           66 Columbus, OH                       458.14  640634.87
                                          262 Springfield, IL                    459.25  641094.13
                                          280 Toledo, OH                         439.91  641534.03
                                           73 Decatur, IL                        393.78  641927.82
                                          265 Springfield, OH                    355.61  642283.42
                                           32 Bloomington, IL                    360.32  642643.75
                                          149 Lima, OH                           338.23  642981.98
                                           89 Evansville, IN                     305.64  643287.62
                                           21 Battle Creek, MI                   342.27  643629.89
                                          154 Louisville, KY                     283.85  643913.74
                                          135 Kenosha, WI                         331.4  644245.14
                                           60 Cincinnati, OH                     331.61  644576.75
                                           58 Chicago, IL                         288.4  644865.14
                                           71 Dayton, OH                         279.24  645144.39
                                          288 Urbana, IL                         278.52  645422.91
                                          114 Hamilton, OH                       256.66  645679.57
                                           51 Champaign, IL                      259.18  645938.75
                                          132 Kalamazoo, MI                      237.21  646175.96
                                          276 Terre Haute, IN                    232.44   646408.4
                                           96 Ft Wayne, IN                        195.3   646603.7
                                          104 Gary, IN                           156.52  646760.22
                                          174 Muncie, IN                         166.43  646926.65
                                          259 South Bend, IN                     118.99  647045.64
                                          123 Indianapolis, IN                   132.47  647178.11
                                          140 Lafayette, IN                       66.81  647244.92
                                            5 Alert, NT                         3369.01  650613.92

1252 rows selected.

Elapsed: 00:01:24.28
</pre>
</div>

### Summary of Results

Here we list the best solutions found for each root leg, i.e. those with PATH\_RNK = 1. We do not know what the optima are.

#### Shortest Routes
```
ROOT_LEG       TOT_DIST
------------ ----------
184 -> 208     55744.86
51 -> 288      56794.75
```

#### Longest Routes
```
ROOT_LEG       TOT_DIST
------------ ----------
183 -> 230    649684.75
5 -> 148      650614.77
```

### Execution Plan for Shortest Routes, USCA312

```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                             | Name        | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                                      |             |      1 |        |   1252 |00:01:24.88 |      10M|    113K|  10654 |       |       |          |
|   1 |  WINDOW SORT                                          |             |      1 |     10 |   1252 |00:01:24.88 |      10M|    113K|  10654 |   267K|   267K|  237K (0)|
|   2 |   NESTED LOOPS OUTER                                  |             |      1 |     10 |   1252 |00:01:24.88 |      10M|    113K|  10654 |       |       |          |
|   3 |    MERGE JOIN                                         |             |      1 |     10 |   1252 |00:01:24.87 |      10M|    113K|  10654 |       |       |          |
|   4 |     TABLE ACCESS BY INDEX ROWID                       | TOWNS       |      1 |      5 |    312 |00:00:00.01 |     205 |      0 |      0 |       |       |          |
|   5 |      INDEX FULL SCAN                                  | SYS_C008004 |      1 |      5 |    312 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|*  6 |     SORT JOIN                                         |             |    312 |     10 |   1252 |00:01:24.87 |      10M|    113K|  10654 |   133K|   133K|  118K (0)|
|   7 |      VIEW                                             |             |      1 |     10 |   1252 |00:01:24.86 |      10M|    113K|  10654 |       |       |          |
|   8 |       WINDOW SORT                                     |             |      1 |     10 |   1252 |00:01:24.86 |      10M|    113K|  10654 |  1895K|   658K| 1684K (0)|
|   9 |        MERGE JOIN CARTESIAN                           |             |      1 |     10 |   1252 |00:01:24.85 |      10M|    113K|  10654 |       |       |          |
|  10 |         VIEW                                          |             |      1 |      1 |    313 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  11 |          CONNECT BY WITHOUT FILTERING                 |             |      1 |        |    313 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  12 |           VIEW                                        |             |      1 |      1 |      1 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  13 |            SORT AGGREGATE                             |             |      1 |      1 |      1 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  14 |             INDEX FULL SCAN                           | SYS_C008004 |      1 |      5 |    312 |00:00:00.01 |       1 |      0 |      0 |       |       |          |
|  15 |         BUFFER SORT                                   |             |    313 |     10 |   1252 |00:01:24.85 |      10M|    113K|  10654 |  9216 |  9216 | 8192  (0)|
|* 16 |          VIEW                                         |             |      1 |     10 |      4 |00:01:24.85 |      10M|    113K|  10654 |       |       |          |
|* 17 |           WINDOW SORT PUSHED RANK                     |             |      1 |     10 |      4 |00:01:24.85 |      10M|    113K|  10654 |  9216 |  9216 | 8192  (0)|
|  18 |            NESTED LOOPS                               |             |      1 |        |      4 |00:00:34.80 |      10M|    113K|  10654 |       |       |          |
|  19 |             NESTED LOOPS                              |             |      1 |     10 |      4 |00:00:34.80 |      10M|    113K|  10654 |       |       |          |
|* 20 |              VIEW                                     |             |      1 |     12 |      4 |00:00:34.80 |      10M|    113K|  10654 |       |       |          |
|  21 |               UNION ALL (RECURSIVE WITH) BREADTH FIRST|             |      1 |        |    192K|00:00:04.53 |      10M|    113K|  10654 |       |       |          |
|* 22 |                VIEW                                   |             |      1 |     10 |      2 |00:00:00.10 |     693 |      1 |      0 |       |       |          |
|* 23 |                 WINDOW SORT PUSHED RANK               |             |      1 |     10 |      3 |00:00:00.10 |     693 |      1 |      0 |  2048 |  2048 | 2048  (0)|
|  24 |                  WINDOW BUFFER                        |             |      1 |     10 |  48516 |00:00:00.08 |     693 |      1 |      0 |  3100K|   779K| 2755K (0)|
|  25 |                   TABLE ACCESS BY INDEX ROWID         | DISTANCES   |      1 |     10 |  48516 |00:00:00.04 |     693 |      1 |      0 |       |       |          |
|* 26 |                    INDEX FULL SCAN                    | DISTANCE_PK |      1 |     10 |  48516 |00:00:00.02 |     212 |      0 |      0 |       |       |          |
|  27 |                WINDOW SORT                            |             |    311 |      2 |    192K|00:00:04.40 |    5664 |      1 |      0 |   619K|   472K|  550K (0)|
|  28 |                 NESTED LOOPS                          |             |    311 |        |    192K|00:00:05.39 |    5664 |      1 |      0 |       |       |          |
|  29 |                  NESTED LOOPS                         |             |    311 |      2 |    192K|00:00:05.08 |    2392 |      0 |      0 |       |       |          |
|  30 |                   RECURSIVE WITH PUMP                 |             |    311 |        |   1242 |00:00:00.01 |       0 |      0 |      0 |       |       |          |
|* 31 |                   INDEX RANGE SCAN                    | DISTANCE_PK |   1242 |      1 |    192K|00:00:06.49 |    2392 |      0 |      0 |       |       |          |
|  32 |                  TABLE ACCESS BY INDEX ROWID          | DISTANCES   |    192K|      1 |    192K|00:00:00.15 |    3272 |      1 |      0 |       |       |          |
|* 33 |              INDEX UNIQUE SCAN                        | DISTANCE_PK |      4 |      1 |      4 |00:00:00.01 |       6 |      0 |      0 |       |       |          |
|  34 |             TABLE ACCESS BY INDEX ROWID               | DISTANCES   |      4 |      1 |      4 |00:00:00.01 |       4 |      0 |      0 |       |       |          |
|  35 |    TABLE ACCESS BY INDEX ROWID                        | DISTANCES   |   1252 |      1 |   1248 |00:00:00.01 |    2498 |      0 |      0 |       |       |          |
|* 36 |     INDEX UNIQUE SCAN                                 | DISTANCE_PK |   1252 |      1 |   1248 |00:00:00.01 |    1250 |      0 |      0 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   6 - access("TWN"."ID"="TOP"."TOWN_ID")
       filter("TWN"."ID"="TOP"."TOWN_ID")
  16 - filter("CIRCUITS"."PATH_RNK"<=:KEEP_NUM)
  17 - filter(ROW_NUMBER() OVER ( PARTITION BY "R"."ROOT_LEG" ORDER BY :SIGN*("R"."TOT_PRICE"+"D"."DST"))<=:KEEP_NUM)
  20 - filter(("R"."LEV"="R"."N_TOWNS" AND "R"."PATH_RNK"<=:KEEP_NUM))
  22 - filter("D"."RNK_BY_DST"<=:KEEP_NUM_ROOT)
  23 - filter(ROW_NUMBER() OVER ( ORDER BY :SIGN*"DST")<=:KEEP_NUM_ROOT) 26 - filter("B">"A")
  31 - access("D"."A"="R"."NXT_ID")
       filter("R"."PATH" NOT LIKE '%|'||LPAD(TO_CHAR("D"."B"),3,'0')||'%')
  33 - access("D"."A"="R"."NXT_ID" AND "D"."B"="R"."ROOT")
  36 - access("DST"."A"="TOP"."TOWN_ID_PRIOR" AND "DST"."B"="TOP"."TOWN_ID")

```

## Notes

- In the closed version of TSP that we have considered here, the starting point does not essentially affect the set of possible solutions; however, it may affect the solutions actually obtained by an approximate algorithm; partitioning by a set of root legs is designed to improve the quality of solutions obtained
- I found that the query uses increasing amounts of temp tablespace as the keep values are increased for the larger problem; in my previous article, on a knapsack problem, I included a PL/SQL solution that had the flexibility to discard all but the best solutions as it went, thus keeping space usage to a minimum; it also had the flexibility of the conventional 'branch and bound' algorithms to retain the best solution values _globally_ and close off paths that could not beat them. This is difficult to do in SQL, but against that, the SQL is much simpler

## Conclusions

Following my earlier two articles on related subjects, we have again seen that the recursive capabilities of Oracle's SQL from v11.2 can provide surprisingly simple approximate solutions to 'hard' combinatorial problems in reasonable execution times.

**20 February 2016**: Added attachment with the input data files, DDL and raw results:

[TSP Files: input data files, DDL and raw results, in GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/sql-for-the-travelling-salesman-problem)

This new (November 2017) GitHub project also includes scripts and will be easier to install: [Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos)

