---
layout: post
title: "Recursive SQL for Network Analysis, and Duality"
date: 2015-09-02
migrated: true
group: recursive-sql
categories: 
  - "oracle"
  - "performance"
  - "pipelined"
  - "plsql"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "v12"
tags: 
  - "connect-by"
  - "network"
  - "pipelined"
  - "recursion"
  - "sql"
  - "subquery-factor"
---
In March 2013 I wrote an article on the use of SQL to group network-structured records into their distinct connected subnetworks, [SQL for Network Grouping](https://brenpatf.github.io/migrated/sql-for-network-grouping/). I looked at two solution approaches commonly put forward on Oracle forums for these types of problem, using Oracle's Connect By recursion, and the more recent recursive subquery factoring, and also put forward a new solution of my own using the Model clause. I noted however that SQL solutions are generally very inefficent compared with a good PL/SQL solution, such as I posted here, [PL/SQL Pipelined Function for Network Analysis](https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/). For the first two methods, I noted:

> 1. Non-hierarchical networks have no root nodes, so the traversal needs to be repeated from every node in the network set
> 2. Hierarchical queries retrieve all possible routes through a network

I also noted that Connect By is more inefficient than recursive subquery factoring, but did not say why, promising a more detailed explanation at a later date. In this article I illustrate the behaviour of both recursive SQL methods through a series of five elementary networks, followed by a simple combination of the five. I then use the foreign key network from Oracle's HR demo (v12 version, with OE and PM schemas included) as a final example.

In this article I consider traversal of a single connected network from a given root node (or several if each root node is specified).

It is shown that the behaviour of Connect By can be understood best by considering it to traverse all paths through a network that is dual to the original network.

## Dual Networks

### Dual network definition

The dual network consists of a set of nodes and links (d-nodes and d-links say) defined thus:

- the d-nodes correspond to each link in the original network that is adjacent (via a node) to at least one other link, including itself if its start and end nodes are the same
- the d-links correspond to each pair of adjacent links where the 'from' link identifier is alphabetically smaller than that of the 'to' link, except for the case of links that are adjacent to themselves where a single d-link has the same 'from' and 'to' link

### Dual network SQL

The d-node identifiers are just the link identifiers, while the d-link identifiers use the adjacency-defining node identifiers with a sequential number (partitioned by node) attached.

```sql
WITH dist_links AS (
SELECT	DISTINCT CASE WHEN lin_2.node_fr IN (lin_1.node_fr, lin_1.node_to) THEN lin_2.node_fr ELSE lin_2.node_to END link_node,
        lin_1.id node_fr_d,
	lin_2.id node_to_d
  FROM links lin_1
  JOIN links lin_2
    ON lin_2.node_fr IN (lin_1.node_fr, lin_1.node_to)
    OR lin_2.node_to IN (lin_1.node_fr, lin_1.node_to)
 WHERE lin_2.id >= lin_1.id
   AND (lin_2.id != lin_1.id OR lin_2.node_fr = lin_1.node_to)
)
SELECT Substr (link_node, 1, Length (link_node)-1) || Row_Number () OVER (PARTITION BY link_node
                            ORDER BY node_fr_d, node_to_d) || '-' || Substr (link_node, -1),
       node_fr_d,
       node_to_d
  FROM dist_links
```

### Dual network characteristics

Dual networks defined as above are generally larger than the original networks and are usually more heavily looped, which explains the inferior performance of Connect by compared with recursive subquery factor solutions. The PL/SQL solution mentioned above, while traversing the entire network, does not traverse all possible routes through it and its performance is thus not adversely affected by the degree of looping.

## SQL Queries

The recursive SQL queries return all routes through the network from the roots supplied. In my attached script I also have versions that filter out repeated links. The pipelined function query returns a single, exhaustive route through the network, distinguishing a set of tree links from loop-closing links; it also returns all subnetworks without requiring input roots.

### Pipelined Function Query (PLF)

See [PL/SQL Pipelined Function for Network Analysis](https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/) for the Pl/SQL function.

```sql
SELECT root_node_id             "Network",
       Count (DISTINCT link_id) OVER (PARTITION BY root_node_id) - 1 "#Links",
       Count (DISTINCT node_id) OVER (PARTITION BY root_node_id) "#Nodes",
       LPad (dirn || ' ', 2*node_level, ' ') || node_id || loop_flag "Node",
       link_id || CASE WHEN link_id = 'ROOT' THEN '_' || Substr (root_node_id, -1) END "Link",
       node_level               "Lev"
  FROM TABLE (Net_Pipe.All_Nets)
 ORDER BY line_no
```

### Recursive Subquery Factor Query (RSF)

```sql
WITH rsf (node_id, prefix, id, lev) AS (
SELECT node_id, '', 'ROOT_' || Substr (node_id, 4, 1), 0
  FROM nodes_v
 WHERE Substr (node_id, 2, 1) = '1'
 UNION ALL
SELECT CASE WHEN l.node_id_to = r.node_id THEN l.node_id_fr ELSE l.node_id_to END,
       CASE WHEN l.node_id_fr = l.node_id_to THEN '= ' WHEN l.node_id_fr = r.node_id THEN '> ' ELSE '< ' END,
       l.link_id id, lev + 1
  FROM rsf r
  JOIN links_v l
    ON (l.node_id_fr = r.node_id OR l.node_id_to = r.node_id)
   AND l.link_id != Nvl (r.id, '0')
) SEARCH DEPTH FIRST BY node_id SET line_no
CYCLE node_id SET is_cycle TO '*' DEFAULT ' '
SELECT LPad (r.prefix || ' ', 2*r.lev) || r.node_id || is_cycle "Node",
        r.id "Link",
        line_no
  FROM rsf r
 ORDER BY line_no
```

### Connect By Query (CBY)

```sql
SELECT node_id_fr || ' > ' || node_id_to  "Nodes",
       LPad (' ', 2 * (LEVEL-1)) || link_id || CASE WHEN CONNECT_BY_ISCYCLE = 1 THEN '*' ELSE ' ' END "Link Path"
  FROM links_v
CONNECT BY NOCYCLE ((node_id_fr = PRIOR node_id_to OR node_id_to = PRIOR node_id_fr OR
                     node_id_fr = PRIOR node_id_fr OR node_id_to = PRIOR node_id_to) /*AND link_id != PRIOR link_id*/)
 START WITH Substr (node_id_fr, 2, 1) = '1' AND Substr (node_id_to, 2, 1) = '2'
 ORDER SIBLINGS BY node_id_to
```

## Five Elementary Networks

Oracle's two forms of SQL recursion treat cycles differently

### Connect By Cycles

> The CONNECT\_BY\_ISCYCLE pseudocolumn returns 1 if the current row has a child which is also its ancestor. Otherwise it returns 0

Connect By queries do not return loop-closing nodes, and the prior node is marked as the cycle node.

### Recursive Subquery Factor Cycles

> A row is considered to form a cycle if one of its ancestor rows has the same values for the cycle columns.

Recursive Subquery Factor queries do return loop-closing nodes, and these nodes are marked as the cycle nodes.

We will see this differing behaviour clearly in the following examples. We will also see that the Connect By output on the original network has exactly the same structure as recursive subquery factor output on the dual network if the loop-closing rows are disregarded. Cycle nodes on both definitions are marked with a '\*' in the outputs below.

### Network 1: 3 nodes in line

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-1.png" alt="Dual Network, 1.3 - net-1" title="Dual Network, 1.3 - net-1" />

### Network 2: Simple fork

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-2.png" alt="Dual Network, 1.3 - net-2" title="Dual Network, 1.3 - net-2" />

### Network 3: 2-node loop

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-3.png" alt="Dual Network, 1.3 - net-3" title="Dual Network, 1.3 - net-3" />

### Network 4: 3-node loop

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-4.png" alt="Dual Network, 1.3 - net-4" title="Dual Network, 1.3 - net-4" />

### Network 5: 2 nodes with a self-loop

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-5.png" alt="Dual Network, 1.3 - net-5" title="Dual Network, 1.3 - net-5" />

## Combination of Elementary Networks

### Combination Network 6

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-6.png" alt="Dual Network, 1.3 - net-6" title="Dual Network, 1.3 - net-6" />

This network has 10 links with 3 loops.

### Combination Network 6: PLF Output

```
Node              Link
----------------- ----------
N1-6
> N2-6            L12-6
  = N2-6*         L22-6
  > N3-6          L23-6
    > N4-6        L34-6
      > N6-6      L46-6
        > N4-6*   L64-6
    > N5-6        L35-6
      > N7-6      L57-6
        > N8-6    L78-6
          < N5-6* L58-6
```

### Combination Network 6: RSF Output

```
Node              Link
----------------- ----------
N1-6
> N2-6            L12-6
 =  N2-6*         L22-6
 >  N3-6          L23-6
   >  N4-6        L34-6
     <  N6-6      L64-6
       <  N4-6*   L46-6
     >  N6-6      L46-6
       >  N4-6*   L64-6
   >  N5-6        L35-6
     >  N7-6      L57-6
       >  N8-6    L78-6
         <  N5-6* L58-6
     >  N8-6      L58-6
       <  N7-6    L78-6
         <  N5-6* L57-6
```

### Combination Network 6: CBY Output

<div class="scrollbox">
<pre>
Nodes           Link Path
--------------- --------------------
N1-6 > N2-6     L12-6*
N2-6 > N2-6       L22-6*
N2-6 > N3-6         L23-6*
N3-6 > N4-6           L34-6*
N6-6 > N4-6             L64-6*
N4-6 > N6-6               L46-6*
N3-6 > N5-6             L35-6*
N5-6 > N7-6               L57-6*
N5-6 > N8-6                 L58-6*
N7-6 > N8-6                   L78-6*
N7-6 > N8-6                 L78-6*
N5-6 > N8-6                   L58-6*
N5-6 > N8-6               L58-6*
N5-6 > N7-6                 L57-6*
N7-6 > N8-6                   L78-6*
N7-6 > N8-6                 L78-6*
N5-6 > N7-6                   L57-6*
N4-6 > N6-6             L46-6*
N6-6 > N4-6               L64-6*
N3-6 > N5-6           L35-6*
N3-6 > N4-6             L34-6*
N6-6 > N4-6               L64-6*
N4-6 > N6-6                 L46-6*
N4-6 > N6-6               L46-6*
N6-6 > N4-6                 L64-6*
N5-6 > N7-6             L57-6*
N5-6 > N8-6               L58-6*
N7-6 > N8-6                 L78-6*
N7-6 > N8-6               L78-6*
N5-6 > N8-6                 L58-6*
N5-6 > N8-6             L58-6*
N5-6 > N7-6               L57-6*
N7-6 > N8-6                 L78-6*
N7-6 > N8-6               L78-6*
N5-6 > N7-6                 L57-6*
N2-6 > N3-6       L23-6*
N2-6 > N2-6         L22-6*
N3-6 > N4-6         L34-6*
N6-6 > N4-6           L64-6*
N4-6 > N6-6             L46-6*
N3-6 > N5-6           L35-6*
N5-6 > N7-6             L57-6*
N5-6 > N8-6               L58-6*
N7-6 > N8-6                 L78-6*
N7-6 > N8-6               L78-6*
N5-6 > N8-6                 L58-6*
N5-6 > N8-6             L58-6*
N5-6 > N7-6               L57-6*
N7-6 > N8-6                 L78-6*
N7-6 > N8-6               L78-6*
N5-6 > N7-6                 L57-6*
N4-6 > N6-6           L46-6*
N6-6 > N4-6             L64-6*
N3-6 > N5-6         L35-6*
N3-6 > N4-6           L34-6*
N6-6 > N4-6             L64-6*
N4-6 > N6-6               L46-6*
N4-6 > N6-6             L46-6*
N6-6 > N4-6               L64-6*
N5-6 > N7-6           L57-6*
N5-6 > N8-6             L58-6*
N7-6 > N8-6               L78-6*
N7-6 > N8-6             L78-6*
N5-6 > N8-6               L58-6*
N5-6 > N8-6           L58-6*
N5-6 > N7-6             L57-6*
N7-6 > N8-6               L78-6*
N7-6 > N8-6             L78-6*
N5-6 > N7-6               L57-6*
</pre>
</div>

### Dual Combination Network 6

<img src="/migrated_images/2015/09/Dual-Network-1.3-net-6-D.png" alt="Dual Network, 1.3 - net-6-D" title="Dual Network, 1.3 - net-6-D" />

This network has 15 links with 6 loops, whereas the original had 10 links with 3 loops.

### Dual Combination Network 6: PLF Output

```
Node                      Link
------------------------- ------
L12-6
> L22-6                   N2-1-6
  = L22-6*                N2-3-6
  > L23-6                 N2-4-6
    < L12-6*              N2-2-6
    > L34-6               N3-1-6
      > L35-6             N3-3-6
        < L23-6*          N3-2-6
        > L57-6           N5-1-6
          > L58-6         N5-3-6
            < L35-6*      N5-2-6
            > L78-6       N8-1-6
              < L57-6*    N7-1-6
      > L46-6             N4-1-6
        > L64-6           N6-1-6
          < L34-6*        N4-2-6
```

### Dual Combination Network 6: RSF Output

<div class="scrollbox">
<pre>
Node                      Link
------------------------- ------
L12-6
> L22-6                   N2-1-6
 =  L22-6*                N2-3-6
 >  L23-6                 N2-4-6
   <  L12-6*              N2-2-6
   >  L34-6               N3-1-6
     >  L35-6             N3-3-6
       <  L23-6*          N3-2-6
       >  L57-6           N5-1-6
         >  L58-6         N5-3-6
           <  L35-6*      N5-2-6
           >  L78-6       N8-1-6
             <  L57-6*    N7-1-6
         >  L78-6         N7-1-6
           <  L58-6       N8-1-6
             <  L35-6*    N5-2-6
             <  L57-6*    N5-3-6
       >  L58-6           N5-2-6
         <  L57-6         N5-3-6
           <  L35-6*      N5-1-6
           >  L78-6       N7-1-6
             <  L58-6*    N8-1-6
         >  L78-6         N8-1-6
           <  L57-6       N7-1-6
             <  L35-6*    N5-1-6
             >  L58-6*    N5-3-6
     >  L46-6             N4-1-6
       >  L64-6           N6-1-6
         <  L34-6*        N4-2-6
     >  L64-6             N4-2-6
       <  L46-6           N6-1-6
         <  L34-6*        N4-1-6
   >  L35-6               N3-2-6
     <  L34-6             N3-3-6
       <  L23-6*          N3-1-6
       >  L46-6           N4-1-6
         >  L64-6         N6-1-6
           <  L34-6*      N4-2-6
       >  L64-6           N4-2-6
         <  L46-6         N6-1-6
           <  L34-6*      N4-1-6
     >  L57-6             N5-1-6
       >  L58-6           N5-3-6
         <  L35-6*        N5-2-6
         >  L78-6         N8-1-6
           <  L57-6*      N7-1-6
       >  L78-6           N7-1-6
         <  L58-6         N8-1-6
           <  L35-6*      N5-2-6
           <  L57-6*      N5-3-6
     >  L58-6             N5-2-6
       <  L57-6           N5-3-6
         <  L35-6*        N5-1-6
         >  L78-6         N7-1-6
           <  L58-6*      N8-1-6
       >  L78-6           N8-1-6
         <  L57-6         N7-1-6
           <  L35-6*      N5-1-6
           >  L58-6*      N5-3-6
> L23-6                   N2-2-6
 <  L22-6                 N2-4-6
   <  L12-6*              N2-1-6
   =  L22-6*              N2-3-6
 >  L34-6                 N3-1-6
   >  L35-6               N3-3-6
     <  L23-6*            N3-2-6
     >  L57-6             N5-1-6
       >  L58-6           N5-3-6
         <  L35-6*        N5-2-6
         >  L78-6         N8-1-6
           <  L57-6*      N7-1-6
       >  L78-6           N7-1-6
         <  L58-6         N8-1-6
           <  L35-6*      N5-2-6
           <  L57-6*      N5-3-6
     >  L58-6             N5-2-6
       <  L57-6           N5-3-6
         <  L35-6*        N5-1-6
         >  L78-6         N7-1-6
           <  L58-6*      N8-1-6
       >  L78-6           N8-1-6
         <  L57-6         N7-1-6
           <  L35-6*      N5-1-6
           >  L58-6*      N5-3-6
   >  L46-6               N4-1-6
     >  L64-6             N6-1-6
       <  L34-6*          N4-2-6
   >  L64-6               N4-2-6
     <  L46-6             N6-1-6
       <  L34-6*          N4-1-6
 >  L35-6                 N3-2-6
   <  L34-6               N3-3-6
     <  L23-6*            N3-1-6
     >  L46-6             N4-1-6
       >  L64-6           N6-1-6
         <  L34-6*        N4-2-6
     >  L64-6             N4-2-6
       <  L46-6           N6-1-6
         <  L34-6*        N4-1-6
   >  L57-6               N5-1-6
     >  L58-6             N5-3-6
       <  L35-6*          N5-2-6
       >  L78-6           N8-1-6
         <  L57-6*        N7-1-6
     >  L78-6             N7-1-6
       <  L58-6           N8-1-6
         <  L35-6*        N5-2-6
         <  L57-6*        N5-3-6
   >  L58-6               N5-2-6
     <  L57-6             N5-3-6
       <  L35-6*          N5-1-6
       >  L78-6           N7-1-6
         <  L58-6*        N8-1-6
     >  L78-6             N8-1-6
       <  L57-6           N7-1-6
         <  L35-6*        N5-1-6
         >  L58-6*        N5-3-6

</pre>
</div>

### Combination Network 6: CBY Original with RSF Dual Output

In the output below I deleted all the loop rows from the RSF output for the dual network and placed the result beside the output for CBY for the original network, using a column-wise copy and paste. It's easy to see then their equivalent structure. Both have 69 rows.

<div class="scrollbox">
<pre>
Network 6: CBY                         Dual Network 6: RSF with loop rows deleted
==============                         ==========================================
Nodes           Link Path              Node                      Link
--------------- -------------------- ------------------------- ------
N1-6 > N2-6     L12-6*                 L12-6
N2-6 > N2-6       L22-6*	       > L22-6                   N2-1-6
N2-6 > N3-6         L23-6*	        >  L23-6                 N2-4-6
N3-6 > N4-6           L34-6*	          >  L34-6               N3-1-6
N6-6 > N4-6             L64-6*	            >  L35-6             N3-3-6
N4-6 > N6-6               L46-6*              >  L57-6           N5-1-6
N3-6 > N5-6             L35-6*	                >  L58-6         N5-3-6
N5-6 > N7-6               L57-6*                  >  L78-6       N8-1-6
N5-6 > N8-6                 L58-6*              >  L78-6         N7-1-6
N7-6 > N8-6                   L78-6*              <  L58-6       N8-1-6
N7-6 > N8-6                 L78-6*            >  L58-6           N5-2-6
N5-6 > N8-6                   L58-6*            <  L57-6         N5-3-6
N5-6 > N8-6               L58-6*                  >  L78-6       N7-1-6
N5-6 > N7-6                 L57-6*              >  L78-6         N8-1-6
N7-6 > N8-6                   L78-6*              <  L57-6       N7-1-6
N7-6 > N8-6                 L78-6*          >  L46-6             N4-1-6
N5-6 > N7-6                   L57-6*          >  L64-6           N6-1-6
N4-6 > N6-6             L46-6*	            >  L64-6             N4-2-6
N6-6 > N4-6               L64-6*              <  L46-6           N6-1-6
N3-6 > N5-6           L35-6*	          >  L35-6               N3-2-6
N3-6 > N4-6             L34-6*	            <  L34-6             N3-3-6
N6-6 > N4-6               L64-6*              >  L46-6           N4-1-6
N4-6 > N6-6                 L46-6*              >  L64-6         N6-1-6
N4-6 > N6-6               L46-6*              >  L64-6           N4-2-6
N6-6 > N4-6                 L64-6*              <  L46-6         N6-1-6
N5-6 > N7-6             L57-6*	            >  L57-6             N5-1-6
N5-6 > N8-6               L58-6*              >  L58-6           N5-3-6
N7-6 > N8-6                 L78-6*              >  L78-6         N8-1-6
N7-6 > N8-6               L78-6*              >  L78-6           N7-1-6
N5-6 > N8-6                 L58-6*              <  L58-6         N8-1-6
N5-6 > N8-6             L58-6*	            >  L58-6             N5-2-6
N5-6 > N7-6               L57-6*              <  L57-6           N5-3-6
N7-6 > N8-6                 L78-6*              >  L78-6         N7-1-6
N7-6 > N8-6               L78-6*              >  L78-6           N8-1-6
N5-6 > N7-6                 L57-6*              <  L57-6         N7-1-6
N2-6 > N3-6       L23-6*	       > L23-6                   N2-2-6
N2-6 > N2-6         L22-6*	        <  L22-6                 N2-4-6
N3-6 > N4-6         L34-6*	        >  L34-6                 N3-1-6
N6-6 > N4-6           L64-6*	          >  L35-6               N3-3-6
N4-6 > N6-6             L46-6*	            >  L57-6             N5-1-6
N3-6 > N5-6           L35-6*	              >  L58-6           N5-3-6
N5-6 > N7-6             L57-6*	                >  L78-6         N8-1-6
N5-6 > N8-6               L58-6*              >  L78-6           N7-1-6
N7-6 > N8-6                 L78-6*              <  L58-6         N8-1-6
N7-6 > N8-6               L78-6*            >  L58-6             N5-2-6
N5-6 > N8-6                 L58-6*            <  L57-6           N5-3-6
N5-6 > N8-6             L58-6*	                >  L78-6         N7-1-6
N5-6 > N7-6               L57-6*              >  L78-6           N8-1-6
N7-6 > N8-6                 L78-6*              <  L57-6         N7-1-6
N7-6 > N8-6               L78-6*          >  L46-6               N4-1-6
N5-6 > N7-6                 L57-6*          >  L64-6             N6-1-6
N4-6 > N6-6           L46-6*	          >  L64-6               N4-2-6
N6-6 > N4-6             L64-6*	            <  L46-6             N6-1-6
N3-6 > N5-6         L35-6*	        >  L35-6                 N3-2-6
N3-6 > N4-6           L34-6*	          <  L34-6               N3-3-6
N6-6 > N4-6             L64-6*	            >  L46-6             N4-1-6
N4-6 > N6-6               L46-6*              >  L64-6           N6-1-6
N4-6 > N6-6             L46-6*	            >  L64-6             N4-2-6
N6-6 > N4-6               L64-6*              <  L46-6           N6-1-6
N5-6 > N7-6           L57-6*	          >  L57-6               N5-1-6
N5-6 > N8-6             L58-6*	            >  L58-6             N5-3-6
N7-6 > N8-6               L78-6*              >  L78-6           N8-1-6
N7-6 > N8-6             L78-6*	            >  L78-6             N7-1-6
N5-6 > N8-6               L58-6*              <  L58-6           N8-1-6
N5-6 > N8-6           L58-6*	          >  L58-6               N5-2-6
N5-6 > N7-6             L57-6*	            <  L57-6             N5-3-6
N7-6 > N8-6               L78-6*              >  L78-6           N7-1-6
N7-6 > N8-6             L78-6*	            >  L78-6             N8-1-6
N5-6 > N7-6               L57-6*              <  L57-6           N7-1-6
</pre>
</div>

### Dual Combination Network 6: CBY Output

34547 rows selected.

\[See SQL files and output files in [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/recursive-sql-for-network-analysis-and-duality)\].

## Oracle's HR/OE/PM Demo Network

### Original Demo Network

<img src="/migrated_images/2015/09/Dual-Network-1.3-HR.png" alt="Dual Network, 1.3 - HR" title="Dual Network, 1.3 - HR" />

This network has 21 links with 6 loops.

### Original Demo Network: PLF Output

```
Node                                          Link                                 Lev
--------------------------------------------- ----------------------------------- ----
COUNTRIES|HR                                  ROOT                                   0
< LOCATIONS|HR                                loc_c_id_fk|HR                         1
  < DEPARTMENTS|HR                            dept_loc_fk|HR                         2
    > EMPLOYEES|HR                            dept_mgr_fk|HR                         3
      < CUSTOMERS|OE                          customers_account_manager_fk|OE        4
        < ORDERS|OE                           orders_customer_id_fk|OE               5
          > EMPLOYEES|HR*                     orders_sales_rep_fk|OE                 6
          < ORDER_ITEMS|OE                    order_items_order_id_fk|OE             6
            > PRODUCT_INFORMATION|OE          order_items_product_id_fk|OE           7
              < INVENTORIES|OE                inventories_product_id_fk|OE           8
                > WAREHOUSES|OE               inventories_warehouses_fk|OE           9
                  > LOCATIONS|HR*             warehouses_location_fk|OE             10
              < ONLINE_MEDIA|PM               loc_c_id_fk|PM                         8
              < PRINT_MEDIA|PM                printmedia_fk|PM                       8
              < PRODUCT_DESCRIPTIONS|OE       pd_product_id_fk|OE                    8
      > DEPARTMENTS|HR*                       emp_dept_fk|HR                         4
      = EMPLOYEES|HR*                         emp_manager_fk|HR                      4
      > JOBS|HR                               emp_job_fk|HR                          4
        < JOB_HISTORY|HR                      jhist_job_fk|HR                        5
          > DEPARTMENTS|HR*                   jhist_dept_fk|HR                       6
          > EMPLOYEES|HR*                     jhist_emp_fk|HR                        6
> REGIONS|HR                                  countr_reg_fk|HR                       1

22 rows selected.

Elapsed: 00:00:00.15
```

### Original Demo Network: RSF Output

<div class="scrollbox">
<pre>
Node                                          Link
--------------------------------------------- -----------------------------------
COUNTRIES|HR
< LOCATIONS|HR                                loc_c_id_fk|HR
  < DEPARTMENTS|HR                            dept_loc_fk|HR
    < EMPLOYEES|HR                            emp_dept_fk|HR
      < CUSTOMERS|OE                          customers_account_manager_fk|OE
        < ORDERS|OE                           orders_customer_id_fk|OE
          > EMPLOYEES|HR*                     orders_sales_rep_fk|OE
          < ORDER_ITEMS|OE                    order_items_order_id_fk|OE
            > PRODUCT_INFORMATION|OE          order_items_product_id_fk|OE
              < INVENTORIES|OE                inventories_product_id_fk|OE
                > WAREHOUSES|OE               inventories_warehouses_fk|OE
                  > LOCATIONS|HR*             warehouses_location_fk|OE
              < ONLINE_MEDIA|PM               loc_c_id_fk|PM
              < PRINT_MEDIA|PM                printmedia_fk|PM
              < PRODUCT_DESCRIPTIONS|OE       pd_product_id_fk|OE
      < DEPARTMENTS|HR*                       dept_mgr_fk|HR
      = EMPLOYEES|HR*                         emp_manager_fk|HR
      > JOBS|HR                               emp_job_fk|HR
        < JOB_HISTORY|HR                      jhist_job_fk|HR
          > DEPARTMENTS|HR*                   jhist_dept_fk|HR
          > EMPLOYEES|HR*                     jhist_emp_fk|HR
      < JOB_HISTORY|HR                        jhist_emp_fk|HR
        > DEPARTMENTS|HR*                     jhist_dept_fk|HR
        > JOBS|HR                             jhist_job_fk|HR
          < EMPLOYEES|HR*                     emp_job_fk|HR
      < ORDERS|OE                             orders_sales_rep_fk|OE
        > CUSTOMERS|OE                        orders_customer_id_fk|OE
          > EMPLOYEES|HR*                     customers_account_manager_fk|OE
        < ORDER_ITEMS|OE                      order_items_order_id_fk|OE
          > PRODUCT_INFORMATION|OE            order_items_product_id_fk|OE
            < INVENTORIES|OE                  inventories_product_id_fk|OE
              > WAREHOUSES|OE                 inventories_warehouses_fk|OE
                > LOCATIONS|HR*               warehouses_location_fk|OE
            < ONLINE_MEDIA|PM                 loc_c_id_fk|PM
            < PRINT_MEDIA|PM                  printmedia_fk|PM
            < PRODUCT_DESCRIPTIONS|OE         pd_product_id_fk|OE
    > EMPLOYEES|HR                            dept_mgr_fk|HR
      < CUSTOMERS|OE                          customers_account_manager_fk|OE
        < ORDERS|OE                           orders_customer_id_fk|OE
          > EMPLOYEES|HR*                     orders_sales_rep_fk|OE
          < ORDER_ITEMS|OE                    order_items_order_id_fk|OE
            > PRODUCT_INFORMATION|OE          order_items_product_id_fk|OE
              < INVENTORIES|OE                inventories_product_id_fk|OE
                > WAREHOUSES|OE               inventories_warehouses_fk|OE
                  > LOCATIONS|HR*             warehouses_location_fk|OE
              < ONLINE_MEDIA|PM               loc_c_id_fk|PM
              < PRINT_MEDIA|PM                printmedia_fk|PM
              < PRODUCT_DESCRIPTIONS|OE       pd_product_id_fk|OE
      > DEPARTMENTS|HR*                       emp_dept_fk|HR
      = EMPLOYEES|HR*                         emp_manager_fk|HR
      > JOBS|HR                               emp_job_fk|HR
        < JOB_HISTORY|HR                      jhist_job_fk|HR
          > DEPARTMENTS|HR*                   jhist_dept_fk|HR
          > EMPLOYEES|HR*                     jhist_emp_fk|HR
      < JOB_HISTORY|HR                        jhist_emp_fk|HR
        > DEPARTMENTS|HR*                     jhist_dept_fk|HR
        > JOBS|HR                             jhist_job_fk|HR
          < EMPLOYEES|HR*                     emp_job_fk|HR
      < ORDERS|OE                             orders_sales_rep_fk|OE
        > CUSTOMERS|OE                        orders_customer_id_fk|OE
          > EMPLOYEES|HR*                     customers_account_manager_fk|OE
        < ORDER_ITEMS|OE                      order_items_order_id_fk|OE
          > PRODUCT_INFORMATION|OE            order_items_product_id_fk|OE
            < INVENTORIES|OE                  inventories_product_id_fk|OE
              > WAREHOUSES|OE                 inventories_warehouses_fk|OE
                > LOCATIONS|HR*               warehouses_location_fk|OE
            < ONLINE_MEDIA|PM                 loc_c_id_fk|PM
            < PRINT_MEDIA|PM                  printmedia_fk|PM
            < PRODUCT_DESCRIPTIONS|OE         pd_product_id_fk|OE
    < JOB_HISTORY|HR                          jhist_dept_fk|HR
      > EMPLOYEES|HR                          jhist_emp_fk|HR
        < CUSTOMERS|OE                        customers_account_manager_fk|OE
          < ORDERS|OE                         orders_customer_id_fk|OE
            > EMPLOYEES|HR*                   orders_sales_rep_fk|OE
            < ORDER_ITEMS|OE                  order_items_order_id_fk|OE
              > PRODUCT_INFORMATION|OE        order_items_product_id_fk|OE
                < INVENTORIES|OE              inventories_product_id_fk|OE
                  > WAREHOUSES|OE             inventories_warehouses_fk|OE
                    > LOCATIONS|HR*           warehouses_location_fk|OE
                < ONLINE_MEDIA|PM             loc_c_id_fk|PM
                < PRINT_MEDIA|PM              printmedia_fk|PM
                < PRODUCT_DESCRIPTIONS|OE     pd_product_id_fk|OE
        < DEPARTMENTS|HR*                     dept_mgr_fk|HR
        > DEPARTMENTS|HR*                     emp_dept_fk|HR
        = EMPLOYEES|HR*                       emp_manager_fk|HR
        > JOBS|HR                             emp_job_fk|HR
          < JOB_HISTORY|HR*                   jhist_job_fk|HR
        < ORDERS|OE                           orders_sales_rep_fk|OE
          > CUSTOMERS|OE                      orders_customer_id_fk|OE
            > EMPLOYEES|HR*                   customers_account_manager_fk|OE
          < ORDER_ITEMS|OE                    order_items_order_id_fk|OE
            > PRODUCT_INFORMATION|OE          order_items_product_id_fk|OE
              < INVENTORIES|OE                inventories_product_id_fk|OE
                > WAREHOUSES|OE               inventories_warehouses_fk|OE
                  > LOCATIONS|HR*             warehouses_location_fk|OE
              < ONLINE_MEDIA|PM               loc_c_id_fk|PM
              < PRINT_MEDIA|PM                printmedia_fk|PM
              < PRODUCT_DESCRIPTIONS|OE       pd_product_id_fk|OE
      > JOBS|HR                               jhist_job_fk|HR
        < EMPLOYEES|HR                        emp_job_fk|HR
          < CUSTOMERS|OE                      customers_account_manager_fk|OE
            < ORDERS|OE                       orders_customer_id_fk|OE
              > EMPLOYEES|HR*                 orders_sales_rep_fk|OE
              < ORDER_ITEMS|OE                order_items_order_id_fk|OE
                > PRODUCT_INFORMATION|OE      order_items_product_id_fk|OE
                  < INVENTORIES|OE            inventories_product_id_fk|OE
                    > WAREHOUSES|OE           inventories_warehouses_fk|OE
                      > LOCATIONS|HR*         warehouses_location_fk|OE
                  < ONLINE_MEDIA|PM           loc_c_id_fk|PM
                  < PRINT_MEDIA|PM            printmedia_fk|PM
                  < PRODUCT_DESCRIPTIONS|OE   pd_product_id_fk|OE
          < DEPARTMENTS|HR*                   dept_mgr_fk|HR
          > DEPARTMENTS|HR*                   emp_dept_fk|HR
          = EMPLOYEES|HR*                     emp_manager_fk|HR
          < JOB_HISTORY|HR*                   jhist_emp_fk|HR
          < ORDERS|OE                         orders_sales_rep_fk|OE
            > CUSTOMERS|OE                    orders_customer_id_fk|OE
              > EMPLOYEES|HR*                 customers_account_manager_fk|OE
            < ORDER_ITEMS|OE                  order_items_order_id_fk|OE
              > PRODUCT_INFORMATION|OE        order_items_product_id_fk|OE
                < INVENTORIES|OE              inventories_product_id_fk|OE
                  > WAREHOUSES|OE             inventories_warehouses_fk|OE
                    > LOCATIONS|HR*           warehouses_location_fk|OE
                < ONLINE_MEDIA|PM             loc_c_id_fk|PM
                < PRINT_MEDIA|PM              printmedia_fk|PM
                < PRODUCT_DESCRIPTIONS|OE     pd_product_id_fk|OE
  < WAREHOUSES|OE                             warehouses_location_fk|OE
    < INVENTORIES|OE                          inventories_warehouses_fk|OE
      > PRODUCT_INFORMATION|OE                inventories_product_id_fk|OE
        < ONLINE_MEDIA|PM                     loc_c_id_fk|PM
        < ORDER_ITEMS|OE                      order_items_product_id_fk|OE
          > ORDERS|OE                         order_items_order_id_fk|OE
            > CUSTOMERS|OE                    orders_customer_id_fk|OE
              > EMPLOYEES|HR                  customers_account_manager_fk|OE
                < DEPARTMENTS|HR              dept_mgr_fk|HR
                  < EMPLOYEES|HR*             emp_dept_fk|HR
                  < JOB_HISTORY|HR            jhist_dept_fk|HR
                    > EMPLOYEES|HR*           jhist_emp_fk|HR
                    > JOBS|HR                 jhist_job_fk|HR
                      < EMPLOYEES|HR*         emp_job_fk|HR
                  > LOCATIONS|HR*             dept_loc_fk|HR
                > DEPARTMENTS|HR              emp_dept_fk|HR
                  > EMPLOYEES|HR*             dept_mgr_fk|HR
                  < JOB_HISTORY|HR            jhist_dept_fk|HR
                    > EMPLOYEES|HR*           jhist_emp_fk|HR
                    > JOBS|HR                 jhist_job_fk|HR
                      < EMPLOYEES|HR*         emp_job_fk|HR
                  > LOCATIONS|HR*             dept_loc_fk|HR
                = EMPLOYEES|HR*               emp_manager_fk|HR
                > JOBS|HR                     emp_job_fk|HR
                  < JOB_HISTORY|HR            jhist_job_fk|HR
                    > DEPARTMENTS|HR          jhist_dept_fk|HR
                      < EMPLOYEES|HR*         emp_dept_fk|HR
                      > EMPLOYEES|HR*         dept_mgr_fk|HR
                      > LOCATIONS|HR*         dept_loc_fk|HR
                    > EMPLOYEES|HR*           jhist_emp_fk|HR
                < JOB_HISTORY|HR              jhist_emp_fk|HR
                  > DEPARTMENTS|HR            jhist_dept_fk|HR
                    < EMPLOYEES|HR*           emp_dept_fk|HR
                    > EMPLOYEES|HR*           dept_mgr_fk|HR
                    > LOCATIONS|HR*           dept_loc_fk|HR
                  > JOBS|HR                   jhist_job_fk|HR
                    < EMPLOYEES|HR*           emp_job_fk|HR
                < ORDERS|OE*                  orders_sales_rep_fk|OE
            > EMPLOYEES|HR                    orders_sales_rep_fk|OE
              < CUSTOMERS|OE                  customers_account_manager_fk|OE
                < ORDERS|OE*                  orders_customer_id_fk|OE
              < DEPARTMENTS|HR                dept_mgr_fk|HR
                < EMPLOYEES|HR*               emp_dept_fk|HR
                < JOB_HISTORY|HR              jhist_dept_fk|HR
                  > EMPLOYEES|HR*             jhist_emp_fk|HR
                  > JOBS|HR                   jhist_job_fk|HR
                    < EMPLOYEES|HR*           emp_job_fk|HR
                > LOCATIONS|HR*               dept_loc_fk|HR
              > DEPARTMENTS|HR                emp_dept_fk|HR
                > EMPLOYEES|HR*               dept_mgr_fk|HR
                < JOB_HISTORY|HR              jhist_dept_fk|HR
                  > EMPLOYEES|HR*             jhist_emp_fk|HR
                  > JOBS|HR                   jhist_job_fk|HR
                    < EMPLOYEES|HR*           emp_job_fk|HR
                > LOCATIONS|HR*               dept_loc_fk|HR
              = EMPLOYEES|HR*                 emp_manager_fk|HR
              > JOBS|HR                       emp_job_fk|HR
                < JOB_HISTORY|HR              jhist_job_fk|HR
                  > DEPARTMENTS|HR            jhist_dept_fk|HR
                    < EMPLOYEES|HR*           emp_dept_fk|HR
                    > EMPLOYEES|HR*           dept_mgr_fk|HR
                    > LOCATIONS|HR*           dept_loc_fk|HR
                  > EMPLOYEES|HR*             jhist_emp_fk|HR
              < JOB_HISTORY|HR                jhist_emp_fk|HR
                > DEPARTMENTS|HR              jhist_dept_fk|HR
                  < EMPLOYEES|HR*             emp_dept_fk|HR
                  > EMPLOYEES|HR*             dept_mgr_fk|HR
                  > LOCATIONS|HR*             dept_loc_fk|HR
                > JOBS|HR                     jhist_job_fk|HR
                  < EMPLOYEES|HR*             emp_job_fk|HR
        < PRINT_MEDIA|PM                      printmedia_fk|PM
        < PRODUCT_DESCRIPTIONS|OE             pd_product_id_fk|OE
> REGIONS|HR                                  countr_reg_fk|HR

199 rows selected.

Elapsed: 00:00:00.30
</pre>
</div>

The output above shows that RSF returned 199 rows unfiltered in 0.3s.

### Original Demo Network: CBY Output

<div class="scrollbox">
<pre>
One tree by Connect By

Nodes                                              Link Path
-------------------------------------------------- ----------------------------------------------------------------------
COUNTRIES|HR > REGIONS|HR                          countr_reg_fk|HR*
LOCATIONS|HR > COUNTRIES|HR                          loc_c_id_fk|HR*
DEPARTMENTS|HR > LOCATIONS|HR                          dept_loc_fk|HR*
EMPLOYEES|HR > DEPARTMENTS|HR                            emp_dept_fk|HR*
JOB_HISTORY|HR > DEPARTMENTS|HR                            jhist_dept_fk|HR*
DEPARTMENTS|HR > EMPLOYEES|HR                                dept_mgr_fk|HR*
EMPLOYEES|HR > EMPLOYEES|HR                                    emp_manager_fk|HR*
CUSTOMERS|OE > EMPLOYEES|HR                                      customers_account_manager_fk|OE*
ORDERS|OE > CUSTOMERS|OE                                           orders_customer_id_fk|OE*
ORDERS|OE > EMPLOYEES|HR                                             orders_sales_rep_fk|OE*
JOB_HISTORY|HR > EMPLOYEES|HR                                          jhist_emp_fk|HR*
EMPLOYEES|HR > JOBS|HR                                                   emp_job_fk|HR*
JOB_HISTORY|HR > JOBS|HR                                                   jhist_job_fk|HR*
JOB_HISTORY|HR > JOBS|HR                                                 jhist_job_fk|HR*
EMPLOYEES|HR > JOBS|HR                                                     emp_job_fk|HR*
EMPLOYEES|HR > JOBS|HR                                                 emp_job_fk|HR*
JOB_HISTORY|HR > EMPLOYEES|HR                                            jhist_emp_fk|HR*
JOB_HISTORY|HR > JOBS|HR                                                   jhist_job_fk|HR*
JOB_HISTORY|HR > JOBS|HR                                                 jhist_job_fk|HR*
JOB_HISTORY|HR > EMPLOYEES|HR                                              jhist_emp_fk|HR*
ORDER_ITEMS|OE > ORDERS|OE                                             order_items_order_id_fk|OE*
ORDER_ITEMS|OE > PRODUCT_INFORMATION|OE                                  order_items_product_id_fk|OE*
INVENTORIES|OE > PRODUCT_INFORMATION|OE                                    inventories_product_id_fk|OE*
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                                      printmedia_fk|PM*
.
.
.
ORDERS|OE > CUSTOMERS|OE                                                           orders_customer_id_fk|OE*
EMPLOYEES|HR > EMPLOYEES|HR                                                        emp_manager_fk|HR*
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                        printmedia_fk|PM*
ONLINE_MEDIA|PM > PRODUCT_INFORMATION|OE                         loc_c_id_fk|PM*
PRODUCT_DESCRIPTIONS|OE > PRODUCT_INFORMATION|OE                   pd_product_id_fk|OE*
PRODUCT_DESCRIPTIONS|OE > PRODUCT_INFORMATION|OE                 pd_product_id_fk|OE*
ONLINE_MEDIA|PM > PRODUCT_INFORMATION|OE                           loc_c_id_fk|PM*
ONLINE_MEDIA|PM > PRODUCT_INFORMATION|OE                       loc_c_id_fk|PM*
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                          printmedia_fk|PM*
PRODUCT_DESCRIPTIONS|OE > PRODUCT_INFORMATION|OE                   pd_product_id_fk|OE*
PRODUCT_DESCRIPTIONS|OE > PRODUCT_INFORMATION|OE                 pd_product_id_fk|OE*
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                            printmedia_fk|PM*
PRODUCT_DESCRIPTIONS|OE > PRODUCT_INFORMATION|OE               pd_product_id_fk|OE*
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                          printmedia_fk|PM*
ONLINE_MEDIA|PM > PRODUCT_INFORMATION|OE                           loc_c_id_fk|PM*
ONLINE_MEDIA|PM > PRODUCT_INFORMATION|OE                         loc_c_id_fk|PM*
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                            printmedia_fk|PM*

4414420 rows selected.

Elapsed: 00:29:33.41

One tree by Connect By filtered

Nodes                                              Link Path                                                              LINK_COUNT
-------------------------------------------------- ---------------------------------------------------------------------- ----------
COUNTRIES|HR > REGIONS|HR                          countr_reg_fk|HR*                                                               1
LOCATIONS|HR > COUNTRIES|HR                          loc_c_id_fk|HR*                                                               1
DEPARTMENTS|HR > LOCATIONS|HR                          dept_loc_fk|HR*                                                        214178
EMPLOYEES|HR > DEPARTMENTS|HR                            emp_dept_fk|HR*                                                      169932
JOB_HISTORY|HR > DEPARTMENTS|HR                            jhist_dept_fk|HR*                                                  272162
DEPARTMENTS|HR > EMPLOYEES|HR                                dept_mgr_fk|HR*                                                  169932
EMPLOYEES|HR > EMPLOYEES|HR                                    emp_manager_fk|HR*                                             207910
CUSTOMERS|OE > EMPLOYEES|HR                                      customers_account_manager_fk|OE*                             132490
ORDERS|OE > CUSTOMERS|OE                                           orders_customer_id_fk|OE*                                   85298
ORDERS|OE > EMPLOYEES|HR                                             orders_sales_rep_fk|OE*                                   72234
JOB_HISTORY|HR > EMPLOYEES|HR                                          jhist_emp_fk|HR*                                       164660
EMPLOYEES|HR > JOBS|HR                                                   emp_job_fk|HR*                                       182784
JOB_HISTORY|HR > JOBS|HR                                                   jhist_job_fk|HR*                                   333192
ORDER_ITEMS|OE > ORDERS|OE                                             order_items_order_id_fk|OE*                             26804
ORDER_ITEMS|OE > PRODUCT_INFORMATION|OE                                  order_items_product_id_fk|OE*                         26804
INVENTORIES|OE > PRODUCT_INFORMATION|OE                                    inventories_product_id_fk|OE*                      428354
PRINT_MEDIA|PM > PRODUCT_INFORMATION|OE                                      printmedia_fk|PM*                                428384
ONLINE_MEDIA|PM > PRODUCT_INFORMATION|OE                                       loc_c_id_fk|PM*                                428384
PRODUCT_DESCRIPTIONS|OE > PRODUCT_INFORMATION|OE                                 pd_product_id_fk|OE*                         428384
INVENTORIES|OE > WAREHOUSES|OE                                               inventories_warehouses_fk|OE*                    428354
WAREHOUSES|OE > LOCATIONS|HR                                                   warehouses_location_fk|OE*                     214178

21 rows selected.

Elapsed: 00:03:03.16
</pre>
</div>

The output above shows that CBY returned 4,414,420 rows unfiltered in 29m33s. Adding filtering reduced the time to 3m03s.

### Dual Demo Network

<img src="/migrated_images/2015/09/Dual-Network-1.3-HR-D.png" alt="Dual Network, 1.3 - HR-D" title="Dual Network, 1.3 - HR-D" />

This network has 52 links with 32 loops, whereas the original had 21 links with 6 loops.

### Dual Demo Network: PLF Output

<div class="scrollbox">
<pre>
Node                                                           Link
-------------------------------------------------------------- -------------------------
countr_reg_fk|HR                                               ROOT
> loc_c_id_fk|HR                                               COUNTRIES|HR-1
  < dept_loc_fk|HR                                             LOCATIONS|HR-1
    > dept_mgr_fk|HR                                           DEPARTMENTS|HR-1
      < customers_account_manager_fk|OE                        EMPLOYEES|HR-1
        > emp_dept_fk|HR                                       EMPLOYEES|HR-2
          < dept_loc_fk|HR*                                    DEPARTMENTS|HR-2
          < dept_mgr_fk|HR*                                    EMPLOYEES|HR-7
          > emp_job_fk|HR                                      EMPLOYEES|HR-12
            < customers_account_manager_fk|OE*                 EMPLOYEES|HR-3
            < dept_mgr_fk|HR*                                  EMPLOYEES|HR-8
            > emp_manager_fk|HR                                EMPLOYEES|HR-16
              < customers_account_manager_fk|OE*               EMPLOYEES|HR-4
              < dept_mgr_fk|HR*                                EMPLOYEES|HR-9
              < emp_dept_fk|HR*                                EMPLOYEES|HR-13
              = emp_manager_fk|HR*                             EMPLOYEES|HR-19
              > jhist_emp_fk|HR                                EMPLOYEES|HR-20
                < customers_account_manager_fk|OE*             EMPLOYEES|HR-5
                < dept_mgr_fk|HR*                              EMPLOYEES|HR-10
                < emp_dept_fk|HR*                              EMPLOYEES|HR-14
                < emp_job_fk|HR*                               EMPLOYEES|HR-17
                < jhist_dept_fk|HR                             JOB_HISTORY|HR-1
                  < dept_loc_fk|HR*                            DEPARTMENTS|HR-3
                  < dept_mgr_fk|HR*                            DEPARTMENTS|HR-4
                  < emp_dept_fk|HR*                            DEPARTMENTS|HR-5
                  > jhist_job_fk|HR                            JOB_HISTORY|HR-2
                    < emp_job_fk|HR*                           JOBS|HR-1
                    < jhist_emp_fk|HR*                         JOB_HISTORY|HR-3
                > orders_sales_rep_fk|OE                       EMPLOYEES|HR-22
                  < customers_account_manager_fk|OE*           EMPLOYEES|HR-6
                  < dept_mgr_fk|HR*                            EMPLOYEES|HR-11
                  < emp_dept_fk|HR*                            EMPLOYEES|HR-15
                  < emp_job_fk|HR*                             EMPLOYEES|HR-18
                  < emp_manager_fk|HR*                         EMPLOYEES|HR-21
                  < order_items_order_id_fk|OE                 ORDERS|OE-2
                    > order_items_product_id_fk|OE             ORDER_ITEMS|OE-1
                      < inventories_product_id_fk|OE           PRODUCT_INFORMATION|OE-2
                        > inventories_warehouses_fk|OE         INVENTORIES|OE-1
                          > warehouses_location_fk|OE          WAREHOUSES|OE-1
                            < dept_loc_fk|HR*                  LOCATIONS|HR-2
                            < loc_c_id_fk|HR*                  LOCATIONS|HR-3
                        > loc_c_id_fk|PM                       PRODUCT_INFORMATION|OE-1
                          > order_items_product_id_fk|OE*      PRODUCT_INFORMATION|OE-5
                          > pd_product_id_fk|OE                PRODUCT_INFORMATION|OE-6
                            < inventories_product_id_fk|OE*    PRODUCT_INFORMATION|OE-3
                            < order_items_product_id_fk|OE*    PRODUCT_INFORMATION|OE-8
                            > printmedia_fk|PM                 PRODUCT_INFORMATION|OE-10
                              < inventories_product_id_fk|OE*  PRODUCT_INFORMATION|OE-4
                              < loc_c_id_fk|PM*                PRODUCT_INFORMATION|OE-7
                              < order_items_product_id_fk|OE*  PRODUCT_INFORMATION|OE-9
                    > orders_customer_id_fk|OE                 ORDERS|OE-1
                      < customers_account_manager_fk|OE*       CUSTOMERS|OE-1
                      > orders_sales_rep_fk|OE*                ORDERS|OE-3

53 rows selected.

Elapsed: 00:00:00.27
</pre>
</div>

### Dual Demo Network: RSF and CBY Results

Neither of the two SQL recursion methods completed within a period of an hour and had to be terminated. The result for CBY on the original network suggests that RSF on the dual network should return somewhere above 4,414,420 rows.

## Conclusion

- We have shown by examples how network traversal by the Connect By (CBY) approach in SQL corresponds to traversal of all routes in a type of dual version of the original network
- This dual version, which has forks converted to loops, tends to be larger and more heavily looped, resulting in worse performance compared with solution by recursive subquery factors (RSF)
- The examples illustrate the different treatment of loop-closing links between the two types of SQL recursion
- The RSF solutions on the dual network in the simpler examples where it completes is seen to be equivalent to the CBY solution on the original network, after allowing for the different treatment of loop-closing links
- On the foreign key network for Oracle's HR/OE/PM demo, which has 21 links, RSF returns 199 rows while CBY returns 4,414,420 rows
- On the dual version of the foreign key network for Oracle's HR/OE/PM demo, which has 52 links, RSF and CBY fail to complete in reasonable times
- The pipelined function method returns the solution on both original and dual in a small fraction of a second

SQL files and output files in [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/recursive-sql-for-network-analysis-and-duality).

Oracle version used: Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production
