---
layout: post
title: "Master-Detail Transaction Reconciliation in SQL (MDTM3)"
date: 2012-10-26
migrated: true
group: general-sql
categories: 
  - "analytics"
  - "oracle"
  - "performance"
  - "qsd"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "listagg"
  - "matching"
  - "oracle"
  - "performance-2"
  - "qsd"
  - "reconciliation"
  - "sql"
  - "subquery-factor"
---

This is the final article in a sequence of three on the subject of master-detail transaction matching. In the first article, [Master-Detail Transaction Matching in SQL (MDTM1)](https://brenpatf.github.io/migrated/master-detail-transaction-matching-in-sql/ "Master-Detail Transaction Matching in SQL (MDTM1)"), I described the problem and divided it into two subproblems, the first being to identify all pairs of matching transactions and the second being to reconcile the pairs so that one transaction matches against at most one other transaction. The underlying motivation here comes from the problem of reconciling intra-company credit and debit transactions where fields may need to match directly, or may need to match after some mapping function is applied, including inversion (contra-matching). We have taken a simple prototype problem defined on Oracle's system tables where only matching conditions are specified. It should be straightforward to extend the techniques demonstrated to more general matching conditions (I have done so myself on the real business problem that prompted this analysis).

The first article developed a series of queries to solve the first subproblem using the idea of pre-aggregation of the detail records as a key performance-enhancing feature. This resulted in a best query that used a temporary table and achieved a time variation that was quadratic in the number of master transactions (we kept the numbers of details per master fixed).

The second article, [Holographic Set Matching in SQL (MDTM2)](https://brenpatf.github.io/migrated/holographic-set-matching-in-sql/ "Holographic Set Matching in SQL (MDTM2)"), took the aggregation a step further, using list aggregation to bypass direct detail set matching altogether, and this enabled linear time variation.

In this third article, we take the last, linear-time query and extend it to solve the second subproblem, providing a solution to the overall problem in a single query that shows the same linear-time variation property in our test results. The sample problem will be the same as in the previous article.

## Output Specification
 The output will be ordered first by section, then by group, then by transaction unique identifier, with paired records appearing together using the first transaction for ordering within the group. The sections are defined thus:

- Reconciled pairs - two records for each matching pair, with no transaction appearing in more than one pair
- Matched but unreconciled transactions - transactions that match others but could not be reconciled because their matching partners are all paired off against other transactions
- Unmatched transactions - transactions not matching any other transaction

## Queries
 We'll include the best query from the last article (L2\_SQF), as well as the new query (RECON) that extends it to solve the overall problem.

- L2\_SQF - solves first subproblem by list aggregation without direct detil matching
- RECON - extends L2\_SQF to solve the overall problem using a sequence of query subfactors

The first query will not be described below, as it appeared in the previous article but will be included in the results section for comparison purposes.

### Query Structure Diagram (QSD)
 <img src="/migrated_images/2012/10/Pair-V1.3.jpg" alt="Pair, V1.3" title="Pair, V1.3" />

### Query Text

<div class="scrollbox">

<pre>
Path Total Price Total Value C
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
), dup as (
SELECT t1.owner                  owner_1,
       t1.table_name             table_name_1,
       t2.owner                  owner_2,
       t2.table_name             table_name_2,
       t1.n_det                  n_det
  FROM tab                       t1
  JOIN tab                       t2
    ON t2.lagg                   = t1.lagg
   AND t2.row_id                 > t1.row_id
), btw AS (
SELECT owner_1       owner_1_0,
       table_name_1  table_name_1_0,
       owner_1,
       table_name_1,
       owner_2,
       table_name_2,
       n_det
  FROM dup
 UNION
SELECT owner_1,
       table_name_1,
       owner_2,
       table_name_2,
       owner_1,
       table_name_1,
       n_det
  FROM dup
), grp AS (
SELECT owner_1_0,
       table_name_1_0,
       owner_1,
       table_name_1,
       owner_2,
       table_name_2,
       n_det,
       Least (owner_1 || '/' || table_name_1, Min (owner_2 || '/' || table_name_2)
         OVER (PARTITION BY owner_1, table_name_1)) grp_uid
  FROM btw
), rnk AS (
SELECT owner_1_0,
       table_name_1_0,
       owner_1,
       table_name_1,
       Dense_Rank () OVER  (PARTITION BY grp_uid ORDER BY owner_1, table_name_1) r1,
       owner_2,
       table_name_2,
       Dense_Rank () OVER  (PARTITION BY grp_uid ORDER BY owner_2, table_name_2) r2,
       n_det,
       grp_uid
  FROM grp
), rec AS (
SELECT owner_1_0,
       table_name_1_0,
       owner_1,
       table_name_1,
       owner_2,
       table_name_2,
       n_det,
       grp_uid
  FROM rnk
 WHERE (r2 = r1 + 1 AND Mod (r1, 2) = 1) OR (r1 = r2 + 1 AND Mod (r2, 2) = 1)
), rcu AS (
SELECT owner_1_0,
       table_name_1_0,
       owner_1,
       table_name_1,
       owner_2,
       table_name_2,
       n_det,
       grp_uid
  FROM rec
 UNION  
SELECT owner_1,
       table_name_1,
       owner_1,
       table_name_1,
       NULL,
       NULL,
       n_det,
       grp_uid
  FROM grp
 WHERE (owner_1, table_name_1) NOT IN (SELECT owner_1, table_name_1 FROM rec)
 UNION  
SELECT owner,
       table_name,
       owner,
       table_name,
       NULL,
       NULL,
       n_det,
       NULL
  FROM tab
 WHERE (owner, table_name) NOT IN (SELECT owner_1, table_name_1 FROM btw)
)
SELECT
       owner_1,
       table_name_1,
       owner_2,
       table_name_2,
       n_det,
       grp_uid
  FROM rcu
 ORDER BY grp_uid,
       CASE WHEN grp_uid IS NULL THEN 3 
               WHEN owner_2 IS NULL THEN 2
               ELSE 1
          END,
       owner_1_0,
       table_name_1_0,
       owner_1,
       table_name_1
</pre>
</div>

### How it Works

The query proceeds by ten stages using query subfactors. The first three stages correspond to query L2\_SQF which then has a main query, whereas we now have another six stages before the main query, as shown:

1. Rank the distinct details \[Group by matching fields, then use Row\_Number to rank by same\]
2. Convert ranks to base 128 \[Use Floor() and Chr() functions; uses 2 characters here\]
3. Aggregate detail ranks for master \[Use Listagg on the rank fixed-length string\]
4. Get all matching pairs one-way \[Self-join on matching aggregate and second rowid greater\]
5. Union in the reverse pairs, with sorting column \[Add in records with uids swapped, but keep uid 1 separately for sorting\]
6. Assign grouping field to each pair \[Take minimum of uid 2 over uid 1, or uid 1 if smaller\]
7. Rank each side of pair within its group \[Use Dense\_Rank() over grouping, ordering by uids\]
8. Retain odd-even sequentially ranked pairs \[Retain pairs with ranks (1,2) or (3,4) etc. and the reverses\]
9. Add in unmatched and matched but unreconciled \[3-way union: first the reconciled; then the matched but unreconciled; then unmatched\]
10. Sort by the source uid 1, then the current uid 1 \[Sort key ensures that reconciled pairs stay together within their matching group\]

Notes:

- In our matching-only sample problem, matching transactions form mutually matching sets, whereas for contra-matching, there are pairs of contra-matching sets as discussed in the first article. The grouping subqueries will therefore differ in the latter case, and for example, pairing could be by matching numerical rank within the respective sets
- The final subquery factor could be incorporated in the main query, but I retain it because the benchmarking framework does not support unions in the main query, and CBO optimises it away in any case

## Results
 Both queries were run on the same sample problem as in the previous article. The output comparison gives the output listings for width parameter of 1, which corresponds to the tables and constraints on my v11.2 system copied to the test tables with owner prefix '\_0' added. The timings and statistics are for widths from 1 to 6.

### Output Comparison

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21163&amp;authkey=AAd9EurAwcdwR0o&amp;em=2" frameborder="0" scrolling="no" width="800" height="346"></iframe>

The output file has tabs for the output from both queries, and a tab with examples from each of the three sections for the second. Notice that OEHR\_EMPLOYEES and OEHR\_JOB\_HISTORY form a reconciled pair, with three detail records, while EMPLOYEES and JOB\_HISTORY are unmatched, with four detail records. This is because, as I mentioned in the first article, I have added an extra record to each of the latter tables' detail tables (i.e. foreign key constraints), the extra record being a duplicate of one of the other three (in terms of the matching fields), but a different duplicate in each case. This tests correct handling of duplicates within the detail sets.

### Performance Comparison

Click on the query name in the file below to jump to the execution plan for the largest data point, and click the tabs to see different views on the performance obtained.

<iframe src="https://skydrive.live.com/embed?cid=95EC670EA6AF8ED1&amp;resid=95EC670EA6AF8ED1%21162&amp;authkey=APSELt7W-9-uiKE&amp;em=2" frameborder="0" scrolling="no" width="800" height="346"></iframe>

- The timings in the file above are roughly consistent with linear-time variation with problem size; if anything L2\_SQF appears sublinear, but the times are fairly small and there was some other activity on the PC at the time
- At the largest data point, RECON takes 5 times as much CPU time as L2\_SQF, and 9 times as much elapsed time
- The differences between elapsed and CPU times for RECON are indicative of significant file I/O activity. This shows up in the disk reads and writes summaries on the statistics tab, and in more detail in the Plans tab, and is caused mainly by reading and writing of the subquery factors to and from disk
- The other main significant factor in the increase in times for the RECON query is the additional sorting; see, for example, steps 31 and 32 in the plan. These effects are the result of the additional subquery factors that were needed to achieve the final result

## Conclusion

- This three-part sequence of articles has focussed on a special category of problem within SQL, but has highlighted a range of SQL techniques that are useful generally, including:
    - Subquery factors
    - Temporary tables
    - Analytic functions
    - Set matching by list aggregation
    - Compact storage of unique identifiers by ranking and base-conversion via the Chr() function
- We have also noted different ways of matching sets, including use of the MINUS set operator and the NOT EXISTS construct, and discussed ways of handling issues such as duplication within a set, and directionality of the set operators
- The importance of polynomial order of solution performance for efficiency has been illustrated dramatically
- The final SQL solution provides a good illustration of the power of modern SQL to solve complex problems using set-based logic combined with sequence in a simpler and faster way than the more conventional procedural approach
- The subquery-sequence approach to SQL is well suited to diagrammatic design techniques
- It is often better to solve complex real-world problems by first working with simpler prototypes
