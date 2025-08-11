---
layout: post
title: "Oracle PL/SQL API Demos Github Module"
date: 2019-11-13
migrated: true
group: func-testing
categories: 
categories: 
  - "json"
  - "oracle"
  - "plsql"
  - "sql"
  - "testing"
tags: 
  - "design-pattern"
  - "json"
  - "performance-2"
  - "plsql"
  - "sql"
  - "unit-test"
---

This post is essentially the readme for my Github module, [Oracle PL/SQL API Demos](https://github.com/BrenPatF/oracle_plsql_api_demos)

This is a Github module demonstrating instrumentation and logging, code timing and unit testing of Oracle PL/SQL APIs.

PL/SQL procedures were written against Oracle's HR demo schema to represent the different kinds of API across two axes: Setter/Getter and Real Time/Batch.

```
Mode          | Setter Example (S)          | Getter Example (G)
--------------|-----------------------------|----------------------------------
Real Time (R) | Web service saving          | Web service getting by ref cursor
Batch (B)     | Batch loading of flat files | View
```

The PL/SQL procedures and view were written originally to demonstrate unit testing, and are as follows:

- RS: Emp\_WS.Save\_Emps - Save a list of new employees to database, returning list of ids with Julian dates; logging errors to err$ table
- RG: Emp\_WS.Get\_Dept\_Emps - For given department id, return department and employee details including salary ratios, excluding employees with job 'AD\_ASST', and returning none if global salary total < 1600, via ref cursor
- BS: Emp\_Batch.Load\_Emps - Load new/updated employees from file via external table
- BG: hr\_test\_view\_v - View returning department and employee details including salary ratios, excluding employees with job 'AD\_ASST', and returning none if global salary total < 1600

Each of these is unit tested, as described below, and in addition there is a driver script, api\_driver.sql, that calls each of them and lists the results of logging and code timing.

I presented on [Writing Clean Code in PL/SQL and SQL](https://www.slideshare.net/brendanfurey7/clean-coding-in-plsql-and-sql) at the Ireland Oracle User Group Conference on 4 April 2019 in Dublin. The modules demonstrated here are written in the style recommended in the presentation where, in particular:

- 'functional' code is preferred
- object-oriented code is used only where necessary, using a package record array approach, rather than type bodies
- record types, defaults and overloading used extensively to provide clean API interfaces

# Screen Recordings on this Module

I initially made a series of screen recordings that are available at the links below, and later condensed each recording to a length that would upload directly to Twitter, i.e. less than 140 seconds. You can find the [Twitter thread here](https://twitter.com/BrenPatF/status/1195226809987674113). Both sets of recordings are also available in the recordings subfolder of the repository. The links below are to the initial, longer set of recordings, and link to the individual mp4 files on GitHub.

## 1 Overview (6 recordings – 48m)

- [1.1 Introduction (5m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/1.1L%20Overview%20-%20Introduction.mp4)
- [1.2 Unit testing (13m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/1.2L%20Overview%20-%20Unit%20Testing.mp4)
- [1.3 Logging and instrumentation (8m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/1.3L%20Overview%20-%20Logging%20and%20Instrumentation.mp4)
- [1.4 Code timing (6m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/1.4L%20Overview%20-%20Code%20Timing.mp4)
- [1.5 Functional PL/SQL I - pure functions; record types; separation of pure and impure (8m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/1.5L%20Overview%20-%20Functional%20PLSQL%20I%20-%20Pure%20Functions.mp4)
- [1.6 Functional PL/SQL II - refactoring for purity (8m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/1.6L%20Overview%20-%20Functional%20PLSQL%20II%20-%20Refactoring.mp4)

## 2 Prerequisite Tools (1 recording – 3m)

- [2.1 Prerequisite tools (3m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/2.1L%20Prerequisite%20Tools.mp4)

## 3 Installation (3 recordings – 15m)

- [3.1 Clone git repository (2m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/3.1L%20Clone%20Git%20Repository.mp4)
- [3.2 Install prerequisite modules (7m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/3.2L%20Install%20Prerequisite%20Modules.mp4)
- [3.3 Install API demo components (6m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/3.3L%20Install%20API%20Demo%20Components.mp4)

## 4 Running the scripts (4 recordings – 30m)

- [4.1 Run unit tests (8m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/4.1L%20Run%20Unit%20Tests.mp4)
- [4.2 Review test results (7m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/4.2L%20Review%20Test%20Results.mp4)
- [4.3 Run API driver (8m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/4.3L%20Run%20API%20Driver.mp4)
- [4.4 Review API driver output (7m)](https://github.com/BrenPatF/oracle_plsql_api_demos/blob/master/recordings/longer/4.4LReview%20API%20Driver%20Output.mp4)

# Unit Testing

The PL/SQL APIs are tested using the Math Function Unit Testing design pattern, with test results in HTML and text format included. The design pattern is based on the idea that all API testing programs can follow a universal design pattern, using the concept of a ‘pure’ function as a wrapper to manage the ‘impurity’ inherent in database APIs. I explained the concepts involved in a presentation at the Ireland Oracle User Group Conference in March 2018:

[The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing)

In this data-driven design pattern a driver program reads a set of scenarios from a JSON file, and loops over the scenarios calling the wrapper function with the scenario as input and obtaining the results as the return value. Utility functions from the Trapit module convert the input JSON into PL/SQL arrays, and, conversely, the output arrays into JSON text that is written to an output JSON file. This latter file contains all the input values and output values (expected and actual), as well as metadata describing the input and output groups. A separate nodejs module can be run to process the output files and create HTML files showing the results: Each unit test (say \`pkg.prc\`) has its own root page \`pkg.prc.html\` with links to a page for each scenario, located within a subfolder \`pkg.prc\`. Here, they have been copied into a subfolder test\_output, as follows:

- tt\_emp\_batch.load\_emps
- tt\_emp\_ws.get\_dept\_emps
- tt\_emp\_ws.save\_emps
- tt\_view\_drivers.hr\_test\_view\_v

Where the actual output record matches expected, just one is represented, while if the actual differs it is listed below the expected and with background colour red. The employee group in scenario 4 of tt\_emp\_ws.save\_emps has two records deliberately not matching, the first by changing the expected salary and the second by adding a duplicate expected record.

Each of the \`pkg.prc\` subfolders also includes a JSON Structure Diagram, \`pkg.prc.png\`, showing the input/output structure of the pure unit test wrapper function. For example:

<img src="/migrated_images/2019/11/tt_emp_ws.save_emps.png" alt="" title="" />

Running a test causes the actual values to be inserted to the JSON object, which is then formatted as HTML pages:

<img src="/migrated_images/2019/11/Oracle-PLSQL-API-Demos-DFD.png" alt="" title="" />

Here is the output JSON for the 4'th scenario of the corresponding test:

```
    "2 valid records, 1 invalid job id (2 deliberate errors)":{
       "inp":{
          "Employee":[
             "LN 4|EM 4|IT_PROG|3000",
             "LN 5|EM 5|NON_JOB|4000",
             "LN 6|EM 6|IT_PROG|5000"
          ]
       },
       "out":{
          "Employee":{
             "exp":[
                "3|LN 4|EM 4|IT_PROG|1000",
                "5|LN 6|EM 6|IT_PROG|5000",
                "5|LN 6|EM 6|IT_PROG|5000"
             ],
             "act":[
                "3|LN 4|EM 4|IT_PROG|3000",
                "5|LN 6|EM 6|IT_PROG|5000"
             ]
          },
          "Output array":{
             "exp":[
                "3|LIKE /^[A-Z -]+[A-Z]$/",
                "0|ORA-02291: integrity constraint (.) violated - parent key not found",
                "5|LIKE /^[A-Z -]+[A-Z]$/"
             ],
             "act":[
                "3|ONE THOUSAND NINE HUNDRED NINETY-EIGHT",
                "0|ORA-02291: integrity constraint (.) violated - parent key not found",
                "5|TWO THOUSAND"
             ]
          },
          "Exception":{
             "exp":[
             ],
             "act":[
             ]
          }
       }
    }
```

Here are images of the unit test summary and 4'th scenario pages for the corresponding test:

<img src="/migrated_images/2019/11/ws-save.png" alt="" title="" />

<img src="/migrated_images/2019/11/sce-4.png" alt="" title="" />

# Logging and Instrumentation

Program instrumentation means including lines of code to monitor the execution of a program, such as tracing lines covered, numbers of records processed, and timing information. Logging means storing such information, in database tables or elsewhere.

The Log\_Set module allows for logging of various data in a lines table linked to a header for a given log, with the logging level configurable at runtime. The module also uses Oracle's DBMS\_Application\_Info API to allow for logging in memory only with information accessible via the V$SESSION and V$SESSION\_LONGOPS views.

The two web service-type APIs, Emp\_WS.Save\_Emps and Emp\_WS.Get\_Dept\_Emps, use a configuration that logs only via DBMS\_Application\_Info, while the batch API, Emp\_Batch.Load\_Emps, also logs to the tables. The view of course does not do any logging itself but calling programs can log the results of querying it.

The driver script api\_driver.sql calls all four of the demo APIs and performs its own logging of the calls and the results returned, including the DBMS\_Application\_Info on exit. The driver logs using a special DEBUG configuration where the log is constructed implicitly by the first Put, and there is no need to pass a log identifier when putting (so debug lines can be easily added in any called package). At the end of the script queries are run that list the contents of the logs created during the session in creation order, first normal logs, then a listing for error logs (of which one is created by deliberately raising an exception handled in WHEN OTHERS).

<img src="/migrated_images/2019/11/Oracle-PLSQL-API-Demos-LogSet-Flow.png" alt="" title="" />

<img src="/migrated_images/2019/11/Oracle-PLSQL-API-Demos-LogSet.png" alt="" title="" />

Here, for example, is the text logged by the driver script for the first call:

```
Call Emp_WS.Save_Emps to save a list of employees passed...
===========================================================
DBMS_Application_Info: Module = EMP_WS: Log id 127
...................... Action = Log id 127 closed at 12-Sep-2019 06:20:2
...................... Client Info = Exit: Save_Emps, 2 inserted
Print the records returned...
=============================
1862 - ONE THOUSAND EIGHT HUNDRED SIXTY-TWO
1863 - ONE THOUSAND EIGHT HUNDRED SIXTY-THREE

```

# Code Timing

The code timing module Timer\_Set is used by the driver script, api\_driver.sql, to time the various calls, and at the end of the main block the results are logged using Log\_Set.

<img src="/migrated_images/2019/11/Oracle-PLSQL-API-Demos-TimerSet-Flow.png" alt="" title="" />

<img src="/migrated_images/2019/11/Oracle-PLSQL-API-Demos-TimerSet.png" alt="" title="" />

The timing results are listed for illustration below:

```
Timer Set: api_driver, Constructed at 12 Sep 2019 06:20:28, written at 06:20:29
===============================================================================
Timer             Elapsed         CPU       Calls       Ela/Call       CPU/Call
------------- ---------- ---------- ---------- ------------- -------------
Save_Emps            0.00        0.00           1        0.00100        0.00000
Get_Dept_Emps        0.00        0.00           1        0.00100        0.00000
Write_File           0.00        0.02           1        0.00300        0.02000
Load_Emps            0.22        0.15           1        0.22200        0.15000
Delete_File          0.00        0.00           1        0.00200        0.00000
View_To_List         0.00        0.00           1        0.00200        0.00000
(Other)              0.00        0.00           1        0.00000        0.00000
------------- ---------- ---------- ---------- ------------- -------------
Total                0.23        0.17           7        0.03300        0.02429
------------- ---------- ---------- ---------- ------------- -------------
[Timer timed (per call in ms): Elapsed: 0.00794, CPU: 0.00873]

```

# Functional PL/SQL

The recordings 1.5 and 1.6 show examples of the functional style of PL/SQL used in the utility packages demonstrated, and here is a diagram from 1.6 illustrating a design pattern identified in refactoring the main subprogram of the unit test programs. 

<img src="/migrated_images/2019/11/Oracle-PLSQL-API-Demos-Nested-subprograms.png" alt="" title="" />

# Installation

## Install 1: Install pre-requisite tools

### Oracle database with HR demo schema

The database installation requires a minimum Oracle version of 12.2, with Oracle's HR demo schema installed [Oracle Database Software Downloads](https://www.oracle.com/database/technologies/oracle-database-software-downloads.html).

If HR demo schema is not installed, it can be got from here: [Oracle Database Sample Schemas](https://docs.oracle.com/cd/E11882_01/server.112/e10831/installation.htm#COMSC001).

### Github Desktop

In order to clone the code as a git repository you need to have the git application installed. I recommend [Github Desktop](https://desktop.github.com/) UI for managing repositories on windows. This depends on the git application, available here: [git downloads](https://git-scm.com/downloads), but can also be installed from within Github Desktop, according to these instructions: [How to install GitHub Desktop](https://www.techrepublic.com/article/how-to-install-github-desktop/).

### nodejs (Javascript backend)

nodejs is needed to run a program that turns the unit test output files into formatted HTML pages. It requires no javascript knowledge to run the program, and nodejs can be installed [here](https://nodejs.org/en/download/).

## Install 2: Clone git repository

The following steps will download the repository into a folder, oracle\_plsql\_api\_demos, within your GitHub root folder:

- Open Github desktop and click \[File/Clone repository...\]
- Paste into the url field on the URL tab: https://github.com/BrenPatF/oracle\_plsql\_api\_demos.git
- Choose local path as folder where you want your GitHub root to be
- Click \[Clone\]

## Install 3: Install pre-requisite modules

The demo install depends on the pre-requisite modules Utils, Trapit, Log\_Set, and Timer\_Set, and \`lib\` and \`app\` schemas refer to the schemas in which Utils and examples are installed, respectively.

The pre-requisite modules can be installed by following the instructions for each module at the module root pages listed in the \`See also\` section below. This allows inclusion of the examples and unit tests for those modules. Alternatively, the next section shows how to install these modules directly without their examples or unit tests here.

### \[Schema: sys; Folder: install\_prereq\] Create lib and app schemas and Oracle directory

- install\_sys.sql creates an Oracle directory, \`input\_dir\`, pointing to 'c:\\input'. Update this if necessary to a folder on the database server with read/write access for the Oracle OS user
- Run script from slqplus:

_SQL> @install\_sys_

### \[Schema: lib; Folder: install\_prereq\\lib\] Create lib components

- Run script from slqplus:

_SQL> @install\_lib\_all_

### \[Schema: app; Folder: install\_prereq\\app\] Create app synonyms

- Run script from slqplus:

_SQL> @c\_syns\_all_

### \[Folder: (npm root)\] Install npm trapit package

The npm trapit package is a nodejs package used to format unit test results as HTML pages.

Open a DOS or Powershell window in the folder where you want to install npm packages, and, with [nodejs](https://nodejs.org/en/download/) installed, run:

_$ npm install trapit_

This should install the trapit nodejs package in a subfolder .\\node\_modules\\trapit

## Install 4: Create Oracle PL/SQL API Demos components

### \[Folder: (root)\]

- Copy the following files from the root folder to the server folder pointed to by the Oracle directory INPUT\_DIR:
    - tt\_emp\_ws.save\_emps\_inp.json
    - tt\_emp\_ws.get\_dept\_emps\_inp.json
    - tt\_emp\_batch.load\_emps\_inp.json
    - tt\_view\_drivers.hr\_test\_view\_v\_inp.json
- There is also a bash script to do this, assuming C:\\input as INPUT\_DIR:

_$ ./cp\_json\_to\_input.sh_

### \[Schema: lib; Folder: lib\]

- Run script from slqplus:

_SQL> @install\_jobs app_

### \[Schema: hr; Folder: hr\]

- Run script from slqplus:

_SQL> @install\_hr app_

### \[Schema: app; Folder: app\]

- Run script from slqplus:

_SQL> @install\_api\_demos lib_

# Running Driver Script and Unit Tests

## Running driver script

### \[Schema: app; Folder: app\]

- Run script from slqplus:

_SQL> @api\_driver_

The output is in api\_driver.log

## Running unit tests

### \[Schema: app; Folder: app\]

- Run script from slqplus:

_SQL> @r\_tests_

Testing is data-driven from the input JSON objects that are loaded from files into the table tt\_units (at install time), and produces JSON output files in the INPUT\_DIR folder, that contain arrays of expected and actual records by group and scenario. These files are:

- tt\_emp\_batch.load\_emps\_out.json
- tt\_emp\_ws.get\_dept\_emps\_out.json
- tt\_emp\_ws.save\_emps\_out.json
- tt\_view\_drivers.hr\_test\_view\_v\_out.json

The output files are processed by a nodejs program that has to be installed separately, from the \`npm\` nodejs repository, as described in the Installation section above. The nodejs program produces listings of the results in HTML and/or text format, and result files are included in the subfolders below test\_output. To run the processor (in Windows), open a DOS or Powershell window in the trapit package folder after placing the output JSON files in the subfolder ./examples/externals and run:

_$ node ./examples/externals/test-externals_

# Operating System/Oracle Versions

## Windows

Tested on Windows 10, should be OS-independent

## Oracle

- Tested on Oracle Database Version 19.3.0.0.0 (minimum required: 12.2)

# See also

- [Utils - Oracle PL/SQL general utilities module](https://github.com/BrenPatF/oracle_plsql_utils)
- [Trapit - Oracle PL/SQL unit testing module](https://github.com/BrenPatF/trapit_oracle_tester)
- [Log\_Set - Oracle logging module](https://github.com/BrenPatF/log_set_oracle)
- [Timer\_Set - Oracle PL/SQL code timing module](https://github.com/BrenPatF/timer_set_oracle)
- [Trapit - nodejs unit test processing package](https://github.com/BrenPatF/trapit_nodejs_tester)

# License

MIT
