---
layout: post
title: "SQL for Shortest Path Problems 2: A Branch and Bound Approach"
date: 2015-05-04
migrated: true
categories: 
  - "analytics"
  - "oracle"
  - "performance"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "branch-and-bound"
  - "combinatorial"
  - "oracle"
  - "performance-2"
  - "recursive"
  - "sql"
  - "subquery-factor"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

I wrote an article a couple of weeks ago, [SQL for Shortest Path Problems](https://brenpatf.github.io/migrated/sql-for-shortest-path-problems/), in which analytic functions are used to truncate sub-optimal routes in SQL recursions for shortest paths through networks. The problem was posed by an OTN poster, [How to use Recursive Subquery Factoring (RSF) to Implement Dijkstra’s shortest path algorithm?](https://community.oracle.com/thread/3698740?sr=stream), who referenced a very simple test network, and included his own SQL to solve it, which turned out to be quite similar to my own effort. The solutions are guaranteed to be optimal if the algorithm terminates normally, which it does on the small test network, and will on any network unless resources such as memory or time are exhausted owing to problem size.

In that article I referenced two earlier articles that I had written (in June/July 2013) that used analytic functions for other combinatorial problems. The usage in those cases was similar syntactically, but pruned out routes that _looked_ inferior to others at a given iteration, so that the final solutions were not guaranteed to be optimal. The motivation was to to be able to use the SQL for exact solutions on smaller problems and for good, maybe sub-optimal, solutions for problems too large to solve exactly.

I wondered how the SQL in the last article would perform on larger networks, and whether further tuning methods could be found, perhaps based on some form of search truncation, as in the earlier articles. The resulting solution methods can be considered as branch and bound algorithms in SQL.

**Updates**

- **19 November 2017**: I have now put the code and the dataset installation onto GitHub: [Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos)
- **July 2025**: See also this 2022 article of mine, [Shortest Path Analysis of Large Networks by SQL and PL/SQL](https://brenpatf.github.io/2022/08/07/shortest-path-analysis-of-large-networks-by-sql-and-plsql.html)

## Test Problems

I took two test problems from this site: [Stanford Large Network Dataset Collection](https://snap.stanford.edu/data).

- [Collaboration network of Arxiv General Relativity category](https://snap.stanford.edu/data/ca-GrQc.html)
- [Friendship network of Brightkite users](https://snap.stanford.edu/data/loc-brightkite.html)

The larger second data set has 428,156 arcs.

Both data sets are of the non-physical type, where there are no differential costs associated with links, and the problem therefore reduces to determining the minimum number of links between a node and other reachable nodes. These non-physical networks tend to be heavily looped, owing to the essentially zero cost of adding a new link. For that reason, I change my problem definition in this article to that of finding a single best path to each reachable node, rather than all, which reduces the solution set size considerably.

It is well known that in non-physical networks, such as social media networks like Linked-In and Facebook, the minimum paths between members tends to remain relatively small as the network size goes up. This will influence the type of algorithm that will be more efficient.

## Approximation Methods: Simple truncation

The most obvious approximative approach would be to simple truncate the search after a certain depth (or level). This actually works quite well and gives good results for highly looped networks, where the minimum paths tend to be much shorter than the number of nodes. However there is no guarantee of optimality, and it will be less effective for less looped networks with longer minimum paths.

### SQL for SP\_RSFONE (simple truncation)

```sql
WITH paths (node, path, lev, rn) AS (
SELECT a.dst, To_Char(a.dst), 1, 1
  FROM arcs_v a
WHERE a.src = &SRC
 UNION ALL
SELECT  a.dst,
        p.path || ',' || a.dst,
        p.lev + 1,
        Row_Number () OVER (PARTITION BY a.dst ORDER BY a.dst)
  FROM paths p
  JOIN arcs_v a
    ON a.src = p.node
 WHERE p.rn = 1
   AND p.lev < &LEVMAX
)  SEARCH DEPTH FIRST BY node SET line_no
CYCLE node SET lp TO '*' DEFAULT ' '
SELECT Substr (LPad ('.', 1 + 2 * (Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) - 1), '.') || node, 2) node,
       Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) lev,
       Max (Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev)) OVER () maxlev,
       Max (lev) intnod,
       Max (Max (lev)) OVER () intmax,
       Max (path) KEEP (DENSE_RANK FIRST ORDER BY lev) path,
       Max (lp) KEEP (DENSE_RANK FIRST ORDER BY lev) lp
  FROM paths
 GROUP BY node
 ORDER BY Max (line_no) KEEP (DENSE_RANK FIRST ORDER BY lev)
```

### Notes on SQL for SP\_RSFONE (simple truncation)

- Row\_Number gives a single row with value 1 per partitioning node, so that we retain only one row per node for the previous iteration
- The global pruning can be done without an additional subquery by grouping with KEEP, since we only want one optimal row per node
- Note that in the double Max to get maxlev, the inner one is the grouping Max, while the outer is an analytic Max over the whole (grouped) result set
- intnod obtains the maximum intermdiate value of lev for a given node
- intmax obtains the maximum intermdiate value of lev over all nodes

## Approximation Methods: Preliminary approximate subquery

A less obvious approach is based on the fact that during the recursion our path pruning can only take into account information available to the current iteration: Other than loops, we can prune out only paths to a given node that are longer than another _at the same level_. But what if we ran an approximate search in advance, in a prior subquery? Then we could outer-join the subquery by node and prune out any paths for which the subquery has found a better cost. This would potentially reduce the total searching without sacrificing guranteed optimality.

### SQL for SP\_RSFTWO (preliminary approximate subquery)

```sql
WITH paths_0 (node, path, lev, rn) AS (
SELECT a.dst, To_Char(a.dst), 1, 1
  FROM arcs_v a
 WHERE a.src = &SRC
 UNION ALL
SELECT a.dst,
       p.path || ',' || a.dst,
       p.lev + 1,
       Row_Number () OVER (PARTITION BY a.dst ORDER BY a.dst)
  FROM paths_0 p
  JOIN arcs_v a
    ON a.src = p.node
 WHERE p.rn = 1
   AND p.lev < &LEVMAX
) CYCLE node SET lp TO '*' DEFAULT ' '
, approx_best_paths AS (
SELECT node,
       Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) lev
  FROM paths_0
 GROUP BY node
), paths (node, path, lev, rn) AS (
SELECT a.dst, To_Char(a.dst), 1, 1
  FROM arcs_v a
WHERE a.src = &SRC
 UNION ALL
SELECT a.dst,
        p.path || ',' || a.dst,
        p.lev + 1,
        Row_Number () OVER (PARTITION BY a.dst ORDER BY a.dst)
  FROM paths p
  JOIN arcs_v a
    ON a.src = p.node
  LEFT JOIN approx_best_paths b
    ON b.node = a.dst
 WHERE p.rn = 1
   AND p.lev < Nvl (b.lev, 1000000)
)  SEARCH DEPTH FIRST BY node SET line_no CYCLE node SET lp TO '*' DEFAULT ' '
SELECT Substr (LPad ('.', 1 + 2 * (Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) - 1), '.') || node, 2) node,
       Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) lev,
       Max (Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev)) OVER () maxlev,
       Max (lev) intnod,
       Max (Max (lev)) OVER () intmax,
       Max (path) KEEP (DENSE_RANK FIRST ORDER BY lev) path,
       Max (lp) KEEP (DENSE_RANK FIRST ORDER BY lev) lp
  FROM paths
 GROUP BY node
 ORDER BY Max (line_no) KEEP (DENSE_RANK FIRST ORDER BY lev)
```

### Notes on SQL for SP\_RSFTWO (preliminary approximate subquery)

- This query has two recursive subquery factors
- path\_0 truncates after an input level is reached
- approx\_best\_paths gets the global best paths found from the approximate recursion
- paths now has an outer join to path\_0 that is used to prune paths that are inferior to any path to the same node found in the prior subquery
- the approximate recursion may not reach all reachable nodes, but the outer join ensures this does not cut off any such nodes incorrectly

## Approximation Methods: Preliminary approximate subquery to GTT

We will see later that the second approach works quite well, but that the CBO does not process the preliminary query very efficiently. For this reason writing the query result instead to a temporary table may be more efficient overall. The table can be indexed, and dynamic sampling allows the CBO to estimate the cardinalities more accurately.

### SQL for SP\_GTTRSF\_I and SP\_GTTRSF\_Q (preliminary approximate query to GTT)

```sql
PROMPT SP_GTTRSF_I
INSERT INTO approx_min_levs
WITH paths_0 (node, path, lev, rn) /* SP_GTTRSF_I */ AS (
SELECT a.dst, To_Char(a.dst), 1, 1
  FROM arcs_v a
WHERE a.src = &SRC
 UNION ALL
SELECT  a.dst,
        p.path || ',' || a.dst,
        p.lev + 1,
        Row_Number () OVER (PARTITION BY a.dst ORDER BY a.dst)
  FROM paths_0 p
  JOIN arcs_v a
    ON a.src = p.node
 WHERE p.rn = 1
   AND p.lev < &LEVMAX
) CYCLE node SET lp TO '*' DEFAULT ' '
SELECT node,
       Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) lev
  FROM paths_0
 GROUP BY node
/
PROMPT SP_GTTRSF_Q
WITH paths (node, path, lev, rn) AS (
SELECT a.dst, To_Char(a.dst), 1, 1
  FROM arcs_v a
WHERE a.src = &SRC
 UNION ALL
SELECT a.dst,
        p.path || ',' || a.dst,
        p.lev + 1,
        Row_Number () OVER (PARTITION BY a.dst ORDER BY a.dst)
  FROM paths p
  JOIN arcs_v a
    ON a.src = p.node
  LEFT JOIN approx_min_levs b
    ON b.node = a.dst
 WHERE p.rn = 1
   AND p.lev < Nvl (b.lev, 1000000)
)  SEARCH DEPTH FIRST BY node SET line_no CYCLE node SET lp TO '*' DEFAULT ' '
SELECT Substr (LPad ('.', 1 + 2 * (Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) - 1), '.') || node, 2) node,
       Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev) lev,
       Max (Max (lev) KEEP (DENSE_RANK FIRST ORDER BY lev)) OVER () maxlev,
       Max (lev) intnod,
       Max (Max (lev)) OVER () intmax,
       Max (path) KEEP (DENSE_RANK FIRST ORDER BY lev) path,
       Max (lp) KEEP (DENSE_RANK FIRST ORDER BY lev) lp
  FROM paths
 GROUP BY node
 ORDER BY Max (line_no) KEEP (DENSE_RANK FIRST ORDER BY lev)
```

### Notes on SQL for preliminary approximate subquery to GTT

- the SQL above consists of an insert of the approximate subquery into a temporary table, followed by the exact recursive query now referencing the table rather than a subquery factor

## Results: Collaboration network of Arxiv General Relativity category
- [Collaboration network of Arxiv General Relativity category](https://snap.stanford.edu/data/ca-GrQc.html)

> Arxiv GR-QC (General Relativity and Quantum Cosmology) collaboration network is from the e-print arXiv and covers scientific collaborations between authors papers submitted to General Relativity and Quantum Cosmology category. If an author i co-authored a paper with author j, the graph contains a undirected edge from i to j. If the paper is co-authored by k authors this generates a completely connected (sub)graph on k nodes.

The data set comes with the reverse arcs already present, making a total of 28,980.

I took the first node in the first line in the data set file as the initial root node, 3466, and tested the three methods above for values of LEVMAX of 5, 10, 15, 20, 25 and 30. The complete results, including execution plans are in the attached file.

The exact solution has 4,158 nodes reached from the source node 3466, with a maximum level of 11. The listing below gives the output from SP\_GTTRSF\_Q for LEVMAX=5.

```
NODE                            LEV  MAXLEV  INTNOD  INTMAX PATH                                                                             LP
------------------------------ ---- ------- ------- ------- -------------------------------------------------------------------------------- --
937                               1      11       1      45 937
5233                              1      11       1      45 5233
8579                              1      11       1      45 8579
..4135                            2      11       2      45 937,4135
....1860                          3      11       3      45 8579,4135,1860
....16498                         3      11       3      45 8579,4135,16498
......19442                       4      11       4      45 8579,4135,16498,19442
..........9890                    6      11       6      45 8579,4135,16498,19442,6264,9890
......22826                       4      11       4      45 8579,4135,16498,22826
..........12260                   6      11       8      45 8579,4135,16498,22826,6804,12260
..........25491                   6      11       8      45 8579,4135,16498,22826,6804,25491
..........25844                   6      11       6      45 8579,4135,16498,22826,6804,25844
..............8037                8      11       9      45 8579,4135,16498,22826,13520,122,4825,8037
..............14621               8      11      11      45 8579,4135,16498,22826,13520,122,4825,14621
..............19608               8      11      10      45 8579,4135,16498,22826,13520,122,4825,19608
..............4836                8      11       9      45 8579,4135,16498,22826,13520,122,9593,4836
............4641                  7      11       9      45 8579,4135,16498,22826,13520,22224,4641
............17937                 7      11       9      45 8579,4135,16498,22826,13520,22224,17937
............21660                 7      11       7      45 8579,4135,16498,22826,13520,22224,21660
............22088                 7      11       7      45 8579,4135,16498,22826,13520,22224,22088
..........22224                   6      11       9      45 8579,4135,16498,22826,13520,22224
........17599                     5      11       5      45 8579,4135,16498,22826,17599
..16258                           2      11       2      45 8579,16258
....1356                          3      11       3      45 8579,16258,1356
....1727                          3      11       3      45 8579,16258,1727
......4241                        4      11       4      45 8579,16258,1727,4241
........5227                      5      11       5      45 8579,16258,1727,4241,5227
........7015                      5      11       5      45 8579,16258,1727,4241,7015
........10476                     5      11       5      45 8579,16258,1727,4241,10476
..............7854                8      11      10      45 8579,16258,1727,4241,10476,4875,11844,7854
............11844                 7      11      10      45 8579,16258,1727,4241,10476,4875,11844
.
. extracted
.
..23429                           2      11       2      45 17038,23429
....4781                          3      11       3      45 17038,23429,4781
....19697                         3      11       3      45 17038,23429,19697
......5519                        4      11       4      45 17038,23429,19697,5519
........26167                     5      11       5      45 17038,23429,19697,5519,26167
......17818                       4      11       4      45 17038,23429,19697,17818
......20260                       4      11       4      45 17038,23429,19697,20260
......23809                       4      11       4      45 17038,23429,19697,23809
........23219                     5      11       5      45 17038,23429,19697,5519,23219
..24578                           2      11       2      45 17038,24578
....18297                         3      11       3      45 17038,24578,18297
......5402                        4      11       4      45 17038,24578,18297,5402
......15947                       4      11       4      45 17038,24578,18297,15947
18720                             1      11       1      45 18720
19607                             1      11       1      45 19607
..3466                            2      11       2      45 937,3466

4158 rows selected.

Elapsed: 00:00:09.78
```

The embedded Excel file below summarises the results with relevant statistics from the query runs and the execution plans. 

<iframe src="https://onedrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95ec670ea6af8ed1%2179876&amp;authkey=ANmvOWidcOFeg7I&amp;em=2" width="800" height="500" frameborder="0" scrolling="no"></iframe>

### Results summary - SP\_RSFONE (simple truncation)

- SP\_RSFONE ran in from 3 seconds to 60 seconds, and returned the exact solution from LEVMAX = 15 on
- It continued iterating up to the maximum level of LEVMAX in each case, although all optimal path lengths are found by level 11

### Results summary - SP\_RSFTWO (preliminary approximate subquery)

This is the execution plan for LEVMAX=5

```
-----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                          | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                                   |                 |      1 |        |   4158 |00:00:27.47 |    2716K|       |       |          |
|   1 |  WINDOW SORT                                       |                 |      1 |     39 |   4158 |00:00:27.47 |    2716K|   372K|   372K|  330K (0)|
|   2 |   SORT GROUP BY                                    |                 |      1 |     39 |   4158 |00:00:27.45 |    2716K|    19M|  6662K|   17M (0)|
|   3 |    VIEW                                            |                 |      1 |     39 |  87755 |00:00:27.27 |    2716K|       |       |          |
|   4 |     UNION ALL (RECURSIVE WITH) DEPTH FIRST         |                 |      1 |        |  87755 |00:00:27.17 |    2716K|    16M|  1524K|   14M (0)|
|*  5 |      INDEX RANGE SCAN                              | ARCS_CA_GRQC_PK |      1 |      6 |      8 |00:00:00.01 |       2 |       |       |          |
|   6 |      WINDOW SORT                                   |                 |     45 |     33 |  87747 |00:00:21.78 |    1664K|   832K|   511K|  739K (0)|
|*  7 |       FILTER                                       |                 |     45 |        |  87747 |00:00:21.27 |    1664K|       |       |          |
|*  8 |        HASH JOIN RIGHT OUTER                       |                 |     45 |     33 |    110K|00:00:21.38 |    1664K|  1817K|  1817K| 1565K (0)|
|   9 |         VIEW                                       |                 |     45 |     39 |    114K|00:00:21.02 |    1655K|       |       |          |
|  10 |          SORT GROUP BY                             |                 |     45 |     39 |    114K|00:00:20.96 |    1655K|   337K|   337K|  299K (0)|
|  11 |           VIEW                                     |                 |     45 |     39 |    689K|00:00:20.23 |    1655K|       |       |          |
|  12 |            UNION ALL (RECURSIVE WITH) BREADTH FIRST|                 |     45 |        |    689K|00:00:19.32 |    1655K|   903K|   523K|  802K (0)|
|* 13 |             INDEX RANGE SCAN                       | ARCS_CA_GRQC_PK |     45 |      6 |    360 |00:00:00.01 |      50 |       |       |          |
|  14 |             WINDOW SORT                            |                 |    225 |     33 |    688K|00:00:04.00 |   44865 |  1116K|   557K|  991K (0)|
|  15 |              NESTED LOOPS                          |                 |    225 |     33 |    688K|00:00:01.27 |   44865 |       |       |          |
|  16 |               RECURSIVE WITH PUMP                  |                 |    225 |        |  67635 |00:00:00.27 |       0 |       |       |          |
|* 17 |               INDEX RANGE SCAN                     | ARCS_CA_GRQC_PK |  67635 |      6 |    688K|00:00:00.57 |   44865 |       |       |          |
|  18 |         NESTED LOOPS                               |                 |     45 |     33 |    110K|00:00:00.21 |    8594 |       |       |          |
|  19 |          RECURSIVE WITH PUMP                       |                 |     45 |        |  10787 |00:00:00.03 |       0 |       |       |          |
|* 20 |          INDEX RANGE SCAN                          | ARCS_CA_GRQC_PK |  10787 |      6 |    110K|00:00:00.11 |    8594 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - access("SRC"=3466)
   7 - filter("P"."LEV"<NVL("B"."LEV",1000000))
   8 - access("B"."NODE"="DST")
  13 - access("SRC"=3466)
  17 - access("SRC"="P"."NODE")
  20 - access("SRC"="P"."NODE")
```

- SP\_RSFTWO ran in from 27 seconds to 552 seconds, and returned the exact solution in all cases
- From the intmax value (Excel file) we can see that only in the case of LEVMAX=5 did the second recursion iterate beyond the optimum level of 11 (in fact to a level of 45). This would be due the preliminary approximate subquery allowing it to discard sub-optimal paths in the second recursion
- The execution plan above shows that Oracle CBO has chosen not to materialize the first recursive subquery, and essentially reruns it at each of the 45 outer iterations
- A better approach for the CBO to have taken would appear to be to form a hash table in memory (where possible) of the first subquery, and run each iteration of the outer recursion outer-joining to that unchanging hash table; or alternatively, to materialize it and outer-join in any other way deemed appropriate
- I tried hinting the subquery to get CBO to materialise or avoid the repetition of the first subquery, but without success, and so decided using a temproray table to materialise it myself would be a good idea

### Results summary - SP\_GTTRSF\_I and SP\_GTTRSF\_Q (preliminary approximate query to GTT)

- SP\_GTTRSF\_I and SP\_GTTRSF\_Q ran in from 5 seconds to 54 seconds combined
- LEVMAX=10 gave slight the better result here: Evidently, the extra work in the insert compared with LEVMAX=5 was over-compensated by the benefit of a better approximate solution
- Once LEVMAX is sufficiently large to give a good approximate solution it is most efficient not to increase it further
- This method is almost as efficient as simply truncating at a given level, but guarantees optimality

### Network analysis

I have my own network analysis program, implemented as a PL/SQL pipelined function. I thought this might be useful to help validate the results. The function returns all distinct subnetworks and is called three times to give different levels of detail. It runs against a view links\_v that must be created for the data source containing the network links. Here is the output:

#### Network Detail
<div class="scrollbox">
<pre>
links_v based on net_CA_GrQc

View dropped.

View created.

  COUNT(*)
----------
     14484

Network detail

Network     #Links  #Nodes    Lev Node       Link
---------- ------- ------- ------ ---------- ----------
1000         13422    4158      0   1000     ROOT
                                1 > 1149     770
                                2 > 12016    4681
                                3 < 1000*    778
                                3 > 18271    7628
                                4 < 13702    3675
.
. Extracted
.
                               11 < 5400     14039
                                8 < 5241     5959
                                7 > 18145    8551
                                8 > 21799    13889
                                9 < 11557*   8553
                                6 > 18271*   14193
                                6 > 19978    14194
                                7 < 13702*   3676
                                7 < 18271*   3671
                                4 < 18152    12071
                                2 > 16234    4682
                                3 > 18373    4686
                                4 < 1149*    4684
                                4 > 19871    4687
                                5 > 25526    5590
                                6 < 18373*   4688
                                2 > 17251    4683
                                2 > 20557    4685
                                2 > 4189     4679
                                3 > 5829     13222
                                1 > 20667    785
10002            1       2      0   10002    ROOT
                                1 < 7468     8487
10056           11       6      0   10056    ROOT
                                1 < 2754     13119
                                2 > 5511     13116
                                3 > 10056*   13122
                                3 > 7265     13120
                                4 > 10056*   13115
                                4 < 2754*    13117
                                4 > 7479     13114
                                5 > 10056*   12036
                                5 > 13390    12037
                                5 < 2754*    13118
                                5 < 5511*    13121
1010             1       2      0   1010     ROOT
                                1 > 2749     13221
10115            2       3      0   10115    ROOT
                                1 > 10134    50
                                1 > 23916    51
1013             2       3      0   1013     ROOT
                                1 > 19011    9224
                                2 < 2123     12933
10133            1       2      0   10133    ROOT
                                1 > 18218    6369
10148            1       2      0   10148    ROOT
                                1 > 10847    7955
10154            5       4      0   10154    ROOT
                                1 > 16830    13630
                                2 < 11410    14133
                                3 < 9224     12381
                                4 > 10154*   12380
                                4 > 16830*   12382
10163           24       8      0   10163    ROOT
                                1 > 12551    7434
                                2 < 1281     7440
                                3 > 10163*   7439
                                3 > 18621    7441
                                4 < 10163*   7435
                                4 < 12551*   5987
                                4 < 17588    5984
                                5 < 12551*   5986
                                5 > 24207    5985
                                6 < 10163*   7436
                                6 < 12551*   5988
                                6 < 1281*    7442
                                6 < 18621*   5982
                                6 > 25287    5981
                                7 < 10163*   7437
                                7 < 12551*   5989
                                7 < 1281*    7443
                                7 < 18621*   5983
                                7 < 3653     7448
                                8 > 10163*   7444
                                8 > 12551*   7445
                                8 < 1281*    7438
                                8 > 18621*   7446
                                8 > 24207*   7447
10178            3       3      0   10178    ROOT
                                1 > 24133    10756
                                2 < 3744     14126
                                3 > 10178*   14125
10180            3       3      0   10180    ROOT
                                1 > 17587    13060
                                2 < 2334     13059
                                3 > 10180*   13058
10252            3       3      0   10252    ROOT
                                1 > 16820    6964
                                2 < 9630     6963
                                3 > 10252*   6962
10317            2       3      0   10317    ROOT
                                1 > 11816    4265
                                1 < 2259     4264
10338           14       7      0   10338    ROOT
                                1 > 12458    5220
                                2 > 12460    8958
                                3 < 10338*   5221
                                3 > 16651    13302
                                4 < 10338*   5223
                                4 < 12458*   8959
                                4 > 19378    13976
                                5 < 10338*   5224
                                5 < 12458*   8960
                                5 < 12460*   13303
                                5 < 7523     12926
                                6 > 10338*   12924
                                6 > 12458*   12925
                                1 > 14844    5222
10341            6       4      0   10341    ROOT
                                1 < 1292     10693
                                2 > 14840    10694
                                3 < 10341*   10695
                                3 < 6648     10697
                                4 > 10341*   10696
                                4 < 1292*    10692
10355            5       6      0   10355    ROOT
                                1 < 2054     3240
                                2 > 13741    3241
                                2 < 1552     13484
                                2 > 8719     3238
                                2 > 9760     3239
10356            1       2      0   10356    ROOT
                                1 > 19062    7274
10382            1       2      0   10382    ROOT
                                1 > 25482    12047
10407            1       2      0   10407    ROOT
                                1 < 6701     13278
10432            2       3      0   10432    ROOT
                                1 > 16891    7463
                                1 < 5774     13628
10433            1       2      0   10433    ROOT
                                1 < 2593     5143
1050             1       2      0   1050     ROOT
                                1 > 17120    4284
1051             3       3      0   1051     ROOT
                                1 > 25664    13897
                                2 < 4245     13898
                                3 < 1051*    13896
10513            2       3      0   10513    ROOT
                                1 > 25059    9437
                                1 > 25595    9438
10517            1       2      0   10517    ROOT
                                1 < 5119     9154
10522            1       2      0   10522    ROOT
                                1 > 10561    11775
10559            1       2      0   10559    ROOT
                                1 > 18245    8169
10564            6       5      0   10564    ROOT
                                1 > 15187    6266
                                2 < 8970     9824
                                3 > 10564*   9823
                                1 < 2984     6265
                                2 > 8731     6264
                                3 > 10564*   6294
10602            1       2      0   10602    ROOT
                                1 > 20805    13140
10637            4       4      0   10637    ROOT
                                1 > 21340    4094
                                2 < 8428     4093
                                3 > 10637*   4091
                                3 > 18603    4092
10638            1       2      0   10638    ROOT
                                1 < 8717     12974
10672            6       4      0   10672    ROOT
                                1 > 12512    11111
                                2 < 4630     11110
                                3 > 10672*   11109
                                3 > 9921     11108
                                4 > 10672*   13681
                                4 > 12512*   13682
10676            1       2      0   10676    ROOT
                                1 > 24961    1798
10677           29      14      0   10677    ROOT
                                1 > 17015    9130
                                2 > 20560    7465
                                3 < 309      7253
                                4 > 10677*   7250
                                4 > 17015*   7251
                                4 > 17453    7252
                                5 < 11030    10700
                                6 > 23946    10701
                                7 < 17453*   4282
                                7 > 25611    10703
                                8 < 11030*   10702
                                8 < 17453*   4283
                                8 < 6012     8987
                                9 > 11030*   8984
                                9 > 17453*   8985
                                9 > 23946*   8986
                                9 > 9591     8983
                               10 > 10677*   4275
                               10 > 11030*   4276
                               10 > 13704    4277
                               10 > 17015*   4278
                               10 > 17453*   4279
                               10 < 2136     4273
                               11 > 17453*   4274
                               10 < 2186     14089
                               10 > 23946*   4280
                               10 > 25611*   4281
                               10 < 309*     7249
                                2 > 21314    7466
10679            4       4      0   10679    ROOT
                                1 < 115      9941
                                2 > 5268     9940
                                3 > 10679*   9939
                                1 < 8671     9015
10682            4       4      0   10682    ROOT
                                1 < 2506     6022
                                2 > 9962     6021
                                3 > 10682*   5990
                                3 > 22428    5991
10792           11       7      0   10792    ROOT
                                1 > 15695    9866
                                2 > 20379    9647
                                2 > 21309    9648
                                3 < 20422    9661
                                4 < 2222     9642
                                5 > 10792*   9640
                                5 > 15695*   9641
                                5 > 21309*   9643
                                5 < 2215     9644
                                6 > 15695*   9645
                                6 > 21309*   9646
10818            3       3      0   10818    ROOT
                                1 > 11016    11965
                                2 > 25596    11967
                                3 < 10818*   11966
10872            3       3      0   10872    ROOT
                                1 > 24250    6959
                                2 < 2666     6961
                                3 > 10872*   6960
10884           10       7      0   10884    ROOT
                                1 > 16353    12670
                                2 < 4958     12669
                                3 > 10884*   12668
                                3 > 9606     12667
                                4 > 10884*   9663
                                4 > 12122    9664
                                5 < 5973     13438
                                6 > 9606*    13437
                                4 > 16353*   9665
                                4 < 2247     9662
10897            1       2      0   10897    ROOT
                                1 < 5548     7565
10904            7       7      0   10904    ROOT
                                1 > 12707    9216
                                2 < 5156     4083
                                3 > 10904*   4082
                                3 > 14282    4084
                                4 < 11035    4183
                                3 > 17933    4085
                                3 > 7483     4081
10906            5       6      0   10906    ROOT
                                1 < 1289     10560
                                2 > 1560     10558
                                2 > 4365     10559
                                1 < 8530     5023
                                2 > 21144    5024
1092             2       3      0   1092     ROOT
                                1 > 23226    5795
                                1 > 3196     5794
10936            3       4      0   10936    ROOT
                                1 > 19509    10723
                                1 > 19510    10724
                                1 > 23687    10725
11009           12       8      0   11009    ROOT
                                1 > 20157    7971
                                2 > 23471    7975
                                3 < 11009*   7974
                                1 > 21344    7972
                                2 < 11721    9753
                                3 > 12151    9751
                                4 > 15221    9756
                                5 < 11721*   9752
                                5 > 21344*   9754
                                4 > 21344*   9757
                                2 > 22990    9755
                                3 < 11009*   7973
11028            6       4      0   11028    ROOT
                                1 > 21171    9834
                                2 > 22730    9836
                                3 < 11028*   9835
                                3 < 5211     9833
                                4 > 11028*   9831
                                4 > 21171*   9832
11054            3       3      0   11054    ROOT
                                1 < 8395     14002
                                2 > 8725     14001
                                3 > 11054*   14003
1107             1       2      0   1107     ROOT
                                1 > 6973     6133
11120            3       4      0   11120    ROOT
                                1 > 16340    13294
                                1 > 17851    13295
                                1 > 20233    13296
11196            3       4      0   11196    ROOT
                                1 < 13       9182
                                2 > 19170    9183
                                2 > 7596     9181
11197            3       3      0   11197    ROOT
                                1 < 844      6408
                                2 > 890      6407
                                3 > 11197*   6409
11215            1       2      0   11215    ROOT
                                1 > 23156    13072
11216            2       3      0   11216    ROOT
                                1 > 19598    5564
                                2 < 13815    5565
11279            5       5      0   11279    ROOT
                                1 > 17750    5108
                                2 > 22976    5109
                                2 > 25609    5110
                                2 < 9832     6048
                                3 > 11279*   6047
11280            1       2      0   11280    ROOT
                                1 < 831      11119
11285            3       3      0   11285    ROOT
                                1 > 12193    7864
                                2 < 9618     7863
                                3 > 11285*   7862
11411            1       2      0   11411    ROOT
                                1 > 24242    13714
11418            5       5      0   11418    ROOT
                                1 < 161      5233
                                2 > 19583    5234
                                3 < 97       12273
                                4 > 161*     12272
                                2 > 3105     5232
11427            4       4      0   11427    ROOT
                                1 > 20414    6376
                                2 < 14170    7561
                                3 < 7632     6737
                                4 > 20414*   6738
11459            1       2      0   11459    ROOT
                                1 < 6706     4089
11462            1       2      0   11462    ROOT
                                1 > 17141    5708
11465            3       3      0   11465    ROOT
                                1 > 11577    12723
                                2 < 3239     5601
                                3 > 11465*   5600
11467            1       2      0   11467    ROOT
                                1 > 19575    13141
11539            7       5      0   11539    ROOT
                                1 > 15227    13103
                                2 > 16655    9932
                                3 < 11539*   13104
                                3 < 782      9931
                                4 > 11539*   9929
                                4 > 15227*   9930
                                2 > 18151    9933
1154             6       5      0   1154     ROOT
                                1 > 12017    4312
                                2 > 12930    4314
                                3 < 1154*    4313
                                3 > 17591    4315
                                4 < 8704     7088
                                5 > 12930*   7087
11561            6       6      0   11561    ROOT
                                1 < 9579     11992
                                2 > 16932    11993
                                3 > 19451    11991
                                4 < 9579*    11994
                                2 > 22381    11995
                                2 < 6534     12067
11566            3       3      0   11566    ROOT
                                1 > 11808    3143
                                2 > 18560    3145
                                3 < 11566*   3144
11579            1       2      0   11579    ROOT
                                1 > 15073    12917
11593           10      10      0   11593    ROOT
                                1 > 18415    9430
                                2 < 16818    8830
                                2 > 23648    3437
                                2 < 4633     3436
                                2 < 7194     886
                                3 > 15322    885
                                3 > 20960    887
                                4 < 6355     883
                                5 > 7194*    882
                                3 > 7542     884
11616           12       9      0   11616    ROOT
                                1 > 14149    5047
                                2 > 14662    5042
                                3 < 11616*   5048
                                2 > 16648    5043
                                2 > 19560    5044
                                3 > 23530    9159
                                4 < 14149*   5046
                                4 < 3750     11990
                                5 > 14149*   11988
                                5 > 19560*   11989
                                2 > 22199    5045
                                2 < 8147     13240
11622            5       5      0   11622    ROOT
                                1 < 3852     12250
                                2 > 7877     12249
                                3 > 11622*   6262
                                3 < 1378     6261
                                3 > 15422    6263
11623            1       2      0   11623    ROOT
                                1 > 23621    9950
11641            1       2      0   11641    ROOT
                                1 > 24462    11178
11642            1       2      0   11642    ROOT
                                1 > 25230    13233
11662           10       5      0   11662    ROOT
                                1 > 14094    9983
                                2 > 19014    9985
                                3 < 11662*   9984
                                3 < 5960     9979
                                4 > 11662*   9977
                                4 > 14094*   9978
                                4 > 8705     9976
                                5 > 11662*   9980
                                5 > 14094*   9981
                                5 > 19014*   9982
11718            1       2      0   11718    ROOT
                                1 > 20766    14024
11823           19       9      0   11823    ROOT
                                1 > 13628    13948
                                2 > 15246    13945
                                3 > 20332    13377
                                4 < 13628*   13946
                                4 < 6350     9193
                                5 > 11823*   9189
                                5 > 13628*   9190
                                5 > 15246*   9191
                                5 > 16014    9192
                                5 > 23710    9194
                                6 < 11823*   13949
                                6 < 13628*   13947
                                6 < 8372     13944
                                7 > 11823*   13940
                                7 > 13628*   13941
                                7 > 15246*   13942
                                7 > 20332*   13943
                                7 < 6350*    9188
                                5 < 4495     9195
11828            1       2      0   11828    ROOT
                                1 < 7807     7330
11842            5       5      0   11842    ROOT
                                1 < 1656     5766
                                2 > 21531    5767
                                3 < 5828     5768
                                4 < 1656*    5765
                                2 > 4040     5764
1187             1       2      0   1187     ROOT
                                1 > 6290     9841
11879            7       6      0   11879    ROOT
                                1 > 26092    11947
                                2 < 21187    10770
                                3 < 4051     7891
                                4 > 21313    7892
                                5 < 21310    13661
                                5 > 26092*   7894
                                4 > 26092*   7893
11893            3       3      0   11893    ROOT
                                1 > 25667    13467
                                2 < 8891     13036
                                3 > 11893*   13035
11911            1       2      0   11911    ROOT
                                1 < 9039     12102
1194             1       2      0   1194     ROOT
                                1 > 21858    1756
11965            3       4      0   11965    ROOT
                                1 < 1556     6619
                                2 > 14096    6620
                                2 > 23154    6621
11969            3       3      0   11969    ROOT
                                1 > 18996    10664
                                2 < 1967     10666
                                3 > 11969*   10665
11971            1       2      0   11971    ROOT
                                1 > 25211    11733
11991            1       2      0   11991    ROOT
                                1 < 3186     12664
12034           15       6      0   12034    ROOT
                                1 > 17935    14102
                                2 < 2116     14098
                                3 > 12034*   14097
                                3 > 9189     14094
                                4 > 12034*   14092
                                4 > 17935*   14093
                                4 > 9765     14090
                                5 > 12034*   14100
                                5 > 17935*   14101
                                5 < 2116*    14095
                                5 > 9774     14099
                                6 > 12034*   14103
                                6 > 17935*   14104
                                6 < 2116*    14096
                                6 < 9189*    14091
12041            1       2      0   12041    ROOT
                                1 > 14384    8493
12042           20      12      0   12042    ROOT
                                1 < 5738     3826
                                2 > 17252    3827
                                3 > 18142    12724
                                4 < 5738*    3828
                                4 < 6538     10717
                                5 > 16040    10715
                                5 > 17252*   10716
                                5 < 5478     13900
                                6 > 5738*    13899
                                6 > 8349     13901
                                7 < 1549     9651
                                8 > 22766    9652
                                9 > 23168    9655
                               10 < 1549*    9653
                               10 < 8349*    9650
                                9 < 8349*    9649
                                8 > 24732    9654
                                7 < 5738*    3825
                                7 < 6538*    10714
                                5 < 5738*    3824
12046            7       8      0   12046    ROOT
                                1 > 25957    7833
                                2 < 17724    4986
                                3 > 25958    4987
                                2 < 19126    9486
                                3 < 14135    13923
                                2 < 8263     3572
                                3 > 17882    3571
12050            1       2      0   12050    ROOT
                                1 < 5364     13463
12113            4       4      0   12113    ROOT
                                1 > 16886    12069
                                2 < 142      12755
                                3 > 18626    12756
                                4 < 16886*   12070
12161            1       2      0   12161    ROOT
                                1 > 19162    4316
12192            3       3      0   12192    ROOT
                                1 < 5839     9611
                                2 > 7026     9610
                                3 > 12192*   9612
12213           21       7      0   12213    ROOT
                                1 > 13558    13457
                                2 > 21710    12987
                                3 < 12213*   13458
                                3 > 22318    13461
                                4 < 12213*   13459
                                4 < 13558*   12988
                                4 > 24888    13456
                                5 < 12213*   13460
                                5 < 13558*   12989
                                5 < 21710*   13462
                                5 < 8669     12995
                                6 > 12213*   12991
                                6 > 13558*   12992
                                6 > 21710*   12993
                                6 > 22318*   12994
                                6 > 9038     12990

Network     #Links  #Nodes    Lev Node       Link
---------- ------- ------- ------ ---------- ----------
12213           21       7      7 > 12213*   13451
                                7 > 13558*   13452
                                7 > 21710*   13453
                                7 > 22318*   13454
                                7 > 24888*   13455
12248            1       2      0   12248    ROOT
                                1 < 98       14149
12252            2       3      0   12252    ROOT
                                1 > 22404    10336
                                2 < 9026     13358
12256            2       3      0   12256    ROOT
                                1 > 17237    6040
                                1 < 5081     6039
12290            2       3      0   12290    ROOT
                                1 > 17307    8012
                                1 > 20031    8013
12307            3       3      0   12307    ROOT
                                1 > 14175    12694
                                2 > 22980    12693
                                3 < 12307*   12695
12320            3       3      0   12320    ROOT
                                1 > 12321    6226
                                2 > 19222    6228
                                3 < 12320*   6227
12338            3       3      0   12338    ROOT
                                1 > 15800    13147
                                2 < 1878     13146
                                3 > 12338*   13145
12387            7       6      0   12387    ROOT
                                1 > 19520    7612
                                2 > 23084    7614
                                3 < 12387*   7613
                                3 < 19931    14057
                                3 > 23553    7616
                                3 > 23554    7617
                                4 < 19520*   7615
12414           13       8      0   12414    ROOT
                                1 > 14431    5783
                                2 > 24439    14022
                                3 < 12414*   5785
                                3 < 18562    5786
                                4 < 12414*   5784
                                4 < 3102     5781
                                5 > 12414*   5780
                                5 > 24439*   5782
                                3 < 20537    13299
                                4 > 20543    13298
                                5 > 21339    10744
                                6 > 24439*   10746
                                5 > 24439*   10745
12473            1       2      0   12473    ROOT
                                1 < 9831     13760
12504            1       2      0   12504    ROOT
                                1 < 9053     9044
12554            1       2      0   12554    ROOT
                                1 < 7636     11747
12616            3       3      0   12616    ROOT
                                1 > 16266    4164
                                2 < 4485     4166
                                3 > 12616*   4165
12617            2       3      0   12617    ROOT
                                1 > 17388    7631
                                2 < 6037     8024
12676            2       3      0   12676    ROOT
                                1 > 14302    9722
                                2 < 5432     13653
12689            6       4      0   12689    ROOT
                                1 > 17560    4479
                                2 > 21697    4477
                                3 < 12689*   4480
                                3 > 25116    4476
                                4 < 12689*   4481
                                4 < 17560*   4478
12703            2       3      0   12703    ROOT
                                1 < 1669     4747
                                1 < 5260     6438
12704            2       3      0   12704    ROOT
                                1 < 7861     7495
                                2 < 2355     7496
12751            1       2      0   12751    ROOT
                                1 < 6307     11094
12798            2       3      0   12798    ROOT
                                1 > 19507    6123
                                2 < 3199     7089
12801            4       4      0   12801    ROOT
                                1 > 15641    13864
                                2 > 19548    9870
                                3 < 12801*   13865
                                2 < 5466     9871
12863            1       2      0   12863    ROOT
                                1 > 18390    11997
12940            2       3      0   12940    ROOT
                                1 < 787      9633
                                2 > 20883    9634
13012            3       3      0   13012    ROOT
                                1 < 1983     12737
                                2 > 20943    12738
                                3 < 13012*   12739
13022            1       2      0   13022    ROOT
                                1 < 8077     8653
13031            4       4      0   13031    ROOT
                                1 > 13334    12391
                                2 < 8738     12997
                                3 > 13031*   12996
                                1 > 13620    12392
13167            3       3      0   13167    ROOT
                                1 > 15485    13919
                                2 < 800      7678
                                3 > 13167*   7677
13202            3       4      0   13202    ROOT
                                1 > 16703    8062
                                2 > 22253    3823
                                3 < 4128     5098
13325           14       8      0   13325    ROOT
                                1 > 14502    9507
                                2 > 15899    9510
                                3 < 13325*   9508
                                2 > 20926    9511
                                3 < 13325*   9509
                                3 < 6735     9505
                                4 > 13325*   9503
                                4 > 14502*   9504
                                4 > 23167    9506
                                5 < 3388     13311
                                6 > 3818     13309
                                7 > 23167*   9127
                                7 > 6735*    9126
                                6 > 6735*    13310
13357            4       4      0   13357    ROOT
                                1 > 25115    9305
                                2 < 8739     9304
                                3 > 13357*   9303
                                1 > 25852    9306
13413            1       2      0   13413    ROOT
                                1 < 9211     13312
13415            3       3      0   13415    ROOT
                                1 > 15386    12935
                                2 < 7585     12980
                                3 > 13415*   12979
1344             1       2      0   1344     ROOT
                                1 > 18126    10351
1346             1       2      0   1346     ROOT
                                1 > 6448     4271
13485            2       3      0   13485    ROOT
                                1 > 21530    5692
                                2 < 5059     1701
13486            3       3      0   13486    ROOT
                                1 > 18688    11102
                                2 < 4259     11104
                                3 > 13486*   11103
13621            1       2      0   13621    ROOT
                                1 > 25388    560
13675            1       2      0   13675    ROOT
                                1 < 4186     4163
13677            3       3      0   13677    ROOT
                                1 > 18288    9861
                                2 > 23206    9863
                                3 < 13677*   9862
13717            1       2      0   13717    ROOT
                                1 < 8558     12265
1376             1       2      0   1376     ROOT
                                1 > 4267     10556
13802            6       4      0   13802    ROOT
                                1 > 16823    14183
                                2 > 19553    13907
                                3 < 13802*   14184
                                3 < 5571     14182
                                4 > 13802*   14180
                                4 > 16823*   14181
1383            11       6      0   1383     ROOT
                                1 > 3377     13218
                                2 < 1660     11777
                                3 > 23068    11778
                                4 > 25631    11781
                                5 < 1660*    11779
                                5 > 25678    11776
                                6 < 1660*    11780
                                6 < 23068*   11782
                                6 < 3377*    11785
                                5 < 3377*    11784
                                4 < 3377*    11783
13932            4       4      0   13932    ROOT
                                1 < 6075     1618
                                2 > 14499    1619
                                3 > 20345    1621
                                4 < 6075*    1620
14               1       2      0   14       ROOT
                                1 > 14171    13498
14130            1       2      0   14130    ROOT
                                1 < 3599     9719
14131            1       2      0   14131    ROOT
                                1 < 372      12066
14132            1       2      0   14132    ROOT
                                1 > 15154    11120
14159            4       4      0   14159    ROOT
                                1 > 24852    6980
                                2 < 5144     10685
                                3 > 14159*   10684
                                2 < 8869     8832
14182            1       2      0   14182    ROOT
                                1 > 17537    12666
14338            1       2      0   14338    ROOT
                                1 > 22729    11180
14339            2       3      0   14339    ROOT
                                1 > 18608    13026
                                2 > 22816    13025
14353            6       5      0   14353    ROOT
                                1 > 14886    4074
                                2 < 2043     9914
                                3 > 6315     9913
                                4 > 14886*   5819
                                2 < 2455     4073
                                3 > 14353*   4072
14358            1       2      0   14358    ROOT
                                1 < 5697     12262
14543            4       4      0   14543    ROOT
                                1 > 21221    7497
                                2 > 25130    7499
                                3 < 14543*   7498
                                3 < 7958     6801
14560            3       3      0   14560    ROOT
                                1 > 24251    3804
                                2 < 2656     3803
                                3 > 14560*   3802
14763            1       2      0   14763    ROOT
                                1 < 4432     14004
14770            9       7      0   14770    ROOT
                                1 > 21713    11906
                                2 > 25470    12922
                                3 < 14770*   11908
                                1 > 23461    11907
                                2 < 3297     11746
                                3 > 17936    11745
                                3 > 5667     11744
                                4 > 14770*   9772
                                4 > 23461*   9773
14771            1       2      0   14771    ROOT
                                1 > 24127    4463
14842            2       3      0   14842    ROOT
                                1 > 24575    9988
                                2 < 8255     13350
14845            3       3      0   14845    ROOT
                                1 > 19521    13500
                                2 < 5232     13502
                                3 > 14845*   13501
14868            3       3      0   14868    ROOT
                                1 > 16226    9830
                                2 < 2201     9829
                                3 > 14868*   9828
14894            2       3      0   14894    ROOT
                                1 < 1670     9778
                                1 > 22927    9777
1490             1       2      0   1490     ROOT
                                1 > 20813    3849
14990            1       2      0   14990    ROOT
                                1 > 25975    13930
15181            1       2      0   15181    ROOT
                                1 < 2331     8932
15182           10       5      0   15182    ROOT
                                1 > 16728    12023
                                2 > 22319    12026
                                3 < 15182*   12024
                                3 > 23424    12028
                                4 < 15182*   12025
                                4 < 16728*   12027
                                4 < 8153     12022
                                5 > 15182*   12019
                                5 > 16728*   12020
                                5 > 22319*   12021
15188            1       2      0   15188    ROOT
                                1 < 6354     5992
15208            1       2      0   15208    ROOT
                                1 > 23759    9082
15218            1       2      0   15218    ROOT
                                1 > 17585    8869
15248            5       5      0   15248    ROOT
                                1 > 17595    13446
                                2 > 19018    8520
                                3 < 17076    4176
                                2 < 8443     13445
                                3 > 15248*   13444
15249            1       2      0   15249    ROOT
                                1 > 15624    3148
15259            2       3      0   15259    ROOT
                                1 > 25136    13139
                                2 < 22000    10613
15358            3       3      0   15358    ROOT
                                1 > 18036    12671
                                2 > 18338    12673
                                3 < 15358*   12672
15374            1       2      0   15374    ROOT
                                1 > 15553    14037
15387            1       2      0   15387    ROOT
                                1 > 15388    4994
15396            7       5      0   15396    ROOT
                                1 > 18238    11870
                                2 > 26180    11865
                                3 < 15396*   11871
                                3 < 4777     11869
                                4 > 15396*   11866
                                4 > 18238*   11867
                                4 > 19064    11868
15401            1       2      0   15401    ROOT
                                1 < 4196     13893
15415            3       4      0   15415    ROOT
                                1 > 24640    2865
                                2 < 24152    5206
                                2 < 4685     11740
15423            1       2      0   15423    ROOT
                                1 > 18144    9875
1551             2       3      0   1551     ROOT
                                1 > 8058     9269
                                2 < 2328     13800
15572            1       2      0   15572    ROOT
                                1 < 7356     7467
15583            4       4      0   15583    ROOT
                                1 > 17785    12058
                                2 > 18628    10047
                                2 > 24272    10048
                                3 < 15583*   12059
15609            1       2      0   15609    ROOT
                                1 > 21809    8705
15642            3       3      0   15642    ROOT
                                1 > 16783    14117
                                2 > 17563    10323
                                3 < 15642*   14118
15668            6       4      0   15668    ROOT
                                1 > 17293    10740
                                2 > 25614    10738
                                3 < 15668*   10741
                                3 > 26006    10743
                                4 < 15668*   10742
                                4 < 17293*   10739
15681            2       3      0   15681    ROOT
                                1 > 20793    8829
                                1 < 7482     8828
15688            1       2      0   15688    ROOT
                                1 > 22692    9551
15706            2       3      0   15706    ROOT
                                1 > 20776    6419
                                1 < 374      12104
15712            1       2      0   15712    ROOT
                                1 > 19596    10679
15824            1       2      0   15824    ROOT
                                1 > 22283    4049
15847            3       3      0   15847    ROOT
                                1 < 348      10620
                                2 > 5660     10619
                                3 > 15847*   10621
1589             1       2      0   1589     ROOT
                                1 > 7603     9073
15962            1       2      0   15962    ROOT
                                1 > 25447    13908
16015            1       2      0   16015    ROOT
                                1 > 22938    13260
16109            9       6      0   16109    ROOT
                                1 > 21808    6661
                                2 < 2784     6660
                                3 > 16109*   6659
                                3 > 5425     6658
                                4 > 16109*   2533
                                4 > 16224    2534
                                5 > 24161    2537
                                6 < 5425*    2536
                                4 > 21808*   2535
16124            3       3      0   16124    ROOT
                                1 > 20807    8756
                                2 > 24334    8758
                                3 < 16124*   8757
16129            3       3      0   16129    ROOT
                                1 < 2298     12973
                                2 < 74       12971
                                3 > 16129*   12972
1621             3       3      0   1621     ROOT
                                1 > 6576     11063
                                2 > 7109     11062
                                3 < 1621*    11064
16281            1       2      0   16281    ROOT
                                1 > 24728    4740
16312            1       2      0   16312    ROOT
                                1 > 16314    12399
16338            1       2      0   16338    ROOT
                                1 > 21390    9527
16358            3       3      0   16358    ROOT
                                1 < 822      11830
                                2 > 9518     11829
                                3 > 16358*   11831
16470            1       2      0   16470    ROOT
                                1 > 17822    1078
16484            1       2      0   16484    ROOT
                                1 < 8302     290
16523            2       3      0   16523    ROOT
                                1 < 4110     9867
                                2 > 17696    9868
16609            2       3      0   16609    ROOT
                                1 < 6170     12054
                                2 > 8881     12053
16620            8       6      0   16620    ROOT
                                1 < 1908     8034
                                2 > 16621    8035
                                3 < 2611     6945
                                4 > 16620*   6944
                                4 < 1908*    8033
                                4 < 376      6942
                                5 > 16620*   6943
                                2 > 17173    8036
16622            1       2      0   16622    ROOT
                                1 > 18454    12390
16643            1       2      0   16643    ROOT
                                1 > 20035    12052
16802            3       3      0   16802    ROOT
                                1 > 23757    9016
                                2 < 830      9018
                                3 > 16802*   9017
16892            3       3      0   16892    ROOT
                                1 > 19572    13495
                                2 < 3062     8588
                                3 > 16892*   8587
16922            1       2      0   16922    ROOT
                                1 > 20925    12923
1695             1       2      0   1695     ROOT
                                1 > 19380    7838
16957            2       3      0   16957    ROOT
                                1 < 3362     9779
                                2 < 3347     9780
16971            3       3      0   16971    ROOT
                                1 > 20170    13022
                                2 < 5228     13024
                                3 > 16971*   13023
17178            8       7      0   17178    ROOT
                                1 > 21024    9774
                                2 < 19712    8585
                                2 > 23215    8586
                                3 < 17178*   9775
                                2 < 8071     12245
                                1 < 2329     13300
                                2 > 24846    13301
                                3 < 17178*   9776
17179            3       3      0   17179    ROOT
                                1 > 24816    5256
                                2 < 9871     5255
                                3 > 17179*   5254
17276            3       3      0   17276    ROOT
                                1 > 18383    4020
                                2 < 9994     4019
                                3 > 17276*   4018
17280            7       5      0   17280    ROOT
                                1 > 19956    7470
                                2 < 2785     13505
                                3 > 17280*   13504
                                3 > 3682     13503
                                4 > 17280*   13506
                                4 > 19956*   13507
                                1 > 24597    7471
17291            3       3      0   17291    ROOT
                                1 < 4017     4627
                                2 > 7888     4626
                                3 > 17291*   4628
17346            3       3      0   17346    ROOT
                                1 > 20319    9991
                                2 < 9149     9916
                                3 > 17346*   9915
17461            3       3      0   17461    ROOT
                                1 < 188      7261
                                2 > 22920    7262
                                3 < 17461*   8965
17468            3       3      0   17468    ROOT
                                1 < 3200     9325
                                2 > 4264     9324
                                3 > 17468*   9326
17470            1       2      0   17470    ROOT
                                1 > 19221    12233
17471            1       2      0   17471    ROOT
                                1 < 3989     10774
17503            1       2      0   17503    ROOT
                                1 < 3748     13137
17690            1       2      0   17690    ROOT
                                1 < 5814     5763
17782           10       5      0   17782    ROOT
                                1 > 18231    10034
                                2 < 1989     10031
                                3 > 17782*   10030
                                3 > 4106     10028
                                4 > 17782*   13352
                                4 > 18231*   13353
                                4 > 7515     13351
                                5 > 17782*   10032
                                5 > 18231*   10033
                                5 < 1989*    10029
17865            3       4      0   17865    ROOT
                                1 > 24159    9131
                                2 < 4276     10554
                                2 < 717      9713
17951            1       2      0   17951    ROOT
                                1 < 833      8539
1798             3       3      0   1798     ROOT
                                1 > 4828     11136
                                2 > 5404     11138
                                3 < 1798*    11137
17992            1       2      0   17992    ROOT
                                1 < 4181     14179
18003            2       3      0   18003    ROOT
                                1 < 5545     8014
                                1 < 6709     3077
18034           18       8      0   18034    ROOT
                                1 < 1981     9113
                                2 > 19879    9114
                                3 < 18034*   9120
                                3 < 5468     9119
                                4 > 18034*   9118
                                4 < 1981*    9109
                                4 > 5668     9116
                                5 > 18034*   11977
                                5 < 1981*    9110
                                5 > 19879*   11978
                                5 > 8187     11976
                                6 > 18034*   9514
                                6 < 1981*    9112
                                6 > 19879*   9515
                                6 < 5468*    9117
                                2 > 25444    9115
                                3 < 6908     11813
                                4 < 1981*    9111
18098            1       2      0   18098    ROOT
                                1 < 3100     7819
18156            1       2      0   18156    ROOT
                                1 < 5387     12715
18171            2       3      0   18171    ROOT
                                1 < 5413     1730
                                1 < 85       7675
18183            1       2      0   18183    ROOT
                                1 < 4278     11748
18194            1       2      0   18194    ROOT
                                1 > 21949    12248
1821             3       3      0   1821     ROOT
                                1 < 187      6296
                                2 > 21386    6297
                                3 < 1821*    6298
18279            3       3      0   18279    ROOT
                                1 < 6389     8481
                                2 > 6914     8480
                                3 > 18279*   8482
1836             1       2      0   1836     ROOT
                                1 > 8400     12006
18388            1       2      0   18388    ROOT
                                1 > 20654    13869
18417            1       2      0   18417    ROOT
                                1 > 19892    13920
18544            3       3      0   18544    ROOT
                                1 > 21596    10321
                                2 < 2332     10320
                                3 > 18544*   10319
18581            1       2      0   18581    ROOT
                                1 < 2938     9797
18596            3       3      0   18596    ROOT
                                1 > 23771    12385
                                2 < 4144     12384
                                3 > 18596*   12383
18669            4       4      0   18669    ROOT
                                1 > 19866    10338
                                2 > 25545    10340
                                3 < 18669*   10339
                                3 < 8538     12898
18721            1       2      0   18721    ROOT
                                1 < 3067     6377
18895            1       2      0   18895    ROOT
                                1 < 7957     8872
18908            1       2      0   18908    ROOT
                                1 > 25531    13413
18970           10       5      0   18970    ROOT
                                1 < 2409     9807
                                2 > 3522     9804
                                3 > 18970*   9810
                                3 > 4286     9808
                                4 > 18970*   11115
                                4 < 2409*    9805
                                4 > 4288     11114
                                5 > 18970*   11116
                                5 < 2409*    9806
                                5 < 3522*    9809
18984            1       2      0   18984    ROOT
                                1 < 2476     6293
19015           10       5      0   19015    ROOT
                                1 > 19444    6633
                                2 > 25139    6635
                                3 < 19015*   6634
                                3 < 5212     6629
                                4 > 19015*   6627
                                4 > 19444*   6628
                                4 > 6628     6626
                                5 > 19015*   6630
                                5 > 19444*   6631
                                5 > 25139*   6632
19052            1       2      0   19052    ROOT
                                1 < 7713     289
19061            1       2      0   19061    ROOT
                                1 < 9094     1873
19145            1       2      0   19145    ROOT
                                1 > 23760    12719
1916             3       3      0   1916     ROOT
                                1 > 20953    6416
                                2 > 24463    6418
                                3 < 1916*    6417
19252            1       2      0   19252    ROOT
                                1 < 3195     6456
19257            3       3      0   19257    ROOT
                                1 > 22961    9462
                                2 < 7477     9461
                                3 > 19257*   9460
19314           15       6      0   19314    ROOT
                                1 > 19880    13363
                                2 > 25496    13371
                                3 < 19314*   13364
                                3 > 25543    13360
                                4 < 19314*   13365
                                4 < 19880*   13372
                                4 > 26019    13368
                                5 < 19314*   13366
                                5 < 19880*   13373
                                5 < 25496*   13361
                                5 > 26048    13370
                                6 < 19314*   13367
                                6 < 19880*   13374
                                6 < 25496*   13362
                                6 < 25543*   13369
194              1       2      0   194      ROOT
                                1 > 8628     7522
19466            3       3      0   19466    ROOT
                                1 > 24128    13695
                                2 < 7543     13697
                                3 > 19466*   13696
19473            1       2      0   19473    ROOT
                                1 > 23812    13340
19502            3       3      0   19502    ROOT
                                1 > 23970    13101
                                2 < 7718     13100
                                3 > 19502*   13099
19554            3       3      0   19554    ROOT
                                1 > 24878    7606
                                2 < 6159     7605
                                3 > 19554*   7604
1969             3       3      0   1969     ROOT
                                1 > 24141    13017
                                2 < 8472     13018
                                3 < 1969*    13016
1976             1       2      0   1976     ROOT
                                1 > 23672    14065
19906            1       2      0   19906    ROOT
                                1 > 26102    9469
19911            1       2      0   19911    ROOT
                                1 > 25487    7946
19932           12       7      0   19932    ROOT
                                1 > 19936    3994
                                2 < 2118     4090
                                2 > 25864    3996
                                3 < 19932*   3995
                                3 < 4199     3990
                                4 > 19932*   3988
                                4 > 19936*   3989
                                4 > 6443     3987
                                5 > 19932*   3991
                                5 > 19936*   3992
                                5 > 25864*   3993
                                2 > 26132    3997
19941            1       2      0   19941    ROOT
                                1 < 6079     9479
2004             1       2      0   2004     ROOT
                                1 > 21191    900
20084            1       2      0   20084    ROOT
                                1 > 22311    11060
20150            3       3      0   20150    ROOT
                                1 > 22312    9946
                                2 > 25226    9948
                                3 < 20150*   9947
20216            3       3      0   20216    ROOT
                                1 < 2223     12050
                                2 > 4266     12049
                                3 > 20216*   12051
20309            3       3      0   20309    ROOT
                                1 < 7267     13033
                                2 > 7280     13032
                                3 > 20309*   13031
20320            1       2      0   20320    ROOT
                                1 < 4643     13929
20333            1       2      0   20333    ROOT
                                1 < 4262     13275
20433            3       3      0   20433    ROOT
                                1 < 5446     11939
                                2 > 5632     11938
                                3 > 20433*   11940
20707            1       2      0   20707    ROOT
                                1 > 24820    13375
20803            2       3      0   20803    ROOT
                                1 > 25231    9268
                                1 < 6356     12964
20914            3       3      0   20914    ROOT
                                1 < 6008     6038
                                2 > 9959     6037
                                3 > 20914*   6036
21018            1       2      0   21018    ROOT
                                1 < 5479     8725
21028            1       2      0   21028    ROOT
                                1 < 8146     7558
2117             1       2      0   2117     ROOT
                                1 > 6300     8692
2119             1       2      0   2119     ROOT
                                1 > 7637     2066
2120             1       2      0   2120     ROOT
                                1 > 8157     288
21206            2       3      0   21206    ROOT
                                1 > 23175    1940
                                2 > 25662    1837
21288            3       4      0   21288    ROOT
                                1 < 8557     6253
                                2 > 26191    6254
                                2 < 8308     6255
21303            1       2      0   21303    ROOT
                                1 < 734      12168
21515            4       4      0   21515    ROOT
                                1 > 21523    9322
                                2 < 7916     8070
                                3 > 21515*   8069
                                3 < 28       8068
21579            1       2      0   21579    ROOT
                                1 < 373      8982
21593            1       2      0   21593    ROOT
                                1 > 22325    9803
21623            1       2      0   21623    ROOT
                                1 > 22605    12034
21684            6       5      0   21684    ROOT
                                1 > 22719    9328
                                2 < 21821    5235
                                2 > 22720    5236
                                3 < 21684*   9329
                                2 > 24169    5237
                                3 < 21684*   9330
22021            1       2      0   22021    ROOT
                                1 < 8811     10312
22110            1       2      0   22110    ROOT
                                1 < 5231     11035
22378            1       2      0   22378    ROOT
                                1 < 2762     4741
22435            1       2      0   22435    ROOT
                                1 < 5406     13288
2248             1       2      0   2248     ROOT
                                1 > 8409     12903
22827            3       3      0   22827    ROOT
                                1 < 5256     13257
                                2 > 5257     13256
                                3 > 22827*   13258
22891            1       2      0   22891    ROOT
                                1 < 25       8058
2304             6       4      0   2304     ROOT
                                1 < 360      11887
                                2 > 7602     11888
                                3 < 2304*    11890
                                3 > 8307     11892
                                4 < 2304*    11891
                                4 < 360*     11889
23094            1       2      0   23094    ROOT
                                1 < 2561     13015
2339             1       2      0   2339     ROOT
                                1 > 24594    3965
23416            1       2      0   23416    ROOT
                                1 < 3074     7925
23465            1       2      0   23465    ROOT
                                1 < 598      4505
23480            1       2      0   23480    ROOT
                                1 > 23652    5173
23772            1       2      0   23772    ROOT
                                1 < 735      12948
23776            1       2      0   23776    ROOT
                                1 < 8708     460
23811            3       3      0   23811    ROOT
                                1 < 4038     13143
                                2 > 7058     13142
                                3 > 23811*   13144
23856            2       3      0   23856    ROOT
                                1 > 24451    5693
                                2 < 4252     1759
23881            1       2      0   23881    ROOT
                                1 < 5577     7611
24202            1       2      0   24202    ROOT
                                1 < 2624     7464
24214            1       2      0   24214    ROOT
                                1 < 4116     13349
24464            1       2      0   24464    ROOT
                                1 < 8336     9272
24504            1       2      0   24504    ROOT
                                1 < 8054     12033
24733            1       2      0   24733    ROOT
                                1 < 6776     13049
24957            3       3      0   24957    ROOT
                                1 < 4019     6240
                                2 > 5626     6239
                                3 > 24957*   6241
25062            1       2      0   25062    ROOT
                                1 < 8735     12072
25095            1       2      0   25095    ROOT
                                1 < 9130     13214
254              1       2      0   254      ROOT
                                1 > 7572     9945
25492            9       6      0   25492    ROOT
                                1 > 25511    13407
                                2 < 8512     13406
                                3 > 25492*   13405
                                3 > 9715     13404
                                4 > 25492*   11133
                                4 > 25511*   11134
                                4 < 4034     11130
                                5 > 9717     11131
                                6 < 9715*    11132
25510            1       2      0   25510    ROOT
                                1 < 3279     9327
2557             1       2      0   2557     ROOT
                                1 > 4272     9656
25676            3       3      0   25676    ROOT
                                1 < 350      3528
                                2 > 951      3527
                                3 > 25676*   3529
2569             1       2      0   2569     ROOT
                                1 > 5546     1820
25853            1       2      0   25853    ROOT
                                1 < 8404     8057
26022            3       3      0   26022    ROOT
                                1 < 4021     8577
                                2 > 4826     8576
                                3 > 26022*   8575
294              1       2      0   294      ROOT
                                1 > 8626     4086
2981             1       2      0   2981     ROOT
                                1 > 4427     9683
3206             2       3      0   3206     ROOT
                                1 > 6535     7976
                                1 > 6579     7977
3844             1       2      0   3844     ROOT
                                1 < 82       9542
410              1       2      0   410      ROOT
                                1 > 4366     13307
4254             1       2      0   4254     ROOT
                                1 > 9337     2681
4954             1       2      0   4954     ROOT
                                1 > 4955     9567
5083             3       3      0   5083     ROOT
                                1 > 5084     3494
                                2 > 8711     3496
                                3 < 5083*    3495
5137             1       2      0   5137     ROOT
                                1 > 6884     9209
5261             1       2      0   5261     ROOT
                                1 > 6282     13379
5409             1       2      0   5409     ROOT
                                1 > 8623     9568
5486             1       2      0   5486     ROOT
                                1 > 8366     13376
5490             1       2      0   5490     ROOT
                                1 > 8474     14046
5492             1       2      0   5492     ROOT
                                1 > 5853     13722
6072             1       2      0   6072     ROOT
                                1 > 8887     914
6161             1       2      0   6161     ROOT
                                1 > 7055     8578
6229             2       3      0   6229     ROOT
                                1 > 6423     7947
                                1 > 7647     7948
6446             1       2      0   6446     ROOT
                                1 > 8036     9161
6669             1       2      0   6669     ROOT
                                1 > 9865     9691
6744             1       2      0   6744     ROOT
                                1 > 9595     4285
7534             1       2      0   7534     ROOT
                                1 > 7536     10346
7588             1       2      0   7588     ROOT
                                1 > 7590     9718
8534             1       2      0   8534     ROOT
                                1 > 8676     12343
8615             1       2      0   8615     ROOT
                                1 < 925      3541

14838 rows selected.

Elapsed: 00:00:10.56
</pre>
</div>

#### Network Summaries

<div class="scrollbox">
<pre>
Network summary 1 - by network

Network     #Links  #Nodes    Max Lev
---------- ------- ------- ----------
21593            2       2          1
21623            2       2          1
22021            2       2          1
22110            2       2          1
22378            2       2          1
22435            2       2          1
2248             2       2          1
22891            2       2          1
23094            2       2          1
2339             2       2          1
23416            2       2          1
23465            2       2          1
23480            2       2          1
23772            2       2          1
23776            2       2          1
23881            2       2          1
24202            2       2          1
24214            2       2          1
24464            2       2          1
24504            2       2          1
24733            2       2          1
25062            2       2          1
25095            2       2          1
254              2       2          1
25510            2       2          1
2557             2       2          1
2569             2       2          1
25853            2       2          1
294              2       2          1
2981             2       2          1
3844             2       2          1
410              2       2          1
4254             2       2          1
4954             2       2          1
5137             2       2          1
5261             2       2          1
5409             2       2          1
5486             2       2          1
5490             2       2          1
5492             2       2          1
6072             2       2          1
6161             2       2          1
6446             2       2          1
6669             2       2          1
6744             2       2          1
7534             2       2          1
7588             2       2          1
8534             2       2          1
8615             2       2          1
15208            2       2          1
15218            2       2          1
15249            2       2          1
15374            2       2          1
15387            2       2          1
15401            2       2          1
15423            2       2          1
15572            2       2          1
15609            2       2          1
15688            2       2          1
15712            2       2          1
15824            2       2          1
1589             2       2          1
15962            2       2          1
16015            2       2          1
16281            2       2          1
16312            2       2          1
16338            2       2          1
16470            2       2          1
16484            2       2          1
16622            2       2          1
16643            2       2          1
16922            2       2          1
1695             2       2          1
17470            2       2          1
17471            2       2          1
17503            2       2          1
17690            2       2          1
17951            2       2          1
17992            2       2          1
18098            2       2          1
18156            2       2          1
18183            2       2          1
18194            2       2          1
1836             2       2          1
18388            2       2          1
18417            2       2          1
18581            2       2          1
18721            2       2          1
18895            2       2          1
18908            2       2          1
18984            2       2          1
19052            2       2          1
19061            2       2          1
19145            2       2          1
19252            2       2          1
194              2       2          1
19473            2       2          1
1976             2       2          1
19906            2       2          1
19911            2       2          1
19941            2       2          1
2004             2       2          1
20084            2       2          1
20320            2       2          1
20333            2       2          1
20707            2       2          1
21018            2       2          1
21028            2       2          1
2117             2       2          1
2119             2       2          1
2120             2       2          1
21303            2       2          1
21579            2       2          1
10002            2       2          1
1010             2       2          1
10133            2       2          1
10148            2       2          1
10356            2       2          1
10382            2       2          1
10407            2       2          1
10433            2       2          1
1050             2       2          1
10517            2       2          1
10522            2       2          1
10559            2       2          1
10602            2       2          1
10638            2       2          1
10676            2       2          1
10897            2       2          1
1107             2       2          1
11215            2       2          1
11280            2       2          1
11411            2       2          1
11459            2       2          1
11462            2       2          1
11467            2       2          1
11579            2       2          1
11623            2       2          1
11641            2       2          1
11642            2       2          1
11718            2       2          1
11828            2       2          1
1187             2       2          1
11911            2       2          1
1194             2       2          1
11971            2       2          1
11991            2       2          1
12041            2       2          1
12050            2       2          1
12161            2       2          1
12248            2       2          1
12473            2       2          1
12504            2       2          1
12554            2       2          1
12751            2       2          1
12863            2       2          1
13022            2       2          1
13413            2       2          1
1344             2       2          1
1346             2       2          1
13621            2       2          1
13675            2       2          1
13717            2       2          1
1376             2       2          1
14               2       2          1
14130            2       2          1
14131            2       2          1
14132            2       2          1
14182            2       2          1
14338            2       2          1
14358            2       2          1
14763            2       2          1
14771            2       2          1
1490             2       2          1
14990            2       2          1
15181            2       2          1
15188            2       2          1
10115            3       3          1
1013             3       3          2
10317            3       3          1
10432            3       3          1
10513            3       3          1
1092             3       3          1
11216            3       3          2
12252            3       3          2
12256            3       3          1
12290            3       3          1
12617            3       3          2
12676            3       3          2
12703            3       3          1
12704            3       3          2
12798            3       3          2
12940            3       3          2
13485            3       3          2
14339            3       3          2
14842            3       3          2
14894            3       3          1
15259            3       3          2
1551             3       3          2
15681            3       3          1
15706            3       3          1
16523            3       3          2
16609            3       3          2
16957            3       3          2
18003            3       3          1
18171            3       3          1
20803            3       3          1
21206            3       3          2
23856            3       3          2
3206             3       3          1
6229             3       3          1
10178            4       3          3
10180            4       3          3
10252            4       3          3
1051             4       3          3
10818            4       3          3
10872            4       3          3
10936            4       4          1
11054            4       3          3
11120            4       4          1
11196            4       4          2
11197            4       3          3
11285            4       3          3
11465            4       3          3
11566            4       3          3
11893            4       3          3
11965            4       4          2
11969            4       3          3
12192            4       3          3
12307            4       3          3
12320            4       3          3
12338            4       3          3
12616            4       3          3
13012            4       3          3
13167            4       3          3
13202            4       4          3
13415            4       3          3
13486            4       3          3
13677            4       3          3
14560            4       3          3
14845            4       3          3
14868            4       3          3
15358            4       3          3
15415            4       4          2
15642            4       3          3
15847            4       3          3
16124            4       3          3
16129            4       3          3
1621             4       3          3
16358            4       3          3
16802            4       3          3
16892            4       3          3
16971            4       3          3
17179            4       3          3
17276            4       3          3
17291            4       3          3
17346            4       3          3
17461            4       3          3
17468            4       3          3
17865            4       4          2
1798             4       3          3
1821             4       3          3
18279            4       3          3
18544            4       3          3
18596            4       3          3
1916             4       3          3
19257            4       3          3
19466            4       3          3
19502            4       3          3
19554            4       3          3
1969             4       3          3
20150            4       3          3
20216            4       3          3
20309            4       3          3
20433            4       3          3
20914            4       3          3
21288            4       4          2
22827            4       3          3
23811            4       3          3
24957            4       3          3
25676            4       3          3
26022            4       3          3
5083             4       3          3
13031            5       4          3
12801            5       4          3
18669            5       4          3
14543            5       4          3
11427            5       4          4
10682            5       4          3
10679            5       4          3
15583            5       4          3
10637            5       4          3
21515            5       4          3
13357            5       4          3
12113            5       4          4
13932            5       4          4
14159            5       4          3
10154            6       4          4
11842            6       5          4
10906            6       6          2
15248            6       5          3
10355            6       6          2
11418            6       5          4
11279            6       5          3
11622            6       5          3
12689            7       4          4
11561            7       6          4
1154             7       5          5
11028            7       4          4
10672            7       4          4
10564            7       5          3
10341            7       4          4
2304             7       4          4
21684            7       5          3
15668            7       4          4
14353            7       5          4
13802            7       4          4
15396            8       5          4
11539            8       5          4
12046            8       8          3
10904            8       7          4
11879            8       6          5
12387            8       6          4
17280            8       5          4
17178            9       7          3
16620            9       6          5
16109           10       6          6
14770           10       7          4
25492           10       6          6
10884           11       7          6
11593           11      10          5
11662           11       5          5
15182           11       5          5
17782           11       5          5
18970           11       5          5
19015           11       5          5
1383            12       6          6
10792           12       7          6
10056           12       6          5
19932           13       7          5
11616           13       9          5
11009           13       8          5
12414           14       8          6
13325           15       8          7
10338           15       7          6
19314           16       6          6
12034           16       6          6
18034           19       8          6
11823           20       9          7
12042           21      12         10
12213           22       7          7
10163           25       8          8
10677           30      14         11
1000         13423    4158       1146

354 rows selected.

Elapsed: 00:00:01.75
Network summary 2 - grouped by numbers of nodes

 #Nodes #Networks
------- ---------
      2       177
      3        98
      4        30
      5        17
      6        12
      7         8
      8         6
      9         2
     10         1
     12         1
     14         1
   4158         1

12 rows selected.

Elapsed: 00:00:01.44
</pre>
</div>

The results show that the data set contains 354 connected networks, with one much larger than the rest, having 4158 nodes. This (more or less) matches with the results we got from source 3466. Actually, we should get one fewer record back than the number of nodes in the network, but as in the earlier article the SQL returns the source node in one record - we can easily fix this, but it's not worthwhile my redoing all the results, so I leave it as is.

As another check, we can run against the second largest network, using 10677 as the source, which should give 14 records. Here is the result for SP\_GTTRSF\_Q, LEVMAX=10.

```
NODE                            LEV  MAXLEV  INTNOD  INTMAX PATH                                                                            
------------------------------ ---- ------- ------- ------- --------------------------------------------------------------------------------
309                               1       2       1       2 309
9591                              1       2       1       2 9591
..2136                            2       2       2       2 9591,2136
..2186                            2       2       2       2 9591,2186
..6012                            2       2       2       2 9591,6012
..11030                           2       2       2       2 9591,11030
..13704                           2       2       2       2 9591,13704
..17453                           2       2       2       2 9591,17453
..23946                           2       2       2       2 9591,23946
..25611                           2       2       2       2 9591,25611
17015                             1       2       1       2 17015
..10677                           2       2       2       2 9591,10677
..20560                           2       2       2       2 309,20560
..21314                           2       2       2       2 17015,21314

14 rows selected.

```

This is consistent with the network output with source node appearing once. The network output (usually I indent the records by level, but the level is very high in some networks here) is:

```
Network     #Links  #Nodes    Lev Node       Link
---------- ------- ------- ------ ---------- ----------
10677           29      14      0   10677    ROOT
                                1 > 17015    9130
                                2 > 20560    7465
                                3 < 309      7253
                                4 > 10677*   7250
                                4 > 17015*   7251
                                4 > 17453    7252
                                5 < 11030    10700
                                6 > 23946    10701
                                7 < 17453*   4282
                                7 > 25611    10703
                                8 < 11030*   10702
                                8 < 17453*   4283
                                8 < 6012     8987
                                9 > 11030*   8984
                                9 > 17453*   8985
                                9 > 23946*   8986
                                9 > 9591     8983
                               10 > 10677*   4275
                               10 > 11030*   4276
                               10 > 13704    4277
                               10 > 17015*   4278
                               10 > 17453*   4279
                               10 < 2136     4273
                               11 > 17453*   4274
                               10 < 2186     14089
                               10 > 23946*   4280
                               10 > 25611*   4281
                               10 < 309*     7249
                                2 > 21314    7466

```

## Results: Friendship network of Brightkite users

[Friendship network of Brightkite users](https://snap.stanford.edu/data/loc-brightkite.html)

> Brightkite was once a location-based social networking service provider where users shared their locations by checking-in. The friendship network was collected using their public API, and consists of 58,228 nodes and 214,078 edges. The network is originally directed but we have constructed a network with undirected edges when there is a friendship in both ways.

The data set comes with the reverse arcs already present, making a total of 428,156 arcs.

I took the first node in the first line in the data set file as the initial root node, 0, and tested only the method SP\_GTTRSF\_I/Q for values of LEVMAX of 5, 10. The complete results, including execution plans are in the attached file.

The exact solution has 56,739 nodes reached from the source node 0, with a maximum level of 10. The output is a bit large to embed, so attaching it in file, but here are the last few lines of the exact solution:

```
122                               1      10       1      13 122
..0                               2      10       2      13 99,0
..6534                            2      10       2      13 122,6534
....15726                         3      10       3      13 40,647,15726
......42871                       4      10       4      13 69,4994,12608,42871
......45149                       4      10       4      13 40,2886,15714,45149
......45850                       4      10       4      13 122,6534,15726,45850
......45851                       4      10       4      13 40,2886,22440,45851
......45852                       4      10       4      13 122,6534,15726,45852
......45853                       4      10       4      13 40,2886,26635,45853
......45854                       4      10       4      13 36,2625,15736,45854
......45855                       4      10       4      13 122,6534,15726,45855
......45856                       4      10       4      13 122,6534,15726,45856
....15744                         3      10       3      13 40,647,15744
....34944                         3      10       3      13 122,6534,34944
....34945                         3      10       3      13 122,6534,34945
....34946                         3      10       3      13 122,6534,34946

56739 rows selected.

Elapsed: 00:01:32.49

```

In summary, as shown in the embedded Excel file, the exact solution is found in a total of 105 seconds and 344 seconds (based on Xplan timings) for LEVMAX=5 and 10 respectively.

<iframe src="https://onedrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95ec670ea6af8ed1%2179873&amp;authkey=AGIzbHO2Ok3HoBk&amp;em=2" width="800" height="300" frameborder="0" scrolling="no"></iframe>

I ran my network analysis program on this network too, and here is the final grouped results, showing consistency with the largest connected network of 56,739 nodes.

```
Network summary 2 - grouped by numbers of nodes

 #Nodes #Networks
------- ---------
      2       362
      3       103
      4        40
      5        21
      6         9
      7         3
      8         2
      9         1
     10         2
     11         2
     49         1
  56739         1

12 rows selected.

Elapsed: 00:00:24.17

```

We can do a further test by using a source from a smaller network. 51944 is a root of the network of size 49 nodes (as the full output shows). Here is the result of sourcing from that, again consistent witht he network analysis program that uses a completely diffferent algorithm:

```
NODE                            LEV  MAXLEV  INTNOD  INTMAX PATH
------------------------------ ---- ------- ------- ------- --------------------------------------------------------------------------------
57077                             1       7       1       9 57077
..51944                           2       7       2       9 57077,51944
..57969                           2       7       2       9 57077,57969
....58151                         3       7       3       9 57077,57969,58151
......58195                       4       7       4       9 57077,57969,58151,58195
........58218                     5       7       5       9 57077,57969,58151,58195,58218
......58196                       4       7       4       9 57077,57969,58151,58196
......58197                       4       7       4       9 57077,57969,58151,58197
......58198                       4       7       4       9 57077,57969,58151,58198
......58199                       4       7       4       9 57077,57969,58151,58199
......58201                       4       7       4       9 57077,57969,58151,58201
......58202                       4       7       4       9 57077,57969,58151,58202
......58203                       4       7       4       9 57077,57969,58151,58203
....58152                         3       7       3       9 57077,57969,58152
......58205                       4       7       4       9 57077,57969,58152,58205
........58219                     5       7       5       9 57077,57969,58152,58205,58219
..........58224                   6       7       6       9 57077,57969,58152,58205,58219,58224
........58220                     5       7       5       9 57077,57969,58152,58205,58220
..........58226                   6       7       8       9 57077,57969,58152,58205,58220,58226
............58227                 7       7       8       9 57077,57969,58152,58205,58220,58225,58227
..........58225                   6       7       9       9 57077,57969,58152,58205,58220,58225
........58221                     5       7       5       9 57077,57969,58152,58205,58221
......58207                       4       7       4       9 57077,57969,58152,58207
....58153                         3       7       3       9 57077,57969,58153
....58154                         3       7       3       9 57077,57969,58154
......58208                       4       7       4       9 57077,57969,58154,58208
......58209                       4       7       4       9 57077,57969,58154,58209
....58156                         3       7       3       9 57077,57969,58156
....58159                         3       7       3       9 57077,57969,58159
......58200                       4       7       4       9 57077,57969,58159,58200
....58160                         3       7       3       9 57077,57969,58160
......58210                       4       7       4       9 57077,57969,58160,58210
........58222                     5       7       5       9 57077,57969,58160,58210,58222
......58211                       4       7       4       9 57077,57969,58160,58211
......58212                       4       7       4       9 57077,57969,58160,58212
........58223                     5       7       5       9 57077,57969,58160,58212,58223
....58161                         3       7       3       9 57077,57969,58161
......58204                       4       7       4       9 57077,57969,58161,58204
......58206                       4       7       4       9 57077,57969,58161,58206
..57970                           2       7       2       9 57077,57970
....58155                         3       7       3       9 57077,57970,58155
....58157                         3       7       3       9 57077,57970,58157
....58158                         3       7       3       9 57077,57970,58158
..57971                           2       7       2       9 57077,57971
..57972                           2       7       2       9 57077,57972
....58162                         3       7       3       9 57077,57972,58162
..57973                           2       7       2       9 57077,57973
..57974                           2       7       2       9 57077,57974
....58163                         3       7       3       9 57077,57974,58163

49 rows selected.

Elapsed: 00:00:00.21

```

See also: [PL/SQL Pipelined Function for Network Analysis](https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/)

