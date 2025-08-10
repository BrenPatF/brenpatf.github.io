---
layout: post
title: "SQL for Network Grouping"
date: 2013-03-23
migrated: true
group: recursive-sql
categories: 
  - "model"
  - "oracle"
  - "performance"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "connect"
  - "hierarchical"
  - "model-2"
  - "network"
  - "recursive"
  - "sql"
  - "subquery-factor"
---

I noticed an interesting question posted on OTN this (European) Saturday morning, [Hierarchical query to combine two groupings into one broad joint grouping](https://forums.oracle.com/ords/apexds/post/hierarchical-query-to-combine-two-groupings-into-one-broad-6594). I quickly realised that the problem posed was an example of a very general class of network problems that arises quite often:

_Given a set of nodes and a rule for pair-wise (non-directional) linking, obtain the set of implied networks_

Usually, in response to such a problem someone will suggest a CONNECT BY query solution. Unfortunately, although hierarchical SQL techniques can be used theoretically to resolve these non-hierarchical networks, they tend to be extremely inefficient for networks of any size and are therefore often impractical. There are two problems in particular:

1. Non-hierarchical networks have no root nodes, so the traversal needs to be repeated from every node in the network set
2. Hierarchical queries retrieve all possible routes through a network

I illustrated the second problem in my last post, [PL/SQL Profiling 1 – Overview](https://brenpatf.github.io/migrated/notes-on-oracles-hierarchical-plsql-profiler), and I intend to write a longer article on the subject of networks at a later date. The most efficient way to traverse generalised networks in Oracle involves the use of PL/SQL, such as in my Scribd article of June 2010, [An Oracle Network Traversal PL SQL Program](http://www.scribd.com/doc/32976987/An-Oracle-Network-Traversal-PL-SQL-Program). For this article, though, I will stick to SQL-only techniques and will write down three solutions in a general format whereby the tables of the specific problem are read by initial subquery factors links\_v and nodes\_v that are used in the rest of the queries. I'll save detailed explanation and performance analysis for the later article (see update at end of this section).

The three queries use two hierarchical methods and a method involving the Model clause:

1. CONNECT BY: This is the least efficient
2. Recursive subquery factors: This is more efficient than the first but still suffers from the two problems above
3. Model clause: This is intended to bypass the performance problems of hierarchical queries, but is still slower than PL/SQL

**\[Update, 2 September 2015\]** I have since written two articles on related subjects, the first, [PL/SQL Pipelined Function for Network Analysis](https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/) describes a PL/SQL program that traverses all networks and lists their structure. The second, [Recursive SQL for Network Analysis, and Duality](https://brenpatf.github.io/migrated/recursive-sql-for-network-analysis-and-duality/), uses a series of examples to illustrate and explain the different characteristics of the first two recursive SQL methods.

## Problem Definition

### Data Structure

I have taken the data structure of the OTN poster, made all fields character, and added three more records comprising a second isolated node (10) and a subnetwork of nodes 08 and 09. ITEM\_ID is taken to be the primary key.

```
SQL> SELECT *
  2    FROM item_groups
  3  /

ITEM_ID    GROUP1   GROUP2
---------- -------- --------
01         A        100
02         A        100
03         A        101
04         B        100
05         B        102
06         C        103
07         D        101
08         E        104
09         E        105
10         F        106

10 rows selected.
```

### Grouping Structure

The poster defines two items to be linked if they share the same value for either GROUP1 or GROUP2 attributes (which could obviously be generalised to any number of attributes), and items are in the same group if they can be connected by a chain of links. Observe that if there were only one grouping attribute then the problem would be trivial as that would itself group the items. Having more than one makes it more interesting and more difficult.

A real world example of such networks can be seen to be sibling networks if one takes people as the nodes and father and mother as the attributes.

### Network Diagram

<img src="/migrated_images/2013/03/Item-Groups-v1.0.jpg" alt="Item Groups, v1.0" title="Item Groups, v1.0" />

## CONNECT BY Solution
 
### SQL

```sql
WITH links_v AS (
SELECT t_fr.item_id node_id_fr,
       t_to.item_id node_id_to,
       t_fr.item_id || '-' || Row_Number() OVER (PARTITION BY t_fr.item_id ORDER BY t_to.item_id) link_id
  FROM item_groups t_fr
  JOIN item_groups t_to
    ON t_to.item_id > t_fr.item_id
   AND (t_to.group1 = t_fr.group1 OR t_to.group2 = t_fr.group2)
), nodes_v AS (
 SELECT item_id node_id
   FROM item_groups
), tree AS (
SELECT link_id, CONNECT_BY_ROOT (link_id) root_id
  FROM links_v
CONNECT BY NOCYCLE (node_id_fr = PRIOR node_id_to OR node_id_to = PRIOR node_id_fr OR 
                     node_id_fr = PRIOR node_id_fr OR node_id_to = PRIOR node_id_to)
), group_by_link AS (
SELECT DISTINCT Min (root_id) OVER (PARTITION BY link_id) group_id, link_id
  FROM tree
), linked_nodes AS (
SELECT g.group_id, l.node_id_fr node_id
  FROM group_by_link g
  JOIN links_v l
    ON l.link_id = g.link_id
 UNION
SELECT g.group_id, l.node_id_to
  FROM group_by_link g
  JOIN links_v l
    ON l.link_id = g.link_id
)
SELECT l.group_id "Network", l.node_id "Node"
  FROM linked_nodes l
 UNION ALL
SELECT '00 (unlinked)', node_id
  FROM nodes_v n
 WHERE n.node_id NOT IN (SELECT node_id FROM linked_nodes)
ORDER BY 1, 2
```

### Output

```
All networks by CONNECT BY - Unlinked nodes share network id 0
Network       Node
------------- ----
00 (unlinked) 06
              10
01-1          01
              02
              03
              04
              05
              07
08-1          08
              09

10 rows selected.
```

### Notes on CONNECT BY Solution

- For convenience I have grouped the unlinked nodes into one dummy network; it's easy to assign them individual identifiers if desired

## Recursive Subquery Factors (RSF) Solution

### SQL


```sql
WITH links_v AS (
SELECT t_fr.item_id node_id_fr,
       t_to.item_id node_id_to,
       t_fr.item_id || '-' || Row_Number() OVER (PARTITION BY t_fr.item_id ORDER BY t_to.item_id) link_id
  FROM item_groups t_fr
  JOIN item_groups t_to
    ON t_to.item_id > t_fr.item_id
   AND (t_to.group1 = t_fr.group1 OR t_to.group2 = t_fr.group2)
), nodes_v AS (
 SELECT item_id node_id
   FROM item_groups
), rsf (node_id, id, root_id) AS (
SELECT node_id, NULL, node_id
  FROM nodes_v
 UNION ALL
SELECT CASE WHEN l.node_id_to = r.node_id THEN l.node_id_fr ELSE l.node_id_to END, 
       l.link_id id, r.root_id
  FROM rsf r
  JOIN links_v l
    ON (l.node_id_fr = r.node_id OR l.node_id_to = r.node_id)
   AND l.link_id != Nvl (r.id, '0')
) CYCLE node_id SET is_cycle TO '*' DEFAULT ' '
SELECT DISTINCT Min (root_id) OVER (PARTITION BY node_id) "Network", node_id "Node"
  FROM rsf
 ORDER BY 1, 2
```

### Output

```
All networks by RSF - Unlinked nodes have their own network ids
Network       Node
------------- ----
01            01
              02
              03
              04
              05
              07
06            06
08            08
              09
10            10

10 rows selected.
```

### Notes on Recursive Subquery Factors (RSF) Solution


- Here I have given the unlinked nodes their own network identifiers; they could equally have been grouped together under a dummy network

## Model Clause Solution

### SQL

<div class="scrollbox">
<pre>
WITH links_v AS (
SELECT t_fr.item_id node_id_fr,
       t_to.item_id node_id_to,
       t_fr.item_id || '-' || Row_Number() OVER (PARTITION BY t_fr.item_id ORDER BY t_to.item_id) link_id
  FROM item_groups t_fr
  JOIN item_groups t_to
    ON t_to.item_id > t_fr.item_id
   AND (t_to.group1 = t_fr.group1 OR t_to.group2 = t_fr.group2)
), nodes_v AS (
 SELECT item_id node_id
   FROM item_groups
), lnk_iter AS (
SELECT *
  FROM links_v
 CROSS JOIN (SELECT 0 iter FROM DUAL UNION SELECT 1 FROM DUAL)
), mod AS (
SELECT *
  FROM lnk_iter
MODEL
  DIMENSION BY (Row_Number() OVER (PARTITION BY iter ORDER BY link_id) rn, iter)
  MEASURES (Row_Number() OVER (PARTITION BY iter ORDER BY link_id) id_rn, link_id id,
        node_id_fr nd1, node_id_to nd2,
        1 lnk_cur,
        CAST ('x' AS VARCHAR2(100)) nd1_cur,
        CAST ('x' AS VARCHAR2(100)) nd2_cur,
        0 net_cur,
        CAST (NULL AS NUMBER) net,
        CAST (NULL AS NUMBER) lnk_prc,
        1 not_done,
        0 itnum)
  RULES UPSERT ALL
  ITERATE(100000) UNTIL (lnk_cur[1, Mod (iteration_number+1, 2)] IS NULL)
  (
    itnum[ANY, ANY] = iteration_number,
    not_done[ANY, Mod (iteration_number+1, 2)] = Count (CASE WHEN net IS NULL THEN 1 END)[ANY, Mod (iteration_number, 2)],
    lnk_cur[ANY, Mod (iteration_number+1, 2)] = 
        CASE WHEN not_done[CV(), Mod (iteration_number+1, 2)] > 0 THEN 
               Nvl (Min (CASE WHEN lnk_prc IS NULL AND net = net_cur THEN id_rn END)[ANY, Mod (iteration_number, 2)],
                    Min (CASE WHEN net IS NULL THEN id_rn END)[ANY, Mod (iteration_number, 2)])
        END,
    lnk_prc[ANY, Mod (iteration_number+1, 2)] = lnk_prc[CV(), Mod (iteration_number, 2)],
    lnk_prc[lnk_cur[1, Mod (iteration_number+1, 2)], Mod (iteration_number+1, 2)] = 1,
    net_cur[ANY, Mod (iteration_number+1, 2)] = 
        CASE WHEN Min (CASE WHEN lnk_prc IS NULL AND net = net_cur THEN id_rn END)[ANY, Mod (iteration_number, 2)] IS NULL THEN
               net_cur[CV(), Mod (iteration_number, 2)] + 1 
             ELSE 
               net_cur[CV(), Mod (iteration_number, 2)] 
        END,
    nd1_cur[ANY, Mod (iteration_number+1, 2)] = nd1[lnk_cur[CV(), Mod (iteration_number+1, 2)], Mod (iteration_number, 2)],
    nd2_cur[ANY, Mod (iteration_number+1, 2)] = nd2[lnk_cur[CV(), Mod (iteration_number+1, 2)], Mod (iteration_number, 2)],
    net[ANY, Mod (iteration_number+1, 2)] = 
        CASE WHEN (nd1[CV(),Mod (iteration_number+1, 2)] IN (nd1_cur[CV(),Mod (iteration_number+1, 2)], nd2_cur[CV(),Mod (iteration_number+1, 2)]) OR 
                   nd2[CV(),Mod (iteration_number+1, 2)] IN (nd1_cur[CV(),Mod (iteration_number+1, 2)], nd2_cur[CV(),Mod (iteration_number+1, 2)]))
                   AND net[CV(),Mod (iteration_number, 2)] IS NULL THEN
               net_cur[CV(),Mod (iteration_number+1, 2)]
             ELSE
               net[CV(),Mod (iteration_number, 2)]
        END
  )
)
SELECT To_Char (net) "Network", nd1 "Node"
  FROM mod
 WHERE not_done = 0
 UNION
SELECT To_Char (net), nd2
  FROM mod
 WHERE not_done = 0
 UNION ALL
SELECT '0 (unlinked)', node_id
  FROM nodes_v n
 WHERE n.node_id NOT IN (SELECT nd1 FROM mod WHERE nd1 IS NOT NULL UNION SELECT nd2 FROM mod WHERE nd2 IS NOT NULL)
ORDER by 1, 2
</pre>
</div>

### Output

```
All networks by Model - Unlinked nodes share network id 00
Network       Node
------------- ----
0 (unlinked)  06
              10
1             01
              02
              03
              04
              05
              07
2             08
              09

10 rows selected.
```

### Notes on Model Clause Solution

- For convenience I have grouped the unlinked nodes into one dummy network; it's easy to assign them individual identifiers if desired
- My _Cyclic Iteration_ technique used here appears to be novel

## Conclusion

- It is always advisable with a new problem in SQL to consider whether it falls into a general class of problems for which solutions have already been found
- Three solution methods for network resolution in pure SQL have been presented and demonstrated on a small test problem; the performance issues mentioned should be considered carefully before applying them on larger problems
- The Model clause solution is likely to be the most efficient on larger, looped networks, but if better performance is required then PL/SQL recursion-based methods would be faster
- For smaller problems with few loops the simpler method of recursive subquery factors may be preferred, or, for versions prior to v11.2, CONNECT BY

