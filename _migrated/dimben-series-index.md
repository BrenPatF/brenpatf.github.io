---
layout: post
title: "Dimensional Benchmarking of SQL Performance"
date: 2016-11-06
group: perf-testing
migrated: true
index: dimben
permalink: /migrated/dimben-series-index/
categories: 
  - "oracle"
  - "performance"
  - "plsql"
  - "sql"
  - "testing"
tags: 
  - "benchmarking"
  - "oracle"
  - "plsql"
  - "sql"
  - "tuning"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
**2025 Note:** This is the overview/index page for a series of articles originally published on my Wordpress blog, starting in November 2016, [A Framework for Dimensional Benchmarking of SQL Query Performance](http://aprogrammerwrites.eu/?p=1833). This overview page describes the framework, while the linked articles give examples of use. More recently, I published an article in June 2025, [Coupons, Caps and Functions](https://brenpatf.github.io/2025/06/29/coupons_caps_functions.html), that uses PowerShell to drive the dimensional benchmarking, gathering just CPU and elapsed times.

<ul>
  <li><a href="/dimben/dimben-1/">DIMBEN 1: Oracle v10-v12 Queries for SQL Bursting Problems</a></li>
  <li><a href="/dimben/dimben-2/">DIMBEN 2: General SQL Bursting Problems</a></li>
  <li><a href="/dimben/dimben-3/">DIMBEN 3: Bracket Parsing SQL</a></li>
  <li><a href="/dimben/dimben-4/">DIMBEN 4: SQL for Fixed-Depth Hierarchies</a></li>
  <li><a href="/dimben/dimben-5/">DIMBEN 5: Hash Join Options in SQL for Fixed-Depth Hierarchies</a></li>
  <li><a href="/dimben/dimben-6/">DIMBEN 6: String Splitting SQL</a></li>
  <li><a href="/dimben/dimben-7/">DIMBEN 7: Oracle DML: A Case Study I - Update vs Merge, An Example</a></li>
  <li><a href="/dimben/dimben-8/">DIMBEN 8: Oracle DML: A Case Study II - Effects of Indexes</a></li>
</ul>

<div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1em;">
  <a href="https://github.com/BrenPatF/dim_bench_sql_oracle" style="font-size: 1.1em;">
    A Framework for Dimensional Benchmarking of SQL Query Performance
  </a>
  <img src="/images/common/github-mark.png" style="height: 32px; margin-left: 12px;" />
</div>

<div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1em;">
  <a href="http://www.slideshare.net/brendanfurey7/dimensional-performance-benchmarking-of-sql" style="font-size: 1.1em;">
    Dimensional Performance Benchmarking of SQL
  </a>
  <img src="/images/common/slideshare-logo-svg-vector.svg" style="height: 36px; margin-left: 12px;" />
</div>

A few years ago I wanted to investigate the performance of different SQL queries for the same problem, and wanted to include analysis of how the queries' performance varied with problem size. In order to do this efficiently I wrote an Oracle framework consisting of tables, packages, types etc., and which I have now published on GitHub, at the link above. As well as the obvious CPU and elapsed times, I included statistics contained in the execution plan tables, and also the differences in v$ view statistics that are gathered in the well known Runstats scripts, (originally developed by Tom Kyte, and for which there now seem to be lots of variations around, such as [RUNSTATS](https://github.com/oracle-developer/runstats)). My approach is to collect these statistics in tables keyed by both query and dimensions to allow for more elaborate reporting, and to easily detect unscaleable queries, for example that use resources at a rate that grows quadratically, or worse, with problem size, as in one of the demo queries. Output goes to both a text log file, and to summary csv files for importing to Excel.

**Update, 26 November 2016:** A notes section has been added discussing design issues and features.

Here is one of the Scribd articles for which I originally developed the framework:

 <iframe class="scribd_iframe_embed" title="SQL Pivot and Prune Queries - Keeping an Eye on Performance" src="https://www.scribd.com/embeds/54433084/content?start_page=1&view_mode=scroll&access_key=key-osozbbqczb7m4qadwiy" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View SQL Pivot and Prune Queries - Keeping an Eye on Performance on Scribd" href="https://www.scribd.com/document/54433084/SQL-Pivot-and-Prune-Queries-Keeping-an-Eye-on-Performance#from_embed" style="color: #098642; text-decoration: underline;"> SQL Pivot and Prune Queries - Keeping an Eye on Performance </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p>
 
## Bench Data Model - ERD

<img src="/migrated_images/2016/11/Bench-1.0-ERD.png" alt="Bench Data Model" title="Bench Data Model" />

## Code Structure Diagram

<img src="/migrated_images/2016/11/Bench-1.1-CSD.png" alt="Code Structure Diagram" title="Code Structure Diagram" />

## Test\_Queries Package Call Structure Table

<div style="overflow: auto; max-height: 700px;">

<table>
<tbody>
<tr>
<th style="text-align: left; background-color: #6690bc; color: #ffffff; padding: 2px;" valign="middle">Level 1</th>
<th style="text-align: left; background-color: #6690bc; color: #ffffff; padding: 2px;" valign="middle">Level 2</th>
<th style="text-align: left; background-color: #6690bc; color: #ffffff; padding: 2px;" valign="middle">Level 3</th>
<th style="text-align: left; background-color: #6690bc; color: #ffffff; padding: 2px;" valign="middle">Package</th>
</tr>
<tr>
<td width="150">Add_Query</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Init_Statistics</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Plan_Lines</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Get_SQL_Id</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Display_Cursor</td>
<td width="150"></td>
<td width="150">DBMS_XPlan</td>
</tr>
<tr>
<td width="150">Write_Plan_Statistics</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Get_Queries</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Flush_Buf</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Init_Time</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Put_Line</td>
<td width="150"></td>
<td width="150">UTL_File</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Increment_Time</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150">Write_Line</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Flush_Buf</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Open_File</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Construct</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Fopen</td>
<td width="150"></td>
<td width="150">UTL_File</td>
</tr>
<tr>
<td width="150">Close_File</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Flush_Buf</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">FClose</td>
<td width="150"></td>
<td width="150">UTL_File</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Times</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150">Outbound_Interface</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Construct</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Open_File</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Line</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Process_Cursor</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Init_Statistics</td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Init_Time</td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Increment_Time</td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Write_Line</td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Flush_Buf</td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Write_Plan_Statistics</td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Write_Plan</td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Write_Times</td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150"></td>
<td width="150">Get_Timer_Stats</td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Close_File</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Write_Size_list</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Write_Twice</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Line</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_CSV_Fields</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Write_Data_Points</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Heading</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Line</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Twice</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Write_Distinct_Plans</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Heading</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Write_Rows</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Heading</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Line</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Twice</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Rows</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Write_Stat</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Rows</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Write_All_Facts</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Distinct_Plans</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Data_Points</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Rows</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Stat</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Heading</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Close_File</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Write_Stats</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Open_File</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_All_Facts</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Term_Run</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Times</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150">Get_Run_Details</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Construct</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Create_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Heading</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Size_list</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Get_Queries</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Set_Data_Point</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Init_Time</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Get_CPU_Time</td>
<td width="150"></td>
<td width="150">DBMS_Utility</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Setup_Data</td>
<td width="150"></td>
<td width="150">Query_Test_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Increment_Time</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150">Run_One</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Construct</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Outbound_Interface</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Times</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150">Create_Run</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150">Main</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Main</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150">Main</td>
<td width="150"></td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Get_Run_Details</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Set_Data_Point</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Run_One</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Increment_Time</td>
<td width="150"></td>
<td width="150">Timer_Set</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Stats</td>
<td width="150"></td>
<td width="150">Test_Queries</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Log</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
<tr>
<td width="150"></td>
<td width="150">Write_Other_Error</td>
<td width="150"></td>
<td width="150">Utils</td>
</tr>
</tbody>
</table>
</div>

## Outputs

### out folder

The demo script Test\_Bur.sql writes the log data to ..\\out\\Test\_Bur.LST.

The program loops over each (W, D) data point included in the driving script lists, and outputs for each query in the group data including:

- Full execution plan
- CPU and elapsed timings of query execution steps, file writing and data setup

On completion of the loops, summary reports are written to both the main log and to summary csv files, mentioned below, with information including

- Distinct execution plans (main log only)
- Data point statistics including setup timings and records created
- Numbers of output records from the queries
- cpu and elapsed times for the queries
- Execution plan statistics
- Numerous v$ After-Before statistic differences (following the Run\_Stats model)

### Oracle directory: output\_dir

For a query group and query with data points W-D, the results of running the query are written to:

\<query\_group>\_\<query\>\_\<W\>-\<D\>.csv

For example:

BURST\_MTH\_QRY\_30-30

Two summary files are written, with the bench run id as part of the name:

\<bench run id\>\_W.csv \<bench run id\>\_D.csv

These files contain all the detailed statistics in csv format, so that they can be imported into Excel and used to obtain graphs. \_W has the width parameter as the row and depth as the column, and \_D the other way round.

## Notes

### Query Timing

Obtaining reliable comparative timings of queries in a test environment is not as straightforward as it may seem. Some of the issues are considered in this article, for example, [Timing an ALL\_ROWS query](http://www.orafaq.com/node/1407). For ad hoc tests, running the query in SQL\*Plus after 'SET AUTOTRACE TRACEONLY' is one of the better approaches. However, in this framework a different approach is taken in order to simulate the performance that might be obtained if the query records were fetched in batches to be processed, say as an outbound interface, where they may be written to a file on the server. To do this, the query select list is converted into CSV format and the records are written to a file, with care taken to separate the timings of the query operations from those of the file processing.

### Hard Parsing

This framework is not intended for testing OLTP SQL but relatively long-running batch-type SQL, where the cost of parsing is generally negligible. As the dataset sizes vary it is possible that the execution plan may vary, so it is important that the SQL engine performs a hard-parse on each execution of a query to ensure plan re-calculation. A hard parse is ensured by appending a placeholder field into the select list CSV string of the transformed queries, which is then replaced before each execution by a random number: The SQL engine considers the queries then to be distinct and therefore re-parses them.

### Code Timing

The processing within the framework is heavily instrumented using the author's own code timing utility package [Code Timing and Object Orientation and Zombies](http://aprogrammerwrites.eu/?p=1632). This is very low footprint in terms both of code and of performance, operating entirely in memory with individual timers keyed by name, and (logically) object oriented so that multiple timer sets can be running at once. Timings are printed to log, and the cpu and elapsed times for the query executed are summed from the individual components for the query, together with the times for any pre-query step:

- Pre SQL
- Open cursor
- First fetch
- Remaining fetches

### Benchmarking Non-Query SQL

The framework is centred around the concept of a group of queries that are run in turn against the same dataset for each dataset point. However, non-query SQL can also be benchmarked in two ways: First, the query can include database PL/SQL functions; and secondly, the query metadata record includes a clob field for pre-query SQL that can be a PL/SQL block, while the actual query could just be 'select 1 from dual'.

### Query Transformation by Regular Expression Processing

The query output is written to file in csv format, includes the hint GATHER\_PLAN\_STATISTICS, and has a placeholder for a random number. Rather than cluttering up the input queries with this formatting, it seemed better to have the framework do the formatting. To this end the input queries instead have a select list with individual expressions and (mandatory) aliases, which can be simple or can be in double-quotes. The aliases form the header line of the csv file. To facilitate formatting the main query select list has to be of the form:

```sql
SELECT 
/* SEL */
        expr_1          alias_1,
        expr_2          alias_2
/* SEL */
```

Each expression must be on a separate line, and the list must be delimited by comment lines /\* SEL \*/ as shown. The query formatting is performed in a procedure Get\_Queries using some fairly complex regular expression processing.

### Statistic Output Formatting

The various kinds of statistic (basic timing, execution plan aggregates, v$ statistics) are generally output in matrix format, both WxD and DxW. First the base numbers are printed for the whole grid for each query; then the last row for each query is printed, the deep or wide 'slice'; then the same two sets of output are printed for the ratios of each number compared to the smallest number at the same data point across all queries.

### Execution Plan Aggregation

After executing a query the execution plan statistics are copied from the system view v$sql\_plan\_statistics\_all into bench\_run\_v$sql\_plan\_stats\_all, and the formatted plan is written to a nested varray in bench\_run\_statistics using DBMS\_XPlan.Display\_Cursor.

At the end useful statistics in the plans are printed in aggregate by query execution, including maximum values of memory used and disk reads and writes, etc.

### Estimated vs Actual Cardinalities

Oracle's Cost Based Optimizer (CBO) uses estimated cardinalities at each step to evaluate candidate execution plans, and using the hint GATHER\_PLAN\_STATISTICS causes the actual cardinalities to be collected. Differences between estimated and actuals are generally recognised as being an important factor in whether or not a 'good' plan is chosen, so the maximum difference is included in the aggregates printed.

### V$ Statistics

The statistics in the system views v$mystat, v$latch, v$sess\_time\_model are written to bench\_run\_v$stats before query execution (value\_before, wait\_before) and after execution (value\_after, wait\_after).

At the end a selection of the (after - before) differences of these statistics is written to log and csv file in the same format as the other statistics, based on the variance across the queries at the highest data point. A simple heuristic is included in the reporting query to restrict the statistics written to those deemed of most interest in comparing the queries, but all of the statistics remain available in bench\_run\_v$stats for ad hoc querying if desired.
