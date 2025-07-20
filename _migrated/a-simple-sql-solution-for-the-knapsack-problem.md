---
layout: post
title: "A Simple SQL Solution for the Knapsack Problem (SKP-1)"
date: 2013-01-06
migrated: true
categories: 
  - "oracle"
  - "performance"
  - "pipelined"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "benchmarking"
  - "performance-2"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

A poster on OTN ([Combination using pl/sql](https://forums.oracle.com/ords/apexds/post/combination-using-pl-sql-8780)) recently asked for an SQL solution to a problem that turned out to be an example of the well known [Knapsack Problem](http://en.wikipedia.org/wiki/Knapsack_problem), for the case of a single knapsack. I posted an SQL query as a solution, and also a solution in PL/SQL because the SQL solution uses a feature only available in Oracle v11.2. In this article I explain how the solutions work and provide the results of a performance analysis that involved randomised test problems of varying computational difficulty. I have taken a more general form of problem than the original poster described, and the solutions here have been improved.

**Update, 14 July 2013**: I used the technique in response to another OTN post here, [SQL for the Fantasy Football Knapsack Problem](http://aprogrammerwrites.eu/?p=p=878). I have extended the idea there to allow for fast approximate solutions making it viable for larger problems, and have also used a similar idea here, [SQL for the Travelling Salesman Problem](https://brenpatf.github.io/migrated/sql-for-the-travelling-salesman-problem) (and in other articles).

**Update, 26 November 2017:** My GitHub repo: [Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos) has simple installation and query scripts for this problem.

## Knapsack Problem (1-Knapsack)

The various forms of knapsack problem have been studied extensively. The problems are known to be computationally difficult and many algorithms have been proposed for both exact and approximate solutions (see reference above). The SQL solution in this article is quite simple and will not be competitive in performance for larger problems in the form described here, but may be interesting for being implemented in pure SQL (and without using Oracle's Model clause, or a purely brute force approach). However, I have later extended the approach to allow for search limiting and have shown this to be viable for larger problems (see links at the top).

The problem can be stated informally, as follows: _Given a set of items, each having positive weight and profit attributes, and a weight limit, find the combinations of items that maximise profit within the weight limit._ Variant versions include the addition of multiple constraints (easy to handle), and inclusion of multiple knapsacks (more difficult). I also have a solution for the multiple knapsacks version described here ([An SQL Solution for the Multiple Knapsack Problem (SKP-m)](https://brenpatf.github.io/migrated/an-sql-solution-for-the-multiple-knapsack-problem-skp-m/)).

The difficulty of the problem arises from the number of possible combinations increasing exponentially with problem size. The number of these (not necessarily feasible) combinations, N(n,1), can be expressed in terms of the number of items, n, in two ways. First, we can use the well known binomial expression for the number of combinations of r items, summed from r = 0 to r =n :

<div class="eq_indent">
    <img src="/migrated_images/2013/01/CodeCogsEqn_pack_1.png" />
</div>
<br />
Second, and more simply, we can observe that including an item in the combination, or not, is a binary choice, leading to:

<div class="eq_indent">
    <img src="/migrated_images/2013/01/CodeCogsEqn_pack_2.png" />
</div>
<br />
This generalises easily to the expression for the multiple knapsack problem, with m knapsacks:

<div class="eq_indent">
    <img src="/migrated_images/2013/01/CodeCogsEqn_pack_3.png" />
</div>
<br />
This can also be expressed using a binomial series as

<div class="eq_indent">
    <img src="/migrated_images/2013/01/CodeCogsEqn_pack_4.png" />
</div>
<br />
Here, <img src="/migrated_images/2013/01/CodeCogsEqn_pack_5.png" /> represents the number of combinations of r items from n, with <img src="/migrated_images/2013/01/CodeCogsEqn_pack_6.png" /> being the number of assignments of the r items to m containers.

Let's look at a simple example problem having four items, with a weight limit of 9, as shown below:

<img src="/migrated_images/2013/01/Packing-v1.3-Items.jpg" alt="Items" title="Items" />

There are 16 \[N(4,1) = 2\*\*4\] possible combinations of these items, having from 0 to 4 items. These are depicted below:

<img src="/migrated_images/2013/01/Packing-v1.3-Combis.jpg" alt="Combinations" title="Combinations" />

We can see that there are two optimal solutions in this case. How to find them using SQL?

## SQL Solution

Oracle's v11.2 implementation of the Ansi standard Recursive Subquery Factoring can be used as the basis for an SQL solution. This would works as follows: _Starting from each item in turn, add items recursively while remaining within the weight limit, and considering only items of id greater than the current id._ The SQL looks like this, where a marker is added for leaf nodes, following an approach from the [Amis technology blog](http://technology.amis.nl/2009/11/14/oracle-11gr2-alternative-for-connect_by_isleaf-function-for-recursive-subquery-factoring-dedicated-to-anton/):

```sql
WITH rsf (nxt_id, lev, tot_weight, tot_profit, path) AS (
SELECT id nxt_id, 0 lev, item_weight tot_weight, item_profit tot_profit, To_Char (id) path
  FROM items
 UNION ALL
SELECT n.id, 
       r.lev + 1, 
       r.tot_weight + n.item_weight,
       r.tot_profit + n.item_profit,
       r.path || ',' || To_Char (n.id)
  FROM rsf r
  JOIN items n
    ON n.id > r.nxt_id
   AND r.tot_weight + n.item_weight <= 9
) SEARCH DEPTH FIRST BY nxt_id SET line_no 
SELECT LPad (To_Char(nxt_id), lev + 1, '*') node,tot_weight, tot_profit,
       CASE WHEN lev >= Lead (lev, 1, lev) OVER (ORDER BY line_no) THEN 'Y' END is_leaf,
       path
  FROM rsf
 ORDER BY line_no

```

and the solution like this:

```
NODE       TOT_WEIGHT TOT_PROFIT I PATH
---------- ---------- ---------- - ------------------------------
1                   3         10   1
*2                  7         30 Y 1,2
*3                  8         40 Y 1,3
*4                  9         50 Y 1,4
2                   4         20   2
*3                  9         50 Y 2,3
3                   5         30 Y 3
4                   6         40 Y 4

8 rows selected.
```

The output contains 8 records, as opposed to the total of 15 non-null combinations, because only feasible items are joined, and permutations are avoided by the constraint that item ids increase along the path. Given positivity of weight and profit, we know that all solutions must be leaves, and we can represent the tree structure above in the following diagram: 

<img src="/migrated_images/2013/01/Packing-v1.3-Leaves2.jpg" alt="Leaves" title="Leaves" />

We can now use the recursive subquery factor as an input to a main query that selects one of the most profitable solutions, or alternatively to a further subquery factor that ranks the solutions in order of descending profit. In the latter case, the main query can select all the most profitable solutions.

In the solution I posted on the OTN thread, I included a subquery factor to restrict the final query section to leaf nodes only. This was because we know that the solutions must be leaf nodes, and usually it is more efficient to filter out non-solution records as early as possible. However, I later realised that the work involved in the filtering might outweigh the saving for the final section, and this turned out to be the case here, as shown in the performance analysis section below. Here are the two queries, without the leaf node filtering:

### Query - KEEP

```sql
WITH rsf (id, lev, tot_weight, tot_profit, path) AS (
SELECT id, 0, item_weight, item_profit, To_Char (id)
  FROM items
 UNION ALL
SELECT n.id, 
       r.lev + 1, 
       r.tot_weight + n.item_weight,
       r.tot_profit + n.item_profit,
       r.path || ',' || To_Char (n.id)
  FROM rsf r
  JOIN items n
    ON n.id > r.id
   AND r.tot_weight + n.item_weight <= 100
)
SELECT Max (tot_weight) KEEP (DENSE_RANK LAST ORDER BY tot_profit) tot_weight,
       Max (tot_profit) KEEP (DENSE_RANK LAST ORDER BY tot_profit) tot_profit,
       Max (path) KEEP (DENSE_RANK LAST ORDER BY tot_profit) path,
       (Max (lev) KEEP (DENSE_RANK LAST ORDER BY tot_profit) + 1) n_items
  FROM rsf
```

### Query - RANK

```sql
WITH rsf (id, lev, tot_weight, tot_profit, path) AS (
SELECT id, 0, item_weight, item_profit, To_Char (id)
  FROM items
 UNION ALL
SELECT n.id, 
       r.lev + 1, 
       r.tot_weight + n.item_weight,
       r.tot_profit + n.item_profit,
       r.path || ',' || To_Char (n.id)
  FROM rsf r
  JOIN items n
    ON n.id > r.id
   AND r.tot_weight + n.item_weight <= 100
)
, paths_ranked AS (
SELECT tot_weight, tot_profit, path, 
       Dense_Rank () OVER (ORDER BY tot_profit DESC) rnk_profit,
       lev
  FROM rsf
)
SELECT tot_weight tot_weight, 
       tot_profit tot_profit, 
       path path, 
       (lev + 1) n_items
  FROM paths_ranked
 WHERE rnk_profit = 1
 ORDER BY tot_weight DESC
```

### Query Structure Diagram

<img src="/migrated_images/2013/01/Packing-v1.3-QSD-Noleaf.jpg" alt="QSD" title="QSD" />

It's worth noting that Oracle's proprietary recursive syntax, Connect By, cannot be used in this way because of the need to accumulate weights forward through the recursion. The new Ansi syntax is only available from v11.2 though, and I thought it might be interesting to implement a solution in PL/SQL that would work in earlier versions, following a similar algorithm, again with recursion.

## PL/SQL Recursive Solution

This is a version in the form of a pipelined function, as I wanted to compare it with the SQL solutions, and be callable from SQL.

### SQL

```
SELECT COLUMN_VALUE sol
  FROM TABLE (Packing_PLF.Best_Fits (100))
 ORDER BY COLUMN_VALUE
```

### Package

<div class="scrollbox">
<pre>
CREATE OR REPLACE PACKAGE BODY Packing_PLF IS

FUNCTION Best_Fits (p_weight_limit NUMBER) RETURN SYS.ODCIVarchar2List PIPELINED IS
  TYPE item_type IS RECORD (
                        item_id                 PLS_INTEGER,
                        item_index_parent       PLS_INTEGER,
                        weight_to_node          NUMBER);
  TYPE item_tree_type IS        TABLE OF item_type;
  g_solution_list               SYS.ODCINumberList;
  g_timer                       PLS_INTEGER := Timer_Set.Construct ('Pipelined Recursion');

  i                             PLS_INTEGER := 0;
  j                             PLS_INTEGER := 0;
  g_item_tree                   item_tree_type;
  g_item                        item_type;
  l_weight                      PLS_INTEGER;
  l_weight_new                  PLS_INTEGER;
  l_best_profit                 PLS_INTEGER := -1;
  l_sol                         VARCHAR2(4000);
  l_sol_cnt                     PLS_INTEGER := 0;

  FUNCTION Add_Node (  p_item_id               PLS_INTEGER,
                       p_item_index_parent     PLS_INTEGER, 
                       p_weight_to_node        NUMBER) RETURN PLS_INTEGER IS
  BEGIN
    g_item.item_id := p_item_id;
    g_item.item_index_parent := p_item_index_parent;
    g_item.weight_to_node := p_weight_to_node;
    IF g_item_tree IS NULL THEN
      g_item_tree := item_tree_type (g_item);
    ELSE
      g_item_tree.Extend;
      g_item_tree (g_item_tree.COUNT) := g_item;
    END IF;
    RETURN g_item_tree.COUNT;
  END Add_Node;

  PROCEDURE Do_One_Level (p_tree_index PLS_INTEGER, p_item_id PLS_INTEGER, p_tot_weight PLS_INTEGER, p_tot_profit PLS_INTEGER) IS
    CURSOR c_nxt IS
    SELECT id, item_weight, item_profit
      FROM items
     WHERE id > p_item_id
       AND item_weight + p_tot_weight <= p_weight_limit;
    l_is_leaf           BOOLEAN := TRUE;
    l_index_list        SYS.ODCINumberList;
  BEGIN
    FOR r_nxt IN c_nxt LOOP
      Timer_Set.Increment_Time (g_timer,  'Do_One_Level/r_nxt');
      l_is_leaf := FALSE;
      Do_One_Level (Add_Node (r_nxt.id, p_tree_index, r_nxt.item_weight + p_tot_weight), r_nxt.id, p_tot_weight + r_nxt.item_weight, p_tot_profit + r_nxt.item_profit);
      Timer_Set.Increment_Time (g_timer,  'Do_One_Level/Do_One_Level');
    END LOOP;
    IF l_is_leaf THEN
      IF p_tot_profit > l_best_profit THEN
        g_solution_list := SYS.ODCINumberList (p_tree_index);
        l_best_profit := p_tot_profit;
      ELSIF p_tot_profit = l_best_profit THEN
        g_solution_list.Extend;
        g_solution_list (g_solution_list.COUNT) := p_tree_index;
      END IF;
    END IF;
    Timer_Set.Increment_Time (g_timer,  'Do_One_Level/leaves');
  END Do_One_Level;

BEGIN
  FOR r_itm IN (SELECT id, item_weight, item_profit FROM items) LOOP
    Timer_Set.Increment_Time (g_timer,  'Root fetches');
    Do_One_Level (Add_Node (r_itm.id, 0, r_itm.item_weight), r_itm.id, r_itm.item_weight, r_itm.item_profit);
  END LOOP;
  FOR i IN 1..g_solution_list.COUNT LOOP
    j := g_solution_list(i);
    l_sol := NULL;
    l_weight := g_item_tree (j).weight_to_node;
    WHILE j != 0 LOOP
      l_sol := l_sol || g_item_tree (j).item_id || ', ';
      j :=  g_item_tree (j).item_index_parent;
    END LOOP;
    l_sol_cnt := l_sol_cnt + 1;
    PIPE ROW ('Solution ' || l_sol_cnt || ' (profit ' || l_best_profit || ', weight ' || l_weight || ') : ' || RTrim (l_sol, ', '));
  END LOOP;
  Timer_Set.Increment_Time (g_timer,  'Write output');
  Timer_Set.Write_Times (g_timer);
EXCEPTION
  WHEN OTHERS THEN
    Timer_Set.Write_Times (g_timer);
    RAISE;
END Best_Fits;

END Packing_PLF;
</pre>
</div>

## Performance Analysis

It will be interesting to see how the solution methods perform as problem size varies, and we will use my own performance benchmarking framework to do this. As the framework is designed to compare performance of SQL queries, I have converted the PL/SQL solution to operate as a pipelined function, and thus be callable from SQL, as noted above. I included a version of the SQL solution, with the leaf filtering mentioned above, XKPLV - this was based on XKEEP, with filtering as in the OTN thread.

I presented on this approach to benchmarking SQL at the Ireland Oracle User Group conference in March 2017, [Dimensional Performance Benchmarking of SQL – IOUG Presentation](http://aprogrammerwrites.eu/?p=2012).

### Test Data Sets

Test data sets are generated as follows, in terms of two integer parameters, _w_ and _d_:

Insert _w_ items with sequential ids, and random weights and profits in the ranges 0-_d_ and 0-1000, respectively, via Oracle's function DBMS\_Random.Value. The maximum weight is fixed at 100.

### Test Results


The embedded Excel file below summarises the results obtained over a grid of data points, with _w_ in (12, 14, 16, 18, 20) and _d_ in (16, 18, 20).

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21519&amp;authkey=AHzGSHotB6qFv30&amp;em=2" width="600" height="600" frameborder="0" scrolling="no"></iframe>

### Notes

- The two versions of the non-leaf SQL solution take pretty much the same time to execute, and are faster than the others
- The leaf version of the SQL solution (XKPLV) is slower than the non-leaf versions, and becomes much worse in terms of elapsed time for the more difficult problems; the step-change in performance can be seen to be due to its greater memory usage, which spills to disk above a certain level
- The pipelined function solution is significantly slower than the other solutions in terms of CPU time, and elapsed time, except in the case of the leaf SQL solution when that solution's memory usage spills to disk. The pipelined function continues to use more memory as the problem difficulty rises, until all available memory is consumed, when it throws an error (but this case is not included in the result set above)

## Conclusion

Oracle's v11.2 implementation of the Ansi SQL feature recursive subquery factoring provides a simple solution for the knapsack problem, that cannot be achieved with Oracle's older Connect By syntax alone.

The method has been described here in its exact form that is viable only for small problems; however, I have later extended the approach to allow for search limiting and have shown this to be viable for larger problems.
