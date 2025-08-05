---
layout: post
title: "PLSPRF 3: Custom Code Timing"
date: 2020-06-28
migrated: true
plsprf_prev: /plsprf/plsprf-2/
categories: 
  - "object"
  - "oracle"
  - "performance"
  - "plsql"
  - "recursive"
  - "sql"
  - "testing"
tags: 
  - "object"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "recursive"
  - "sql"
  - "tuning"
---
#### Part 3 in a series on: PL/SQL Profiling

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

This article describes the use of a custom code timing PL/SQL package, Timer\_Set, on two example program structures. The examples are designed to illustrate its behaviour over as many different scenarios as possible, while keeping the examples as simple as possible.

For this custom code timing demonstration I created a new version of the package used by the the Oracle profiling demos, and driver scripts (prefixed ts\_) for the same examples.

These use an 'object-oriented' timing package that I wrote a few years ago [Timer\_Set: Oracle PL/SQL code timing module on GitHub](https://github.com/BrenPatF/timer_set_oracle) to instrument at procedure and section level. It is often considered good practice to implement timing and other instrumentation permanently in production code.

In both examples a new timer set object is created, calls are made to increment timers within the set, and at the end a report on the timings is written to log. The way the timer set operates in general is illustrated by a diagram taken from the GitHub module:

<img src="/migrated_images/2020/06/Oracle-PLSQL-API-Demos-TimerSet-Flow.png" alt="Timer Set Usage" title="Timer Set Usage" />

## Setup

The GitHub project linked to above includes scripts for setup of prerequisites such as grants and tables, and for installation of the custom code used for this demonstration. As described in the overview article, there are two example scripts profiled.

- Example 1: General. This covers a wide range of scenarios
- Example 2: Sleep. This covers the particular example of DBMS\_Lock.Sleep

## Timer Set Data Model

A logical data model is shown below. There are no physical tables involved.

<img src="/migrated_images/2020/06/plsql_profiling-TS-ERD.png" alt="ERD" title="ERD" />

## Example 1: General

The example was described in [PL/SQL Profiling](/migrated/plsprf-series-index/). The driver script is shown below:

```sql
SET TIMING ON
PROMPT B1: A_Calls_B 
DECLARE
  l_call_count       PLS_INTEGER := 0;
BEGIN
  Timer_Set_Test.Init;
  Timer_Set_Test.A_Calls_B(l_call_count);
END;
/
PROMPT SQL: Static DB function call
SELECT Timer_Set_Test.DBFunc
  FROM DUAL;
PROMPT B2: Static DB function; dynamic SQL; object constructor
DECLARE
  l_cur              SYS_REFCURSOR;
  l_ret_val          PLS_INTEGER;
  l_tab_count        Table_Count_Type;
BEGIN
  SELECT Timer_Set_Test.DBFunc
    INTO l_ret_val
    FROM DUAL;
  OPEN l_cur FOR 'SELECT Count(*) FROM all_tab_columns'; 
  Timer_Set_Test.Increment_Time('Open cursor');
  FETCH l_cur INTO l_ret_val; 
  Timer_Set_Test.Increment_Time('Fetch from cursor');
  CLOSE l_cur;
  Timer_Set_Test.Increment_Time('Close cursor');

  l_tab_count := Table_Count_Type('EMP');
  Timer_Set_Test.Increment_Time('Construct object');
END;
/
PROMPT B3: R_Calls_R; write times
DECLARE
  l_call_count       PLS_INTEGER := 0;
BEGIN
  Timer_Set_Test.R_Calls_R(l_call_count);
  Timer_Set_Test.Write_Times;
END;
/
SET TIMING OFF
```

The script is structured as an anonymous block, B1, then a stand-alone SQL query, followed by two more anonymous blocks, B2 and B3. The timer set is constructed in the first block within the call:

```sql
  Timer_Set_Test.Init
```

The script then increments timers at several points, again through calls to Timer\_Set\_Test, while Timer\_Set\_Test itself has timer calls inside. The results of the timing are listed at the end by the call:

```sql
  Timer_Set_Test.Write_Times;
```

### Results for Example 1: General

```
Timer Set: Timer_Set_Test, Constructed at 27 Jun 2020 07:52:51, written at 07:52:52
===================================================================================
Timer                      Elapsed         CPU       Calls       Ela/Call       CPU/Call
---------------------- ---------- ---------- ---------- ------------- -------------
A_Calls_B, section one        0.01        0.02           2        0.00600        0.01000
A_Calls_B, section two        0.03        0.02           2        0.01450        0.01000
B_Calls_A: 2                  0.03        0.03           1        0.03100        0.03000
B_Calls_A: 4                  0.06        0.06           1        0.06100        0.06000
DBFunc                        0.08        0.06           2        0.03850        0.03000
Open cursor                   0.00        0.00           1        0.00100        0.00000
Fetch from cursor             0.27        0.29           1        0.27400        0.29000
Close cursor                  0.00        0.00           1        0.00000        0.00000
Construct object              0.03        0.01           1        0.02500        0.01000
R_Calls_R                     0.04        0.03           2        0.02000        0.01500
(Other)                       0.00        0.00           1        0.00100        0.00000
---------------------- ---------- ---------- ---------- ------------- -------------
Total                         0.55        0.52          15        0.03673        0.03467
---------------------- ---------- ---------- ---------- ------------- -------------
[Timer timed (per call in ms): Elapsed: 0.00971, CPU: 0.01068]
```

#### Notes on Output

- Calls, CPU and elapsed times have been captured at the section level for A\_Calls\_B
- Observe that, while R\_Calls\_R and A\_Calls\_B aggregate over all calls, B\_Calls\_A records values by call; this is implemented simply by including a value that changes with call in the timer name:
    
    ```
      Increment_Time('B_Calls_A: ' || x_call_no);
    ```
    
- The output shows how much of the elapsed time comes from CPU usage; in particular, note that R\_Calls\_R calls an inlined procedure Rest\_a\_While that does a square root operation in a loop to consume CPU time, and we can see that elapsed and CPU times are the same
- The timer set object is designed to be very low footprint; here 9 statements (calls to Increment\_Time), plus a small global overhead, produced 10 result lines, plus associated information
- The Total line values are calculated using timing differences betwee reporting and construction of the timer set
- The (Other) line values are calculated using the diffeences between the Total line values and the sums of the specific line values
- The Timer timed line allows the overhead of the timing itself to be estimated
- The 'object-oriented' approach allows multiple programs to be be timed at multiple levels, without interference between timings
- Formatting such as column widths and decimal places can be specified as parameters in the reporting API call, and here take the default values

## Example 2: Sleep

The example was described in [PL/SQL Profiling](/migrated/plsprf-series-index/). The driver script is shown below:

```sql
SET TIMING ON
PROMPT B1: Construct timer set; DBMS_Lock.Sleep, 3 + 6; insert to trigger_tab; write timer set
DECLARE
  l_timer_set       PLS_INTEGER;
BEGIN
  l_timer_set := Timer_Set.Construct('Profiling DBMS_Lock.Sleep');
  DBMS_Lock.Sleep(3);
  Timer_Set.Increment_Time(l_timer_set, '3 second sleep');
  INSERT INTO trigger_tab VALUES (2, 0.5);
  Timer_Set.Increment_Time(l_timer_set, 'INSERT INTO trigger_tab VALUES (2, 0.5)');
  DBMS_Lock.Sleep(6);
  Timer_Set.Increment_Time(l_timer_set, '6 second sleep');
  Utils.W(Timer_Set.Format_Results(l_timer_set));
END;
/
SET TIMING OFF
```

The script constructs a timer set, then makes calls to a system procedure, DBMS\_Lock.Sleep, which sleeps without using CPU time, then inserts to a table with a Before Insert trigger that calls a custom sleep procedure, Utils.Sleep. Utils.Sleep itself calls DBMS\_Lock.Sleep to do non-CPU sleeping and also runs a mathematical operation in a loop to use CPU time. Timers are incremented after each main call, and the results of the timing are writtten out at the end.

### Results for Example 2: Sleep

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

#### Notes on Output

- The two direct calls to the sleep procedure DBMS\_Lock.Sleep consumed elapsed time of 3 and 6 seconds, as specified in the calls, but no CPU time
- The insert statement consumed 2 seconds elapsed time, of which 1 second comes from CPU time, matching the field values, which the trigger uses to determine elapsed time and fraction of CPU time

## Timer Set Custom Code Timing Feature Summary

We can summarise the features of Timer Set custom code timing in the following points:

- Results are organised as tables of measures for lists of timers in one or more sets
- Results are reported at the level of code section, and timing is aggregated between calls to API methods
- Measures reported are elapsed times, numbers of calls, and CPU times
- External program units may be included in the profiling: calls can be timed; internal profiling requires code changes
- Profiling is performed, after initial setup, by means of instrumentation within the program units to be profiled, including an API call to return results

I applied my Timer\_Set code timing package to some demo PL/SQL APIs in a Github module that also demonstrtates logging and unit testing. This is described here: [Oracle PL/SQL API Demos Github Module](http://aprogrammerwrites.eu/?p=2637)

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
