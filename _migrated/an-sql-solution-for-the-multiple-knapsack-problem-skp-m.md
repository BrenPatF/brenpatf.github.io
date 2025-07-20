---
layout: post
title: "An SQL Solution for the Multiple Knapsack Problem (SKP-m)"
date: 2013-01-20
migrated: true
categories: 
  - "analytics"
  - "oracle"
  - "performance"
  - "pipelined"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "benchmarking"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

In my last article, [A Simple SQL Solution for the Knapsack Problem (SKP-1)](https://brenpatf.github.io/migrated/a-simple-sql-solution-for-the-knapsack-problem/), I presented an SQL solution for the well known knapsack problem in its simpler 1-knapsack form (and it is advisable to read the first article before this one). Here I present an SQL solution for the problem in its more difficult multiple-knapsack form. The solution is a modified version of one I posted on OTN, [SQL Query for mapping a set of batches to a class rooms group](https://forums.oracle.com/ords/apexds/post/sql-query-for-mapping-a-set-of-batches-to-a-class-rooms-gro-3778), and I describe two versions of it, one in pure SQL, and another that includes a database function. The earlier article provided the solutions as comma-separated strings of item identifiers, and in this article also the solutions are first obtained as delimited strings. However, as there are now containers as well as items, we extend the SQL to provide solutions with item and container names in separate fields within records for each container-item pair. The solution was presented here initially, as before, more for its theoretical interest than for practical applicability. Much research has been done on procedural algorithms for this important, but computationally difficult class of problems.

**Update, 26 November 2017:** My GitHub repo: [Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos) has simple installation and query scripts for this problem. I should also note that after this post I went on to use similar techniques on other combinatorial problems, such as [SQL for the Fantasy Football Knapsack Problem](http://aprogrammerwrites.eu/?p=p=878); I extended the idea there to allow for fast approximate solutions making it viable for larger problems, and have also used a similar idea here, [SQL for the Travelling Salesman Problem](http://aprogrammerwrites.eu/?p=p=896) (and in other articles).

We will consider the same simple example problem as in the earlier article, having four items, but now with two containers with individual weight limits of 8 and 10. As noted in the earlier article, the problem can be considered as that of assigning each item to one of the containers, or to none, leading directly to the expression <img src="/migrated_images/2013/01/CodeCogsEqn_multi_1.png" /> for the number of not necessarily feasible assignment sets for the example. We can again depict the 16 possible item combinations in a diagram, with the container limits added.

<img src="/migrated_images/2013/01/Multi-v1.1-Combis.jpg" /> 

There are 81 assignments of 4 items to 2 containers, disregarding the capacity limits \[N(4,2) = 3\*\*4\].

From the diagram we see that the first 7 combinations meet the container-1 limit of 8, while the first 10 meet the container-2 limit of 10. The task is to pick a pair of disjoint combinations from these sets that maximises profit. We can see that there is one optimal solution in this case, in which items 1 and 3 are assigned to container 1, while items 2 and 4 are assigned to container 2, with a profit of 100. How to find it using SQL?

## SQL Solution

The solution to the single knapsack problem worked by joining items recursively in increasing order of item id, accumulating the total weights and profits, and terminating a sequence when no more items can be added within the weight limit. The item sequences were accumulated as comma-separated strings, and the optimal solutions obtained by analytic ranking of the profits.

For the multiple knapsack problem, it's not quite as simple, but a similar approach may be a good starting point. Previously our anchor branch in the recursion selected all items below the single maximum weight, but we now have containers with individual weights. If we now join the containers table we can find all items falling within the maximum weights by container. The recursion can then proceed to find all feasible item combinations by container. Here is the SQL for this:

```sql
WITH rsf_itm (con_id, max_weight, itm_id, lev, tot_weight, tot_profit, path) AS (
SELECT c.id, 
       c.max_weight,
       i.id,
       0,
       i.item_weight,
       i.item_profit, 
       ',' || i.id || ','
  FROM items i
  JOIN containers c
    ON i.item_weight <= c.max_weight
 UNION ALL
SELECT r.con_id,
       r.max_weight,
       i.id, 
       r.lev + 1, 
       r.tot_weight + i.item_weight,
       r.tot_profit + i.item_profit,
       r.path || i.id || ','
  FROM rsf_itm r
  JOIN items i
    ON i.id > r.itm_id
   AND r.tot_weight + i.item_weight <= r.max_weight
 ORDER BY 1, 2
) SEARCH DEPTH FIRST BY con_id, itm_id SET line_no
SELECT con_id,
       max_weight,
       LPad (To_Char(itm_id), 2*lev + 1, ' ') itm_id, 
       path itm_path,
       tot_weight, tot_profit
  FROM rsf_itm
 ORDER BY line_no
```

and here is the resulting output:

```
CON_ID MAX_WEIGHT ITM_ID ITM_PATH    TOT_WEIGHT TOT_PROFIT
------ ---------- ------ ----------- ---------- ----------
     1          8 1      ,1,                  3         10
                    2    ,1,2,                7         30
                    3    ,1,3,                8         40
                  2      ,2,                  4         20
                  3      ,3,                  5         30
                  4      ,4,                  6         40
     2         10 1      ,1,                  3         10
                    2    ,1,2,                7         30
                    3    ,1,3,                8         40
                    4    ,1,4,                9         50
                  2      ,2,                  4         20
                    3    ,2,3,                9         50
                    4    ,2,4,               10         60
                  3      ,3,                  5         30
                  4      ,4,                  6         40

15 rows selected.
```

Looking at this, we can see that the overall solution will comprise one feasible combination of items for each container, with the constraint that no item appears in more than one container. This suggests that we could perform a second recursion in a similar way to the first, but this time using the results of the first as input, and joining the feasible combinations of containers of higher id only. If we again accumulate the sequence in a delimited string, regular expression functionality could be used to avoid joining combinations with items already included. The following SQL does this recursion:

```sql
WITH rsf_itm (con_id, max_weight, itm_id, tot_weight, tot_profit, path) AS (
SELECT c.id, 
       c.max_weight,
       i.id, 
       i.item_weight,
       i.item_profit, 
       ',' || i.id || ','
  FROM items i
  JOIN containers c
    ON i.item_weight <= c.max_weight
 UNION ALL
SELECT r.con_id,
       r.max_weight,
       i.id, 
       r.tot_weight + i.item_weight,
       r.tot_profit + i.item_profit,
       r.path || i.id || ','
  FROM rsf_itm r
  JOIN items i
    ON i.id > r.itm_id
   AND r.tot_weight + i.item_weight <= r.max_weight
)
, rsf_con (con_id, con_itm_set, con_itm_path, lev, tot_weight, tot_profit) AS (
SELECT con_id,
       ':' || con_id || ':' || path,
       ':' || con_id || ':' || path,
       0,
       tot_weight,
       tot_profit
  FROM rsf_itm
 UNION ALL
SELECT r_i.con_id,
       ':' || r_i.con_id || ':' || r_i.path,
       r_c.con_itm_path ||  ':' || r_i.con_id || ':' || r_i.path,
       r_c.lev + 1, 
       r_c.tot_weight + r_i.tot_weight,
       r_c.tot_profit + r_i.tot_profit
  FROM rsf_con r_c
  JOIN rsf_itm r_i
    ON r_i.con_id > r_c.con_id
 WHERE RegExp_Instr (r_c.con_itm_path || r_i.path, ',(\d+),.*?,\1,') = 0
) SEARCH DEPTH FIRST BY con_id SET line_no
SELECT
       LPad (' ', 2*lev, ' ') || con_itm_set con_itm_set,
       con_itm_path,
       tot_weight, tot_profit
  FROM rsf_con
 ORDER BY line_no
```

Notice the use of RegExp\_Instr, which takes the current sequence with potential new combination appended as its source string, and looks for a match against the search string ',(\\d+),.\*?,\\1,'. The function returns 0 if no match is found, meaning no duplicate item was found. The sequence includes the container id using a different delimiter, a colon, at the start of each combination. The search string can be explained as follows:

,(\\d+), = a sequence of one or more digits with a comma either side, and the digit sequence saved for referencing .\*?,\\1, = a sequence of any characters, followed by the saved digit sequence within commas. The ? specifies a non-greedy search, meaning stop searching as soon as a match is found

The result of the query is:

```
CON_ITM_SET          CON_ITM_PATH         TOT_WEIGHT TOT_PROFIT
-------------------- -------------------- ---------- ----------
:1:,1,               :1:,1,                        3         10
  :2:,2,             :1:,1,:2:,2,                  7         30
  :2:,3,             :1:,1,:2:,3,                  8         40
  :2:,4,             :1:,1,:2:,4,                  9         50
  :2:,2,3,           :1:,1,:2:,2,3,               12         60
  :2:,2,4,           :1:,1,:2:,2,4,               13         70
:1:,2,               :1:,2,                        4         20
  :2:,1,             :1:,2,:2:,1,                  7         30
  :2:,3,             :1:,2,:2:,3,                  9         50
  :2:,1,3,           :1:,2,:2:,1,3,               12         60
  :2:,4,             :1:,2,:2:,4,                 10         60
  :2:,1,4,           :1:,2,:2:,1,4,               13         70
:1:,1,2,             :1:,1,2,                      7         30
  :2:,3,             :1:,1,2,:2:,3,               12         60
  :2:,4,             :1:,1,2,:2:,4,               13         70
:1:,3,               :1:,3,                        5         30
  :2:,1,             :1:,3,:2:,1,                  8         40
  :2:,2,             :1:,3,:2:,2,                  9         50
  :2:,1,2,           :1:,3,:2:,1,2,               12         60
  :2:,4,             :1:,3,:2:,4,                 11         70
  :2:,1,4,           :1:,3,:2:,1,4,               14         80
  :2:,2,4,           :1:,3,:2:,2,4,               15         90
:1:,1,3,             :1:,1,3,                      8         40
  :2:,2,             :1:,1,3,:2:,2,               12         60
  :2:,4,             :1:,1,3,:2:,4,               14         80
  :2:,2,4,           :1:,1,3,:2:,2,4,             18        100
:1:,4,               :1:,4,                        6         40
  :2:,1,             :1:,4,:2:,1,                  9         50
  :2:,2,             :1:,4,:2:,2,                 10         60
  :2:,1,2,           :1:,4,:2:,1,2,               13         70
  :2:,3,             :1:,4,:2:,3,                 11         70
  :2:,1,3,           :1:,4,:2:,1,3,               14         80
  :2:,2,3,           :1:,4,:2:,2,3,               15         90
:2:,1,               :2:,1,                        3         10
:2:,2,               :2:,2,                        4         20
:2:,1,2,             :2:,1,2,                      7         30
:2:,3,               :2:,3,                        5         30
:2:,1,3,             :2:,1,3,                      8         40
:2:,4,               :2:,4,                        6         40
:2:,1,4,             :2:,1,4,                      9         50
:2:,2,3,             :2:,2,3,                      9         50
:2:,2,4,             :2:,2,4,                     10         60

42 rows selected.
```

We can see that the optimal solutions can be obtained from the output again using analytic ranking by profit, and in this case the solution with a profit of 100 is the optimal one, with sequence ':1:,1,3,:2:,2,4,'. In the full solution, as well as selecting out the top-ranking solutions, we have extended the query to output the items and containers by name, in distinct fields with a record for every solution/container/item combination. For the example problem above, the output is:

```
    SOL_ID S_WT  S_PR  C_ID C_NAME          M_WT C_WT  I_ID I_NAME     I_WT I_PR
---------- ---- ----- ----- --------------- ---- ---- ----- ---------- ---- ----
         1   18   100     1 Item 1             8    8     1 Item 1        3   10
                                                          3 Item 3        5   30
                          2 Item 2            10   10     2 Item 2        4   20
                                                          4 Item 4        6   40
```

### SQL-Only Solution - XSQL

There are various techniques in SQL for splitting string columns into multiple rows and columns. We will take one of the more straightforward ones that uses the DUAL table with CONNECT BY to generate rows against which to anchor the string-parsing.

<div class="scrollbox">

<pre>
WITH rsf_itm (con_id, max_weight, itm_id, lev, tot_weight, tot_profit, path) AS (
SELECT c.id, 
       c.max_weight,
       i.id, 
       0, 
       i.item_weight,
       i.item_profit, 
       ',' || i.id || ','
  FROM items i
  JOIN containers c
    ON i.item_weight <= c.max_weight
 UNION ALL
SELECT r.con_id,
       r.max_weight,
       i.id, 
       r.lev + 1, 
       r.tot_weight + i.item_weight,
       r.tot_profit + i.item_profit,
       r.path || i.id || ','
  FROM rsf_itm r
  JOIN items i
    ON i.id > r.itm_id
   AND r.tot_weight + i.item_weight <= r.max_weight
)
, rsf_con (con_id, con_path, itm_path, tot_weight, tot_profit, lev) AS (
SELECT con_id,
       To_Char(con_id),
       ':' || con_id || '-' || (lev + 1) || ':' || path,
       tot_weight,
       tot_profit,
       0
  FROM rsf_itm
 UNION ALL
SELECT r_i.con_id,
       r_c.con_path || ',' || r_i.con_id,
       r_c.itm_path ||  ':' || r_i.con_id || '-' || (r_i.lev + 1) || ':' || r_i.path,
       r_c.tot_weight + r_i.tot_weight,
       r_c.tot_profit + r_i.tot_profit,
       r_c.lev + 1
  FROM rsf_con r_c
  JOIN rsf_itm r_i
    ON r_i.con_id > r_c.con_id
   AND RegExp_Instr (r_c.itm_path || r_i.path, ',(\d+),.*?,\1,') = 0
)
, paths_ranked AS (
SELECT itm_path || ':' itm_path, tot_weight, tot_profit, lev + 1 n_cons,
       Rank () OVER (ORDER BY tot_profit DESC) rnk
  FROM rsf_con
), best_paths AS (
SELECT itm_path, tot_weight, tot_profit, n_cons,
       Row_Number () OVER (ORDER BY tot_weight DESC) sol_id
  FROM paths_ranked
 WHERE rnk = 1
), row_gen AS (
SELECT LEVEL lev
  FROM DUAL
CONNECT BY LEVEL <= (SELECT Count(*) FROM items)
), con_v AS (
SELECT  r.lev con_ind, b.sol_id, b.tot_weight, b.tot_profit,
        Substr (b.itm_path, Instr (b.itm_path, ':', 1, 2*r.lev - 1) + 1, 
                            Instr (b.itm_path, ':', 1, 2*r.lev) - Instr (b.itm_path, ':', 1, 2*r.lev - 1) - 1)
           con_nit_id,
        Substr (b.itm_path, Instr (b.itm_path, ':', 1, 2*r.lev) + 1, 
                            Instr (b.itm_path, ':', 1, 2*r.lev + 1) - Instr (b.itm_path, ':', 1, 2*r.lev) - 1)
           itm_str
  FROM best_paths b
  JOIN row_gen r
    ON r.lev <= b.n_cons
), con_split AS (
SELECT sol_id, tot_weight, tot_profit,
       Substr (con_nit_id, 1, Instr (con_nit_id, '-', 1) - 1) con_id,
       Substr (con_nit_id, Instr (con_nit_id, '-', 1) + 1) n_items,
       itm_str
  FROM con_v
), itm_v AS (
SELECT  c.sol_id, c.con_id, c.tot_weight, c.tot_profit,
        Substr (c.itm_str, Instr (c.itm_str, ',', 1, r.lev) + 1, 
                            Instr (c.itm_str, ',', 1, r.lev + 1) - Instr (c.itm_str, ',', 1, r.lev) - 1)
           itm_id
  FROM con_split c
  JOIN row_gen r
    ON r.lev <= c.n_items
)
SELECT v.sol_id sol_id,
       v.tot_weight s_wt, 
       v.tot_profit s_pr, 
       c.id c_id, 
       c.name c_name, 
       c.max_weight m_wt,
       Sum (i.item_weight) OVER (PARTITION BY v.sol_id, c.id) c_wt,
       i.id i_id, 
       i.name i_name, 
       i.item_weight i_wt, 
       i.item_profit i_pr
  FROM itm_v v
  JOIN containers c
    ON c.id = To_Number (v.con_id)
  JOIN items i
    ON i.id = To_Number (v.itm_id)
 ORDER BY sol_id, con_id, itm_id
</pre>
</div>

### SQL with Function Solution - XFUN

The SQL techniques for string-splitting are quite cumbersome, and a better approach may be the use of a pipelined function that allows the string-parsing to be done in PL/SQL, a procedural language that is better suited to the task.

<div class="scrollbox">

<pre>
WITH rsf_itm (con_id, max_weight, itm_id, tot_weight, tot_profit, path) AS (
SELECT c.id, 
       c.max_weight,
       i.id, 
       i.item_weight,
       i.item_profit, 
       ',' || i.id || ','
  FROM items i
  JOIN containers c
    ON i.item_weight <= c.max_weight
 UNION ALL
SELECT r.con_id,
       r.max_weight,
       i.id, 
       r.tot_weight + i.item_weight,
       r.tot_profit + i.item_profit,
       r.path || i.id || ','
  FROM rsf_itm r
  JOIN items i
    ON i.id > r.itm_id
   AND r.tot_weight + i.item_weight <= r.max_weight
 ORDER BY 1, 2
)
, rsf_con (con_id, itm_path, tot_weight, tot_profit) AS (
SELECT con_id,
       ':' || con_id || ':' || path,
       tot_weight,
       tot_profit
  FROM rsf_itm
 UNION ALL
SELECT r_i.con_id,
       r_c.itm_path ||  ':' || r_i.con_id || ':' || r_i.path,
       r_c.tot_weight + r_i.tot_weight,
       r_c.tot_profit + r_i.tot_profit
  FROM rsf_con r_c
  JOIN rsf_itm r_i
    ON r_i.con_id > r_c.con_id
   AND RegExp_Instr (r_c.itm_path || r_i.path, ',(\d+),.*?,\1,') = 0
)
, paths_ranked AS (
SELECT itm_path || ':' itm_path, tot_weight, tot_profit, Rank () OVER (ORDER BY tot_profit DESC) rn,
       Row_Number () OVER (ORDER BY tot_profit DESC, tot_weight DESC) sol_id
  FROM rsf_con
), itm_v AS (
SELECT s.con_id, s.itm_id, p.itm_path, p.tot_weight, p.tot_profit, p.sol_id
  FROM paths_ranked p
 CROSS JOIN TABLE (Multi.Split_String (p.itm_path)) s
 WHERE rn = 1
)
SELECT v.sol_id sol_id,
       v.tot_weight s_wt, 
       v.tot_profit s_pr, 
       c.id c_id, 
       c.name c_name, 
       c.max_weight m_wt,
       Sum (i.item_weight) OVER (PARTITION BY v.sol_id, c.id) c_wt,
       i.id i_id, 
       i.name i_name, 
       i.item_weight i_wt, 
       i.item_profit i_pr
  FROM itm_v v
  JOIN containers c
    ON c.id = To_Number (v.con_id)
  JOIN items i
    ON i.id = To_Number (v.itm_id)
 ORDER BY sol_id, con_id, itm_id
</pre>
</div>

### Pipelined Database Function

```sql
CREATE OR REPLACE TYPE con_itm_type AS OBJECT (con_id NUMBER, itm_id NUMBER);
/
CREATE OR REPLACE TYPE con_itm_list_type AS VARRAY(100) OF con_itm_type;
/
CREATE OR REPLACE PACKAGE BODY Multi IS

FUNCTION Split_String (p_string VARCHAR2) RETURN con_itm_list_type PIPELINED IS

  l_pos_colon_1           PLS_INTEGER := 1;
  l_pos_colon_2           PLS_INTEGER;
  l_pos_comma_1           PLS_INTEGER;
  l_pos_comma_2           PLS_INTEGER;
  l_con                   PLS_INTEGER;
  l_itm                   PLS_INTEGER;
BEGIN
  LOOP
    l_pos_colon_2 := Instr (p_string, ':', l_pos_colon_1 + 1, 1);
    EXIT WHEN l_pos_colon_2 = 0;
    l_con := To_Number (Substr (p_string, l_pos_colon_1 + 1, l_pos_colon_2 - l_pos_colon_1 - 1));
    l_pos_colon_1 := Instr (p_string, ':', l_pos_colon_2 + 1, 1);
    l_pos_comma_1 := l_pos_colon_2 + 1;
    LOOP
      l_pos_comma_2 := Instr (p_string, ',', l_pos_comma_1 + 1, 1);
      EXIT WHEN l_pos_comma_2 = 0 OR l_pos_comma_2 > l_pos_colon_1;
      l_itm := To_Number (Substr (p_string, l_pos_comma_1 + 1, l_pos_comma_2 - l_pos_comma_1 - 1));
      PIPE ROW (con_itm_type (l_con, l_itm));
      l_pos_comma_1 := l_pos_comma_2;
    END LOOP;
  END LOOP;

END Split_String;

END Multi;
```

### Query Structure Diagram (embedded directly)

The QSD shows both queries in a single diagram as the early query blocks are almost the same (the main difference is that the strings contain a bit more information for XSQL to facilitate the later splitting). The directly-embedded version shows the whole query, but it may be hard to read the detail, so it is followed by a larger, scrollable version within Excel.

<img src="/migrated_images/2013/01/Multi-v1.1-QSD.jpg" alt="QSD showing both versions of SQL" title="QSD showing both versions of SQL" />

### Query Structure Diagram (embedded via Excel)
 
This is the larger, scrollable version.

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21522&amp;authkey=AGOpKbJtjEmkwFs&amp;em=2" width="800" height="1500" frameborder="0" scrolling="no"></iframe>

## Performance Analysis

As in the previous article, we will see how the solution methods perform as problem size varies, using my own performance benchmarking framework.

I presented on this approach to benchmarking SQL at the Ireland Oracle User Group conference in March 2017, [Dimensional Performance Benchmarking of SQL – IOUG Presentation](http://aprogrammerwrites.eu/?p=2012).

### Test Data Sets

Test data sets are generated as follows, in terms of two integer parameters, _w_ and _d_:

- Insert _w_ containers with sequential ids and random maximum weights between 1 and 100
- Insert _d_ items with sequential ids and random weights and profits in the ranges 1-60 and 1-10000, respectively, via Oracle's function DBMS\_Random.Value

### Test Results

The embedded Excel file below summarises the results obtained over a grid of data points, with _w_ in (1, 2, 3) and _d_ in (8, 10, 12, 14, 16, 18).

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21525&amp;authkey=AJd-3CQVA_VgMNI&amp;em=2" width="800" height="750" frameborder="0" scrolling="no"></iframe>

The graphs tab below shows 3-d graphs of the number of rows processed and the CPU time for XFUN.

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21524&amp;authkey=ADohGLQUKiFrqnA&amp;em=2" width="800" height="750" frameborder="0" scrolling="no"></iframe>

### Notes

- There is not much difference in performance between the two query versions, no doubt because the number of solution records is generally small compared with rows processed in the recursions
- Notice that the timings correlate well with the rows processed, but not so well with the numbers of base records. The nature of the problem means that some of the randomised data sets turn out to be much harder to solve than others
- Notice the estimated rows on step 36 of the execution plan for the pipelined function solution. The value of 8168 is a fixed value that Oracle assumes since it has no statistics to go on. We could improve this by using the (undocumented) cardinality hint to provide a smaller estimate
- I extended my benchmarking framework for this article to report the intermediate numbers of rows processed, as well as the cardinality estimates and derived errors in these estimates (maximum for each plan). It is obvious from the nature of the problem that Oracle's Cost Based Optimiser (CBO) is not going to be able to make good cardinality estimates

## Conclusion

Oracle's v11.2 implementation of the Ansii SQL feature recursive subquery factoring provides a means for solving the knapsack problem, in its multiple knapsack form, in SQL. The solution is not practical for large problems, for which procedural techniques that have been extensively researched should be considered. However, the techniques used may be of interest for combinatorial problems that are small enough to be handled in SQL, and for other types of problem in general.
