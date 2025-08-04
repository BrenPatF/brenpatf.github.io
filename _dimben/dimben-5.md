---
layout: post
title: "DIMBEN 5: Hash Join Options in SQL for Fixed-Depth Hierarchies"
date: 2017-03-05
migrated: true
dimben_prev: /dimben/dimben-4/
dimben_next: /dimben/dimben-6/
categories: 
  - "performance"
  - "sql"
  - "testing"
tags: 
  - "oracle"
  - "performance-2"
  - "swap_join_inputs"
  - "tuning"
  - "use_hash"
---
#### Part 5 in a series on: Dimensional Benchmarking of SQL Performance

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

In a recent article, [Dimensional Benchmarking of SQL for Fixed-Depth Hierarchies](/dimben/dimben-4/) I compared the performance characteristics of three traversal methods, two recursive and one non-recursive, using my own benchmarking package, available on GitHub and described on the [index page](/migrated/dimben-series-index/) for this series of articles, on a test problem of a fixed level organization structure hierarchy, with 5 levels for performance testing and 3 levels for functional testing.

> It is interesting to note that all joins in the execution plan are hash joins, and in the sequence you would expect. The first three are in the default join 'sub-order' that defines whether the joined table or the prior rowset (the default) is used to form the hash table, while the last two are in the reverse order, corresponding to the swap\_join\_inputs hint. I wrote a short note on that subject, A Note on Oracle Join Orders and Hints, last year, and have now written an article using the largest data point in the current problem to explore performance variation across the possible sub-orders.

This article uses the dimensional performance framework to test all 32 possible combinations in the given sequence of tables.

## Notation and Hints

### Dot-Bracket Code

In the note mentioned above I found a concise notation for representing a sequence of hash joins. A bracketted expression represents a rowset, and a '.' represents a hash-join between a table and a rowset. As I noted there the standard idea of join order does not fully specify the hash join, and I found it convenient to use the idea of outer-level order - the sequence in which tables are joined to each other - and 'inner-level order' - which of the two factors goes on the 'left' side of the join, to form the hash table.

As Oracle documentation refers to the idea of swapping join inputs in a hash join to mean making the table the hash table, i.e. the left side of the join, we can think of that as being the non-default inner order.

When joining table c to a prior rowset, say (a.b), the default with (a.b) forming the hash table on the left is represented by:

(a.b).c

This can be forced by the hints use\_hash(c) and no\_swap\_join\_inputs(c).

In our example, defaulting each inner-join order is represented by:

((((s1.o1).s2).s3).s4).o5

The non-default is represented by:

c.(a.b)

This can be forced by the hints use\_hash(c) and swap\_join\_inputs(c).

In our example, undefaulting each inner-join order is represented by:

o5.(s4.(s3.(s2.(o1.s1))))

### Binary Code

In enumerating all combinations, it is convenient to represent a combination by 5 0/1 digits. I use 0 to mean the default and 1 to mean the non-default. Therefore

- 00000 is equivalent to ((((s1.o1).s2).s3).s4).o5
- 11111 is equivalent to o5.(s4.(s3.(s2.(o1.s1))))

### Hints

To ensure that the desired options are followed, we need three types of hint:

- leading - to guarentee sequence of outer-level ordering
- use\_hash - to ensure hash joins
- swap\_join\_inputs/no\_swap\_join\_inputs - to force swap or no swap on the inner-order

As mentioned in my earlier note, the inner-order on the first two tables can only be forced by the leading hint.

## Graph of CPU, ElapsedTime, and disk\_writes by Plan

<img src="/migrated_images/2017/03/Plan-Statistics.png" alt="Statistics by Plan" title="Statistics by Plan" />

From the graph we observe that a step-change in performance occurs between plan 16 and 17, with the second half of the plans consuming around half as much elapsed time as the first half, and that this correlates very strongly with disk accesses. There is also a roughly 1 second increase in CPU time in the second half, which is no doubt a knock-on effect of the disk access processing.

It's clear that the final join to orgs with alias o5 is much faster when applied in the 'swapped' inner-order. It is well known that hash joins become much slower when the size of the hash table becomes too large for it to fit into memory and it has to spill to disk, and this is presumably what is happening here. We can see in my earlier article that the CBO, left to itself, in fact chooses one of the better plans.

## Execution Plans for H01, H16, H17, H32

### H01\_QRY

- Dot-Bracket Code: ((((s1.o1).s2).s3).s4).o5
- Binary Code: 00000
- Hints: leading(s1 o1 s2 s3 s4) use\_hash(o1 s2 s3 s4 o5) no\_swap\_join\_inputs(s2) no\_swap\_join\_inputs(s3) no\_swap\_join\_inputs(s4) no\_swap\_join\_inputs(o5)

```
Plan hash value: 892313792

----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation               | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |               |      1 |        |   3330K|00:00:19.68 |     782 |  96903 |  96872 |       |       |          |         |
|   1 |  SORT ORDER BY          |               |      1 |   3905K|   3330K|00:00:19.68 |     782 |  96903 |  96872 |   454M|  7027K|  163M (1)|     400K|
|*  2 |   HASH JOIN             |               |      1 |   3905K|   3330K|00:00:13.64 |     778 |  45725 |  45694 |   458M|    17M|  137M (1)|     372K|
|*  3 |    HASH JOIN            |               |      1 |   3905K|   3330K|00:00:00.38 |     771 |      0 |      0 |  4258K|  1196K| 6842K (0)|         |
|*  4 |     HASH JOIN           |               |      1 |    133K|  30520 |00:00:00.01 |     580 |      0 |      0 |   973K|   973K| 1314K (0)|         |
|*  5 |      HASH JOIN          |               |      1 |   4575 |    780 |00:00:00.01 |     389 |      0 |      0 |  1126K|  1126K| 1250K (0)|         |
|*  6 |       HASH JOIN         |               |      1 |     56 |     56 |00:00:00.01 |     198 |      0 |      0 |  1214K|  1214K| 1130K (0)|         |
|*  7 |        TABLE ACCESS FULL| ORG_STRUCTURE |      1 |     56 |     56 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|   8 |        TABLE ACCESS FULL| ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
|   9 |       TABLE ACCESS FULL | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|  10 |      TABLE ACCESS FULL  | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|  11 |     TABLE ACCESS FULL   | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|  12 |    TABLE ACCESS FULL    | ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("O5"."ID"="S4"."CHILD_ORG_ID")
   3 - access("S4"."ORG_ID"="S3"."CHILD_ORG_ID")
   4 - access("S3"."ORG_ID"="S2"."CHILD_ORG_ID")
   5 - access("S2"."ORG_ID"="S1"."CHILD_ORG_ID")
   6 - access("O1"."ID"="S1"."ORG_ID")
   7 - filter("S1"."STRUCT_LEVEL"=1)
```

### H16\_QRY

- Dot-Bracket Code: (s4.(s3.(s2.(o1.s1)))).o5
- Binary Code: 11110
- Hints: leading(o1 s1 s2 s3 s4) use\_hash(s1 s2 s3 s4 o5) swap\_join\_inputs(s2) swap\_join\_inputs(s3) swap\_join\_inputs(s4) no\_swap\_join\_inputs(o5)

```
Plan hash value: 1359118369

----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation               | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |               |      1 |        |   3330K|00:00:31.46 |     782 |  96902 |  96871 |       |       |          |         |
|   1 |  SORT ORDER BY          |               |      1 |   3905K|   3330K|00:00:31.46 |     782 |  96902 |  96871 |   453M|  7021K|  163M (1)|     400K|
|*  2 |   HASH JOIN             |               |      1 |   3905K|   3330K|00:00:24.45 |     778 |  45725 |  45694 |   458M|    17M|  135M (1)|     371K|
|*  3 |    HASH JOIN            |               |      1 |   3905K|   3330K|00:00:00.49 |     771 |      0 |      0 |  2764K|  1580K| 4059K (0)|         |
|   4 |     TABLE ACCESS FULL   | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|*  5 |     HASH JOIN           |               |      1 |    133K|  30520 |00:00:00.01 |     580 |      0 |      0 |  2764K|  1580K| 4104K (0)|         |
|   6 |      TABLE ACCESS FULL  | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|*  7 |      HASH JOIN          |               |      1 |   4575 |    780 |00:00:00.01 |     389 |      0 |      0 |  2764K|  1580K| 4104K (0)|         |
|   8 |       TABLE ACCESS FULL | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|*  9 |       HASH JOIN         |               |      1 |     56 |     56 |00:00:00.01 |     198 |      0 |      0 |  1519K|  1519K| 1546K (0)|         |
|  10 |        TABLE ACCESS FULL| ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
|* 11 |        TABLE ACCESS FULL| ORG_STRUCTURE |      1 |     56 |     56 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|  12 |    TABLE ACCESS FULL    | ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("O5"."ID"="S4"."CHILD_ORG_ID")
   3 - access("S4"."ORG_ID"="S3"."CHILD_ORG_ID")
   5 - access("S3"."ORG_ID"="S2"."CHILD_ORG_ID")
   7 - access("S2"."ORG_ID"="S1"."CHILD_ORG_ID")
   9 - access("O1"."ID"="S1"."ORG_ID")
  11 - filter("S1"."STRUCT_LEVEL"=1)
```

### H17\_QRY

- Dot-Bracket Code: o5.((((s1.o1).s2).s3).s4)
- Binary Code: 00001
- Hints: leading(s1 o1 s2 s3 s4) use\_hash(o1 s2 s3 s4 o5) no\_swap\_join\_inputs(s2) no\_swap\_join\_inputs(s3) no\_swap\_join\_inputs(s4) swap\_join\_inputs(o5)

```
Plan hash value: 2901541859

----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation               | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |               |      1 |        |   3330K|00:00:12.65 |     782 |  51181 |  51181 |       |       |          |         |
|   1 |  SORT ORDER BY          |               |      1 |   3905K|   3330K|00:00:12.65 |     782 |  51181 |  51181 |   454M|  7031K|  163M (1)|     400K|
|*  2 |   HASH JOIN             |               |      1 |   3905K|   3330K|00:00:02.58 |     778 |      0 |      0 |  1519K|  1519K| 1435K (0)|         |
|   3 |    TABLE ACCESS FULL    | ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
|*  4 |    HASH JOIN            |               |      1 |   3905K|   3330K|00:00:00.34 |     771 |      0 |      0 |  4258K|  1196K| 6851K (0)|         |
|*  5 |     HASH JOIN           |               |      1 |    133K|  30520 |00:00:00.01 |     580 |      0 |      0 |   973K|   973K| 1345K (0)|         |
|*  6 |      HASH JOIN          |               |      1 |   4575 |    780 |00:00:00.01 |     389 |      0 |      0 |  1126K|  1126K| 1260K (0)|         |
|*  7 |       HASH JOIN         |               |      1 |     56 |     56 |00:00:00.01 |     198 |      0 |      0 |  1214K|  1214K| 1138K (0)|         |
|*  8 |        TABLE ACCESS FULL| ORG_STRUCTURE |      1 |     56 |     56 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|   9 |        TABLE ACCESS FULL| ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
|  10 |       TABLE ACCESS FULL | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|  11 |      TABLE ACCESS FULL  | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|  12 |     TABLE ACCESS FULL   | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("O5"."ID"="S4"."CHILD_ORG_ID")
   4 - access("S4"."ORG_ID"="S3"."CHILD_ORG_ID")
   5 - access("S3"."ORG_ID"="S2"."CHILD_ORG_ID")
   6 - access("S2"."ORG_ID"="S1"."CHILD_ORG_ID")
   7 - access("O1"."ID"="S1"."ORG_ID")
   8 - filter("S1"."STRUCT_LEVEL"=1)
```

### H32\_QRY

- Dot-Bracket Code: o5.(s4.(s3.(s2.(o1.s1))))
- Binary Code: 11111
- Hints: leading(o1 s1 s2 s3 s4) use\_hash(s1 s2 s3 s4 o5) swap\_join\_inputs(s2) swap\_join\_inputs(s3) swap\_join\_inputs(s4) swap\_join\_inputs(o5)

```
Plan hash value: 1006677044

----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation               | Name          | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |               |      1 |        |   3330K|00:00:13.57 |     782 |  51173 |  51173 |       |       |          |         |
|   1 |  SORT ORDER BY          |               |      1 |   3905K|   3330K|00:00:13.57 |     782 |  51173 |  51173 |   454M|  7030K|  163M (1)|     400K|
|*  2 |   HASH JOIN             |               |      1 |   3905K|   3330K|00:00:01.30 |     778 |      0 |      0 |  1519K|  1519K| 1563K (0)|         |
|   3 |    TABLE ACCESS FULL    | ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
|*  4 |    HASH JOIN            |               |      1 |   3905K|   3330K|00:00:00.29 |     771 |      0 |      0 |  2764K|  1580K| 4085K (0)|         |
|   5 |     TABLE ACCESS FULL   | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|*  6 |     HASH JOIN           |               |      1 |    133K|  30520 |00:00:00.01 |     580 |      0 |      0 |  2764K|  1580K| 4102K (0)|         |
|   7 |      TABLE ACCESS FULL  | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|*  8 |      HASH JOIN          |               |      1 |   4575 |    780 |00:00:00.01 |     389 |      0 |      0 |  2764K|  1580K| 4102K (0)|         |
|   9 |       TABLE ACCESS FULL | ORG_STRUCTURE |      1 |  27288 |  27288 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
|* 10 |       HASH JOIN         |               |      1 |     56 |     56 |00:00:00.01 |     198 |      0 |      0 |  1519K|  1519K| 1547K (0)|         |
|  11 |        TABLE ACCESS FULL| ORGS          |      1 |    944 |    944 |00:00:00.01 |       7 |      0 |      0 |       |       |          |         |
|* 12 |        TABLE ACCESS FULL| ORG_STRUCTURE |      1 |     56 |     56 |00:00:00.01 |     191 |      0 |      0 |       |       |          |         |
----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("O5"."ID"="S4"."CHILD_ORG_ID")
   4 - access("S4"."ORG_ID"="S3"."CHILD_ORG_ID")
   6 - access("S3"."ORG_ID"="S2"."CHILD_ORG_ID")
   8 - access("S2"."ORG_ID"="S1"."CHILD_ORG_ID")
  10 - access("O1"."ID"="S1"."ORG_ID")
  12 - filter("S1"."STRUCT_LEVEL"=1)
```

The output log is in [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/benchmarking-of-hash-join-options-in-sql-for-fixed-depth-hierarchies).

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
