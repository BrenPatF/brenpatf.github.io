---
layout: post
title: "DBAPIT 2: Views"
date: 2016-05-21
migrated: true
categories: 
  - "design"
  - "oracle"
  - "plsql"
  - "sql"
  - "testing"
  - "utplsql"
  - "web-services"
tags: 
  - "analytics"
  - "design-pattern"
  - "oracle"
  - "plsql"
  - "sql"
  - "unit-test"
  - "utplsql"
  - "web-service"
dbapit_prev: /dbapit/dbapit-1/
dbapit_next: /dbapit/dbapit-3/
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
#### Part 2 in a series on: Design Patterns for Database API Testing

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

<img src="/migrated_images/2016/05/Mode-B_G.png" alt="Mode: View" title="Mode: View" />

In the second article in the series a design pattern is presented for testing database views.

We start by discussing when and how to test views. Unlike in the design paatern of the first article in the series, [DBAPIT 1: Web Service Saving](/dbapit/dbapit-1/), test data has to be created during testing of views, and a very general approach to creating and selecting the test data is proposed. The use case for the design pattern is described, and scenarios and subscenarios are defined conceptually. Next, the output from the testing is presented and code extracts are shown.

## When to Test Views

Views can be simple or complex, or, as I categorise them in [Brendan's 2-Page Oracle Programming Standards](https://brenpatf.github.io/migrated/brendans-2-page-oracle-programming-standards/), _thin_ or _thick_, where _thick_ views include table joins and _thin_ views don't. Thin views do not normally require testing while it may or may not be appropriate to test thick views.

As explained in the second part of the first article mentioned above, method-based testing is a bad idea, and occurs when the test suite is based on testing all the methods in a package, rather than units of (external) behaviour (often corresponding to procedures prefixed AIP in a common naming convention). Similarly, we can consider views in the same way as methods and ask whether they represent testable units of behaviour or are merely internal code structures, which should not normally have individual automated tests for the reasons given there.

Good examples of views that should be tested would be those that form the basis of complex data extraction to file, by ETL tools such as Informatica, or those that form the basis of reporting tools such as Business Objects. In fact, it is very good practice to place SQL for these tools into views precisely so that they can be tested.

## How to Test Views Using a PL/SQL Testing Framework

In order to leverage a PL/SQL API testing framework to also test views, the API test package procedures call a library procedure passing the name of the relevant view: The library procedure returns the result of querying the view as an array of delimited strings, and the API test procedures then compare the results against their own expected results.

Each API test procedure will have its own setup local procedure to create test data, and we need to discuss the issue of distinguishing test data from pre-existing data.

## Test Data

In the earlier article on database save procedures, we did not create any test data within the testing code itself, but the base procedure did create data, and those were queried back for assertion. In order to select only the data created by the procedure call a prefix was used in one of the string fields which was assumed not to exist already. This is a workable approach in some cases, but may not be possible in general. Let us consider the different types of database data that may affect our testing:

- Data created by the base code being tested
- Data created by test code to be read by the base code
- Data not created by test code to be read by base code

In order to verify that the program calls are giving results as expected, the test code needs to know all the data that influence the results, not necessarily just directly created data. Our view testing use case described below has an example where the results depend on an aggregate of all the records on the database. This is a problem when we have a shared database, where we cannot freeze the data at the time of test development. In order to handle this problem, we propose to borrow a technique used in Oracle's ebusinees applications.

### Partitioning Views with System Contexts

In Oracle ebusiness's multi-org implementations, transactions are partitioned by a numeric identifier for the organization owning the transaction. This org\_id value is stored in a column in the base table on transaction creation. Within the application code the base table is not queried directly, but through a view that restricts records returned to those for the organization corresponding to the current role of the application user, which is stored in the userenv system context (this is true up to release 11.5, but the mechanism changed in release 12.1).

See [SYS\_CONTEXT](http://docs.oracle.com/cd/E11882_01/server.112/e41084/functions184.htm#SQLRF06117) for information on the system context database feature, and [Oracle E-Business Suite Multiple Organizations Implementation Guide (12.1)](https://docs.oracle.com/cd/E18727_01/doc.121/e13423/T443823T443827.htm) for release 12.1 multi-org implementation in Oracle ebusiness.

### Partitioning Views for Testing

We propose to use views in a similar way to the multi-org views, to restrict records to those created in the testing session, by means of a ttid column on the base table that will hold the session id. The new optional column is added to those tables where this approach is required, and view are created on the tables. Our testing utility package Utils\_TT sets a context variable to the value 'TT' to signify testing mode, and the session id is set to a package variable in the general utilities package Utils.

Any base code that inserts data into the tables has to check for test mode, and if set, put the session id into the ttid field, and if not, leave it blank. The views use the following clause:

```sql
 WHERE (ttid = SYS_Context ('userenv', 'sessionid') OR 
        Substr (Nvl (SYS_Context ('userenv', 'client_info'), 'XX'), 1, 2) != 'TT')
```

Both test code and base code now query the views instead of the base tables. As the base code to write to the tables has to account for the new column, it is necessary for the column to be added in all instances including production. If this seems a little drastic, consider the importance that you attach to testing, and bear in mind that the earlier, less general, approaches may suffice in many cases. In these design pattern demos I use the general solution.

## Design Pattern Use Case for Testing Views

Modern Oracle SQL is very powerful and can apply complex logic within a single statement, reducing the need for more complex procedural code. In order to show how to test SQL, we will devise a test view, HR\_Test\_V, having a range of features that we might want to test in general:

- Inner joins suppress driving records where there is no joining record
- Outer joins return driving records where there is no joining record
- Analytic functions that partition by some key, and return aggregates on the returned record set
- Functions based on aggregates over records that include those not in the returned record set
- Constraints based on aggregates over records that include those not in the returned record set
- Constraints on column values

The view functionality can be described in words as:

- Selected values
    - Employee name, department name, and salary
    - Manager's name
    - Ratio of employee's salary to the department average (returned employees only)
    - Ratio of employee's salary to the average salary of all employees
- Constraints
    - Exclude employees in job 'AD\_ASST'
    - Exclude employees without a department
    - Do not return any records if the total salary of all employees is below 1600
- Outer join
    - Include employees both with and without a manager

The view SQL is:

```sql
CREATE OR REPLACE VIEW hr_test_view_v AS
WITH all_emps AS (
        SELECT Avg (salary) avg_sal, SUM (salary) sal_tot_g
          FROM employees e
)
SELECT e.last_name, d.department_name, m.last_name manager, e.salary,
       Round (e.salary / Avg (e.salary) OVER (PARTITION BY e.department_id), 2) sal_rat,
       Round (e.salary / a.avg_sal, 2) sal_rat_g
  FROM all_emps a
 CROSS JOIN employees e
  JOIN departments d
    ON d.department_id = e.department_id
  LEFT JOIN employees m
    ON m.employee_id = e.manager_id
 WHERE e.job_id != 'AD_ASST'
   AND a.sal_tot_g >= 1600
```

## Scenarios

In our use case we can create employees with and without a department, and with and without a manager in the same scenario to test the different types of join. We also test several other scenarios.

**2025 Note:** In October 2021 I published [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html), which proposes a new approach to scenario selection that tends to result in a larger number of scenarios, with clearer descriptions.

## View Test Output

### Data setup section

```
SCENARIO 1: DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep, Employees created in setup: DS-1 - 4 emps, 1 dep (10), emp-3 has no dep, emp-4 has bad job
=========================================================================================================================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1493             10              IT_PROG           1000
2         1494             10        1493  IT_PROG           2000
3         1495                       1493  IT_PROG           3000
4         1496             10        1493  AD_ASST           4000

SCENARIO 2: DS-2, testing same as 1 but with extra emp in another dep, Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
===============================================================================================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1497             10              IT_PROG           1000
2         1498             10        1497  IT_PROG           2000
3         1499                       1497  IT_PROG           3000
4         1500             10        1497  AD_ASST           4000
5         1501             20        1497  IT_PROG           5000

SCENARIO 3: DS-2, passing 'WHERE dep=10', Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
==================================================================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1502             10              IT_PROG           1000
2         1503             10        1502  IT_PROG           2000
3         1504                       1502  IT_PROG           3000
4         1505             10        1502  AD_ASST           4000
5         1506             20        1502  IT_PROG           5000

SCENARIO 4: DS-3, Salaries total 1500 (< threshold of 1600), Employees created in setup: DS-3 - As dataset 2 but with salaries * 0.1, total below reporting threshold of 1600
=============================================================================================================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1507             10              IT_PROG            100
2         1508             10        1507  IT_PROG            200
3         1509                       1507  IT_PROG            300
4         1510             10        1507  AD_ASST            400
5         1511             20        1507  IT_PROG            500
```

### Notes on data setup section

- There are three data sets, and four scenarios, each of which references a data set
- The call to set up the data for a scenario writes out all the data created
- A header provides a description of the features (or _subscenarios_) in the data set
- In the output above scenarios 2 and 3 use the same data set, DS-2

### Results section

<div class="scrollbox">
<pre>
SQL>BEGIN

  Utils.Clear_Log;
  Utils_TT.Run_Suite (Utils_TT.c_tt_suite_bren);

EXCEPTION
  WHEN OTHERS THEN
    Utils.Write_Other_Error;
END;
/
SQL> @L_Log_Default

TEXT
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

TRAPIT TEST: TT_View_Drivers.tt_HR_Test_View_V
==============================================

Employees created in setup: DS-1 - 4 emps, 1 dep (10), emp-3 has no dep, emp-4 has bad job
==========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1518             10              IT_PROG           1000
2         1519             10        1518  IT_PROG           2000
3         1520                       1518  IT_PROG           3000
4         1521             10        1518  AD_ASST           4000

Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1522             10              IT_PROG           1000
2         1523             10        1522  IT_PROG           2000
3         1524                       1522  IT_PROG           3000
4         1525             10        1522  AD_ASST           4000
5         1526             20        1522  IT_PROG           5000

Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1527             10              IT_PROG           1000
2         1528             10        1527  IT_PROG           2000
3         1529                       1527  IT_PROG           3000
4         1530             10        1527  AD_ASST           4000
5         1531             20        1527  IT_PROG           5000

Employees created in setup: DS-3 - As dataset 2 but with salaries * 0.1, total below reporting threshold of 1600
================================================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1532             10              IT_PROG            100
2         1533             10        1532  IT_PROG            200
3         1534                       1532  IT_PROG            300
4         1535             10        1532  AD_ASST            400
5         1536             20        1532  IT_PROG            500

SCENARIO 1: DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep {
===============================================================================================
    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date  Job      Salary  Manager Id  department Id
            ----------- --------- ----- --------- ------- ------ ---------- -------------
                   1518  LN_1       EM_1   09-JUL-16  IT_PROG    1000                         10
                   1519  LN_2       EM_2   09-JUL-16  IT_PROG    2000        1518             10
                   1520  LN_3       EM_3   09-JUL-16  IT_PROG    3000        1518
                   1521  LN_4       EM_4   09-JUL-16  AD_ASST    4000        1518             10
        }
        =
        GROUP Where {
        =============
            Where
            -----
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 2, Expected = 2 {
        ================================================
            F?  Name  Department      Manager  Salary  Salary Ratio (dep)  Salary Ratio (overall)
            -- ---- -------------- ------- ------ ------------------ ----------------------
                LN_1  Administration             1000                 .67                      .4
                LN_2  Administration  LN_1       2000                1.33                      .8
        } 0 failed, of 2: SUCCESS
        =========================
} 0 failed, of 2: SUCCESS
=========================

SCENARIO 2: DS-2, testing same as 1 but with extra emp in another dep {
=======================================================================
    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date  Job      Salary  Manager Id  department Id
            ----------- --------- ----- --------- ------- ------ ---------- -------------
                   1522  LN_1       EM_1   09-JUL-16  IT_PROG    1000                         10
                   1523  LN_2       EM_2   09-JUL-16  IT_PROG    2000        1522             10
                   1524  LN_3       EM_3   09-JUL-16  IT_PROG    3000        1522
                   1525  LN_4       EM_4   09-JUL-16  AD_ASST    4000        1522             10
                   1526  LN_5       EM_5   09-JUL-16  IT_PROG    5000        1522             20
        }
        =
        GROUP Where {
        =============
            Where
            -----
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 3, Expected = 3 {
        ================================================
            F?  Name  Department      Manager  Salary  Salary Ratio (dep)  Salary Ratio (overall)
            -- ---- -------------- ------- ------ ------------------ ----------------------
                LN_1  Administration             1000                 .67                     .33
                LN_2  Administration  LN_1       2000                1.33                     .67
                LN_5  Marketing       LN_1       5000                   1                    1.67
        } 0 failed, of 3: SUCCESS
        =========================
} 0 failed, of 3: SUCCESS
=========================

SCENARIO 3: DS-2, passing 'WHERE dep=10' {
==========================================
    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date  Job      Salary  Manager Id  department Id
            ----------- --------- ----- --------- ------- ------ ---------- -------------
                   1527  LN_1       EM_1   09-JUL-16  IT_PROG    1000                         10
                   1528  LN_2       EM_2   09-JUL-16  IT_PROG    2000        1527             10
                   1529  LN_3       EM_3   09-JUL-16  IT_PROG    3000        1527
                   1530  LN_4       EM_4   09-JUL-16  AD_ASST    4000        1527             10
                   1531  LN_5       EM_5   09-JUL-16  IT_PROG    5000        1527             20
        }
        =
        GROUP Where {
        =============
            Where
            --------------------------------
            department_name='Administration'
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 2, Expected = 2 {
        ================================================
            F?  Name  Department      Manager  Salary  Salary Ratio (dep)  Salary Ratio (overall)
            -- ---- -------------- ------- ------ ------------------ ----------------------
                LN_1  Administration             1000                 .67                     .33
                LN_2  Administration  LN_1       2000                1.33                     .67
        } 0 failed, of 2: SUCCESS
        =========================
} 0 failed, of 2: SUCCESS
=========================

SCENARIO 4: DS-3, Salaries total 1500 (< threshold of 1600) {
=============================================================
    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date  Job      Salary  Manager Id  department Id
            ----------- --------- ----- --------- ------- ------ ---------- -------------
                   1532  LN_1       EM_1   09-JUL-16  IT_PROG     100                         10
                   1533  LN_2       EM_2   09-JUL-16  IT_PROG     200        1532             10
                   1534  LN_3       EM_3   09-JUL-16  IT_PROG     300        1532
                   1535  LN_4       EM_4   09-JUL-16  AD_ASST     400        1532             10
                   1536  LN_5       EM_5   09-JUL-16  IT_PROG     500        1532             20
        }
        =
        GROUP Where {
        =============
            Where
            -----
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 0, Expected = 0: SUCCESS
        =======================================================
} 0 failed, of 1: SUCCESS
=========================

TIMING: Actual = 48, Expected <= 1: FAILURE
===========================================

SUMMARY for TT_View_Drivers.tt_HR_Test_View_V
=============================================

Scenario                                                                           # Failed  # Tests  Status
--------------------------------------------------------------------------------- -------- ------- -------
DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep         0        2  SUCCESS
DS-2, testing same as 1 but with extra emp in another dep                                 0        3  SUCCESS
DS-2, passing 'WHERE dep=10'                                                              0        2  SUCCESS
DS-3, Salaries total 1500 (< threshold of 1600)                                           0        1  SUCCESS
Timing                                                                                    1        1  FAILURE
--------------------------------------------------------------------------------- -------- ------- -------
Total                                                                                     1        9  FAILURE
--------------------------------------------------------------------------------- -------- ------- -------

Timer Set: TT_View_Drivers.tt_HR_Test_View_V, Constructed at 09 Jul 2016 13:32:42, written at 13:32:42
======================================================================================================
[Timer timed: Elapsed (per call): 0.01 (0.000013), CPU (per call): 0.02 (0.000020), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Setup          0.11        0.00             4        0.02675        0.00000
Caller         0.19        0.06             4        0.04750        0.01500
(Other)        0.03        0.03             1        0.02700        0.03000
------- ---------- ---------- ------------ ------------- -------------
Total          0.32        0.09             9        0.03600        0.01000
------- ---------- ---------- ------------ ------------- -------------
</pre>
</div>

### Notes on results section

- In a view test there is only one group, namely the selected data set

The second part of the article is here: [Design Patterns for Database Unit Testing 2: Views 2 - Code](http://aprogrammerwrites.eu/?p=1691)

In the first part of this two part article,[Design Patterns for Database API Testing 2: Views 1 - Design](http://aprogrammerwrites.eu/?p=1689) I presented a design pattern for unit testing views, using an example based on Oracle's HR demo schema, and here I list the code for the main test procedure and a couple of the utility procedures, with notes.

A structure diagram shows how the PL/SQL packages relate to each other, and sections of the code are listed with notes.

## Package Structure Diagram
 <img src="/migrated_images/2016/05/TRAPIT-Testing-csd-1-2.png" alt="Package Structure Diagram" title="Package Structure Diagram" />

## Call Structure Table
<img src="/migrated_images/2016/05/TRAPIT-CST-2.png" alt="Call Structure Table" title="Call Structure Table" />

## TT\_View\_Drivers.tt\_HR\_Test\_View\_V - View Test Procedure

### Declare section

<div class="scrollbox">
<pre>
PROCEDURE tt_HR_Test_View_V IS
  c_view_name           CONSTANT VARCHAR2(61) := 'HR_Test_View_V';
  c_proc_name           CONSTANT VARCHAR2(61) := 'TT_View_Drivers.tt_' || c_view_name;
  c_dep_id_1            CONSTANT PLS_INTEGER := 10;
  c_dep_id_2            CONSTANT PLS_INTEGER := 20;
  c_dep_nm_1            CONSTANT VARCHAR2(100) := 'Administration';
  c_dep_nm_2            CONSTANT VARCHAR2(100) := 'Marketing';
  c_job_bad             CONSTANT VARCHAR2(100) := 'AD_ASST';
  c_job_good            CONSTANT VARCHAR2(100) := 'IT_PROG';
  c_base_sal            CONSTANT PLS_INTEGER := 1000;
  c_ln_pre              CONSTANT VARCHAR2(10) := DML_API_TT_HR.c_ln_pre;
  c_sel_lis             CONSTANT L1_chr_arr := L1_chr_arr ('last_name', 'department_name', 'manager', 'salary', 'sal_rat', 'sal_rat_g');
  c_where_lis           CONSTANT L1_chr_arr := L1_chr_arr (NULL, NULL, 'department_name=''Administration''', NULL);
  c_dataset_3lis        CONSTANT L3_chr_arr := L3_chr_arr (
                             L2_chr_arr (L1_chr_arr ('4 emps, 1 dep (10), emp-3 has no dep, emp-4 has bad job'),
-- dep           job          salary
                               L1_chr_arr (c_dep_id_1,   c_job_good,  '1000'),
                               L1_chr_arr (c_dep_id_1,   c_job_good,  '2000'),
                               L1_chr_arr (NULL,         c_job_good,  '3000'),
                               L1_chr_arr (c_dep_id_1,   c_job_bad,   '4000')
                                             ),
                             L2_chr_arr (L1_chr_arr ('As dataset 1 but with extra emp-5, in second dep (20)'),
                               L1_chr_arr (c_dep_id_1,   c_job_good,  '1000'),
                               L1_chr_arr (c_dep_id_1,   c_job_good,  '2000'),
                               L1_chr_arr (NULL,         c_job_good,  '3000'),
                               L1_chr_arr (c_dep_id_1,   c_job_bad,   '4000'),
                               L1_chr_arr (c_dep_id_2,   c_job_good,  '5000')
                                             ),
                             L2_chr_arr (L1_chr_arr ('As dataset 2 but with salaries * 0.1, total below reporting threshold of 1600'),
                               L1_chr_arr (c_dep_id_1,   c_job_good,  '100'),
                               L1_chr_arr (c_dep_id_1,   c_job_good,  '200'),
                               L1_chr_arr (NULL,         c_job_good,  '300'),
                               L1_chr_arr (c_dep_id_1,   c_job_bad,   '400'),
                               L1_chr_arr (c_dep_id_2,   c_job_good,  '500')
                                             )
                        );
  c_exp_2lis            CONSTANT L2_chr_arr := L2_chr_arr (
                                               L1_chr_arr (
                                       Utils.List_Delim (c_ln_pre || '1',   c_dep_nm_1, NULL,            '1000', '.67',   '.4'),
                                       Utils.List_Delim (c_ln_pre || '2',   c_dep_nm_1, c_ln_pre || '1', '2000',  '1.33', '.8')
                                               ),
                                               L1_chr_arr (
                                       Utils.List_Delim (c_ln_pre || '1',   c_dep_nm_1, NULL,            '1000', '.67',  '.33'),
                                       Utils.List_Delim (c_ln_pre || '2',   c_dep_nm_1, c_ln_pre || '1', '2000',  '1.33', '.67'),
                                       Utils.List_Delim (c_ln_pre || '5',   c_dep_nm_2, c_ln_pre || '1', '5000',  '1',    '1.67')
                                               ),
                                               L1_chr_arr (
                                       Utils.List_Delim (c_ln_pre || '1',   c_dep_nm_1, NULL,            '1000', '.67',   '.33'),
                                       Utils.List_Delim (c_ln_pre || '2',   c_dep_nm_1, c_ln_pre || '1', '2000',  '1.33', '.67')
                                               ),
                                               tt_Utils.c_empty_list
                        );
  c_scenario_ds_lis     CONSTANT L1_num_arr := L1_num_arr (1, 2, 2, 3);
  c_scenario_lis        CONSTANT L1_chr_arr := L1_chr_arr (
                               'DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep',
                               'DS-2, testing same as 1 but with extra emp in another dep',
                               'DS-2, passing ''WHERE dep=10''',
                               'DS-3, Salaries total 1500 (< threshold of 1600)');
  c_inp_group_lis       CONSTANT L1_chr_arr := L1_chr_arr ('Employee', 'Where');
  c_inp_field_2lis      CONSTANT L2_chr_arr := L2_chr_arr (
                                                        L1_chr_arr (
                                                                '*Employee Id',
                                                                'Last Name',
                                                                'Email',
                                                                'Hire Date',
                                                                'Job',
                                                                '*Salary',
                                                                '*Manager Id',
                                                                '*department Id'),
                                                        L1_chr_arr (
                                                                'Where')
  );
  c_out_field_2lis      CONSTANT L2_chr_arr :=  L2_chr_arr ( L1_chr_arr (
                                'Name',
                                'Department',
                                'Manager',
                                '*Salary',
                                '*Salary Ratio (dep)',
                                '*Salary Ratio (overall)'));
  l_act_2lis                      L2_chr_arr := L2_chr_arr();
  c_ms_limit            CONSTANT PLS_INTEGER := 1;
  l_timer_set                    PLS_INTEGER;
  l_inp_3lis                     L3_chr_arr := L3_chr_arr();
</pre>
</div>

### Notes on declare section

- Data sets, scenarios, expected values etc. are stored in generic arrays, where:
    - L1\_chr\_arr is type of array of VARCHAR2(4000), same as standard type SYS.ODCIVarchar2List
    - L2\_chr\_arr is a type of array of L1\_chr\_arr
    - L3\_chr\_arr is a type of array of L2\_chr\_arr

### Setup section

```sql
  PROCEDURE Setup_Array IS
  BEGIN
    l_act_2lis.EXTEND (c_exp_2lis.COUNT);
    l_inp_3lis.EXTEND (c_exp_2lis.COUNT);
    FOR i IN 1..c_exp_2lis.COUNT LOOP
      l_inp_3lis (i) := L2_chr_arr();
      l_inp_3lis (i).EXTEND(2);
      l_inp_3lis(i)(2) := L1_chr_arr (c_where_lis(i));
    END LOOP;
  END Setup_Array;

  /***************************************************************************************************
  Setup_DB: Create test records for a given scenario for testing view
  ***************************************************************************************************/
  PROCEDURE Setup_DB (p_call_ind           PLS_INTEGER,   -- scenario index
                      x_inp_lis        OUT L1_chr_arr) IS -- input list, first group, employees
    l_emp_id            PLS_INTEGER;
    l_mgr_id            PLS_INTEGER;
    l_len_lis           L1_num_arr := L1_num_arr (1, -11, -13, -10, 10, -10);
  BEGIN
    Utils.Heading ('Employees created in setup: DS-' || p_call_ind || ' - ' || c_dataset_3lis (p_call_ind)(1)(1));
    Utils.Col_Headers (L1_chr_arr ('#', 'Employee id', 'Department id', 'Manager', 'Job id', 'Salary'), l_len_lis);
    x_inp_lis := L1_chr_arr();
    x_inp_lis.EXTEND (c_dataset_3lis (p_call_ind).COUNT - 1);
    FOR i IN 2..c_dataset_3lis (p_call_ind).COUNT LOOP
      l_emp_id := DML_API_TT_HR.Ins_Emp (
                            p_emp_ind  => i - 1,
                            p_dep_id   => c_dataset_3lis (p_call_ind)(i)(1),
                            p_mgr_id   => l_mgr_id,
                            p_job_id   => c_dataset_3lis (p_call_ind)(i)(2),
                            p_salary   => c_dataset_3lis (p_call_ind)(i)(3),
                            x_rec      => x_inp_lis(i - 1));
      Utils.Pr_List_As_Line (L1_chr_arr ((i-1), l_emp_id, Nvl (c_dataset_3lis (p_call_ind)(i)(1), ' '), Nvl (To_Char(l_mgr_id), ' '), c_dataset_3lis (p_call_ind)(i)(2), c_dataset_3lis (p_call_ind)(i)(3)), l_len_lis);
      IF i = 2 THEN
        l_mgr_id := l_emp_id;
      END IF;
    END LOOP;
  END Setup_DB;
```

### Notes on setup section

- c\_dataset\_3lis contains the data for all data sets indexed by (data set, record, field)
- Setup\_Array is called once to do some setup on the arrays
- Setup\_DB is called for a single data set at a time in each scenario
- Description of the data set is contained in the array and printed out
- Data set is printed out in tabular format. In the most recent version of the utility code, this is not strictly necessary, because all the input data is printed out before the outputs

### 'Pure' API wrapper procedure

```sql
  PROCEDURE Purely_Wrap_API (p_scenario_ds      VARCHAR2,      -- index of input dataset
                             p_where            VARCHAR2,      -- input where clause for
                             x_inp_lis_1    OUT L1_chr_arr,    -- first input group, employees
                             x_act_lis      OUT L1_chr_arr) IS -- generic actual values list (for scenario)
  BEGIN
    Setup_DB (p_scenario_ds, x_inp_lis_1);
    Timer_Set.Increment_Time (l_timer_set, Utils_TT.c_setup_timer);
     x_act_lis := Utils_TT.Get_View (
                            p_view_name         => c_view_name,
                            p_sel_field_lis     => c_sel_lis,
                            p_where             => p_where,
                            p_timer_set         => l_timer_set);
    ROLLBACK;
  END Purely_Wrap_API;
```

### Notes on 'Pure' API wrapper procedure

- Setup\_DB is called to create the data set for the scenario
- Get\_View returns the results of the query on the view as 2-level array
- Get\_View rolls back after getting the results, so the inserted test records are removed from the database

### Main section

```sql
BEGIN
--
-- Every testing main section should be similar to this, with array setup, then loop over scenarios
-- making a 'pure'(-ish) call to specific, local Purely_Wrap_API, with single assertion call outside
-- the loop
--
  l_timer_set := Utils_TT.Init (c_proc_name);
  Setup_Array;
  FOR i IN 1..c_exp_2lis.COUNT LOOP
    Purely_Wrap_API (c_scenario_ds_lis(i), c_where_lis(i), l_inp_3lis(i)(1), l_act_2lis(i));
  END LOOP;
  Utils_TT.Is_Deeply (c_proc_name, c_scenario_lis, l_inp_3lis, l_act_2lis, c_exp_2lis, l_timer_set, c_ms_limit,
                      c_inp_group_lis, c_inp_field_2lis, c_out_group_lis, c_out_field_2lis);
EXCEPTION
  WHEN OTHERS THEN
    Utils.Write_Other_Error;
    RAISE;
END tt_HR_Test_View_V;
```

### Notes on main section

- It's quite short isn't it? 😊
- Setup is called to do array setup
- Main section loops over the scenarios calling Purely\_Wrap\_API
- Is\_Deeply is called to do all the assertions within nested loops, then print the results

**2025 Note:** In the latest version of the unit testing framework, [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester), the main section is in a library module.

## Utils\_TT - Test Utility Procedures
 We will include only one procedure from this package in the body of the article. See gitHub link for the full code. ### Is\_Deeply - to check results from testing

```sql
PROCEDURE Is_Deeply (p_proc_name                 VARCHAR2,      -- calling procedure
                     p_test_lis                  L1_chr_arr,    -- test descriptions
                     p_inp_3lis                  L3_chr_arr,    -- actual result strings
                     p_act_3lis                  L3_chr_arr,    -- actual result strings
                     p_exp_3lis                  L3_chr_arr,    -- expected result strings
                     p_timer_set                 PLS_INTEGER,   -- timer set index
                     p_ms_limit                  PLS_INTEGER,   -- call time limit in ms
                     p_inp_group_lis             L1_chr_arr,    -- input group names
                     p_inp_fields_2lis           L2_chr_arr,    -- input fields descriptions
                     p_out_group_lis             L1_chr_arr,    -- output group names
                     p_fields_2lis               L2_chr_arr) IS -- test fields descriptions

  l_num_fails_sce                L1_num_arr :=  L1_num_arr();
  l_num_tests_sce                L1_num_arr :=  L1_num_arr();
  l_tot_fails                    PLS_INTEGER := 0;
  l_tot_tests                    PLS_INTEGER := 0;
.
.
.
(private procedures - see gitHub project, https://github.com/BrenPatF/trapit_oracle_tester, for full code listings)
.
.
.
BEGIN
  Detail_Section (l_num_fails_sce, l_num_tests_sce);
  Summary_Section (l_num_fails_sce, l_num_tests_sce, l_tot_fails, l_tot_tests);
  Set_Global_Summary (l_tot_fails, l_tot_tests + 1);
END Is_Deeply;
```

### Notes on Is\_Deeply

- This is the base version of Is\_Deeply with 3-level arrays of expected and actuals
- An inner loop asserts actual values (which are records) against expected
- The final assertion is against average call time
- It is expected that all assertion within a test procedure will be via a single call to one of the versions of this procedure, making a big reduction in code compared with traditional unit testing approaches
- After final assertion a call is made to write out all the results, by scenario, with all inputs printed first, followed by actuals (and expected, where they differ); this means that the test outputs now become precise and accurate documents of what the program does

### Get\_View - run a query dynamically on a view and return result set as array of strings

```sql
FUNCTION Get_View (p_view_name         VARCHAR2,               -- name of view
                   p_sel_field_lis     L1_chr_arr,             -- list of fields to select
                   p_where             VARCHAR2 DEFAULT NULL,  -- optional where clause
                   p_timer_set         PLS_INTEGER)            -- timer set handle
                   RETURN              L1_chr_arr IS           -- list of delimited result records
  l_cur            SYS_REFCURSOR;
  l_sql_txt        VARCHAR2(32767) := 'SELECT Utils.List_Delim (L1_chr_arr (';
  l_result_lis     L1_chr_arr;
  l_len            PLS_INTEGER;

BEGIN
  FOR i IN 1..p_sel_field_lis.COUNT LOOP
    l_sql_txt := l_sql_txt || p_sel_field_lis(i) || ',';
  END LOOP;
  l_sql_txt := RTrim (l_sql_txt, ',') || ')) FROM ' || p_view_name || ' WHERE ' || Nvl (p_where, '1=1 ') || 'ORDER BY 1';

  OPEN l_cur FOR l_sql_txt;
  FETCH l_cur BULK COLLECT -- ut, small result set, hence no need for limit clause
   INTO l_result_lis;
  CLOSE l_cur;

  Timer_Set.Increment_Time (p_timer_set, Utils_TT.c_call_timer);
  ROLLBACK;
  RETURN tt_Utils.List_or_Empty (l_result_lis);
END Get_View;
```

### Notes on Get\_View

- A query string is constructed from the input list of fields and optional where clause
- The fields are concatenated with delimiters and returned into an array of strings
- A rollback occurs to remove any test data created, so as not to interfere with any subsequent call
- If no data was returned from the query, we return a default 1-record listing containing the string 'EMPTY'

### Package block section

```sql
BEGIN

  DBMS_Application_Info.Set_Client_Info (client_info => 'TT');
  Utils.c_session_id_if_TT := SYS_Context ('userenv', 'sessionid');

END Utils_TT;
```

### Notes on package block section

- client\_info is set to 'TT', meaning the session operates in test mode
- The session id is stored in a package variable
- This id is referenced in the testing views and in insertion of test records with utid column

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
