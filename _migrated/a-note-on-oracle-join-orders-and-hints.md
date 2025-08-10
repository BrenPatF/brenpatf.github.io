---
layout: post
migrated: true
title: "A Note on Oracle Join Orders and Hints"
date: 2016-07-17
group: perf-testing
categories: 
  - "performance"
  - "sql"
tags: 
  - "hint"
  - "sql"
  - "tuning"
---

I read an interesting article this week on a company internal blog (thanks, Deepak), which pointed out that many people seem to think that the hint USE\_HASH needs two parameters to specify the hint, when in fact only the _right_ table alias is required. Where more than one alias is given the hint is effectively treated as separate one-alias hints. Here is what the manual says, [Oracle SQL Manual 12.1 - Hints - USE\_HASH](http://docs.oracle.com/database/121/SQLRF/sql_elements006.htm#BABBABGJ):

<img src="/migrated_images/2016/07/use_hash_hint.gif" alt="use_hash_hint" title="use_hash_hint" />

> The USE\_HASH hint instructs the optimizer to join each specified table with another row source using a hash join. For example:
> 
> SELECT /\*+ USE\_HASH(l h) \*/ \* FROM orders h, order\_items l WHERE l.order\_id = h.order\_id AND l.order\_id > 2400;

Observe that in the example given, with two tables, there can be only one join in the chosen plan but both aliases are given in the hint. Only one of the hints can take effect here, but which one? It depends on the join order chosen by the the Cost Based Optimizer (CBO). In the sections on USE\_MERGE and USE\_NL, the manual says:

> Use of the USE\_NL and USE\_MERGE hints is recommended with the LEADING and ORDERED hints. The optimizer uses those hints when the referenced table is forced to be the inner table of a join. The hints are ignored if the referenced table is the outer table.

My colleague linked to an article by Jonathan Lewis that considers join order in relation to hash joins in some detail, [Quiz Night](https://jonathanlewis.wordpress.com/2010/12/10/quiz-night-10/).

> You have NOT defined a hash join completely until you have specified which rowsource should be used as the build table and which as the probe table – so every time you supply the use\_hash() hint for a table, you should also supply the swap\_join\_inputs() hint or the no\_swap\_join\_inputs() hint.

In the article, JL asks how many execution plans are possible for a four-table query where he says he specifies both join order (leading hint) and join method for each of the three joins (use\_hash). The answer is 8, which might be a surprise if you accept the specification at face value, as that would imply that a single plan has been fully specified. The explanation of the apparent paradox is of course given in the quote above. JL goes on to enumerate each possible plan, obtained by different combinations of the swap\_join\_inputs and no\_swap\_join\_inputs hints. He then asserts:

> Note the extreme change in shape and apparent order of tables in the plan. Despite this the join order really is t1 -> t2 -> t3 -> t4 in every case. I’ll give a quick description of the first and last plans to explain this.

The explanation seems plausible, but I can see a possible problem with it. It is possible to generate exactly the same plans using the hint _leading (t2 t1 t3 t4)_ and different combinations of the swap hints. I could then equally plausibly assert that _the join order really is t2 -> t1 -> t3 -> t4 in every case_. Both assertions can't be right can they? The explanation is that we need to consider two levels of ordering, and not just for hash joins. First, I will demonstrate how to get identical plans with different leading hints. I will use Oracle's HR schema, and show the hints necessary to get all hash join plans (with current statistics - I am not fully specifying the plans in general):

```
SELECT *
  FROM employees e
  JOIN departments d
 USING (department_id)
  JOIN locations l
 USING (location_id)
```

You can see that hash joins are only possible between e and d, and between d and l (or rowsets that include the relevant keys).

We can use a notation x.y to mean hash-join y as the right table, using x to form the hash-table, and (x.y) to denote the resulting rowset. Then I believe there are exactly 8 possible hash-join permutations, each of which I was able to hint using leading, use\_hash, swap\_join\_inputs and no\_swap\_join\_input hints as follows:

```
1. leading (l) use_hash (d)
===========================
Combo              SJI
------- ----
(l.d).e  
e.(l.d)            e
(d.l).e            d
e.(d.l)            d, e

2. leading (e)
==============
Combo              SJI    NSJI
------- ---- ----
l.(e.d)
(e.d).l                   d
l.(d.e)            d
(d.e).l            d      l
```

Now, look at the following output where I have got exactly the same plan (the first combo above) using two different leading hints (putting in the first two aliases to be sure). The first has:

```
/*+ leading (l d)  use_hash (d) */
```

The second has:

```
/*+ leading (d l) use_hash (l e) swap_join_inputs (l) */
```

The plans that result, showing identical plan hash value of 2684174912 are:

```
SQL_ID  ds98jqmdbn9ym, child number 0
-------------------------------------
SELECT  /*+ leading (l d)  use_hash (d) GATHER_PLAN_STATISTICS
Lead_l_Hash_d */      *   FROM employees e   JOIN departments d  USING
(department_id)   JOIN locations l  USING (location_id)

Plan hash value: 2684174912

------------------------------------------------------------------------------------------------------------------------
| Id  | Operation           | Name        | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |             |      1 |        |    106 |00:00:00.01 |      25 |       |       |          |
|*  1 |  HASH JOIN          |             |      1 |    106 |    106 |00:00:00.01 |      25 |   796K|   796K| 1241K (0)|
|*  2 |   HASH JOIN         |             |      1 |     27 |     27 |00:00:00.01 |      12 |   835K|   835K| 1122K (0)|
|   3 |    TABLE ACCESS FULL| LOCATIONS   |      1 |     23 |     23 |00:00:00.01 |       6 |       |       |          |
|   4 |    TABLE ACCESS FULL| DEPARTMENTS |      1 |     27 |     27 |00:00:00.01 |       6 |       |       |          |
|   5 |   TABLE ACCESS FULL | EMPLOYEES   |      1 |    107 |    107 |00:00:00.01 |      13 |       |       |          |
------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   2 - access("D"."LOCATION_ID"="L"."LOCATION_ID")

SQL_ID  fcbh8httjtpj3, child number 0
-------------------------------------
SELECT  /*+ leading (d l) use_hash (l e) swap_join_inputs (l)
GATHER_PLAN_STATISTICS Lead_d_Hash_l_e_SJI_l */      *   FROM employees
e   JOIN departments d  USING (department_id)   JOIN locations l  USING
(location_id)

Plan hash value: 2684174912

------------------------------------------------------------------------------------------------------------------------
| Id  | Operation           | Name        | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT    |             |      1 |        |    106 |00:00:00.01 |      25 |       |       |          |
|*  1 |  HASH JOIN          |             |      1 |    106 |    106 |00:00:00.01 |      25 |   796K|   796K| 1240K (0)|
|*  2 |   HASH JOIN         |             |      1 |     27 |     27 |00:00:00.01 |      12 |   835K|   835K| 1109K (0)|
|   3 |    TABLE ACCESS FULL| LOCATIONS   |      1 |     23 |     23 |00:00:00.01 |       6 |       |       |          |
|   4 |    TABLE ACCESS FULL| DEPARTMENTS |      1 |     27 |     27 |00:00:00.01 |       6 |       |       |          |
|   5 |   TABLE ACCESS FULL | EMPLOYEES   |      1 |    107 |    107 |00:00:00.01 |      13 |       |       |          |
------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   2 - access("D"."LOCATION_ID"="L"."LOCATION_ID")
```

So, here we seem to have shown that join order l -> d -> e = join order d -> l -> e, since they have the same plan. I think that to understand this we need to separate out two levels of join order. In the two-table example I took from the manual I stated there could be only one join order in a given plan as there is only one join, meaning by join order which table appears on the left and which on the right.

### Outer-level join order

This can be specified by a number assigned to each table matching the sequence number of the join in which it features. The first two tables therefore both have a sequence number of 1. Another way of looking at it can be found by considering my bracketing notation above. In general, the outer-level join sequence number for an n-table query, OS(i) = n - number of brackets enclosing table i - 1

For (l.d).e, l and d have OS = 1 and e has OS = 2. The same is true for (d.l).e.

It is in this sense of join order that JL's queries all have the same join order, and that a single plan can arise from different outer-level join orders, once inner-join order is factored in.

### Inner-level join order

This is simply the side on which a table is joined to a rowset in a given join, say 1 for the left side, and 2 for the right side in a hash join.

Note that the leading hint treats both levels of join order, but behaves differently between hash joins and other types of join.

### Leading hint in hash join

- Determines the outer-level join order
- Defaults the inner-level join order
- Swap\_join\_inputs operation overrides the inner-level default join order

### Leading hint in other types of join

- Determines the outer-level join order
- Determines the inner-level join order on the first two tables
- Inner-level order not applicable after first two tables

**See also a later article I wrote:** [DIMBEN 5: Hash Join Options in SQL for Fixed-Depth Hierarchies](/dimben/dimben-5/)

You may also be interested to read a comment by the author of the article I cited above, Jonathan Lewis, on my Wordpress blog, where this appeared originally: [A Note on Oracle Join Orders and Hints](http://aprogrammerwrites.eu/?p=1758)