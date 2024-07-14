---
layout: post
title:  "OPICO 2: SQL for Item Sequence GenerationX"
date:   2024-07-01 06:00:00 +0100
tags:   ["oracle", "optimization", "combination", "knapsack", "sql"]
---
### Part 2 in a series on: Optimization Problems with Items and Categories in Oracle
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

<blockquote>The knapsack problem is a problem in combinatorial optimization: Given a set of items, each with a weight and a value, determine the number of each item to include in a collection so that the total weight is less than or equal to a given limit and the total value is as large as possible.</blockquote>

- [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem)

The knapsack problem and many other problems in combinatorial optimization require the selection of a subset of items to maximize an objective function subject to constraints. A common approach to solving these problems algorithmically involves recursively generating sequences of items of increasing length in a search for the best subset that meets the constraints.

I applied this kind of approach using SQL for a number of problems, starting in January 2013 with [A Simple SQL Solution for the Knapsack Problem (SKP-1)](https://brenpatf.github.io/560), and I wrote a summary article, [Knapsacks and Networks in SQL](https://brenpatf.github.io/2232), in December 2017 when I put the code onto GitHub, [sql_demos - Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos).

This is the second in a series of eight articles that aim to provide a more formal treatment of  algorithms for item sequence generation and optimization, together with practical implementations, examples and unit testing in SQL and PL/SQL.

#### List of Articles
- [OPICO 1: Algorithms for Item Sequence Generation](https://brenpatf.github.io/2024/06/30/opico-1-algorithms-for-generation.html)
- <strong>[OPICO 2: SQL for Item Sequence Generation](https://brenpatf.github.io/2024/07/07/opico-2-sql_for_item_sequence_generation.html)</strong>
- [OPICO 3: Algorithms for Item/Category Optimization]() [Available: 14 July 2024]
- [OPICO 4: Recursive SQL for Item/Category Optimization]() [Available: 21 July 2024]
- [OPICO 5: Tuning Recursive SQL for Item/Category Optimization]() [Available: 28 July 2024]
- [OPICO 6: Mixed SQL and PL/SQL Methods for Item/Category Optimization]() [Available: 4 August 2024]
- [OPICO 7: Verification]() [Available: 11 August 2024]
- [OPICO 8: Automation]() [Available: 18 August 2024]

#### GitHub (see README for references)
- [Optimization Problems with Items and Categories in Oracle](https://github.com/BrenPatF/item_category_optimization_oracle)

#### Twitter
- [Thread with Short Recordings](https://x.com/BrenPatF/status/1807642673748033675)

In the first article I reviewed methods for recursive generation of the item sequences in a generic way that is not specific to SQL or any programming language.

In the current article, I go on to explain how to use SQL to implement the algorithms described generically in the previous one. First, I discuss the structure of SQL for recursion in Oracle, with particular reference to item sequence generation. Next, I show the SQL for generating sequences of each of the four sequence types described in the first article, using the simple case of selecting 3 items from 6 as an example.

<img src="pics/database-schema-1895779_1280.png" style="width: 100%; max-width: 100%;" /><br />
[Image by <a href="https://pixabay.com/users/mcmurryjulie-2375405/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1895779">mcmurryjulie</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1895779">Pixabay</a>]
# Contents
[&darr; 1 Item Sequence Types](#1-item-sequence-types)<br />
[&darr; 2 Recursive SQL and Item Sequences](#2-recursive-sql-and-item-sequences)<br />
[&darr; 3 Item Sequence Generation by Pure SQL](#3-item-sequence-generation-by-pure-sql)<br />
[&darr; 4 Item Sequence Generation by Mixed SQL and PL/SQL](#4-item-sequence-generation-by-mixed-sql-and-plsql)<br />
[&darr; 5 Conclusion](#5-conclusion)<br />

## 1 Item Sequence Types
[&uarr; Contents](#contents)<br />

We discussed recursion methods for four types of sequence in the first article in purely algorithmic terms. In this article, we'll look at SQL and PL/SQL methods for generating the sequences. Initially, we'll take a simple example of sequences of length 2 from 3 items to illustrate the different types of sequence. We can depict the solution sets for these examples below:

<img src="png/2-3_sequences.png">

Later we'll use a larger problem size of sequences of length 3 from 6 items, to illustrate the SQL.

## 2 Recursive SQL and Item Sequences
[&uarr; Contents](#contents)<br />
[&darr; Recursive Query Structure](#recursive-query-structure)<br />
[&darr; Recursion Data Flow](#recursion-data-flow)<br />
[&darr; Storage of Item Sequences as Paths](#storage-of-item-sequences-as-paths)<br />

Here is the Oracle documentation on [Recursive Subquery Factoring](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html#GUID-CFA006CA-6FF1-4972-821E-6996142A51C6__I2077142), and here is my own article from February 2020 on [Analytic and Recursive SQL by Example](https://brenpatf.github.io/2702#rsf).

In order to generate item sequences we can take advantage of the ability of recursive subquery factors to carry forward item paths as an expression concatenating an additional item at each iteration. The query structure has the recursive factor called by a main query, where the recursive factor has the standard structure of a union between an anchor branch and a recursive branch that selects from the recursive factor itself.

The anchor branch here selects '0' from dual as a kind of dummy item, with the path initialized to null. The recursive branch adds the next items on to the path for each record in the previous iteration, with join condition according to the type of sequence sought. The where condition terminates iteration when the  desired path length is reached.

Only the main query has access to the records generated from all iterations, and in our case we need to restrict to the last iteration, which has the full length sequences in the paths. Note that we include the iteration number, as level, in the select list as well as in the where clause and this ensures that a cycle is not possible, even in the sequence types where repeated items are allowed.

### Recursive Query Structure
[&uarr; 2 Recursive SQL and Item Sequences](#2-recursive-sql-and-item-sequences)<br />

<img src="png/SQL for Item Sequence Recursion, v1.2 - QSD.png">

### Recursion Data Flow
[&uarr; 2 Recursive SQL and Item Sequences](#2-recursive-sql-and-item-sequences)<br />

In the following diagram we see how the recursion process works in the case of the simple example of finding all 3-item combination from 6 items.

<img src="png/SQL for Item Sequence Recursion, v1.0 - RDF.png">

### Storage of Item Sequences as Paths
[&uarr; 2 Recursive SQL and Item Sequences](#2-recursive-sql-and-item-sequences)<br />
[&darr; Can we Use Nested Tables instead?](#can-we-use-nested-tables-instead)<br />

The item sequences may be stored as concatenations of item identifiers, or item paths, using either delimiters between items, or fixed length identifiers. In this case, if we want the individual items returned, we would need to split the path strings into sets of records: This is easy enough to do using standard row generation and string-splitting techniques, as we'll see later.

#### Can we Use Nested Tables instead?
[&uarr; Storage of Item Sequences as Paths](#storage-of-item-sequences-as-paths)<br />

It might be thought that Oracle's nested table type could be used to store the item sequences in an array format, and that this might be a more elegant approach. However, the syntax for collecting and then splitting the items again does not seem particularly preferable; more importantly, in Oracle 19.3 I have found that, although it appears to work at first glance, the results are not correct in general, with Oracle finding cycles where none exist. This is demonstrated later.

## 3 Item Sequence Generation by Pure SQL
[&uarr; Contents](#contents)<br />
[&darr; MP: Multiset Permutation [Items may repeat, order matters]](#mp-multiset-permutation-items-may-repeat-order-matters)<br />
[&darr; MC: Multiset Combination [Items may repeat, order does not matter]](#mc-multiset-combination-items-may-repeat-order-does-not-matter)<br />
[&darr; SP: Set Permutation [Items may not repeat, order matters]](#sp-set-permutation-items-may-not-repeat-order-matters)<br />
[&darr; SC: Set Combination [Items may not repeat, order does not matter]](#sc-set-combination-items-may-not-repeat-order-does-not-matter)<br />
[&darr; SC Using Nested Tables: Set Combination [Items may not repeat, order does not matter]](#sc-using-nested-tables-set-combination-items-may-not-repeat-order-does-not-matter)<br />

In this section we display the SQL queries for generating all item sequences of a given size, :SEQ_SIZE, from a set of items stored in an items table, for each of the four types of sequence discussed in the first article in the series.

The output is listed for a simple example of selecting 3 items from 6, with items having price and value. The total prices and values are accumulated through the recursion and listed here. For simplicity, the queries print the paths as a string without splitting into records.

Script on GitHub: item_seqs.sql
### MP: Multiset Permutation [Items may repeat, order matters]
[&uarr; 3 Item Sequence Generation by Pure SQL](#3-item-sequence-generation-by-pure-sql)<br />
[&darr; SQL: All permutations of :SEQ_SIZE items from set, allowing repetition](#sql-all-permutations-of-seq_size-items-from-set-allowing-repetition)<br />
[&darr; Join Condition: All permutations of :SEQ_SIZE items from set, allowing repetition](#join-condition-all-permutations-of-seq_size-items-from-set-allowing-repetition)<br />
[&darr; Output: All 216 permutations of 3 items from 6, allowing repetition](#output-all-216-permutations-of-3-items-from-6-allowing-repetition)<br />

#### SQL: All permutations of :SEQ_SIZE items from set, allowing repetition
[&uarr; MP: Multiset Permutation [Items may repeat, order matters]](#mp-multiset-permutation-items-may-repeat-order-matters)<br />

```sql
WITH tree_walk(id, lev, tot_price, tot_value, path) AS (
    SELECT '0', 0, 0, 0, '' path
      FROM DUAL
     UNION ALL
    SELECT itm.id,
           trw.lev + 1,
           trw.tot_price + itm.item_price,
           trw.tot_value + itm.item_value,
           trw.path || itm.id
      FROM tree_walk trw
     CROSS JOIN items itm
     WHERE trw.lev < :SEQ_SIZE
) CYCLE lev SET c TO '*' DEFAULT ' '
SELECT path, tot_price, tot_value, cf
  FROM tree_walk
 WHERE lev = :SEQ_SIZE
```

#### Join Condition: All permutations of :SEQ_SIZE items from set, allowing repetition
[&uarr; MP: Multiset Permutation [Items may repeat, order matters]](#mp-multiset-permutation-items-may-repeat-order-matters)<br />

For this case we want to simply join all items with all path records from the previous iteration, which is implemented by a CROSS JOIN in the recursive member, with the termination condition being in the WHERE clause. Here's what the Oracle documentation says about cycles:
<blockquote>If you omit the CYCLE clause, then the recursive WITH clause returns an error if cycles are discovered. In this case, a row forms a cycle if one of its ancestor rows has the same values for all the columns in the column alias list for query_name that are referenced in the WHERE clause of the recursive member.</blockquote>
As noted earlier, including the iteration number (lev) in the SELECT list ensures that there can't be any real cycles, so based on this quote we should not need the CYCLE clause. However, omitting it does in fact lead to the error: 'ORA-32044: cycle detected while executing recursive WITH query' in Oracle v19.3. If we replace the CROSS JOIN with an inner JOIN with the WHERE becoming an ON then the CYCLE clause can indeed be omitted without error.<br /><br />

The number of sequences of type MP of size r from a set of n items = <img src="png/n_r.png">. For example, when n = 6 and r = 3, the number of MP sequences is:
<img src="png/6^3.png">
#### Output: All 216 permutations of 3 items from 6, allowing repetition
[&uarr; MP: Multiset Permutation [Items may repeat, order matters]](#mp-multiset-permutation-items-may-repeat-order-matters)<br />

<div class="scrollbox">
<pre>
Path Total Price Total Value C
---- ----------- ----------- -
111            3           3
112            4           4
113            5           5
114            3           8
115            4           6
116            5           4
121            4           4
122            5           5
123            6           6
124            4           9
125            5           7
126            6           5
131            5           5
132            6           6
133            7           7
134            5          10
135            6           8
136            7           6
141            3           8
142            4           9
143            5          10
144            3          13
145            4          11
146            5           9
151            4           6
152            5           7
153            6           8
154            4          11
155            5           9
156            6           7
161            5           4
162            6           5
163            7           6
164            5           9
165            6           7
166            7           5
211            4           4
212            5           5
213            6           6
214            4           9
215            5           7
216            6           5
221            5           5
222            6           6
223            7           7
224            5          10
225            6           8
226            7           6
231            6           6
232            7           7
233            8           8
234            6          11
235            7           9
236            8           7
241            4           9
242            5          10
243            6          11
244            4          14
245            5          12
246            6          10
251            5           7
252            6           8
253            7           9
254            5          12
255            6          10
256            7           8
261            6           5
262            7           6
263            8           7
264            6          10
265            7           8
266            8           6
311            5           5
312            6           6
313            7           7
314            5          10
315            6           8
316            7           6
321            6           6
322            7           7
323            8           8
324            6          11
325            7           9
326            8           7
331            7           7
332            8           8
333            9           9
334            7          12
335            8          10
336            9           8
341            5          10
342            6          11
343            7          12
344            5          15
345            6          13
346            7          11
351            6           8
352            7           9
353            8          10
354            6          13
355            7          11
356            8           9
361            7           6
362            8           7
363            9           8
364            7          11
365            8           9
366            9           7
411            3           8
412            4           9
413            5          10
414            3          13
415            4          11
416            5           9
421            4           9
422            5          10
423            6          11
424            4          14
425            5          12
426            6          10
431            5          10
432            6          11
433            7          12
434            5          15
435            6          13
436            7          11
441            3          13
442            4          14
443            5          15
444            3          18
445            4          16
446            5          14
451            4          11
452            5          12
453            6          13
454            4          16
455            5          14
456            6          12
461            5           9
462            6          10
463            7          11
464            5          14
465            6          12
466            7          10
511            4           6
512            5           7
513            6           8
514            4          11
515            5           9
516            6           7
521            5           7
522            6           8
523            7           9
524            5          12
525            6          10
526            7           8
531            6           8
532            7           9
533            8          10
534            6          13
535            7          11
536            8           9
541            4          11
542            5          12
543            6          13
544            4          16
545            5          14
546            6          12
551            5           9
552            6          10
553            7          11
554            5          14
555            6          12
556            7          10
561            6           7
562            7           8
563            8           9
564            6          12
565            7          10
566            8           8
611            5           4
612            6           5
613            7           6
614            5           9
615            6           7
616            7           5
621            6           5
622            7           6
623            8           7
624            6          10
625            7           8
626            8           6
631            7           6
632            8           7
633            9           8
634            7          11
635            8           9
636            9           7
641            5           9
642            6          10
643            7          11
644            5          14
645            6          12
646            7          10
651            6           7
652            7           8
653            8           9
654            6          12
655            7          10
656            8           8
661            7           5
662            8           6
663            9           7
664            7          10
665            8           8
666            9           6

216 rows selected.
</pre>
</div>

### MC: Multiset Combination [Items may repeat, order does not matter]
[&uarr; 3 Item Sequence Generation by Pure SQL](#3-item-sequence-generation-by-pure-sql)<br />
[&darr; SQL: All combinations of :SEQ_SIZE items from set, allowing repetition](#sql-all-combinations-of-seq_size-items-from-set-allowing-repetition)<br />
[&darr; Join Condition: All combinations of :SEQ_SIZE items from set, allowing repetition](#join-condition-all-combinations-of-seq_size-items-from-set-allowing-repetition)<br />
[&darr; Output: All 56 combinations of 3 items from 6, allowing repetition](#output-all-56-combinations-of-3-items-from-6-allowing-repetition)<br />

#### SQL: All combinations of :SEQ_SIZE items from set, allowing repetition
[&uarr; MC: Multiset Combination [Items may repeat, order does not matter]](#mc-multiset-combination-items-may-repeat-order-does-not-matter)<br />

```sql
WITH tree_walk(id, lev, tot_price, tot_value, path) AS (
    SELECT '0', 0, 0, 0, '' path
      FROM DUAL
     UNION ALL
    SELECT itm.id,
           trw.lev + 1,
           trw.tot_price + itm.item_price,
           trw.tot_value + itm.item_value,
           trw.path || itm.id
      FROM tree_walk trw
      JOIN items itm
        ON itm.id >= trw.id
       AND trw.lev < :SEQ_SIZE
)
SELECT path, tot_price, tot_value
  FROM tree_walk
 WHERE lev = :SEQ_SIZE
```

#### Join Condition: All combinations of :SEQ_SIZE items from set, allowing repetition
[&uarr; MC: Multiset Combination [Items may repeat, order does not matter]](#mc-multiset-combination-items-may-repeat-order-does-not-matter)<br />

```sql
        ON itm.id >= trw.id
```

In this case order is not significant, so we avoid duplication of equivalent sequences by joining items only of equal or higher rank. The same CYCLE issue occurs as in the previous query if we make the second JOIN condition
```sql
       AND trw.lev < :SEQ_SIZE
```

into a WHERE clause.

#### Output: All 56 combinations of 3 items from 6, allowing repetition
[&uarr; MC: Multiset Combination [Items may repeat, order does not matter]](#mc-multiset-combination-items-may-repeat-order-does-not-matter)<br />

The number of sequences of type MC of size r from a set of n items is given by the number of combinations of r items from a set of (n+r-1) items, which, using the combinatorial formula for choosing r items from (n+r-1) is:

<img src="png/nr1Cr.png">

This is not so obvious, and is usually demonstrated by means of a [Stars and bars](https://en.wikipedia.org/wiki/Stars_and_bars_\(combinatorics\)) diagram.

For example, when n = 6 and r = 3, the number of MC sequences is:

<img src="png/8C3.png">

<div class="scrollbox"><pre>
Path Total Price Total Value
---- ----------- -----------
111            3           3
112            4           4
113            5           5
114            3           8
115            4           6
116            5           4
122            5           5
123            6           6
124            4           9
125            5           7
126            6           5
133            7           7
134            5          10
135            6           8
136            7           6
144            3          13
145            4          11
146            5           9
155            5           9
156            6           7
166            7           5
222            6           6
223            7           7
224            5          10
225            6           8
226            7           6
233            8           8
234            6          11
235            7           9
236            8           7
244            4          14
245            5          12
246            6          10
255            6          10
256            7           8
266            8           6
333            9           9
334            7          12
335            8          10
336            9           8
344            5          15
345            6          13
346            7          11
355            7          11
356            8           9
366            9           7
444            3          18
445            4          16
446            5          14
455            5          14
456            6          12
466            7          10
555            6          12
556            7          10
566            8           8
666            9           6

56 rows selected.
</pre></div>

### SP: Set Permutation [Items may not repeat, order matters]
[&uarr; 3 Item Sequence Generation by Pure SQL](#3-item-sequence-generation-by-pure-sql)<br />
[&darr; SQL: All permutations of :SEQ_SIZE items from set](#sql-all-permutations-of-seq_size-items-from-set)<br />
[&darr; Join Condition: All permutations of :SEQ_SIZE items from set](#join-condition-all-permutations-of-seq_size-items-from-set)<br />
[&darr; Output: All 120 permutations of 3 items from 6](#output-all-120-permutations-of-3-items-from-6)<br />

#### SQL: All permutations of :SEQ_SIZE items from set
[&uarr; SP: Set Permutation [Items may not repeat, order matters]](#sp-set-permutation-items-may-not-repeat-order-matters)<br />

```sql
WITH tree_walk(id, lev, tot_price, tot_value, path) AS (
    SELECT '0', 0, 0, 0, '' path
      FROM DUAL
     UNION ALL
    SELECT itm.id,
           trw.lev + 1,
           trw.tot_price + itm.item_price,
           trw.tot_value + itm.item_value,
           trw.path || itm.id
      FROM tree_walk trw
      JOIN items itm
        ON Nvl(trw.path,'x') NOT LIKE '%' || itm.id || '%'
     WHERE trw.lev < :SEQ_SIZE
)
SELECT path, tot_price, tot_value
  FROM tree_walk
 WHERE lev = :SEQ_SIZE
```

#### Join Condition: All permutations of :SEQ_SIZE items from set
[&uarr; SP: Set Permutation [Items may not repeat, order matters]](#sp-set-permutation-items-may-not-repeat-order-matters)<br />

<pre>
        ON Nvl(trw.path,'x') NOT LIKE '%' || itm.id || '%'
</pre>
In this case order is significant, and item repetition is not allowed, so we need to exclude duplicate items and cannot do so as simply as in the case of combinations where we can take items in order. The path string is searched using NOT LIKE with wildcards to ensure the item is not already there. In my simple example the item ids are 1-digit; for larger, more realistic data sets, you would just include delimiters in the search.

The CYCLE issue that we encountered in the multiset sequence type does not occur here when we keep the termination condition as the WHERE clause
```sql
       WHERE trw.lev < :SEQ_SIZE
```

#### Output: All 120 permutations of 3 items from 6
[&uarr; SP: Set Permutation [Items may not repeat, order matters]](#sp-set-permutation-items-may-not-repeat-order-matters)<br />

The number of sequences of type SP of size r from a set n items is given by the usual permutation formula:

<img src="png/nPr.png">

For example, when n = 6 and r = 3, the number of SP sequences is:

<img src="png/6P3.png">

<div class="scrollbox"><pre>
Path Total Price Total Value
---- ----------- -----------
123            6           6
124            4           9
125            5           7
126            6           5
132            6           6
134            5          10
135            6           8
136            7           6
142            4           9
143            5          10
145            4          11
146            5           9
152            5           7
153            6           8
154            4          11
156            6           7
162            6           5
163            7           6
164            5           9
165            6           7
213            6           6
214            4           9
215            5           7
216            6           5
231            6           6
234            6          11
235            7           9
236            8           7
241            4           9
243            6          11
245            5          12
246            6          10
251            5           7
253            7           9
254            5          12
256            7           8
261            6           5
263            8           7
264            6          10
265            7           8
312            6           6
314            5          10
315            6           8
316            7           6
321            6           6
324            6          11
325            7           9
326            8           7
341            5          10
342            6          11
345            6          13
346            7          11
351            6           8
352            7           9
354            6          13
356            8           9
361            7           6
362            8           7
364            7          11
365            8           9
412            4           9
413            5          10
415            4          11
416            5           9
421            4           9
423            6          11
425            5          12
426            6          10
431            5          10
432            6          11
435            6          13
436            7          11
451            4          11
452            5          12
453            6          13
456            6          12
461            5           9
462            6          10
463            7          11
465            6          12
512            5           7
513            6           8
514            4          11
516            6           7
521            5           7
523            7           9
524            5          12
526            7           8
531            6           8
532            7           9
534            6          13
536            8           9
541            4          11
542            5          12
543            6          13
546            6          12
561            6           7
562            7           8
563            8           9
564            6          12
612            6           5
613            7           6
614            5           9
615            6           7
621            6           5
623            8           7
624            6          10
625            7           8
631            7           6
632            8           7
634            7          11
635            8           9
641            5           9
642            6          10
643            7          11
645            6          12
651            6           7
652            7           8
653            8           9
654            6          12

120 rows selected.
</pre></div>

### SC: Set Combination [Items may not repeat, order does not matter]
[&uarr; 3 Item Sequence Generation by Pure SQL](#3-item-sequence-generation-by-pure-sql)<br />
[&darr; SQL: All combinations of :SEQ_SIZE items from set](#sql-all-combinations-of-seq_size-items-from-set)<br />
[&darr; Join Condition: All combinations of :SEQ_SIZE items from set](#join-condition-all-combinations-of-seq_size-items-from-set)<br />
[&darr; Output: All 20 combinations of 3 items from 6](#output-all-20-combinations-of-3-items-from-6)<br />

#### SQL: All combinations of :SEQ_SIZE items from set
[&uarr; SC: Set Combination [Items may not repeat, order does not matter]](#sc-set-combination-items-may-not-repeat-order-does-not-matter)<br />

```sql
WITH tree_walk(id, lev, tot_price, tot_value, path) AS (
    SELECT '0', 0, 0, 0, CAST (NULL AS VARCHAR2(400))
      FROM DUAL
     UNION ALL
    SELECT itm.id,
           trw.lev + 1,
           trw.tot_price + itm.item_price,
           trw.tot_value + itm.item_value,
           trw.path || itm.id
      FROM tree_walk trw
      JOIN items itm
        ON itm.id > trw.id
     WHERE trw.lev < :SEQ_SIZE
)
SELECT path, tot_price, tot_value
  FROM tree_walk
 WHERE lev = :SEQ_SIZE
```

#### Join Condition: All combinations of :SEQ_SIZE items from set
[&uarr; SC: Set Combination [Items may not repeat, order does not matter]](#sc-set-combination-items-may-not-repeat-order-does-not-matter)<br />

<pre>
        ON itm.id > trw.id
</pre>
In this case order is not significant, so we avoid duplication of equivalent sequences by joining items only of higher rank (since item repetition is not allowed). The CYCLE issue that we encountered in the multiset sequence type does not occur here when we keep the termination condition as the WHERE clause
<pre>
       WHERE trw.lev < :SEQ_SIZE
</pre>

#### Output: All 20 combinations of 3 items from 6
[&uarr; SC: Set Combination [Items may not repeat, order does not matter]](#sc-set-combination-items-may-not-repeat-order-does-not-matter)<br />

The number of sequences of type SC of size r from a set n items is given by the usual combinations formula:

<img src="png/nCr.png">

For example, when n = 6 and r = 3, the number of SC sequences is:

<img src="png/6C3.png">

<div class="scrollbox"><pre>
Path Total Price Total Value
---- ----------- -----------
123            6           6
124            4           9
125            5           7
126            6           5
134            5          10
135            6           8
136            7           6
145            4          11
146            5           9
156            6           7
234            6          11
235            7           9
236            8           7
245            5          12
246            6          10
256            7           8
345            6          13
346            7          11
356            8           9
456            6          12

20 rows selected.
</pre></div>

### SC Using Nested Tables: Set Combination [Items may not repeat, order does not matter]
[&uarr; 3 Item Sequence Generation by Pure SQL](#3-item-sequence-generation-by-pure-sql)<br />
[&darr; Nested Table SQL: All combinations of :SEQ_SIZE items from set](#nested-table-sql-all-combinations-of-seq_size-items-from-set)<br />
[&darr; Nested Table Result without CYCLE Clause - 3 items from 6](#nested-table-result-without-cycle-clause---3-items-from-6)<br />
[&darr; Nested Table Result with CYCLE Clause - 3 items from 6](#nested-table-result-with-cycle-clause---3-items-from-6)<br />

In this section we show that using nested tables to store the path instead of a string leads to incorrect results due to Oracle indicating cycles where none exist. This was tested on Oracle 19.3, with script on GitHub: item_seqs_nt.sql.

#### Nested Table SQL: All combinations of :SEQ_SIZE items from set
[&uarr; SC Using Nested Tables: Set Combination [Items may not repeat, order does not matter]](#sc-using-nested-tables-set-combination-items-may-not-repeat-order-does-not-matter)<br />

```sql
WITH tree_walk(item_id, lev, tot_price, tot_value,  path) AS (
    SELECT '0', 0, 0, 0, char_nt()
      FROM DUAL
     UNION ALL
    SELECT itm.id,
           trw.lev + 1,
           trw.tot_price + itm.item_price,
           trw.tot_value + itm.item_value,
           trw.path MULTISET UNION char_nt(itm.id)
      FROM tree_walk trw
      JOIN items itm
        ON itm.id > trw.item_id
     WHERE trw.lev < :SEQ_SIZE
)
SELECT ListAgg(pth.COLUMN_VALUE, '') path,
       trw.tot_price,
       trw.tot_value
  FROM tree_walk trw
 CROSS APPLY TABLE(trw.path) pth
 WHERE trw.lev = :SEQ_SIZE
 GROUP BY trw.tot_price,
       trw.tot_value
 ORDER BY 1
```

In the query above the path is stored as a nested table of type char_nt where the type definition is:
<pre>
CREATE OR REPLACE TYPE char_nt AS TABLE OF VARCHAR2(4000)
</pre>
The item sequences are now accumulated via the expression:
```sql
           trw.path MULTISET UNION char_nt(itm.id)
```

The main subquery uses ListAgg to join the items together to allow simpler comparison with the earlier queries that accumulate the path as a string.
#### Nested Table Result without CYCLE Clause - 3 items from 6
[&uarr; SC Using Nested Tables: Set Combination [Items may not repeat, order does not matter]](#sc-using-nested-tables-set-combination-items-may-not-repeat-order-does-not-matter)<br />

```sql
  JOIN items itm
       *
ERROR at line 11:
ORA-32044: cycle detected while executing recursive WITH query
```

#### Nested Table Result with CYCLE Clause - 3 items from 6
[&uarr; SC Using Nested Tables: Set Combination [Items may not repeat, order does not matter]](#sc-using-nested-tables-set-combination-items-may-not-repeat-order-does-not-matter)<br />

In order to avoid the error above we can add in the CYCLE clause after the tree_walk subquery. First, let's try with lev as the cycle column, and remember that this is the iteration number and so can't possibly repeat for any descendant row:
```sql
    ) CYCLE lev SET c TO '*' DEFAULT ' '
```

The result is then (adding in c to the SELECT list)
```
Nested table path: 10 (should be 20) combinations of 3 items from 6 with cycle clause on lev

Path Total Price Total Value C
---- ----------- ----------- -
123            6           6
124            4           9
125            5           7
126            6           5
134            5          10 *
135            6           8 *
136            7           6 *
145            4          11 *
146            5           9 *
156            6           7 *

10 rows selected.
```

Only 10 rows are returned instead of the correct number, 20, and 6 of these are incorrectly marked as cycles.

Now, let's try with item_id, lev as the cycle columns:
```sql
    ) CYCLE item_id, lev SET c TO '*' DEFAULT ' '
```

The result is then:
```
Nested table path: All 20 combinations of 3 items from 6 with cycle clause on item_id, lev

Path Total Price Total Value C
---- ----------- ----------- -
123            6           6
124            4           9
125            5           7
126            6           5
134            5          10
135            6           8
136            7           6
145            4          11
146            5           9
156            6           7
234            6          11 *
235            7           9 *
236            8           7
245            5          12 *
246            6          10
256            7           8
345            6          13 *
346            7          11
356            8           9
456            6          12

20 rows selected.
```

Now we see the correct result set, but note that 4 records are incorrectly marked as cycles. This means that they would not be iterated further if we wanted sequences of more than 3 in length.

## 4 Item Sequence Generation by Mixed SQL and PL/SQL
[&uarr; Contents](#contents)<br />
[&darr; Recursion vs Iteration](#recursion-vs-iteration)<br />
[&darr; Join Condition Function](#join-condition-function)<br />
[&darr; Shared Procedures and Function](#shared-procedures-and-function)<br />
[&darr; PL/SQL Recursion](#plsql-recursion)<br />
[&darr; PL/SQL Iteration](#plsql-iteration)<br />

### Recursion vs Iteration
[&uarr; 4 Item Sequence Generation by Mixed SQL and PL/SQL](#4-item-sequence-generation-by-mixed-sql-and-plsql)<br />

In the previous section we showed how recursive SQL can be used to generate item sequences following the approach specified algorithmically in the first article. If we use the procedural language PL/SQL, with embedded SQL, we can also use recursive functions to achieve the same results. In addition we will see that we can generate the sequences without recursion, but using simple iteration in PL/SQL.

When using recursive SQL intermediate paths are stored internally without explicit arrays or tables, but when using PL/SQL as the generative mechanism we need to use either an array or a temporary table (although with recursive function and array, only an array type is needed with results piped from the function).

This gives us an additional four methods to add to our pure SQL method. The two methods using arrays are implemented as pipelined functions that are called directly from a query or view. The two methods using a temporary table are implemented as procedures since a query can't call a function that writes to a table, with the main query or view then querying the result records in the temporary table.

### Join Condition Function
[&uarr; 4 Item Sequence Generation by Mixed SQL and PL/SQL](#4-item-sequence-generation-by-mixed-sql-and-plsql)<br />

We can create a function for the join condition on items being joined to the current paths. We wil use this in the queries in this section on small example problems, but will see in a later article that using a function in this way can adversely affect performance.

```sql
FUNCTION Seq_Type_Condition_YN(
            p_seq_type                     VARCHAR2,
            p_item_id                      VARCHAR2,
            p_item_id_prior                VARCHAR2,
            p_path                         VARCHAR2)
            RETURN                         VARCHAR2 IS
  l_ret_yn VARCHAR2(1) := 'N';
BEGIN
  CASE p_seq_type
    WHEN 'MP' THEN
      l_ret_yn := 'Y';
    WHEN 'MC' THEN
      IF p_item_id >= p_item_id_prior THEN l_ret_yn := 'Y'; END IF;
    WHEN 'SP' THEN
      IF Nvl(p_path,'x') NOT LIKE '%' || p_item_id || '%' THEN l_ret_yn := 'Y'; END IF;
    WHEN 'SC' THEN
      IF p_item_id > p_item_id_prior THEN l_ret_yn := 'Y'; END IF;
  END CASE;
  RETURN l_ret_yn;
END Seq_Type_Condition_YN;
```

### Shared Procedures and Function
[&uarr; 4 Item Sequence Generation by Mixed SQL and PL/SQL](#4-item-sequence-generation-by-mixed-sql-and-plsql)<br />
[&darr; Temporary Table Insert Procedures](#temporary-table-insert-procedures)<br />
[&darr; Array Root Function - initial_Path](#array-root-function---initial_path)<br />

#### Temporary Table Insert Procedures
[&uarr; Shared Procedures and Function](#shared-procedures-and-function)<br />

The two temporary table procedures can share two procedures for insertion into the small_paths temporary table:
- insert_Initial_Path: This inserts the (dummy) root record into the temporary table, after deleting any existing records
- insert_Paths: This inserts subsequent records into the temporary table
    - Insert, selecting from small_paths
        - joining items according to the sequence type, via a call to Seq_Type_Condition_YN
    - Delete the prior level records from small_paths

```sql
PROCEDURE insert_Initial_Path IS
BEGIN
  DELETE small_paths;
  INSERT INTO small_paths (
     path, lev, tot_price, tot_value, item_id
  ) VALUES ('', 0, 0, 0, '0');
END insert_Initial_Path;

PROCEDURE insert_Paths(
            p_seq_type                     VARCHAR2,
            p_lev                          PLS_INTEGER) IS
BEGIN
  INSERT INTO small_paths (
     path, lev, tot_price, tot_value, item_id
  )
  SELECT smp.path || itm.id,
         p_lev,
         smp.tot_price + itm.item_price,
         smp.tot_value + itm.item_value,
         itm.id
    FROM small_paths smp
    JOIN items itm
      ON Item_Seqs.Seq_Type_Condition_YN(
                  p_seq_type       => p_seq_type,
                  p_item_id        => itm.id,
                  p_item_id_prior  => smp.item_id,
                  p_path           => smp.path) = 'Y';
  DELETE small_paths WHERE lev = p_lev - 1;
END insert_Paths;
```

#### Array Root Function - initial_Path
[&uarr; Shared Procedures and Function](#shared-procedures-and-function)<br />

The two array functions can share a function that returns the (dummy) root record:
- initial_Path: This returns the (dummy) root record

```sql
FUNCTION initial_Path
            RETURN                         small_paths%ROWTYPE IS
  l_path_req   small_paths%ROWTYPE;
BEGIN
  SELECT '', 0, 0, 0, '0' INTO l_path_req FROM DUAL;
  RETURN l_path_req;
END initial_Path;

```

### PL/SQL Recursion
[&uarr; 4 Item Sequence Generation by Mixed SQL and PL/SQL](#4-item-sequence-generation-by-mixed-sql-and-plsql)<br />
[&darr; Temporary Table Recursion - Pop_Table_Recurse](#temporary-table-recursion---pop_table_recurse)<br />
[&darr; Array Recursion - Array_Recurse](#array-recursion---array_recurse)<br />

The diagram shows process and data flows for sequence generation by PL/SQL recursion using either array or temporary table for storage of intermediate paths.

<img src="png/PLS-Recurse.png">

#### Temporary Table Recursion - Pop_Table_Recurse
[&uarr; PL/SQL Recursion](#plsql-recursion)<br />

The temporary table recursion procedure uses the shared insertion procedures.

- If level = 0 then
  - Insert (dummy) root record, by calling insert_Initial_Path
  - Return
- End if
- Call (recursively) Pop_Table_Recurse passing level - 1
- Insert records at current level, then delete prior level, by calling insert_Paths

```sql
PROCEDURE Pop_Table_Recurse(
            p_seq_type                     VARCHAR2,
            p_lev                          PLS_INTEGER := NULL) IS
BEGIN
  IF p_lev = 0 THEN
    insert_Initial_Path;
    RETURN;
  END IF;
  Pop_Table_Recurse(p_seq_type, Nvl(p_lev, g_seq_size) - 1);
  insert_Paths(p_seq_type => p_seq_type,
               p_lev      => Nvl(p_lev, g_seq_size));
END Pop_Table_Recurse;
```

#### Array Recursion - Array_Recurse
[&uarr; PL/SQL Recursion](#plsql-recursion)<br />

The array recursion function uses the shared function initial_Path.

- If level = 0 then
    - Pipe (dummy) root record, by calling initial_Path
    - Return
- End if
- Loop over query returning the next level paths
    - This selects from a recursive call to Array_Recurse. passing level - 1
        - joining items according to the sequence type, via a call to Seq_Type_Condition_YN
    - Pipe the record
- End loop

```sql
FUNCTION Array_Recurse(
            p_seq_type                     VARCHAR2,
            p_lev                          PLS_INTEGER := NULL)
            RETURN                         paths_arr PIPELINED IS
BEGIN
  IF p_lev = 0 THEN
    PIPE ROW(initial_Path);
    RETURN;
  END IF;
  FOR rec IN (
    SELECT smp.path || itm.id path,
           smp.lev + 1 lev,
           smp.tot_price + itm.item_price tot_price,
           smp.tot_value + itm.item_value tot_value,
           itm.id item_id
      FROM Array_Recurse(p_seq_type, Nvl(p_lev, g_seq_size) - 1) smp
      JOIN items itm
        ON Item_Seqs.Seq_Type_Condition_YN(
                      p_seq_type       => p_seq_type,
                      p_item_id        => itm.id,
                      p_item_id_prior  => smp.item_id,
                      p_path           => smp.path) = 'Y') LOOP
    PIPE ROW(rec);
  END LOOP;
END Array_Recurse;
```
Note that, although we need to define an array type, paths_arr, and make it the return type of the function, we do not need to declare any array instances explicitly.

### PL/SQL Iteration
[&uarr; 4 Item Sequence Generation by Mixed SQL and PL/SQL](#4-item-sequence-generation-by-mixed-sql-and-plsql)<br />
[&darr; Temporary Table Iteration - Pop_Table_Iterate](#temporary-table-iteration---pop_table_iterate)<br />
[&darr; Array Iteration - Array_Iterate](#array-iteration---array_iterate)<br />

The diagram shows process and data flows for sequence generation by PL/SQL iteration using either array or table for storage of intermediate paths.

<img src="png/PLS-Iterate.png">

#### Temporary Table Iteration - Pop_Table_Iterate
[&uarr; PL/SQL Iteration](#plsql-iteration)<br />

The temporary table iteration procedure uses the shared insertion procedures.

- Insert (dummy) root record, by calling insert_Initial_Path
- Loop from 1 to :SEQ_SIZE
    - Insert records at current level, then delete prior level, by calling insert_Paths
- End loop

```sql
PROCEDURE Pop_Table_Iterate(
            p_seq_type                     VARCHAR2) IS
BEGIN
  insert_initial_Path;
  FOR i IN 1..g_seq_size LOOP
    insert_Paths(p_seq_type => p_seq_type,
                 p_lev      => i);
  END LOOP;
END Pop_Table_Iterate;
```

#### Array Iteration - Array_Iterate
[&uarr; PL/SQL Iteration](#plsql-iteration)<br />

The array iteration function uses the shared function initial_Path.

- Declare arrays for paths and new paths
- Initialize paths array with (dummy) root record, by calling initial_Path
- Loop from 1 to :SEQ_SIZE
    - Select into the new paths array:
        - driving from the paths array
        - joining items according to the sequence type, via a call to Seq_Type_Condition_YN
    - This selects from a recursive call to Array_Recurse. passing level - 1
    - Set the paths array equal to the new paths array
- End loop
- Loop over the paths array
    - Pipe the record
- End loop

```sql
FUNCTION Array_Iterate(
            p_seq_type                     VARCHAR2)
            RETURN                         paths_arr PIPELINED IS
  l_paths_lis       paths_arr := paths_arr(initial_Path);
  l_paths_new_lis   paths_arr;
BEGIN
  FOR i IN 1..g_seq_size LOOP
    SELECT smp.path || itm.id,
           smp.lev + 1,
           smp.tot_price + itm.item_price,
           smp.tot_value + itm.item_value,
           itm.id
      BULK COLLECT INTO l_paths_new_lis
      FROM TABLE(l_paths_lis) smp
      JOIN items itm
        ON Item_Seqs.Seq_Type_Condition_YN(
                    p_seq_type       => p_seq_type,
                    p_item_id        => itm.id,
                    p_item_id_prior  => smp.item_id,
                    p_path           => smp.path) = 'Y';

    l_paths_lis := l_paths_new_lis;
  END LOOP;
  FOR i IN 1..l_paths_lis.COUNT LOOP
    PIPE ROW(l_paths_lis(i));
  END LOOP;
END Array_Iterate;
```
Note that, in order to collect a new set of records into an array based on the prior set, we need to define two arrays and move the new set to the current array after each iteration.

## 5 Conclusion
[&uarr; Contents](#contents)<br />

In the first article we showed in an abstract way how recursive techniques may be used to generate sequences of items that can form the basis of solutions for larger problems involving constraints and value optimization.

In this article we demonstrated how these algorithms can be implemented using recursive SQL. We also demonstrated how PL/SQL can be used to implement both recursive and iterative versions with embedded SQL.

In the next article we will extend consideration beyond just the generation of sequences to optimization problems where we want to select sequences that maximize a value measure subject to constraints. This will follow a mathematical approach similar to the first article for similar reasons.

- [OPICO 1-8: Optimization Problems with Items and Categories in Oracle](#list-of-articles)
