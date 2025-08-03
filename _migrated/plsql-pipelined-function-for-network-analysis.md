---
layout: post
title: "PL/SQL Pipelined Function for Network Analysis"
date: 2015-05-10
migrated: true
group: recursive
categories: 
  - "oracle"
  - "pipelined"
  - "plsql"
  - "recursive"
  - "sql"
tags: 
  - "grouping"
  - "network"
  - "plsql"
  - "recursive"
  - "sql"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">

In March 2013 I wrote this [SQL for Network Grouping](https://brenpatf.github.io/migrated/sql-for-network-grouping/), describing some options in SQL for solving a general class of network problems, while noting that these would have serious performance issues, and that the most efficient approach would involve PL/SQL. I had already published a Scribd article, in June 2010, on using PL/SQL to traverse a single connected network, [An Oracle Network Traversal PL SQL Program](http://www.scribd.com/doc/32976987/An-Oracle-Network-Traversal-PL-SQL-Program), but had not at that time extended the approach to cover all networks.

Last weekend, I posted an article, [SQL for Shortest Path Problems 2: A Branch and Bound Approach](https://brenpatf.github.io/sql-for-shortest-path-problems-2-a-branch-and-bound-approach/), that included results from my more general package for network analysis, so I thought it was time to post an article on that package, and this is it. GitHub: [Brendan's network structural analysis Oracle package](https://github.com/BrenPatF/plsql_network)

**July 2025**: See also this 2022 article of mine, [Shortest Path Analysis of Large Networks by SQL and PL/SQL](https://brenpatf.github.io/2022/08/07/shortest-path-analysis-of-large-networks-by-sql-and-plsql.html)

## PL/SQL Package

### Code

<div class="scrollbox">
<pre>
CREATE OR REPLACE PACKAGE Net_Pipe AS
/**************************************************************************************************

Author:         Brendan Furey
Date:           10 May 2015
Description:    Brendan's network analysis PL/SQL package, (https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/).
                Pipelined function returns a record for each link in all connected subnetworks 
		specified by the view links_v. The root_node_id field identifies the subnetwork that
		a link belongs to. Use SQL to list the network in detail, or at any desired level of
		aggregation. Here is an example call:

SQL
===
SELECT root_node_id             "Network",
       Count (DISTINCT link_id) OVER (PARTITION BY root_node_id) - 1 "#Links",
       Count (DISTINCT node_id) OVER (PARTITION BY root_node_id) "#Nodes",
       node_level "Lev",
       LPad (dirn || ' ', Least (2*node_level, 60), ' ') || node_id || loop_flag "Node",
       link_id                  "Link"
  FROM TABLE (Net_Pipe.All_Nets)
 ORDER BY line_no

Output sample for an Oracle v12.1 foreign key network
=====================================================
Network       #Links  #Nodes Lev  Node                                     Link
------------ ------ ------ --- --------------------------------------- -------------------------------
COUNTRIES|HR      21      16   0  COUNTRIES|HR                             ROOT
                               1  < LOCATIONS|HR                           loc_c_id_fk|hr
                               2    < DEPARTMENTS|HR                       dept_loc_fk|hr                                3      > EMPLOYEES|HR                       dept_mgr_fk|hr
                               4        < CUSTOMERS|OE                     customers_account_manager_fk|oe
                               5          < ORDERS|OE                      orders_customer_id_fk|oe                                6            > EMPLOYEES|HR*                orders_sales_rep_fk|oe
                               6            < ORDER_ITEMS|OE               order_items_order_id_fk|oe                                7              > PRODUCT_INFORMATION|OE     order_items_product_id_fk|oe
                               8                < INVENTORIES|OE           inventories_product_id_fk|oe                                9                  > WAREHOUSES|OE          inventories_warehouses_fk|oe
                              10                    > LOCATIONS|HR*        warehouses_location_fk|oe
                               8                < ONLINE_MEDIA|PM          loc_c_id_fk|pm
                               8                < PRINT_MEDIA|PM           printmedia_fk|pm
                               8                < PRODUCT_DESCRIPTIONS|OE  pd_product_id_fk|oe                                4        > DEPARTMENTS|HR*                  emp_dept_fk|hr
                               4        = EMPLOYEES|HR*                    emp_manager_fk|hr
                               4        > JOBS|HR                          emp_job_fk|hr
                               5          < JOB_HISTORY|HR                 jhist_job_fk|hr                                6            > DEPARTMENTS|HR*              jhist_dept_fk|hr
                               6            > EMPLOYEES|HR*                jhist_emp_fk|hr
                               1  > REGIONS|HR                             countr_reg_fk|hr

1 of 69 subnetworks is shown above
***************************************************************************************************/

TYPE net_rec_type IS RECORD (
		root_node_id				VARCHAR2(100),
		dirn					VARCHAR2(1),
		node_id					VARCHAR2(100),
		link_id					VARCHAR2(100),
                node_level                              NUMBER,
		loop_flag				VARCHAR2(1),
		line_no 				NUMBER
);
TYPE net_tab_type IS TABLE OF net_rec_type;

FUNCTION All_Nets RETURN net_tab_type PIPELINED;

END Net_Pipe;
/
SHO ERR
CREATE OR REPLACE PACKAGE BODY Net_Pipe AS
/**************************************************************************************************

Author:         Brendan Furey
Date:           10 May 2015
Description:    Brendan's network analysis PL/SQL package, (http://aprogrammerwrites.eu/?p=1426).
                Pipelined function returns a record for each link in all connected subnetworks 
		specified by the view links_v. The root_node_id field identifies the subnetwork that
		a link belongs to. Use SQL to list the network in detail, or at any desired level of
		aggregation. See spec for an example call.

***************************************************************************************************/

c_root_link_id          CONSTANT VARCHAR2(100) := 'ROOT';
c_loop_flag             CONSTANT VARCHAR2(1) := '*';
c_dirn_fr               CONSTANT VARCHAR2(1) := '<'; c_dirn_to               CONSTANT VARCHAR2(1) := '>';
c_dirn_sj               CONSTANT VARCHAR2(1) := '=';

g_root_node_id                   VARCHAR2(100);
g_line_no                        NUMBER;

PROCEDURE Write_Log (p_line VARCHAR2) IS
BEGIN

  DBMS_Output.Put_Line (p_line);

END Write_Log;

FUNCTION All_Nets RETURN net_tab_type PIPELINED IS
  g_net_tab				net_tab_type;
  TYPE id_hash_type IS                  TABLE OF PLS_INTEGER INDEX BY VARCHAR2(61);
  g_node_hash                           id_hash_type;
  g_link_hash                           id_hash_type;

  l_is_loop                             BOOLEAN;

  PROCEDURE Init_Net IS
  BEGIN

    g_net_tab := net_tab_type ();
    g_link_hash.DELETE;

  END Init_net;

  PROCEDURE Add_Net (p_root_node_id     VARCHAR2,
                     p_dirn             VARCHAR2,
                     p_node_id          VARCHAR2,
                     p_link_id          VARCHAR2,
                     p_node_level       PLS_INTEGER,
                     x_is_loop          OUT BOOLEAN) IS

    l_net_rec				net_rec_type;

  BEGIN

-- IF p_link_id != c_root_link_id AND g_link_hash.EXISTS (p_link_id) THEN
    IF g_link_hash.EXISTS (p_link_id) THEN
      x_is_loop := TRUE;
      RETURN;
    ELSE
      g_link_hash (p_link_id) := 1;
    END IF;

    g_line_no                   := g_line_no + 1;
    l_net_rec.line_no           := g_line_no;
    l_net_rec.root_node_id      := p_root_node_id;
    l_net_rec.dirn              := p_dirn;
    l_net_rec.node_id           := p_node_id;
    l_net_rec.link_id           := p_link_id;
    l_net_rec.node_level        := p_node_level;

    x_is_loop := g_node_hash.EXISTS (p_node_id);
    IF x_is_loop THEN

      l_net_rec.loop_flag := c_loop_flag;

    ELSE

      g_node_hash (p_node_id) := 1;

    END IF;
    g_net_tab.EXTEND;
    g_net_tab (g_net_tab.COUNT)  := l_net_rec;

  END Add_Net;

  PROCEDURE Expand_Node (p_node_id VARCHAR2, p_link_id_prior VARCHAR2, p_node_level PLS_INTEGER) IS

    CURSOR lin_csr IS
    SELECT link_id, node_id_fr node_id, c_dirn_fr dirn
      FROM links_v
     WHERE node_id_to   = p_node_id
       AND node_id_to  != node_id_fr
       AND link_id     != p_link_id_prior
     UNION
    SELECT link_id, node_id_to, c_dirn_to
      FROM links_v
     WHERE node_id_fr   = p_node_id
       AND node_id_to  != node_id_fr
       AND link_id     != p_link_id_prior
     UNION
    SELECT link_id, node_id_to, c_dirn_sj
      FROM links_v
     WHERE node_id_fr   = p_node_id
       AND node_id_to   = node_id_fr
       AND link_id     != p_link_id_prior
     ORDER BY 2, 1;
    TYPE lin_tab_type IS TABLE OF lin_csr%ROWTYPE;
    l_lin_tab		lin_tab_type;

  BEGIN

    OPEN lin_csr;
    FETCH lin_csr BULK COLLECT -- avoids too many open cursors
     INTO l_lin_tab;
    CLOSE lin_csr;

    FOR i IN 1..l_lin_tab.COUNT LOOP

      Add_Net (g_root_node_id, l_lin_tab(i).dirn, l_lin_tab(i).node_id, l_lin_tab(i).link_id, p_node_level + 1, l_is_loop);
      IF NOT l_is_loop THEN

        Expand_Node (l_lin_tab(i).node_id, l_lin_tab(i).link_id, p_node_level + 1);

      END IF;

    END LOOP;

  END Expand_Node;

BEGIN

  g_line_no := 0;
  FOR r_nod IN (
          SELECT node_id_fr root_node_id
            FROM links_v
           UNION
          SELECT node_id_to
            FROM links_v) LOOP

    IF g_node_hash.EXISTS (r_nod.root_node_id) THEN CONTINUE; END IF;

    g_root_node_id := r_nod.root_node_id;
    Init_Net;
    Add_Net (g_root_node_id, ' ', g_root_node_id, c_root_link_id, 0, l_is_loop);

    Expand_Node (g_root_node_id, c_root_link_id, 0);

    FOR i IN 1..g_net_tab.COUNT LOOP

      PIPE ROW (g_net_tab(i));

    END LOOP;

  END LOOP;

END All_Nets;

END Net_Pipe;
/
SHO ERR
</pre>
</div>

### Array Structure Diagram

<img src="/migrated_images/2015/05/Networks-PLSQL-v1.0-Arrays.jpg" alt="Networks - PLSQL, v1.0 - Arrays" title="Networks - PLSQL, v1.0 - Arrays" />

### Neighbour SQL Structure Diagram

<img src="/migrated_images/2015/05/Networks-PLSQL-v1.0-Neighbour.jpg" alt="Networks - PLSQL, v1.0 - Neighbour" title="Networks - PLSQL, v1.0 - Neighbour" />

### Pseudocode - All\_Nets

- Loop over all nodes in the input links view
    - If the node has been visited continue to the next one
    - Initialise a new subnetwork with root node
    - Delete link hash array and output array
    - Call recursive procedure. Expand\_Node, to expand the whole subnetwork connected to the current root node
    - Loop over the output array
        - Pipe the record out
    - End loop
- End Loop

### Pseudocode - Expand\_Node

- Fetch all neighbours of the input node into an array, except the node from the prior link
- Loop over neighbours array
    - Call Add\_Net to possibly add new node to the output array
        - If link has been visited then
            - Return with link loop flag set
        - Else
            - Set link visited hash
        - End if
        - Set the fields of the new output record
        - If node has been visited then
            - Set the loop flag in the output record
        - Else
            - Set node visited hash
        - End if
        - Add new record to the output array
    - End call
    - If link was not a loop then
        - Call Expand\_Node recursively
    - End if
- End loop

### Notes on Code

- The arrays are a limitation on scalability
- The size of the arrays are determined by the largest subnetwork, except for the nodes hash array which will hold all nodes at the end
- Temporary tables could be used in place of the arrays if necessary

## SQL Calling Code

```sql
PROMPT Network detail
SELECT root_node_id             "Network",
       Count (DISTINCT link_id) OVER (PARTITION BY root_node_id) - 1 "#Links",
       Count (DISTINCT node_id) OVER (PARTITION BY root_node_id) "#Nodes",
       node_level "Lev",
       LPad (dirn || ' ', Least (2*node_level, 60), ' ') || node_id || loop_flag "Node",
       link_id                  "Link"
  FROM TABLE (Net_Pipe.All_Nets)
 ORDER BY line_no
/
PROMPT Network summary 1 - by network
SELECT root_node_id             "Network",
       Count (DISTINCT link_id) "#Links",
       Count (DISTINCT node_id) "#Nodes",
       Max (node_level) "Max Lev"
  FROM TABLE (Net_Pipe.All_Nets)
 GROUP BY root_node_id
 ORDER BY 2
/
PROMPT Network summary 2 - grouped by numbers of nodes
WITH network_counts AS (
SELECT root_node_id,
       Count (DISTINCT node_id) n_nodes
  FROM TABLE (Net_Pipe.All_Nets)
 GROUP BY root_node_id
)
SELECT n_nodes "#Nodes",
       COUNT(*) "#Networks"
  FROM network_counts
 GROUP BY n_nodes
 ORDER BY 1
/
```

## Test Network 1: Oracle v12.1 Foreign Key Network

This network comes from Oracles foreign key constraints as specified in it's ALL\_CONSTRAINTS view. I copied the constraints of type foreign key from the view into my own table, and created primary key and indexes that I deemed appropriate, and gathered statistics on the table.

### Output for Foreign Key Network

The detailed output with 815 records took 1.5 seconds, while the two summary outputs took a fraction of a second each.

#### Network detail

<div class="scrollbox">
<pre>
View links_v based on fk_link

View dropped.

View created.

Network detail

Network                                     #Links  #Nodes    Lev Node                                                                   Link
------------------------------------------ ------- ------- ------ ---------------------------------------------------------------------- ------------------------------------------
APEX$ARCHIVE_CONTENTS|APEX_040200                1       2      0 APEX$ARCHIVE_CONTENTS|APEX_040200                                      ROOT
                                                                1 > APEX$ARCHIVE_HEADER|APEX_040200                                      sys_c008573|apex_040200
APEX$_WS_FILES|APEX_040200                       4       5      0 APEX$_WS_FILES|APEX_040200                                             ROOT
                                                                1 > APEX$_WS_ROWS|APEX_040200                                            apex$_ws_files_fk|apex_040200
                                                                2   < APEX$_WS_LINKS|APEX_040200                                         apex$_ws_links_fk|apex_040200
                                                                2   < APEX$_WS_NOTES|APEX_040200                                         apex$_ws_notes_fk|apex_040200
                                                                2   < APEX$_WS_TAGS|APEX_040200                                          apex$_ws_tags_fk|apex_040200
AQ$_INTERNET_AGENTS|SYSTEM                       1       2      0 AQ$_INTERNET_AGENTS|SYSTEM                                             ROOT
                                                                1 < AQ$_INTERNET_AGENT_PRIVS|SYSTEM                                      agent_must_be_created|system ARCS|TEST                                        2       2      0 ARCS|TEST                                                              ROOT                                                                 1 > NODES|TEST                                                           arcs_fk1|test
                                                                2   < ARCS|TEST*                                                         arcs_fk2|test ATTRIBUTE_TRANSFORMATIONS$|SYS                   1       2      0 ATTRIBUTE_TRANSFORMATIONS$|SYS                                         ROOT                                                                 1 > TRANSFORMATIONS$|SYS                                                 attribute_transformations_fk|sys
BENCH_RUNS|BENCH                                 9       9      0 BENCH_RUNS|BENCH                                                       ROOT
                                                                1 < BENCH_RUN_DATA_POINTS|BENCH                                          rdp_rcn_fk|bench
                                                                2   < BENCH_RUN_STATISTICS|BENCH                                         brs_rdp_fk|bench
                                                                3     < BENCH_RUN_V$SQL_PLAN_STATS_ALL|BENCH                             rps_rst_fk|bench
                                                                3     < BENCH_RUN_V$STATS|BENCH                                          rvs_rst_fk|bench                                                                 3     > QUERIES|BENCH                                                    brs_qry_fk|bench
                                                                4       > QUERY_GROUPS|BENCH                                             qry_qgp_fk|bench
                                                                5         < BENCH_RUNS|BENCH*                                            brn_qgp_fk|bench                                                                 1 > LOG_HEADERS|BRENDAN                                                  brn_log_fk|bench
                                                                2   < LOG_LINES|BRENDAN                                                  lin_hdr_fk|brendan
BSLN_BASELINES|DBSNMP                            2       3      0 BSLN_BASELINES|DBSNMP                                                  ROOT
                                                                1 < BSLN_STATISTICS|DBSNMP                                               bsln_statistics_fk|dbsnmp
                                                                1 < BSLN_THRESHOLD_PARAMS|DBSNMP                                         bsln_thresholds_fk|dbsnmp
CHANNELS|SH                                     10       8      0 CHANNELS|SH                                                            ROOT
                                                                1 < COSTS|SH                                                             costs_channel_fk|sh                                                                 2   > PRODUCTS|SH                                                        costs_product_fk|sh
                                                                3     < SALES|SH                                                         sales_product_fk|sh                                                                 4       > CHANNELS|SH*                                                   sales_channel_fk|sh
                                                                4       > CUSTOMERS|SH                                                   sales_customer_fk|sh
                                                                5         > COUNTRIES|SH                                                 customers_country_fk|sh
                                                                4       > PROMOTIONS|SH                                                  sales_promo_fk|sh
                                                                5         < COSTS|SH*                                                    costs_promo_fk|sh                                                                 4       > TIMES|SH                                                       sales_time_fk|sh
                                                                5         < COSTS|SH*                                                    costs_time_fk|sh CLOUD|GSMADMIN_INTERNAL                          1       2      0 CLOUD|GSMADMIN_INTERNAL                                                ROOT                                                                 1 > GSM|GSMADMIN_INTERNAL                                                sys_c004716|gsmadmin_internal
CODE$|DVSYS                                     25      20      0 CODE$|DVSYS                                                            ROOT
                                                                1 < COMMAND_RULE$|DVSYS                                                  command_rule$_fk1|dvsys                                                                 2   > RULE_SET$|DVSYS                                                    command_rule$_fk|dvsys
                                                                3     < FACTOR$|DVSYS                                                    factor$_fk1|dvsys
                                                                4       < FACTOR_LINK$|DVSYS                                             factor_link$_fk1|dvsys                                                                 5         > FACTOR$|DVSYS*                                               factor_link$_fk|dvsys
                                                                5         < IDENTITY_MAP$|DVSYS                                          identity_map$_fk1|dvsys                                                                 6           > CODE$|DVSYS*                                               identity_map$_fk2|dvsys
                                                                6           > IDENTITY$|DVSYS                                            identity_map$_fk|dvsys
                                                                7             > FACTOR$|DVSYS*                                           identity$_fk|dvsys
                                                                7             < POLICY_LABEL$|DVSYS                                      identity_label$_fk|dvsys
                                                                4       < FACTOR_SCOPE$|DVSYS                                            factor_scope$_fk|dvsys                                                                 4       > FACTOR_TYPE$|DVSYS                                             factor$_fk|dvsys
                                                                4       < MAC_POLICY_FACTOR$|DVSYS                                       mac_policy_factor$_fk|dvsys                                                                 5         > MAC_POLICY$|DVSYS                                            mac_policy_factor$_fk1|dvsys
                                                                6           > CODE$|DVSYS*                                               mac_policy$_fk1|dvsys
                                                                3     < MONITOR_RULE$|DVSYS                                              monitor_rule$_fk1|dvsys
                                                                3     < REALM_AUTH$|DVSYS                                                realm_auth$_fk1|dvsys                                                                 4       > REALM$|DVSYS                                                   realm_auth$_fk|dvsys
                                                                5         < REALM_COMMAND_RULE$|DVSYS                                    realm_command_rule$_fk2|dvsys                                                                 6           > CODE$|DVSYS*                                               realm_command_rule$_fk1|dvsys
                                                                6           > RULE_SET$|DVSYS*                                           realm_command_rule$_fk|dvsys
                                                                5         < REALM_OBJECT$|DVSYS                                          realm_object$_fk|dvsys
                                                                3     < ROLE$|DVSYS                                                      role$_fk|dvsys
                                                                3     < RULE_SET_RULE$|DVSYS                                             rule_set_rule$_fk|dvsys                                                                 4       > RULE$|DVSYS                                                    rule_set_rule$_fk1|dvsys
COUNTRIES|HR                                    21      16      0 COUNTRIES|HR                                                           ROOT
                                                                1 < LOCATIONS|HR                                                         loc_c_id_fk|hr
                                                                2   < DEPARTMENTS|HR                                                     dept_loc_fk|hr                                                                 3     > EMPLOYEES|HR                                                     dept_mgr_fk|hr
                                                                4       < CUSTOMERS|OE                                                   customers_account_manager_fk|oe
                                                                5         < ORDERS|OE                                                    orders_customer_id_fk|oe                                                                 6           > EMPLOYEES|HR*                                              orders_sales_rep_fk|oe
                                                                6           < ORDER_ITEMS|OE                                             order_items_order_id_fk|oe                                                                 7             > PRODUCT_INFORMATION|OE                                   order_items_product_id_fk|oe
                                                                8               < INVENTORIES|OE                                         inventories_product_id_fk|oe                                                                 9                 > WAREHOUSES|OE                                        inventories_warehouses_fk|oe
                                                               10                   > LOCATIONS|HR*                                      warehouses_location_fk|oe
                                                                8               < ONLINE_MEDIA|PM                                        loc_c_id_fk|pm
                                                                8               < PRINT_MEDIA|PM                                         printmedia_fk|pm
                                                                8               < PRODUCT_DESCRIPTIONS|OE                                pd_product_id_fk|oe                                                                 4       > DEPARTMENTS|HR*                                                emp_dept_fk|hr
                                                                4       = EMPLOYEES|HR*                                                  emp_manager_fk|hr
                                                                4       > JOBS|HR                                                        emp_job_fk|hr
                                                                5         < JOB_HISTORY|HR                                               jhist_job_fk|hr                                                                 6           > DEPARTMENTS|HR*                                            jhist_dept_fk|hr
                                                                6           > EMPLOYEES|HR*                                              jhist_emp_fk|hr
                                                                1 > REGIONS|HR                                                           countr_reg_fk|hr
CSW_DOMAININFO$|MDSYS                            1       2      0 CSW_DOMAININFO$|MDSYS                                                  ROOT
                                                                1 > CSW_RECORD_TYPES$|MDSYS                                              sys_c006217|mdsys
DAM_CONFIG_PARAM$|SYS                            1       2      0 DAM_CONFIG_PARAM$|SYS                                                  ROOT
                                                                1 > DAM_PARAM_TAB$|SYS                                                   dam_config_param_fk1|sys
DATABASE_POOL_ADMIN|GSMADMIN_INTERNAL            6       5      0 DATABASE_POOL_ADMIN|GSMADMIN_INTERNAL                                  ROOT
                                                                1 > DATABASE_POOL|GSMADMIN_INTERNAL                                      sys_c004723|gsmadmin_internal
                                                                2   < DATABASE|GSMADMIN_INTERNAL                                         sys_c004731|gsmadmin_internal
                                                                3     < SERVICE_PREFERRED_AVAILABLE|GSMADMIN_INTERNAL                    sys_c004739|gsmadmin_internal                                                                 4       > DATABASE_POOL|GSMADMIN_INTERNAL*                               sys_c004738|gsmadmin_internal
                                                                4       > SERVICE|GSMADMIN_INTERNAL                                      sys_c004740|gsmadmin_internal
                                                                5         > DATABASE_POOL|GSMADMIN_INTERNAL*                             sys_c004736|gsmadmin_internal
DBFS$_MOUNTS|SYS                                 1       2      0 DBFS$_MOUNTS|SYS                                                       ROOT
                                                                1 > DBFS$_STORES|SYS                                                     sys_c003936|sys
DBFS_SFS$_FSTP|SYS                               1       2      0 DBFS_SFS$_FSTP|SYS                                                     ROOT
                                                                1 > DBFS_SFS$_FST|SYS                                                    sys_c004004|sys
DBFS_SFS$_FS|SYS                                 3       4      0 DBFS_SFS$_FS|SYS                                                       ROOT
                                                                1 > DBFS_SFS$_VOL|SYS                                                    sys_c003978|sys
                                                                2   < DBFS_SFS$_SNAP|SYS                                                 sys_c003970|sys                                                                 2   > DBFS_SFS$_TAB|SYS                                                  sys_c003961|sys
DBMSHP_FUNCTION_INFO|BENCH                       3       3      0 DBMSHP_FUNCTION_INFO|BENCH                                             ROOT
                                                                1 < DBMSHP_PARENT_CHILD_INFO|BENCH                                       sys_c0010218|bench                                                                 2   > DBMSHP_FUNCTION_INFO|BENCH*                                        sys_c0010219|bench
                                                                1 > DBMSHP_RUNS|BENCH                                                    sys_c0010217|bench
DBMS_PARALLEL_EXECUTE_CHUNKS$|SYS                1       2      0 DBMS_PARALLEL_EXECUTE_CHUNKS$|SYS                                      ROOT
                                                                1 > DBMS_PARALLEL_EXECUTE_TASK$|SYS                                      fk_dbms_parallel_execute_1|sys
DEF$_CALLDEST|SYSTEM                            14      14      0 DEF$_CALLDEST|SYSTEM                                                   ROOT
                                                                1 > DEF$_DESTINATION|SYSTEM                                              def$_call_destination|system
                                                                2   < REPCAT$_REPSCHEMA|SYSTEM                                           repcat$_repschema_dest|system                                                                 3     > REPCAT$_REPCAT|SYSTEM                                            repcat$_repschema_prnt|system
                                                                4       < REPCAT$_FLAVORS|SYSTEM                                         repcat$_flavors_fk1|system
                                                                4       < REPCAT$_FLAVOR_OBJECTS|SYSTEM                                  repcat$_flavor_objects_fk1|system
                                                                4       < REPCAT$_REPGROUP_PRIVS|SYSTEM                                  repcat$_repgroup_privs_fk|system
                                                                4       < REPCAT$_REPOBJECT|SYSTEM                                       repcat$_repobject_prnt|system
                                                                5         < REPCAT$_GENERATED|SYSTEM                                     repcat$_repgen_prnt2|system                                                                 6           > REPCAT$_REPOBJECT|SYSTEM*                                  repcat$_repgen_prnt|system
                                                                5         < REPCAT$_KEY_COLUMNS|SYSTEM                                   repcat$_key_columns_prnt|system
                                                                5         < REPCAT$_REPCOLUMN|SYSTEM                                     repcat$_repcolumn_fk|system
                                                                5         < REPCAT$_REPPROP|SYSTEM                                       repcat$_repprop_prnt|system
                                                                4       < REPCAT$_SITES_NEW|SYSTEM                                       repcat$_sites_new_fk2|system                                                                 5         > REPCAT$_EXTENSION|SYSTEM                                     repcat$_sites_new_fk1|system
DEPT|SCOTT                                       1       2      0 DEPT|SCOTT                                                             ROOT
                                                                1 < EMP|SCOTT                                                            fk_deptno|scott DR$THS_BT|CTXSYS                                 4       4      0 DR$THS_BT|CTXSYS                                                       ROOT                                                                 1 > DR$THS_PHRASE|CTXSYS                                                 sys_c005023|ctxsys
                                                                2   < DR$THS_BT|CTXSYS*                                                  sys_c005024|ctxsys
                                                                2   < DR$THS_FPHRASE|CTXSYS                                              sys_c005021|ctxsys                                                                 2   > DR$THS|CTXSYS                                                      sys_c005017|ctxsys
FLIGHTS|TEST                                     1       2      0 FLIGHTS|TEST                                                           ROOT
                                                                1 > SECTORS|TEST                                                         fli_sec_fk|test
HS$_BASE_CAPS|SYS                               11      10      0 HS$_BASE_CAPS|SYS                                                      ROOT
                                                                1 < HS$_CLASS_CAPS|SYS                                                   hs$_class_caps_fk2|sys                                                                 2   > HS$_FDS_CLASS|SYS                                                  hs$_class_caps_fk1|sys
                                                                3     < HS$_CLASS_DD|SYS                                                 hs$_class_dd_fk1|sys                                                                 4       > HS$_BASE_DD|SYS                                                hs$_class_dd_fk2|sys
                                                                5         < HS$_INST_DD|SYS                                              hs$_inst_dd_fk2|sys                                                                 6           > HS$_FDS_INST|SYS                                           hs$_inst_dd_fk1|sys
                                                                7             > HS$_FDS_CLASS|SYS*                                       hs$_fds_inst_fk1|sys
                                                                7             < HS$_INST_CAPS|SYS                                        hs$_inst_caps_fk1|sys                                                                 8               > HS$_BASE_CAPS|SYS*                                     hs$_inst_caps_fk2|sys
                                                                7             < HS$_INST_INIT|SYS                                        hs$_inst_init_fk1|sys
                                                                3     < HS$_CLASS_INIT|SYS                                               hs$_class_init_fk1|sys HS$_PARALLEL_HISTOGRAM_DATA|SYS                  3       4      0 HS$_PARALLEL_HISTOGRAM_DATA|SYS                                        ROOT                                                                 1 > HS$_PARALLEL_METADATA|SYS                                            hs_parallel_histogram_data_fk|sys
                                                                2   < HS$_PARALLEL_PARTITION_DATA|SYS                                    hs_parallel_partition_data_fk|sys
                                                                2   < HS$_PARALLEL_SAMPLE_DATA|SYS                                       hs_parallel_sample_data_fk|sys
MVIEW$_ADV_AJG|SYSTEM                           14      13      0 MVIEW$_ADV_AJG|SYSTEM                                                  ROOT
                                                                1 < MVIEW$_ADV_FJG|SYSTEM                                                mview$_adv_fjg_fk|system
                                                                2   < MVIEW$_ADV_GC|SYSTEM                                               mview$_adv_gc_fk|system                                                                 1 > MVIEW$_ADV_LOG|SYSTEM                                                mview$_adv_ajg_fk|system
                                                                2   < MVIEW$_ADV_CLIQUE|SYSTEM                                           mview$_adv_clique_fk|system
                                                                2   < MVIEW$_ADV_ELIGIBLE|SYSTEM                                         mview$_adv_eligible_fk|system
                                                                2   < MVIEW$_ADV_EXCEPTIONS|SYSTEM                                       mview$_adv_exception_fk|system
                                                                2   < MVIEW$_ADV_FILTERINSTANCE|SYSTEM                                   mview$_adv_filterinstance_fk|system
                                                                2   < MVIEW$_ADV_INFO|SYSTEM                                             mview$_adv_info_fk|system
                                                                2   < MVIEW$_ADV_JOURNAL|SYSTEM                                          mview$_adv_journal_fk|system
                                                                2   < MVIEW$_ADV_LEVEL|SYSTEM                                            mview$_adv_level_fk|system
                                                                3     < MVIEW$_ADV_ROLLUP|SYSTEM                                         mview$_adv_rollup_cfk|system                                                                 4       > MVIEW$_ADV_LEVEL|SYSTEM*                                       mview$_adv_rollup_pfk|system
                                                                4       > MVIEW$_ADV_LOG|SYSTEM*                                         mview$_adv_rollup_fk|system
                                                                2   < MVIEW$_ADV_OUTPUT|SYSTEM                                           mview$_adv_output_fk|system MVIEW$_ADV_BASETABLE|SYSTEM                      1       2      0 MVIEW$_ADV_BASETABLE|SYSTEM                                            ROOT                                                                 1 > MVIEW$_ADV_WORKLOAD|SYSTEM                                           mview$_adv_basetable_fk|system
OGIS_GEOMETRY_COLUMNS|MDSYS                      1       2      0 OGIS_GEOMETRY_COLUMNS|MDSYS                                            ROOT
                                                                1 > OGIS_SPATIAL_REFERENCE_SYSTEMS|MDSYS                                 fk_srid|mdsys
OLS$AUDIT|LBACSYS                               21      14      0 OLS$AUDIT|LBACSYS                                                      ROOT
                                                                1 > OLS$POL|LBACSYS                                                      sys_c006346|lbacsys
                                                                2   < OLS$COMPARTMENTS|LBACSYS                                           ols_comp_pol_fk|lbacsys
                                                                3     < OLS$USER_COMPARTMENTS|LBACSYS                                    ols_user_comp_fk|lbacsys                                                                 4       > OLS$USER_LEVELS|LBACSYS                                        ols_user_comp_level_fk|lbacsys
                                                                5         > OLS$LEVELS|LBACSYS                                           ols_user_def_fk|lbacsys
                                                                6           > OLS$POL|LBACSYS*                                           ols_level_pol_fk|lbacsys
                                                                6           < OLS$USER_LEVELS|LBACSYS*                                   ols_user_max_fk|lbacsys
                                                                6           < OLS$USER_LEVELS|LBACSYS*                                   ols_user_min_fk|lbacsys
                                                                6           < OLS$USER_LEVELS|LBACSYS*                                   ols_user_row_fk|lbacsys                                                                 5         > OLS$POL|LBACSYS*                                             ols_user_level_pol_fk|lbacsys
                                                                5         < OLS$USER_GROUPS|LBACSYS                                      ols_user_grp_level_fk|lbacsys                                                                 6           > OLS$GROUPS|LBACSYS                                         ols_user_grp_fk|lbacsys
                                                                7             = OLS$GROUPS|LBACSYS*                                      ols_group_parent|lbacsys
                                                                7             > OLS$POL|LBACSYS*                                         ols_group_pol_fk|lbacsys
                                                                2   < OLS$LAB|LBACSYS                                                    ols_label_policy_fk|lbacsys
                                                                2   < OLS$POLS|LBACSYS                                                   sys_c006243|lbacsys
                                                                2   < OLS$POLT|LBACSYS                                                   sys_c006248|lbacsys
                                                                2   < OLS$PROFILE|LBACSYS                                                sys_c006251|lbacsys
                                                                3     < OLS$USER|LBACSYS                                                 sys_c006257|lbacsys                                                                 4       > OLS$POL|LBACSYS*                                               sys_c006256|lbacsys
                                                                2   < OLS$PROG|LBACSYS                                                   sys_c006262|lbacsys OLS_DIR_BUSINESSES|MDSYS                         1       2      0 OLS_DIR_BUSINESSES|MDSYS                                               ROOT                                                                 1 > OLS_DIR_BUSINESS_CHAINS|MDSYS                                        olsfk3|mdsys
OLS_DIR_CATEGORIES|MDSYS                         3       3      0 OLS_DIR_CATEGORIES|MDSYS                                               ROOT
                                                                1 = OLS_DIR_CATEGORIES|MDSYS*                                            olsfk1|mdsys
                                                                1 < OLS_DIR_CATEGORIZATIONS|MDSYS                                        olsfk5|mdsys                                                                 1 > OLS_DIR_CATEGORY_TYPES|MDSYS                                         olsfk2|mdsys
ORDDCM_ANON_ACTION_TYPES|ORDDATA                69      47      0 ORDDCM_ANON_ACTION_TYPES|ORDDATA                                       ROOT
                                                                1 < ORDDCM_ANON_ATTRS_WRK|ORDDATA                                        orddcm_anon_attrs_w_fk3|orddata                                                                 2   > ORDDCM_DOCS_WRK|ORDDATA                                            orddcm_anon_attrs_w_fk1|orddata
                                                                3     < ORDDCM_ANON_RULES_WRK|ORDDATA                                    orddcm_anon_rules_w_fk1|orddata                                                                 4       > ORDDCM_ANON_ACTION_TYPES|ORDDATA*                              orddcm_anon_rules_w_fk3|orddata
                                                                4       > ORDDCM_ANON_RULE_TYPES|ORDDATA                                 orddcm_anon_rules_w_fk2|orddata
                                                                5         < ORDDCM_ANON_RULES|ORDDATA                                    orddcm_anon_rules_fk2|orddata                                                                 6           > ORDDCM_ANON_ACTION_TYPES|ORDDATA*                          orddcm_anon_rules_fk3|orddata
                                                                6           > ORDDCM_DOCS|ORDDATA                                        orddcm_anon_rules_fk1|orddata
                                                                7             < ORDDCM_ANON_ATTRS|ORDDATA                                orddcm_anon_attrs_fk1|orddata                                                                 8               > ORDDCM_ANON_ACTION_TYPES|ORDDATA*                      orddcm_anon_attrs_fk3|orddata
                                                                7             < ORDDCM_CT_DAREFS|ORDDATA                                 orddcm_ct_darefs_fk2|orddata                                                                 8               > ORDDCM_DICT_ATTRS|ORDDATA                              orddcm_ct_darefs_fk1|orddata
                                                                9                 > ORDDCM_PRV_ATTRS|ORDDATA                             orddcm_dict_attrs_fk2|orddata
                                                               10                   > ORDDCM_DOCS|ORDDATA*                               orddcm_prv_attrs_fk1|orddata
                                                               10                   > ORDDCM_VR_DT_MAP|ORDDATA                           orddcm_prv_attrs_fk2|orddata
                                                               11                     < ORDDCM_PRV_ATTRS_WRK|ORDDATA                     orddcm_prv_attrs_w_fk2|orddata
                                                               12                       < ORDDCM_DICT_ATTRS_WRK|ORDDATA                  orddcm_dict_attrs_w_fk2|orddata
                                                               13                         < ORDDCM_CT_DAREFS_WRK|ORDDATA                 orddcm_ct_darefs_w_fk1|orddata                                                                14                           > ORDDCM_DOCS_WRK|ORDDATA*                   orddcm_ct_darefs_w_fk2|orddata
                                                               13                         > ORDDCM_STD_ATTRS_WRK|ORDDATA                 orddcm_dict_attrs_w_fk1|orddata
                                                               14                           > ORDDCM_DOCS_WRK|ORDDATA*                   orddcm_sd_attrs_w_fk2|orddata
                                                               14                           > ORDDCM_VR_DT_MAP|ORDDATA*                  orddcm_sd_attrs_w_fk1|orddata
                                                               12                       > ORDDCM_DOCS_WRK|ORDDATA*                       orddcm_prv_attrs_w_fk1|orddata
                                                               11                     < ORDDCM_STD_ATTRS|ORDDATA                         orddcm_sd_attrs_fk1|orddata
                                                               12                       < ORDDCM_DICT_ATTRS|ORDDATA*                     orddcm_dict_attrs_fk1|orddata                                                                12                       > ORDDCM_DOCS|ORDDATA*                           orddcm_sd_attrs_fk2|orddata
                                                                7             < ORDDCM_CT_LOCATORPATHS|ORDDATA                           orddcm_ct_lp_fk1|orddata                                                                 8               > ORDDCM_CT_PRED_SET|ORDDATA                             orddcm_ct_lp_fk2|orddata
                                                                9                 < ORDDCM_CT_MACRO_DEP|ORDDATA                          orddcm_ct_md_fk1|orddata                                                                10                   > ORDDCM_CT_PRED_SET|ORDDATA*                        orddcm_ct_md_fk2|orddata
                                                                9                 < ORDDCM_CT_MACRO_PAR|ORDDATA                          orddcm_ct_mp_fk|orddata                                                                 9                 = ORDDCM_CT_PRED_SET|ORDDATA*                          orddcm_ct_ps_fk2|orddata                                                                 9                 > ORDDCM_CT_PRED|ORDDATA                               orddcm_ct_ps_fk1|orddata
                                                               10                   < ORDDCM_CT_ACTION|ORDDATA                           orddcm_ct_a_fk1|orddata
                                                               10                   < ORDDCM_CT_PRED_OPRD|ORDDATA                        orddcm_ct_po_fk|orddata
                                                               10                   < ORDDCM_CT_PRED_PAR|ORDDATA                         orddcm_ct_pp_fk|orddata                                                                10                   = ORDDCM_CT_PRED|ORDDATA*                            orddcm_ct_pred_fk1|orddata                                                                10                   = ORDDCM_CT_PRED|ORDDATA*                            orddcm_ct_pred_fk2|orddata                                                                 9                 > ORDDCM_DOCS|ORDDATA*                                 orddcm_ct_ps_fk3|orddata
                                                                7             < ORDDCM_DOC_REFS|ORDDATA                                  sys_c005130|orddata                                                                 8               > ORDDCM_DOCS|ORDDATA*                                   sys_c005131|orddata
                                                                7             > ORDDCM_DOC_TYPES|ORDDATA                                 orddcm_docs_fk1|orddata
                                                                8               < ORDDCM_DOCS_WRK|ORDDATA*                               orddcm_docs_w_fk1|orddata
                                                                8               < ORDDCM_INSTALL_DOCS|ORDDATA                            orddcm_i_docs_fk1|orddata
                                                                7             < ORDDCM_MAPPING_DOCS|ORDDATA                              orddcm_mapping_docs_fk1|orddata
                                                                8               < ORDDCM_MAPPED_PATHS|ORDDATA                            orddcm_mapped_paths_fk2|orddata
                                                                7             < ORDDCM_RT_PREF_PARAMS|ORDDATA                            orddcm_pref_params_fk1|orddata
                                                                7             < ORDDCM_STORED_TAGS|ORDDATA                               orddcm_stored_tags_fk1|orddata
                                                                7             < ORDDCM_UID_DEFS|ORDDATA                                  orddcm_uid_defs_fk1|orddata
                                                                3     < ORDDCM_CT_LOCATORPATHS_WRK|ORDDATA                               orddcm_ct_lp_w_fk1|orddata                                                                 4       > ORDDCM_CT_PRED_SET_WRK|ORDDATA                                 orddcm_ct_lp_w_fk2|orddata
                                                                5         < ORDDCM_CT_MACRO_DEP_WRK|ORDDATA                              orddcm_ct_md_w_fk1|orddata                                                                 6           > ORDDCM_CT_PRED_SET_WRK|ORDDATA*                            orddcm_ct_md_w_fk2|orddata
                                                                5         < ORDDCM_CT_MACRO_PAR_WRK|ORDDATA                              orddcm_ct_mp_w_fk|orddata                                                                 5         = ORDDCM_CT_PRED_SET_WRK|ORDDATA*                              orddcm_ct_ps_w_fk2|orddata                                                                 5         > ORDDCM_CT_PRED_WRK|ORDDATA                                   orddcm_ct_ps_w_fk1|orddata
                                                                6           < ORDDCM_CT_ACTION_WRK|ORDDATA                               orddcm_ct_a_w_fk1|orddata
                                                                6           < ORDDCM_CT_PRED_OPRD_WRK|ORDDATA                            orddcm_ct_po_w_fk|orddata
                                                                6           < ORDDCM_CT_PRED_PAR_WRK|ORDDATA                             orddcm_ct_pp_w_fk|orddata                                                                 6           = ORDDCM_CT_PRED_WRK|ORDDATA*                                orddcm_ct_pred_w_fk1|orddata                                                                 6           = ORDDCM_CT_PRED_WRK|ORDDATA*                                orddcm_ct_pred_w_fk2|orddata                                                                 5         > ORDDCM_DOCS_WRK|ORDDATA*                                     orddcm_ct_ps_w_fk3|orddata
                                                                3     < ORDDCM_DOC_REFS_WRK|ORDDATA                                      sys_c005428|orddata                                                                 4       > ORDDCM_DOCS_WRK|ORDDATA*                                       sys_c005429|orddata
                                                                3     < ORDDCM_MAPPING_DOCS_WRK|ORDDATA                                  orddcm_mapping_docs_w_fk1|orddata
                                                                4       < ORDDCM_MAPPED_PATHS_WRK|ORDDATA                                orddcm_mapped_paths_w_fk2|orddata
                                                                3     < ORDDCM_RT_PREF_PARAMS_WRK|ORDDATA                                orddcm_pref_params_w_fk1|orddata
                                                                3     < ORDDCM_STORED_TAGS_WRK|ORDDATA                                   orddcm_stored_tags_w_fk1|orddata
                                                                3     < ORDDCM_UID_DEFS_WRK|ORDDATA                                      orddcm_uid_defs_w_fk1|orddata
PLANETS|TEST                                     1       2      0 PLANETS|TEST                                                           ROOT
                                                                1 < PLANET_CLIMATES|TEST                                                 plc_pla_fk|test PLSQL_PROFILER_DATA|BENCH                        2       3      0 PLSQL_PROFILER_DATA|BENCH                                              ROOT                                                                 1 > PLSQL_PROFILER_UNITS|BENCH                                           sys_c0010214|bench
                                                                2   > PLSQL_PROFILER_RUNS|BENCH                                          sys_c0010211|bench
REGISTRY$DEPENDENCIES|SYS                        5       4      0 REGISTRY$DEPENDENCIES|SYS                                              ROOT
                                                                1 > REGISTRY$|SYS                                                        dependencies_fk|sys
                                                                2   < REGISTRY$DEPENDENCIES|SYS*                                         dependencies_req_fk|sys
                                                                2   < REGISTRY$PROGRESS|SYS                                              registry_progress_fk|sys
                                                                2   < REGISTRY$SCHEMAS|SYS                                               registry_schema_fk|sys
                                                                2   = REGISTRY$|SYS*                                                     registry_parent_fk|sys
REPCAT$_AUDIT_ATTRIBUTE|SYSTEM                   5       6      0 REPCAT$_AUDIT_ATTRIBUTE|SYSTEM                                         ROOT
                                                                1 < REPCAT$_AUDIT_COLUMN|SYSTEM                                          repcat$_audit_column_f1|system                                                                 2   > REPCAT$_CONFLICT|SYSTEM                                            repcat$_audit_column_f2|system
                                                                3     < REPCAT$_RESOLUTION|SYSTEM                                        repcat$_resolution_f3|system
                                                                4       < REPCAT$_PARAMETER_COLUMN|SYSTEM                                repcat$_parameter_column_f1|system                                                                 4       > REPCAT$_RESOLUTION_METHOD|SYSTEM                               repcat$_resolution_f1|system
REPCAT$_COLUMN_GROUP|SYSTEM                      1       2      0 REPCAT$_COLUMN_GROUP|SYSTEM                                            ROOT
                                                                1 < REPCAT$_GROUPED_COLUMN|SYSTEM                                        repcat$_grouped_column_f1|system REPCAT$_DDL|SYSTEM                               1       2      0 REPCAT$_DDL|SYSTEM                                                     ROOT                                                                 1 > REPCAT$_REPCATLOG|SYSTEM                                             repcat$_ddl_prnt|system
REPCAT$_INSTANTIATION_DDL|SYSTEM                13      13      0 REPCAT$_INSTANTIATION_DDL|SYSTEM                                       ROOT
                                                                1 > REPCAT$_REFRESH_TEMPLATES|SYSTEM                                     repcat$_instantiation_ddl_fk1|system
                                                                2   < REPCAT$_TEMPLATE_OBJECTS|SYSTEM                                    repcat$_template_objects_fk1|system
                                                                3     < REPCAT$_OBJECT_PARMS|SYSTEM                                      repcat$_object_parms_fk2|system                                                                 4       > REPCAT$_TEMPLATE_PARMS|SYSTEM                                  repcat$_object_parms_fk1|system
                                                                5         > REPCAT$_REFRESH_TEMPLATES|SYSTEM*                            repcat$_template_parms_fk1|system
                                                                5         < REPCAT$_USER_PARM_VALUES|SYSTEM                              repcat$_user_parm_values_fk1|system                                                                 3     > REPCAT$_OBJECT_TYPES|SYSTEM                                      repcat$_template_objects_fk3|system
                                                                4       < REPCAT$_SITE_OBJECTS|SYSTEM                                    repcat$_site_objects_fk1|system                                                                 5         > REPCAT$_TEMPLATE_SITES|SYSTEM                                repcat$_site_object_fk2|system
                                                                2   < REPCAT$_TEMPLATE_REFGROUPS|SYSTEM                                  repcat$_template_refgroups_fk1|system                                                                 2   > REPCAT$_TEMPLATE_STATUS|SYSTEM                                     repcat$_refresh_templates_fk2|system
                                                                2   > REPCAT$_TEMPLATE_TYPES|SYSTEM                                      repcat$_refresh_templates_fk1|system
                                                                2   < REPCAT$_USER_AUTHORIZATIONS|SYSTEM                                 repcat$_user_authorization_fk2|system
REPCAT$_PRIORITY_GROUP|SYSTEM                    1       2      0 REPCAT$_PRIORITY_GROUP|SYSTEM                                          ROOT
                                                                1 < REPCAT$_PRIORITY|SYSTEM                                              repcat$_priority_f1|system
ROADS|TEST                                       1       2      0 ROADS|TEST                                                             ROOT
                                                                1 < ROAD_EVENTS|TEST                                                     rev_roa_fk|test SCHEDULER$_JOB_OUTPUT|SYS                        1       2      0 SCHEDULER$_JOB_OUTPUT|SYS                                              ROOT                                                                 1 > SCHEDULER$_JOB_RUN_DETAILS|SYS                                       scheduler$_job_output_fk|sys
SDO_COORD_AXES|MDSYS                            30      14      0 SDO_COORD_AXES|MDSYS                                                   ROOT
                                                                1 > SDO_COORD_AXIS_NAMES|MDSYS                                           coord_axis_foreign_axis|mdsys
                                                                1 > SDO_COORD_SYS|MDSYS                                                  coord_axis_foreign_cs|mdsys
                                                                2   < SDO_COORD_REF_SYS|MDSYS                                            coord_ref_sys_foreign_cs|mdsys
                                                                3     < SDO_COORD_OPS|MDSYS                                              coord_operation_foreign_source|mdsys                                                                 4       = SDO_COORD_OPS|MDSYS*                                           coord_operation_foreign_legacy|mdsys                                                                 4       > SDO_COORD_OP_METHODS|MDSYS                                     coord_operation_foreign_method|mdsys
                                                                5         < SDO_COORD_OP_PARAM_USE|MDSYS                                 coord_op_para_use_foreign_meth|mdsys                                                                 6           > SDO_COORD_OP_PARAMS|MDSYS                                  coord_op_para_use_foreign_para|mdsys
                                                                7             < SDO_COORD_OP_PARAM_VALS|MDSYS                            coord_op_para_val_foreign_para|mdsys                                                                 8               > SDO_COORD_OPS|MDSYS*                                   coord_op_para_val_foreign_op|mdsys
                                                                8               > SDO_COORD_OP_METHODS|MDSYS*                            coord_op_para_val_foreign_meth|mdsys
                                                                8               > SDO_UNITS_OF_MEASURE|MDSYS                             coord_op_para_val_foreign_uom|mdsys
                                                                9                 < SDO_COORD_AXES|MDSYS*                                coord_axis_foreign_uom|mdsys                                                                 9                 > SDO_ELLIPSOIDS|MDSYS                                 ellipsoid_foreign_legacy|mdsys
                                                               10                   < SDO_DATUMS|MDSYS                                   datum_foreign_ellipsoid|mdsys
                                                               11                     < SDO_COORD_REF_SYS|MDSYS*                         coord_ref_sys_foreign_datum|mdsys                                                                11                     = SDO_DATUMS|MDSYS*                                datum_foreign_legacy|mdsys                                                                11                     > SDO_PRIME_MERIDIANS|MDSYS                        datum_foreign_meridian|mdsys
                                                               12                       > SDO_UNITS_OF_MEASURE|MDSYS*                    prime_meridian_foreign_uom|mdsys
                                                               10                   > SDO_UNITS_OF_MEASURE|MDSYS*                        ellipsoid_foreign_uom|mdsys
                                                                9                 = SDO_UNITS_OF_MEASURE|MDSYS*                          unit_of_measure_foreign_legacy|mdsys
                                                                9                 = SDO_UNITS_OF_MEASURE|MDSYS*                          unit_of_measure_foreign_uom|mdsys
                                                                4       > SDO_COORD_REF_SYS|MDSYS*                                       coord_operation_foreign_target|mdsys
                                                                4       < SDO_COORD_REF_SYS|MDSYS*                                       coord_ref_sys_foreign_proj|mdsys
                                                                3     < SDO_COORD_OP_PATHS|MDSYS                                         coord_op_path_foreign_source|mdsys                                                                 4       > SDO_COORD_REF_SYS|MDSYS*                                       coord_op_path_foreign_target|mdsys
                                                                3     = SDO_COORD_REF_SYS|MDSYS*                                         coord_ref_sys_foreign_geog|mdsys
                                                                3     = SDO_COORD_REF_SYS|MDSYS*                                         coord_ref_sys_foreign_horiz|mdsys
                                                                3     = SDO_COORD_REF_SYS|MDSYS*                                         coord_ref_sys_foreign_legacy|mdsys
                                                                3     = SDO_COORD_REF_SYS|MDSYS*                                         coord_ref_sys_foreign_vert|mdsys
SDO_WS_CONFERENCE_PARTICIPANTS|MDSYS             1       2      0 SDO_WS_CONFERENCE_PARTICIPANTS|MDSYS                                   ROOT
                                                                1 > SDO_WS_CONFERENCE|MDSYS                                              sdo_ws_conf_part_fk|mdsys
TSDP_ASSOCIATION$|SYS                            8       9      0 TSDP_ASSOCIATION$|SYS                                                  ROOT
                                                                1 > TSDP_POLICY$|SYS                                                     tsdp_association$fkpo|sys
                                                                2   < TSDP_SUBPOL$|SYS                                                   tsdp_subpol$fk|sys
                                                                3     < TSDP_CONDITION$|SYS                                              tsdp_condition$fk|sys
                                                                3     < TSDP_PARAMETER$|SYS                                              tsdp_parameter$fk|sys
                                                                3     < TSDP_PROTECTION$|SYS                                             tsdp_protection$fkpc|sys                                                                 4       > TSDP_SENSITIVE_DATA$|SYS                                       tsdp_protection$fksd|sys
                                                                1 > TSDP_SENSITIVE_TYPE$|SYS                                             tsdp_association$fkst|sys
                                                                2   > TSDP_SOURCE$|SYS                                                   tsdp_sensitive_type$fk|sys
WFS_FEATUREINSTANCEMETADATA$|MDSYS               6       7      0 WFS_FEATUREINSTANCEMETADATA$|MDSYS                                     ROOT
                                                                1 > WFS_FEATURETYPE$|MDSYS                                               sys_c006201|mdsys
                                                                2   < WFS_FEATURETYPEATTRS$|MDSYS                                        sys_c006202|mdsys
                                                                2   < WFS_FEATURETYPENESTEDSDOS$|MDSYS                                   sys_c006206|mdsys
                                                                2   < WFS_FEATURETYPESIMPLETAGATTRS$|MDSYS                               sys_c006203|mdsys
                                                                2   < WFS_FEATURETYPETAGS$|MDSYS                                         sys_c006200|mdsys
                                                                2   < WFS_FEATURETYPEXMLCOLINFO$|MDSYS                                   sys_c006205|mdsys
WI$_CAPTURE_FILE|SYS                            10       9      0 WI$_CAPTURE_FILE|SYS                                                   ROOT
                                                                1 < WI$_EXECUTION_ORDER|SYS                                              wi$_execution_order_fk1|sys                                                                 2   > WI$_TEMPLATE|SYS                                                   wi$_execution_order_fk2|sys
                                                                3     < WI$_FREQUENT_PATTERN_ITEM|SYS                                    wi$_frequent_pattern_item_fk2|sys                                                                 4       > WI$_FREQUENT_PATTERN|SYS                                       wi$_frequent_pattern_item_fk1|sys
                                                                5         > WI$_JOB|SYS                                                  wi$_frequent_pattern_fk1|sys
                                                                6           < WI$_CAPTURE_FILE|SYS*                                      wi$_capture_file_fk1|sys
                                                                6           < WI$_FREQUENT_PATTERN_METADATA|SYS                          wi$_freq_pattern_metadata_fk1|sys
                                                                6           < WI$_TEMPLATE|SYS*                                          wi$_template_fk1|sys
                                                                3     < WI$_OBJECT|SYS                                                   wi$_object_fk1|sys
                                                                3     < WI$_STATEMENT|SYS                                                wi$_statement_fk1|sys
WRM$_DATABASE_INSTANCE|SYS                       1       2      0 WRM$_DATABASE_INSTANCE|SYS                                             ROOT
                                                                1 < WRM$_SNAPSHOT|SYS                                                    wrm$_snapshot_fk|sys
WWV_FLOWS|APEX_040200                          334     264      0 WWV_FLOWS|APEX_040200                                                  ROOT
                                                                1 < WWV_FLOW_APP_COMMENTS|APEX_040200                                    wwv_flow_app_comments_fk|apex_040200
                                                                1 < WWV_FLOW_AUTHENTICATIONS|APEX_040200                                 wwv_flow_authentications_fk|apex_040200
                                                                2   < WWV_FLOWS|APEX_040200*                                             wwv_flows_fk_authentication|apex_040200
                                                                1 < WWV_FLOW_BANNER|APEX_040200                                          wwv_flow_banner_fk|apex_040200
                                                                1 < WWV_FLOW_BUTTON_TEMPLATES|APEX_040200                                wwv_flow_buttont_fk|apex_040200
                                                                2   < WWV_FLOW_PAGE_PLUG_TEMPLATES|APEX_040200                           wwv_flow_plug_temp_button_fk|apex_040200                                                                 3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_plug_temp_fk|apex_040200
                                                                3     > WWV_FLOW_FIELD_TEMPLATES|APEX_040200                             wwv_flow_plug_temp_field_fk|apex_040200
                                                                4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_field_temp_f_fk|apex_040200
                                                                4       < WWV_FLOW_PAGE_PLUG_TEMPLATES|APEX_040200*                      wwv_flow_plug_temp_req_fld_fk|apex_040200
                                                                3     < WWV_FLOW_PLUG_TMPL_DISP_POINTS|APEX_040200                       wwv_plug_tmpl_dp_parent_fk|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_plug_tmpl_dp_fk|apex_040200
                                                                1 < WWV_FLOW_CALS|APEX_040200                                            wwv_flow_cal_to_flow_fk|apex_040200                                                                 2   > WWV_FLOW_PAGE_PLUGS|APEX_040200                                    wwv_flow_plug_calendar_fk|apex_040200
                                                                3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_plug_to_flow_fk|apex_040200
                                                                3     < WWV_FLOW_FLASH_CHARTS_5|APEX_040200                              wwv_flow_flash_charts_5_fk2|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_flash_charts_5_fk|apex_040200
                                                                4       < WWV_FLOW_FLASH_CHART5_SERIES|APEX_040200                       wwv_flow_flash_5_series_fk|apex_040200
                                                                4       < WWV_FLOW_FLASH_CHARTS_5_DASH|APEX_040200                       wwv_flow_flash_charts5_dash_fk|apex_040200
                                                                3     < WWV_FLOW_FLASH_CHARTS|APEX_040200                                wwv_flow_flash_charts_fk2|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_flash_charts_fk|apex_040200
                                                                4       < WWV_FLOW_FLASH_CHART_SERIES|APEX_040200                        wwv_flow_flash_chart_series_fk|apex_040200
                                                                3     < WWV_FLOW_PAGE_DA_ACTIONS|APEX_040200                             wwv_flow_page_da_a_ar_fk|apex_040200                                                                 4       > WWV_FLOW_PAGE_DA_EVENTS|APEX_040200                            wwv_flow_page_da_a_evnt_fk|apex_040200
                                                                5         > WWV_FLOWS|APEX_040200*                                       wwv_flow_page_da_e_flow_fk|apex_040200
                                                                5         > WWV_FLOW_PAGE_PLUGS|APEX_040200*                             wwv_flow_page_da_e_tr_fk|apex_040200
                                                                5         > WWV_FLOW_STEPS|APEX_040200                                   wwv_flow_page_da_e_page_fk|apex_040200
                                                                6           > WWV_FLOWS|APEX_040200*                                     wwv_flow_step_flow_fk|apex_040200
                                                                6           < WWV_FLOW_PAGE_DA_ACTIONS|APEX_040200*                      wwv_flow_page_da_a_page_fk|apex_040200
                                                                6           < WWV_FLOW_PAGE_PLUGS|APEX_040200*                           wwv_flow_plug_to_page_fk|apex_040200
                                                                6           < WWV_FLOW_STEP_BRANCHES|APEX_040200                         wwv_flow_step_branches_fk2|apex_040200                                                                 7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_step_branches_fk|apex_040200
                                                                7             < WWV_FLOW_STEP_BRANCH_ARGS|APEX_040200                    wwv_flow_step_branch_args_fk|apex_040200
                                                                6           < WWV_FLOW_STEP_BUTTONS|APEX_040200                          wwv_flow_step_buttons_fk2|apex_040200                                                                 7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_step_buttons_fk1|apex_040200
                                                                7             > WWV_FLOW_PAGE_PLUGS|APEX_040200*                         wwv_flow_step_buttons_plug_fk|apex_040200
                                                                6           < WWV_FLOW_STEP_COMPUTATIONS|APEX_040200                     wwv_flow_step_comp_fk2|apex_040200                                                                 7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_step_comp_fk|apex_040200
                                                                6           < WWV_FLOW_STEP_ITEMS|APEX_040200                            wwv_flow_step_items_fk2|apex_040200                                                                 7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_step_items_fk|apex_040200
                                                                7             > WWV_FLOW_PAGE_PLUGS|APEX_040200*                         wwv_flow_step_items_plug_fk|apex_040200
                                                                7             < WWV_FLOW_STEP_ITEM_HELP|APEX_040200                      wwv_flow_item_helptext_fk|apex_040200                                                                 8               > WWV_FLOWS|APEX_040200*                                 wwv_flow_page_helptext_fk|apex_040200
                                                                6           < WWV_FLOW_STEP_PROCESSING|APEX_040200                       wwv_flow_step_proc_fk2|apex_040200                                                                 7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_step_proc_fk|apex_040200
                                                                7             < WWV_FLOW_CALS|APEX_040200*                               wwv_flow_step_process_fk|apex_040200                                                                 7             > WWV_FLOW_PAGE_PLUGS|APEX_040200*                         wwv_flow_step_proc_reg_fk|apex_040200
                                                                7             < WWV_FLOW_WS_PROCESS_PARMS_MAP|APEX_040200                wwv_flow_ws_map_fk2|apex_040200                                                                 8               > WWV_FLOW_WS_PARAMETERS|APEX_040200                     wwv_flows_ws_map_fk1|apex_040200
                                                                9                 > WWV_FLOW_WS_OPERATIONS|APEX_040200                   wwv_flow_ws_parms_fk|apex_040200
                                                               10                   > WWV_FLOW_SHARED_WEB_SERVICES|APEX_040200           wwv_flow_ws_opers_fk|apex_040200
                                                               11                     > WWV_FLOWS|APEX_040200*                           wwv_flow_ws_fk|apex_040200
                                                                6           < WWV_FLOW_STEP_VALIDATIONS|APEX_040200                      wwv_flow_step_val_fk2|apex_040200                                                                 7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_step_val_fk|apex_040200
                                                                7             > WWV_FLOW_PAGE_PLUGS|APEX_040200*                         wwv_flow_step_val_to_reg_fk|apex_040200
                                                                6           > WWV_FLOW_USER_INTERFACES|APEX_040200                       wwv_flow_step_ui_fk|apex_040200
                                                                7             > WWV_FLOWS|APEX_040200*                                   wwv_flow_ui_flow_fk|apex_040200
                                                                7             > WWV_FLOW_STEPS|APEX_040200*                              wwv_flow_user_int_page_fk|apex_040200
                                                                7             > WWV_FLOW_UI_TYPES|APEX_040200                            wwv_flow_ui_type_fk|apex_040200
                                                                8               < WWV_FLOW_THEMES|APEX_040200                            wwv_flow_theme_ui_type_fk|apex_040200                                                                 9                 > WWV_FLOWS|APEX_040200*                               wwv_flow_themes_2f_fk|apex_040200
                                                                8               = WWV_FLOW_UI_TYPES|APEX_040200*                         wwv_flow_ui_type_based_on_fk|apex_040200
                                                                8               < WWV_FLOW_UI_TYPE_FEATURES|APEX_040200                  wwv_flow_ui_type_feature_fk1|apex_040200                                                                 9                 > WWV_FLOW_BUILDER_FEATURES|APEX_040200                wwv_flow_ui_type_feature_fk2|apex_040200
                                                                3     < WWV_FLOW_PAGE_GENERIC_ATTR|APEX_040200                           wwv_flow_genattr_to_region_fk|apex_040200
                                                                3     = WWV_FLOW_PAGE_PLUGS|APEX_040200*                                 wwv_flow_plug_parent_fk|apex_040200
                                                                3     < WWV_FLOW_QUERY_DEFINITION|APEX_040200                            query_def_to_region_fk|apex_040200
                                                                4       < WWV_FLOW_QUERY_COLUMN|APEX_040200                              query_column_to_query_fk|apex_040200                                                                 5         > WWV_FLOW_QUERY_OBJECT|APEX_040200                            query_column_to_qry_object_fk|apex_040200
                                                                6           > WWV_FLOW_QUERY_DEFINITION|APEX_040200*                     query_object_to_query_fk|apex_040200
                                                                4       < WWV_FLOW_QUERY_CONDITION|APEX_040200                           query_condition_to_query_fk|apex_040200
                                                                3     < WWV_FLOW_REGION_CHART_SER_ATTR|APEX_040200                       wwv_flow_seattr_to_region_fk|apex_040200
                                                                3     < WWV_FLOW_REGION_REPORT_COLUMN|APEX_040200                        report_column_to_region_fk|apex_040200
                                                                3     < WWV_FLOW_REGION_REPORT_FILTER|APEX_040200                        sys_c007367|apex_040200
                                                                3     < WWV_FLOW_REGION_UPD_RPT_COLS|APEX_040200                         wwv_flow_urc_to_plug_fk|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_urc_to_flow_fk|apex_040200
                                                                3     < WWV_FLOW_TREE_REGIONS|APEX_040200                                wwv_flow_treeregion_fk2|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_treeregion_fk|apex_040200
                                                                3     < WWV_FLOW_WORKSHEETS|APEX_040200                                  wwv_flow_worksheets_reg_fk|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_worksheets_flow_fk|apex_040200
                                                                4       < WWV_FLOW_WORKSHEET_COLUMNS|APEX_040200                         wwv_flow_worksheet_columns_fk|apex_040200                                                                 5         > WWV_FLOWS|APEX_040200*                                       wwv_flow_worksheet_col_fk|apex_040200
                                                                5         > WWV_FLOW_WORKSHEET_COL_GROUPS|APEX_040200                    wwv_flow_worksheet_col_grps_fk|apex_040200
                                                                6           > WWV_FLOWS|APEX_040200*                                     wwv_flow_worksheet_col_grp_fk|apex_040200
                                                                6           > WWV_FLOW_WORKSHEETS|APEX_040200*                           wwv_flow_worksheet_col_grws_fk|apex_040200
                                                                6           > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200                      wwv_flow_worksheet_col_grp_fk2|apex_040200
                                                                7             > WWV_FLOW_WORKSHEETS|APEX_040200*                         wwwv_flow_ws_websheet_attr_fk|apex_040200
                                                                7             < WWV_FLOW_WORKSHEET_COLUMNS|APEX_040200*                  wwv_flow_worksheet_col_fk2|apex_040200
                                                                7             < WWV_FLOW_WORKSHEET_COMPUTATION|APEX_040200               wwv_flow_ws_computation_fk|apex_040200                                                                 8               > WWV_FLOW_WORKSHEET_RPTS|APEX_040200                    wwv_flow_ws_comp_cols_fk|apex_040200
                                                                9                 > WWV_FLOW_WORKSHEET_CATEGORIES|APEX_040200            wwv_flow_worksheet_rpts_fk|apex_040200
                                                               10                   > WWV_FLOWS|APEX_040200*                             wwv_flow_worksheet_cat_fk2|apex_040200
                                                                9                 < WWV_FLOW_WORKSHEET_CONDITIONS|APEX_040200            wwv_flow_worksheet_cond_fk|apex_040200                                                                10                   > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200*             wwv_flow_ws_condition_fk|apex_040200
                                                                9                 < WWV_FLOW_WORKSHEET_GROUP_BY|APEX_040200              wwv_flow_ws_groupby_fk2|apex_040200                                                                10                   > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200*             wwv_flow_ws_groupby_fk|apex_040200
                                                                9                 < WWV_FLOW_WORKSHEET_NOTIFY|APEX_040200                wwv_flow_worksheet_notify_fk2|apex_040200                                                                10                   > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200*             wwv_flow_worksheet_notify_fk4|apex_040200
                                                                9                 > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200*               wwv_flow_ws_rpt_fk|apex_040200
                                                                7             < WWV_FLOW_WORKSHEET_LOVS|APEX_040200                      wwv_flow_worksheet_lovs_fk2|apex_040200                                                                 8               > WWV_FLOW_WORKSHEETS|APEX_040200*                       wwv_flow_worksheet_lovs_fk|apex_040200
                                                                8               < WWV_FLOW_WORKSHEET_LOV_ENTRIES|APEX_040200             wwv_flow_worksheet_lov_ent_fk2|apex_040200                                                                 9                 > WWV_FLOW_WORKSHEETS|APEX_040200*                     wwv_flow_worksheet_lov_ent_fk|apex_040200
                                                                9                 > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200*               wwv_flow_worksheet_lov_ent_fk3|apex_040200
                                                                7             > WWV_FLOW_WS_APPLICATIONS|APEX_040200                     wwv_flow_ws_websheet_attr_fk2|apex_040200
                                                                8               < WWV_FLOW_PKG_APP_MAP|APEX_040200                       wwv_flow_pkg_app_map_fk2|apex_040200                                                                 9                 > WWV_FLOWS|APEX_040200*                               wwv_flow_pkg_app_map_fk|apex_040200
                                                                8               < WWV_FLOW_WS_APP_SUG_OBJECTS|APEX_040200                wwv_flow_ws_app_so_fk1|apex_040200
                                                                8               < WWV_FLOW_WS_COL_VALIDATIONS|APEX_040200                wwv_flow_ws_col_val_fk3|apex_040200                                                                 9                 > WWV_FLOW_WORKSHEETS|APEX_040200*                     wwv_flow_ws_col_val_fk|apex_040200
                                                                9                 > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040200*               wwv_flow_ws_col_val_fk2|apex_040200
                                                                8               < WWV_FLOW_WS_CUSTOM_AUTH_SETUPS|APEX_040200             wwv_flow_ws_auth_setups_fk|apex_040200
                                                                8               < WWV_FLOW_WS_DATA_GRID_Q|APEX_040200                    wwv_flow_ws_data_grid_q_fk|apex_040200                                                                 9                 > WWV_FLOW_WS_WEBPAGES|APEX_040200                     wwv_flow_ws_data_grid_q_fk2|apex_040200
                                                               10                   > WWV_FLOW_WS_APPLICATIONS|APEX_040200*              wwv_flow_ws_webpages_fk|apex_040200
                                                               10                   = WWV_FLOW_WS_WEBPAGES|APEX_040200*                  wwv_flow_ws_webpages_fk2|apex_040200
                                                                1 < WWV_FLOW_CAL_TEMPLATES|APEX_040200                                   wwv_flow_cal_templ_to_flow_fk|apex_040200
                                                                1 < WWV_FLOW_COMPUTATIONS|APEX_040200                                    wwv_flow_computations_fk|apex_040200
                                                                1 < WWV_FLOW_CUSTOM_AUTH_SETUPS|APEX_040200                              wwv_flow_auth_setups_fk|apex_040200
                                                                1 < WWV_FLOW_DYNAMIC_TRANSLATIONS$|APEX_040200                           wwv_flow_dynamic_trans_fk1|apex_040200
                                                                1 < WWV_FLOW_ENTRY_POINTS|APEX_040200                                    wwv_flow_entry_points_fk|apex_040200
                                                                2   < WWV_FLOW_ENTRY_POINT_ARGS|APEX_040200                              wwv_flow_entry_point_args_fk|apex_040200
                                                                1 < WWV_FLOW_ICON_BAR_ATTRIBUTES|APEX_040200                             wwv_flow_iconbarattr_fk|apex_040200
                                                                1 < WWV_FLOW_ICON_BAR|APEX_040200                                        wwv_flow_icon_bar_fk|apex_040200
                                                                1 < WWV_FLOW_INSTALL_BUILD_OPT|APEX_040200                               wwv_flow_install_build_opt_fk|apex_040200                                                                 2   > WWV_FLOW_INSTALL|APEX_040200                                       wwv_flow_install_build_opt_fk3|apex_040200
                                                                3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_install_fk|apex_040200
                                                                3     < WWV_FLOW_INSTALL_CHECKS|APEX_040200                              wwv_flow_install_checks_fk3|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_install_checks_fk|apex_040200
                                                                3     < WWV_FLOW_INSTALL_SCRIPTS|APEX_040200                             wwv_flow_install_scripts_fk3|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_install_scripts_fk|apex_040200
                                                                2   > WWV_FLOW_PATCHES|APEX_040200                                       wwv_flow_install_build_opt_fk4|apex_040200
                                                                3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_patches_fk|apex_040200
                                                                1 < WWV_FLOW_ITEMS|APEX_040200                                           wwv_flow_items_fk|apex_040200
                                                                1 < WWV_FLOW_LANGUAGE_MAP|APEX_040200                                    wwv_flow_lang_flow_id_fk|apex_040200
                                                                1 < WWV_FLOW_LISTS_OF_VALUES$|APEX_040200                                wwv_flow_lov_fk|apex_040200
                                                                2   < WWV_FLOW_LIST_OF_VALUES_DATA|APEX_040200                           wwv_flow_lov_data_fk|apex_040200
                                                                2   < WWV_FLOW_LOAD_TABLES|APEX_040200                                   wwv_flow_load_tab_fk2|apex_040200                                                                 3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_load_tab_fk1|apex_040200
                                                                3     < WWV_FLOW_LOAD_TABLE_LOOKUPS|APEX_040200                          wwv_flow_load_tab_lk_fk2|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_load_tab_lk_fk1|apex_040200
                                                                3     < WWV_FLOW_LOAD_TABLE_RULES|APEX_040200                            wwv_flow_load_tab_rule_fk2|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_load_tab_rule_fk1|apex_040200
                                                                1 < WWV_FLOW_LISTS|APEX_040200                                           wwv_flow_lists_flow_fk|apex_040200
                                                                2   < WWV_FLOW_LIST_ITEMS|APEX_040200                                    wwv_flow_list_items_fk|apex_040200
                                                                3     = WWV_FLOW_LIST_ITEMS|APEX_040200*                                 parent_list_item_fk|apex_040200
                                                                1 < WWV_FLOW_LIST_TEMPLATES|APEX_040200                                  wwv_flow_list_template_fk|apex_040200
                                                                1 < WWV_FLOW_LOCK_PAGE|APEX_040200                                       sys_c007221|apex_040200
                                                                1 < WWV_FLOW_MENUS|APEX_040200                                           wwv_flow_menus_flow_fk|apex_040200
                                                                2   < WWV_FLOW_MENU_OPTIONS|APEX_040200                                  wwv_flow_opt_menus_fk|apex_040200
                                                                1 < WWV_FLOW_MENU_TEMPLATES|APEX_040200                                  wwv_flow_menus_t_flow_fk|apex_040200
                                                                1 < WWV_FLOW_MESSAGES$|APEX_040200                                       wwv_flow_messages_fk|apex_040200
                                                                1 < WWV_FLOW_PAGES_RESERVED|APEX_040200                                  wwv_flow_pages_reserved_fk|apex_040200
                                                                1 < WWV_FLOW_PAGE_CACHE|APEX_040200                                      wwv_flow_page_cache_fk|apex_040200
                                                                2   < WWV_FLOW_PAGE_CODE_CACHE|APEX_040200                               wwv_flow_page_code_cache_fk|apex_040200
                                                                1 < WWV_FLOW_PAGE_GROUPS|APEX_040200                                     sys_c007397|apex_040200
                                                                1 < WWV_FLOW_PAGE_SUBMISSIONS|APEX_040200                                wwv_flow_page_sub_fk|apex_040200                                                                 2   > WWV_FLOW_SESSIONS$|APEX_040200                                     wwv_flow_page_sub_sess_fk|apex_040200
                                                                3     < WWV_FLOW_COLLECTIONS$|APEX_040200                                wwv_flow_collection_fk|apex_040200
                                                                4       < WWV_FLOW_COLLECTION_MEMBERS$|APEX_040200                       wwv_flow_collection_membes_fk|apex_040200
                                                                3     < WWV_FLOW_DATA|APEX_040200                                        wwv_flow_data_session_fk|apex_040200
                                                                3     < WWV_FLOW_REQUEST_VERIFICATIONS|APEX_040200                       wwv_flow_request_verif_fk|apex_040200
                                                                3     < WWV_FLOW_RT$USER_SESSIONS|APEX_040200                            wwv_flow_rt$user_sess_fk1|apex_040200                                                                 4       > WWV_FLOW_RT$APPROVALS|APEX_040200                              wwv_flow_rt$user_sess_fk|apex_040200
                                                                5         < WWV_FLOW_RT$APPROVAL_PRIVS|APEX_040200                       wwv_flow_rt$app_privs_fk|apex_040200                                                                 6           > WWV_FLOW_RT$PRIVILEGES|APEX_040200                         wwv_flow_rt$app_privs_fk2|apex_040200
                                                                7             < WWV_FLOW_RT$CLIENT_PRIVILEGES|APEX_040200                wwv_flow_rt$client_privs_fk2|apex_040200                                                                 8               > WWV_FLOW_RT$CLIENTS|APEX_040200                        wwv_flow_rt$client_privs_fk|apex_040200
                                                                9                 > WWV_FLOWS|APEX_040200*                               wwv_flow_rt$clients_appid_fk|apex_040200
                                                                9                 < WWV_FLOW_RT$APPROVALS|APEX_040200*                   wwv_flow_rt$approvals_fk|apex_040200
                                                                7             < WWV_FLOW_RT$HANDLERS|APEX_040200                         wwv_flow_rt$handlers_priv_fk|apex_040200
                                                                8               < WWV_FLOW_RT$ERRORS|APEX_040200                         wwv_flow_rt$errors_handler_fk|apex_040200
                                                                8               < WWV_FLOW_RT$PARAMETERS|APEX_040200                     wwv_flow_rt$params_handler_fk|apex_040200                                                                 8               > WWV_FLOW_RT$TEMPLATES|APEX_040200                      wwv_flow_rt$handlers_temps_fk|apex_040200
                                                                9                 > WWV_FLOW_RT$MODULES|APEX_040200                      wwv_flow_rt$temps_mod_fk|apex_040200
                                                               10                   > WWV_FLOW_RT$PRIVILEGES|APEX_040200*                wwv_flow_rt$modules_priv_fk|apex_040200
                                                                7             < WWV_FLOW_RT$PRIVILEGE_GROUPS|APEX_040200                 wwv_flow_rt$priv_groups_fk|apex_040200                                                                 8               > WWV_FLOW_FND_USER_GROUPS|APEX_040200                   wwv_flow_rt$priv_groups_fk2|apex_040200
                                                                9                 < WWV_FLOW_FND_GROUP_USERS|APEX_040200                 wwv_flow_fnd_gu_int_g_fk|apex_040200
                                                                5         < WWV_FLOW_RT$PENDING_APPROVALS|APEX_040200                    wwv_flow_rt$pend_apprv_fk|apex_040200
                                                                3     < WWV_FLOW_SC_TRANS|APEX_040200                                    wwv_flow_sc_trans_fk2|apex_040200
                                                                3     < WWV_FLOW_TREE_STATE|APEX_040200                                  wwv_flow_tree_state$fk|apex_040200
                                                                1 < WWV_FLOW_PAGE_TMPL_DISP_POINTS|APEX_040200                           wwv_page_tmpl_dp_fk|apex_040200                                                                 2   > WWV_FLOW_TEMPLATES|APEX_040200                                     wwv_page_tmpl_dp_parent_fk|apex_040200
                                                                3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_templates_fk|apex_040200
                                                                1 < WWV_FLOW_PLUGINS|APEX_040200                                         wwv_flow_plugin_flow_fk|apex_040200
                                                                2   < WWV_FLOW_PLUGIN_ATTRIBUTES|APEX_040200                             wwv_flow_plugin_attr_parent_fk|apex_040200                                                                 3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_plugin_attr_flow_fk|apex_040200
                                                                3     = WWV_FLOW_PLUGIN_ATTRIBUTES|APEX_040200*                          wwv_flow_plugin_attr_depend_fk|apex_040200
                                                                3     < WWV_FLOW_PLUGIN_ATTR_VALUES|APEX_040200                          wwv_flow_plugin_attrv_attr_fk|apex_040200                                                                 4       > WWV_FLOWS|APEX_040200*                                         wwv_flow_plugin_attrv_flow_fk|apex_040200
                                                                2   < WWV_FLOW_PLUGIN_EVENTS|APEX_040200                                 wwv_flow_plugin_evnt_parent_fk|apex_040200                                                                 3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_plugin_evnt_flow_fk|apex_040200
                                                                2   < WWV_FLOW_PLUGIN_FILES|APEX_040200                                  wwv_flow_plugin_file_parent_fk|apex_040200                                                                 3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_plugin_file_flow_fk|apex_040200
                                                                1 < WWV_FLOW_PLUGIN_SETTINGS|APEX_040200                                 wwv_flow_plugin_set_flow_fk|apex_040200
                                                                1 < WWV_FLOW_POPUP_LOV_TEMPLATE|APEX_040200                              wwv_flow_fk_poplov_temp|apex_040200
                                                                1 < WWV_FLOW_PROCESSING|APEX_040200                                      wwv_flow_processing_fk|apex_040200
                                                                1 < WWV_FLOW_REPORT_LAYOUTS|APEX_040200                                  wwv_flow_report_layoutse_fk|apex_040200
                                                                1 < WWV_FLOW_REQUIRED_ROLES|APEX_040200                                  wwv_flow_req_roles_fk|apex_040200
                                                                1 < WWV_FLOW_ROW_TEMPLATES|APEX_040200                                   wwv_flow_row_template_fk|apex_040200
                                                                1 < WWV_FLOW_SECURITY_SCHEMES|APEX_040200                                wwv_flow_sec_schemes_fk|apex_040200
                                                                1 < WWV_FLOW_SHARED_QRY_SQL_STMTS|APEX_040200                            wwv_flow_sqry_sql_flow_fk|apex_040200                                                                 2   > WWV_FLOW_SHARED_QUERIES|APEX_040200                                wwv_flow_sqry_sql_sqry_fk|apex_040200
                                                                3     > WWV_FLOWS|APEX_040200*                                           wwv_flow_shdqry_flow_fk|apex_040200
                                                                1 < WWV_FLOW_SHORTCUTS|APEX_040200                                       wwv_flow_shortcuts_to_flow_fk|apex_040200
                                                                1 < WWV_FLOW_TABS|APEX_040200                                            wwv_flow_tabs_fk|apex_040200
                                                                1 < WWV_FLOW_TEMPLATE_PREFERENCES|APEX_040200                            wwv_flow_templ_pref_fk|apex_040200
                                                                1 < WWV_FLOW_THEME_DISPLAY_POINTS|APEX_040200                            wwv_theme_disp_point_fk|apex_040200
                                                                1 < WWV_FLOW_THEME_STYLES|APEX_040200                                    wwv_flow_theme_style_flow_fk|apex_040200
                                                                1 < WWV_FLOW_TOPLEVEL_TABS|APEX_040200                                   wwv_flow_toplev_tab_fk|apex_040200
                                                                1 < WWV_FLOW_TRANSLATABLE_TEXT$|APEX_040200                              wwv_flow_trans_text_fk|apex_040200
                                                                1 < WWV_FLOW_TREES|APEX_040200                                           wwv_flow_tree_fk|apex_040200
                                                                1 < WWV_FLOW_VALIDATIONS|APEX_040200                                     wwv_flow_val_fk|apex_040200
                                                                1 < WWV_MIG_GENERATED_APPLICATIONS|APEX_040200                           wwv_mig_gen_app_flow_id_fk|apex_040200                                                                 2   > WWV_MIG_PROJECTS|APEX_040200                                       wwv_mig_gen_app_proj_id_fk|apex_040200
                                                                3     < WWV_MIG_ACCESS|APEX_040200                                       wwv_mig_acc_fk|apex_040200
                                                                3     < WWV_MIG_FORMS|APEX_040200                                        wwv_mig_forms_project_id_fk|apex_040200
                                                                4       < WWV_MIG_FRM_MODULES|APEX_040200                                wwv_mig_frm_modules_file_id_fk|apex_040200
                                                                5         < WWV_MIG_FRM_FORMMODULES|APEX_040200                          wwv_mig_frm_frmmdl_mdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_ALERTS|APEX_040200                             wwv_mig_frm_alrt_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_ATTACHEDLIBRARY|APEX_040200                    wwv_mig_frm_atlib_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_BLOCKS|APEX_040200                             wwv_mig_frm_blk_frmmdl_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_BLK_DSA|APEX_040200                          wwv_mig_frm_blk_dsa_blk_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_BLK_DSC|APEX_040200                          wwv_mig_frm_blk_dsc_blk_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_BLK_ITEMS|APEX_040200                        wwv_mig_frm_bi_blk_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_BLK_ITEM_LIE|APEX_040200                   wwv_mig_frm_bi_lie_item_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_BLK_ITEM_RADIO|APEX_040200                 wwv_mig_frm_bir_item_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_BLK_ITEM_TRIGGERS|APEX_040200              wwv_mig_frm_bi_trg_item_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_REV_BLK_ITEMS|APEX_040200                  wwv_mig_frm_rev_bi_item_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_BLK_RELATIONS|APEX_040200                    wwv_mig_frm_blk_rel_blk_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_BLK_TRIGGERS|APEX_040200                     wwv_mig_frm_blk_trg_blk_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_REV_BLOCKS|APEX_040200                       wwv_mig_frm_rev_blocks_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_CANVAS|APEX_040200                             wwv_mig_frm_canvs_frmmdl_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_CNVS_GRAPHICS|APEX_040200                    wwv_mig_frm_cg_cnvs_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_CNVG_COMPOUNDTEXT|APEX_040200              wwv_mig_frm_cpdtxt_grphs_id_fk|apex_040200
                                                                9                 < WWV_MIG_FRM_CPDTXT_TEXTSEGMENT|APEX_040200           wwv_mig_frm_txtsgmt_cpd_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_CNVS_TABPAGE|APEX_040200                     wwv_mig_frm_ctp_cnvs_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_COORDINATES|APEX_040200                        wwv_mig_frm_crdnt_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_EDITOR|APEX_040200                             wwv_mig_frm_edtr_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_FMB_MENU|APEX_040200                           wwv_mig_frm_menu_frmmdl_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_FMB_MENU_MENUITEM|APEX_040200                wwv_mig_fmb_menuitem_menuid_fk|apex_040200
                                                                8               < WWV_MIG_FRM_FMB_MENUITEM_ROLE|APEX_040200              wwv_mig_fmb_mnuitemrl_mitm_fk|apex_040200
                                                                6           < WWV_MIG_FRM_LOV|APEX_040200                                wwv_mig_frm_lov_frmmdl_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_LOVCOLUMNMAPPING|APEX_040200                 wwv_mig_frm_lvcm_frmmdl_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_REV_LOVCOLMAPS|APEX_040200                 wwv_mig_frm_rev_lcm_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_REV_LOV|APEX_040200                          wwv_mig_frm_rev_lov_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_MODULEPARAMETER|APEX_040200                    wwv_mig_frm_mdlpr_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_OBJECTGROUP|APEX_040200                        wwv_mig_frm_objgp_frmmdl_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_OBJECTGROUPCHILD|APEX_040200                 wwv_mig_frm_objgpc_objgp_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_PROGRAMUNIT|APEX_040200                        wwv_mig_frm_pgut_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_PROPERTYCLASS|APEX_040200                      wwv_mig_frm_ppcl_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_RECORDGROUPS|APEX_040200                       wwv_mig_frm_recgp_frmmdl_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_RECORDGROUPCOLUMN|APEX_040200                wwv_mig_frm_rgc_recgp_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_REPORT|APEX_040200                             wwv_mig_frm_rpt_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_REV_FORMMODULES|APEX_040200                    wwv_mig_frm_rev_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_TRIGGERS|APEX_040200                           wwv_mig_frm_trg_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_VISUALATTRIBUTES|APEX_040200                   wwv_mig_frm_visat_frmmdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_WINDOWS|APEX_040200                            wwv_mig_frm_wndow_frmmdl_id_fk|apex_040200
                                                                3     < WWV_MIG_FRM_MENUS|APEX_040200                                    wwv_mig_menus_project_id_fk|apex_040200
                                                                4       < WWV_MIG_FRM_MENUS_MODULES|APEX_040200                          wwv_mig_mnu_modules_file_id_fk|apex_040200
                                                                5         < WWV_MIG_FRM_MENUS_MENUMODULES|APEX_040200                    wwv_mig_mnu_mnumdl_mdl_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_MENUSMODULEROLES|APEX_040200                   wwv_mig_mmodrole_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_MENUS_PROGRAMUNIT|APEX_040200                  wwv_mig_mnu_progunit_id_fk|apex_040200
                                                                6           < WWV_MIG_FRM_MENU|APEX_040200                               wwv_mig_mnu_id_fk|apex_040200
                                                                7             < WWV_MIG_FRM_MENU_MENUITEM|APEX_040200                    wwv_mig_mnuitem_id_fk|apex_040200
                                                                8               < WWV_MIG_FRM_MENUITEM_ROLE|APEX_040200                  wwv_mig_mnuitemrole_id_fk|apex_040200
                                                                3     < WWV_MIG_FRM_REV_APEX_APP|APEX_040200                             wwv_mig_frm_rev_apex_app_fk|apex_040200
                                                                3     < WWV_MIG_OLB|APEX_040200                                          wwv_mig_olb_project_id_fk|apex_040200
                                                                4       < WWV_MIG_OLB_MODULES|APEX_040200                                wwv_mig_olb_modules_file_id_fk|apex_040200
                                                                5         < WWV_MIG_OLB_OBJECTLIBRARY|APEX_040200                        wwv_mig_olb_objlib_mdl_id_fk|apex_040200
                                                                6           < WWV_MIG_OLB_BLOCK|APEX_040200                              wwv_mig_olb_block_objlib_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_BLK_DATASOURCECOL|APEX_040200                wwv_mig_olb_blk_dsc_blk_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_BLK_ITEM|APEX_040200                         wwv_mig_olb_blk_item_blk_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_BLK_ITEM_LIE|APEX_040200                   wwv_mig_olb_bil_item_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_BLK_ITEM_TRIGGER|APEX_040200               wwv_mig_olb_bit_item_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_BLK_TRIGGER|APEX_040200                      wwv_mig_olb_blk_trgr_blk_id_fk|apex_040200
                                                                6           < WWV_MIG_OLB_CANVAS|APEX_040200                             wwv_mig_olb_canvs_objlib_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_CNVS_GRAPHICS|APEX_040200                    wwv_mig_olb_cg_cnvs_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_CG_COMPOUNDTEXT|APEX_040200                wwv_mig_olb_cg_ct_grphs_id_fk|apex_040200
                                                                9                 < WWV_MIG_OLB_CG_CT_TEXTSEGMENT|APEX_040200            wwv_mig_olb_cg_ct_ts_ct_id_fk|apex_040200
                                                                6           < WWV_MIG_OLB_OBJECTLIBRARYTAB|APEX_040200                   wwv_mig_olb_olt_objlib_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_ALERT|APEX_040200                        wwv_mig_olb_olt_alrt_olt_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_BLOCK|APEX_040200                        wwv_mig_olb_t_block_olt_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_OLT_BLK_ITEM|APEX_040200                   wwv_mig_olb_olt_bi_blk_id_fk|apex_040200
                                                                9                 < WWV_MIG_OLB_OLT_BLK_ITEM_TRIGR|APEX_040200           wwv_mig_olb_olt_bit_item_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_CANVAS|APEX_040200                       wwv_mig_olb_t_canvas_olt_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_OLT_CNVS_GRAPHICS|APEX_040200              wwv_mig_olb_olt_cg_cnvs_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_GRAPHICS|APEX_040200                     wwv_mig_olb_t_grphcs_olt_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_ITEM|APEX_040200                         wwv_mig_olb_olt_item_olt_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_MENU|APEX_040200                         wwv_mig_olb_olt_menu_olt_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_OLT_MENU_MENUITEM|APEX_040200              wwv_mig_olb_olt_mmi_menu_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_OBJECTGROUP|APEX_040200                  wwv_mig_olb_t_objgrp_olt_id_fk|apex_040200
                                                                8               < WWV_MIG_OLB_OLT_OB_OBJGRPCHILD|APEX_040200             wwv_mig_olb_olt_ob_ogc_obid_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_REPORT|APEX_040200                       wwv_mig_olb_t_report_olt_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_TABPAGE|APEX_040200                      wwv_mig_olb_t_tabpage_oltid_fk|apex_040200
                                                                8               < WWV_MIG_OLB_OLT_TABPG_GRAPHICS|APEX_040200             wwv_mig_olb_olt_tpg_tp_id_fk|apex_040200
                                                                9                 < WWV_MIG_OLB_T_TP_G_GRAPHICS|APEX_040200              wwv_mig_olb_t_tp_gg_g_id_fk|apex_040200
                                                               10                   < WWV_MIG_OLB_T_TP_GG_CPDTXT|APEX_040200             wwv_mig_olb_t_tp_gg_ct_g_id_fk|apex_040200
                                                               11                     < WWV_MIG_OLB_T_TP_GG_CT_TXTSGT|APEX_040200        wwv_mig_olb_ttpggctts_ctid_fk|apex_040200
                                                               10                   < WWV_MIG_OLB_T_TP_GG_GRAPHICS|APEX_040200           wwv_mig_olb_t_tp_ggg_g_id_fk|apex_040200
                                                               11                     < WWV_MIG_OLB_T_TP_GGG_CPDTXT|APEX_040200          wwv_mig_olb_ttp_ggg_ct_gid_fk|apex_040200
                                                               12                       < WWV_MIG_OLB_T_TP_GGG_CT_TXTSGT|APEX_040200     wwv_mig_olb_ttpgggctts_ctid_fk|apex_040200
                                                               11                     < WWV_MIG_OLB_T_TP_GGG_GRAPHICS|APEX_040200        wwv_mig_olb_t_tp_gggg_g_id_fk|apex_040200
                                                               12                       < WWV_MIG_OLB_T_TP_GGGG_CPDTXT|APEX_040200       wwv_mig_olb_ttpggggct_g_id_fk|apex_040200
                                                               13                         < WWV_MIG_OLB_T_TP_GGGG_CT_TXSGT|APEX_040200   wwv_mig_olb_ttpggggcts_ctid_fk|apex_040200
                                                               12                       < WWV_MIG_OLB_T_TP_GGGG_GRAPHICS|APEX_040200     wwv_mig_olb_ttpggggg_g_id_fk|apex_040200
                                                               13                         < WWV_MIG_OLB_T_TP_GGGGG_CPDTXT|APEX_040200    wwv_mig_olb_ttpgggggct_g_id_fk|apex_040200
                                                               14                           < WWV_MIG_OLB_T_TP_GGGGG_CT_TXST|APEX_040200 wwv_mig_olb_ttp5gcts_ct_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_VISUALATTRBUTE|APEX_040200               wwv_mig_olb_olt_va_olt_id_fk|apex_040200
                                                                7             < WWV_MIG_OLB_OLT_WINDOW|APEX_040200                       wwv_mig_olb_olt_wndow_oltid_fk|apex_040200
                                                                6           < WWV_MIG_OLB_PROGRAMUNIT|APEX_040200                        wwv_mig_olb_pu_objlib_id_fk|apex_040200
                                                                6           < WWV_MIG_OLB_PROPERTYCLASS|APEX_040200                      wwv_mig_olb_pc_objlib_id_fk|apex_040200
                                                                6           < WWV_MIG_OLB_VISUALATTRIBUTE|APEX_040200                    wwv_mig_olb_va_objlib_id_fk|apex_040200
                                                                6           < WWV_MIG_OLB_WINDOW|APEX_040200                             wwv_mig_olb_wndow_objlib_id_fk|apex_040200
                                                                3     < WWV_MIG_PLSQL_LIBS|APEX_040200                                   wwv_mig_plls_project_id_fk|apex_040200
                                                                3     < WWV_MIG_PROJECT_COMPONENTS|APEX_040200                           wwv_mig_proj_comp_fk|apex_040200
                                                                3     < WWV_MIG_PROJECT_TRIGGERS|APEX_040200                             wwv_mig_proj_trig_fk|apex_040200
                                                                3     < WWV_MIG_RPTS|APEX_040200                                         wwv_mig_rpts_project_id_fk|apex_040200
                                                                4       < WWV_MIG_REPORT|APEX_040200                                     wwv_mig_rep_file_id_fk|apex_040200
                                                                5         < WWV_MIG_RPT_DATA|APEX_040200                                 wwv_mig_repdata_id_fk|apex_040200
                                                                6           < WWV_MIG_RPT_DATASRC|APEX_040200                            wwv_mig_repsrc_id_fk|apex_040200
                                                                7             < WWV_MIG_RPT_DATASRC_GRP|APEX_040200                      wwv_mig_grp_id_fk|apex_040200
                                                                8               < WWV_MIG_RPT_GRP_DATAITEM|APEX_040200                   wwv_mig_grp_dataitem_id_fk|apex_040200
                                                                9                 < WWV_MIG_RPT_GRP_DATAITEM_DESC|APEX_040200            wwv_mig_grp_itemdesc_id_fk|apex_040200
                                                                9                 < WWV_MIG_RPT_GRP_DATAITEM_PRIV|APEX_040200            wwv_mig_grp_itempriv_id_fk|apex_040200
                                                                8               < WWV_MIG_RPT_GRP_FIELD|APEX_040200                      wwv_mig_grp_fld_id_fk|apex_040200
                                                                8               < WWV_MIG_RPT_GRP_FILTER|APEX_040200                     wwv_mig_grp_fltr_id_fk|apex_040200
                                                                8               < WWV_MIG_RPT_GRP_FORMULA|APEX_040200                    wwv_mig_grp_form_id_fk|apex_040200
                                                                8               < WWV_MIG_RPT_GRP_ROWDELIM|APEX_040200                   wwv_mig_grp_row_id_fk|apex_040200
                                                                8               < WWV_MIG_RPT_GRP_SUMMARY|APEX_040200                    wwv_mig_grp_sum_id_fk|apex_040200
                                                                7             < WWV_MIG_RPT_DATASRC_SELECT|APEX_040200                   wwv_mig_select_id_fk|apex_040200
                                                                6           < WWV_MIG_RPT_DATA_SUMMARY|APEX_040200                       wwv_mig_repsum_id_fk|apex_040200
                                                                5         < WWV_MIG_RPT_REPORTPRIVATE|APEX_040200                        wwv_mig_rptpriv_id_fk|apex_040200
WWV_FLOW_ADVISOR_CATEGORIES|APEX_040200          2       3      0 WWV_FLOW_ADVISOR_CATEGORIES|APEX_040200                                ROOT
                                                                1 < WWV_FLOW_ADVISOR_CHECKS|APEX_040200                                  wwv_flow_adv_chk_cat_fk|apex_040200
                                                                2   < WWV_FLOW_ADVISOR_CHECK_MSGS|APEX_040200                            wwv_flow_adv_chk_msg_check_fk|apex_040200
WWV_FLOW_BUGS|APEX_040200                       13      11      0 WWV_FLOW_BUGS|APEX_040200                                              ROOT
                                                                1 < WWV_FLOW_TEAMDEV_TAG_CLOUD|APEX_040200                               wwv_flow_teamdev_tc_b|apex_040200                                                                 2   > WWV_FLOW_FEATURES|APEX_040200                                      wwv_flow_teamdev_tc_f|apex_040200
                                                                3     = WWV_FLOW_FEATURES|APEX_040200*                                   wwv_flow_features_par_feat_fk|apex_040200
                                                                3     < WWV_FLOW_FEATURE_HISTORY|APEX_040200                             wwv_flow_feature_hist_fk|apex_040200
                                                                3     < WWV_FLOW_FEATURE_PROGRESS|APEX_040200                            wwv_flow_feature_prog_fk|apex_040200
                                                                3     < WWV_FLOW_TEAM_FILES|APEX_040200                                  wwv_flow_team_files_fk1|apex_040200                                                                 4       > WWV_FLOW_EVENTS|APEX_040200                                    wwv_flow_team_files_fk3|apex_040200
                                                                4       > WWV_FLOW_FEEDBACK|APEX_040200                                  wwv_flow_team_files_fk4|apex_040200
                                                                5         < WWV_FLOW_FEEDBACK_FOLLOWUP|APEX_040200                       wwv_flow_feedback_fup_fk|apex_040200
                                                                5         < WWV_FLOW_TEAM_FILES|APEX_040200*                             wwv_flow_team_files_fk5|apex_040200                                                                 4       > WWV_FLOW_TASKS|APEX_040200                                     wwv_flow_team_files_fk2|apex_040200
                                                                5         < WWV_FLOW_TASK_PROGRESS|APEX_040200                           wwv_flow_task_prog_fk|apex_040200
                                                                5         < WWV_FLOW_TEAMDEV_TAG_CLOUD|APEX_040200*                      wwv_flow_teamdev_tc_t|apex_040200 WWV_FLOW_DATA_LOAD_BAD_LOG|APEX_040200           1       2      0 WWV_FLOW_DATA_LOAD_BAD_LOG|APEX_040200                                 ROOT                                                                 1 > WWV_FLOW_DATA_LOAD_UNLOAD|APEX_040200                                wwv_flow_data_load_bad_log_fk1|apex_040200
WWV_FLOW_DICTIONARY_VIEWS|APEX_040200            1       1      0 WWV_FLOW_DICTIONARY_VIEWS|APEX_040200                                  ROOT
                                                                1 = WWV_FLOW_DICTIONARY_VIEWS|APEX_040200*                               wwv_flow_dict_view_parent_fk|apex_040200
WWV_FLOW_FILE_OBJECTS$|FLOWS_FILES               5       6      0 WWV_FLOW_FILE_OBJECTS$|FLOWS_FILES                                     ROOT
                                                                1 < WWV_FLOW_IMPORT_EXPORT|APEX_040200                                   wwv_flow_import_export_fk|apex_040200
                                                                1 < WWV_FLOW_SW_BINDS|APEX_040200                                        wwv_flow_sw_bind_fk|apex_040200
                                                                1 < WWV_FLOW_SW_RESULTS|APEX_040200                                      wwv_flow_sw_result_fk|apex_040200
                                                                2   < WWV_FLOW_SW_DETAIL_RESULTS|APEX_040200                             wwv_flow_sw_d_result_fk|apex_040200
                                                                1 < WWV_FLOW_SW_STMTS|APEX_040200                                        wwv_flow_sw_stmts_fk|apex_040200 WWV_FLOW_FLASH_MAP_FILES|APEX_040200             2       3      0 WWV_FLOW_FLASH_MAP_FILES|APEX_040200                                   ROOT                                                                 1 > WWV_FLOW_FLASH_MAP_FOLDERS|APEX_040200                               wwv_flow_flash_map_files_fk|apex_040200
                                                                1 < WWV_FLOW_FLASH_MAP_REGIONS|APEX_040200                               wwv_flow_flash_map_reg_fk|apex_040200 WWV_FLOW_HNT_ARGUMENT_INFO|APEX_040200           1       2      0 WWV_FLOW_HNT_ARGUMENT_INFO|APEX_040200                                 ROOT                                                                 1 > WWV_FLOW_HNT_PROCEDURE_INFO|APEX_040200                              wwv_flow_hnt_arg_info_proc_fk|apex_040200
WWV_FLOW_HNT_COLUMN_DICT|APEX_040200             1       2      0 WWV_FLOW_HNT_COLUMN_DICT|APEX_040200                                   ROOT
                                                                1 < WWV_FLOW_HNT_COL_DICT_SYN|APEX_040200                                wwv_flow_hnt_col_dict_syn_fk|apex_040200 WWV_FLOW_HNT_COLUMN_INFO|APEX_040200             4       4      0 WWV_FLOW_HNT_COLUMN_INFO|APEX_040200                                   ROOT                                                                 1 > WWV_FLOW_HNT_GROUPS|APEX_040200                                      wwv_flow_hnt_col_info_grp_fk|apex_040200
                                                                2   > WWV_FLOW_HNT_TABLE_INFO|APEX_040200                                wwv_flow_hnt_groups_tab_fk|apex_040200
                                                                3     < WWV_FLOW_HNT_COLUMN_INFO|APEX_040200*                            wwv_flow_hnt_col_info_tab_fk|apex_040200
                                                                1 < WWV_FLOW_HNT_LOV_DATA|APEX_040200                                    wwv_flow_hnt_lov_data_col_fk|apex_040200 WWV_FLOW_MAIL_ATTACHMENTS|APEX_040200            1       2      0 WWV_FLOW_MAIL_ATTACHMENTS|APEX_040200                                  ROOT                                                                 1 > WWV_FLOW_MAIL_QUEUE|APEX_040200                                      wwv_flow_mail_attachments_fk1|apex_040200
WWV_FLOW_MODELS|APEX_040200                      4       4      0 WWV_FLOW_MODELS|APEX_040200                                            ROOT
                                                                1 < WWV_FLOW_MODEL_PAGES|APEX_040200                                     wwv_flow_model_pages_fk|apex_040200
                                                                2   = WWV_FLOW_MODEL_PAGES|APEX_040200*                                  wwv_flow_model_pages_fk2|apex_040200
                                                                2   < WWV_FLOW_MODEL_PAGE_REGIONS|APEX_040200                            wwv_flow_mpr_fk|apex_040200
                                                                3     < WWV_FLOW_MODEL_PAGE_COLS|APEX_040200                             wwv_flow_model_page_cols_fk|apex_040200 WWV_FLOW_PKG_APPLICATIONS|APEX_040200            4       3      0 WWV_FLOW_PKG_APPLICATIONS|APEX_040200                                  ROOT                                                                 1 > WWV_FLOW_PKG_APP_CATEGORIES|APEX_040200                              wwv_flow_pkg_app_fk1|apex_040200
                                                                2   < WWV_FLOW_PKG_APPLICATIONS|APEX_040200*                             wwv_flow_pkg_app_fk2|apex_040200
                                                                2   < WWV_FLOW_PKG_APPLICATIONS|APEX_040200*                             wwv_flow_pkg_app_fk3|apex_040200
                                                                1 < WWV_FLOW_PKG_APP_IMAGES|APEX_040200                                  wwv_flow_pkg_app_images_fk1|apex_040200 WWV_FLOW_QB_SAVED_COND|APEX_040200               3       4      0 WWV_FLOW_QB_SAVED_COND|APEX_040200                                     ROOT                                                                 1 > WWV_FLOW_QB_SAVED_QUERY|APEX_040200                                  sys_c007435|apex_040200
                                                                2   < WWV_FLOW_QB_SAVED_JOIN|APEX_040200                                 sys_c007442|apex_040200
                                                                2   < WWV_FLOW_QB_SAVED_TABS|APEX_040200                                 sys_c007449|apex_040200
WWV_FLOW_RESTRICTED_SCHEMAS|APEX_040200          1       2      0 WWV_FLOW_RESTRICTED_SCHEMAS|APEX_040200                                ROOT
                                                                1 < WWV_FLOW_RSCHEMA_EXCEPTIONS|APEX_040200                              wwv_flow_rschema_exceptions_fk|apex_040200 WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040200       1       1      0 WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040200                             ROOT                                                                 1 = WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040200*                          wwv_mig_olb_xmltagtablemap_fk|apex_040200 WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040200           1       1      0 WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040200                                 ROOT                                                                 1 = WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040200*                              wwv_mig_frm_xmltagtablemap_fk|apex_040200 WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040200          1       1      0 WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040200                                ROOT                                                                 1 = WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040200*                             wwv_mig_mnu_xmltagtablemap_fk|apex_040200 WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040200           1       1      0 WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040200                                 ROOT                                                                 1 = WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040200*                              wwv_mig_rpt_xmltagtablemap_fk|apex_040200 WWV_PURGE_DATAFILES|APEX_040200                  4       5      0 WWV_PURGE_DATAFILES|APEX_040200                                        ROOT                                                                 1 > WWV_PURGE_WORKSPACES|APEX_040200                                     wwv_purge_datafiles_fk1|apex_040200
                                                                2   < WWV_PURGE_EMAILS|APEX_040200                                       wwv_purge_emails_fk1|apex_040200
                                                                3     < WWV_PURGE_WORKSPACE_RESPONSES|APEX_040200                        wwv_purge_workspace_resp_fk1|apex_040200
                                                                2   < WWV_PURGE_SCHEMAS|APEX_040200                                      wwv_purge_schemas_fk1|apex_040200 XS$ACE_PRIV|SYS                                 36      25      0 XS$ACE_PRIV|SYS                                                        ROOT                                                                 1 > XS$ACE|SYS                                                           xs$ace_priv_fk1|sys
                                                                2   > XS$ACL|SYS                                                         xs$ace_fk1|sys
                                                                3     < XS$ACL_PARAM|SYS                                                 xs$acl_param_fk2|sys                                                                 4       > XS$POLICY_PARAM|SYS                                            xs$acl_param_fk1|sys
                                                                5         > XS$DSEC|SYS                                                  xs$policy_param_fk1|sys
                                                                6           < XS$ATTR_SEC|SYS                                            xs$attr_sec_fk1|sys                                                                 7             > XS$OBJ|SYS                                               xs$attr_sec_fk2|sys
                                                                8               < XS$ACE_PRIV|SYS*                                       xs$ace_priv_fk2|sys
                                                                8               < XS$ACL|SYS*                                            xs$acl_fk1|sys
                                                                8               < XS$ACL|SYS*                                            xs$acl_fk2|sys
                                                                8               < XS$ACL|SYS*                                            xs$acl_fk3|sys
                                                                8               < XS$AGGR_PRIV|SYS                                       xs$aggr_priv_fk2|sys
                                                                8               < XS$DSEC|SYS*                                           xs$dsec_fk|sys
                                                                8               < XS$INSTSET_ACL|SYS                                     xs$instset_acl_fk2|sys                                                                 9                 > XS$INSTSET_RULE|SYS                                  xs$instset_acl_fk1|sys
                                                               10                   > XS$INSTSET_LIST|SYS                                xs$instset_rule_fk|sys
                                                               11                     > XS$DSEC|SYS*                                     xs$dsec_instset_fk|sys
                                                               11                     < XS$INSTSET_INH|SYS                               xs$instset_inh_fk|sys
                                                               12                       < XS$INSTSET_INH_KEY|SYS                         xs$instset_inh_key_fk|sys
                                                                8               < XS$NSTMPL|SYS                                          xs$nstmpl_fk1|sys                                                                 9                 > XS$ACL|SYS*                                          xs$nstmp1_fk2|sys
                                                                9                 < XS$NSTMPL_ATTR|SYS                                   xs$nstmpl_attr_fk|sys
                                                                8               < XS$PRIN|SYS                                            xs$prin_fk1|sys
                                                                9                 < XS$PROXY_ROLE|SYS                                    xs$proxy_role_fk2|sys                                                                10                   > XS$OBJ|SYS*                                        xs$proxy_role_fk1|sys
                                                                9                 < XS$ROLE_GRANT|SYS                                    xs$role_grant_fk1|sys                                                                10                   > XS$PRIN|SYS*                                       xs$role_grant_fk2|sys
                                                                8               < XS$PRIV|SYS                                            xs$priv_fk1|sys                                                                 9                 > XS$SECCLS|SYS                                        xs$priv_fk2|sys
                                                               10                   > XS$OBJ|SYS*                                        xs$seccls_fk1|sys
                                                               10                   < XS$SECCLS_H|SYS                                    xs$seccls_h_fk1|sys                                                                11                     > XS$OBJ|SYS*                                      xs$seccls_h_fk2|sys
                                                                8               < XS$ROLESET_ROLES|SYS                                   xs$roleset_roles_fk2|sys                                                                 9                 > XS$ROLESET|SYS                                       xs$roleset_roles_fk1|sys
                                                               10                   > XS$OBJ|SYS*                                        xs$roleset_fk|sys
                                                                8               > XS$TENANT|SYS                                          xs$obj_fk|sys

815 rows selected.

Elapsed: 00:00:01.45
</pre>
</div>

#### Network summaries

<div class="scrollbox">
<pre>
Network summary 1 - by network

Network                                     #Links  #Nodes    Max Lev
------------------------------------------ ------- ------- ----------
WWV_FLOW_HNT_ARGUMENT_INFO|APEX_040200           2       2          1
AQ$_INTERNET_AGENTS|SYSTEM                       2       2          1
WWV_FLOW_DICTIONARY_VIEWS|APEX_040200            2       1          1
WWV_FLOW_DATA_LOAD_BAD_LOG|APEX_040200           2       2          1
ATTRIBUTE_TRANSFORMATIONS$|SYS                   2       2          1
WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040200           2       1          1
CLOUD|GSMADMIN_INTERNAL                          2       2          1
CSW_DOMAININFO$|MDSYS                            2       2          1
DAM_CONFIG_PARAM$|SYS                            2       2          1
DBFS$_MOUNTS|SYS                                 2       2          1
DBFS_SFS$_FSTP|SYS                               2       2          1
WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040200          2       1          1
WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040200           2       1          1
DBMS_PARALLEL_EXECUTE_CHUNKS$|SYS                2       2          1
DEPT|SCOTT                                       2       2          1
WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040200       2       1          1
FLIGHTS|TEST                                     2       2          1
WWV_FLOW_RESTRICTED_SCHEMAS|APEX_040200          2       2          1
MVIEW$_ADV_BASETABLE|SYSTEM                      2       2          1
OGIS_GEOMETRY_COLUMNS|MDSYS                      2       2          1
OLS_DIR_BUSINESSES|MDSYS                         2       2          1
WWV_FLOW_MAIL_ATTACHMENTS|APEX_040200            2       2          1
PLANETS|TEST                                     2       2          1
WWV_FLOW_HNT_COLUMN_DICT|APEX_040200             2       2          1
REPCAT$_COLUMN_GROUP|SYSTEM                      2       2          1
REPCAT$_DDL|SYSTEM                               2       2          1
REPCAT$_PRIORITY_GROUP|SYSTEM                    2       2          1
ROADS|TEST                                       2       2          1
SCHEDULER$_JOB_OUTPUT|SYS                        2       2          1
SDO_WS_CONFERENCE_PARTICIPANTS|MDSYS             2       2          1
WRM$_DATABASE_INSTANCE|SYS                       2       2          1
APEX$ARCHIVE_CONTENTS|APEX_040200                2       2          1
WWV_FLOW_FLASH_MAP_FILES|APEX_040200             3       3          1
WWV_FLOW_ADVISOR_CATEGORIES|APEX_040200          3       3          2
BSLN_BASELINES|DBSNMP                            3       3          1
ARCS|TEST                                        3       2          2
PLSQL_PROFILER_DATA|BENCH                        3       3          2
HS$_PARALLEL_HISTOGRAM_DATA|SYS                  4       4          2
DBFS_SFS$_FS|SYS                                 4       4          2
DBMSHP_FUNCTION_INFO|BENCH                       4       3          2
WWV_FLOW_QB_SAVED_COND|APEX_040200               4       4          2
OLS_DIR_CATEGORIES|MDSYS                         4       3          1
WWV_PURGE_DATAFILES|APEX_040200                  5       5          3
APEX$_WS_FILES|APEX_040200                       5       5          2
WWV_FLOW_PKG_APPLICATIONS|APEX_040200            5       3          2
WWV_FLOW_MODELS|APEX_040200                      5       4          3
DR$THS_BT|CTXSYS                                 5       4          2
WWV_FLOW_HNT_COLUMN_INFO|APEX_040200             5       4          3
REGISTRY$DEPENDENCIES|SYS                        6       4          2
REPCAT$_AUDIT_ATTRIBUTE|SYSTEM                   6       6          4
WWV_FLOW_FILE_OBJECTS$|FLOWS_FILES               6       6          2
WFS_FEATUREINSTANCEMETADATA$|MDSYS               7       7          2
DATABASE_POOL_ADMIN|GSMADMIN_INTERNAL            7       5          5
TSDP_ASSOCIATION$|SYS                            9       9          4
BENCH_RUNS|BENCH                                10       9          5
CHANNELS|SH                                     11       8          5
WI$_CAPTURE_FILE|SYS                            11       9          6
HS$_BASE_CAPS|SYS                               12      10          8
WWV_FLOW_BUGS|APEX_040200                       14      11          5
REPCAT$_INSTANTIATION_DDL|SYSTEM                14      13          5
MVIEW$_ADV_AJG|SYSTEM                           15      13          4
DEF$_CALLDEST|SYSTEM                            15      14          6
COUNTRIES|HR                                    22      16         10
OLS$AUDIT|LBACSYS                               22      14          7
CODE$|DVSYS                                     26      20          7
SDO_COORD_AXES|MDSYS                            31      14         12
XS$ACE_PRIV|SYS                                 37      25         12
ORDDCM_ANON_ACTION_TYPES|ORDDATA                70      47         14
WWV_FLOWS|APEX_040200                          335     264         14

69 rows selected.

Elapsed: 00:00:00.20
Network summary 2 - grouped by numbers of nodes

 #Nodes  #Networks
------- ----------
      1          5
      2         28
      3          7
      4          7
      5          3
      6          2
      7          1
      8          1
      9          3
     10          1
     11          1
     13          2
     14          3
     16          1
     20          1
     25          1
     47          1
    264          1

18 rows selected.

Elapsed: 00:00:00.17
</pre>
</div>

### Diagram of a Foreign Key Subnetwork

This diagram shows the trajectory that the algorithm took through the subnetwork of HR tables that includes the COUNTRIES table, with tree links in red, and loop closing links in blue. It may help to understand the working of the algorithm.

<img src="/migrated_images/2015/05/Networks-PLSQL-v1.0-HR.jpg" alt="Networks - PLSQL, v1.0 - HR" title="Networks - PLSQL, v1.0 - HR" />

## Test Network 2: Friendship network of Brightkite users

I took my second, much larger test network from this site: [Friendship network of Brightkite users](https://snap.stanford.edu/data/loc-brightkite.html)

The page describes the network as having 58,228 nodes and 214,078 links, but the data set has 428,156 links, with each pair of nodes that is linked having links provided in both directions. My package traverses links in either direction so I did not require the second link, and copied only one of the links into a table for the network analysis. I created primary key and indexes that I deemed appropriate and gathered statistics on the table.

### Output for Brightkite Network

The detailed output with 214,625 records took 103 seconds, while the two summary outputs took 27 and 23 seconds. Most of the detailed execution time is of course due to the writing of the records to file. The output is too large to embed in full, so I cut out most of the detailed output.

<div class="scrollbox">
<pre>
links_v based on Net_Brightkite

View dropped.

View created.

Network detail

Network     #Links  #Nodes    Lev Node                                                                   Link
---------- ------- ------- ------ ---------------------------------------------------------------------- ----------
0           212945   56739      0 0                                                                      ROOT
                                1 > 1                                                                    135
                                2   > 123                                                                955
                                3     < 11                                                               961
                                4       < 0*                                                             15                                 4       > 124                                                            974
                                5         < 1*                                                           969
                                5         < 123*                                                         982                                 5         > 125                                                          988
                                6           < 1*                                                         983
                                6           < 11*                                                        986                                 6           > 127                                                        993
                                7             < 1*                                                       990
                                7             < 11*                                                      992
                                7             < 5                                                        991
                                8               < 0*                                                     142
                                8               < 1*                                                     143                                 8               > 11*                                                    18
                                8               > 123*                                                   958
                                8               > 124*                                                   972
                                8               > 125*                                                   984
                                8               > 129                                                    996
                                9                 < 1*                                                   995
                                9                 < 125*                                                 999                                 9                 > 131                                                  1010
                               10                   < 1*                                                 1007                                10                   > 133                                                1024
                               11                     < 1*                                               1018
                               11                     < 129*                                             1023                                11                     > 2915                                             15317
                               12                       > 10857                                          77629
                               13                         < 1446                                         77626                                14                           > 10334                                      92481
                               15                             < 10198                                    92510                                16                               > 10332                                  91634
                               17                                 > 10333                                92473
                               18                                   > 10336                              92550
                               19                                     < 10331                            92548                                20                                       > 10338                          92638
                               21                                         < 10332*                       92639                                21                                         > 10342                        93374
                               22                                           < 10336*                     93373                                22                                           > 10344                      93470
                               23                                             < 10331*                   93469                                23                                             > 10345                    93502
                               24                                               < 10331*                 93499
                               24                                               < 10338*                 93500
                               24                                               < 10342*                 93501                                24                                               > 10349                  95200
                               25                                                 < 10331*               95191
                               25                                                 < 10332*               95192
                               25                                                 < 10333*               95193
                               25                                                 < 10334*               95194
                               25                                                 < 10336*               95195
                               25                                                 < 10337                95196
                               26                                                   < 10336*             92600                                26                                                   > 10340              93311
                               27                                                     < 10336*           93310                                27                                                     > 10346            94326
                               28                                                       < 10200          94320
                               29                                                         < 2812         89521                                30                                                           > 10198*     89503
                               30                                                           > 10332*     91600
                               30                                                           > 10333*     91656
.
. (extracted for brevity)
.
				
				7             > 871                                                      5496
                                8               < 11*                                                    5495                                 7             > 873                                                      5499
                                8               < 11*                                                    5498                                 6           > 6553                                                       54725
                                5         > 6548                                                         54712
                                5         > 6550                                                         54718
                                4       > 859                                                            5471
                                4       > 862                                                            5477
                                4       > 869                                                            5493
                                4       > 870                                                            5494
                                4       > 872                                                            5497
                                3     > 6535                                                             53954
                                4       > 34947                                                          140148
                                3     > 6536                                                             53955
                                3     > 6537                                                             54701
                                3     > 6538                                                             54702
                                3     > 6539                                                             54703
                                3     > 6540                                                             54704
                                3     > 6541                                                             54705
                                3     > 6543                                                             54706
                                3     > 6545                                                             54708
                                3     > 6546                                                             54709
                                2   > 126                                                                989
                                3     > 141                                                              1100
                                4       < 1*                                                             1099                                 2   > 128                                                                994
                                1 > 107                                                                  673
                                1 > 73                                                                   432
10020            1       2      0 10020                                                                  ROOT
                                1 > 40400                                                                206894
10061            3       4      0 10061                                                                  ROOT
                                1 > 40442                                                                207014
                                1 > 40443                                                                207015
                                2   > 54793                                                              197339
10454            1       2      0 10454                                                                  ROOT
                                1 > 40962                                                                211394
10541            1       2      0 10541                                                                  ROOT
                                1 > 41084                                                                211598
10569            1       2      0 10569                                                                  ROOT
                                1 > 41147                                                                211688
10572            1       2      0 10572                                                                  ROOT
                                1 > 41148                                                                211689
11030            2       3      0 11030                                                                  ROOT
                                1 > 41526                                                                189843
                                1 > 41527                                                                189844
11053            2       3      0 11053                                                                  ROOT
                                1 > 17537                                                                124571
                                1 > 41557                                                                189900
11136            1       2      0 11136                                                                  ROOT
                                1 > 27830                                                                172081
11615            3       3      0 11615                                                                  ROOT
                                1 > 42164                                                                194725
                                2   > 42165                                                              194727
                                3     < 11615*                                                           194726 11628           13      10      0 11628                                                                  ROOT                                 1 > 26719                                                                182224
                                2   > 42195                                                              194766
                                3     < 11628*                                                           194765
                                3     < 42193                                                            194767
                                4       < 11628*                                                         194762                                 4       > 42194                                                          194764
                                5         < 11628*                                                       194763                                 5         > 42195*                                                       194768
                                1 > 42188                                                                194757
                                1 > 42189                                                                194758
                                1 > 42190                                                                194759
                                1 > 42191                                                                194760
                                1 > 42192                                                                194761
11686            1       2      0 11686                                                                  ROOT
                                1 > 42284                                                                196157
11687            5       5      0 11687                                                                  ROOT
                                1 > 13207                                                                79434
                                2   > 43916                                                              210413
                                3     > 43917                                                            210415
                                4       < 13207*                                                         210414
                                2   < 7637                                                               79433 11713            1       2      0 11713                                                                  ROOT                                 1 > 40482                                                                208266
11770            1       2      0 11770                                                                  ROOT
                                1 > 13304                                                                80582
11778            2       3      0 11778                                                                  ROOT
                                1 > 40832                                                                210039
                                1 > 42426                                                                197700
11802            1       2      0 11802                                                                  ROOT
                                1 > 27316                                                                165028
11831            1       2      0 11831                                                                  ROOT
                                1 > 26846                                                                182516
11945            4       4      0 11945                                                                  ROOT
                                1 > 13842                                                                134375
                                2   > 42671                                                              199448
                                3     < 11945*                                                           199447                                 2   > 42672                                                              199449
11982            3       4      0 11982                                                                  ROOT
                                1 > 42736                                                                200800
                                2   > 55250                                                              199061
                                2   > 55251                                                              199062
12004            1       2      0 12004                                                                  ROOT
                                1 > 42778                                                                200897
12063            4       4      0 12063                                                                  ROOT
                                1 > 42874                                                                202418
                                2   > 42876                                                              202421
                                3     < 12063*                                                           202420                                 1 > 42875                                                                202419
12095            1       2      0 12095                                                                  ROOT
                                1 > 42933                                                                202516
12115            2       3      0 12115                                                                  ROOT
                                1 > 42964                                                                202586
                                2   > 53624                                                              191170
12145            2       3      0 12145                                                                  ROOT
                                1 > 42999                                                                203879
                                1 > 43000                                                                203880
12592            1       2      0 12592                                                                  ROOT
                                1 > 17004                                                                135534
12593            1       2      0 12593                                                                  ROOT
                                1 > 43361                                                                205795
12601            1       2      0 12601                                                                  ROOT
                                1 > 43367                                                                205802
12709            1       2      0 12709                                                                  ROOT
                                1 > 43438                                                                207113
12780            1       2      0 12780                                                                  ROOT
                                1 < 361                                                                  70125 12781            7       5      0 12781                                                                  ROOT                                 1 > 43503                                                                207232
                                2   > 43504                                                              207234
                                3     < 12781*                                                           207233                                 3     > 55481                                                            185736
                                4       < 43503*                                                         185735                                 3     > 55482                                                            185738
                                4       < 43503*                                                         185737 13125            2       3      0 13125                                                                  ROOT                                 1 > 41059                                                                211552
                                2   > 54899                                                              197454
13150            1       2      0 13150                                                                  ROOT
                                1 > 43850                                                                210327
13152            2       3      0 13152                                                                  ROOT
                                1 > 13412                                                                65309
                                1 > 43851                                                                210328
13188            3       4      0 13188                                                                  ROOT
                                1 > 13197                                                                79386
                                1 > 43895                                                                210381
                                2   > 55583                                                              185879
13237            1       2      0 13237                                                                  ROOT
                                1 > 43973                                                                210493
13277            1       2      0 13277                                                                  ROOT
                                1 > 13284                                                                80519
13321            5       6      0 13321                                                                  ROOT
                                1 > 13563                                                                68463
                                2   > 44299                                                              213314
                                2   > 44300                                                              213315
                                2   > 44301                                                              213316
                                1 > 41602                                                                189980
13409            1       2      0 13409                                                                  ROOT
                                1 > 36374                                                                151029
13423            1       2      0 13423                                                                  ROOT
                                1 > 44127                                                                211897
13441            3       4      0 13441                                                                  ROOT
                                1 > 44159                                                                211938
                                1 > 44160                                                                211939
                                1 > 44161                                                                211940
13496           10       8      0 13496                                                                  ROOT
                                1 > 44228                                                                212038
                                2   > 55671                                                              185988
                                3     > 55672                                                            185990
                                4       < 44228*                                                         185989                                 2   > 55673                                                              185991
                                2   > 55674                                                              185992
                                3     > 55675                                                            185994
                                4       < 44228*                                                         185993                                 4       > 55676                                                          185996
                                5         < 44228*                                                       185995 13508            1       2      0 13508                                                                  ROOT                                 1 > 44235                                                                212045
13516            1       2      0 13516                                                                  ROOT
                                1 > 44241                                                                212052
13519            1       2      0 13519                                                                  ROOT
                                1 > 44247                                                                212058
13521            1       2      0 13521                                                                  ROOT
                                1 > 44248                                                                212059
13542            3       4      0 13542                                                                  ROOT
                                1 > 44268                                                                212093
                                2   > 55684                                                              186004
                                2   > 55685                                                              186005
13544            2       3      0 13544                                                                  ROOT
                                1 > 44275                                                                212105
                                2   > 55687                                                              186007
13608            2       3      0 13608                                                                  ROOT
                                1 > 44352                                                                213393
                                1 > 44353                                                                213394
13708            2       3      0 13708                                                                  ROOT
                                1 > 35547                                                                144721
                                2   > 53642                                                              191189
13813            5       4      0 13813                                                                  ROOT
                                1 > 44466                                                                213572
                                2   > 44467                                                              213574
                                3     < 13813*                                                           213573                                 3     > 44468                                                            213576
                                4       < 44466*                                                         213575 13846            3       3      0 13846                                                                  ROOT                                 1 > 44528                                                                213677
                                2   > 44529                                                              213679
                                3     < 13846*                                                           213678 1469             2       3      0 1469                                                                   ROOT                                 1 > 20244                                                                110822
                                2   > 48178                                                              213748
14692            1       2      0 14692                                                                  ROOT
                                1 > 45193                                                                194940
15577            1       2      0 15577                                                                  ROOT
                                1 > 45714                                                                198098
15579            3       3      0 15579                                                                  ROOT
                                1 > 45717                                                                198103
                                2   > 45718                                                              198105
                                3     < 15579*                                                           198104 15636            1       2      0 15636                                                                  ROOT                                 1 > 45766                                                                198188
15658            2       3      0 15658                                                                  ROOT
                                1 > 45775                                                                198198
                                2   > 56016                                                              186375
15663            2       3      0 15663                                                                  ROOT
                                1 > 28650                                                                181215
                                1 > 45779                                                                198205
15670            1       2      0 15670                                                                  ROOT
                                1 > 45785                                                                198213
15675            6       6      0 15675                                                                  ROOT
                                1 > 15678                                                                136346
                                2   > 37514                                                              158682
                                2   > 45788                                                              198217
                                3     < 15675*                                                           198216                                 1 > 45787                                                                198215
                                1 > 45789                                                                198218
15689            1       2      0 15689                                                                  ROOT
                                1 > 45800                                                                198234
16399            2       3      0 16399                                                                  ROOT
                                1 > 46195                                                                201208
                                1 > 46196                                                                201209
16525            2       3      0 16525                                                                  ROOT
                                1 > 46293                                                                201380
                                1 > 46294                                                                201381
16526            2       3      0 16526                                                                  ROOT
                                1 > 46295                                                                201382
                                1 > 46296                                                                201383
16794            1       2      0 16794                                                                  ROOT
                                1 > 46442                                                                202723
16825            2       3      0 16825                                                                  ROOT
                                1 > 16826                                                                134322
                                2   > 46457                                                              202742
16827            1       2      0 16827                                                                  ROOT
                                1 > 46458                                                                202743
17012            1       2      0 17012                                                                  ROOT
                                1 > 46469                                                                202756
1704             1       2      0 1704                                                                   ROOT
                                1 > 21114                                                                102970
17047            1       2      0 17047                                                                  ROOT
                                1 > 36999                                                                155474
17062            1       2      0 17062                                                                  ROOT
                                1 > 17065                                                                135663
17074            1       2      0 17074                                                                  ROOT
                                1 > 46518                                                                202834
17084            3       4      0 17084                                                                  ROOT
                                1 > 46523                                                                202843
                                2   > 56157                                                              186530
                                2   > 56158                                                              186531
17256            2       3      0 17256                                                                  ROOT
                                1 > 46647                                                                204245
                                1 < 813                                                                  120656 17287            1       2      0 17287                                                                  ROOT                                 1 > 26319                                                                177029
1731             1       2      0 1731                                                                   ROOT
                                1 > 21183                                                                103185
17401            2       3      0 17401                                                                  ROOT
                                1 > 46765                                                                204433
                                1 < 850                                                                  123169 17459            2       3      0 17459                                                                  ROOT                                 1 > 46784                                                                204462
                                1 > 46785                                                                204463
18379            1       2      0 18379                                                                  ROOT
                                1 > 47262                                                                207545
18420            6       6      0 18420                                                                  ROOT
                                1 > 47291                                                                207578
                                2   < 47290                                                              207579                                 2   > 56269                                                              186649
                                1 > 47292                                                                207580
                                2   > 47293                                                              207582
                                3     < 18420*                                                           207581 18476            3       3      0 18476                                                                  ROOT                                 1 > 47311                                                                207604
                                2   > 47312                                                              207606
                                3     < 18476*                                                           207605 18776            1       2      0 18776                                                                  ROOT                                 1 > 47479                                                                209033
19029            1       2      0 19029                                                                  ROOT
                                1 > 47589                                                                209189
19907            2       3      0 19907                                                                  ROOT
                                1 > 47951                                                                210876
                                2   > 56377                                                              186770
19925            1       2      0 19925                                                                  ROOT
                                1 > 47953                                                                210878
20119            5       6      0 20119                                                                  ROOT
                                1 > 48086                                                                212360
                                2   > 56414                                                              186815
                                2   > 56415                                                              186816
                                2   > 56416                                                              186817
                                1 > 48087                                                                212361
20137            1       2      0 20137                                                                  ROOT
                                1 > 48100                                                                212382
20143            2       3      0 20143                                                                  ROOT
                                1 > 48108                                                                212391
                                1 > 48109                                                                212392
20149            1       2      0 20149                                                                  ROOT
                                1 > 48120                                                                212404
20150            1       2      0 20150                                                                  ROOT
                                1 > 48121                                                                212405
21785            1       2      0 21785                                                                  ROOT
                                1 > 48786                                                                190587
21795            1       2      0 21795                                                                  ROOT
                                1 > 48787                                                                190588
21823            5       4      0 21823                                                                  ROOT
                                1 > 48792                                                                190594
                                2   > 48793                                                              190596
                                3     < 21823*                                                           190595                                 3     > 56528                                                            186954
                                4       < 48792*                                                         186953 22211            1       2      0 22211                                                                  ROOT                                 1 > 48904                                                                190750
22386            3       4      0 22386                                                                  ROOT
                                1 > 48959                                                                190818
                                1 > 48960                                                                190819
                                1 > 48961                                                                190820
22850            1       2      0 22850                                                                  ROOT
                                1 > 49168                                                                192303
22949            4       4      0 22949                                                                  ROOT
                                1 > 49200                                                                192352
                                2   > 49201                                                              192354
                                3     < 22949*                                                           192353                                 2   > 56579                                                              187007
24435            1       2      0 24435                                                                  ROOT
                                1 > 49761                                                                195562
24477            1       2      0 24477                                                                  ROOT
                                1 > 49778                                                                195580
24642            1       2      0 24642                                                                  ROOT
                                1 > 49852                                                                196899
24937            5       5      0 24937                                                                  ROOT
                                1 > 50011                                                                197113
                                1 > 50012                                                                197114
                                2   > 50014                                                              197117
                                3     < 24937*                                                           197116                                 1 > 50013                                                                197115
26072            1       2      0 26072                                                                  ROOT
                                1 > 50470                                                                200114
26403            1       2      0 26403                                                                  ROOT
                                1 > 50578                                                                200255
26418            1       2      0 26418                                                                  ROOT
                                1 > 50583                                                                200262
26460            9       7      0 26460                                                                  ROOT
                                1 > 27747                                                                170819
                                1 > 50605                                                                200286
                                2   > 50609                                                              200292
                                3     < 26460*                                                           200291
                                3     < 50607                                                            200293
                                4       < 26460*                                                         200288                                 4       > 50608                                                          200290
                                5         < 26460*                                                       200289                                 1 > 50606                                                                200287
26630            3       4      0 26630                                                                  ROOT
                                1 > 50671                                                                201521
                                2   > 56815                                                              187279
                                2   > 56816                                                              187280
26641            1       2      0 26641                                                                  ROOT
                                1 > 50678                                                                201530
26744            1       2      0 26744                                                                  ROOT
                                1 > 50724                                                                201604
26766            1       2      0 26766                                                                  ROOT
                                1 > 50730                                                                201609
26803            1       2      0 26803                                                                  ROOT
                                1 > 50743                                                                201625
26977            1       2      0 26977                                                                  ROOT
                                1 > 50775                                                                201661
26979            1       2      0 26979                                                                  ROOT
                                1 > 50776                                                                201662
27113            1       2      0 27113                                                                  ROOT
                                1 > 50817                                                                201717
27134            1       2      0 27134                                                                  ROOT
                                1 > 50824                                                                201725
27187            4       5      0 27187                                                                  ROOT
                                1 > 36938                                                                154207
                                2   < 7559                                                               154206                                 3     > 36936                                                            154204
                                3     > 36937                                                            154205
27336            2       3      0 27336                                                                  ROOT
                                1 > 50892                                                                201822
                                2   > 56861                                                              187325
27342            1       2      0 27342                                                                  ROOT
                                1 > 50899                                                                201832
27442            1       2      0 27442                                                                  ROOT
                                1 > 50940                                                                201889
27481            1       2      0 27481                                                                  ROOT
                                1 > 50947                                                                203034
27487            1       2      0 27487                                                                  ROOT
                                1 > 50949                                                                203036
27490            5       5      0 27490                                                                  ROOT
                                1 > 50951                                                                203038
                                2   > 56878                                                              187346
                                3     > 56880                                                            187349
                                4       < 50951*                                                         187348                                 2   > 56879                                                              187347
27542            4       4      0 27542                                                                  ROOT
                                1 > 50980                                                                203083
                                2   > 50981                                                              203085
                                3     < 27542*                                                           203084                                 2   > 56893                                                              187362
27545            1       2      0 27545                                                                  ROOT
                                1 > 50982                                                                203086
27548            1       2      0 27548                                                                  ROOT
                                1 > 50987                                                                203094
27572            6       7      0 27572                                                                  ROOT
                                1 > 51006                                                                203118
                                1 > 51007                                                                203119
                                1 > 51008                                                                203120
                                1 > 51009                                                                203121
                                2   > 56898                                                              187367
                                1 > 51010                                                                203122
27621           25      10      0 27621                                                                  ROOT
                                1 > 51047                                                                203177
                                2   > 56915                                                              187385
                                3     > 56917                                                            187388
                                4       < 51047*                                                         187387
                                4       < 56916                                                          187389
                                5         < 51047*                                                       187386                                 5         > 56918                                                        187392
                                6           < 51047*                                                     187390
                                6           < 56915*                                                     187391
                                6           < 56917*                                                     187393                                 6           > 56919                                                      187397
                                7             < 51047*                                                   187394
                                7             < 56915*                                                   187395
                                7             < 56917*                                                   187396                                 7             > 56920                                                    187402
                                8               < 51047*                                                 187398
                                8               < 56915*                                                 187399
                                8               < 56917*                                                 187400
                                8               < 56918*                                                 187401                                 8               > 56921                                                  187407
                                9                 < 51047*                                               187403
                                9                 < 56915*                                               187404
                                9                 < 56917*                                               187405
                                9                 < 56918*                                               187406                                 2   > 56922                                                              187408
27737            1       2      0 27737                                                                  ROOT
                                1 > 51091                                                                203240
27761            3       3      0 27761                                                                  ROOT
                                1 > 51107                                                                203257
                                2   > 51108                                                              203259
                                3     < 27761*                                                           203258 27763            1       2      0 27763                                                                  ROOT                                 1 > 51109                                                                203260
27765            1       2      0 27765                                                                  ROOT
                                1 > 51110                                                                203261
27771            2       3      0 27771                                                                  ROOT
                                1 > 51111                                                                203262
                                1 < 9603                                                                 171954 27788            1       2      0 27788                                                                  ROOT                                 1 > 51119                                                                203271
27789            1       2      0 27789                                                                  ROOT
                                1 > 51120                                                                203272
27790            1       2      0 27790                                                                  ROOT
                                1 > 51121                                                                203273
27831            1       2      0 27831                                                                  ROOT
                                1 > 27926                                                                173361
27838            1       2      0 27838                                                                  ROOT
                                1 > 51143                                                                203302
27844            2       3      0 27844                                                                  ROOT
                                1 > 51147                                                                203315
                                1 > 51148                                                                203316
27880            1       2      0 27880                                                                  ROOT
                                1 > 51152                                                                203320
27882            1       2      0 27882                                                                  ROOT
                                1 > 51153                                                                203321
27888            1       2      0 27888                                                                  ROOT
                                1 > 51155                                                                203323
27894            1       2      0 27894                                                                  ROOT
                                1 > 51156                                                                203324
27897            1       2      0 27897                                                                  ROOT
                                1 > 51158                                                                203327
27915            1       2      0 27915                                                                  ROOT
                                1 > 51165                                                                203337
27933            2       3      0 27933                                                                  ROOT
                                1 > 51167                                                                203339
                                1 > 51168                                                                203340
27943            1       2      0 27943                                                                  ROOT
                                1 > 51170                                                                203342
27960            1       2      0 27960                                                                  ROOT
                                1 > 51178                                                                203355
27988            6       7      0 27988                                                                  ROOT
                                1 < 3107                                                                 173478                                 2   > 27989                                                              173479
                                2   > 9506                                                               90402
                                1 > 51194                                                                203374
                                2   > 56955                                                              187443
                                2   > 56956                                                              187444
28061            1       2      0 28061                                                                  ROOT
                                1 > 51236                                                                204652
28073            1       2      0 28073                                                                  ROOT
                                1 > 51238                                                                204654
28125            1       2      0 28125                                                                  ROOT
                                1 > 51261                                                                204696
28145            1       2      0 28145                                                                  ROOT
                                1 > 41990                                                                193070
28192            1       2      0 28192                                                                  ROOT
                                1 > 51297                                                                204757
28211            1       2      0 28211                                                                  ROOT
                                1 > 51301                                                                204763
28297            2       3      0 28297                                                                  ROOT
                                1 > 51313                                                                204777
                                1 > 51314                                                                204778
28430            1       2      0 28430                                                                  ROOT
                                1 > 51349                                                                204819
28517            1       2      0 28517                                                                  ROOT
                                1 > 51388                                                                204865
28518            3       4      0 28518                                                                  ROOT
                                1 > 51389                                                                204866
                                1 > 51390                                                                204867
                                1 > 51391                                                                204868
28522            3       4      0 28522                                                                  ROOT
                                1 > 51394                                                                204871
                                1 > 51395                                                                204872
                                2   > 56981                                                              187470
28523            3       4      0 28523                                                                  ROOT
                                1 > 51396                                                                204873
                                2   > 56982                                                              187471
                                1 > 51397                                                                204874
28527            2       3      0 28527                                                                  ROOT
                                1 > 36758                                                                153905
                                1 > 37951                                                                162947
28531            1       2      0 28531                                                                  ROOT
                                1 > 51401                                                                204878
28547            1       2      0 28547                                                                  ROOT
                                1 > 51409                                                                204885
28548            1       2      0 28548                                                                  ROOT
                                1 > 51410                                                                204886
28551            1       2      0 28551                                                                  ROOT
                                1 > 51413                                                                204888
28557            1       2      0 28557                                                                  ROOT
                                1 > 51420                                                                204907
28560            1       2      0 28560                                                                  ROOT
                                1 > 51421                                                                204908
28607            2       3      0 28607                                                                  ROOT
                                1 > 51446                                                                204945
                                1 > 51447                                                                204946
28826            1       2      0 28826                                                                  ROOT
                                1 > 51519                                                                206265
28842            1       2      0 28842                                                                  ROOT
                                1 > 51532                                                                206279
28891            1       2      0 28891                                                                  ROOT
                                1 > 51562                                                                206318
28894            2       3      0 28894                                                                  ROOT
                                1 > 51563                                                                206319
                                1 > 51564                                                                206320
29024            5       6      0 29024                                                                  ROOT
                                1 > 37722                                                                160197
                                2   > 37909                                                              161730
                                2   > 54205                                                              194231
                                2   > 54206                                                              194232
                                2   < 8038                                                               160196 29117            1       2      0 29117                                                                  ROOT                                 1 > 51629                                                                206405
29341            3       3      0 29341                                                                  ROOT
                                1 > 29342                                                                171035
                                2   < 3403                                                               171034                                 3     > 29341*                                                           171033
29759            1       2      0 29759                                                                  ROOT
                                1 > 51871                                                                207897
29850            1       2      0 29850                                                                  ROOT
                                1 > 51905                                                                207939
29851            1       2      0 29851                                                                  ROOT
                                1 > 50614                                                                200297
29852            1       2      0 29852                                                                  ROOT
                                1 > 51906                                                                207940
29853            3       4      0 29853                                                                  ROOT
                                1 > 51907                                                                207941
                                1 > 51908                                                                207942
                                1 > 51909                                                                207943
29858            3       3      0 29858                                                                  ROOT
                                1 > 51911                                                                207947
                                2   > 51912                                                              207949
                                3     < 29858*                                                           207948 29867            1       2      0 29867                                                                  ROOT                                 1 > 51916                                                                207953
29876            1       2      0 29876                                                                  ROOT
                                1 > 51925                                                                207961
29893            1       2      0 29893                                                                  ROOT
                                1 > 51931                                                                207969
29895            1       2      0 29895                                                                  ROOT
                                1 > 51932                                                                207970
29898            5       5      0 29898                                                                  ROOT
                                1 > 51934                                                                207971
                                2   < 51933                                                              207972                                 3     > 57071                                                            187563
                                3     > 57072                                                            187564
                                4       < 51934*                                                         187565 29901            2       3      0 29901                                                                  ROOT                                 1 > 35577                                                                144761
                                1 > 51935                                                                207973
29902            4       4      0 29902                                                                  ROOT
                                1 > 51936                                                                207974
                                2   > 51937                                                              207976
                                3     < 29902*                                                           207975                                 3     > 57075                                                            187566
29916            1       2      0 29916                                                                  ROOT
                                1 > 51945                                                                207985
30075            1       2      0 30075                                                                  ROOT
                                1 > 51979                                                                208026
30473            1       2      0 30473                                                                  ROOT
                                1 > 52103                                                                208181
31960            2       3      0 31960                                                                  ROOT
                                1 > 52531                                                                211063
                                2   > 57190                                                              187690
32410            2       3      0 32410                                                                  ROOT
                                1 > 52720                                                                211344
                                2   > 57227                                                              187729
33215            2       3      0 33215                                                                  ROOT
                                1 > 52960                                                                212795
                                1 > 52961                                                                212796
34215            1       2      0 34215                                                                  ROOT
                                1 > 53231                                                                189451
34428            3       4      0 34428                                                                  ROOT
                                1 > 53291                                                                189528
                                2   > 57316                                                              187824
                                1 > 53292                                                                189529
35114            1       2      0 35114                                                                  ROOT
                                1 > 53503                                                                191044
35129            1       2      0 35129                                                                  ROOT
                                1 > 53515                                                                191053
35189            3       4      0 35189                                                                  ROOT
                                1 > 53524                                                                191064
                                2   > 57350                                                              187859
                                1 > 53525                                                                191065
35196            1       2      0 35196                                                                  ROOT
                                1 > 53527                                                                191067
35283            2       3      0 35283                                                                  ROOT
                                1 > 53549                                                                191091
                                1 > 53550                                                                191092
35299            1       2      0 35299                                                                  ROOT
                                1 > 53561                                                                191104
35302            1       2      0 35302                                                                  ROOT
                                1 > 53563                                                                191106
35326            1       2      0 35326                                                                  ROOT
                                1 > 53568                                                                191111
35378            2       3      0 35378                                                                  ROOT
                                1 > 53594                                                                191142
                                1 > 53595                                                                191143
35381            1       2      0 35381                                                                  ROOT
                                1 > 53596                                                                191144
35394            1       2      0 35394                                                                  ROOT
                                1 > 53601                                                                191149
35440            2       3      0 35440                                                                  ROOT
                                1 > 44249                                                                212060
                                2   < 37918                                                              212061 35456            1       2      0 35456                                                                  ROOT                                 1 > 53613                                                                191162
35464            1       2      0 35464                                                                  ROOT
                                1 > 53615                                                                191164
35474            1       2      0 35474                                                                  ROOT
                                1 > 53617                                                                191166
35481            1       2      0 35481                                                                  ROOT
                                1 > 38326                                                                142051
35484            2       3      0 35484                                                                  ROOT
                                1 > 53625                                                                191171
                                1 > 53626                                                                191172
35540            2       3      0 35540                                                                  ROOT
                                1 > 53639                                                                191186
                                1 > 53640                                                                191187
35558            1       2      0 35558                                                                  ROOT
                                1 > 53645                                                                191192
35581            2       3      0 35581                                                                  ROOT
                                1 > 53653                                                                191200
                                1 > 53654                                                                191201
35608            1       2      0 35608                                                                  ROOT
                                1 > 53660                                                                191208
35609            1       2      0 35609                                                                  ROOT
                                1 > 53661                                                                191209
35661            4       5      0 35661                                                                  ROOT
                                1 > 53671                                                                191220
                                2   > 57372                                                              187880
                                3     > 58030                                                            188637
                                1 > 53672                                                                191221
35748            1       2      0 35748                                                                  ROOT
                                1 > 53684                                                                191233
35751            1       2      0 35751                                                                  ROOT
                                1 > 53687                                                                191235
35755            1       2      0 35755                                                                  ROOT
                                1 > 53688                                                                191236
35759            1       2      0 35759                                                                  ROOT
                                1 > 46206                                                                201221
35763            1       2      0 35763                                                                  ROOT
                                1 > 53689                                                                191237
35766            1       2      0 35766                                                                  ROOT
                                1 > 49404                                                                193820
35767            1       2      0 35767                                                                  ROOT
                                1 > 53690                                                                191238
35769            1       2      0 35769                                                                  ROOT
                                1 > 53692                                                                191241
35772            2       3      0 35772                                                                  ROOT
                                1 > 53696                                                                191246
                                1 > 53697                                                                191247
35781            2       3      0 35781                                                                  ROOT
                                1 > 53700                                                                191250
                                2   > 57376                                                              187884
35784            1       2      0 35784                                                                  ROOT
                                1 > 53701                                                                191251
35790           10      11      0 35790                                                                  ROOT
                                1 > 35792                                                                146323
                                2   < 35791                                                              146324                                 3     > 43770                                                            208945
                                2   > 35793                                                              146325
                                3     > 43773                                                            208948
                                2   > 43772                                                              208947
                                3     > 55557                                                            185848
                                1 > 43771                                                                208946
                                1 > 53704                                                                191254
                                1 > 53705                                                                191255
35803            1       2      0 35803                                                                  ROOT
                                1 < 6945                                                                 146340 35938            2       3      0 35938                                                                  ROOT                                 1 > 53740                                                                192495
                                1 > 53741                                                                192496
35953            1       2      0 35953                                                                  ROOT
                                1 < 7020                                                                 147809 36004            1       2      0 36004                                                                  ROOT                                 1 > 53760                                                                192518
36006            4       5      0 36006                                                                  ROOT
                                1 > 53763                                                                192521
                                2   > 57387                                                              187895
                                1 > 53764                                                                192522
                                1 > 53765                                                                192523
36008            1       2      0 36008                                                                  ROOT
                                1 > 53767                                                                192526
36022            1       2      0 36022                                                                  ROOT
                                1 > 53777                                                                192538
36031            3       4      0 36031                                                                  ROOT
                                1 > 53784                                                                192545
                                2   > 57389                                                              187897
                                2   > 57390                                                              187898
36033            1       2      0 36033                                                                  ROOT
                                1 > 53787                                                                192548
36185            5       5      0 36185                                                                  ROOT
                                1 > 53833                                                                192606
                                2   > 57399                                                              187907
                                3     > 57400                                                            187909
                                4       < 53833*                                                         187908                                 3     > 58037                                                            188643
36253            3       4      0 36253                                                                  ROOT
                                1 > 53846                                                                192620
                                2   > 57404                                                              187916
                                1 > 53847                                                                192621
36272            1       2      0 36272                                                                  ROOT
                                1 > 44014                                                                210540
36328            3       4      0 36328                                                                  ROOT
                                1 > 53867                                                                192645
                                1 > 53868                                                                192646
                                1 > 53869                                                                192647
36375            1       2      0 36375                                                                  ROOT
                                1 > 53880                                                                192659
36378            1       2      0 36378                                                                  ROOT
                                1 > 53881                                                                192660
36382            2       3      0 36382                                                                  ROOT
                                1 > 53883                                                                192662
                                2   < 36383                                                              192663 36385            1       2      0 36385                                                                  ROOT                                 1 > 53884                                                                192664
36399            1       2      0 36399                                                                  ROOT
                                1 > 53487                                                                191024
36405            1       2      0 36405                                                                  ROOT
                                1 < 7264                                                                 151063 36533            1       2      0 36533                                                                  ROOT                                 1 > 53908                                                                192694
36612            1       2      0 36612                                                                  ROOT
                                1 > 53923                                                                192711
36652            2       3      0 36652                                                                  ROOT
                                1 > 53937                                                                192728
                                2   > 57419                                                              187929
36821            1       2      0 36821                                                                  ROOT
                                1 > 53975                                                                192770
36919            1       2      0 36919                                                                  ROOT
                                1 > 53988                                                                192783
36965            4       4      0 36965                                                                  ROOT
                                1 > 41327                                                                213153
                                2   < 41326                                                              213154                                 2   > 41328                                                              213156
                                3     < 36965*                                                           213155 36981            1       2      0 36981                                                                  ROOT                                 1 > 54010                                                                192809
37044            2       3      0 37044                                                                  ROOT
                                1 > 54025                                                                192829
                                2   > 57438                                                              187950
37084            1       2      0 37084                                                                  ROOT
                                1 > 54037                                                                192843
37108            1       2      0 37108                                                                  ROOT
                                1 < 7638                                                                 155669 37247            1       2      0 37247                                                                  ROOT                                 1 > 54073                                                                194065
37270            4       4      0 37270                                                                  ROOT
                                1 > 54079                                                                194071
                                2   > 54080                                                              194073
                                3     < 37270*                                                           194072                                 2   > 57445                                                              187957
37277            1       2      0 37277                                                                  ROOT
                                1 > 54081                                                                194074
37342           10       6      0 37342                                                                  ROOT
                                1 > 54091                                                                194084
                                2   > 54092                                                              194086
                                3     < 37342*                                                           194085                                 3     > 54093                                                            194089
                                4       < 37342*                                                         194087
                                4       < 54091*                                                         194088                                 2   > 54094                                                              194091
                                3     < 37342*                                                           194090                                 2   > 54095                                                              194093
                                3     < 37342*                                                           194092 37346            1       2      0 37346                                                                  ROOT                                 1 > 54097                                                                194095
37348            1       2      0 37348                                                                  ROOT
                                1 < 7775                                                                 157227
37351            1       2      0 37351                                                                  ROOT
                                1 < 7777                                                                 157234 37420            1       2      0 37420                                                                  ROOT                                 1 > 54126                                                                194139
37421            1       2      0 37421                                                                  ROOT
                                1 > 54127                                                                194140
37515            1       2      0 37515                                                                  ROOT
                                1 > 54145                                                                194159
37516            1       2      0 37516                                                                  ROOT
                                1 > 54146                                                                194160
37585            1       2      0 37585                                                                  ROOT
                                1 > 54165                                                                194185
37689            1       2      0 37689                                                                  ROOT
                                1 < 8026                                                                 160140 37715            1       2      0 37715                                                                  ROOT                                 1 > 53607                                                                191155
37916            1       2      0 37916                                                                  ROOT
                                1 > 54252                                                                194287
37953            1       2      0 37953                                                                  ROOT
                                1 > 53560                                                                191103
38033            1       2      0 38033                                                                  ROOT
                                1 < 8218                                                                 163161 38066            1       2      0 38066                                                                  ROOT                                 1 > 54301                                                                194354
38281            1       2      0 38281                                                                  ROOT
                                1 < 8404                                                                 140828 38328            1       2      0 38328                                                                  ROOT                                 1 > 43779                                                                208954
38829            1       2      0 38829                                                                  ROOT
                                1 < 8738                                                                 145346 38856            1       2      0 38856                                                                  ROOT                                 1 > 38857                                                                145386
39024            1       2      0 39024                                                                  ROOT
                                1 > 54494                                                                195793
39188            1       2      0 39188                                                                  ROOT
                                1 > 39190                                                                148484
39294            1       2      0 39294                                                                  ROOT
                                1 > 54547                                                                195855
39619            1       2      0 39619                                                                  ROOT
                                1 < 9256                                                                 202005
39628            1       2      0 39628                                                                  ROOT
                                1 < 9276                                                                 202019 39723            1       2      0 39723                                                                  ROOT                                 1 > 54633                                                                195952
39736            1       2      0 39736                                                                  ROOT
                                1 > 48737                                                                190514
39741            3       4      0 39741                                                                  ROOT
                                1 > 54636                                                                195956
                                1 > 54637                                                                195957
                                1 > 54638                                                                195958
39762            1       2      0 39762                                                                  ROOT
                                1 > 54644                                                                195965
39775            2       3      0 39775                                                                  ROOT
                                1 > 54646                                                                195967
                                2   > 57529                                                              188056
39785            1       2      0 39785                                                                  ROOT
                                1 > 54647                                                                195968
39820            1       2      0 39820                                                                  ROOT
                                1 > 53513                                                                191052
39979            1       2      0 39979                                                                  ROOT
                                1 > 54680                                                                196005
40009            3       3      0 40009                                                                  ROOT
                                1 > 54684                                                                196009
                                2   > 54685                                                              196011
                                3     < 40009*                                                           196010 40137            1       2      0 40137                                                                  ROOT                                 1 > 54730                                                                197267
40189            1       2      0 40189                                                                  ROOT
                                1 > 54735                                                                197272
40212            2       3      0 40212                                                                  ROOT
                                1 > 54741                                                                197275
                                1 > 54742                                                                197276
40325            4       5      0 40325                                                                  ROOT
                                1 > 54762                                                                197301
                                2   > 57570                                                              188126
                                2   > 57571                                                              188127
                                1 > 54763                                                                197302
40326            1       2      0 40326                                                                  ROOT
                                1 < 9963                                                                 206790 40365            1       2      0 40365                                                                  ROOT                                 1 > 54772                                                                197310
40368            1       2      0 40368                                                                  ROOT
                                1 > 54773                                                                197311
40377            1       2      0 40377                                                                  ROOT
                                1 > 54774                                                                197312
40813            1       2      0 40813                                                                  ROOT
                                1 > 54838                                                                197386
40814            1       2      0 40814                                                                  ROOT
                                1 > 54839                                                                197387
40842            1       2      0 40842                                                                  ROOT
                                1 > 54840                                                                197388
40847            1       2      0 40847                                                                  ROOT
                                1 > 54843                                                                197390
40853            1       2      0 40853                                                                  ROOT
                                1 > 54845                                                                197392
40856            3       4      0 40856                                                                  ROOT
                                1 > 54846                                                                197393
                                2   > 57593                                                              188149
                                3     > 58067                                                            188683
40859            1       2      0 40859                                                                  ROOT
                                1 > 54847                                                                197394
40879            2       3      0 40879                                                                  ROOT
                                1 > 54850                                                                197400
                                1 > 54851                                                                197401
40880            1       2      0 40880                                                                  ROOT
                                1 > 54852                                                                197402
40882            1       2      0 40882                                                                  ROOT
                                1 > 54854                                                                197404
40886            1       2      0 40886                                                                  ROOT
                                1 > 44198                                                                211997
40890            2       3      0 40890                                                                  ROOT
                                1 > 54855                                                                197405
                                2   > 57594                                                              188150
40896            4       4      0 40896                                                                  ROOT
                                1 > 54856                                                                197406
                                2   > 57595                                                              188151
                                3     > 57596                                                            188153
                                4       < 54856*                                                         188152 40898            4       4      0 40898                                                                  ROOT                                 1 > 54857                                                                197407
                                2   > 57597                                                              188154
                                3     > 57598                                                            188156
                                4       < 54857*                                                         188155 40907            1       2      0 40907                                                                  ROOT                                 1 > 54860                                                                197411
40917            9       9      0 40917                                                                  ROOT
                                1 > 54861                                                                197412
                                2   > 57599                                                              188157
                                3     > 57600                                                            188159
                                4       < 54861*                                                         188158                                 2   > 57601                                                              188160
                                2   > 57602                                                              188161
                                3     > 58068                                                            188684
                                2   > 57603                                                              188162
                                2   > 57604                                                              188163
40918            1       2      0 40918                                                                  ROOT
                                1 > 54862                                                                197413
40925            1       2      0 40925                                                                  ROOT
                                1 > 54865                                                                197415
40926            3       4      0 40926                                                                  ROOT
                                1 > 54866                                                                197416
                                1 > 54867                                                                197417
                                1 > 54868                                                                197418
40927            1       2      0 40927                                                                  ROOT
                                1 > 52038                                                                208097
41026            1       2      0 41026                                                                  ROOT
                                1 > 54893                                                                197448
41172            1       2      0 41172                                                                  ROOT
                                1 > 54922                                                                197484
41212            1       2      0 41212                                                                  ROOT
                                1 > 54837                                                                197385
41304            3       4      0 41304                                                                  ROOT
                                1 > 54935                                                                197498
                                1 > 54936                                                                197499
                                1 > 54937                                                                197500
41312            1       2      0 41312                                                                  ROOT
                                1 > 54944                                                                197507
41342            1       2      0 41342                                                                  ROOT
                                1 > 54957                                                                197524
41354            1       2      0 41354                                                                  ROOT
                                1 > 54958                                                                197525
41599            1       2      0 41599                                                                  ROOT
                                1 > 55001                                                                197568
41608            2       3      0 41608                                                                  ROOT
                                1 > 55003                                                                197570
                                2   > 55664                                                              185981
41919            1       2      0 41919                                                                  ROOT
                                1 > 55059                                                                197631
41992            1       2      0 41992                                                                  ROOT
                                1 > 55073                                                                198841
42002            1       2      0 42002                                                                  ROOT
                                1 > 55083                                                                198864
42003            1       2      0 42003                                                                  ROOT
                                1 > 55084                                                                198865
42033            1       2      0 42033                                                                  ROOT
                                1 > 42034                                                                193161
42186            1       2      0 42186                                                                  ROOT
                                1 > 55113                                                                198896
42252            1       2      0 42252                                                                  ROOT
                                1 > 55132                                                                198921
42474            2       3      0 42474                                                                  ROOT
                                1 > 55182                                                                198988
                                2   > 57645                                                              188203
42494            1       2      0 42494                                                                  ROOT
                                1 > 55188                                                                198994
42517            1       2      0 42517                                                                  ROOT
                                1 > 55197                                                                199004
43247            3       4      0 43247                                                                  ROOT
                                1 > 55401                                                                185668
                                2   > 57680                                                              188242
                                2   > 57681                                                              188243
43584            1       2      0 43584                                                                  ROOT
                                1 > 55497                                                                185755
43749            1       2      0 43749                                                                  ROOT
                                1 > 55555                                                                185846
43805            2       3      0 43805                                                                  ROOT
                                1 > 55563                                                                185854
                                1 > 55564                                                                185855
43942           27      11      0 43942                                                                  ROOT
                                1 > 54717                                                                197257
                                2   > 55599                                                              185898
                                3     < 43942*                                                           185897
                                3     < 55593                                                            185899
                                3     < 55594                                                            185900                                 4       > 55595                                                          185890
                                5         > 55597                                                        185893
                                6           < 55594*                                                     185892                                 6           > 55598                                                      185896
                                7             < 55594*                                                   185894
                                7             < 55595*                                                   185895                                 7             > 55599*                                                   185904
                                7             > 55600                                                    185908
                                8               < 55594*                                                 185905
                                8               < 55595*                                                 185906
                                8               < 55597*                                                 185907
                                8               < 55599*                                                 185909                                 8               > 57744                                                  188332
                                9                 < 55594*                                               188327
                                9                 < 55595*                                               188328
                                9                 < 55597*                                               188329
                                9                 < 55598*                                               188330
                                9                 < 55599*                                               188331                                 6           > 55599*                                                     185903
                                5         > 55599*                                                       185901
                                3     < 55596                                                            185902
                                4       < 43942*                                                         185891 43955            1       2      0 43955                                                                  ROOT                                 1 > 55602                                                                185911
43968            2       3      0 43968                                                                  ROOT
                                1 > 55606                                                                185916
                                2   > 57746                                                              188334
43977            2       3      0 43977                                                                  ROOT
                                1 > 55609                                                                185918
                                2   > 57748                                                              188336
44008            1       2      0 44008                                                                  ROOT
                                1 > 55615                                                                185927
44009            2       3      0 44009                                                                  ROOT
                                1 > 55616                                                                185928
                                2   > 57749                                                              188337
44011            1       2      0 44011                                                                  ROOT
                                1 > 55617                                                                185929
44013            3       3      0 44013                                                                  ROOT
                                1 > 55618                                                                185930
                                2   > 55619                                                              185932
                                3     < 44013*                                                           185931 44017            1       2      0 44017                                                                  ROOT                                 1 > 55620                                                                185933
44023            1       2      0 44023                                                                  ROOT
                                1 > 55624                                                                185939
44084            1       2      0 44084                                                                  ROOT
                                1 > 55644                                                                185961
44086            1       2      0 44086                                                                  ROOT
                                1 > 55645                                                                185962
44091            1       2      0 44091                                                                  ROOT
                                1 > 53610                                                                191159
44122            1       2      0 44122                                                                  ROOT
                                1 > 55652                                                                185970
44123            1       2      0 44123                                                                  ROOT
                                1 > 55653                                                                185971
44128            3       4      0 44128                                                                  ROOT
                                1 > 55655                                                                185972
                                2   > 57757                                                              188345
                                3     > 58099                                                            188718
44189            2       3      0 44189                                                                  ROOT
                                1 > 55661                                                                185978
                                2   > 57758                                                              188346
44191            1       2      0 44191                                                                  ROOT
                                1 > 55662                                                                185979
44201            1       2      0 44201                                                                  ROOT
                                1 > 55667                                                                185984
44203            2       3      0 44203                                                                  ROOT
                                1 > 55668                                                                185985
                                2   > 57759                                                              188347
44231            2       3      0 44231                                                                  ROOT
                                1 > 55677                                                                185997
                                1 > 55678                                                                185998
44250            1       2      0 44250                                                                  ROOT
                                1 > 53599                                                                191147
44251            1       2      0 44251                                                                  ROOT
                                1 > 55682                                                                186002
44274            1       2      0 44274                                                                  ROOT
                                1 > 55686                                                                186006
44308            9       5      0 44308                                                                  ROOT
                                1 > 55695                                                                186015
                                2   > 55696                                                              186017
                                3     < 44308*                                                           186016                                 3     > 55697                                                            186020
                                4       < 44308*                                                         186018
                                4       < 55695*                                                         186019                                 4       > 55698                                                          186023
                                5         < 44308*                                                       186021
                                5         < 55696*                                                       186022 45010            1       2      0 45010                                                                  ROOT                                 1 > 55865                                                                186211
45014            1       2      0 45014                                                                  ROOT
                                1 > 55866                                                                186212
45052            1       2      0 45052                                                                  ROOT
                                1 > 55878                                                                186223
45101            1       2      0 45101                                                                  ROOT
                                1 > 55891                                                                186239
45505            2       3      0 45505                                                                  ROOT
                                1 > 55979                                                                186335
                                2   > 57809                                                              188400
45690            1       2      0 45690                                                                  ROOT
                                1 > 56005                                                                186362
45913            1       2      0 45913                                                                  ROOT
                                1 > 45914                                                                199621
46025            1       2      0 46025                                                                  ROOT
                                1 > 56061                                                                186425
46127            1       2      0 46127                                                                  ROOT
                                1 > 56084                                                                186450
46455            1       2      0 46455                                                                  ROOT
                                1 > 56142                                                                186512
46639            2       3      0 46639                                                                  ROOT
                                1 > 55339                                                                185595
                                1 > 56175                                                                186548
46770            1       2      0 46770                                                                  ROOT
                                1 > 50528                                                                200191
46932            1       2      0 46932                                                                  ROOT
                                1 > 56219                                                                186594
47011            1       2      0 47011                                                                  ROOT
                                1 > 56231                                                                186606
47267            1       2      0 47267                                                                  ROOT
                                1 > 47296                                                                207585
47277            1       2      0 47277                                                                  ROOT
                                1 > 56267                                                                186647
47364            1       2      0 47364                                                                  ROOT
                                1 > 56279                                                                186659
47436            1       2      0 47436                                                                  ROOT
                                1 > 56286                                                                186666
47486            1       2      0 47486                                                                  ROOT
                                1 > 56295                                                                186675
48021            1       2      0 48021                                                                  ROOT
                                1 > 55064                                                                198833
48035            1       2      0 48035                                                                  ROOT
                                1 > 56402                                                                186801
48504            1       2      0 48504                                                                  ROOT
                                1 > 56479                                                                186903
48614            4       5      0 48614                                                                  ROOT
                                1 > 56500                                                                186924
                                2   > 57874                                                              188475
                                1 > 56501                                                                186925
                                1 > 56502                                                                186926
48664            1       2      0 48664                                                                  ROOT
                                1 > 56511                                                                186936
48913            1       2      0 48913                                                                  ROOT
                                1 > 56547                                                                186973
48980            2       3      0 48980                                                                  ROOT
                                1 > 56551                                                                186977
                                2   > 57878                                                              188479
49284            1       2      0 49284                                                                  ROOT
                                1 > 56596                                                                187026
49552            1       2      0 49552                                                                  ROOT
                                1 > 56639                                                                187072
49595            2       3      0 49595                                                                  ROOT
                                1 > 56646                                                                187079
                                2   > 57894                                                              188495
49878            1       2      0 49878                                                                  ROOT
                                1 > 56692                                                                187148
50472            3       3      0 50472                                                                  ROOT
                                1 > 56785                                                                187248
                                2   > 56786                                                              187250
                                3     < 50472*                                                           187249 50504            1       2      0 50504                                                                  ROOT                                 1 > 56790                                                                187253
50508            5       6      0 50508                                                                  ROOT
                                1 > 50509                                                                200166
                                1 > 56791                                                                187254
                                1 > 56792                                                                187255
                                1 > 56793                                                                187256
                                1 > 56794                                                                187257
50530            1       2      0 50530                                                                  ROOT
                                1 > 56797                                                                187260
50610            1       2      0 50610                                                                  ROOT
                                1 > 54722                                                                197259
50691            1       2      0 50691                                                                  ROOT
                                1 > 56820                                                                187282
50728            1       2      0 50728                                                                  ROOT
                                1 > 56826                                                                187289
50729            1       2      0 50729                                                                  ROOT
                                1 > 56827                                                                187290
50746            1       2      0 50746                                                                  ROOT
                                1 > 56829                                                                187292
50805            2       3      0 50805                                                                  ROOT
                                1 > 56838                                                                187302
                                2   > 57931                                                              188533
50806            1       2      0 50806                                                                  ROOT
                                1 > 56839                                                                187303
50885            2       3      0 50885                                                                  ROOT
                                1 > 56859                                                                187323
                                2   > 57934                                                              188536
51020            2       3      0 51020                                                                  ROOT
                                1 > 56902                                                                187371
                                2   > 57947                                                              188549
51082            1       2      0 51082                                                                  ROOT
                                1 > 56931                                                                187419
51095            1       2      0 51095                                                                  ROOT
                                1 > 56932                                                                187420
51294            1       2      0 51294                                                                  ROOT
                                1 > 56966                                                                187454
51295            1       2      0 51295                                                                  ROOT
                                1 > 56967                                                                187455
51358            1       2      0 51358                                                                  ROOT
                                1 > 56976                                                                187464
51407            1       2      0 51407                                                                  ROOT
                                1 > 56983                                                                187472
51439            2       3      0 51439                                                                  ROOT
                                1 > 56990                                                                187479
                                1 > 56991                                                                187480
51575            1       2      0 51575                                                                  ROOT
                                1 > 57002                                                                187491
51696            2       3      0 51696                                                                  ROOT
                                1 > 57027                                                                187518
                                1 > 57028                                                                187519
51697            2       3      0 51697                                                                  ROOT
                                1 > 57029                                                                187520
                                1 > 57030                                                                187521
51778            1       2      0 51778                                                                  ROOT
                                1 > 57042                                                                187533
51903            1       2      0 51903                                                                  ROOT
                                1 > 57066                                                                187559
51944           94      49      0 51944                                                                  ROOT
                                1 > 57077                                                                187568
                                2   > 57969                                                              188571
                                3     > 57970                                                            188573
                                4       < 57077*                                                         188572                                 4       > 58155                                                          188780
                                5         < 57969*                                                       188779                                 5         > 58157                                                        188785
                                6           < 57969*                                                     188783
                                6           < 57970*                                                     188784                                 4       > 58158                                                          188787
                                5         < 57969*                                                       188786
                                5         < 58154                                                        188788
                                6           < 57969*                                                     188778                                 6           > 58156                                                      188782
                                7             < 57969*                                                   188781                                 7             > 58158*                                                   188789
                                6           > 58208                                                      185385
                                6           > 58209                                                      185386
                                3     > 57971                                                            188575
                                4       < 57077*                                                         188574                                 3     > 58151                                                            188772
                                4       > 58152                                                          188774
                                5         < 57969*                                                       188773                                 5         > 58153                                                        188777
                                6           < 57969*                                                     188775
                                6           < 58151*                                                     188776                                 6           > 58159                                                      188793
                                7             < 57969*                                                   188790
                                7             < 58151*                                                   188791
                                7             < 58152*                                                   188792                                 7             > 58160                                                    188798
                                8               < 57969*                                                 188794
                                8               < 58151*                                                 188795
                                8               < 58152*                                                 188796
                                8               < 58153*                                                 188797                                 8               > 58161                                                  188804
                                9                 < 57969*                                               188799
                                9                 < 58151*                                               188800
                                9                 < 58152*                                               188801
                                9                 < 58153*                                               188802
                                9                 < 58159*                                               188803                                 9                 > 58204                                                185376
                               10                   < 58151*                                             185372
                               10                   < 58153*                                             185373
                               10                   < 58159*                                             185374
                               10                   < 58160*                                             185375                                10                   > 58206                                              185383
                               11                     < 58152*                                           185378
                               11                     < 58153*                                           185379
                               11                     < 58159*                                           185380
                               11                     < 58160*                                           185381
                               11                     < 58161*                                           185382                                 8               > 58210                                                  185387
                                9                 > 58222                                                185401
                                8               > 58211                                                  185388
                                9                 > 58212                                                185390
                               10                   < 58160*                                             185389                                10                   > 58223                                              185402
                                7             > 58200                                                    185362
                                8               < 58151*                                                 185361                                 5         > 58205                                                        185377
                                6           > 58219                                                      185398
                                7             > 58224                                                    185403
                                6           > 58220                                                      185399
                                7             > 58225                                                    185404
                                8               > 58226                                                  185406
                                9                 < 58220*                                               185405                                 8               > 58227                                                  185407
                                6           > 58221                                                      185400
                                5         > 58207                                                        185384
                                4       > 58195                                                          188837
                                5         > 58199                                                        185360
                                6           < 58151*                                                     185359                                 6           > 58202                                                      185366
                                7             < 58151*                                                   185364
                                7             < 58195*                                                   185365
                                7             < 58201                                                    185367
                                8               < 58151*                                                 185363                                 5         > 58218                                                        185397
                                4       > 58196                                                          185354
                                5         > 58197                                                        185356
                                6           < 58151*                                                     185355                                 6           > 58198                                                      185358
                                7             < 58151*                                                   185357                                 7             > 58203                                                    185371
                                8               < 58151*                                                 185368
                                8               < 58196*                                                 185369
                                8               < 58197*                                                 185370                                 2   > 57972                                                              188576
                                3     > 57973                                                            188578
                                4       < 57077*                                                         188577                                 3     > 58162                                                            188805
                                2   > 57974                                                              188579
                                3     > 58163                                                            188806
52712            1       2      0 52712                                                                  ROOT
                                1 > 57224                                                                187725
53283            1       2      0 53283                                                                  ROOT
                                1 > 57312                                                                187820
53490            1       2      0 53490                                                                  ROOT
                                1 > 57339                                                                187847
53510            2       3      0 53510                                                                  ROOT
                                1 > 53511                                                                191050
                                1 > 53512                                                                191051
53514            1       2      0 53514                                                                  ROOT
                                1 > 57346                                                                187855
53618            2       3      0 53618                                                                  ROOT
                                1 > 57365                                                                187873
                                2   > 58027                                                              188634
53622            1       2      0 53622                                                                  ROOT
                                1 > 57366                                                                187874
53623            1       2      0 53623                                                                  ROOT
                                1 > 57367                                                                187875
53698            1       2      0 53698                                                                  ROOT
                                1 > 57375                                                                187883
53712            1       2      0 53712                                                                  ROOT
                                1 > 57379                                                                187887
53799            4       5      0 53799                                                                  ROOT
                                1 > 56828                                                                187291
                                1 > 57392                                                                187900
                                2   > 58034                                                              188640
                                1 > 57393                                                                187901
53824            1       2      0 53824                                                                  ROOT
                                1 > 57397                                                                187905
53839            1       2      0 53839                                                                  ROOT
                                1 > 53840                                                                192614
53853            2       3      0 53853                                                                  ROOT
                                1 > 57409                                                                187920
                                1 > 57410                                                                187921
53999            1       2      0 53999                                                                  ROOT
                                1 > 57430                                                                187941
54087            2       3      0 54087                                                                  ROOT
                                1 > 57446                                                                187958
                                2   > 58049                                                              188660
54481           14       6      0 54481                                                                  ROOT
                                1 > 57502                                                                188019
                                2   > 57503                                                              188021
                                3     < 54481*                                                           188020                                 3     > 57504                                                            188024
                                4       < 54481*                                                         188022
                                4       < 57502*                                                         188023                                 4       > 57505                                                          188028
                                5         < 54481*                                                       188025
                                5         < 57502*                                                       188026
                                5         < 57503*                                                       188027                                 5         > 57506                                                        188032
                                6           < 54481*                                                     188029
                                6           < 57502*                                                     188030
                                6           < 57503*                                                     188031 54676            3       4      0 54676                                                                  ROOT                                 1 > 57531                                                                188058
                                1 > 57532                                                                188059
                                1 > 57533                                                                188060
54678            7       5      0 54678                                                                  ROOT
                                1 > 57535                                                                188062
                                2   > 57536                                                              188064
                                3     < 54678*                                                           188063                                 3     > 57537                                                            188067
                                4       < 54678*                                                         188065
                                4       < 57535*                                                         188066                                 2   > 58056                                                              188668
54711           10       5      0 54711                                                                  ROOT
                                1 > 57542                                                                188072
                                2   > 57543                                                              188074
                                3     < 54711*                                                           188073                                 3     > 57544                                                            188077
                                4       < 54711*                                                         188075
                                4       < 57542*                                                         188076                                 4       > 57545                                                          188081
                                5         < 54711*                                                       188078
                                5         < 57542*                                                       188079
                                5         < 57543*                                                       188080 54715            1       2      0 54715                                                                  ROOT                                 1 > 54716                                                                197256
54719            1       2      0 54719                                                                  ROOT
                                1 > 57548                                                                188083
54720            1       2      0 54720                                                                  ROOT
                                1 > 57549                                                                188084
54721            1       2      0 54721                                                                  ROOT
                                1 > 57550                                                                188085
54723            1       2      0 54723                                                                  ROOT
                                1 > 57552                                                                188086
54724            8       5      0 54724                                                                  ROOT
                                1 > 57553                                                                188087
                                2   > 57554                                                              188089
                                3     < 54724*                                                           188088                                 3     > 58058                                                            188671
                                4       < 57553*                                                         188670                                 4       > 58059                                                          188674
                                5         < 57553*                                                       188672
                                5         < 57554*                                                       188673 54736            4       5      0 54736                                                                  ROOT                                 1 > 57556                                                                188091
                                1 > 57557                                                                188092
                                2   > 58060                                                              188675
                                1 > 57558                                                                188093
54737            1       2      0 54737                                                                  ROOT
                                1 > 57559                                                                188094
54739            1       2      0 54739                                                                  ROOT
                                1 > 57560                                                                188095
54756           28       8      0 54756                                                                  ROOT
                                1 > 57561                                                                188096
                                2   > 57562                                                              188098
                                3     < 54756*                                                           188097                                 3     > 57563                                                            188101
                                4       < 54756*                                                         188099
                                4       < 57561*                                                         188100                                 4       > 57564                                                          188105
                                5         < 54756*                                                       188102
                                5         < 57561*                                                       188103
                                5         < 57562*                                                       188104                                 5         > 57565                                                        188110
                                6           < 54756*                                                     188106
                                6           < 57561*                                                     188107
                                6           < 57562*                                                     188108
                                6           < 57563*                                                     188109                                 6           > 57566                                                      188116
                                7             < 54756*                                                   188111
                                7             < 57561*                                                   188112
                                7             < 57562*                                                   188113
                                7             < 57563*                                                   188114
                                7             < 57564*                                                   188115                                 7             > 57567                                                    188123
                                8               < 54756*                                                 188117
                                8               < 57561*                                                 188118
                                8               < 57562*                                                 188119
                                8               < 57563*                                                 188120
                                8               < 57564*                                                 188121
                                8               < 57565*                                                 188122 54758            2       3      0 54758                                                                  ROOT                                 1 > 57568                                                                188124
                                1 > 57569                                                                188125
54765            1       2      0 54765                                                                  ROOT
                                1 > 57572                                                                188128
54841            1       2      0 54841                                                                  ROOT
                                1 > 57592                                                                188148
54864            1       2      0 54864                                                                  ROOT
                                1 > 57605                                                                188164
54871            1       2      0 54871                                                                  ROOT
                                1 > 54872                                                                197423
54896            1       2      0 54896                                                                  ROOT
                                1 > 57606                                                                188165
54897            1       2      0 54897                                                                  ROOT
                                1 > 57607                                                                188166
54974            1       2      0 54974                                                                  ROOT
                                1 > 54975                                                                197541
55071            1       2      0 55071                                                                  ROOT
                                1 > 57623                                                                188181
55081            2       3      0 55081                                                                  ROOT
                                1 > 57624                                                                188182
                                1 > 57625                                                                188183
55082            2       3      0 55082                                                                  ROOT
                                1 > 57627                                                                188184
                                1 > 57628                                                                188185
55204            1       2      0 55204                                                                  ROOT
                                1 > 57647                                                                188205
55205            1       2      0 55205                                                                  ROOT
                                1 > 55207                                                                199011
55206            6       4      0 55206                                                                  ROOT
                                1 > 57648                                                                188206
                                2   > 57649                                                              188208
                                3     < 55206*                                                           188207                                 3     > 57650                                                            188211
                                4       < 55206*                                                         188209
                                4       < 57648*                                                         188210 55410            1       2      0 55410                                                                  ROOT                                 1 > 57683                                                                188245
55411            1       2      0 55411                                                                  ROOT
                                1 > 57684                                                                188246
55412            1       2      0 55412                                                                  ROOT
                                1 > 57685                                                                188247
55415            1       2      0 55415                                                                  ROOT
                                1 > 57686                                                                188248
55416            1       2      0 55416                                                                  ROOT
                                1 > 57688                                                                188249
55417            2       3      0 55417                                                                  ROOT
                                1 > 57690                                                                188250
                                2   > 58082                                                              188700
55418            1       2      0 55418                                                                  ROOT
                                1 > 57691                                                                188251
55420            2       3      0 55420                                                                  ROOT
                                1 > 57693                                                                188252
                                2   > 58084                                                              188702
55421            1       2      0 55421                                                                  ROOT
                                1 > 57694                                                                188253
55422            1       2      0 55422                                                                  ROOT
                                1 > 57695                                                                188254
55424            5       5      0 55424                                                                  ROOT
                                1 > 55428                                                                185685
                                2   > 55429                                                              185687
                                3     < 55424*                                                           185686                                 1 > 55430                                                                185688
                                1 > 55431                                                                185689
55425            3       3      0 55425                                                                  ROOT
                                1 > 55426                                                                185682
                                2   > 55427                                                              185684
                                3     < 55425*                                                           185683 55432            1       2      0 55432                                                                  ROOT                                 1 > 57697                                                                188255
55434            2       3      0 55434                                                                  ROOT
                                1 > 57699                                                                188256
                                1 > 57700                                                                188257
55436            1       2      0 55436                                                                  ROOT
                                1 > 57701                                                                188258
55437            1       2      0 55437                                                                  ROOT
                                1 > 57702                                                                188259
55439            2       3      0 55439                                                                  ROOT
                                1 > 57703                                                                188260
                                2   > 58087                                                              188705
55440            1       2      0 55440                                                                  ROOT
                                1 > 55441                                                                185692
55442            1       2      0 55442                                                                  ROOT
                                1 > 57704                                                                188261
55532            1       2      0 55532                                                                  ROOT
                                1 > 57728                                                                188310
55607            1       2      0 55607                                                                  ROOT
                                1 > 57747                                                                188335
55639            1       2      0 55639                                                                  ROOT
                                1 > 57754                                                                188342
55654            1       2      0 55654                                                                  ROOT
                                1 > 57756                                                                188344
55871            2       3      0 55871                                                                  ROOT
                                1 > 57790                                                                188378
                                1 > 57791                                                                188379
55942            1       2      0 55942                                                                  ROOT
                                1 > 57801                                                                188391
56020            3       3      0 56020                                                                  ROOT
                                1 > 57816                                                                188412
                                2   > 57817                                                              188414
                                3     < 56020*                                                           188413 56090            3       4      0 56090                                                                  ROOT                                 1 > 57825                                                                188422
                                2   > 58114                                                              188733
                                2   > 58115                                                              188734
56126            1       2      0 56126                                                                  ROOT
                                1 > 57828                                                                188425
56143            1       2      0 56143                                                                  ROOT
                                1 > 57831                                                                188428
56144            3       3      0 56144                                                                  ROOT
                                1 > 57832                                                                188429
                                2   > 57833                                                              188431
                                3     < 56144*                                                           188430 56150            3       4      0 56150                                                                  ROOT                                 1 > 57834                                                                188432
                                1 > 57835                                                                188433
                                1 > 57836                                                                188434
56185            1       2      0 56185                                                                  ROOT
                                1 > 57841                                                                188439
56221            1       2      0 56221                                                                  ROOT
                                1 > 57846                                                                188444
56315            1       2      0 56315                                                                  ROOT
                                1 > 57851                                                                188449
56420            1       2      0 56420                                                                  ROOT
                                1 > 56421                                                                186822
56587            6       6      0 56587                                                                  ROOT
                                1 > 57889                                                                188490
                                2   > 58126                                                              188744
                                3     > 58127                                                            188746
                                4       < 57889*                                                         188745                                 4       > 58184                                                          188826
                                1 > 57890                                                                188491
56747            4       5      0 56747                                                                  ROOT
                                1 > 57916                                                                188518
                                2   > 58136                                                              188756
                                3     > 58186                                                            188828
                                2   > 58137                                                              188757
56787            1       2      0 56787                                                                  ROOT
                                1 > 57921                                                                188523
56817            1       2      0 56817                                                                  ROOT
                                1 > 57924                                                                188526
56819            1       2      0 56819                                                                  ROOT
                                1 > 57925                                                                188527
56855            2       3      0 56855                                                                  ROOT
                                1 > 57933                                                                188535
                                2   > 58139                                                              188759
56888            1       2      0 56888                                                                  ROOT
                                1 > 56889                                                                187359
56890            1       2      0 56890                                                                  ROOT
                                1 > 57943                                                                188545
57068            1       2      0 57068                                                                  ROOT
                                1 > 57963                                                                188564
57073            2       3      0 57073                                                                  ROOT
                                1 > 57964                                                                188565
                                1 > 57965                                                                188566
57074            5       5      0 57074                                                                  ROOT
                                1 > 57966                                                                188567
                                1 > 57967                                                                188568
                                2   > 57968                                                              188570
                                3     < 57074*                                                           188569                                 2   > 58150                                                              188771
57258            1       2      0 57258                                                                  ROOT
                                1 > 58007                                                                188613
57308            1       2      0 57308                                                                  ROOT
                                1 > 58017                                                                188623
57319            2       3      0 57319                                                                  ROOT
                                1 > 58018                                                                188624
                                2   > 58171                                                              188814
57359            3       3      0 57359                                                                  ROOT
                                1 > 58024                                                                188630
                                2   > 58025                                                              188632
                                3     < 57359*                                                           188631 57364            1       2      0 57364                                                                  ROOT                                 1 > 58026                                                                188633
57406            1       2      0 57406                                                                  ROOT
                                1 > 58042                                                                188654
57412            1       2      0 57412                                                                  ROOT
                                1 > 58046                                                                188657
57413            1       2      0 57413                                                                  ROOT
                                1 > 58047                                                                188658
57458            1       2      0 57458                                                                  ROOT
                                1 > 58050                                                                188661
57546            1       2      0 57546                                                                  ROOT
                                1 > 57547                                                                188082
57551            1       2      0 57551                                                                  ROOT
                                1 > 58057                                                                188669
57582            3       3      0 57582                                                                  ROOT
                                1 > 58063                                                                188678
                                2   > 58064                                                              188680
                                3     < 57582*                                                           188679 57591            2       3      0 57591                                                                  ROOT                                 1 > 58065                                                                188681
                                1 > 58066                                                                188682
57613            1       2      0 57613                                                                  ROOT
                                1 > 58069                                                                188685
57626            4       4      0 57626                                                                  ROOT
                                1 > 58072                                                                188688
                                2   > 58073                                                              188690
                                3     < 57626*                                                           188689                                 1 > 58074                                                                188691
57687            3       4      0 57687                                                                  ROOT
                                1 > 58080                                                                188698
                                2   > 58175                                                              188818
                                3     > 58213                                                            185391
57689            2       3      0 57689                                                                  ROOT
                                1 > 58081                                                                188699
                                2   > 58176                                                              188819
57692            2       3      0 57692                                                                  ROOT
                                1 > 58083                                                                188701
                                2   > 58177                                                              188820
57696            1       2      0 57696                                                                  ROOT
                                1 > 58085                                                                188703
57698            1       2      0 57698                                                                  ROOT
                                1 > 58086                                                                188704
57824            1       2      0 57824                                                                  ROOT
                                1 > 58113                                                                188732
57953            5       5      0 57953                                                                  ROOT
                                1 > 58142                                                                188762
                                2   > 58143                                                              188764
                                3     < 57953*                                                           188763                                 2   > 58189                                                              188831
                                2   > 58190                                                              188832
58033            1       2      0 58033                                                                  ROOT
                                1 > 58172                                                                188815
58045            1       2      0 58045                                                                  ROOT
                                1 > 58174                                                                188817
58090            1       2      0 58090                                                                  ROOT
                                1 > 58178                                                                188821
58118            1       2      0 58118                                                                  ROOT
                                1 > 58119                                                                188737
58182            3       3      0 58182                                                                  ROOT
                                1 > 58183                                                                188825
                                2   > 58214                                                              185393
                                3     < 58182*                                                           185392 6542             1       2      0 6542                                                                   ROOT                                 1 > 6544                                                                 54707

214625 rows selected.

Elapsed: 00:01:42.58
</pre>
</div>

#### Network summaries

<div class="scrollbox">
<pre>
Network summary 1 - by network

Network     #Links  #Nodes    Max Lev
---------- ------- ------- ----------
10020            2       2          1
10454            2       2          1
10541            2       2          1
10569            2       2          1
10572            2       2          1
11136            2       2          1
11686            2       2          1
11713            2       2          1
11770            2       2          1
11802            2       2          1
11831            2       2          1
12004            2       2          1
12095            2       2          1
12592            2       2          1
12593            2       2          1
12601            2       2          1
12709            2       2          1
12780            2       2          1
13150            2       2          1
13237            2       2          1
13277            2       2          1
13409            2       2          1
13423            2       2          1
13508            2       2          1
13516            2       2          1
13519            2       2          1
13521            2       2          1
14692            2       2          1
15577            2       2          1
15636            2       2          1
15670            2       2          1
15689            2       2          1
16794            2       2          1
16827            2       2          1
17012            2       2          1
1704             2       2          1
17047            2       2          1
17062            2       2          1
17074            2       2          1
17287            2       2          1
1731             2       2          1
18379            2       2          1
18776            2       2          1
19029            2       2          1
19925            2       2          1
20137            2       2          1
20149            2       2          1
20150            2       2          1
21785            2       2          1
21795            2       2          1
22211            2       2          1
22850            2       2          1
24435            2       2          1
24477            2       2          1
24642            2       2          1
26072            2       2          1
26403            2       2          1
26418            2       2          1
26641            2       2          1
26744            2       2          1
26766            2       2          1
26803            2       2          1
26977            2       2          1
26979            2       2          1
27113            2       2          1
27134            2       2          1
27342            2       2          1
27442            2       2          1
27481            2       2          1
27487            2       2          1
27545            2       2          1
27548            2       2          1
27737            2       2          1
27763            2       2          1
27765            2       2          1
27788            2       2          1
27789            2       2          1
27790            2       2          1
27831            2       2          1
27838            2       2          1
27880            2       2          1
27882            2       2          1
27888            2       2          1
27894            2       2          1
27897            2       2          1
27915            2       2          1
27943            2       2          1
27960            2       2          1
28061            2       2          1
28073            2       2          1
28125            2       2          1
28145            2       2          1
28192            2       2          1
28211            2       2          1
28430            2       2          1
28517            2       2          1
28531            2       2          1
28547            2       2          1
28548            2       2          1
28551            2       2          1
28557            2       2          1
28560            2       2          1
28826            2       2          1
28842            2       2          1
28891            2       2          1
29117            2       2          1
29759            2       2          1
29850            2       2          1
29851            2       2          1
29852            2       2          1
29867            2       2          1
29876            2       2          1
29893            2       2          1
29895            2       2          1
29916            2       2          1
30075            2       2          1
30473            2       2          1
34215            2       2          1
35114            2       2          1
35129            2       2          1
35196            2       2          1
35299            2       2          1
35302            2       2          1
35326            2       2          1
35381            2       2          1
35394            2       2          1
35456            2       2          1
35464            2       2          1
35474            2       2          1
35481            2       2          1
35558            2       2          1
35608            2       2          1
35609            2       2          1
35748            2       2          1
35751            2       2          1
35755            2       2          1
35759            2       2          1
35763            2       2          1
35766            2       2          1
35767            2       2          1
35769            2       2          1
35784            2       2          1
35803            2       2          1
35953            2       2          1
36004            2       2          1
36008            2       2          1
36022            2       2          1
36033            2       2          1
36272            2       2          1
36375            2       2          1
36378            2       2          1
36385            2       2          1
36399            2       2          1
36405            2       2          1
36533            2       2          1
36612            2       2          1
36821            2       2          1
36919            2       2          1
36981            2       2          1
37084            2       2          1
37108            2       2          1
37247            2       2          1
37277            2       2          1
37346            2       2          1
37348            2       2          1
37351            2       2          1
37420            2       2          1
37421            2       2          1
37515            2       2          1
37516            2       2          1
37585            2       2          1
37689            2       2          1
37715            2       2          1
37916            2       2          1
37953            2       2          1
38033            2       2          1
38066            2       2          1
38281            2       2          1
38328            2       2          1
38829            2       2          1
38856            2       2          1
39024            2       2          1
39188            2       2          1
39294            2       2          1
39619            2       2          1
39628            2       2          1
39723            2       2          1
39736            2       2          1
39762            2       2          1
39785            2       2          1
39820            2       2          1
39979            2       2          1
40137            2       2          1
40189            2       2          1
40326            2       2          1
40365            2       2          1
40368            2       2          1
40377            2       2          1
40813            2       2          1
40814            2       2          1
40842            2       2          1
40847            2       2          1
40853            2       2          1
40859            2       2          1
40880            2       2          1
40882            2       2          1
40886            2       2          1
40907            2       2          1
40918            2       2          1
40925            2       2          1
40927            2       2          1
41026            2       2          1
41172            2       2          1
41212            2       2          1
41312            2       2          1
41342            2       2          1
41354            2       2          1
41599            2       2          1
41919            2       2          1
41992            2       2          1
42002            2       2          1
42003            2       2          1
42033            2       2          1
42186            2       2          1
42252            2       2          1
42494            2       2          1
42517            2       2          1
43584            2       2          1
43749            2       2          1
43955            2       2          1
44008            2       2          1
44011            2       2          1
44017            2       2          1
44023            2       2          1
44084            2       2          1
44086            2       2          1
44091            2       2          1
44122            2       2          1
44123            2       2          1
44191            2       2          1
44201            2       2          1
44250            2       2          1
44251            2       2          1
44274            2       2          1
45010            2       2          1
45014            2       2          1
45052            2       2          1
45101            2       2          1
45690            2       2          1
45913            2       2          1
46025            2       2          1
46127            2       2          1
46455            2       2          1
46770            2       2          1
46932            2       2          1
47011            2       2          1
47267            2       2          1
47277            2       2          1
47364            2       2          1
47436            2       2          1
47486            2       2          1
48021            2       2          1
48035            2       2          1
48504            2       2          1
48664            2       2          1
48913            2       2          1
49284            2       2          1
49552            2       2          1
49878            2       2          1
50504            2       2          1
50530            2       2          1
50610            2       2          1
50691            2       2          1
50728            2       2          1
50729            2       2          1
50746            2       2          1
50806            2       2          1
51082            2       2          1
51095            2       2          1
51294            2       2          1
51295            2       2          1
51358            2       2          1
51407            2       2          1
51575            2       2          1
51778            2       2          1
51903            2       2          1
52712            2       2          1
53283            2       2          1
53490            2       2          1
53514            2       2          1
53622            2       2          1
53623            2       2          1
53698            2       2          1
53712            2       2          1
53824            2       2          1
53839            2       2          1
53999            2       2          1
54715            2       2          1
54719            2       2          1
54720            2       2          1
54721            2       2          1
54723            2       2          1
54737            2       2          1
54739            2       2          1
54765            2       2          1
54841            2       2          1
54864            2       2          1
54871            2       2          1
54896            2       2          1
54897            2       2          1
54974            2       2          1
55071            2       2          1
55204            2       2          1
55205            2       2          1
55410            2       2          1
55411            2       2          1
55412            2       2          1
55415            2       2          1
55416            2       2          1
55418            2       2          1
55421            2       2          1
55422            2       2          1
55432            2       2          1
55436            2       2          1
55437            2       2          1
55440            2       2          1
55442            2       2          1
55532            2       2          1
55607            2       2          1
55639            2       2          1
55654            2       2          1
55942            2       2          1
56126            2       2          1
56143            2       2          1
56185            2       2          1
56221            2       2          1
56315            2       2          1
56420            2       2          1
56787            2       2          1
56817            2       2          1
56819            2       2          1
56888            2       2          1
56890            2       2          1
57068            2       2          1
57258            2       2          1
57308            2       2          1
57364            2       2          1
57406            2       2          1
57412            2       2          1
57413            2       2          1
57458            2       2          1
57546            2       2          1
57551            2       2          1
57613            2       2          1
57696            2       2          1
57698            2       2          1
57824            2       2          1
58033            2       2          1
58045            2       2          1
58090            2       2          1
58118            2       2          1
6542             2       2          1
12145            3       3          1
13125            3       3          2
13152            3       3          1
13544            3       3          2
13608            3       3          1
13708            3       3          2
1469             3       3          2
15658            3       3          2
15663            3       3          1
16399            3       3          1
16525            3       3          1
16526            3       3          1
16825            3       3          2
17256            3       3          1
17401            3       3          1
17459            3       3          1
19907            3       3          2
20143            3       3          1
27336            3       3          2
27771            3       3          1
27844            3       3          1
27933            3       3          1
28297            3       3          1
45505            3       3          2
46639            3       3          1
48980            3       3          2
49595            3       3          2
50805            3       3          2
50885            3       3          2
51020            3       3          2
51439            3       3          1
51696            3       3          1
51697            3       3          1
53510            3       3          1
53618            3       3          2
53853            3       3          1
54087            3       3          2
54758            3       3          1
55081            3       3          1
55082            3       3          1
55417            3       3          2
55420            3       3          2
55434            3       3          1
55439            3       3          2
55871            3       3          1
56855            3       3          2
57073            3       3          1
57319            3       3          2
57591            3       3          1
57689            3       3          2
57692            3       3          2
28527            3       3          1
28607            3       3          1
28894            3       3          1
29901            3       3          1
31960            3       3          2
32410            3       3          2
33215            3       3          1
35283            3       3          1
35378            3       3          1
35440            3       3          2
35484            3       3          1
35540            3       3          1
35581            3       3          1
35772            3       3          1
35781            3       3          2
35938            3       3          1
36382            3       3          2
36652            3       3          2
37044            3       3          2
39775            3       3          2
40212            3       3          1
40879            3       3          1
40890            3       3          2
41608            3       3          2
42474            3       3          2
43805            3       3          1
43968            3       3          2
43977            3       3          2
44009            3       3          2
44189            3       3          2
44203            3       3          2
44231            3       3          1
11030            3       3          1
11053            3       3          1
11778            3       3          1
12115            3       3          2
50472            4       3          3
54676            4       4          1
55425            4       3          3
56020            4       3          3
56090            4       4          2
56144            4       3          3
56150            4       4          1
57359            4       3          3
57582            4       3          3
57687            4       4          3
58182            4       3          3
29341            4       3          3
29853            4       4          1
29858            4       3          3
34428            4       4          2
35189            4       4          2
36031            4       4          2
36253            4       4          2
36328            4       4          1
39741            4       4          1
40009            4       3          3
40856            4       4          3
40926            4       4          1
41304            4       4          1
43247            4       4          2
44013            4       3          3
44128            4       4          3
10061            4       4          2
11615            4       3          3
11982            4       4          2
13188            4       4          2
13441            4       4          1
13542            4       4          2
13846            4       3          3
15579            4       3          3
17084            4       4          2
18476            4       3          3
22386            4       4          1
26630            4       4          2
27761            4       3          3
28518            4       4          1
28522            4       4          2
28523            4       4          2
40325            5       5          2
48614            5       5          2
37270            5       4          3
36965            5       4          3
22949            5       4          3
36006            5       5          2
54736            5       5          2
35661            5       5          3
29902            5       4          3
11945            5       4          3
27187            5       5          3
53799            5       5          2
40898            5       4          4
56747            5       5          3
27542            5       4          3
57626            5       4          3
12063            5       4          3
40896            5       4          4
36185            6       5          4
20119            6       6          2
29898            6       5          4
29024            6       6          2
57953            6       5          3
57074            6       5          3
55424            6       5          3
27490            6       5          4
21823            6       4          4
24937            6       5          3
50508            6       6          1
13813            6       4          4
13321            6       6          2
11687            6       5          4
15675            7       6          3
55206            7       4          4
18420            7       6          3
27572            7       7          2
56587            7       6          4
27988            7       7          2
54678            8       5          4
12781            8       5          4
54724            9       5          5
26460           10       7          5
40917           10       9          4
44308           10       5          5
13496           11       8          5
35790           11      11          3
54711           11       5          5
37342           11       6          4
11628           14      10          5
54481           15       6          6
27621           26      10          9
43942           28      11          9
54756           29       8          8
51944           95      49         11
0           212946   56739      15225

547 rows selected.

Elapsed: 00:00:26.91
Network summary 2 - grouped by numbers of nodes

 #Nodes  #Networks
------- ----------
      2        362
      3        103
      4         40
      5         21
      6          9
      7          3
      8          2
      9          1
     10          2
     11          2
     49          1
  56739          1

12 rows selected.

Elapsed: 00:00:23.13
</pre>
</div>
