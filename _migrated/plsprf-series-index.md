---
layout: post
title: "PL/SQL Profiling"
date: 2013-03-05
group: perf-testing
migrated: true
index: plsprf
permalink: /migrated/plsprf-series-index/
categories: 
  - "object"
  - "oracle"
  - "performance"
  - "plsql"
  - "recursive"
  - "sql"
  - "testing"
tags: 
  - "benchmarking"
  - "cpu"
  - "hierarchical"
  - "hprof"
  - "oracle"
  - "performance"
  - "plsql"
  - "profiling"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
---

This article describes at overview level the use of three PL/SQL profiling methods on two example program structures. The examples are designed to illustrate profiler behaviour over as many different scenarios as possible, while keeping the examples as simple as possible. The first two profilers are Oracle's hierarchical and flat profiling tools, while the third is the author's own custom code timing package, Timer\_Set. It's based on an article published in March 2013 on the hierarchical profiler and updated shortly thereafter with the inclusion of Oracle's older flat profiler and of custom code timing. In June 2020 installation and source code were put onto GitHub, and the article was restructured into an overview article with the detail on the three profiling methods as separate articles. This is the overview and the links to the other three detailed articles are:

<ul>
  <li><a href="/plsprf/plsprf-1/">PLSPRF 1: Hierarchical Profiler</a></li>
  <li><a href="/plsprf/plsprf-2/">PLSPRF 2: Flat Profiler</a></li>
  <li><a href="/plsprf/plsprf-3/">PLSPRF 3: Custom Code Timing</a></li>
</ul>

All source code, including installation scripts, and short screen recordings, is available on GitHub:
- [GitHub: Oracle PL/SQL Profiling](https://github.com/BrenPatF/plsql_profiling)

- [Twitter thread with recordings attached](https://twitter.com/BrenPatF/status/1277482621967183877)

## Scenarios

<img src="/migrated_images/2013/03/Einstein_Simple_41ur1b0DkJL._AC_.jpg" alt="Einstein Simple" title="Einstein Simple" />

This phrase is often attributed to Albert Einstein, although the attribution is apparently questionable: [Everything Should Be Made as Simple as Possible, But Not Simpler]( http://quoteinvestigator.com/2011/05/13/einstein-simple/). In any case it's not a bad approach to follow in testing, whether in 'closed' unit testing, where there are known expected results, or in 'open' exploratory tesing where we are investigating application behaviour, as here. I discussed the issue of test scenario coverage in a 2018 presentation: [Database API Viewed As A Mathematical Function: Insights into Testing – OUG Ireland Conference 2018](http://aprogrammerwrites.eu/?p=2318).

At the time of the original article in 2013, I looked at Oracle's hierarchical profiler tool with a view to using it in an upcoming project. In order to understand the tool properly, I felt it would be a good idea to start by using it to profile a test program that would be as simple as possible while covering as wide a range of scenarios as possible. I then added a second simple program to cover some additional scenarios, and also tested the use of Oracle's flat profiler, and of custom code timing using my own Timer\_Set package.

This first article explains the scenarios covered by the two example programs, briefly summarises the results of each method on the second and simpler example, and finishes with a comparison of the features of the three methods. Fuller discussions of each profiling method, with application to both example programs can be found in the specific articles linked above.

The test programs consist of two driving scripts that have one or more PL/SQL blocks with calls to various system and custom packaged procedures and functions.

### Example 1: General

The test program for example 1 covers the following scenarios:

- Multiple root calls
- Recursive procedure calls (procedure calling itself: R\_Calls\_R)
- Mutually recursive procedure calls (procedures call each other: A\_Calls\_B and B\_Calls\_A)
- Program unit called by multiple program units (child with multiple parents: Database Function)
- Procedure 'inlined' within PL/SQL (Rest\_a\_While)
- Type body code (Object Constructor)
- Static SQL from Sqlplus (SELECT 1)
- Static SQL within PL/SQL (SELECT 2)
- Dynamic SQL within PL/SQL (SELECT 3)
- CPU-consuming sleep (Rest\_a\_While)

#### Call Structure Diagram

<img src="/migrated_images/2013/03/plsql_profiling-csd-gen.png" alt="Oracle PL/SQL Profiling CSD general" title="Oracle PL/SQL Profiling CSD general" />

### Example 2: Sleep

The test program for example 2 covers the following scenarios:

- External system program unit (DBMS\_Lock.Sleep in SYS schema)
- External custom program unit (Utils.Sleep in LIB schema)
- Sleep that uses no CPU time, only elapsed time (DBMS\_Lock.Sleep)
- Sleep that uses CPU time (Utils.Sleep)
- Trigger code (Before Insert trigger SLEEP\_BI)
- Two calls from the same parent (DBMS\_Lock.Sleep from anonymous block twice)
- Program unit as root call and as child call (DBMS\_Lock.Sleep)

#### Call Structure Diagram

<img src="/migrated_images/2013/03/plsql_profiling-csd-slp.png" alt="Oracle PL/SQL Profiling CSD sleep" title="Oracle PL/SQL Profiling CSD sleep" />

## Profiling Overview

In this section I show the data model and a summary of the results of each method on the second and simpler example. For fuller discussion with both examples see the specific articles for each method.

### Hierarchical Profiler Overview

[PLSPRF 1: Hierarchical Profiler](/plsprf/plsprf-1/)

#### Hierarchical Profiler: Data Model

<img src="/migrated_images/2013/03/plsql_profiling-HP-ERD.png" alt="Oracle PL/SQL Profiling HP ERD" title="Oracle PL/SQL Profiling HP ERD" />

#### Network Diagram for Example 2: Sleep

<img src="/migrated_images/2013/03/plsql_profiling-net-slp-time.png" alt="Oracle PL/SQL Profiling net sleep time" title="Oracle PL/SQL Profiling net sleep time" />

 The extended network diagram with function names and times included shows 9 seconds of time used by the SLEEP function in two root-level calls and 1 second in a third call as the child of the custom Utils Sleep function.

##### Network Query Output for Example 2: Sleep

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

### Flat Profiler Overview

[PLSPRF 2: Flat Profiler](/plsprf/plsprf-2/)

#### Flat Profiler: Data Model

<img src="/migrated_images/2013/03/plsql_profiling-DP-ERD.png" alt="Oracle PL/SQL Profiling DP ERD" title="Oracle PL/SQL Profiling DP ERD" />

#### Flat Profiler: Results Extract

The records produced in the functions table, PLSQL\_PROFILER\_DATA, are listed below in order of unit name, then unit number and line number.

```
Seconds  Microsecs   Min S   Max S    Calls Unit                 Unit# Type            Line# Line Text
------- ---------- ------- ------- -------- -------------------- ----- --------------- ----- ------------------------------------------------------------------
  0.000          0   0.000   0.000        0 <anonymous>              1 ANONYMOUS BLOCK     1
  0.000          2   0.000   0.000        0 <anonymous>              1 ANONYMOUS BLOCK     5
  0.000         23   0.000   0.000        1 <anonymous>              1 ANONYMOUS BLOCK     9
  0.000         10   0.000   0.000        1 <anonymous>              1 ANONYMOUS BLOCK    11
  0.000        199   0.000   0.000        1 <anonymous>              1 ANONYMOUS BLOCK    13
  0.000          3   0.000   0.000        1 <anonymous>              1 ANONYMOUS BLOCK    15
  0.000          5   0.000   0.000        1 <anonymous>              1 ANONYMOUS BLOCK    17
  0.000          0   0.000   0.000        0 <anonymous>              1 ANONYMOUS BLOCK    19
  0.000          2   0.000   0.000        0 SLEEP_BI                 2 TRIGGER             1 TRIGGER sleep_bi
  0.000         20   0.000   0.000        1 SLEEP_BI                 2 TRIGGER             2 BEFORE INSERT
  0.000          1   0.000   0.000        1 SLEEP_BI                 2 TRIGGER             4 FOR EACH ROW

11 rows selected.
```

### Timer Set Overview

[PLSPRF 3: Custom Code Timing](/plsprf/plsprf-3/)

#### Timer Set: Data Model
Oracle PL/SQL Profiling
<img src="/migrated_images/2013/03/plsql_profiling-TS-ERD.png" alt="Oracle PL/SQL Profiling TimerSet ERD" title=" TimerSet ERD" />

#### Timer Set: Example of Call Structure

<img src="/migrated_images/2013/03/Oracle-PLSQL-API-Demos-TimerSet-Flow.png" alt="Oracle PL/SQL API Demos TimerSet Flow" title="Oracle PL/SQL API Demos TimerSet Flow" />

#### Timer Set: Results Extract

```
Timer Set: Profiling DBMS_Lock.Sleep, Constructed at 27 Jun 2020 07:53:00, written at 07:53:11
==============================================================================================
Timer                                       Elapsed         CPU       Calls       Ela/Call       CPU/Call
--------------------------------------- ---------- ---------- ---------- ------------- -------------
3 second sleep                                 3.00        0.00           1        3.00000        0.00000
INSERT INTO trigger_tab VALUES (2, 0.5)        2.00        1.00           1        1.99900        1.00000
6 second sleep                                 6.00        0.00           1        6.00000        0.00000
(Other)                                        0.00        0.00           1        0.00000        0.00000
--------------------------------------- ---------- ---------- ---------- ------------- -------------
Total                                         11.00        1.00           4        2.74975        0.25000
--------------------------------------- ---------- ---------- ---------- ------------- -------------
[Timer timed (per call in ms): Elapsed: 0.00980, CPU: 0.00980]
```

## Conclusion

Oracle's hierarchical profiler and flat profiler offer different ways of profiling elapsed time and call usage for PL/SQL programs. After initial setup they perform profiling of code automatically between start and stop API calls, without the need to alter the code being profiled. The level of profiling and organisation of data collection are not configurable. They are intended for one-off use or occasional trouble-shooting, rather than as ongoing instrumentation.

Custom code timing requires insertion of timing code within the profiled code, but offers more flexibility in terms of detail gathered, and also captures CPU time usage. Use of a centralized API such as Timer\_Set helps to minimize the code required. It can be used both for trouble-shooting and for ongoing production instrumentation.

### Features Summary

<img src="/migrated_images/2013/03/Features-Table-w30.png" alt="Features Table w30" title="Features Table w30" />
