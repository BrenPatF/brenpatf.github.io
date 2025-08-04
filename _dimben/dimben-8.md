---
layout: post
title: "DIMBEN 8: Oracle DML: A Case Study II - Effects of Indexes"
date: 2017-09-24
migrated: true
dimben_prev: /dimben/dimben-7/
categories: 
  - "oracle"
  - "performance"
  - "plsql"
  - "sql"
  - "testing"
tags: 
  - "cpu"
  - "oracle"
  - "performance-2"
  - "sql"
  - "tuning"
  - "update"
---
#### Part 8 in a series on: Dimensional Benchmarking of SQL Performance

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

This is the second part of a two-part article. The first part, [Benchmarking Oracle DML: A Case Study I - Update vs Merge, An Example](/dimben/dimben-7/), compares an Update and a Merge statement for performance in updating a table involving a subquery, in the absence of indexes. The first part describes the problem and the mechanism for generating parameterised test data.

In this second part, we are interested in the effects on performance of indexes for DML statements that affect a large proportion of the table. To that end, we take as data sets the 1-dimensional 'shallow slice' of data set points from part 1 where the updates apply to about half of the total records. We'll run the statements in the presence of: (i) No indexes; (ii) product id index only; (iii) product id and sales date indexes. Note that the only updated column is sales date.

The idea behind the analysis is of course that when performing a large batch DML we may be able to drop the indexes first, then recreate them after the DML, depending on our environment. Obviously, if we save time on the DML this will be offset to some extent by the need to recreate the indexes. Therefore we will also time the index creations, and for good measure we'll include a timing of the well-known CTAS approach for bulk updates, where a new table is created by selecting from the table to be updated, and then the old table dropped and the new one renamed.

Tom Kyte discusses issues around this kind of bulk update in a 2014 Oracle Magazine article (referenced also in part 1 of this current article) [On Table Updates and SQL Plan Baselines](http://www.oracle.com/technetwork/issue-archive/2014/14-jul/o44asktom-2196080.html). He notes, in particular, that the CTAS approach benefits from avoiding undo creation.

_Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production_

## DML and DDL Statements

In this part 2 we have two groups to test: The first is DML, including the update from part 1, and adding an insert and a delete statement. The group actually includes the three versions of the merge from part 1, but as those were always slower than the update, we'll exclude them from the article.

The second group has the two create index statements and the Create Table As Select. We can add timings from this group to the DML timings to compare DML in the presence of one or both indexes, with doing it without indexes, then recreating afterwards. We can also compare the update approaches with the CTAS approach with index creation added in to its timings.

### DML Statements (Group DMLSALES)

In addition to the SQL code below, there is also a condition 'WHERE 1=1' added to all the statements except the index creations, which is a placeholder with the framework package replacing the '1's with the formatted timestamp mentioned in part 1.

#### Update (UPD\_DML)

This statement updates the records with minimum date by product with the hard-coded minimum date.

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

#### Insert (INS\_DML)

This statement selects the records that the update updates and re-inserts them with the hard-coded minimum date replacing the minimums by product.

```sql
INSERT INTO product_sales
WITH date_mins AS (
    SELECT product_id
      FROM product_sales
     GROUP BY product_id
     HAVING Min(sales_date) != DATE '1900-01-01'
)
SELECT product_id, DATE '1900-01-01'
  FROM date_mins
```

#### Delete (DEL\_DML)

This statement deletes the records where the update updated them.

```sql
DELETE product_sales sd
  WHERE 1=1 AND (product_id, sales_date) IN (
    SELECT product_id, Min(sales_date)
      FROM product_sales
     WHERE 1=1
     GROUP BY product_id
    HAVING Min(sales_date) != DATE '1900-01-01'
    )
```

### DDL Statements (Group DDLSALES)

In this group we need a post\_query\_sql step to drop the created objects so that the execution at the next data point will work.

#### Create product\_id index (PRD\_DDL)

##### pre\_query\_sql

```sql
CREATE INDEX ps_prd_n1 ON product_sales (product_id)
```

##### post\_query\_sql

```sql
DROP INDEX ps_prd_n1
```

#### Create sales\_date index (SDT\_DDL)

```sql
CREATE INDEX ps_date_n1 ON product_sales (sales_date)
```

##### post\_query\_sql

```sql
DROP INDEX ps_date_n1
```

#### Create table as select (CRE\_DDL)

##### pre\_query\_sql

```sql
CREATE TABLE product_sales_ctas AS 
SELECT product_id,
       CASE WHEN sales_date = Min(sales_date) OVER (PARTITION BY product_id) THEN DATE '2017-01-01' ELSE sales_date END sales_date
  FROM product_sales
```

##### post\_query\_sql

```sql
DROP TABLE product_sales_ctas
```

## Data Sets

The same data generator code was used as in part 1, but this time we are interested in DML where a large proportion of the records are affected, so will take the 'shallow' data set only, where D=2 and W is in (1, 4, 7, 10). These lead to sizes of (200K, 800K, 1.4M, 2M) records of which about half are updated or deleted or are copied by the insert.

For the DML statement group a batch is run for the given data set, with indexes present as follows:

0: No indexes 1: ps\_prd\_n1 (product id) 2: ps\_prd\_n1 (product id), ps\_date\_n1 (sales date)

For the DDL statement group a batch is run for the given data set, with post statement DDL dropping the created object.

## Results

The detailed results can be seen in the embedded file below, including for the merge versions that are not included in the diagrams later.

<iframe src="https://1drv.ms/x/c/95ec670ea6af8ed1/UQTRjq-mDmfsIICVEPsBAAAAAPCZSaMHKdFXloI" width="100%" height="600" frameborder="0" scrolling="no"></iframe>

This is also available from a [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/benchmarking-oracle-dml-a-case-study-ii-effects-of-indexes).

## Graphs

Although the DML statements were run against four data points, with results as in the embedded file above, we show graphs only at the wide point W=10, having 1M products with two records each. The graphs take the number of indexes as the x-axis. Scrollboxes are used to show elapsed time graphs at the top, while CPU and %CPU can be seen by scrolling down.

### DML Times by Indexes

#### Elapsed Times: DML

<img src="/migrated_images/2017/09/DML_ELA.png" alt="" title="" />

#### CPU Times: DML

<img src="/migrated_images/2017/09/DML_CPU.png" alt="" title="" />

#### %CPU/Elapsed Times: DML

<img src="/migrated_images/2017/09/DML_CPU_PCT.png" alt="" title="" />

- The 1-index case has the index on product id, which is not an updated column, and so the time for the update is about the same as with no indexes
- The insert is much faster than both delete and update in all cases

### DML Times _Due to_ Indexes

Here we subtract the times in the 0-index case from the others to get estimates for the contributions to total times attributable to index processing.

#### %Elapsed Times due to Indexes: DML

<img src="/migrated_images/2017/09/DML_ELA_PCT_IND1.png" alt="" title="" />

#### %CPU Times due to Indexes: DML

<img src="/migrated_images/2017/09/DML_CPU_PCT_IND.png" alt="" title="" />

- The insert shows the greatest percentages due to indexes, having relatively small time when there are no indexes
- As noted above, an index on a non-updated column has no effect on update time, but does affect insert and delete

### Combined Update and Index Creation Times

Here we add the index creation times to the pure DML times to compare the total times by direct update with the time taken when you drop them first, then re-create them after the update. We also include the CTAS method.

#### Elapsed Total Times: Update

<img src="/migrated_images/2017/09/UPD_ELA_TOT.png" alt="" title="" />

#### CPU Total Times: Update

<img src="/migrated_images/2017/09/UPD_CPU_TOT.png" alt="" title="" />

#### %CPU/Elapsed Times: Update

<img src="/migrated_images/2017/09/UPD_CPU_PCT_TOT.png" alt="" title="" />

- For the 2-index case, where one of the indexes is on the updated column, the elapsed time is two and a half times as great for the direct update compared with dropping and re-creating the indexes
- You could save a bit of time by leaving the non-updated-column index in place as that has no impact on the update (although I did not do this here)
- The CTAS method was much faster than the others
- If you scroll down you will see that the %CPU time is very high for CTAS, close to 100%, whereas for the other methods it's less than a third. This is no doubt related to the absence of undo processing noted by Tom Kyte in the article linked earlier:

> And remember, they never create UNDO for the CREATE TABLE or CREATE INDEX commands

### Combined Insert/Delete and Index Creation Times

Here we add the index creation times to the pure DML times to compare the total times by direct update with the time taken when you drop them first, then re-create them after the update.

#### Elapsed Total Times: Insert/Delete

<img src="/migrated_images/2017/09/INS_ELA_TOT.png" alt="" title="" />

#### CPU Total Times: Insert/Delete

<img src="/migrated_images/2017/09/INS_CPU_TOT.png" alt="" title="" />

#### %CPU/Elapsed Times: Insert/Delete

<img src="/migrated_images/2017/09/INS_CPU_PCT_TOT.png" alt="" title="" />

- For the 2-index case, the elapsed times are about four, and two and a half, times as great for the direct DML compared with dropping and re-creating the indexes, for insert and delete respectively
- In fact, the times taken in creating the indexes are quite small compared to the DML, so that the times increase much more slowly with number of indexes for the drop/re-create methods
- The %CPU time is significantly higher for the drop and re-create indexes methods

## Conclusion

In part 2 of this article we compared timings for the DML statements on the example problem with and without indexes where a large proportion of records are affected. Findings include:

- Dropping the indexes before doing the DML, then adding them back again usually gives much better performance than applying the DML directly, depending on the type of DML and the columns in the index
- The CTAS method for updates is even faster, and can also be applied for the inserts and deletes, although we didn't include this here
- Graphs show that CTAS has very high %CPU, reflecting the absence of undo processing mentioned in the linked article from Oracle Magazine

The example problem, together with all code used in both parts of this article, and the revisions made to the framework are available here: [A Framework for Dimensional Benchmarking of SQL Performance](https://github.com/BrenPatF/dim_bench_sql_oracle). The framework, as presented at the 2017 Ireland Oracle User Group conference, [Dimensional Performance Benchmarking of SQL - IOUG Presentation](http://www.slideshare.net/brendanfurey7/dimensional-performance-benchmarking-of-sql) has had significant upgrades made to to allow benchmarking of both DML and DDL (previously it allowed for DML as a pre-query step only, for example to materialise a subquery with indexes).

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
