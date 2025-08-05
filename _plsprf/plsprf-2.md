---
layout: post
title: "PLSPRF 2: Flat Profiler"
date: 2020-06-28
migrated: true
plsprf_prev: /plsprf/plsprf-1/
plsprf_next: /plsprf/plsprf-3/
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
  - "object"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "profiling"
  - "recursive"
  - "sql"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
#### Part 2 in a series on: PL/SQL Profiling

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

This article describes the use of Oracle's flat PL/SQL profiler (DBMS\_PROFILER) on two example program structures. The examples are designed to illustrate its behaviour over as many different scenarios as possible, while keeping the examples as simple as possible. 

## Setup

The flat profiler setup and use is described in the Oracle document [DBMS\_PROFILER](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_PROFILER.html#GUID-8ED8AEAB-F134-4265-8117-219E1C01A53E). The GitHub project linked to above includes scripts for setup of prerequisites such as grants and tables, and for installation of the custom code used for this demonstration. As described in the overview article, there are two example scripts profiled.

- Example 1: General. This covers a wide range of scenarios
- Example 2: Sleep. This covers the particular example of DBMS\_Lock.Sleep

## PL/SQL Flat Profiler: Data Model

<img src="/migrated_images/2020/06/plsql_profiling-DP-ERD.png" alt="" title="" />

## Example 1: General

The example was described in [PL/SQL Profiling](/migrated/plsprf-series-index/). The driver script is shown below:

```sql
SET TIMING ON
PROMPT B1: Start profiling; A_Calls_B 
VAR RUN_ID NUMBER
DECLARE
  l_call_count       PLS_INTEGER := 0;
  l_result           PLS_INTEGER;
BEGIN
  l_result := DBMS_Profiler.Start_Profiler(
          run_comment => 'Profile for small test program with recursion',
          run_number  => :RUN_ID);
  Utils.W('Start: profile run id = ' || :RUN_ID);
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
DECLARE
  l_call_count       PLS_INTEGER := 0;
BEGIN
  Prof_Test.R_Calls_R(l_call_count);
  Utils.W('Stop: result = ' || DBMS_Profiler.Stop_Profiler);
  END;
/
SET TIMING OFF
@dprof_queries :RUN_ID
```

The script is structured as an anonymous block, B1, then a stand-alone SQL query, followed by two more anonymous blocks, B2 and B3. Profiling is started by a call to DBMS\_Profiler.Start\_Profiler in the first block.

The last block stops the profiling. The custom queries are run at the end from the script dprof\_queries.sql, passing in the run identifier that's been saved in a bind variable.

### Results for Example 1: General

The results in this section come from the script dprof\_queries.sql that queries the tables populated by the profiler.

#### Example 1: General - Run Header

The record produced in the run table, PLSQL\_PROFILER\_RUNS, was:

```
 Run Id Time     Seconds  Microsecs
------- -------- ------- ----------
      1 07:52:12   0.766     766000
```

#### Example 1: General - Profiler Data Summaries

Profiler data overall summary (PLSQL\_PROFILER\_DATA):

```
Seconds  Microsecs    Calls
------- ---------- --------
  0.731     730945       70
```

Profiler data summary by unit (PLSQL\_PROFILER\_DATA):

```
Unit                 Unit# Seconds  Microsecs    Calls
-------------------- ----- ------- ---------- --------
<anonymous>              1   0.000         47        3
<anonymous>              3   0.000         63        2
<anonymous>              4   0.000        103        2
<anonymous>              5   0.402     401821        6
<anonymous>              7   0.000        103        2
<anonymous>              8   0.000         87        3
PROF_TEST                2   0.309     308722       48
TABLE_COUNT_TYPE         6   0.020      20000        4
```

#### Example 1: General - Profiler Data by Unit

The records produced in the functions table, PLSQL\_PROFILER\_DATA, are listed below in order of unit name, then unit number and line number. The table is joined to PLSQL\_PROFILER\_UNITS to get the unit name and other details, which are then used to outer-join to the system view ALL\_SOURCE to get the line of source code for stored units (i.e. not for anonymous blocks).

<div class="scrollbox">
<pre>
Seconds  Microsecs   Min S   Max S    Calls Unit                 Unit# Type            Line# Line Text
------- ---------- ------- ------- -------- -------------------- ----- --------------- ----- ------------------------------------------------------------------
  0.000          0   0.000   0.000        0 &lt;anonymous&gt;              1 ANONYMOUS BLOCK     1
  0.000          0   0.000   0.000        0 &lt;anonymous&gt;              1 ANONYMOUS BLOCK     2
  0.000          2   0.000   0.000        0 &lt;anonymous&gt;              1 ANONYMOUS BLOCK     6
  0.000         26   0.000   0.000        1 &lt;anonymous&gt;              1 ANONYMOUS BLOCK    10
  0.000         18   0.000   0.000        1 &lt;anonymous&gt;              1 ANONYMOUS BLOCK    11
  0.000          1   0.000   0.000        1 &lt;anonymous&gt;              1 ANONYMOUS BLOCK    13
  0.000         63   0.000   0.000        2 &lt;anonymous&gt;              3 ANONYMOUS BLOCK     1
  0.000        103   0.000   0.000        2 &lt;anonymous&gt;              4 ANONYMOUS BLOCK     1
  0.000          8   0.000   0.000        0 &lt;anonymous&gt;              5 ANONYMOUS BLOCK     1
  0.000        139   0.000   0.000        1 &lt;anonymous&gt;              5 ANONYMOUS BLOCK     8
  0.151     150687   0.151   0.151        1 &lt;anonymous&gt;              5 ANONYMOUS BLOCK    12
  0.251     250899   0.251   0.251        1 &lt;anonymous&gt;              5 ANONYMOUS BLOCK    13
  0.000         30   0.000   0.000        1 &lt;anonymous&gt;              5 ANONYMOUS BLOCK    14
  0.000         55   0.000   0.000        1 &lt;anonymous&gt;              5 ANONYMOUS BLOCK    16
  0.000          3   0.000   0.000        1 &lt;anonymous&gt;              5 ANONYMOUS BLOCK    18
  0.000        103   0.000   0.000        2 &lt;anonymous&gt;              7 ANONYMOUS BLOCK     1
  0.000          4   0.000   0.000        0 &lt;anonymous&gt;              8 ANONYMOUS BLOCK     1
  0.000          0   0.000   0.000        1 &lt;anonymous&gt;              8 ANONYMOUS BLOCK     2
  0.000         44   0.000   0.000        1 &lt;anonymous&gt;              8 ANONYMOUS BLOCK     5
  0.000         39   0.000   0.000        1 &lt;anonymous&gt;              8 ANONYMOUS BLOCK     7
  0.000          0   0.000   0.000        0 &lt;anonymous&gt;              8 ANONYMOUS BLOCK     9
  0.000          1   0.000   0.000        0 PROF_TEST                2 PACKAGE BODY        1 PACKAGE BODY Prof_Test AS
  0.000          0   0.000   0.000        1 PROF_TEST                2 PACKAGE BODY       44 g_num NUMBER := 0;
  0.000          3   0.000   0.000        0 PROF_TEST                2 PACKAGE BODY       71 PROCEDURE A_Calls_B(
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       75 x_call_no := x_call_no + 1;
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       76 PRAGMA INLINE (Rest_a_While, 'YES');
  0.017      16691   0.004   0.013        2 PROF_TEST                2 PACKAGE BODY       77 Rest_a_While(1000 * x_call_no);
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       79 PRAGMA INLINE (Rest_a_While, 'YES'); -- Both pragmas are required
  0.034      33515   0.009   0.025        2 PROF_TEST                2 PACKAGE BODY       80 Rest_a_While(2000 * x_call_no);
  0.000          1   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       82 IF x_call_no < 4 THEN
  0.000          1   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       83 B_Calls_A(x_call_no);
  0.000          2   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       86 END A_Calls_B;
  0.000          3   0.000   0.000        0 PROF_TEST                2 PACKAGE BODY       94 PROCEDURE B_Calls_A(
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       98 x_call_no := x_call_no + 1;
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY       99 PRAGMA INLINE (Rest_a_While, 'YES');
  0.126     126418   0.044   0.083        2 PROF_TEST                2 PACKAGE BODY      100 Rest_a_While(5000 * x_call_no);
  0.000          1   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      102 IF x_call_no < 4 THEN
  0.000          0   0.000   0.000        1 PROF_TEST                2 PACKAGE BODY      103 A_Calls_B(x_call_no);
  0.000          1   0.000   0.000        1 PROF_TEST                2 PACKAGE BODY      106 END B_Calls_A;
  0.000          3   0.000   0.000        0 PROF_TEST                2 PACKAGE BODY      114 PROCEDURE R_Calls_R(
  0.000         40   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      118 DBMS_Output.Put_Line('In R_Calls_R, x_call_no = ' || x_call_no);
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      119 x_call_no := x_call_no + 1;
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      120 PRAGMA INLINE (Rest_a_While, 'YES');
  0.041      40688   0.015   0.026        2 PROF_TEST                2 PACKAGE BODY      121 Rest_a_While(3000 * x_call_no);
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      123 IF x_call_no < 2 THEN
  0.000          0   0.000   0.000        1 PROF_TEST                2 PACKAGE BODY      124 R_Calls_R(x_call_no);
  0.000          2   0.000   0.000        1 PROF_TEST                2 PACKAGE BODY      127 END R_Calls_R;
  0.000          3   0.000   0.000        0 PROF_TEST                2 PACKAGE BODY      134 FUNCTION DBFunc
  0.000          0   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      138 PRAGMA INLINE (Rest_a_While, 'YES');
  0.091      91346   0.042   0.049        2 PROF_TEST                2 PACKAGE BODY      139 Rest_a_While(10000);
  0.000          2   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      141 RETURN 99;
  0.000          2   0.000   0.000        2 PROF_TEST                2 PACKAGE BODY      143 END DBFunc;
  0.000          0   0.000   0.000        1 PROF_TEST                2 PACKAGE BODY      145 END Prof_Test;
  0.000          3   0.000   0.000        0 TABLE_COUNT_TYPE         6 TYPE BODY          50 CONSTRUCTOR FUNCTION Table_Count_Type(
  0.000          1   0.000   0.000        1 TABLE_COUNT_TYPE         6 TYPE BODY          54 SELF.search_str := p_search_str;
  0.020      19991   0.020   0.020        1 TABLE_COUNT_TYPE         6 TYPE BODY          55 SELECT Count(*)
  0.000          3   0.000   0.000        1 TABLE_COUNT_TYPE         6 TYPE BODY          60 RETURN;
  0.000          2   0.000   0.000        1 TABLE_COUNT_TYPE         6 TYPE BODY          62 END;

58 rows selected.
</pre>
</div>

#### Notes on Output

The manual has notes on interpreting output, [DBMS\_PROFILER Operational Notes](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_PROFILER.html#GUID-8ED8AEAB-F134-4265-8117-219E1C01A53E), and we can add some notes here on the output above.

##### Unit Names and Numbers

Unit numbers appear to be consecutive integers generated at run time in the order in which the units are encountered. Unit names for saved source code units such as packages and types are the names of the stored units. Anonymous blocks of PL/SQL are not stored as saved units and so do not have their own names, and are assigned the name <anonymous>, but separate top-level blocks have distinct unit numbers.

##### Named Units and Source Lines

Named units such as package and type bodies have source code lines accessible in the system view ALL\_SOURCE, and these are displayed in the output above. See the query section below for the join code.

##### Anonymous Blocks

We can't include the source text for anonymous blocks as it is not stored in the database. However, we can generally work out which unit numbers correspond to which blocks manually, and then use the line numbers to identify the corresponding source lines, with the line numbers being counted from the first line in the block as line 1. For example, line 11 in the output above for unit 1 used about 1ms and can be seen to correspond to the 11'th line in the block labelled B1 in the driver script:

```
  Prof_Test.A_Calls_B(l_call_count);
```

##### PRAGMA INLINE

Calls to the Rest\_a\_While procedure in package Prof\_Test are preceded by the inline pragma that causes the procedure code to be inlined by the PL/SQL engine. In this case we do not see any references to lines numbers 49-59 in Prof\_Test where the procedure is located, only to the calling statements on lines 75, 78, 98, 119 and 137.

## Example 2: Sleep

The example was described in [PL/SQL Profiling](/migrated/plsprf-series-index/). The driver script is shown below:

```sql
SET TIMING ON
VAR RUN_ID NUMBER
PROMPT B1: Start profiling; DBMS_Lock.Sleep, 3 + 6; stop profiling
DECLARE
  l_result           PLS_INTEGER;
BEGIN
  l_result := DBMS_Profiler.Start_Profiler(
          run_comment => 'Profile for DBMS_Lock.Sleep example',
          run_number  => :RUN_ID);
  Utils.W('Start: profile run id = ' || :RUN_ID);
  DBMS_Lock.Sleep(3);
  INSERT INTO trigger_tab VALUES (2, 0.5);
  DBMS_Lock.Sleep(6);
  Utils.W('Stop: result = ' || DBMS_Profiler.Stop_Profiler);
END;
/
SET TIMING OFF
@dprof_queries :RUN_ID
```

The script runs the start profiling procedure, then makes calls to a system procedure, DBMS\_Lock.Sleep, which sleeps without using CPU time, then inserts to a table with a Before Insert trigger that calls a custom sleep procedure, Utils.Sleep, and finally calls a custom utility that stops the profiling and analyses the trace file created. Utils.Sleep itself calls DBMS\_Lock.Sleep to do non-CPU sleeping and also runs a mathematical operation in a loop to use CPU time. The custom queries are run at the end from the script hprof\_queries.sql, passing in the run identifier that's been saved in a bind variable.

### Results for Example 2: Sleep

The results in this section come from the script dprof\_queries.sql that queries the tables populated by the profiler.

This second example was added after I came across an AskTom post concerning a discrepancy between reported times at the aggregate level and detail levels for the flat profiler. I posted a suggestion that using the hierarchical profiler might resolve the problem [Try the hierarchical profiler...](http://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:458240723799#6245362800346270441), and then added this second example to my original article in 2013 (since extended).

#### Example 2: Sleep - Run Header

The record produced in the run table, PLSQL\_PROFILER\_RUNS, was:

```
 Run Id Time     Seconds  Microsecs
------- -------- ------- ----------
      2 07:52:22  11.000   11000000
```

#### Example 2: Sleep - Profiler Data Summaries

Profiler data overall summary (PLSQL\_PROFILER\_DATA):

```
Seconds  Microsecs    Calls
------- ---------- --------
  0.000        265        7
```

Profiler data summary by unit (PLSQL\_PROFILER\_DATA):

```
Unit                 Unit# Seconds  Microsecs    Calls
-------------------- ----- ------- ---------- --------
<anonymous>              1   0.000        242        5
SLEEP_BI                 2   0.000         23        2
```

#### Example 2: Sleep - Profiler Data by Unit and Line

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

#### Notes on Output

##### Calls to Units with EXECUTE ONLY Access

The manual states "you cannot use the package to profile units for which EXECUTE ONLY access has been granted". In this example there are calls to two units where this applies: the system package DBMS\_Lock, and the custom utility package Utils, which is in a different schema (lib) from the one (app) in which the script is run.

In the output above we can see the lines from which the calls are made but nothing within the units called.

##### Aggregate/Detail Timing Discrepancy

As was noted in the AskTom thread referenced above, where the flat profiler does not provide data for program units, such as DBMS\_Lock.Sleep, the timings at the detail level do not add up to the overall time recorded in the runs table. As there were three calls using elapsed time of 11 seconds in total the total recorded in the runs table is 11 seconds, while this 11 seconds is missing from the detail records, which add up to only 265 microseconds in total.

## Queries

The queries are in the script dprof\_queries.sql. All queries are for a given RUNID, passed in as a sqlplus parameter.

### Run Header Query

```sql
SELECT runid,
       To_Char(run_date, 'hh24:mi:ss') run_date,
       Round(run_total_time/1000000000, 3) seconds,
       Round(run_total_time/1000, 0)  micro_s
  FROM plsql_profiler_runs
 WHERE runid = &RUN_ID
```

- RUN\_TOTAL\_TIME in nanoseconds is converted to seconds and microseconds

### Profiler Data Overall Summary Query

```sql
SELECT Round(Sum(dat.total_time/1000000000), 3) seconds,
       Round(Sum(dat.total_time/1000), 0)  micro_s,
       Sum(dat.total_occur) calls
  FROM plsql_profiler_data dat
 WHERE dat.runid = &RUN_ID
```

- This query sums the times and calls in PLSQL\_PROFILER\_DATA for the given RUNID

### Profiler Data Summary by Unit Query

```sql
SELECT unt.unit_name,
       dat.unit_number,
       Round(Sum(dat.total_time/1000000000), 3) seconds,
       Round(Sum(dat.total_time/1000), 0)  micro_s,
       Sum(dat.total_occur) calls
  FROM plsql_profiler_data dat
  JOIN plsql_profiler_units unt
    ON unt.runid = dat.runid
   AND unt.unit_number = dat.unit_number
 WHERE dat.runid = &RUN_ID
 GROUP BY unt.unit_name,
          dat.unit_number
 ORDER BY 1, 2, 3
```

- This query sums the times and calls in PLSQL\_PROFILER\_DATA by unit name and number, for the given RUNID

### Lines Query

```sql
SELECT Round(dat.total_time/1000000000, 3) seconds,
       Round(dat.total_time/1000, 0)  micro_s,
       Round(dat.min_time/1000000000, 3) min_secs,
       Round(dat.max_time/1000000000, 3) max_secs,
       dat.total_occur calls,
       unt.unit_name,
       dat.unit_number,
       unt.unit_type,
       dat.line#,
       Trim(src.text) text
  FROM plsql_profiler_data dat
  JOIN plsql_profiler_units unt
    ON unt.runid            = dat.runid
   AND unt.unit_number      = dat.unit_number
  LEFT JOIN all_source      src
    ON src.type             != 'ANONYMOUS BLOCK'
   AND src.name             = unt.unit_name 
   AND src.line             = dat.line#
   AND src.owner            = unt.unit_owner
   AND src.type             = unt.unit_type
 WHERE dat.runid            = &RUN_ID
 ORDER BY unt.unit_name, unt.unit_number, dat.line#
```

- This query lists the times and calls in PLSQL\_PROFILER\_DATA, for the given RUNID
- PLSQL\_PROFILER\_UNITS is joined on UNIT\_NUMBER
- The view ALL\_SOURCE is outer-joined on UNIT\_NAME, UNIT\_OWNER, UNIT\_TYPE, LINE# to give the source line for stored source
- Anonymous blocks do not have any saved source lines

## Flat Profiler Feature Summary

We can summarise the features of the Flat Profiler in the following points:

- Results are organised as lists of measures by line
- Results are reported at the line level
- Measures reported are elapsed times and numbers of calls, but not CPU times
- External program units may not be included in the profiling (they are included only if the user can debug the unit)
- Profiling is performed, after initial setup, by means of before and after API calls, followed by querying of results in tables

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
