---
layout: post
title: "PLSPRF 1: Hierarchical Profiler"
date: 2020-06-28
migrated: true
plsprf_next: /plsprf/plsprf-2/
categories: 
  - "object"
  - "oracle"
  - "performance"
  - "plsql"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "testing"
tags: 
  - "benchmarking"
  - "hierarchical"
  - "hprof"
  - "network"
  - "object"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "profiling"
  - "qsd"
  - "sql"
  - "subquery-factor"
  - "tuning"
---
#### Part 1 in a series on: PL/SQL Profiling

<div class="plsprf-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 PLSPRF Series:</strong>
  <a href="/migrated/plsprf-series-index/">Index</a>
  {% if page.plsprf_prev %}
    | <a href="{{ page.plsprf_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.plsprf_next %}
    | <a href="{{ page.plsprf_next }}">Next ▶</a>
  {% endif %}
</div>

This article describes the use of Oracle's hierarchical PL/SQL profiler (DBMS\_HProf) on two example program structures. The examples are designed to illustrate its behaviour over as many different scenarios as possible, while keeping the examples as simple as possible.

## Setup

The hierarchical profiler setup and use is described in the Oracle document [Using the PL/SQL Hierarchical Profiler](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/hierarchical-profiler.html). The GitHub project linked to above includes scripts for setup of prerequisites such as grants and tables, and for installation of the custom code used for this demonstration. As described in the overview article, there are two example scripts profiled.

- Example 1: General. This covers a wide range of scenarios
- Example 2: Sleep. This covers the particular example of DBMS\_Lock.Sleep

## PL/SQL Hierarchical Profiler: Data Model

<img src="/migrated_images/2020/06/plsql_profiling-HP-ERD.png" alt="ERD" title="ERD" />

## Example 1: General

The example was described in [PL/SQL Profiling](/migrated/plsprf-series-index/). The driver script is shown below:

```sql
SET TIMING ON
PROMPT B1: Start profiling; A_Calls_B 
DECLARE
  l_call_count       PLS_INTEGER := 0;
BEGIN
  HProf_Utils.Start_Profiling;
  Prof_Test.A_Calls_B(l_call_count);
END;
/
PROMPT SQL: Static DB function call
SELECT Prof_Test.DBFunc
  FROM DUAL;
PROMPT B2: Static DB function; dynamic SQL; object constructor
DECLARE
  l_cur              SYS_REFCURSOR;
  l_ret_val          PLS_INTEGER;
  l_tab_count        Table_Count_Type;
BEGIN
  SELECT Prof_Test.DBFunc
    INTO l_ret_val
    FROM DUAL;
  OPEN l_cur FOR 'SELECT Count(*) FROM all_tab_columns'; 
  FETCH l_cur INTO l_ret_val; 
  CLOSE l_cur;
  l_tab_count := Table_Count_Type('EMP');
END;
/
PROMPT B3: R_Calls_R; stop profiling
VAR RUN_ID NUMBER
DECLARE
  l_call_count       PLS_INTEGER := 0;
BEGIN
  Prof_Test.R_Calls_R(l_call_count);
  :RUN_ID := HProf_Utils.Stop_Profiling(
    p_run_comment => 'Profile for small test program with recursion',
    p_filename    => 'hp_example_general.html');
  Utils.W('Run id is ' || :RUN_ID);
END;
/
SET TIMING OFF
@hprof_queries :RUN_ID
```

The script is structured as an anonymous block, B1, then a stand-alone SQL query, followed by two more anonymous blocks, B2 and B3. Profiling is started by a call to DBMS\_Profiler.Start\_Profiler in the first block.

The last block includes a call to a custom utility, HProf\_Utils.Stop\_Profiling, that stops the profiling and analyses the trace file created in two ways:

- Writing a (standard) HTML report on the results using a DBMS\_HProf call
- Writing the results to standard tables created during installation using a DBMS\_HProf call

The custom queries are run at the end from the script hprof\_queries.sql, passing in the run identifier that's been saved in a bind variable.

### Results for Example 1: General

The results in this section come from the script hprof\_queries.sql that queries the tables populated by the analyse step.

The record produced in the run table, DBMSHP\_RUNS, was:

```
 Run Id Run Time                      Microsecs Seconds Comment
------- ---------------------------- ---------- ------- ------------------------------------------------------------
     12 02-JUN-20 06.30.37.128000 AM     823833   0.820 Profile for small test program with recursion
```

The records produced in the functions table, DBMSHP\_FUNCTION\_INFO, were:

```
Owner Module            Sy Function                  Line# Subtree MicroS Function MicroS Calls
----- ---------------- --- ------------------------- ----- -------------- --------------- -----
APP   HPROF_UTILS        4 STOP_PROFILING               61             26              26     1
APP   PROF_TEST          5 A_CALLS_B                    71         128199            9631     1
APP   PROF_TEST          6 A_CALLS_B@1                  71          87951           27332     1
APP   PROF_TEST          7 B_CALLS_A                    63         118568           30617     1
APP   PROF_TEST          8 B_CALLS_A@1                  63          60619           60619     1
APP   PROF_TEST          9 DBFUNC                      134          62114           62114     2
APP   PROF_TEST         10 R_CALLS_R                   114          27684            9234     1
APP   PROF_TEST         11 R_CALLS_R@1                 114          18428           18426     1
APP   PROF_TEST         12 __pkg_init                    0              3               3     1
APP   TABLE_COUNT_TYPE  13 TABLE_COUNT_TYPE             50         521332              31     1
APP   TABLE_COUNT_TYPE  22 __static_sql_exec_line55     55         521301          521301     1
SYS   DBMS_HPROF        14 STOP_PROFILING              747              0               0     1
SYS   DBMS_HPROF        23 __static_sql_exec_line700   700             67              67     1
SYS   DBMS_OUTPUT       15 GET_LINES                   180             79              79     3
SYS   DBMS_OUTPUT       16 NEW_LINE                    117              0               0     2
SYS   DBMS_OUTPUT       17 PUT                          77             23              23     2
SYS   DBMS_OUTPUT       18 PUT_LINE                    109             24               1     2
                         1 __anonymous_block             0        1818326             590     5
                         2 __plsql_vm                    0        1848468              17     6
                         3 __plsql_vm@1                  0          31993               4     1
                        19 __dyn_sql_exec_line12        12         133537          133537     1
                        20 __sql_fetch_line13           13        1100678         1100678     1
                        21 __static_sql_exec_line8       8          34400            2407     1

23 rows selected.
```

The "Subtree MicroS" and "Function MicroS" values are the total times in microseconds for the subtree including function, and function-only processing, respectively.

In the "Function" column, as well as procedure and function names in upper case, there are a number of names with a prefix "\_\_". The meanings of these are outlined in a table in the Oracle document mentioned above and reproduced below:

<img src="/migrated_images/2020/06/HProf-Operations-Screenshot.png" alt="Function Names" title="Function Names" />

Notice that some of the function names have a suffix "@1". These correspond to recursive calls, as will be clear from the network diagram later.

The records produced in the functions parent-child table, DBMSHP\_PARENT\_CHILD\_INFO, (with joins to DBMSHP\_FUNCTION\_INFO to get parent and child function details) were:

```
Parent: Module            Sy Function                       Child:  Module            Sy Function                       Subtree MicroS Function MicroS Calls
------- ---------------- --- ------------------------------ ------- ---------------- --- ------------------------------ -------------- --------------- -----
APP     HPROF_UTILS        4 STOP_PROFILING                 SYS     DBMS_HPROF        14 STOP_PROFILING                              0               0     1
APP     PROF_TEST          5 A_CALLS_B                      APP     PROF_TEST          7 B_CALLS_A                              118568           30617     1
APP     PROF_TEST          7 B_CALLS_A                      APP     PROF_TEST          6 A_CALLS_B@1                             87951           27332     1
APP     PROF_TEST         11 R_CALLS_R@1                    SYS     DBMS_OUTPUT       18 PUT_LINE                                    2               0     1
APP     PROF_TEST          6 A_CALLS_B@1                    APP     PROF_TEST          8 B_CALLS_A@1                             60619           60619     1
APP     PROF_TEST         10 R_CALLS_R                      APP     PROF_TEST         11 R_CALLS_R@1                             18428           18426     1
APP     PROF_TEST         10 R_CALLS_R                      SYS     DBMS_OUTPUT       18 PUT_LINE                                   22               1     1
APP     TABLE_COUNT_TYPE  13 TABLE_COUNT_TYPE               APP     TABLE_COUNT_TYPE  22 __static_sql_exec_line55               521301          521301     1
SYS     DBMS_OUTPUT       18 PUT_LINE                       SYS     DBMS_OUTPUT       17 PUT                                        23              23     2
SYS     DBMS_OUTPUT       18 PUT_LINE                       SYS     DBMS_OUTPUT       16 NEW_LINE                                    0               0     2
                           1 __anonymous_block                                        21 __static_sql_exec_line8                 34400            2407     1
                           1 __anonymous_block                                        20 __sql_fetch_line13                    1100678         1100678     1
                           1 __anonymous_block                                        19 __dyn_sql_exec_line12                  133537          133537     1
                           1 __anonymous_block              SYS     DBMS_OUTPUT       15 GET_LINES                                  79              79     3
                           1 __anonymous_block              APP     TABLE_COUNT_TYPE  13 TABLE_COUNT_TYPE                       521332              31     1
                           1 __anonymous_block              APP     PROF_TEST         10 R_CALLS_R                               27684            9234     1
                           3 __plsql_vm@1                   APP     PROF_TEST          9 DBFUNC                                  31989           31989     1
                           2 __plsql_vm                     APP     PROF_TEST          9 DBFUNC                                  30125           30125     1
                           1 __anonymous_block              APP     HPROF_UTILS        4 STOP_PROFILING                             26              26     1
                           2 __plsql_vm                                                1 __anonymous_block                     1818326             590     5
                          21 __static_sql_exec_line8                                   3 __plsql_vm@1                            31993               4     1

21 rows selected.
```

The "Subtree MicroS" and "Function MicroS" values are the total times in microseconds for the subtree including function, and function-only processing, respectively, for the child function while called from _all instances of_ the parent.

### Network Diagrams for Example 1: General

The DBMSHP\_PARENT\_CHILD\_INFO holds parent-child links between functions, while the function detail information is held in DBMSHP\_FUNCTION\_INFO. The following three columns occur in both tables (descriptions from the Oracle doc):

- SUBTREE\_ELAPSED\_TIME - Elapsed time, in microseconds, for subprogram, including time spent in descendant subprograms
- FUNCTION\_ELAPSED\_TIME - Elapsed time, in microseconds, for subprogram, excluding time spent in descendant subprograms
- CALLS - Number of calls to subprogram

In the DBMSHP\_FUNCTION\_INFO they are aggregates for the subprogram overall, whereas in the DBMSHP\_PARENT\_CHILD\_INFO they are aggregates for the subprogram when called from its parent.

The data in the tables represent a directed network (or 'digraph') of nodes and links. A root node is defined as a node that has no parent node, and we may have isolated root nodes that do not appear in the links table. It can sometimes be useful when querying network data to add a dummy node as a parent of all the root nodes, creating a single root for the network and ensuring all the non-dummy nodes appear as child nodes in the adjusted links set. Here is a diagram of the adjusted network for this example, with just the links and the function numbers included:

<img src="/migrated_images/2020/06/plsql_profiling-net-gen.png" alt="General Basic Network Diagram" title="General Basic Network Diagram" />
Notice that there are four (real) root nodes, two of which are isolated. Notice also that there are two loops in the network, whereby nodes (functions) 9 and 18 can be reached by two different paths each. However, there are no cycles since no node is both ancestor and descendant of another.

We can display more of the information the HProf utility produces in extended versions of the network diagram, where we'll separate the network into its four subnetworks, shown in two diagrams.

<img src="/migrated_images/2020/06/plsql_profiling-net-gen-time-1.png" alt="General Extended Network Diagram 1" title="General Extended Network Diagram 1" />
This diagram, of the first subnetwork, has the function names and shows the total elapsed times used by the functions within the nodes, and the elapsed times used by the functions when called by their parents within the links.

#### Loops and Hierarchies

The diagram shows two loops, where there are two routes between the loop start and end points, indicated by different colours. The second loop has two child nodes coming from the end point, and hierarchical queries (both CONNECT BY and recursive subquery factors in Oracle) cause the links to them to be duplicated. We'll see how to filter out the duplicates in the queries section below.

<img src="/migrated_images/2020/06/plsql_profiling-net-gen-time-2.png" alt="General Extended Network Diagram 2" title="General Extended Network Diagram 2" />
This diagram is for the other three subnetworks.

### Network Query Output for Example 1: General

```
Function tree                   Sy Owner Module           Inst.  Subtree MicroS Function MicroS Calls Row
------------------------------ --- ----- ---------------- ------ -------------- --------------- ----- ---
__plsql_vm                       2                                      1848468              17     6   1
  __anonymous_block              1                                      1818326             590     5   2
    __sql_fetch_line13          20                                      1100678         1100678     1   3
    TABLE_COUNT_TYPE            13 APP   TABLE_COUNT_TYPE                521332              31     1   4
      __static_sql_exec_line55  22 APP   TABLE_COUNT_TYPE                521301          521301     1   5
    __dyn_sql_exec_line12       19                                       133537          133537     1   6
    __static_sql_exec_line8     21                                        34400            2407     1   7
      __plsql_vm@1               3                                        31993               4     1   8
        DBFUNC                   9 APP   PROF_TEST        1 of 2          31989           31989     1   9
    R_CALLS_R                   10 APP   PROF_TEST                        27684            9234     1  10
      R_CALLS_R@1               11 APP   PROF_TEST                        18428           18426     1  11
        PUT_LINE                18 SYS   DBMS_OUTPUT      1 of 2              2               0     1  12
          PUT                   17 SYS   DBMS_OUTPUT      1 of 2             23              23     2  13
          NEW_LINE              16 SYS   DBMS_OUTPUT      1 of 2              0               0     2  14
      PUT_LINE                  18 SYS   DBMS_OUTPUT      2 of 2             22               1     1  15
    GET_LINES                   15 SYS   DBMS_OUTPUT                         79              79     3  18
    STOP_PROFILING               4 APP   HPROF_UTILS                         26              26     1  19
      STOP_PROFILING            14 SYS   DBMS_HPROF                           0               0     1  20
  DBFUNC                         9 APP   PROF_TEST        2 of 2          30125           30125     1  21
A_CALLS_B                        5 APP   PROF_TEST                       128199            9631     1  22
  B_CALLS_A                      7 APP   PROF_TEST                       118568           30617     1  23
    A_CALLS_B@1                  6 APP   PROF_TEST                        87951           27332     1  24
      B_CALLS_A@1                8 APP   PROF_TEST                        60619           60619     1  25
__static_sql_exec_line700       23 SYS   DBMS_HPROF                          67              67     1  26
__pkg_init                      12 APP   PROF_TEST                            3               3     1  27

25 rows selected.
```

The report query script hprof\_queries.sql queries the network in two ways, using Oracle's proprietary Connect By syntax, and also using the ANSI standard recursive subquery factoring. The output above comes from the recursive subquery factoring query. The query code is discussed in a later section.

#### Notes on Output

##### Recursive Calls - "@1" Suffix

The hierarchical profiler appends a suffix "@1" on to functions called recursively, as in A\_CALLS\_B@1 and B\_CALLS\_A@1 which is an example of mutual recursion between A\_CALLS\_B and B\_CALLS\_A.

##### Anonymous Block (\_\_anonymous\_block)

This function corresponds to invocations of anonymous blocks, obviously enough. However, there is an apparent anomaly in the number of calls listed, 5, because the driving program has only three such blocks, and there are none in the called PL/SQL code. I would surmise that the apparent discrepancy arises from the enabling of SERVEROUTPUT, which appears to result in a secondary block being associated with each explicit SQL\*Plus block, that issues a call to GET\_LINES to process buffered output.

##### PL/SQL Engine (\_\_plsql\_vm)

This function corresponds to external invocations of PL/SQL such as from a SQL\*Plus session. There are 6 calls, 5 of them presumably being linked with the external anonymous blocks, and the 6'th with DBFUNC, where a PL/SQL function is called from a SQL statement from SQL\*Plus.

Notice that the SQL statement calling a database function from within PL/SQL generates the recursive call to the engine, \_\_plsql\_vm@1

##### Package Initialization (\_\_pkg\_init)

The Prof\_Test package has a package variable g\_num that is initialized to 0 on first invocation, which gives rise to the \_\_pkg\_init function in the output.

##### Second Root (A\_CALLS\_B)

The above function does not have the \_\_plsql\_vm/\_\_anonymous\_block ancestry that might be expected because profiling only started within the enclosing block.

##### Inlined Procedure (Rest\_a\_While)

I wrote a small procedure, Rest\_a\_While, to generate some elapsed time in the recursive procedures, but preceded it with the INLINE pragma, an optimization feature new in 11g. This had the desired effect of removing the calls from the profiling output and including the times in the calling procedures. Rest\_a\_While does not make the obvious call to DBMS\_Lock.Sleep because that procedure cannot be inlined. [subprogram inlining in 11g](http://www.oracle-developer.net/display.php?id=502) provides some analysis of the inlining feature.

##### Link Duplication

As mentioned earlier, hierarchical queries cause links to be duplicated below any loops, as the queries follow all paths. However, in the output above we have a single record for each of the links, including the additional root node dummy links. This is achieved by filtering out the duplicates in an additional subquery. The “Inst.” column lists the instance number of a function having more than one instance, and the children of any such function are only listed once with the gaps in the “Row” column indicating where duplicates have been suppressed (rows 13 and 14 are omitted here).

## Example 2: Sleep

The example was described in [PL/SQL Profiling](/migrated/plsprf-series-index/). The driver script is shown below:

```sql
SET TIMING ON
VAR RUN_ID NUMBER
BEGIN
  HProf_Utils.Start_Profiling;
  DBMS_Lock.Sleep(3);
  DBMS_Lock.Sleep(6);
  :RUN_ID := HProf_Utils.Stop_Profiling(
    p_run_comment => 'Profile for DBMS_Lock.Sleep',
    p_filename    => 'hp_example_sleep.html');
  Utils.W('Run id is ' || :RUN_ID);
END;
/
SET TIMING OFF
@hprof_queries :RUN_ID
```

The script runs the start profiling procedure, then makes calls to a system procedure, DBMS\_Lock.Sleep, which sleeps without using CPU time, then inserts to a table with a Before Insert trigger that calls a custom sleep procedure, Utils.Sleep, and finally calls a custom utility that stops the profiling and analyses the trace file created. Utils.Sleep itself calls DBMS\_Lock.Sleep to do non-CPU sleeping and also runs a mathematical operation in a loop to use CPU time. The trace file is analysed in two ways:

- Writing a (standard) HTML report on the results using a DBMS\_HProf call
- Writing the results to standard tables created during installation using a DBMS\_HProf call

The custom queries are run at the end from the script hprof\_queries.sql, passing in the run identifier that's been saved in a bind variable.

### Results for Example 2: Sleep

The results in this section come from the script hprof\_queries.sql that queries the tables populated by the analyse step.

The record produced in the run table, DBMSHP\_RUNS, was:

```
 Run Id Run Time                      Microsecs Seconds Comment
------- ---------------------------- ---------- ------- ------------------------------------------------------------
     22 18-JUN-20 12.49.56.816000 PM   10997855  11.000 Profile for DBMS_Lock.Sleep
```

The records produced in the functions table, DBMSHP\_FUNCTION\_INFO, were:

```
Owner Module            Sy Function                  Line# Subtree MicroS Function MicroS Calls
----- ---------------- --- ------------------------- ----- -------------- --------------- -----
APP   HPROF_UTILS        2 STOP_PROFILING               61             87              87     1
APP   SLEEP_BI           3 SLEEP_BI                      0        1999602             327     1
LIB   UTILS              4 SLEEP                       351        1999268          999837     1
LIB   UTILS              5 __pkg_init                    0              2               2     1
LIB   UTILS              6 __pkg_init                    0              5               5     1
SYS   DBMS_HPROF         7 STOP_PROFILING              747              0               0     1
SYS   DBMS_HPROF        11 __static_sql_exec_line700   700             77              77     1
SYS   DBMS_LOCK          8 SLEEP                       207        9997626         9997626     3
SYS   DBMS_LOCK          9 __pkg_init                    0              3               3     1
                         1 __plsql_vm                    0        1999606               4     1
                        10 __static_sql_exec_line6       6        2005266            5660     1

11 rows selected.
```

The "Subtree MicroS" and "Function MicroS" values are the total times in microseconds for the subtree including function, and function-only processing, respectively.

The records produced in the functions parent-child table, DBMSHP\_PARENT\_CHILD\_INFO, (with joins to DBMSHP\_FUNCTION\_INFO to get parent and child function details) were:

```
Parent: Module            Sy Function                       Child:  Module            Sy Function                       Subtree MicroS Function MicroS Calls
------- ---------------- --- ------------------------------ ------- ---------------- --- ------------------------------ -------------- --------------- -----
APP     HPROF_UTILS        2 STOP_PROFILING                 SYS     DBMS_HPROF         7 STOP_PROFILING                              0               0     1
APP     SLEEP_BI           3 SLEEP_BI                       LIB     UTILS              5 __pkg_init                                  2               2     1
APP     SLEEP_BI           3 SLEEP_BI                       LIB     UTILS              6 __pkg_init                                  5               5     1
APP     SLEEP_BI           3 SLEEP_BI                       LIB     UTILS              4 SLEEP                                 1999268          999837     1
LIB     UTILS              4 SLEEP                          SYS     DBMS_LOCK          8 SLEEP                                  999431          999431     1
                          10 __static_sql_exec_line6                                   1 __plsql_vm                            1999606               4     1
                           1 __plsql_vm                     APP     SLEEP_BI           3 SLEEP_BI                              1999602             327     1

7 rows selected.
```

The "Subtree MicroS" and "Function MicroS" values are the total times in microseconds for the subtree including function, and function-only processing, respectively, for the child function while called from _all instances of_ the parent.

### Network Diagrams for Example 2: Sleep

![](images/plsql_profiling - net-slp.png)

<img src="/migrated_images/2020/06/plsql_profiling-net-slp.png" alt="Sleep Basic Network Diagram" title="Sleep Basic Network Diagram" />

The network diagram shows 5 nodes appearing as (real) roots, of which one (8) is also linked as a child.

<img src="/migrated_images/2020/06/plsql_profiling-net-slp-time.png" alt="Sleep Extended Network Diagram" title="Sleep Extended Network Diagram" />

The extended network diagram with function names and times included shows 9 seconds of time used by the SLEEP function in two root-level calls and 1 second in a third call as the child of the custom Utils Sleep function.

### Network Query Output for Example 2: Sleep

```
Function tree                   Sy Owner Module           Inst.  Subtree MicroS Function MicroS Calls Row
------------------------------ --- ----- ---------------- ------ -------------- --------------- ----- ---
SLEEP                            8 SYS   DBMS_LOCK        1 of 2        8998195         8998195     2   1
__static_sql_exec_line6         10                                      2005266            5660     1   2
  __plsql_vm                     1                                      1999606               4     1   3
    SLEEP_BI                     3 APP   SLEEP_BI                       1999602             327     1   4
      SLEEP                      4 LIB   UTILS                          1999268          999837     1   5
        SLEEP                    8 SYS   DBMS_LOCK        2 of 2         999431          999431     1   6
      __pkg_init                 6 LIB   UTILS                                5               5     1   7
      __pkg_init                 5 LIB   UTILS                                2               2     1   8
STOP_PROFILING                   2 APP   HPROF_UTILS                         87              87     1   9
  STOP_PROFILING                 7 SYS   DBMS_HPROF                           0               0     1  10
__static_sql_exec_line700       11 SYS   DBMS_HPROF                          77              77     1  11
__pkg_init                       9 SYS   DBMS_LOCK                            3               3     1  12

12 rows selected.
```

## Oracle HProf HTML Reports

A standard HTML report can be produced after the profiling is stopped, by a call to one of the Analyze methods. The custom procedure below calls this method to write the report to a CLOB field, then calls DBMS\_XSLProcessor.CLOB2File to write the contents to file.

The second Analyze call populates the tables read by the custom queries.

```
FUNCTION Stop_Profiling(
            p_run_comment                  VARCHAR2,                       -- run comment
            p_filename                     VARCHAR2) RETURN PLS_INTEGER IS -- file name for html report
  l_report_clob   CLOB;
BEGIN

  DBMS_HProf.Stop_Profiling;
  DBMS_HProf.Analyze(trace_id => g_trace_id, report_clob => l_report_clob);
  DBMS_XSLProcessor.CLOB2File(cl => l_report_clob, flocation => 'INPUT_DIR', fname => p_filename);
  RETURN DBMS_HProf.Analyze(trace_id => g_trace_id, run_comment => p_run_comment);

END Stop_Profiling;
```

The driver scripts specify the file name hp\_example\_general.html and hp\_example\_sleep.html, and these are written to the folder in the Oracle directory INPUT\_DIR. The files for the examples are included in the associated GitHub project. The report starts with a list of the included subreports, as in the screenshot below:

<img src="/migrated_images/2020/06/HProf_List_html.png" alt="Elapsed Time Analysis" title="Elapsed Time Analysis" />

Here is a screenshot for the first of the subreports for the general example:

<img src="/migrated_images/2020/06/HProf_FET_By_SET_html.png" alt="Subreport 1" title="Subreport 1" />

## Network Queries

The queries are in the script hprof\_queries.sql, and we discuss the network queries in this section. The script also contains simple queries against the base tables. All queries are for a given RUNID, passed in as a sqlplus parameter.

### Connect By Query

```sql
WITH links (node_fr, node_to, owner, module, function, sub_t, fun_t, calls) AS (
    SELECT  0, fni.symbolid, fni.owner, fni.module, fni.function, fni.subtree_elapsed_time, fni.function_elapsed_time, fni.calls
      FROM dbmshp_function_info fni
     WHERE fni.runid = &RUN_ID
       AND NOT EXISTS (SELECT 1 FROM dbmshp_parent_child_info pci WHERE pci.childsymid = fni.symbolid AND pci.runid = fni.runid)
     UNION ALL
    SELECT pci.parentsymid,
           pci.childsymid,
           fi.owner,
           fi.module ,
           fi.function,
           pci.subtree_elapsed_time,
           pci.function_elapsed_time,
           pci.calls
      FROM dbmshp_parent_child_info  pci
      JOIN dbmshp_function_info      fi 
        ON pci.runid                 = fi.runid 
       AND pci.childsymid            = fi.symbolid
     WHERE pci.runid                 = &RUN_ID
)
SELECT RPAD(' ', (LEVEL-1)*2, ' ') || function ftree,
       node_to sy,
       owner,
       module,
       sub_t,
       fun_t,
       calls
  FROM links
CONNECT BY PRIOR node_to = node_fr
START WITH node_fr = 0
ORDER SIBLINGS BY sub_t DESC, fun_t DESC, calls DESC, node_to
```

#### Notes on Connect By Query

##### Links Subquery

In the basic network diagrams we added a dummy root node so that all real nodes appear as child nodes, and this is implemented in the links subquery here as a union between new links to the real root nodes and the original links. This allows the main query connecting the revised links set to cover all nodes.

##### Sibling Ordering

The ORDER SIBLINGS BY clause allows us to order sibling nodes by subtree elapsed time descending, with additional tie-break columns.

##### Link Duplication

The network for the first example has two loops, and the second loop has two child nodes coming from the end point. Hierarchical queries (both CONNECT BY and recursive subquery factors in Oracle) cause the links below the loop to be duplicated as all paths are followed. This is seen in the output below for the first example, with 28 rows compared with 26 rows for the output displayed earlier, using a query with filtering implemented. The query can be extended to filter out the duplicates using analytic functions, as is implemented for the recursive subquery factoring query.

```
Function tree                   Sy Owner Module           Subtree MicroS Function MicroS Calls
------------------------------ --- ----- ---------------- -------------- --------------- -----
__plsql_vm                       2                               1848468              17     6
  __anonymous_block              1                               1818326             590     5
    __sql_fetch_line13          20                               1100678         1100678     1
    TABLE_COUNT_TYPE            13 APP   TABLE_COUNT_TYPE         521332              31     1
      __static_sql_exec_line55  22 APP   TABLE_COUNT_TYPE         521301          521301     1
    __dyn_sql_exec_line12       19                                133537          133537     1
    __static_sql_exec_line8     21                                 34400            2407     1
      __plsql_vm@1               3                                 31993               4     1
        DBFUNC                   9 APP   PROF_TEST                 31989           31989     1
    R_CALLS_R                   10 APP   PROF_TEST                 27684            9234     1
      R_CALLS_R@1               11 APP   PROF_TEST                 18428           18426     1
        PUT_LINE                18 SYS   DBMS_OUTPUT                   2               0     1
          PUT                   17 SYS   DBMS_OUTPUT                  23              23     2
          NEW_LINE              16 SYS   DBMS_OUTPUT                   0               0     2
      PUT_LINE                  18 SYS   DBMS_OUTPUT                  22               1     1
        PUT                     17 SYS   DBMS_OUTPUT                  23              23     2
        NEW_LINE                16 SYS   DBMS_OUTPUT                   0               0     2
    GET_LINES                   15 SYS   DBMS_OUTPUT                  79              79     3
    STOP_PROFILING               4 APP   HPROF_UTILS                  26              26     1
      STOP_PROFILING            14 SYS   DBMS_HPROF                    0               0     1
  DBFUNC                         9 APP   PROF_TEST                 30125           30125     1
A_CALLS_B                        5 APP   PROF_TEST                128199            9631     1
  B_CALLS_A                      7 APP   PROF_TEST                118568           30617     1
    A_CALLS_B@1                  6 APP   PROF_TEST                 87951           27332     1
      B_CALLS_A@1                8 APP   PROF_TEST                 60619           60619     1
__static_sql_exec_line700       23 SYS   DBMS_HPROF                   67              67     1
__pkg_init                      12 APP   PROF_TEST                     3               3     1

27 rows selected.
```

### Recursive Subquery Factoring Query

```sql
WITH pci_sums (childsymid, subtree_elapsed_time, function_elapsed_time, calls) AS (
    SELECT childsymid, Sum(subtree_elapsed_time), 
                       Sum(function_elapsed_time), 
                       Sum(calls)
      FROM dbmshp_parent_child_info pci
     WHERE runid = &RUN_ID
     GROUP BY childsymid
), full_tree (runid, lev, node_id, sub_t, fun_t, calls, link_id) AS (
    SELECT fni.runid, 0, fni.symbolid,
           fni.subtree_elapsed_time - Nvl(psm.subtree_elapsed_time, 0),
           fni.function_elapsed_time - Nvl(psm.function_elapsed_time, 0),
           fni.calls - Nvl(psm.calls, 0), 'root' || ROWNUM
      FROM dbmshp_function_info fni
      LEFT JOIN pci_sums psm
        ON psm.childsymid = fni.symbolid
     WHERE fni.runid = &RUN_ID
       AND fni.calls - Nvl(psm.calls, 0) > 0
     UNION ALL
    SELECT ftr.runid, 
           ftr.lev + 1, 
           pci.childsymid, 
           pci.subtree_elapsed_time, 
           pci.function_elapsed_time, 
           pci.calls,
           pci.parentsymid || '-' || pci.childsymid
      FROM full_tree ftr
      JOIN dbmshp_parent_child_info pci
        ON pci.parentsymid = ftr.node_id
       AND pci.runid = ftr.runid
) SEARCH DEPTH FIRST BY sub_t DESC, fun_t DESC, calls DESC, node_id SET rn
, tree_ranked AS (
    SELECT runid, node_id, lev, rn, 
           sub_t, fun_t, calls, 
           Row_Number () OVER (PARTITION BY node_id ORDER BY rn) node_rn,
           Count (*) OVER (PARTITION BY node_id) node_cnt,
           Row_Number () OVER (PARTITION BY link_id ORDER BY rn) link_rn
      FROM full_tree
)
SELECT RPad (' ', trr.lev*2, ' ') || fni.function ftree,
       fni.symbolid sy, fni.owner, fni.module,
       CASE WHEN trr.node_cnt > 1 THEN trr.node_rn || ' of ' || trr.node_cnt END "Inst.",
       trr.sub_t, trr.fun_t, trr.calls, 
       trr.rn "Row"
  FROM tree_ranked trr
  JOIN dbmshp_function_info fni
    ON fni.symbolid = trr.node_id
   AND fni.runid = trr.runid
 WHERE trr.link_rn = 1
 ORDER BY trr.rn
```

#### Notes on Recursive Subquery Factoring Query

##### Root Nodes and Child Sums Subquery

A function may be called both as a child of another function, and also at the top level, as the second example showed. We detect this situation by counting the number of calls for the functions in the links table (dbmshp\_parent\_child\_info) and comparing with the number of calls in the functions table (dbmshp\_function\_info). The difference corresponds to the number of root-level calls, and the root-level timings are the timing differences between the function timings and the sums of the link timings.

##### Link Duplication

As noted earlier, hierarchical queries (both Connect By and recursive subquery factors in Oracle) cause the links below any loops to be duplicated as all paths are followed. In this query an additional subquery has been added, tree\_ranked, that ranks the nodes and links by order of appearance. The node rank is used just for information in the main block, while the link rank is used to eliminate duplicate links. Gaps in the “Row” column indicate where duplicates have been suppressed.

It's worth remembering this because it's a general feature of SQL for querying hierarchies, and one that seems not to be widely understood. For larger hierarchies it can cause serious performance problems, and may justify a PL/SQL programmed solution that need not suffer the same problem, such as: [PL/SQL Pipelined Function for Network Analysis](https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/).

Here is a query structure diagram for the recursive subquery factoring query:

<img src="/migrated_images/2020/06/plsql_profiling-qsd.png" alt="QSD" title="QSD" />

I wrote about query diagramming in August 2012: [Query Structure Diagramming](https://brenpatf.github.io/migrated/query-structure-diagramming-two-examples/)

## Hierarchical Profiler Feature Summary

We can summarise the features of the Hierarchical Profiler in the following points:

- Results are organised as lists of measures by program unit and by 'unit-within-caller', and can be represented in a call hierarchy
- Results are reported at the level of program unit call or 'tracked operation'; all calls to a given program unit within a single parent unit are aggregated
- Measures reported are elapsed times and numbers of calls, but not CPU times
- External program units are included in the profiling
- Profiling is performed, after initial setup, by means of before and after API calls, followed by querying of results in tables, or viewing of HTML reports

<div class="plsprf-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 PLSPRF Series:</strong>
  <a href="/migrated/plsprf-series-index/">Index</a>
  {% if page.plsprf_prev %}
    | <a href="{{ page.plsprf_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.plsprf_next %}
    | <a href="{{ page.plsprf_next }}">Next ▶</a>
  {% endif %}
</div>
