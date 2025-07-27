---
layout: post
title: "SQL for Shortest Path Problems"
date: 2015-04-19
migrated: true
categories: 
  - "oracle"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "dijkstra"
  - "path"
  - "recursive"
  - "shortest"
  - "sql"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

I came across an interesting problem on OTN last weekend: [How to use Recursive Subquery Factoring (RSF) to Implement Dijkstra’s shortest path algorithm?](https://community.oracle.com/thread/3698740?sr=stream).

The problem is to find the shortest routes through a network to each point from a given source, and the poster helpfully includes both SQL for a test data set, and links to the origin of the data set, [The Stagecoach Problem](http://www.ime.unicamp.br/~andreani/MS515/capitulo7.pdf), and to the Wikipedia page on the algorithm mentioned, [Dijkstra's algorithm](http://en.wikipedia.org/wiki/Dijkstra%27s_algorithm).

I took the problem definition and worked out my own solution and posted it on the thread. I hadn't realised at the time, but as another poster pointed out, the solution I posted is actually very similar to the query that the original poster had included. It uses the same basic idea of traversing the network recursively, while using analytic functions to exclude from further progress any routes to a given node that are inferior to another at the same iteration.

The main difference is that I began with a subquery to add in the links in the reverse direction in order to make the network undirected, which seems to me to be natural for the source problem referenced above. However, the same query can be used in both cases if you separate out the possible addition of reverse links onto the database table. In this article I take a closer look at how the query works in both cases, and I hope the article may have some wider value in relation to SQL recursion also.

I have used similar techniques in the past for other combinatorial problems, including these more complex examples: [SQL for the Fantasy Football Knapsack Problem](http://aprogrammerwrites.eu/?p=878) (June 2013), and [SQL for the Travelling Salesman Problem](https://brenpatf.github.io/migrated/sql-for-the-travelling-salesman-problem) (July 2013).

**Updates**

- **4 May 2015**: I have just posted [SQL for Shortest Path Problems 2: A Branch and Bound Approach](https://brenpatf.github.io/migrated/sql-for-shortest-path-problems-2-a-branch-and-bound-approach/)
- **19 November 2017**: I have now put the code and the dataset installation onto GitHub: [Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos)
- **July 2025**: See also this 2022 article of mine, [Shortest Path Analysis of Large Networks by SQL and PL/SQL](https://brenpatf.github.io/2022/08/07/shortest-path-analysis-of-large-networks-by-sql-and-plsql.html)

## Network Diagram

I copied the diagram in the article referenced above.

### Test Network

<img src="/migrated_images/2015/04/Dijkstra-network.jpg" alt="Dijkstra - network" title="Dijkstra - network" />

### Shortest Path Routes for A-J

<img src="/migrated_images/2015/04/Dijkstra-solutions.jpg" alt="Dijkstra - solutions" title="Dijkstra - solutions" />

## Main Query

### SQL

```sql
WITH paths (node, path, cost, rnk, lev) AS (
SELECT a.dst, a.src || ',' || a.dst, a.distance, 1, 1
  FROM arcs a
WHERE a.src = :SRC
 UNION ALL
SELECT a.dst, 
        p.path || ',' || a.dst, 
        p.cost + a.distance, 
        Rank () OVER (PARTITION BY a.dst ORDER BY p.cost + a.distance),
        p.lev + 1
  FROM paths p
  JOIN arcs a
    ON a.src = p.node
   AND p.rnk = 1
)  SEARCH DEPTH FIRST BY node SET line_no
CYCLE node SET lp TO '*' DEFAULT ' '
, paths_ranked AS (
SELECT lev, node, path, cost, Rank () OVER (PARTITION BY node ORDER BY cost) rnk_t, lp, line_no
  FROM paths
  WHERE rnk = 1
)
SELECT LPad (node, 1 + 2* (lev - 1), '.') node, lev, path, cost, lp
  FROM paths_ranked
  WHERE rnk_t = 1
  ORDER BY line_no
```

### How It Works

- Get all links from the source
    - Initialise path as (source node, destination node) string
    - Initialise cost as the distance for each link
- At each iteration:
    - Join from all records in the previous iteration that meet certain criteria to all reachable destinations
    - Accumulate the route by adding the new node on to the path string
    - Accumulate the route cost by adding the new distance on to the cost value
    - Get the analytic rank of the new route in descending order of cost, partitioning by destination node
- The criteria for joining to a record in the previous iteration are:
    - Exclude previous iteration routes that form a loop (with one special exception noted later)
    - Include only the previous iteration routes that have a rank of 1 for cost to their destination node
- After the recursion is complete, use a new subquery on the recursive factor to:
    - Rank the routes by cost, again partitioning by destination node, giving a global rank to each route, for each destination node
    - Include only the routes having the iteration-level rank of 1
- The main query selects out the routes having global rank of 1

### Why It Works

- A recursive traversal of the network without the ranking criterion clearly obtains all non-looping routes
    - Obviously no optimal route can contain a loop
- Excluding the sub-optimal routes with respect to each node at the iteration level excludes no overall-optimal routes
    - No optimal route can contain a sub-optimal sub-route
- The exclusions within the recursion only exclude routes that are sub-optimal with respect to other routes at the same iteration (or level), so the final global ranking step is necessary

## Directed Case

### Solution - one-way from A

The solution below is listed in depth-first recursion order, with indentation, which displays the network structure nicely. THe column LP has a '\*' if the path ends in a loop, but of course there are none in the final solution.

```
NODE                  LEV PATH                  COST LP
-------------------- ---- -------------------- ----- --
B                       1 A,B                      2
..G                     2 A,B,G                    8
C                       1 A,C                      4
..E                     2 A,C,E                    7
....H                   3 A,C,E,H                  8
......J                 4 A,C,E,H,J               11
..G                     2 A,C,G                    8
D                       1 A,D                      3
..E                     2 A,D,E                    7
....H                   3 A,D,E,H                  8
......J                 4 A,D,E,H,J               11
..F                     2 A,D,F                    4
....I                   3 A,D,F,I                  7
......J                 4 A,D,F,I,J               11
..G                     2 A,D,G                    8

15 rows selected.
```

### Solution - one-way, all intermediate records

The output below displays all the intermediate records traversed, with global and iteration-level ranks. It was obtained by dropping the constraints outside the recursive subquery. It may be helpful in understanding how the method works, and we can also get some idea of the efficiency by comparing the number of records returned with the number when we simply traverse all non-looping routes (next section).

Note that there are again no routes ending in a loop. However, we do need to specify the CYCLE keyword because one could arise in the general case. This is a pretty artifical data set.

```
NODE                  LEV PATH                  COST  RNK_T  RNK LP
-------------------- ---- -------------------- ----- ------ ---- --
B                       1 A,B                      2      1    1
..E                     2 A,B,E                    9      3    3
..F                     2 A,B,F                    6      2    2
..G                     2 A,B,G                    8      1    1
....H                   3 A,B,G,H                 11      4    4
....I                   3 A,B,G,I                 11      2    2
C                       1 A,C                      4      1    1
..E                     2 A,C,E                    7      1    1
....H                   3 A,C,E,H                  8      1    1
......J                 4 A,C,E,H,J               11      1    1
....I                   3 A,C,E,I                 11      2    2
..F                     2 A,C,F                    6      2    2
..G                     2 A,C,G                    8      1    1
....H                   3 A,C,G,H                 11      4    4
....I                   3 A,C,G,I                 11      2    2
D                       1 A,D                      3      1    1
..E                     2 A,D,E                    7      1    1
....H                   3 A,D,E,H                  8      1    1
......J                 4 A,D,E,H,J               11      1    1
....I                   3 A,D,E,I                 11      2    2
..F                     2 A,D,F                    4      1    1
....H                   3 A,D,F,H                 10      3    3
....I                   3 A,D,F,I                  7      1    1
......J                 4 A,D,F,I,J               11      1    1
..G                     2 A,D,G                    8      1    1
....H                   3 A,D,G,H                 11      4    4
....I                   3 A,D,G,I                 11      2    2

27 rows selected.
```

### Solution - one-way, all records

The output below displays all the records traversed, when we simply traverse all non-looping routes. There are 48 records, compared with 27 records when we use our analytic pruning technique.

That does not seem a great improvement, but as we'll see when we look at the results for the undirected problem, it's because imposing directionality drastically reduces the number of available routes.

```
NODE                  LEV PATH                  COST  RNK_T  RNK LP
-------------------- ---- -------------------- ----- ------ ---- --
B                       1 A,B                      2      1    1
..E                     2 A,B,E                    9      3    3
....H                   3 A,B,E,H                 10      3    3
......J                 4 A,B,E,H,J               13      4    4
....I                   3 A,B,E,I                 13      9    9
......J                 4 A,B,E,I,J               17     18   18
..F                     2 A,B,F                    6      2    2
....H                   3 A,B,F,H                 12      8    8
......J                 4 A,B,F,H,J               15     11   11
....I                   3 A,B,F,I                  9      2    2
......J                 4 A,B,F,I,J               13      4    4
..G                     2 A,B,G                    8      1    1
....H                   3 A,B,G,H                 11      5    5
......J                 4 A,B,G,H,J               14      8    8
....I                   3 A,B,G,I                 11      4    4
......J                 4 A,B,G,I,J               15     11   11
C                       1 A,C                      4      1    1
..E                     2 A,C,E                    7      1    1
....H                   3 A,C,E,H                  8      1    1
......J                 4 A,C,E,H,J               11      1    1
....I                   3 A,C,E,I                 11      4    4
......J                 4 A,C,E,I,J               15     11   11
..F                     2 A,C,F                    6      2    2
....H                   3 A,C,F,H                 12      8    8
......J                 4 A,C,F,H,J               15     11   11
....I                   3 A,C,F,I                  9      2    2
......J                 4 A,C,F,I,J               13      4    4
..G                     2 A,C,G                    8      1    1
....H                   3 A,C,G,H                 11      5    5
......J                 4 A,C,G,H,J               14      8    8
....I                   3 A,C,G,I                 11      4    4
......J                 4 A,C,G,I,J               15     11   11
D                       1 A,D                      3      1    1
..E                     2 A,D,E                    7      1    1
....H                   3 A,D,E,H                  8      1    1
......J                 4 A,D,E,H,J               11      1    1
....I                   3 A,D,E,I                 11      4    4
......J                 4 A,D,E,I,J               15     11   11
..F                     2 A,D,F                    4      1    1
....H                   3 A,D,F,H                 10      3    3
......J                 4 A,D,F,H,J               13      4    4
....I                   3 A,D,F,I                  7      1    1
......J                 4 A,D,F,I,J               11      1    1
..G                     2 A,D,G                    8      1    1
....H                   3 A,D,G,H                 11      5    5
......J                 4 A,D,G,H,J               14      8    8
....I                   3 A,D,G,I                 11      4    4
......J                 4 A,D,G,I,J               15     11   11

48 rows selected.
```

## Undirected Case

### Solution - two-way from A

```
NODE                       LEV PATH                       COST LP
------------------------- ---- ------------------------- ----- --
B                            1 A,B                           2
..A                          2 A,B,A                         4
..G                          2 A,B,G                         8
C                            1 A,C                           4
..E                          2 A,C,E                         7
....H                        3 A,C,E,H                       8
......J                      4 A,C,E,H,J                    11
..G                          2 A,C,G                         8
D                            1 A,D                           3
..E                          2 A,D,E                         7
....H                        3 A,D,E,H                       8
......J                      4 A,D,E,H,J                    11
..F                          2 A,D,F                         4
....I                        3 A,D,F,I                       7
......J                      4 A,D,F,I,J                    11
..G                          2 A,D,G                         8

16 rows selected.
```

Notice that the solution is identical to that of the directed case, with one interesting exception. The path A,B,A is listed as a route to destination A and it is not regarded by Oracle's recursion algorithm as a loop. That is because the the second node A appears in this path as a destination node for the first time, as the initial A appears only in the path variable.

### Solution - two-way, all intermediate records - breadth first

The previous outputs appear according to the ordering for depth first searching, which as I noted, displays the network structure nicely. However, the following output was obtained using the breadth first search option, and this shows more clearly the way the algorithm works.

<div class="scrollbox">
<pre>
NODE                       LEV PATH                       COST  RNK_T  RNK LP
------------------------- ---- ------------------------- ----- ------ ---- --
B                            1 A,B                           2      1    1
C                            1 A,C                           4      1    1
D                            1 A,D                           3      1    1
..A                          2 A,B,A                         4      1    1
..A                          2 A,D,A                         6      2    2
..A                          2 A,C,A                         8      3    3
..E                          2 A,D,E                         7      1    1
..E                          2 A,C,E                         7      1    1
..E                          2 A,B,E                         9      3    3
..F                          2 A,D,F                         4      1    1
..F                          2 A,C,F                         6      2    2
..F                          2 A,B,F                         6      2    2
..G                          2 A,D,G                         8      1    1
..G                          2 A,B,G                         8      1    1
..G                          2 A,C,G                         8      1    1
....B                        3 A,B,A,B                       6      2    1 *
....B                        3 A,D,F,B                       8      3    2
....B                        3 A,B,G,B                      14      5    3 *
....B                        3 A,D,G,B                      14      5    3
....B                        3 A,D,E,B                      14      5    3
....B                        3 A,C,G,B                      14      5    3
....B                        3 A,C,E,B                      14      5    3
....C                        3 A,D,F,C                       6      2    1
....C                        3 A,B,A,C                       8      3    2
....C                        3 A,D,E,C                      10      4    3
....C                        3 A,C,E,C                      10      4    3 *
....C                        3 A,C,G,C                      12      6    5 *
....C                        3 A,D,G,C                      12      6    5
....C                        3 A,B,G,C                      12      6    5
....D                        3 A,D,F,D                       5      2    1 *
....D                        3 A,B,A,D                       7      3    2
....D                        3 A,D,E,D                      11      4    3 *
....D                        3 A,C,E,D                      11      4    3
....D                        3 A,D,G,D                      13      6    5 *
....D                        3 A,B,G,D                      13      6    5
....D                        3 A,C,G,D                      13      6    5
....H                        3 A,C,E,H                       8      1    1
....H                        3 A,D,E,H                       8      1    1
....H                        3 A,D,F,H                      10      3    3
....H                        3 A,B,G,H                      11      5    4
....H                        3 A,C,G,H                      11      5    4
....H                        3 A,D,G,H                      11      5    4
....I                        3 A,D,F,I                       7      1    1
....I                        3 A,D,G,I                      11      2    2
....I                        3 A,C,E,I                      11      2    2
....I                        3 A,D,E,I                      11      2    2
....I                        3 A,C,G,I                      11      2    2
....I                        3 A,B,G,I                      11      2    2
......A                      4 A,D,F,C,A                    10      4    1
......E                      4 A,D,F,C,E                     9      3    1
......E                      4 A,D,E,H,E                     9      3    1 *
......E                      4 A,C,E,H,E                     9      3    1 *
......E                      4 A,D,F,I,E                    11      7    4
......F                      4 A,D,F,C,F                     8      4    1 *
......F                      4 A,D,F,I,F                    10      5    2 *
......F                      4 A,D,E,H,F                    14      6    3
......F                      4 A,C,E,H,F                    14      6    3
......G                      4 A,D,F,C,G                    10      4    1
......G                      4 A,D,F,I,G                    10      4    1
......G                      4 A,C,E,H,G                    11      6    3
......G                      4 A,D,E,H,G                    11      6    3
......J                      4 A,D,E,H,J                    11      1    1
......J                      4 A,D,F,I,J                    11      1    1
......J                      4 A,C,E,H,J                    11      1    1
........B                    5 A,D,F,C,A,B                  12      4    1
........B                    5 A,D,F,C,G,B                  16     10    2
........B                    5 A,D,F,I,G,B                  16     10    2
........B                    5 A,D,F,C,E,B                  16     10    2
........C                    5 A,D,F,C,E,C                  12      6    1 *
........C                    5 A,D,F,I,G,C                  14     10    2
........C                    5 A,D,F,C,A,C                  14     10    2 *
........C                    5 A,D,F,C,G,C                  14     10    2 *
........D                    5 A,D,F,C,E,D                  13      6    1 *
........D                    5 A,D,F,C,A,D                  13      6    1 *
........D                    5 A,D,F,C,G,D                  15     11    3 *
........D                    5 A,D,F,I,G,D                  15     11    3 *
........H                    5 A,D,F,C,E,H                  10      3    1
........H                    5 A,D,F,C,G,H                  13      8    2
........H                    5 A,D,F,I,G,H                  13      8    2
........H                    5 A,D,E,H,J,H                  14     10    4 *
........H                    5 A,D,F,I,J,H                  14     10    4
........H                    5 A,C,E,H,J,H                  14     10    4 *
........I                    5 A,D,F,I,G,I                  13      7    1 *
........I                    5 A,D,F,C,E,I                  13      7    1
........I                    5 A,D,F,C,G,I                  13      7    1
........I                    5 A,D,E,H,J,I                  15     10    4
........I                    5 A,C,E,H,J,I                  15     10    4
........I                    5 A,D,F,I,J,I                  15     10    4 *
..........A                  6 A,D,F,C,A,B,A                14      5    1 *
..........E                  6 A,D,F,C,E,H,E                11      7    1 *
..........E                  6 A,D,F,C,G,I,E                17      9    2
..........E                  6 A,D,F,C,E,I,E                17      9    2 *
..........E                  6 A,D,F,C,A,B,E                19     11    4
..........F                  6 A,D,F,C,A,B,F                16      8    1 *
..........F                  6 A,D,F,C,E,H,F                16      8    1 *
..........F                  6 A,D,F,C,E,I,F                16      8    1 *
..........F                  6 A,D,F,C,G,I,F                16      8    1 *
..........G                  6 A,D,F,C,E,H,G                13      8    1
..........G                  6 A,D,F,C,E,I,G                16      9    2
..........G                  6 A,D,F,C,G,I,G                16      9    2 *
..........G                  6 A,D,F,C,A,B,G                18     11    4
..........J                  6 A,D,F,C,E,H,J                13      4    1
..........J                  6 A,D,F,C,G,I,J                17      5    2
..........J                  6 A,D,F,C,E,I,J                17      5    2
............B                7 A,D,F,C,E,H,G,B              19     13    1
............C                7 A,D,F,C,E,H,G,C              17     13    1 *
............D                7 A,D,F,C,E,H,G,D              18     13    1 *
............H                7 A,D,F,C,E,H,J,H              16     13    1 *
............H                7 A,D,F,C,E,H,G,H              16     13    1 *
............I                7 A,D,F,C,E,H,G,I              16     13    1
............I                7 A,D,F,C,E,H,J,I              17     14    2
..............A              8 A,D,F,C,E,H,G,B,A            21      6    1
..............E              8 A,D,F,C,E,H,G,I,E            20     12    1 *
..............E              8 A,D,F,C,E,H,G,B,E            26     13    2 *
..............F              8 A,D,F,C,E,H,G,I,F            19     12    1 *
..............F              8 A,D,F,C,E,H,G,B,F            23     13    2 *
..............G              8 A,D,F,C,E,H,G,I,G            19     12    1 *
..............G              8 A,D,F,C,E,H,G,B,G            25     13    2 *
..............J              8 A,D,F,C,E,H,G,I,J            20      7    1
................B            9 A,D,F,C,E,H,G,B,A,B          23     14    1 *
................C            9 A,D,F,C,E,H,G,B,A,C          25     14    1 *
................D            9 A,D,F,C,E,H,G,B,A,D          24     14    1 *
................H            9 A,D,F,C,E,H,G,I,J,H          23     15    1 *
................I            9 A,D,F,C,E,H,G,I,J,I          24     15    1 *

124 rows selected.
</pre>
</div>

We can make a number of observations from this output:

- There are 124 records, which is quite an increase on the number for the directed case (27)
- Looking at the three level two routes to F, via D, C and B respectively, A,D,F is cheaper than the other two, and so in the level 3 records, we see only that sub-route to F and not the other two
- A number of the routes end in a loop, and none of these is passed on to the next level
- If we look at the seven level 3 routes to B, we see that the cheapest is A,B,A,B. This is marked as a cycle because of the second B but was included because the second A did not count as a cycle for the reason given earlier. As this route is the only one ranked 1 at this level to B it causes the elimination of the other six routes, and is itself eliminated for being a loop; this is good because it is obvious that none of them can be part of an optimal higher level route
- We might consider a change in structure to get the source node to count as a loop if it is visited. This can be achieved by anchoring the recursion from a branch that selects the source node from dual. However, when I tried this (not displayed but in the GitHub container subfolder) I found that the number of intermediate solutions actually increased to 128, probably for reasons related to the previous point
- We can see that all of the final solution routes had been obtained by level 4, at 64 records processed, whereas the recursion continues to level 9, at 124 records processed. We might ask whether there is some way of recognising this and terminating the recursion earlier. The answer is that there is not, because it is quite possible that a very cheap route could be composed of a large number of short links - it just happens that our data set does not contain such a route
- A final question that we might ask is how efficient has the query been in relation to the total number of possible routes. We need to again run the query modified to traverse all non-looping routes (noting that Oracle includes loops in the ouptut, but just doesn't progress them). When I did this I got 19281 rows (not displayed but included in the [GitHub container subfolder](https://github.com/BrenPatF/wp_ghp_migration/tree/master/sql-for-shortest-path-problems)). This means the solution algorithm has only traversed 0.6% of the possible routes - a big improvement on the directional case

## Source Changed to J

### Solution - one-way from J

There are no available routes in this case.

### Solution - two-way from J

```
NODE                       LEV PATH                       COST LP
------------------------- ---- ------------------------- ----- --
H                            1 J,H                           3
..E                          2 J,H,E                         4
....B                        3 J,H,E,B                      11
....C                        3 J,H,E,C                       7
......A                      4 J,H,E,C,A                    11
....D                        3 J,H,E,D                       8
......A                      4 J,H,E,D,A                    11
..G                          2 J,H,G                         6
..J                          2 J,H,J                         6
I                            1 J,I                           4
..F                          2 J,I,F                         7
....B                        3 J,I,F,B                      11
....D                        3 J,I,F,D                       8
......A                      4 J,I,F,D,A                    11

14 rows selected.
```

As expected the three optimal routes from A to J when sourcing from A are returned in reverse order when sourcing from J.

## Execution Plan

Here is the execution plan obtained for the undirected solution query, showing the 124 records returned by the recursion reducing to 16 in the final solution.

```
-------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                  | Name    | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                           |         |      1 |        |     16 |00:00:00.01 |      58 |       |       |          |
|   1 |  SORT ORDER BY                             |         |      1 |     20 |     16 |00:00:00.01 |      58 |  2048 |  2048 | 2048  (0)|
|*  2 |   VIEW                                     |         |      1 |     20 |     16 |00:00:00.01 |      58 |       |       |          |
|*  3 |    WINDOW SORT PUSHED RANK                 |         |      1 |     20 |     16 |00:00:00.01 |      58 |  6144 |  6144 | 6144  (0)|
|*  4 |     VIEW                                   |         |      1 |     20 |     58 |00:00:00.01 |      58 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) DEPTH FIRST|         |      1 |        |    124 |00:00:00.01 |      58 | 13312 | 13312 |12288  (0)|
|   6 |       TABLE ACCESS BY INDEX ROWID BATCHED  | ARCS    |      1 |      4 |      3 |00:00:00.01 |       2 |       |       |          |
|*  7 |        INDEX RANGE SCAN                    | ARCS_PK |      1 |      4 |      3 |00:00:00.01 |       1 |       |       |          |
|   8 |       WINDOW SORT                          |         |      8 |     16 |    121 |00:00:00.01 |      56 |  4096 |  4096 | 4096  (0)|
|*  9 |        HASH JOIN                           |         |      8 |     16 |    121 |00:00:00.01 |      56 |  1753K|  1753K| 1185K (0)|
|  10 |         TABLE ACCESS FULL                  | ARCS    |      8 |     40 |    320 |00:00:00.01 |      56 |       |       |          |
|  11 |         RECURSIVE WITH PUMP                |         |      8 |        |     31 |00:00:00.01 |       0 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("RNK_T"=1)
   3 - filter(RANK() OVER ( PARTITION BY "NODE" ORDER BY "COST")<=1)
   4 - filter("RNK"=1)
   7 - access("A"."SRC"=:SRC)
   9 - access("A"."SRC"="P"."NODE")
```

Oracle version:

Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options

[Code, in GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/sql-for-shortest-path-problems)
