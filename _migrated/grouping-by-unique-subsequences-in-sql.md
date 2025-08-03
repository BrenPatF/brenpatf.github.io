---
layout: post
title: "Grouping by Unique Subsequences in SQL"
date: 2012-09-08
migrated: true
group: general-sql
categories: 
  - "erd"
  - "model"
  - "oracle"
  - "performance"
  - "pipelined"
  - "qsd"
  - "sql"
tags: 
  - "cpu"
  - "model-2"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "qsd"
  - "sql"
---

Recently a deceptively simple question was asked on Oracle’s OTN PL/SQL forum that most posters initially misunderstood, myself included ([Grouping Non Consecutive Rows](https://forums.oracle.com/ords/apexds/post/grouping-non-consecutive-rows-4325)). The question requested a solution for the problem of assigning group numbers to a sequence of records such that each group held no duplicate values of a particular field, and each group was as long as possible. One of the posters produced a solution involving the Model clause and I tried to find a solution without using that clause, just to see if it could be done. Having found such a solution, not without some difficulty, I then solved the problem via a pipelined PL/SQL function having felt that that would be more efficient than straight SQL.

I think all three solutions have their own interest, especially since forum posters frequently assert, incorrectly, that SQL solutions are always faster than those using PL/SQL. I’ll explain how each solution works in this article and will run them through my benchmarking framework. We’ll see that the PL/SQL solution time increases linearly with sequence length, while the others increase quadratically, and PL/SQL is orders of magnitude faster for even small problem sizes. I used Oracle Database v11.2 XE running on a Samsung X120 dedicated PC.

I use my [Query Structure Diagramming](https://brenpatf.github.io/migrated/query-structure-diagramming-two-examples/) technique to illustrate the SQL solutions.

## Functional Test Data

The problem data structure is quite simple, and I have defined my own version to emphasise generality, and to include partitioning, which was not part of the initial posting. There is one table with a two-column primary key, and I added an index on the partition field plus the value field.

### Table Value\_Subseqs

| Column | Type |
| --- | --- |
| part\_key\* | Char(10) |
| ind\* | Number |
| val | Char(10) |

### Index VSB\_N1

- part\_key
- val

### Test Cases

Functional testing was done on a number of simple sequences, including the sequence of single characters, ABABB, which is used to illustrate the first solution query.

## SQL Pairs Solution

### How It Works

The first solution for this problem starts by obtaining the possible start and end points of the groups: all pairs that contain no duplicate. In the table below, the rows represent the possible pairs with X marking each value in the ranges. It’s easy to see the solution for this small test problem and the correct pairs are highlighted in yellow. 

<img src="/migrated_images/2012/09/sql_pairs_matrix.png" alt="SQL pairs matrix" title="SQL pairs matrix" />

From the table it’s possible to work out a solution in SQL, given the pair set. The first correct pair is the longest starting from 1, while each subsequent correct pair is the longest beginning one element after the previous ends. This is of course a kind of tree-walk, which leads to the following solution steps (where everything is implicitly partitioned):

1. Within a subquery factor, self-join the table with the second record having a higher ind than the first
2. Add a NOT EXISTS subquery that checks for duplicate val in any pair between the outer potantial start and end point pair, using another self-join
3. Add in the single element pairs via a UNION
4. Within another subquery factor, obtain the rank of each pair in terms of its length descending, as well as the starting ind value (if it might not always be 1)
5. In another subquery factor perform the tree-walk as mentioned, using the condition ‘rank = 1’ to include only the longest ranges from any point
6. Include the the analytic function Row\_Number in the Select list, which will be the group number
7. The main query selects from the last subquery factor, then joins the original table

### Notes
 The query is of course likely to be performance-intensive owing to the initial subquery factor with its nested self-joins across ranges.

### Query Diagram

<img src="/migrated_images/2012/09/Subsequences-V1.0-Pairs.jpg" alt="Subsequences, V1.0 - Pairs" title="Subsequences, V1.0 - Pairs" />

The diagram notation follows and extends notation developed earlier, including my previous blog article, and is intended to be largely self-explanatory. In this diagram, I have added a new notation for joins that are not simple foreign key joins, in which I label the join and use a note to explain it. Oracle v11.2 introduced Recursive Subquery Factors that extend tree-walk functionality. I used the older Connect By syntax in the query since it works on older versions, but found it easier to represent in the diagram as though it were the newer implementation – the diagram shows what’s happening logically.

### SQL

```sql
WITH sqf AS (
SELECT p1.part_key, p1.ind ind_1, p2.ind ind_2
  FROM value_subseqs p1
  JOIN value_subseqs p2
    ON p2.ind           > p1.ind
   AND p2.part_key      = p1.part_key
   AND NOT EXISTS (SELECT 1
                     FROM value_subseqs p3
                     JOIN value_subseqs p4
                       ON p4.val        = p3.val
                      AND p4.part_key   = p3.part_key
                    WHERE p3.ind        BETWEEN p1.ind AND p2.ind
                      AND p4.ind        BETWEEN p3.ind + 1 AND p2.ind
                      AND p3.part_key   = p1.part_key
                      AND p4.part_key   = p2.part_key)
 UNION ALL
SELECT part_key, ind, ind
  FROM value_subseqs p1
), rnk AS (
SELECT part_key, ind_1, ind_2,
        Row_Number() OVER (PARTITION BY part_key, ind_1 ORDER BY ind_2 - ind_1 DESC) rn,
        Min(ind_1) OVER (PARTITION BY part_key) ind_1_beg
  FROM sqf
), grp AS (
SELECT part_key, ind_1, ind_2, Row_Number() OVER
       (PARTITION BY part_key ORDER BY ind_1, ind_2) grp_no
  FROM rnk
CONNECT BY ind_1 = PRIOR ind_2 + 1
   AND part_key = PRIOR part_key
   AND rn = 1
 START WITH ind_1 = ind_1_beg AND rn = 1
)
SELECT
    p.part_key,
    p.ind,
    p.val,
    g.grp_no
  FROM grp g
  JOIN value_subseqs p
    ON p.ind            BETWEEN g.ind_1 AND g.ind_2
   AND p.part_key       = g.part_key
 ORDER BY p.part_key, p.ind
```

## Model Solution


### How It Works
 In the OTN thread several solutions were proposed that used Oracle’s Model clause or recursive subquery factoring, but I think only one was a general solution, since the others used string variables to successively concatenate strings over arbitrary numbers of rows and would break when the 4000 character SQL limit is hit. The general Model solution (which was not by me, but I’ve reformatted it and applied it to my table) worked by defining two measures, for group start indices and group number. The rules specified two passes for the two measures: The first pass counts the distinct values and compares with the count of all values, between the previous group start and the current index; the second uses the group starts to set the group numbers.

1. Form the basic Select, with all the table columns required, and append a group number placeholder 
2. Add the Model keyword, partitioning by part\_key, dimensioning by analytic function Row\_Number, ordering by ind within part\_key, with val, group start and group number as measures
3. Initialise group start and group number to 1 in the measures clause
4. Define the first rule to obtain the group start date for all rows after the first as the previous group start, unless there is a difference between the two counts, in which case take the new index.
5. Define the second rule to obtain the group number for all rows as the previous group number, unless the group start has changed, in which case take the previous group number + 1.

### Query Diagram
 <img src="/migrated_images/2012/09/Subsequences-V1.0-Model.jpg" alt="Subsequences, V1.0 - Model" title="Subsequences, V1.0 - Model" />

- Partition - processing is to be performed separately by one or more columns; the same meaning as in analytic functions
- Dimension - columns by which the array is dimensioned; can included analytic functions, as here
- Measures - remaining columns that may be calculated or updated by the rules, possibly including placeholders from the main query
- Rules - a set of rules that specify measure calculation; rules are processed sequentially, unless otherwise specified; in the diagram:
    - n - the current dimension value
    - F(n-1,n) - denotes that the value depends on values from previous and current rows (and so on, ‘..’ denotes a range)
    - ^ - denotes that the calculation progresses in ascending order by dimension; this is the default so does not have to be coded

### SQL

```sql
SELECT
    part_key,
    rn,
    val,
    g grp_no
  FROM value_subseqs
 MODEL
   PARTITION BY (part_key)
   DIMENSION BY (Row_number() OVER (PARTITION BY part_key ORDER BY ind) rn)
   MEASURES (val, 1 g, 1 s)
   RULES (
     s[rn > 1] = CASE COUNT (DISTINCT val)[rn BETWEEN s[CV() - 1] AND CV()]
                   WHEN COUNT (val)[rn BETWEEN s[CV() - 1] AND cv()] THEN s[cv() - 1]
                   ELSE CV(rn)
                 END,
     g[rn > 1] = CASE s[CV()]
                    WHEN s[CV() - 1] THEN g[CV() - 1]
                    ELSE g[CV() - 1] + 1
                 END
   )
 ORDER BY part_key, rn
```

## Pipelined Function Hash Solution


### How It Works

This approach is based on pipelined database functions, which are specified to return array types. Pipelining means that Oracle transparently returns the records in batches while processing continues, thus avoiding memory problems, and returning initial rows more quickly. See [Pipelined Functions (AskTom)](http://asktom.oracle.com/pls/asktom/f?p=100:11:::::P11_QUESTION_ID:19481671347143 "Pipelined Functions") for some other examples of use. Within the function there is a simple cursor loop over the ordered sequence. A PL/SQL index-by array stores values for the current group, and allows duplicate checking to take place without any additional searching or sorting. The array is reset whenever the group changes.

### Types

Two database types are specified, the first being an object with fields for the table columns and an extra field for the group to be derived; the second is an array of the nested table form with elements of the first type.

### Function Pseudocode

```
Loop over a cursor selecting the records in order
	If the partition field value changes then
		Reset group and index-by array
	Else if the current value is already in the index-by array then
		Increment group number and reset index-by array
	End if
	Add the value to the index-by array
	Pipe the row out
End loop
```

### Function Definition (within package)

```sql
FUNCTION Hash_Array RETURN value_subseq_list_type PIPELINED IS

  TYPE value_list_type      IS TABLE OF PLS_INTEGER INDEX BY VARCHAR2(4000);
  l_value_list_NULL         value_list_type;
  l_value_list              value_list_type;
  l_group_no                PLS_INTEGER;
  old_part_key              VARCHAR2(10);

BEGIN

  FOR r_val IN (SELECT part_key, ind, val FROM value_subseqs ORDER BY part_key, ind) LOOP

    IF r_val.part_key != old_part_key OR old_part_key IS NULL THEN
      old_part_key := r_val.part_key;
      l_group_no := 1;
      l_value_list := l_value_list_NULL;
    ELSIF l_value_list.Exists (r_val.val) THEN
      l_group_no := l_group_no + 1;
      l_value_list := l_value_list_NULL;
    END IF;
    l_value_list (r_val.val) := 1;
    PIPE ROW (value_subseq_type (r_val.part_key, r_val.ind, r_val.val, l_group_no));

  END LOOP;

END Hash_Array;
```

### SQL
 Just select the fields named in the record from the function wrapped in the TABLE keyword.

```
SELECT
    part_key,
    ind,
    val,
    grp_no
  FROM TABLE (Subseq_Groups.Hash_Array)
 ORDER BY part_key, ind
```

## Pipelined Function Sim Solution
 This solution was added after performance testing on the the first data set, below. It is a sort of hybrid between the Model and pipelined function approach created to try to understand the apparent quadratic variation of Model with sequence length noted in testing.

### How It Works
 This function uses the same database types as the Hash solution, but with the same counting idea as the Model solution within an old-fashioned nested cursor structure that one would not expect to perform efficiently.

### Function Pseudocode

```
Loop over a cursor selecting the records in order
	If the partition field value changes then
		Reset group and group starting index
	Else
		Select counts of val and distinct val from the table between the group starting and current indices
		If difference in counts then
			Increment group number and reset group starting index
		End if
	End if
	Pipe the row out
End loop
```

### Function Definition (within package)

```sql
FUNCTION Sim_Model RETURN value_subseq_list_type PIPELINED IS

  l_group_no                PLS_INTEGER;
  l_ind_beg                 PLS_INTEGER;
  l_is_dup                  PLS_INTEGER;
  old_part_key              VARCHAR2(10);

BEGIN

  FOR r_val IN (SELECT part_key, ind, val FROM value_subseqs ORDER BY part_key, ind) LOOP

    IF r_val.part_key != old_part_key OR old_part_key IS NULL THEN
      old_part_key := r_val.part_key;
      l_group_no := 1;
      l_ind_beg := r_val.ind;
    ELSE
      SELECT Count(val) - Count(DISTINCT val)
        INTO l_is_dup
        FROM value_subseqs
       WHERE part_key = r_val.part_key
         AND ind      BETWEEN l_ind_beg AND r_val.ind;
      IF l_is_dup > 0 THEN
        l_group_no := l_group_no + 1;
        l_ind_beg := r_val.ind;
      END IF;
    END IF;
    PIPE ROW (value_subseq_type (r_val.part_key, r_val.ind, r_val.val, l_group_no));

  END LOOP;

END Sim_Model;
```

### SQL
 Just select the fields named in the record from the function wrapped in the TABLE keyword.

```sql
SELECT
    part_key,
    ind,
    val,
    grp_no
  FROM TABLE (Subseq_Groups.Sim_Model)
 ORDER BY part_key, ind
```

## Performance Analysis

 In [SQL Pivot and Prune Queries - Keeping an Eye on Performance](http://www.scribd.com/doc/54433084/SQL-Pivot-and-Prune-Queries-Keeping-an-Eye-on-Performance "SQL Pivot and Prune Queries - Keeping an Eye on Performance"), in May 2011, I first applied an approach to performance testing of SQL queries whereby the queries are tested across a 2-dimensional domain, using a testing framework developed for that work. The same approach has been followed here, although the framework has been substantially improved and extended since then. In this case it may be of interest to observe performance across the following two dimensions:

- Number of records per partition key
- Average group size

However, as I prefer to randomise the test data, the group size is known only after querying the data set, so I will use a proxy dimension instead. The value field will be a string of 20 random capital letters (A-T) of length equal to the proxy dimension. As this length increases so will the average group size. Generally execution time would be expected to be proportional to number of partition keys when the other dimensions are fixed, and I will use 2 partition key values throughout.

## Data Set 1
The first data set is chosen to keep the CPU times within reason for all queries, which limits the possible ranges (we’ll eliminate one query later and extend the ranges).

### Output Record Counts

| Depth/Width |   W1   |   W2   |   W4   |
| --- | --- | --- | --- |
| Records/Part> |   100   |   200   |   400   |
| D1 |   5   |   6   |   5   |
| D2 |   25   |   24   |   24   |
| D3 |   100   |   80   |   89   |
| D4 |   100   |   200   |   267   |

### CPU Times (Seconds)

| Query |  |   W1   |   W2   |   W4   |   W/Prior W Average   |
| --- | --- | --- | --- | --- | --- |
|   SQL Pairs   |   D1   |   2.11   |   8.49   |   33.99   |   4   |
| |   D2   |   4.88   |   13.66   |   42.15   |   2.9   |
| |   D3   |   10.3   |   46.71   |   255.64   |   5   |
| |   D4   |   10.33   |   78.25   |   595.58   |   7.6   |
|    Model   |   D1   |   0.32   |   1.08   |   4.23   |   3.6   |
| |   D2   |   0.31   |   1.14   |   4.23   |   3.7   |
| |   D3   |   0.34   |   1.16   |   4.65   |   3.7   |
| |   D4   |   0.36   |   1.36   |   4.86   |   3.7   |
|    Hash   |   D1   |   0.03   |   0.03   |   0.03   |   1   |
| |   D2   |   0   |   0.01   |   0.03   |   2   |
| |   D3   |   0.01   |   0.01   |   0.02   |   1.5   |
| |   D4   |   0.02   |   0.01   |   0.02   |   1.3   |

<img src="/migrated_images/2012/09/data_set_1_graph.png" alt="Dataset 1 graph" title="Dataset 1 graph" />

### Discussion

We can see immediately that on all data points there is the same performance ranking of the three queries and the differences are extreme. On W4-D4, SQL Pairs takes 123 times as long as Model, which in turn takes 243 times as long as Hash.

### SQL Pairs
 The Average Ratio figure in the table above is the average ratio between successive columns across the width dimension. On D1 this is 4, meaning that the CPU time has risen by the square of the width factor increase. On D8 it’s 7.6, being almost the cube of the factor. It appears that the performance varies with the square of the sequence length, except when the group size reaches the sequence length, when it becomes the cube. We would not consider this query for real problems when faster alternatives exist.

### Model
 The Average Ratio figure on all depths is about 3.7, meaning that the CPU time has risen by almost the square of the width factor increase. This is very surprising because, considering the algorithm one would assume to be effected by our query, if the group size remains constant then the work done in computing each group ought to be constant too, and the total work ought to rise by the same factor as the sequence length. We’ll look at this further in our second, larger data set, where we’ll also consider an apparently similar algorithm implemented in PL/SQL.

### Hash
 The Average Ratio figure varies between 1 and 2, but the CPU times are really too small for the figure to be considered reliable (note that the zero figure for W1-D2 was replaced by 0.005 in order to allow the log graph to include it). We can safely say, though, that this is faster by orders of magnitude even on very small problems, and will again look at a larger data set to see whether more can be said.

## Data Set 2 (Second Query Set)

 The second data set is chosen to give wider ranges, after excluding the Pairs query. A second function was added to replace it, labelled ‘Sim’ below and described above.

### Output Record Counts

| Depth/Width |   W1   |   W2   |   W4   |   W8   |   W16   |   W32   |
| --- | --- | --- | --- | --- | --- | --- |
| Records/Part> |   100   |   200   |   400   |   800   |   1600   |   3200   |
| D1 |   5   |   5   |   5   |   5   |   5   |   5   |
| D2 |   18   |   25   |   23   |   24   |   25   |   23   |
| D3 |   50   |   80   |   89   |   84   |   119   |   110   |
| D4 |   100   |   200   |   200   |   267   |   356   |   582   |
| D5 |   100   |   200   |   400   |   533   |   1600   |   2133   |
| D6 |   100   |   200   |   400   |   800   |   1600   |   3200   |

### CPU Times (Seconds)

| Query |  |   W1   |   W2   |   W4   |   W8   |   W16   |   W32   |   W/Prior W Average   |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
|   Hash   | D1 |   0.02   |   0.01   |   0.01   |   0.03   |   0.06   |   0.10   |   1.6   |
| | D2 |   0.02   |   0.01   |   0.02   |   0.04   |   0.06   |   0.09   |   1.5   |
| | D3 |   0.02   |   0.02   |   0.02   |   0.04   |   0.06   |   0.09   |   1.4   |
| | D4 |   0.01   |   0.02   |   0.01   |   0.03   |   0.06   |   0.11   |   1.9   |
| | D5 |   0.01   |   0.03   |   0.02   |   0.05   |   0.06   |   0.09   |   1.8   |
| | D6 |   0.01   |   0.02   |   0.01   |   0.05   |   0.06   |   0.09   |   2.0   |
|   Model   | D1 |   0.28   |   1.01   |   4.05   |   16.28   |   63.34   |   257.46   |   3.9   |
| | D2 |   0.27   |   1.06   |   4.00   |   15.97   |   66.27   |   249.99   |   3.9   |
| | D3 |   0.29   |   1.14   |   4.24   |   16.28   |   62.91   |   250.50   |   3.9   |
| | D4 |   0.33   |   1.24   |   4.74   |   17.66   |   69.00   |   263.34   |   3.8   |
| | D5 |   0.33   |   1.26   |   5.10   |   19.17   |   81.59   |   314.39   |   3.9   |
| | D6 |   0.33   |   1.28   |   4.99   |   20.55   |   80.80   |   331.38   |   4.0   |
|   Sim   | D1 |   0.25   |   0.37   |   0.74   |   1.70   |   3.14   |   6.18   |   1.9   |
| | D2 |   0.28   |   0.59   |   1.21   |   2.42   |   4.73   |   9.28   |   2.0   |
| | D3 |   0.33   |   0.67   |   1.43   |   2.79   |   5.86   |   11.28   |   2.0   |
| | D4 |   0.34   |   0.77   |   1.58   |   3.15   |   6.97   |   14.71   |   2.1   |
| | D5 |   0.36   |   0.73   |   1.69   |   3.61   |   9.64   |   22.58   |   2.3   |
| | D6 |   0.36   |   0.73   |   1.71   |   3.79   |   9.43   |   26.45   |   2.4   |

<img src="/migrated_images/2012/09/data_set_2_graph.png" alt="Dataset 2 graph" title="Dataset 2 graph" />

### Discussion

We can see immediately that on all data points there is the same performance ranking of the three queries and the differences are extreme. On W32-D6, Model takes 12 times as long as Sim, which in turn takes 294 times as long as Hash, Model being 3,678 slower than Hash. (I have included the elapsed times, which are very close to the CPU times, in the shared XL file above. I think the table data are being read from buffer cache throughout).

### Model
The Average Ratio figure on all depths is about 3.9, meaning that the CPU time has risen by almost the square of the width factor increase. It seems that the internal algorithm applied from the Model query here is doing something different from what we would expect, and with dire consequences for performance. For what it's worth, here is the last Execution Plan:

```
---------------------------------------------------------------------------------------------------------------------------
| Id  | Operation            | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |               |      1 |        |   6400 |00:05:33.45 |      30 |       |       |          |
|   1 |  SORT ORDER BY       |               |      1 |   6437 |   6400 |00:05:33.45 |      30 |   478K|   448K|  424K (0)|
|   2 |   SQL MODEL ORDERED  |               |      1 |   6437 |   6400 |05:57:01.43 |      30 |  1156K|   974K| 1002K (0)|
|   3 |    WINDOW SORT       |               |      1 |   6437 |   6400 |00:00:00.03 |      30 |   337K|   337K|  299K (0)|
|   4 |     TABLE ACCESS FULL| VALUE_SUBSEQS |      1 |   6437 |   6400 |00:00:00.01 |      30 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------
```

### Hash
The Average Ratio figure varies between 1.4 and 2, but the CPU times are probaby still too small for the figure to be considered reliable. We can again say, though, that this is faster than the Model solution by orders of magnitude even on very small problems and that the CPU time does not appear to be rising faster than linearly, so that the performance advantage will increase with problem size.

### Sim
The Average Ratio figure varies between 1.9 and 2.4, and up to D3 is not more than 2, which indicates a pretty exact proportionality between sequence length and CPU time. This is consistent with our expectation of linearity so long as the group sizes are smaller than sequence lengths. For D6, where the group sizes are the same as the sequence lengths, we would expect to see a quadratic term (number of counts doubles, and work done in sorting/counting also doubles, if that is linear), and time in fact trebles between W16 and W32. The results from this solution support the view that there is some internal implementation problem with Model for this example that is causing its quadratic CPU time variation. In my August 2011 revision of [Forming Range-Based Break Groups with Advanced SQL](http://www.scribd.com/doc/57696875/Forming-Range-Based-Break-Groups-With-Advanced-SQL "Forming Range-Based Break Groups with Advanced SQL"), I noted significant under-performance in a query using analytic functions, that also seemed to due to an internal implementation problem, and found a work-around that made the query perform as expected. I believe both examples illustrate well the power of this kind of dimensional performance analysis.

## Model Clause vs Pipelined Functions

I googled model performance problems and the two top-ranked articles are briefly discussed in the first two subsections below. 

### [MODEL Performance Tuning](http://www.sqlsnippets.com/en/topic-12100.html "MODEL Performance Tuning ")
The author states: ‘In some cases MODEL query performance can even be so poor that it renders the query unusable. One of the keys to writing efficient and scalable MODEL queries seems to be keeping session memory use to a minimum’. Our first case would seem to fall into one of those cases, Model being 3,678 times slower than the function, at 332 seconds, on a problem of only 3,200 records (times two partition keys) and rising quadratically. The article considers the effects of changing various features of a test Model problem on the memory usage, assuming that that is one cause of poor performance. 

### [From Pipelined Function to Model](http://www.oreillynet.com/network/2004/08/10/MODELclause.html "From Pipelined Function to Model")

This article (no longer available on 13 July 2025) is interesting in that it takes pretty much the opposite line to my own. For a different problem, the author started with a pipelined function solution, and then went to Model on performance grounds. He says: ‘Since we were returning the rows in a pipelined (streaming) fashion, the performance was fine initially. It was when the function was called constantly and then joined with other tables that we ran into trouble’. The problem he identifies seems to be principally the fact that Oracle’s Cost Based Optimiser (CBO) cannot accurately predict the cardinality of the function, and assigns a default, on his system (as on mine) of 8168. This can cause poor execution plans when joined with other tables. His initial solution was to use the CARDINALITY hint, which worked but is undocumented and inflexible. He also notes possible performance issues caused by Oracle’s translating the PL/SQL table into an SQL result set and goes on to propose a Model solution to avoid this problem. Unfortunately, the author does not provide any results on significant data sets to demonstrate the performance differences. 

The following article looks specifically at the cardinality issue. _[setting cardinality for pipelined and table functions](http://www.oracle-developer.net/display.php?id=427 "setting cardinality for pipelined and table functions")_ The author (Adrian Billington) considers four techniques that he labels thus:

- CARDINALITY hint (9i+) undocumented
- OPT\_ESTIMATE hint (10g+) undocumented
- DYNAMIC\_SAMPLING hint (11.1.0.7+)
- Extensible Optimiser (10g+)

The first two are not recommended on the grounds of being undocumented. The third option appears quite a lot simpler than the fourth and I will look at that approach in a second example problem, in my next article [List Aggregation in Oracle - Comparing Three Methods](https://brenpatf.github.io/migrated/list-aggregation-in-oracle-comparing-three-methods/).

### Discussion
 
The pipelined function approach is plainly much faster than the SQL solutions in this example, but one has to be aware of possible issues such as the cardinality issue mentioned. One also needs to be aware that pure SQL statements are ‘read-consistent’ in Oracle, but this is not the case when functions are called that themselves do SQL. Context switches between the SQL and PL/SQL engines, as well as the work done in translating between collections and SQL record sets, are often cited as performance reasons for preferring SQL-only solutions. As we have seen though, these issues, while real, can be dwarfed by algorithmic differences.

## Conclusion

- For the subsequence grouping problem addressed, using a pipelined function is faster by far than the SQL-only solutions identified
- Although widely asserted, the notion that any query processing will be executed more efficiently in pure SQL than in PL/SQL is a myth
- The Model solution using embedded aggregate Counts is much slower than expected and its quadratic CPU variation suggests performance problems within Oracle’s internal implementation of the Model clause
- Dimensional performance analysis is very powerful although its use appears to be extremely rare in the Oracle community
- It is suggested that diagrammatic techniques, such as my Query Structure Diagramming, although also very rarely used, offer important advantages for query documentation and design

