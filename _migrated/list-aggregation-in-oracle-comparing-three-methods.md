---
layout: post
title: "List Aggregation in Oracle - Comparing Three Methods"
date: 2012-09-30
migrated: true
categories: 
  - "data-model"
  - "erd"
  - "model"
  - "oracle"
  - "performance"
  - "pipelined"
  - "sql"
tags: 
  - "benchmarking"
  - "cpu"
  - "erd"
  - "oracle"
  - "performance-2"
  - "qsd"
  - "sql"
---

In my last article, [Grouping by Unique Subsequences in Oracle](https://brenpatf.github.io/migrated/grouping-by-unique-subsequences-in-sql/), I compared three solutions to a querying problem in Oracle. I found that a solution using a pipelined function was fastest across a range of test data sets, while another using Oracle’s Model clause turned out to be extremely inefficient, and very unscaleable owing to quadratic variation in execution times.

I mentioned in the article the issue of possible poor cardinality estimates by Oracle’s Cost-Based Optimiser (CBO) for pipelined functions, referencing an article by Adrian Billington, [setting cardinality for pipelined and table functions](http://www.oracle-developer.net/display.php?id=427 "http://www.oracle-developer.net/display.php?id=427"), that considers four techniques for improving these estimates.

In this article, I want to use another, very common, querying problem, to see in more detail how one of these techniques works, namely the dynamic sampling hint, and to compare the performance again of pipelined functions against the Model clause on a second example amenable to both methods.

In previous articles I have generally focussed on elapsed and CPU times to measure performance, but Oracle provides a range of metrics in instrumenting query execution. My benchmarking framework captures many of these, and we'll use them to try to understand the performance variation, again using a 2-dimensional domain of test data. We also capture the execution plans used across the domain and display visually the changes across the domain.

The problem is that of list aggregation, and various solution methods are available depending on one’s Oracle version. In Oracle v11.2 a specific built-in function has been provided, Listagg, and Adrian Billington has compared it with earlier SQL techniques ([listagg function in 11g release 2](http://www.oracle-developer.net/display.php?id=515 "http://www.oracle-developer.net/display.php?id=515")), including a Model solution. He looks only at pure SQL solutions, but we will take both the Model and Listagg solutions, and add in a pipelined function solution. We’ll take a simple test problem using Oracle’s demo HR schema, which will be: Return, for each department, its id, manager name, and a comma-separated, ordered list of its employee names.

## Test Data

 The HR tables employees and departments were copied structurally to test versions with \_t suffixes and records were inserted programmatically by the performance testing packages.

### ERD

<img src="/migrated_images/2012/09/Stragg-V1.0-ERD.jpg" alt="Stragg, V1.0 - ERD" title="Stragg, V1.0 - ERD" />

## Listagg Solution

### How It Works

The first solution for this problem uses the aggregation function ListAgg, which is new in Oracle v11.2. The query groups employees by department, joins departments to get the name, and employees again to get the manager’s name.

### Query Diagram

 <img src="/migrated_images/2012/09/Stragg-V1.0-Listagg.jpg" alt="Stragg, V1.0 - Listagg" title="Stragg, V1.0 - Listagg" />

### SQL


```sql
SELECT e.department_id,
       m.last_name manager,
       ListAgg (e.last_name, ',') WITHIN GROUP (ORDER BY e.last_name) emp_names
  FROM employees_t e
  JOIN departments_t d
    ON d.department_id = e.department_id
  JOIN employees_t m
    ON m.employee_id = d.manager_id
 GROUP BY 
       e.department_id,
       m.last_name
 ORDER BY e.department_id
```

## Model Solution

### How It Works


1. Within an inline view, form the basic Select, with the department\_id column, and append placeholders for the employee list and row number
2. Add the Model keyword, partitioning by department\_id, dimensioning by analytic function Row\_Number, ordering by name within department, with name and name list as measures
3. Define the only rule to prepend the list from the next element with the current name, going backwards (so the first record will be the one you want)
4. Join the other tables to the inline view in the main query, strip off the last ‘,’, and filter out all except the first record for the department

### Query Diagram
 <img src="/migrated_images/2012/09/Stragg-V1.0-Model.jpg" alt="Stragg, V1.0 - Model" title="Stragg, V1.0 - Model" />

### SQL


```sql
SELECT v.department_id, 
       e.last_name manager,
       RTrim (v.emp_names, ',') emp_names  
  FROM (
SELECT department_id,
       emp_names,
       rn
  FROM employees_t 
    MODEL  
      PARTITION BY (department_id)  
      DIMENSION BY (Row_Number() OVER 
                       (PARTITION BY department_id ORDER BY last_name) rn)
      MEASURES (last_name, CAST(NULL AS VARCHAR2(4000)) emp_names) 
      RULES (
        emp_names[ANY] ORDER BY rn DESC = last_name[CV()] || ',' || emp_names[CV()+1] 
      ) 
) v
  JOIN departments_t d
    ON d.department_id = v.department_id
  JOIN employees_t e
    ON e.employee_id = d.manager_id
 WHERE v.rn = 1 
 ORDER BY v.department_id
```

## Pipelined Function Solution

### How It Works

This approach is based on pipelined database functions, which are specified to return array types. Pipelining means that Oracle transparently returns the records in batches while processing continues, thus avoiding memory problems, and returning initial rows more quickly. Within the function there is a simple cursor loop over the employees, joining departments to get the manager id. A string variable accumulates the list of employees, until the department changes, when the record is piped out, and the string reset to the new employee. The last record has to be piped after exiting the loop.

### Types

Two database types are specified, the first being an object with fields for the department and manager ids and the employee name list; the second is an array of the nested table form with elements of the first type.

### Function Pseudocode


```
Loop over a cursor selecting the records in order 
    If the department changes or first record then
        If the department changes then
            Pipe the row out
        End if
        Reset variables to current record values
    Else
        Append the current employee name to the name list
    End if
End loop
If the last department is not null then
    Pipe the row out using saved values
End if
```

### SQL

Just select the fields named in the record from the function wrapped in the TABLE keyword, and join employees to get the manager name.

### Query Diagram

<img src="/migrated_images/2012/09/Stragg-V1.0-Function.jpg" alt="Stragg, V1.0 - Function" title="Stragg, V1.0 - Function" />

### Function Definition (within package)


```sql
FUNCTION Dep_Emps RETURN dep_emps_list_type PIPELINED IS

  l_emp_names	        VARCHAR2(4000);
  old_manager_id        PLS_INTEGER;
  old_department_id     PLS_INTEGER;

BEGIN

  FOR r_val IN (SELECT e.department_id, d.manager_id, e.last_name 
                  FROM employees_t e
                  JOIN departments_t d
                    ON d.department_id = e.department_id
                 ORDER BY e.department_id, e.last_name) LOOP

    IF r_val.department_id != old_department_id OR old_department_id IS NULL THEN
      IF r_val.department_id != old_department_id THEN
        PIPE ROW (dep_emps_type (r_val.department_id, r_val.manager_id, l_emp_names));
      END IF;
      old_department_id := r_val.department_id;
      old_manager_id := r_val.manager_id;
      l_emp_names := r_val.last_name;
    ELSE
      l_emp_names := l_emp_names || ',' || r_val.last_name;
    END IF;

  END LOOP;
  IF old_department_id IS NOT NULL THEN
    PIPE ROW (dep_emps_type (old_department_id, old_manager_id, l_emp_names));
  END IF;

END Dep_Emps;
```

### SQL

```sql
SELECT d.department_id,
       e.last_name manager,
       d.emp_names
  FROM TABLE (Stragg.Dep_Emps) d
  JOIN employees_t e
    ON e.employee_id = d.manager_id
 ORDER BY d.department_id
```

## Performance Analysis

 As in the previous article, I have benchmarked across a 2-dimensional domain, in this case width being the number of employees per department, and depth the number of departments. For simplicity, a single department, ‘Accounting’, and employee, ‘John Chen’, were used as templates and inserted repeatedly with suffixes on names and new ids.

The three queries above were run on all data points, and in addition the function query was run with a hint: DYNAMIC\_SAMPLING (d 5).

### Record Counts (total employees and employees per department)
 
<iframe width="800" height="250" frameborder="0" scrolling="no" src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICVjAAAAAAAAJR4sIH7WGHBq1Y?em=2&wdAllowInteractivity=False&wdHideGridlines=True&wdHideHeaders=True&wdDownloadButton=True&wdInConfigurator=True&wdInConfigurator=True"></iframe>

### Timings

<iframe width="800" height="450" frameborder="0" scrolling="no" src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICViQAAAAAAAGGFr70NRUIGvrk?em=2&wdAllowInteractivity=False&wdHideGridlines=True&wdHideHeaders=True&wdDownloadButton=True&wdInConfigurator=True&wdInConfigurator=True"></iframe>

In the embedded Excel file above, the four solutions are labelled as follows:

- F = Pipelined Function solution
- D = Pipelined Function with Dynamic Sampling hint solution
- L = Listagg solution
- M = Model solution

For the data points, font colour and fill colour signify:

- Fill colours correspond to distinct execution plans whose hash values can be found later in the same tab. The formatted outputs can be found in the Plans tab for all distinct plans for selected data points (with hyperlinks)
- White font signifies that the smallest elapsed time for the given data point occurred for the given solution, for all the tables except CPU time
- For the CPU time table higher in the tab, white font signifies that the smallest CPU time for the given data point occurred for the given solution
- Red font indicates that the solution incurred more than 500 disk reads (see the disk reads table later in the tab)

### Graphs
 
<iframe width="800" height="450" frameborder="0" scrolling="no" src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICVigAAAAAAANPJ91OBdGLR88Y?em=2&wdAllowInteractivity=False&wdHideGridlines=True&wdHideHeaders=True&wdDownloadButton=True&wdInConfigurator=True&wdInConfigurator=True"></iframe>

### Comparison

 The CPU time for all four queries increases approximately in proportion with either width or depth dimension when the other is fixed, which is not surprising. The elapsed times are very similar to the CPU times for all except the larger problems using Model, which we’ll discuss in a later section. For the largest data point, the times rank in the following order: 

<iframe width="800" height="190" frameborder="0" scrolling="no" src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICViwAAAAAAALZcY8nbPuiUbnU?em=2&wdAllowInteractivity=False&wdHideGridlines=True&wdHideHeaders=True&wdDownloadButton=True&wdInConfigurator=True&wdInConfigurator=True"></iframe>

The following points can be made:

- The function query without the dynamic sampling hint was fastest for the largest data point, and by a significant margin over the next best, Listagg
- The earlier detailed tables show that this was also true for the triangle of data points starting three points back in each dimension, indicating that this is the case once the problem size gets large enough
- Similarly, the function query with dynamic sampling was in third place for all these larger problems
- Model was always the slowest query, generally by a factor of about 8 over the fastest in terms of CPU time, but very much worse in elapsed time for problems above a certain size, and we’ll look at this in more detail next.

### Model Performance Discontinuity

In Adrian Billington’s article mentioned above ([listagg function in 11g release 2](http://www.oracle-developer.net/display.php?id=515 "http://www.oracle-developer.net/display.php?id=515")), he took a single large-ish data point for his problem and found that his Model query took 308 seconds compared with 6 seconds for Listagg. He puts the Model performance down to '_an enormous number of direct path reads/writes to/from the temporary tablespace_'. In my last article, I also quoted the anonymous author of [MODEL Performance Tuning](http://www.sqlsnippets.com/en/topic-12100.html "http://www.sqlsnippets.com/en/topic-12100.html"): _'In some cases MODEL query performance can even be so poor that it renders the query unusable. One of the keys to writing efficient and scalable MODEL queries seems to be keeping session memory use to a minimum'_.

My benchmarking framework records the metrics from the view v$sql\_plan\_stats\_all, which is used by DBMS\_XPlan.Display\_Cursor to write the execution plan, and prints to a CSV file aggregates over the plan of several of them, for example: Max (last\_disk\_reads). These are are shown for the Model solution in the tab displayed below of the embedded Excel file (the next tab has 3-d graphs but they may not display in a browser).

The tables show that memory increases with the problem size in each direction up to a maximum, at which points the number of disk reads jumps from a very low level and then rises with problem size. The maximum memory points have red font. These transitions represent discontinuities where there is a jump in the elapsed times, although CPU times continue to rise smoothly. So while Model is always slower than the other solutions, its much greater use of memory causes the discrepancy to increase dramatically when the processing spills to disk, which is consistent with, and extends, the observations of the authors mentioned. 

<iframe width="800" height="500" frameborder="0" scrolling="no" src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICVjQAAAAAAAKuMwVYHQDLyliY?em=2&wdAllowInteractivity=False&wdHideGridlines=True&wdHideHeaders=True&wdDownloadButton=True&wdInConfigurator=True&wdInConfigurator=True"></iframe>

### Model and Listagg Cardinality Estimates

Here is the execution plan for the last data point:

```
----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation               | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |               |      1 |        |   1024 |00:01:25.46 |    5798 |    164K|    164K|       |       |          |         |
|   1 |  SORT ORDER BY          |               |      1 |    260K|   1024 |00:01:25.46 |    5798 |    164K|    164K|  2320K|   704K| 2062K (0)|         |
|*  2 |   HASH JOIN             |               |      1 |    260K|   1024 |00:01:20.86 |    5798 |    164K|    164K|   909K|   909K| 1230K (0)|         |
|*  3 |    HASH JOIN            |               |      1 |   1024 |   1024 |00:00:00.11 |    2901 |      0 |      0 |   935K|   935K| 1228K (0)|         |
|   4 |     TABLE ACCESS FULL   | DEPARTMENTS_T |      1 |   1024 |   1024 |00:00:00.01 |       6 |      0 |      0 |       |       |          |         |
|   5 |     TABLE ACCESS FULL   | EMPLOYEES_T   |      1 |    260K|    262K|00:00:00.40 |    2895 |      0 |      0 |       |       |          |         |
|*  6 |    VIEW                 |               |      1 |    260K|   1024 |00:01:20.68 |    2897 |    164K|    164K|       |       |          |         |
|   7 |     BUFFER SORT         |               |      1 |    260K|    262K|00:01:19.56 |    2897 |    164K|    164K|   297M|  5726K|   45M (0)|     265K|
|   8 |      SQL MODEL ORDERED  |               |      1 |    260K|    262K|01:28:30.86 |    2895 |    131K|    131K|  1044M|    28M|   51M (1)|         |
|   9 |       WINDOW SORT       |               |      1 |    260K|    262K|00:00:01.01 |    2895 |      0 |      0 |  7140K|  1067K| 6346K (0)|         |
|  10 |        TABLE ACCESS FULL| EMPLOYEES_T   |      1 |    260K|    262K|00:00:00.50 |    2895 |      0 |      0 |       |       |          |         |
----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("D"."DEPARTMENT_ID"="V"."DEPARTMENT_ID")
   3 - access("E"."EMPLOYEE_ID"="D"."MANAGER_ID")
   6 - filter("V"."RN"=1)
```

Notice that at line 6 the cardinality estimate is 260K while the actual rows returned was 1024: The CBO has simply ignored the filtering down to department level by row number, and assumed the number of employees! We may ask whether this mis-estimate has affected the subsequent plan. Well the result set at line 6 makes the second step in a hash join to another row set of actual cardinality 1024, so probably it has not made a significant difference in this case. It’s worth comparing the execution plan for Listagg:

```
---------------------------------------------------------------------------------------------------------------------------
| Id  | Operation            | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |               |      1 |        |   1024 |00:00:02.11 |    5796 |       |       |          |
|   1 |  SORT GROUP BY       |               |      1 |    185K|   1024 |00:00:02.11 |    5796 |    15M|  1999K|   14M (0)|
|*  2 |   HASH JOIN          |               |      1 |    260K|    262K|00:00:02.17 |    5796 |   921K|   921K| 1195K (0)|
|*  3 |    HASH JOIN         |               |      1 |   1024 |   1024 |00:00:00.11 |    2901 |   935K|   935K| 1229K (0)|
|   4 |     TABLE ACCESS FULL| DEPARTMENTS_T |      1 |   1024 |   1024 |00:00:00.01 |       6 |       |       |          |
|   5 |     TABLE ACCESS FULL| EMPLOYEES_T   |      1 |    260K|    262K|00:00:00.41 |    2895 |       |       |          |
|   6 |    TABLE ACCESS FULL | EMPLOYEES_T   |      1 |    260K|    262K|00:00:00.46 |    2895 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("D"."DEPARTMENT_ID"="E"."DEPARTMENT_ID")
   3 - access("M"."EMPLOYEE_ID"="D"."MANAGER_ID")
```

Notice that the cardinality estimates are accurate apart from at line 1 for the Sort Group By, where a similar error has been made in not allowing for the reduction in rows caused by the grouping. Again it doesn’t seem to have affected the plan adversely.

### Memory Usage and Buffers

A few observations can be made about these statistics:

- Buffers is sometimes seen as a good, fundamental measure of performance, but we can see that both function solutions have figures of about 9K, while the other two have very similar figures of about 6K, and these do not correlate well with actual time performance here
- The buffers figure for Model at the minimum data point is more than half that for the maximum, unlike the other solutions where it is 1-3%. The ratio of records is only 0.2%, so this is hard to understand
- Similarly, the memory usage for Model starts very high, 3.1M, before rising to its maximum of 54M. For pipelined functions the figures go from 6K to 2.8M, and for Listagg from 29K to 15M

### Dynamic Sampling Effects

Here is the execution plan for the pipelined function solution at point (W256-D512), with my own timing output from 'Timer Set...' on:

```
----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name        | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |             |      1 |        |    512 |00:00:00.65 |    6097 |       |       |          |
|   1 |  SORT ORDER BY                      |             |      1 |   8168 |    512 |00:00:00.65 |    6097 |  1186K|   567K| 1054K (0)|
|*  2 |   HASH JOIN                         |             |      1 |   8168 |    512 |00:00:00.62 |    6097 |  1520K|   901K| 1706K (0)|
|   3 |    COLLECTION ITERATOR PICKLER FETCH| DEP_EMPS    |      1 |   8168 |    512 |00:00:00.54 |    3202 |       |       |          |
|   4 |    TABLE ACCESS FULL                | EMPLOYEES_T |      1 |    130K|    131K|00:00:00.20 |    2895 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("E"."EMPLOYEE_ID"=VALUE(KOKBF$))

Timer Set: Cursor, Constructed at 23 Sep 2012 07:49:43, written at 07:49:45
===========================================================================
[Timer timed: Elapsed (per call): 0.05 (0.000045), CPU (per call): 0.04 (0.000040), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed          CPU          Calls        Ela/Call        CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Open cursor               0.01         0.00              1         0.01000         0.00000
First fetch               0.65         0.63              1         0.64700         0.63000
Write to file             0.06         0.06              2         0.03050         0.03000
Remaining fetches         0.00         0.00              1         0.00000         0.00000
Write plan                0.64         0.61              1         0.64200         0.61000
(Other)                   0.11         0.05              1         0.11400         0.05000
----------------- ---------- ---------- ------------ ------------- -------------
Total                     1.47         1.35              7         0.21057         0.19286
----------------- ---------- ---------- ------------ ------------- -------------
```

Here is the output for the pipelined function solution with dynamic sampling at the same point (W256-D512):

```
-----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name        | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |             |      1 |        |    512 |00:00:00.59 |    4228 |       |       |          |
|   1 |  SORT ORDER BY                       |             |      1 |    512 |    512 |00:00:00.59 |    4228 |  1186K|   567K| 1054K (0)|
|   2 |   NESTED LOOPS                       |             |      1 |        |    512 |00:00:00.59 |    4228 |       |       |          |
|   3 |    NESTED LOOPS                      |             |      1 |    512 |    512 |00:00:00.58 |    3716 |       |       |          |
|   4 |     COLLECTION ITERATOR PICKLER FETCH| DEP_EMPS    |      1 |    512 |    512 |00:00:00.56 |    3202 |       |       |          |
|*  5 |     INDEX UNIQUE SCAN                | EMP_PK      |    512 |      1 |    512 |00:00:00.01 |     514 |       |       |          |
|   6 |    TABLE ACCESS BY INDEX ROWID       | EMPLOYEES_T |    512 |      1 |    512 |00:00:00.01 |     512 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - access("E"."EMPLOYEE_ID"=VALUE(KOKBF$))

Note
-----
   - dynamic sampling used for this statement (level=2)

Timer Set: Cursor, Constructed at 23 Sep 2012 07:49:45, written at 07:49:47
===========================================================================
[Timer timed: Elapsed (per call): 0.05 (0.000045), CPU (per call): 0.05 (0.000050), calls: 1000, '***' denotes corrected line below]

Timer                  Elapsed          CPU          Calls        Ela/Call        CPU/Call
----------------- ---------- ---------- ------------ ------------- -------------
Open cursor               0.55         0.55              1         0.54500         0.55000
First fetch               0.59         0.57              1         0.59300         0.57000
Write to file             0.06         0.06              2         0.03100         0.03000
Remaining fetches         0.00         0.00              1         0.00000         0.00000
Write plan                0.59         0.58              1         0.59300         0.58000
(Other)                   0.06         0.03              1         0.05800         0.03000
----------------- ---------- ---------- ------------ ------------- -------------
Total                     1.85         1.79              7         0.26443         0.25571
----------------- ---------- ---------- ------------ ------------- -------------
```

Notice that in the plan without dynamic sampling the cardinality estimate for the row set returned from the function is 8168, nearly 8 times the actual rows. When dynamic sampling is added the cardinality estimate is exactly right. Also the times reported in the plan are similar but show dynamic sampling to be faster. How can that be, when the table of results earlier showed the query with dynamic sampling taking nearly twice as long?

My framework breaks down the times for the steps in running the query and writing the results out. From these we see that the difference is largely accounted for by the function query taking only 0.01 seconds to open the cursor while with dynamic sampling it takes .55 seconds. Evidently the dynamic sampling hint is causing a call to the function at the query parsing stage which is not accounted for in the execution plan statistics. (Notice incidentally that Oracle is apparaently performing a deferred open in both cases, where the actual cursor opening is performed at the time of first fetching.)

If one looks at the colour-coded table of elapsed times above, it can be seen that dynamic sampling causes a single plan to be chosen for all data points with depth below 1024, while without the hint two plans are chosen that with dynamic sampling are chosen only at depth 1024. It can also be seen that the dynamic sampling version is faster overall in most cases, but at the highest depth is slower because the non-hinted plan is the same. The default cardinality estimate (8168) was too high until that point.

## Conclusion

 We have applied three techniques for list aggregation to one specific problem, and for that problem the following concluding remarks can be made:

- The Model solution is much inferior in performance to both the new Listagg native function, and custom pipelined function solutions
- The variation in performance has been analysed and for the model solution shown to become dramatically worse when problem size causes earlier in-memory processing to spill to disk
- The dynamic sampling hint has been applied to the pipelined function solution and shown to give correct cardinality estimates. In our example, the performance effect was negative for the largest problem sizes owing to the overhead involved, but in other cases it was positive
- We have oberved that both Model and Listagg solutions also have cardinality estimation problems, but have not analysed these further
- The native Listagg solution is significantly, and surprisingly, slower than the custom pipelined function solution at the largest data points considered

We may say more generally:

- Performance analysis across a 2-dimensional domain of data sets can provide more insight than just looking at one large zero-dimensional case
- Microsoft Excel graphs and other functionality have proved very useful in helping to visualise what is happening in terms of performance
- Our findings tend to bear out a common suspicion of Model clause on performance grounds, and further support the view that pipelined functions can be very performance-effective, used appropriately
