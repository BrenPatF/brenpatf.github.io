---
layout: post
title: "DBAPIT 3: Batch Loading of Flat Files"
date: 2016-09-24
migrated: true
categories: 
  - "oracle"
  - "plsql"
  - "testing"
  - "utplsql"
tags: 
  - "external-table"
  - "oracle"
  - "plsql"
  - "unit-test"
  - "utplsql"
dbapit_prev: /dbapit/dbapit-2/
dbapit_next: /dbapit/dbapit-4/
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
#### Part 3 in a series on: Design Patterns for Database API Testing

<div class="dbapit-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 DBAPIT Series:</strong>
  <a href="/migrated/dbapit-series-index/">Index</a>
  {% if page.dbapit_prev %}
    | <a href="{{ page.dbapit_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.dbapit_next %}
    | <a href="{{ page.dbapit_next }}">Next ▶</a>
  {% endif %}
</div>

<img src="/migrated_images/2016/09/Mode-B_S.png" alt="Mode: Batch Loading" title="Mode: Batch Loading" />

In the third article in the series a design pattern is presented for testing procedures for batch loading of data.

A procedure loads data into a database table from file by means of external tables. There are many possible variants of this kind of interface, and I tried to combine the features from past projects that seemed to work best. In particular:

- The load procedure is assumed to be executed from an operating system script (normally Unix) that manages the input files, copying from source files into a single file for reading by the external table, looping over multiple files where necessary, and archiving processed files
    - It is possible to dynamically map the external table to multiple files, but that approach involves executing DDL and seems to be generally more complex
    - The idea is to do the different types of processing at the level most appropriate, so avoiding excessive file processing within the database
- SQL operations are performed at set level, rather than within a loop, and the set concerned comprises the entire contents of the external table
    - This assumes that the Oracle internal working space requirements (such as size of TEMP tablespace) are not exceeded
    - The alternative approach of fetching batches of records into arrays for loading tends to be more complex, and less efficient
    - It may be preferable to restrict the size of the input files where necessary instead, either at the source end, or from the controlling operating system script
    - Where loading occurs from staging tables (which is not in scope here) the use of the DBMS\_Parallel\_Execute package to control transaction size looks an attractive modern approach (from v11, [DBMS\_PARALLEL\_EXECUTE](https://oracle-base.com/articles/11g/dbms_parallel_execute_11gR2))
- Metadata tables are used for specifying job control parameters, and for recording run statistics
    - Run statistics include job status and numbers of records succeeding and failing at both external table and database level
    - A percentage threshold is included in the job control table, that causes the job to fail if a higher percentage of records fails
- Oracle's DML LOG ERRORS clause is used to divert failing records into an errors table, while processing the other records normally
    - While simpler than other approaches to error handling, prior to v12.1 this clause had a significant performance overhead, but from v12.1 this is no longer the case (see [DML Error Logging in Oracle 10g Database Release 2](https://oracle-base.com/articles/10g/dml-error-logging-10gr2) - it has a table of comparative timings up to v12.1 at the end)
    - The standard Oracle err$ table structure is used, with the addition of two columns, one to identify the associated job run, and the other the utid column used by the framework to identify test data
    - In a replication environment a unique identifier would be needed, but is not included here
    - The table is also used to capture records that fail custom validation (here when a record passed as an update does not match an existing record)

## Requirement Summary


- The procedure reads employee records from a single flat file
- Employee id is an optional field, and if passed the record is treated as an update, otherwise it is an insert
- A job statistics table is populated with a record on each run, and stores numbers of successful and failed records etc.
- Records that fail at the database level are inserted into an errors table, extended from Oracle's err$ table format
- The external table has all fields defined as 4000 bytes character fields to minimise records failing to be read from the external table
- An audit date column is included, and is set to the current date if any change is made in a record; if an unchanged record is detected no update should be made
- The load program takes two parameters that would be passed in by an operating system calling script:
    - An identifier for the file, that may be a timestamped original file name (but the external table file has a fixed name)
    - A line count for the file
- The passed file identifier is saved in the job statistics table, and a repeat identifier will only be accepted if previous records all have failed status
- A job control table stores an error percentage threshold, above which the run is considered unsuccessfull, and an exception is raised

## Notes on Testing Flat File Loads

- It is considered best practice to keep testing code as self-contained as possible, and in order to avoid dependence on external data files, Oracle's UTL\_File package is used to delete and create the test files from arrays within the test code
    - The Utils package contains two wrapper procedures to facilitate this: Delete\_File and Write\_File

## Extended ERD

<img src="/migrated_images/2016/09/unit-testing-two-ff-erd-h.png" alt="Extended ERD" title="Extended ERD" />

## Design Pattern Groups

The testing framework is centred around the concept of input and output groups, representing the data sets that respectively form the inputs to, and outputs from, the program. The records in each group are printed by the framework with column headers for each scenario. These groups are identified by the developer, and in this case they are as noted below.

### Input Groups

- Parameters
- Input File
- Batch Job Table
- Job Statistics Table
- Employees Table

### Output Groups

- Employees Table
- Errors Table
- Job Statistics Table
- Exception
- Timing of average call

## Test Scenarios

### Key

The scenario descriptions start with two sets of counts, followed by a verbal description. The first set is on the records coming in from file:

- NV - new valid records
- OV - old valid records
- OU - old unchanged records
- NI - new invalid records
- OI - old valid records
- EI - external table invalid records

The second set is on the existing records in job statistics and employees (other than are in the input file; these are counted as old records in the first set of counts):

- J - job statistics records
- E - employees records

### Scenario List

1. NV/OV/OU/NI/OI/EI: 1/0/0/0/0/0. Existing J/E: 0/0. \[1 valid new record from scratch\]
2. NV/OV/OU/NI/OI/EI: 1/1/1/0/0/0. Existing J/E: 1/0. \[3 valid records of each kind\]
3. NV/OV/OU/NI/OI/EI: 0/0/0/0/1/0. Existing J/E: 1/1. Uid not found \[1 invalid old - exception\]
4. NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. Email too long \[1 invalid new - exception\]
5. NV/OV/OU/NI/OI/EI: 1/0/0/0/1/0. Existing J/E: 1/1. Name too long \[1 valid new, 1 invalid old - no exception\]
6. NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. Invalid job \[1 invalid new - exception\]
7. NV/OV/OU/NI/OI/EI: 0/1/0/1/1/0. Existing J/E: 1/2. 2 invalid jobs \[1 valid old, 2 invalid: old and new - no exception\]
8. NV/OV/OU/NI/OI/EI: 0/1/0/0/0/1. Existing J/E: 1/2. Name 4001ch \[1 valid old, 1 invalid new for external table - no exception; also file had previously failed\]
9. NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. \[File already processed - exception\]

**2025 Note:** In October 2021 I published [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html), which proposes a new approach to scenario selection that tends to result in a larger number of scenarios, with clearer descriptions.

## Package Structure Diagram
<img src="/migrated_images/2016/09/TRAPIT-Testing-csd-1-2.png" alt="Package Structure Diagram" title="Package Structure Diagram" />

## Call Structure Table – TT_Emp_Batch.tt_AIP_Load_Emps
<img src="/migrated_images/2016/09/TRAPIT-CST-3.png" alt="Call Structure Table" title="Call Structure Table" />

## Test Output

<div class="scrollbox">
<pre>
TRAPIT TEST for TT_Emp_Batch.tt_AIP_Load_Emps
=============================================
employees.dat was not present to delete!

SCENARIO 1: NV/OV/OU/NI/OI/EI: 1/0/0/0/0/0. Existing J/E: 0/0. [1 valid new record from scratch] {
==================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      1
        }
        =
        GROUP File {
        ============
            Line
            ------------------------------------
            ,LN 1,EM 1,01-JAN-2010,IT_PROG,10000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table (No records)
        ===================================
        GROUP Employees Table (No records)
        ==================================
    OUTPUTS
    =======
        GROUP Employee: Actual = 1, Expected = 1 {
        ==========================================
            F?  Employee Id  Name  Email  Hired        Job      Salary  Updated
            -- ----------- ---- ----- ----------- ------- ------ -----------
                       1627  LN 1  EM 1   01-JAN-2010  IT_PROG  10000   11-SEP-2016
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Error: Actual = 0, Expected = 0: SUCCESS
        ==============================================
        GROUP Job Statistic: Actual = 1, Expected = 1 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                               3  LOAD_EMPS     employees_20160801.dat               1                  0                  0  11-SEP-2016  11-SEP-2016  S
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================
} 0 failed, of 4: SUCCESS
=========================

SCENARIO 2: NV/OV/OU/NI/OI/EI: 1/1/1/0/0/0. Existing J/E: 1/0. [3 valid records of each kind] {
===============================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      3
        }
        =
        GROUP File {
        ============
            Line
            -----------------------------------------
            ,LN 1,EM 1,01-JAN-2010,IT_PROG,10000
            1629,LN 2,EM 2,01-JAN-2010,IT_PROG,20000
            1630,LN 3U,EM 3,01-JAN-2010,IT_PROG,30000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                           5  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1629  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
                   1630  LN 3  EM 3   01-JAN-2010  IT_PROG   30000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 3, Expected = 3 {
        ==========================================
            F?  Employee Id  Name   Email  Hired        Job      Salary  Updated
            -- ----------- ----- ----- ----------- ------- ------ -----------
                       1629  LN 2   EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
                       1630  LN 3U  EM 3   01-JAN-2010  IT_PROG  30000   11-SEP-2016
                       1631  LN 1   EM 1   01-JAN-2010  IT_PROG  10000   11-SEP-2016
        } 0 failed, of 3: SUCCESS
        =========================
        GROUP Error: Actual = 0, Expected = 0: SUCCESS
        ==============================================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                               5  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
                               6  LOAD_EMPS     employees_20160801.dat               2                  0                  0  11-SEP-2016  11-SEP-2016  S
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================
} 0 failed, of 7: SUCCESS
=========================

SCENARIO 3: NV/OV/OU/NI/OI/EI: 0/0/0/0/1/0. Existing J/E: 1/1. Uid not found [1 invalid old - exception] {
==========================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      1
        }
        =
        GROUP File {
        ============
            Line
            --------------------------------------
            99,LN 1,EM 1,01-JAN-2010,IT_PROG,10000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                           8  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1633  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 1, Expected = 1 {
        ==========================================
            F?  Employee Id  Name  Email  Hired        Job      Salary  Updated
            -- ----------- ---- ----- ----------- ------- ------ -----------
                       1633  LN 2  EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Error: Actual = 1, Expected = 1 {
        =======================================
            F?  Job Statistic Id  ORA_ERR_TAG$  ORA_ERR_MESG$       ORA_ERR_OPTYP$  Employee Id  Name  Email  Hired        Job      Salary
            -- ---------------- ------------ ------------------ -------------- ----------- ---- ----- ----------- ------- ------
                               9                Employee not found  PK                       99  LN 1  EM 1   01-JAN-2010  IT_PROG  10000
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                               8  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
                               9  LOAD_EMPS     employees_20160801.dat               0                  0                  1  11-SEP-2016  11-SEP-2016  F
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 1, Expected = 1 {
        ===========================================
            F?  Message
            -- ------------------------------------------------------
                ORA-20001: Batch failed with too many invalid records!
        } 0 failed, of 1: SUCCESS
        =========================
} 0 failed, of 5: SUCCESS
=========================

SCENARIO 4: NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. Email too long [1 invalid new - exception] {
===========================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      1
        }
        =
        GROUP File {
        ============
            Line
            ------------------------------------------------------------------
            ,LN 1,EM 1123456789012345678901234567890,01-JAN-2010,IT_PROG,10000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =

        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                          11  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1635  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 1, Expected = 1 {
        ==========================================
            F?  Employee Id  Name  Email  Hired        Job      Salary  Updated
            -- ----------- ---- ----- ----------- ------- ------ -----------
                       1635  LN 2  EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Error: Actual = 1, Expected = 1 {
        =======================================
            F?  Job Statistic Id  ORA_ERR_TAG$  ORA_ERR_MESG$                                                                             ORA_ERR_OPTYP$  Employee Id  Name  Email                               Hired        Job      Salary
            -- ---------------- ------------ ---------------------------------------------------------------------------------------- -------------- ----------- ---- ---------------------------------- ----------- ------- ------
                              12                ORA-12899: value too large for column "HR"."EMPLOYEES"."EMAIL" (actual: 34, maximum: 25)  I                      1636  LN 1  EM 1123456789012345678901234567890  01-JAN-2010  IT_PROG  10000
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                              11  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
                              12  LOAD_EMPS     employees_20160801.dat               0                  0                  1  11-SEP-2016  11-SEP-2016  F
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 1, Expected = 1 {
        ===========================================
            F?  Message
            -- ------------------------------------------------------
                ORA-20001: Batch failed with too many invalid records!
        } 0 failed, of 1: SUCCESS
        =========================
} 0 failed, of 5: SUCCESS
=========================

SCENARIO 5: NV/OV/OU/NI/OI/EI: 1/0/0/0/1/0. Existing J/E: 1/1. Name too long [1 valid new, 1 invalid old - no exception] {
==========================================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      2
        }
        =
        GROUP File {
        ============
            Line
            ----------------------------------------------------------------------
            1638,LN 1123456789012345678901234567890,EM 1,01-JAN-2010,IT_PROG,10000
            ,LN 3,EM 3,01-JAN-2010,IT_PROG,30000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                          14  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1638  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 2, Expected = 2 {
        ==========================================
            F?  Employee Id  Name  Email  Hired        Job      Salary  Updated
            -- ----------- ---- ----- ----------- ------- ------ -----------
                       1638  LN 2  EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
                       1639  LN 3  EM 3   01-JAN-2010  IT_PROG  30000   11-SEP-2016
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Error: Actual = 1, Expected = 1 {
        =======================================
            F?  Job Statistic Id  ORA_ERR_TAG$  ORA_ERR_MESG$                                                                                 ORA_ERR_OPTYP$  Employee Id  Name                                Email  Hired        Job      Salary
            -- ---------------- ------------ -------------------------------------------------------------------------------------------- -------------- ----------- ---------------------------------- ----- ----------- ------- ------
                              15                ORA-12899: value too large for column "HR"."EMPLOYEES"."LAST_NAME" (actual: 34, maximum: 25)  U                      1638  LN 1123456789012345678901234567890  EM 1   01-JAN-2010  IT_PROG  10000
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                              14  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
                              15  LOAD_EMPS     employees_20160801.dat               1                  0                  1  11-SEP-2016  11-SEP-2016  S
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================
} 0 failed, of 6: SUCCESS
=========================

SCENARIO 6: NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. Invalid job [1 invalid new - exception] {
========================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      1
        }
        =
        GROUP File {
        ============
            Line
            ------------------------------------
            ,LN 1,EM 1,01-JAN-2010,NON_JOB,10000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                          17  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1641  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 1, Expected = 1 {
        ==========================================
            F?  Employee Id  Name  Email  Hired        Job      Salary  Updated
            -- ----------- ---- ----- ----------- ------- ------ -----------
                       1641  LN 2  EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Error: Actual = 1, Expected = 1 {
        =======================================
            F?  Job Statistic Id  ORA_ERR_TAG$  ORA_ERR_MESG$                                                                    ORA_ERR_OPTYP$  Employee Id  Name  Email  Hired        Job      Salary
            -- ---------------- ------------ ------------------------------------------------------------------------------- -------------- ----------- ---- ----- ----------- ------- ------
                              18                ORA-02291: integrity constraint (HR.EMP_JOB_FK) violated - parent key not found  I                      1642  LN 1  EM 1   01-JAN-2010  NON_JOB  10000
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                              17  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
                              18  LOAD_EMPS     employees_20160801.dat               0                  0                  1  11-SEP-2016  11-SEP-2016  F
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 1, Expected = 1 {
        ===========================================
            F?  Message
            -- ------------------------------------------------------
                ORA-20001: Batch failed with too many invalid records!
        } 0 failed, of 1: SUCCESS
        =========================
} 0 failed, of 5: SUCCESS
=========================

SCENARIO 7: NV/OV/OU/NI/OI/EI: 0/1/0/1/1/0. Existing J/E: 1/2. 2 invalid jobs [1 valid old, 2 invalid: old and new - no exception] {
====================================================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      3
        }
        =
        GROUP File {
        ============
            Line
            -----------------------------------------
            ,LN 1,EM 1,01-JAN-2010,NON_JOB,10000
            1644,LN 2,EM 2,01-JAN-2010,NON_JOB,20000
            1645,LN 3U,EM 3,01-JAN-2010,IT_PROG,30000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                          20  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1644  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
                   1645  LN 3  EM 3   01-JAN-2010  IT_PROG   30000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 2, Expected = 2 {
        ==========================================
            F?  Employee Id  Name   Email  Hired        Job      Salary  Updated
            -- ----------- ----- ----- ----------- ------- ------ -----------
                       1644  LN 2   EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
                       1645  LN 3U  EM 3   01-JAN-2010  IT_PROG  30000   11-SEP-2016
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Error: Actual = 2, Expected = 2 {
        =======================================
            F?  Job Statistic Id  ORA_ERR_TAG$  ORA_ERR_MESG$                                                                    ORA_ERR_OPTYP$  Employee Id  Name  Email  Hired        Job      Salary
            -- ---------------- ------------ ------------------------------------------------------------------------------- -------------- ----------- ---- ----- ----------- ------- ------
                              21                ORA-02291: integrity constraint (HR.EMP_JOB_FK) violated - parent key not found  U                      1644  LN 2  EM 2   01-JAN-2010  NON_JOB  20000
                              21                ORA-02291: integrity constraint (HR.EMP_JOB_FK) violated - parent key not found  I                      1646  LN 1  EM 1   01-JAN-2010  NON_JOB  10000
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                              20  LOAD_EMPS     employees_20160101.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
                              21  LOAD_EMPS     employees_20160801.dat               1                  0                  2  11-SEP-2016  11-SEP-2016  S
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================
} 0 failed, of 7: SUCCESS
=========================

SCENARIO 8: NV/OV/OU/NI/OI/EI: 0/1/0/0/0/1. Existing J/E: 1/2. Name 4001ch [1 valid old, 1 invalid new for external table - no exception; also file had previously failed] {
============================================================================================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      2
        }
        =
        GROUP File {
        ============
            Line
            --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
            ,123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567
            1649,LN 3U,EM 3,01-JAN-2010,IT_PROG,30000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                          23  LOAD_EMPS     employees_20160801.dat               0                  0                  2  01-JAN-2010  01-JAN-2010  F
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1648  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
                   1649  LN 3  EM 3   01-JAN-2010  IT_PROG   30000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 2, Expected = 2 {
        ==========================================
            F?  Employee Id  Name   Email  Hired        Job      Salary  Updated
            -- ----------- ----- ----- ----------- ------- ------ -----------
                       1648  LN 2   EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
                       1649  LN 3U  EM 3   01-JAN-2010  IT_PROG  30000   11-SEP-2016
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Error: Actual = 0, Expected = 0: SUCCESS
        ==============================================
        GROUP Job Statistic: Actual = 2, Expected = 2 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                              23  LOAD_EMPS     employees_20160801.dat               0                  0                  2  01-JAN-2010  01-JAN-2010  F
                              24  LOAD_EMPS     employees_20160801.dat               1                  1                  0  11-SEP-2016  11-SEP-2016  S
        } 0 failed, of 2: SUCCESS
        =========================
        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================
} 0 failed, of 6: SUCCESS
=========================

SCENARIO 9: NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. [File already processed - exception] {
=====================================================================================================

    INPUTS
    ======
        GROUP Parameter {
        =================
            File Name               Count
            ---------------------- -----
            employees_20160801.dat      1
        }
        =
        GROUP File {
        ============
            Line
            ------------------------------------
            ,LN 1,EM 1,01-JAN-2010,NON_JOB,10000
        }
        =
        GROUP Batch Job Table {
        =======================
            Name       Fail Percent
            --------- ------------
            LOAD_EMPS            70
        }
        =
        GROUP Statistics Table {
        ========================
            Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                          26  LOAD_EMPS     employees_20160801.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        }
        =
        GROUP Employees Table {
        =======================
            Employee Id  Name  Email  Hired        Job      Salary  Manager Id  Department Id  Updated
            ----------- ---- ----- ----------- ------- ------ ---------- ------------- -----------
                   1651  LN 2  EM 2   01-JAN-2010  IT_PROG   20000                             01-JAN-2010
        }
        =
    OUTPUTS
    =======
        GROUP Employee: Actual = 1, Expected = 1 {
        ==========================================
            F?  Employee Id  Name  Email  Hired        Job      Salary  Updated
            -- ----------- ---- ----- ----------- ------- ------ -----------
                       1651  LN 2  EM 2   01-JAN-2010  IT_PROG  20000   01-JAN-2010
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Error: Actual = 0, Expected = 0: SUCCESS
        ==============================================
        GROUP Job Statistic: Actual = 1, Expected = 1 {
        ===============================================
            F?  Job Statistic Id  Batch job Id  File Name               Records Loaded  Records Failed ET  Records Failed DB  Start Time   End Time     Status
            -- ---------------- ------------ ---------------------- -------------- ----------------- ----------------- ----------- ----------- ------
                              26  LOAD_EMPS     employees_20160801.dat              10                  0                  2  01-JAN-2010  01-JAN-2010  S
        } 0 failed, of 1: SUCCESS
        =========================
        GROUP Exception: Actual = 1, Expected = 1 {
        ===========================================
            F?  Message
            -- --------------------------------------------------------
                ORA-20002: File has already been processed successfully!
        } 0 failed, of 1: SUCCESS
        =========================
} 0 failed, of 4: SUCCESS
=========================

TIMING: Actual = 306, Expected <= 2: FAILURE
============================================

SUMMARY for TT_Emp_Batch.tt_AIP_Load_Emps
=========================================

Scenario                                                                                                                                                        # Failed  # Tests  Status
-------------------------------------------------------------------------------------------------------------------------------------------------------------- -------- ------- -------
NV/OV/OU/NI/OI/EI: 1/0/0/0/0/0. Existing J/E: 0/0. [1 valid new record from scratch]                                                                                   0        4  SUCCESS
NV/OV/OU/NI/OI/EI: 1/1/1/0/0/0. Existing J/E: 1/0. [3 valid records of each kind]                                                                                      0        7  SUCCESS
NV/OV/OU/NI/OI/EI: 0/0/0/0/1/0. Existing J/E: 1/1. Uid not found [1 invalid old - exception]                                                                           0        5  SUCCESS
NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. Email too long [1 invalid new - exception]                                                                          0        5  SUCCESS
NV/OV/OU/NI/OI/EI: 1/0/0/0/1/0. Existing J/E: 1/1. Name too long [1 valid new, 1 invalid old - no exception]                                                           0        6  SUCCESS
NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. Invalid job [1 invalid new - exception]                                                                             0        5  SUCCESS
NV/OV/OU/NI/OI/EI: 0/1/0/1/1/0. Existing J/E: 1/2. 2 invalid jobs [1 valid old, 2 invalid: old and new - no exception]                                                 0        7  SUCCESS
NV/OV/OU/NI/OI/EI: 0/1/0/0/0/1. Existing J/E: 1/2. Name 4001ch [1 valid old, 1 invalid new for external table - no exception; also file had previously failed]         0        6  SUCCESS
NV/OV/OU/NI/OI/EI: 0/0/0/1/0/0. Existing J/E: 1/1. [File already processed - exception]                                                                                0        4  SUCCESS
Timing                                                                                                                                                                 1        1  FAILURE
-------------------------------------------------------------------------------------------------------------------------------------------------------------- -------- ------- -------
Total                                                                                                                                                                  1       50  FAILURE
-------------------------------------------------------------------------------------------------------------------------------------------------------------- -------- ------- -------

Timer Set: TT_Emp_Batch.tt_AIP_Load_Emps, Constructed at 11 Sep 2016 16:10:46, written at 16:10:49
==================================================================================================
[Timer timed: Elapsed (per call): 0.04 (0.000036), CPU (per call): 0.03 (0.000030), calls: 1000, '***' denotes corrected line below]

Timer           Elapsed         CPU         Calls       Ela/Call       CPU/Call
----------- ---------- ---------- ------------ ------------- -------------
Setup              1.10        0.30             9        0.12189        0.03333
Caller             1.53        1.16             5        0.30560        0.23200
Get_Tab_Lis        0.75        0.49             9        0.08378        0.05444
Get_Err_Lis        0.02        0.03             9        0.00244        0.00333
Get_Jbs_Lis        0.00        0.00             9        0.00011        0.00000
(Other)            0.34        0.35             1        0.33500        0.35000
----------- ---------- ---------- ------------ ------------- -------------
Total              3.74        2.33            42        0.08898        0.05548
----------- ---------- ---------- ------------ ------------- -------------

Suite Summary
=============

Package.Procedure                  Tests  Fails         ELA         CPU
--------------------------------- ----- ----- ---------- ----------
TT_Emp_WS.tt_AIP_Save_Emps            17      3        0.10        0.09
TT_Emp_WS.tt_AIP_Get_Dept_Emps         7      1        0.11        0.12
TT_View_Drivers.tt_HR_Test_View_V      9      0        0.12        0.12
TT_Emp_Batch.tt_AIP_Load_Emps         50      1        3.74        2.33
--------------------------------- ----- ----- ---------- ----------
Total                                 83      5        4.07        2.66
--------------------------------- ----- ----- ---------- ----------
Others error in (): ORA-20001: Suite BRENDAN returned error status: ORA-06512: at "DP_1.UTILS_TT", line 151
ORA-06512: at "DP_1.UTILS_TT", line 818
ORA-06512: at line 5
</pre>
</div>

<div class="dbapit-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 DBAPIT Series:</strong>
  <a href="/migrated/dbapit-series-index/">Index</a>
  {% if page.dbapit_prev %}
    | <a href="{{ page.dbapit_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.dbapit_next %}
    | <a href="{{ page.dbapit_next }}">Next ▶</a>
  {% endif %}
</div>
