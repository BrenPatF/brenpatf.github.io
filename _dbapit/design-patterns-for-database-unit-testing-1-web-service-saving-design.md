---
title: "Design Patterns for Database API Testing 1: Web Service Saving 1 - Design"
date: 2016-05-08
categories: 
  - "design"
  - "plsql"
  - "sql"
  - "testing"
  - "utplsql"
tags: 
  - "oracle"
  - "plsql"
  - "unit-test"
  - "utplsql"
  - "web-service"
---

Last October I gave a presentation on database unit testing with utPLSQL, [Oracle Unit Testing with utPLSQL](http://aprogrammerwrites.eu/?p=1545). I mentioned design patterns as a way of reducing the effort of building unit tests and outlined some strategies for coding them effectively.

In the current set of articles, I develop the ideas further, starting from the idea that all database APIs can be considered in terms of the axes:

- direction (i.e. getter or setter, noting that setters can also 'get')
- mode (i.e. real time or batch)

For each cell in the implied matrix, I construct an example API (or view) with specified requirements against Oracle's HR demo schema, and use this example to construct a testing program with appropriate scenarios as a design pattern. Concepts and common patterns and anti-patterns in automated API testing are discussed throughout, and these are largely independent of testing framework used. However, the examples use my own lightweight independent framework that is designed to help avoid many API testing anti-patterns. The code is available on GitHub here, [BrenPatF/db\_unit\_test](https://github.com/BrenPatF/db_unit_test), and includes both framework and design pattern examples.

Behind the four examples, there is an underlying design pattern that involves wrapping the API call in a 'pure' procedure, called once per scenario, with the output 'actuals' array including everything affected by the API, whether as output parameters, or on database tables, etc. The inputs are also extended from the API parameters to include any other effective inputs. Assertion takes place after all scenarios and is against the extended outputs, with extended inputs also listed. This concept of the 'pure' function, central to [Functional Programming](https://en.wikipedia.org/wiki/Functional_programming), has important advantages in automated testing. I explained the concepts involved in a presentation at the Oracle User Group Ireland Conference in March 2018:

[The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing)

[![](images/Mode-R_S2.png)](http://aprogrammerwrites.eu/?attachment_id=2363) In this first example, I present a design pattern for web service 'save' procedures by means of a conceptual discussion, together with a working example of base code and test code for a procedure to save new employees in Oracle's well-known HR demonstration schema.

Design patterns involve abstraction and conceptual separation of general features of a situation from the particular. Therefore we will start with a fairly abstract discussion of automated API testing for the database, before proceeding to discuss the use case in question, describe the test cases, and show the results. The code is presented in another article, [Design Patterns for Database API Testing 1: Web Service Saving 2 - Code](http://aprogrammerwrites.eu/?p=1616). That article lists various extremely common antipatterns in database unit testing, and how they may be avoided. The code itself centralises as much as possible in order to make specific test code as small as possible, and is structured very differently from most unit testing code that I have seen.

This is the first in a series of four design patterns, with the second described again in two parts, starting here, [Design Patterns for Database API Testing 2: Views 1 - Design](http://aprogrammerwrites.eu/?p=1689). For an overview of the testing framework see [TRAPIT - TRansactional API Testing in Oracle](http://aprogrammerwrites.eu/?p=1723).

**General Discussion of Database Unit Testing**

The underlying functionality for unit testing could be described logically as:

- Given a list of test inputs, X and a list of expected outputs, E, for function F:
- For each x in X, with e in E:
    - Apply y = F(x)
    - Assert y = e

As the Functional Programming community knows well, functions having well-defined parameter inputs, returning values as outputs, and with no 'side-effects', are the easiest to test reliably. The difficulty with database unit testing is that most use cases do not fall into that category; instead, database procedures can read from and write to the database as well as using input and output parameters. This means that theoretically the inputs and outputs could include the whole database (at least); furthermore the database is a shared resource, so other parties can alter the data we are dealing with. One important consequence of these facts is that much of the thinking on best practices for unit testing, coming as it does from the non-database world, is not applicable here. So what to do?

_Pragmatic testing_

To make progress, we note that our purpose with unit testing is not to formally prove that our programs work, but rather includes the following aims:

- Improve code quality within a Test Driven Development (TDD) approach
- Provide regression tests to allow safe code-refactoring
- Detect quickly changes external to the code that cause it to fail, such as reference data changes

That being so, we can note the following guidelines:

- Testing code is written, as part of TDD, by the developer of the base code, who can identify the relevant database inputs and outputs
- Some, but not necessarily all, test data may be created in a setup step; static reference data that are required for the code to work usually should not be created in setup
- Testing code should be instrumented and logged liberally
- The base code should be timed and a time limit included in the testing; this will help to quickly identify issues such as necessary indexes being dropped
- Consideration should be given to running the unit test suites in performance and other instances

**Design Pattern Use Case for Web Service Save Procedure**

- Purpose of procedure is to save a set of new records to a database table
- Surrogate primary key is generated
- Input is an array of objects with the records to be saved
- Output is an array of objects containing the new primary key plus a description
- For records failing validation, zero is returned, plus the error message, and the valid records will still be saved

**ERD of Input and Output Data Structures in Relation to Scenarios**

[![Unit Testing - ERD](images/Unit-Testing-ERD.png)](http://aprogrammerwrites.eu/?attachment_id=1621)

- In the diagram, a scenario corresponds to a web service call with a set of input records
- The result of the call can be described as a set of output groups, each having a set of records
- In our case the output array and the base table form two output groups, with a global group for average call timing
- The logical diagram in terms of sets of records can be translated into an array structure diagram

[![Unit Testing - ASD](images/Unit-Testing-ASD.png)](http://aprogrammerwrites.eu/?attachment_id=1622)

If we follow a similarly generic approach at the coding level, it becomes very easy to extend a simple example by adding groups, fields and records as necessary.

**General Unit Test Design Process**

The design process involves two high level steps

- Identify a good set of scenarios with corresponding input records
- Identify the expected outputs for each output group identified, for each scenario (there will also be a global group, for timing)

**Design Pattern Scenarios**

[![Unit Testing - HR](images/Unit-Testing-HR.png)](http://aprogrammerwrites.eu/?attachment_id=1623)

The procedure inserts records in Oracle's HR employees table, and we identify four test scenarios:

1. Passing a single valid record
2. Passing a single invalid record
3. Trying to pass a record with an invalid type
4. Passing multiple valid records, and one invalid record

**Design Pattern Output Groups**

- Records inserted into table employees
- Records returned in output parameter array
- Timing of average call

**Test Results Output**

The output below is for a failing run, where the time limit is breached, and I also have deliberately entered incorrect expected values for two records, to show the difference in formatting between success and failure output group records. I like to include the output tables on completion of development in my technical design document.

```
TRAPIT TEST: TT_Emp_WS.tt_AIP_Save_Emps
=======================================

SCENARIO 1: 1 valid record {
============================

    INPUTS
    ======

        GROUP Employee {
        ================

            Name  Email  Job      Salary
            ---- ----- ------- ------
            LN 1  EM 1   IT_PROG    1000

        }
        =

    OUTPUTS
    =======

        GROUP Employee: Actual = 1, Expected = 1 {
        ==========================================

            F?  Employee id  Name  Email  Job      Salary
            -- ----------- ---- ----- ------- ------
                       1513  LN 1  EM 1   IT_PROG    1000

        } 0 failed, of 1: SUCCESS
        =========================

        GROUP Output array: Actual = 1, Expected = 1 {
        ==============================================

            F?  Employee id  Description
            -- ----------- ----------------------------------
                       1513  ONE THOUSAND FIVE HUNDRED THIRTEEN

        } 0 failed, of 1: SUCCESS
        =========================

        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================

} 0 failed, of 3: SUCCESS
=========================

SCENARIO 2: 1 invalid job id {
==============================

    INPUTS
    ======

        GROUP Employee {
        ================

            Name  Email  Job      Salary
            ---- ----- ------- ------
            LN 2  EM 2   NON_JOB    1500

        }
        =

    OUTPUTS
    =======

        GROUP Employee: Actual = 0, Expected = 0: SUCCESS
        =================================================

        GROUP Output array: Actual = 1, Expected = 1 {
        ==============================================

            F?  Employee id  Description
            -- ----------- -------------------------------------------------------------------
                          0  ORA-02291: integrity constraint (.) violated - parent key not found

        } 0 failed, of 1: SUCCESS
        =========================

        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================

} 0 failed, of 3: SUCCESS
=========================

SCENARIO 3: 1 invalid number {
==============================

    INPUTS
    ======

        GROUP Employee {
        ================

            Name  Email  Job      Salary
            ---- ----- ------- ------
            LN 3  EM 3   IT_PROG   2000x

        }
        =

    OUTPUTS
    =======

        GROUP Employee: Actual = 0, Expected = 0: SUCCESS
        =================================================

        GROUP Output array: Actual = 0, Expected = 0: SUCCESS
        =====================================================

        GROUP Exception: Actual = 1, Expected = 1 {
        ===========================================

            F?  Error message
            -- -------------------------------------------------------------------------------
                ORA-06502: PL/SQL: numeric or value error: character to number conversion error

        } 0 failed, of 1: SUCCESS
        =========================

} 0 failed, of 3: SUCCESS
=========================

SCENARIO 4: 2 valid records, 1 invalid job id (2 deliberate errors) {
=====================================================================

    INPUTS
    ======

        GROUP Employee {
        ================

            Name  Email  Job      Salary
            ---- ----- ------- ------
            LN 4  EM 4   IT_PROG    3000
            LN 5  EM 5   NON_JOB    4000
            LN 6  EM 6   IT_PROG    5000

        }
        =

    OUTPUTS
    =======

        GROUP Employee: Actual = 2, Expected = 3 {
        ==========================================

            F?  Employee id  Name  Email  Job      Salary
            -- ----------- ---- ----- ------- ------
            F          1515  LN 4  EM 4   IT_PROG    3000
            >          1515  LN 4  EM 4   IT_PROG    1000
                       1517  LN 6  EM 6   IT_PROG    5000
            F
            >          1517  LN 6  EM 6   IT_PROG    5000

        } 2 failed, of 3: FAILURE
        =========================

        GROUP Output array: Actual = 3, Expected = 3 {
        ==============================================

            F?  Employee id  Description
            -- ----------- -------------------------------------------------------------------
                       1515  ONE THOUSAND FIVE HUNDRED FIFTEEN
                          0  ORA-02291: integrity constraint (.) violated - parent key not found
                       1517  ONE THOUSAND FIVE HUNDRED SEVENTEEN

        } 0 failed, of 3: SUCCESS
        =========================

        GROUP Exception: Actual = 0, Expected = 0: SUCCESS
        ==================================================

} 2 failed, of 7: FAILURE
=========================

TIMING: Actual = 222, Expected <= 2: FAILURE
============================================

SUMMARY for TT_Emp_WS.tt_AIP_Save_Emps
======================================

Scenario                                                 # Failed  # Tests  Status
------------------------------------------------------- -------- ------- -------
1 valid record                                                  0        3  SUCCESS
1 invalid job id                                                0        3  SUCCESS
1 invalid number                                                0        3  SUCCESS
2 valid records, 1 invalid job id (2 deliberate errors)         2        7  FAILURE
Timing                                                          1        1  FAILURE
------------------------------------------------------- -------- ------- -------
Total                                                           3       17  FAILURE
------------------------------------------------------- -------- ------- -------

Timer Set: TT_Emp_WS.tt_AIP_Save_Emps, Constructed at 09 Jul 2016 13:32:40, written at 13:32:42
===============================================================================================
[Timer timed: Elapsed (per call): 0.01 (0.000013), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Setup          0.04        0.01             1        0.03700        0.01000
Caller         0.67        0.03             3        0.22200        0.01000
SELECT         0.60        0.00             3        0.19933        0.00000
(Other)        0.41        0.03             1        0.40500        0.03000
------- ---------- ---------- ------------ ------------- -------------
Total          1.71        0.07             8        0.21325        0.00875
------- ---------- ---------- ------------ ------------- -------------

```

_Notes on output_

- In the event of the suite failing, as here, the utility code ensures that an Oracle error is generated. This can then be trapped by a calling Unix script from a scheduled Jenkins job to send out emails

**Conclusions**

- A design pattern has been presented for database web service save procedures, with scenarios and output results
- The implementation (presented in the part 2 article) was against an Oracle database publicly available demonstration schema, and used Brendan's database API testing framework
- The main ideas could be applied with any database technology and any testing framework
- It is suggested that any proposed alternative testing framework can be compared by implementing this design pattern, or similar
- Three more design patterns are presented in further articles in this series, the second being for testing of views ([Design Patterns for Database Unit Testing 2: Views 1 - Design](http://aprogrammerwrites.eu/?p=1689))

<script type="text/javascript">var addthis_config = {"data_track_addressbar":true};</script>

<script type="text/javascript" src="//s7.addthis.com/js/300/addthis_widget.js#pubid=ra-50fd09b73778290b"></script>
