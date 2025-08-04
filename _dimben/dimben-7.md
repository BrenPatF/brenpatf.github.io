---
layout: post
title: "DIMBEN 7: Oracle DML: A Case Study I - Update vs Merge, An Example"
date: 2017-09-24
migrated: true
dimben_prev: /dimben/dimben-6/
dimben_next: /dimben/dimben-8/
categories: 
  - "oracle"
  - "performance"
  - "plsql"
  - "sql"
  - "testing"
tags: 
  - "benchmarking"
  - "cpu"
  - "merge"
  - "oracle"
  - "performance-2"
  - "sql"
  - "tuning"
  - "update"
---
#### Part 7 in a series on: Dimensional Benchmarking of SQL Performance

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

This is the first part of a two-part article. The second part is here: [Benchmarking Oracle DML: A Case Study II - Effects of Indexes](/dimben/dimben-8/).

Some time ago I was involved in performing a one-off update of a column in an Oracle table of 250 million records, of which about 50 million would be updated. In the initial attempt, in development, the update ran for a very long time before aborting with the error:

> ORA-30036: unable to extend segment by 8 in undo tablespace 'UNDOTBS'

I noted that the updated column featured in two indexes, and realised that the update would likely entail much more work in updating the indexes than in the update of the table. I reasoned that, because the index data are physically stored in a way that depends on the values, changing the values could involve a lot of physical restructuring on disk. Updating the values in the table, on the other hand, probably would not involve much restructuring of the table data, if the storage requirements of the new values were similar to those of the old ones, which they were. Anyway, we changed the process to have it drop the indexes, do the update, then recreate the indexes. There are other, possibly even faster, ways of doing this (as we'll see), but the change allowed the whole process to complete in around an hour.

Some time later I noticed an OTN thread, [Improve query performance instead of aggregrate function](https://community.oracle.com/thread/4073630), in which the poster requested help in improving the performance of an Oracle update statement (the title is a little misleading). Recalling my earlier experience, I suggested that dropping any indexes that included the updated column would improve performance. As it turned out, the poster stated that the table did not have any indexes, and other posters suggested various alternative ways of doing the update.

In the example there is a product sales table having product id and sales date columns (and a few others unspecified), and the update sets the sales date to a constant value for the earliest sales date for each product. The data model and SQL in the thread are relatively simple, and it occurred to me that it would be interesting to use the example to do a case study of the performance impact of indexes on updates and other DML statements.

In this two-part article I'll use parameterised datasets to do two sets of comparisons: First, we'll compare the performance of the original poster's update statement with a slightly modified version of another poster's solution using 'merge', across a 2-dimensional grid of dataset points with no indexes. Second (in part 2), we'll compare the performance of both forms of update, plus related delete and insert statements on the 1-dimensional 'slice' of dataset points where the updates apply to about half of the total records. In the second set, we'll run the statements in the presence of: (i) No indexes; (ii) product id index only; (iii) product id and sales date indexes, and we'll also compare with a Create Table As Select (CTAS) approach.

To do the comparisons, I use my own Oracle benchmarking framework, which has been upgraded during this work to cover DML and DDL more fully, and is available on GitHub and described on the [index page](/migrated/dimben-series-index/) for this series of articles.

_Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production_

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr"><a href="https://t.co/Cy479AwFMS">https://t.co/Cy479AwFMS</a> twittified: Performance case study of DML Update vs Merge, no indexes. SQL, 4x4 dataset <a href="https://twitter.com/hashtag/brenThread?src=hash&amp;ref_src=twsrc%5Etfw">#brenThread</a> <a href="https://twitter.com/hashtag/oracle?src=hash&amp;ref_src=twsrc%5Etfw">#oracle</a> ?<br>1/9 <a href="https://t.co/uLWTBgX3kn">pic.twitter.com/uLWTBgX3kn</a></p>— Brendan Furey (@BrenPatF) <a href="https://twitter.com/BrenPatF/status/914513881044783104?ref_src=twsrc%5Etfw">October 1, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## Test Data

### Data Structure

```sql
CREATE TABLE product_sales (product_id NUMBER, sales_date DATE)
/
CREATE INDEX ps_date_n1 ON product_sales (sales_date)
/
CREATE INDEX ps_prd_n1 ON product_sales (product_id)
/
```

### Data Generator

The data generation procedure takes two parameters, a width parameter being the number of products, and a depth parameter being the number of records per product. Rows are generated using two cross-joined subqueries that each generate rows via the common 'select from dual connect by' method, as follows:

```sql
INSERT INTO product_sales
WITH prod_gen AS (
  SELECT LEVEL + (i - 1)*c_max_rowgen product_id
    FROM DUAL
    CONNECT BY LEVEL <= l_wide_batch_sizes(i)
), day_gen AS (
  SELECT LEVEL rn
    FROM DUAL
    CONNECT BY LEVEL <= p_point_deep
)
SELECT p.product_id, c_start_date + Mod (Abs (DBMS_Random.Random), c_n_days_in_century)
  FROM prod_gen p
 CROSS JOIN day_gen d;
```

Note that:

- Product ids are sequential
- Dates are randomized across the century from 1 January 1900
- A call is made to DBMS\_Random.Seed at the start of the procedure to ensure each call with the same parameters will get the same (pseudo-)random dates
- The insert occurs within a loop with index i in order to limit the number of rows generated at once (see below for reason)

The reason for limiting the number of rows generated by inserting within a loop is that the Oracle tree-walk mechanism uses memory increasing with number of levels traversed, and I hit the dreaded

> Completed with error: ORA-30009: Not enough memory for CONNECT BY operation

There are various ways of avoiding this, including this, [Generating lots of rows using connect by – safely!](http://blog.tanelpoder.com/2008/06/08/generating-lots-of-rows-using-connect-by-safely/), which suggests cross-joining as many tree-walk subqueries as are necessary to generate the overall number from tree-walks of smaller size. In our situation, however, this approach is problematic because we pass in the desired number as a parameter. To get the exact number desired we would have to create the statement dynamically and create a set of subqueries with the subquery limits being products of the prime factors of the number. This is impractical and in any case the highest prime factor could be too large. For this reason the inserts are performed in batches within a loop over an array containing the batch sizes.

Lower depth values correspond to larger proportions of records to be updated, with smaller numbers of values to be sorted within the product id partitions. For example, at depth 2, around half the records are updated, while at depth 100 around 1% are updated.

## Test Case 1: Update vs Merge, no Indexes

Both update and merge statements below are based on statements in the thread mentioned above. I reformatted them and altered the merge to make it consistent with the update in updating all records of minimum date where duplication occurs.

One other change may be worth highlighting: As Steven Feuerstein noted recently, [About the Date Literal in Oracle Database](http://stevenfeuersteinonplsql.blogspot.cz/2017/08/using-date-literal-in-oracle-database.html), the date literal seems to be under-used by Oracle developers, but it is neater than using To\_Date with an explicit format mask. I replaced

> TO\_DATE ('01.01.2017', 'dd.mm.yyyy')

with the literal equivalent

> DATE '2017-01-01'

I incidentally also changed the year for my test data.

### Update/Merge Statements

#### Update (UPD\_DML)

```sql
UPDATE product_sales sd
   SET sd.sales_date = DATE '1900-01-01'
 WHERE 1=1 AND sd.sales_date = ( 
   SELECT Min(sd2.sales_date)
     FROM product_sales sd2
    WHERE sd.product_id = sd2.product_id   
 )
   AND sd.sales_date != DATE '1900-01-01'
```

This is essentially the same statement as in the original post by _user12251389_.

#### Merge (MRG\_DML)

```sql
MERGE INTO product_sales tgt
USING (SELECT *
       FROM (
         SELECT rowid arowid, product_id, DATE '1900-01-01' sales_date,
                sales_date AS old_sales_date,
                Rank() OVER (PARTITION BY product_id ORDER BY sales_date) rn
         FROM   product_sales    
       )
       WHERE rn = 1 AND 0 = Decode(sales_date, old_sales_date, 1, 0)) src
   ON (tgt.rowid = src.arowid)
 WHEN MATCHED THEN UPDATE
  SET tgt.sales_date = src.sales_date
```

This is essentially the same statement as in the post by responder _Paulzip_, except that where he had Row\_Number I have put Rank to allow for duplicate updating in order to be consistent with the update statement.

For my performance testing I added two hinted versions of the above merge, the first of which has the following hint:

> no\_swap\_join\_inputs(@"SEL$F5BB74E1" "TGT"@"SEL$1")

while the second has two hints. These rather strange-looking hints will be explained below in relation to execution plans.

## Results

The four SQL statements were run across a 2-dimensional grid of width and depth data points. After each update the number of records is saved against data point and SQL statement, and the transaction is rolled back. The elapsed and CPU times, and the CPU percentages, are displayed below for wide and deep slices of the grid in the two scrollboxes below. Of course, data creation and rollback times are not included, although the instrumentation reports them separately in the log file.

The detailed results can be seen in the embedded file below.

<iframe src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICVDvsBAAAAAK6LZ8ch_uJcp3A" width="100%" height="600" frameborder="0" scrolling="no"></iframe>

This is also available from a [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/benchmarking-oracle-dml-a-case-study-i-update-vs-merge-an-example).

### Wide Slice Graphs

In the wide slice there are 10 million products and from 2 to 100 dates per product, with a maximum of 100 million records. At D=2 about half the records are updated, and at D=100 about 1% are updated.

#### Elapsed Times: Wide Slice

<img src="/migrated_images/2017/09/10_W_Wide_ELA.png" alt="" title="" />

#### CPU Times: Wide Slice

<img src="/migrated_images/2017/09/10_W_Wide_CPU.png" alt="" title="" />

#### CPU Percentages: Wide Slice

<img src="/migrated_images/2017/09/10_W_Wide_CPU_PCT.png" alt="" title="" />

### Deep Slice Graphs

In the deep slice there are 100 dates per product and from 100,000 to 1,000,000 products, with a maximum of 100 million records. About 1% of the records are updated.

#### Elapsed Times: Deep Slice

<img src="/migrated_images/2017/09/10_W_Deep_ELA1.png" alt="" title="" />

#### CPU Times: Deep Slice

<img src="/migrated_images/2017/09/10_W_Deep_CPU.png" alt="" title="" />

#### CPU Percentages: Deep Slice

<img src="/migrated_images/2017/09/10_W_Deep_CPU_PCT.png" alt="" title="" />

The results show:

- The update SQL (UPD\_DML) is faster at all data points than the merges, being generally twice as fast or better at the deep data points
- The
- At the shallow data points (D=2), the timings are much closer, reflecting in part the fact that proportionally more times goes to doing the actual updating
- The two hinted versions of the merge are significantly faster than the unhinted version (MRG\_DML), and we'll discuss this in relation to execution plans below

It is interesting to note that the update statement from the original poster in the OTN thread is faster than (the small variation on) the more complex merge statement proposed in the thread to improve performance! I considered whether my substitution of Rank for Row\_Number might have been to blame, and found that it did have a significant effect on the execution plan, where it caused a hash join to be used in place of a nested loops join. In fact, the hinted version MRG\_HT2 has the same plan as would the Row\_Number version, and is faster than the unhinted merge, but still slower than the update.

### Execution Plans

The benchmarking framework ensures that the SQL engine hard-parses, and thus re-calculates the optimal execution plan, at each instance, by inserting a functionally meaningless condition x=x into the statement, where x is the number given by the current timestamp to the millisecond formatted thus: _To\_Char(SYSTIMESTAMP, 'yymmddhh24missff3')_. This results in the SQL id, which is the hash of the SQL text, being different each time.

The execution plan for each SQL statement execution is printed to log, and was the same for each data point. The plans are listed in the scrollbox below at the highest data point.

#### Execution Plan for Update (UPD\_DML)

```
--------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------
|   0 | UPDATE STATEMENT       |               |      1 |        |      0 |00:01:04.49 |    1516K|   5474 |       |       |          |
|   1 |  UPDATE                | PRODUCT_SALES |      1 |        |      0 |00:01:04.49 |    1516K|   5474 |       |       |          |
|*  2 |   HASH JOIN            |               |      1 |   1001K|    998K|00:00:56.15 |     496K|   5474 |    53M|  8523K|   52M (0)|
|   3 |    VIEW                | VW_SQ_1       |      1 |   1001K|    997K|00:00:37.33 |     248K|      0 |       |       |          |
|*  4 |     FILTER             |               |      1 |        |    997K|00:00:37.15 |     248K|      0 |       |       |          |
|   5 |      SORT GROUP BY     |               |      1 |   1001K|   1000K|00:00:37.09 |     248K|      0 |    70M|    15M|   62M (0)|
|   6 |       TABLE ACCESS FULL| PRODUCT_SALES |      1 |    100M|    100M|00:00:01.62 |     248K|      0 |       |       |          |
|*  7 |    TABLE ACCESS FULL   | PRODUCT_SALES |      1 |     99M|     99M|00:00:12.53 |     248K|   5474 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("SD"."SALES_DATE"="MIN(SD2.SALES_DATE)" AND "SD"."PRODUCT_ID"="ITEM_1")
   4 - filter(MIN("SD2"."SALES_DATE")<>TO_DATE(' 1900-01-01 00:00:00', 'syyyy-mm-dd hh24:mi:ss'))
   7 - filter("SD"."SALES_DATE"<>TO_DATE(' 1900-01-01 00:00:00', 'syyyy-mm-dd hh24:mi:ss'))

```

#### Execution Plan for Merge (MRG\_DML)

```
--------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
--------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | MERGE STATEMENT             |               |      1 |        |      0 |00:02:30.40 |    1516K|    428K|    428K|       |       |          |         |
|   1 |  MERGE                      | PRODUCT_SALES |      1 |        |      0 |00:02:30.40 |    1516K|    428K|    428K|       |       |          |         |
|   2 |   VIEW                      |               |      1 |        |    998K|00:02:02.03 |     496K|    428K|    428K|       |       |          |         |
|*  3 |    HASH JOIN                |               |      1 |    100M|    998K|00:02:01.77 |     496K|    428K|    428K|  2047M|    52M|   55M (1)|    3329K|
|   4 |     TABLE ACCESS FULL       | PRODUCT_SALES |      1 |    100M|    100M|00:00:09.96 |     248K|      0 |      0 |       |       |          |         |
|*  5 |     VIEW                    |               |      1 |    100M|    998K|00:01:33.29 |     248K|  15903 |  15903 |       |       |          |         |
|*  6 |      WINDOW SORT PUSHED RANK|               |      1 |    100M|   1001K|00:01:33.33 |     248K|  15903 |  15903 |    70M|  2904K|   97M (1)|         |
|   7 |       TABLE ACCESS FULL     | PRODUCT_SALES |      1 |    100M|    100M|00:00:11.63 |     248K|      0 |      0 |       |       |          |         |
--------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - access("TGT".ROWID="from$_subquery$_007"."AROWID")
   5 - filter(("RN"=1 AND DECODE(INTERNAL_FUNCTION("SALES_DATE"),"OLD_SALES_DATE",1,0)=0))
   6 - filter(RANK() OVER ( PARTITION BY "PRODUCT_ID" ORDER BY "SALES_DATE")<=1)

```

#### Execution Plan for Merge with Join Inputs Hint(MHT\_DML)

```
----------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | MERGE STATEMENT             |               |      1 |        |      0 |00:02:02.01 |    1516K|  19175 |  15903 |       |       |          |
|   1 |  MERGE                      | PRODUCT_SALES |      1 |        |      0 |00:02:02.01 |    1516K|  19175 |  15903 |       |       |          |
|   2 |   VIEW                      |               |      1 |        |    998K|00:01:51.06 |     496K|  19175 |  15903 |       |       |          |
|*  3 |    HASH JOIN                |               |      1 |    100M|    998K|00:01:50.84 |     496K|  19175 |  15903 |    77M|  5796K|  107M (0)|
|*  4 |     VIEW                    |               |      1 |    100M|    998K|00:01:32.03 |     248K|  15903 |  15903 |       |       |          |
|*  5 |      WINDOW SORT PUSHED RANK|               |      1 |    100M|   1001K|00:01:32.12 |     248K|  15903 |  15903 |    70M|  2904K|   97M (1)|
|   6 |       TABLE ACCESS FULL     | PRODUCT_SALES |      1 |    100M|    100M|00:00:10.95 |     248K|      0 |      0 |       |       |          |
|   7 |     TABLE ACCESS FULL       | PRODUCT_SALES |      1 |    100M|    100M|00:00:10.90 |     248K|   3272 |      0 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - access("TGT".ROWID="from$_subquery$_007"."AROWID")
   4 - filter(("RN"=1 AND DECODE(INTERNAL_FUNCTION("SALES_DATE"),"OLD_SALES_DATE",1,0)=0))
   5 - filter(RANK() OVER ( PARTITION BY "PRODUCT_ID" ORDER BY "SALES_DATE")<=1)

```

#### Execution Plan for Merge with Nested Loops Hint(MH2\_DML)

```
------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | MERGE STATEMENT               |               |      1 |        |      0 |00:02:01.99 |    1516K|  27343 |  15903 |       |       |          |
|   1 |  MERGE                        | PRODUCT_SALES |      1 |        |      0 |00:02:01.99 |    1516K|  27343 |  15903 |       |       |          |
|   2 |   VIEW                        |               |      1 |        |    998K|00:01:34.39 |     496K|  27343 |  15903 |       |       |          |
|   3 |    NESTED LOOPS               |               |      1 |    100M|    998K|00:01:34.16 |     496K|  27343 |  15903 |       |       |          |
|*  4 |     VIEW                      |               |      1 |    100M|    998K|00:01:30.08 |     248K|  15903 |  15903 |       |       |          |
|*  5 |      WINDOW SORT PUSHED RANK  |               |      1 |    100M|   1001K|00:01:30.03 |     248K|  15903 |  15903 |    70M|  2904K|   97M (1)|
|   6 |       TABLE ACCESS FULL       | PRODUCT_SALES |      1 |    100M|    100M|00:00:11.43 |     248K|      0 |      0 |       |       |          |
|   7 |     TABLE ACCESS BY USER ROWID| PRODUCT_SALES |    998K|      1 |    998K|00:00:04.03 |     247K|  11440 |      0 |       |       |          |
------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   4 - filter(("RN"=1 AND DECODE(INTERNAL_FUNCTION("SALES_DATE"),"OLD_SALES_DATE",1,0)=0))
   5 - filter(RANK() OVER ( PARTITION BY "PRODUCT_ID" ORDER BY "SALES_DATE")<=1)

```

The choices made by Oracle's Cost Based Optimiser (CBO) in construction of the execution plan are crucially dependent on the cardinality estimates it makes at each step. The outputs above display these estimates along with the actual numbers of rows returned. We know that the total number of records is 100M, and that approximately 1M of these are updated at the extreme wide/deep data point shown. How accurate are the cardinality estimates in the plans above? Let's take them in turn.

##### UPD\_DML
Here the update subquery results in a hash join between the table and an internal view in which the records from a separate scan of the table are sorted and filtered to produce the records with minimum date value by product. The estimated cardinality of the view in step #3 is 1001K, which is close to the actual cardinality of 997K.

The view is used as the _build_ table with the table itself in step #7 being used as the _probe_ table. This looks like the right strategy because the smaller rowset is generally preferred as the build table, used to build the hash table for the join.

##### MRG\_DML
The merge statement also has at its heart a hash join between the table and an internal view, but this time the build and probe tables are reversed, and we observe that the cardinality estimate for the view, in step #5, is 100M, whereas the actual is 998K. The CBO has not been able to detect that the _rn = 1_ condition on the rank function would reduce the cardinality by a factor of about a hundred, so either choice of build table would look similar.

##### MHT\_DML
I wondered how big an effect making the 'wrong' choice for the build table would have, and so looked to include a hint to force the 'correct' choice, and made this the statement MHT\_DML. I wrote an article on the subject of hash join 'inner' ordering (as I called it there) last year, [A Note on Oracle Join Orders and Hints](https://brenpatf.github.io/migrated/a-note-on-oracle-join-orders-and-hints/), which used a simple 3-table query with no subqueries. In simple cases such as that one it is easy to force the choice of build table using the hints _(no\_)swap\_join\_inputs (tab)_ where tab is the alias of the table to be joined to the current rowset.

In more complicated situations with subqueries, such as we have in our merge statement, it is a little harder since we need to specify the target using internal query block names that are not in the original statement. Fortunately, there is an easy way to get the desired hint: The execution plans above are displayed using Oracle's _DBMS\_XPlan.Display\_Cursor_ API, and if you pass the keyword OUTLINE to this API, it returns the list of fully specified hints that determine the execution plan. We can extract the relevant hints from this list and modify if necessary. In the unhinted outline there is the hint:

> swap\_join\_inputs(@"SEL$F5BB74E1" "TGT"@"SEL$1")

so to reverse the build/probe choice we simply change swap\_join\_inputs to no\_swap\_join\_inputs.

This improves performance by 19% at the extreme data point.

Incidentally, Tom Kyte discusses in detail how to use these outline hints to investigate different plans and to create baselines using SQL Plan Management in a 2014 Oracle Magazine article (referenced also in part 2 of this current article): [On Table Updates and SQL Plan Baselines](http://www.oracle.com/technetwork/issue-archive/2014/14-jul/o44asktom-2196080.html).

Another way of getting at the hint syntax is to use SQL Developer, as shown here:

<img src="/migrated_images/2017/09/SQLDev_Hint_KrAe2o7sED.gif" alt="" title="" />

##### MH2\_DML
As mentioned above, I wondered what effect the use of Rank instead of the Row\_Number used in the OTN thread had on performance. To check this I replaced Rank with Row\_Number, ran an Explain Plan, and found quite a big difference in the plan (a hash join changing to a nested loops join), despite the fact that the difference in actual cardinality is extremely small so that the same plan should be optimal for both.

I followed the same approach as in MHT\_DML to obtain the hints that would force the same plan as the Row\_Number version via the SQL outline. This time two hints were required (you could just take the whole outline set of course):

> leading(@"SEL$F5BB74E1" "from$\_subquery$\_007"@"SEL$2" "TGT"@"SEL$1") use\_nl(@"SEL$F5BB74E1" "TGT"@"SEL$1")

This version perfroms slightly better in CPU terms than the firsted hinted version, with smaller differences in elapsed times, and they perform very similarly at the higher data points.

## Conclusion

In part 1 of this article we demonstrated the use of my benchmarking framework for comparing DML with detailed timings, statistics and execution plans across a 2-dimensional grid. In terms of the problem addressed, and general learnings, a couple of points can be made:

- The 'obvious' Update turned out to be faster as well as simpler than a more complicated Merge; Oracle's own transformation of the update subquery, into a join between the table and an internal view, performed better than the hand-crafted attempt
- The OUTLINE parameter to DBMS\_XPlan.Display\_Cursor is very useful to extract more difficult hint syntax (you can also get it from SQL Developer, by right-clicking the hints displayed below an execution plan)
- We also showed, using a gif example, how to get these hints from SQL Developer
- Regarding memory problems when generating large numbers of rows for test data, we linked to one solution, and provided an alternative for when that one is inapplicable

The example problem, together with all code used in both parts of this article, and the revisions made to the framework are available here: [A Framework for Dimensional Benchmarking of SQL Performance](https://github.com/BrenPatF/dim_bench_sql_oracle).

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
