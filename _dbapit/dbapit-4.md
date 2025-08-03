---
layout: post
title: "DBAPIT 4: REF Cursor Getter"
date: 2016-10-01
migrated: true
categories: 
  - "design"
  - "oracle"
  - "plsql"
  - "sql"
  - "testing"
tags: 
  - "design-pattern"
  - "oracle"
  - "plsql"
  - "sql"
  - "unit-test"
  - "web-service"
dbapit_prev: /dbapit/dbapit-3/
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
#### Part 4 in a series on: Design Patterns for Database API Testing

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

<img src="/migrated_images/2016/10/Mode-R_G.png" alt="Mode: Cursor Getter" title="Mode: Cursor Getter" />

In the fourth article in the series a design pattern is presented for testing procedures returning REF cursors, such as might be used by a web service or a report.

## Requirement Summary

Modern Oracle SQL is very powerful and can apply complex logic within a single statement, reducing the need for more complex procedural code. In order to show how to API test SQL that might be used in a batch getter module, we previously devised a test view, HR\_Test\_V, having a range of features that we might want to test in general. We can use similar SQL to demonstrate API testing of a real time getter procedure that might be used by a web service, where reference cursors are often used as output parameters. The following list of features to test is taken, in slightly modified form, from [DBAPIT 2: Views](/dbapit/dbapit-2/).

- Inner joins suppress driving records where there is no joining record
- Outer joins return driving records where there is no joining record
- Analytic functions that partition by some key, and return aggregates on the returned record set
- Functions based on aggregates over records that include those not in the returned record set
- Constraints based on aggregates over records that include those not in the returned record set
- Constraints on column values

The SQL functionality can be described in words as:

- Selected values
    - Employee name, department name, and salary
    - Manager's name
    - Ratio of employee's salary to the department average (returned employees only)
    - Ratio of employee's salary to the average salary of all employees
- Constraints
    - Exclude employees in job 'AD\_ASST'
    - Return employees for a department passed as a bind parameter
    - Do not return any records if the total salary of all employees is below 1600
- Outer join
    - Include employees both with and without a manager

The REF cursor SQL is:

```sql
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
   AND d.department_id = :1
```

## Notes on API Testing REF Cursor Procedures

- A new utility function has been added to Utils\_TT, Cursor\_to\_Array, that converts an open reference cursor to a delimited list of strings (created from an initial stand-alone procedure: [A Utility for Reading REF Cursors into a List of Delimited Strings](http://aprogrammerwrites.eu/?p=1777))
- Using this utility, very little code needs to be written once the test data has been set up: One call to return the reference cursor, and a second to return the actual values in a list, to be passed at the end in a single call to the library results checker

## ERD

<img src="/migrated_images/2016/10/unit-testing-three-erd_rc.png" alt="ERD" title="ERD" />

## Design Pattern Groups

The API testing framework is centred around the concept of input and output groups, representing the data sets that respectively form the inputs to, and outputs from, the program. The records in each group are printed by the framework with column headers for each scenario. These groups are identified by the developer, and in this case they are as noted below.

### Input Groups

- Employees Table
- Department Parameter

### Output Groups

- Select results
- Timing of average call

## Test Scenarios

The scenario descriptions start with a data set code, followed by a verbal description.

1. DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep (10) - pass dep 10
2. DS-2, testing same as 1 but with extra emp in another dep (20) - pass dep 10
3. DS-2, as second scenario, but - pass dep 20
4. DS-2, as second scenario, but - pass null dep
5. DS-3, Salaries total 1500 (< threshold of 1600, so return nothing) - pass dep 10

**2025 Note:** In October 2021 I published [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html), which proposes a new approach to scenario selection that tends to result in a larger number of scenarios, with clearer descriptions.

## Package Structure Diagram
<img src="/migrated_images/2016/10/Unit-Testing-Three-v1.2-CSD_RC.png" alt="Package Structure" title="Package Structure" />

## Call Structure Table - TT\_Emp\_WS.tt\_AIP\_Get\_Dept\_Emps
<img src="/migrated_images/2016/10/TRAPIT-CST-4.png" alt="Call Structure" title="Call Structure" />

## TT\_Emp\_WS.tt\_AIP\_Get\_Dept\_Emps - Emp REF Cursor Test Procedure

### Declare section

<div class="scrollbox">
<pre>
PROCEDURE tt_AIP_Get_Dept_Emps IS

  c_proc_name           CONSTANT VARCHAR2(61) := 'TT_Emp_WS.tt_AIP_Get_Dept_Emps';
  c_dep_id_1            CONSTANT PLS_INTEGER := 10;
  c_dep_id_2            CONSTANT PLS_INTEGER := 20;
  c_dep_nm_1            CONSTANT VARCHAR2(100) := 'Administration';
  c_dep_nm_2            CONSTANT VARCHAR2(100) := 'Marketing';
  c_job_bad             CONSTANT VARCHAR2(100) := 'AD_ASST';
  c_job_good            CONSTANT VARCHAR2(100) := 'IT_PROG';
  c_base_sal            CONSTANT PLS_INTEGER := 1000;
  c_out_group_lis       CONSTANT L1_chr_arr := L1_chr_arr ('Select results');

  c_ln_pre              CONSTANT VARCHAR2(10) := DML_API_TT_HR.c_ln_pre;

  c_dep_lis             CONSTANT L1_chr_arr := L1_chr_arr (c_dep_id_1, c_dep_id_1, c_dep_id_2, NULL, c_dep_id_1);

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
                                       Utils.List_Delim (c_ln_pre || '2',   c_dep_nm_1, c_ln_pre || '1', '2000',  '1.33', '.67')
                                               ),
                                               L1_chr_arr (
                                       Utils.List_Delim (c_ln_pre || '5',   c_dep_nm_2, c_ln_pre || '1', '5000',  '1',    '1.67')
                                               ),
                                               Utils_TT.c_empty_list,
                                               Utils_TT.c_empty_list
                        );

  c_scenario_ds_lis     CONSTANT L1_num_arr := L1_num_arr (1, 2, 2, 2, 3);
  c_scenario_lis        CONSTANT L1_chr_arr := L1_chr_arr (
                               'DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep (10) - pass dep 10',
                               'DS-2, testing same as 1 but with extra emp in another dep (20) - pass dep 10',
                               'DS-2, as second scenario, but - pass dep 20',
                               'DS-2, as second scenario, but - pass null dep',
                               'DS-3, Salaries total 1500 (< threshold of 1600, so return nothing) - pass dep 10');

  c_inp_group_lis       CONSTANT L1_chr_arr := L1_chr_arr ('Employee', 'Department Parameter');
  c_inp_field_2lis      CONSTANT L2_chr_arr := L2_chr_arr (
                                                        L1_chr_arr (
                                                                '*Employee Id',
                                                                'Last Name',
                                                                'Email',
                                                                'Hire Date',
                                                                'Job',
                                                                '*Salary',
                                                                '*Manager Id',
                                                                '*Department Id',
                                                                'Updated'),
                                                        L1_chr_arr (
                                                                '*Department Id')
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
  l_emp_csr                      SYS_REFCURSOR;
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
    l_inp_3lis (i)(2) := L1_chr_arr (c_dep_lis(i));
  END LOOP;
END Setup_Array;
/***************************************************************************************************

Setup_DB: Create test records for a given scenario for testing AIP_Get_Dept_Emps

***************************************************************************************************/
PROCEDURE Setup_DB (p_call_ind           PLS_INTEGER,   -- index of input dataset
                    x_inp_lis        OUT L1_chr_arr) IS -- input list, employees
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
- Data set is printed out in tabular format. In the current version of the utility code, this is not strictly necessary, because all the input data is printed out before the outputs

### 'Pure' API wrapper procedure

```sql
  PROCEDURE Purely_Wrap_API (p_scenario_ds      PLS_INTEGER,   -- index of input dataset
                             p_dep_id           PLS_INTEGER,   -- input department id
                             x_inp_lis      OUT L1_chr_arr,    -- generic inputs list (for scenario)
                             x_act_lis      OUT L1_chr_arr) IS -- generic actual values list (for scenario)
  BEGIN

    Setup_DB (p_scenario_ds, x_inp_lis);
    Timer_Set.Increment_Time (l_timer_set, Utils_TT.c_setup_timer);

    Emp_WS.AIP_Get_Dept_Emps (p_dep_id  => p_dep_id,
                              x_emp_csr => l_emp_csr);
    x_act_lis := Utils_TT.List_or_Empty (Utils_TT.Cursor_to_Array (x_csr => l_emp_csr));
    Timer_Set.Increment_Time (l_timer_set, Utils_TT.c_call_timer);
    ROLLBACK;

  END Purely_Wrap_API;
```

### Notes on 'Pure' API wrapper procedure

- Setup\_DB is called to create the data set for the scenario
- Emp\_WS.AIP\_Get\_Dept\_Emps returns the ref cursor
- Utils\_TT.Cursor\_to\_Array gets the ref cursor result set into an array (which may be empty)

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

END tt_AIP_Get_Dept_Emps;
```

### Notes on main section

- It's quite short isn't it? 😊
- Setup is called to do array setup
- Main section loops over the scenarios calling Purely\_Wrap\_API
- Is\_Deeply is called to do all the assertions within nested loops, then print the results

**2025 Note:** In the latest version of the unit testing framework, [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester), the main section is in a library module.

## Test Output

<div class="scrollbox">
<pre>
TRAPIT TEST: TT_Emp_WS.tt_AIP_Get_Dept_Emps
===========================================

Employees created in setup: DS-1 - 4 emps, 1 dep (10), emp-3 has no dep, emp-4 has bad job
==========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1658             10              IT_PROG           1000
2         1659             10        1658  IT_PROG           2000
3         1660                       1658  IT_PROG           3000
4         1661             10        1658  AD_ASST           4000

Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1662             10              IT_PROG           1000
2         1663             10        1662  IT_PROG           2000
3         1664                       1662  IT_PROG           3000
4         1665             10        1662  AD_ASST           4000
5         1666             20        1662  IT_PROG           5000

Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1667             10              IT_PROG           1000
2         1668             10        1667  IT_PROG           2000
3         1669                       1667  IT_PROG           3000
4         1670             10        1667  AD_ASST           4000
5         1671             20        1667  IT_PROG           5000

Employees created in setup: DS-2 - As dataset 1 but with extra emp-5, in second dep (20)
========================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1672             10              IT_PROG           1000
2         1673             10        1672  IT_PROG           2000
3         1674                       1672  IT_PROG           3000
4         1675             10        1672  AD_ASST           4000
5         1676             20        1672  IT_PROG           5000

Employees created in setup: DS-3 - As dataset 2 but with salaries * 0.1, total below reporting threshold of 1600
================================================================================================================

#  Employee id  Department id     Manager  Job id          Salary
- ----------- ------------- ---------- ---------- ----------
1         1677             10              IT_PROG            100
2         1678             10        1677  IT_PROG            200
3         1679                       1677  IT_PROG            300
4         1680             10        1677  AD_ASST            400
5         1681             20        1677  IT_PROG            500

SCENARIO 1: DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep (10) - pass dep 10 {
==================================================================================================================

    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date    Job      Salary  Manager Id  Department Id  Updated
            ----------- --------- ----- ----------- ------- ------ ---------- ------------- -----------
                   1658  LN_1       EM_1   01-OCT-2016  IT_PROG    1000                         10  01-OCT-2016
                   1659  LN_2       EM_2   01-OCT-2016  IT_PROG    2000        1658             10  01-OCT-2016
                   1660  LN_3       EM_3   01-OCT-2016  IT_PROG    3000        1658                 01-OCT-2016
                   1661  LN_4       EM_4   01-OCT-2016  AD_ASST    4000        1658             10  01-OCT-2016
        }
        =
        GROUP Department Parameter {
        ============================
            Department Id
            -------------
                       10
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

SCENARIO 2: DS-2, testing same as 1 but with extra emp in another dep (20) - pass dep 10 {
==========================================================================================

    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date    Job      Salary  Manager Id  Department Id  Updated
            ----------- --------- ----- ----------- ------- ------ ---------- ------------- -----------
                   1662  LN_1       EM_1   01-OCT-2016  IT_PROG    1000                         10  01-OCT-2016
                   1663  LN_2       EM_2   01-OCT-2016  IT_PROG    2000        1662             10  01-OCT-2016
                   1664  LN_3       EM_3   01-OCT-2016  IT_PROG    3000        1662                 01-OCT-2016
                   1665  LN_4       EM_4   01-OCT-2016  AD_ASST    4000        1662             10  01-OCT-2016
                   1666  LN_5       EM_5   01-OCT-2016  IT_PROG    5000        1662             20  01-OCT-2016
        }
        =
        GROUP Department Parameter {
        ============================
            Department Id
            -------------
                       10
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

SCENARIO 3: DS-2, as second scenario, but - pass dep 20 {
=========================================================

    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date    Job      Salary  Manager Id  Department Id  Updated
            ----------- --------- ----- ----------- ------- ------ ---------- ------------- -----------
                   1667  LN_1       EM_1   01-OCT-2016  IT_PROG    1000                         10  01-OCT-2016
                   1668  LN_2       EM_2   01-OCT-2016  IT_PROG    2000        1667             10  01-OCT-2016
                   1669  LN_3       EM_3   01-OCT-2016  IT_PROG    3000        1667                 01-OCT-2016
                   1670  LN_4       EM_4   01-OCT-2016  AD_ASST    4000        1667             10  01-OCT-2016
                   1671  LN_5       EM_5   01-OCT-2016  IT_PROG    5000        1667             20  01-OCT-2016
        }
        =
        GROUP Department Parameter {
        ============================
            Department Id
            -------------
                       20
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 1, Expected = 1 {
        ================================================
            F?  Name  Department  Manager  Salary  Salary Ratio (dep)  Salary Ratio (overall)
            -- ---- ---------- ------- ------ ------------------ ----------------------
                LN_5  Marketing   LN_1       5000                   1                    1.67
        } 0 failed, of 1: SUCCESS
        =========================
} 0 failed, of 1: SUCCESS
=========================

SCENARIO 4: DS-2, as second scenario, but - pass null dep {
===========================================================

    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date    Job      Salary  Manager Id  Department Id  Updated
            ----------- --------- ----- ----------- ------- ------ ---------- ------------- -----------
                   1672  LN_1       EM_1   01-OCT-2016  IT_PROG    1000                         10  01-OCT-2016
                   1673  LN_2       EM_2   01-OCT-2016  IT_PROG    2000        1672             10  01-OCT-2016
                   1674  LN_3       EM_3   01-OCT-2016  IT_PROG    3000        1672                 01-OCT-2016
                   1675  LN_4       EM_4   01-OCT-2016  AD_ASST    4000        1672             10  01-OCT-2016
                   1676  LN_5       EM_5   01-OCT-2016  IT_PROG    5000        1672             20  01-OCT-2016
        }
        =
        GROUP Department Parameter {
        ============================
            Department Id
            -------------
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 0, Expected = 0: SUCCESS
        =======================================================
} 0 failed, of 1: SUCCESS
=========================

SCENARIO 5: DS-3, Salaries total 1500 (< threshold of 1600, so return nothing) - pass dep 10 {
==============================================================================================

    INPUTS
    ======
        GROUP Employee {
        ================
            Employee Id  Last Name  Email  Hire Date    Job      Salary  Manager Id  Department Id  Updated
            ----------- --------- ----- ----------- ------- ------ ---------- ------------- -----------
                   1677  LN_1       EM_1   01-OCT-2016  IT_PROG     100                         10  01-OCT-2016
                   1678  LN_2       EM_2   01-OCT-2016  IT_PROG     200        1677             10  01-OCT-2016
                   1679  LN_3       EM_3   01-OCT-2016  IT_PROG     300        1677                 01-OCT-2016
                   1680  LN_4       EM_4   01-OCT-2016  AD_ASST     400        1677             10  01-OCT-2016
                   1681  LN_5       EM_5   01-OCT-2016  IT_PROG     500        1677             20  01-OCT-2016
        }
        =
        GROUP Department Parameter {
        ============================
            Department Id
            -------------
                       10
        }
        =
    OUTPUTS
    =======
        GROUP Select results: Actual = 0, Expected = 0: SUCCESS
        =======================================================
} 0 failed, of 1: SUCCESS
=========================

TIMING: Actual = 7, Expected <= 1: FAILURE
==========================================

SUMMARY for TT_Emp_WS.tt_AIP_Get_Dept_Emps
==========================================

Scenario                                                                                              # Failed  # Tests  Status
---------------------------------------------------------------------------------------------------- -------- ------- -------
DS-1, testing inner, outer joins, analytic over dep, and global ratios with 1 dep (10) - pass dep 10         0        2  SUCCESS
DS-2, testing same as 1 but with extra emp in another dep (20) - pass dep 10                                 0        2  SUCCESS
DS-2, as second scenario, but - pass dep 20                                                                  0        1  SUCCESS
DS-2, as second scenario, but - pass null dep                                                                0        1  SUCCESS
DS-3, Salaries total 1500 (< threshold of 1600, so return nothing) - pass dep 10                             0        1  SUCCESS
Timing                                                                                                       1        1  FAILURE
---------------------------------------------------------------------------------------------------- -------- ------- -------
Total                                                                                                        1        8  FAILURE
---------------------------------------------------------------------------------------------------- -------- ------- -------

Timer Set: TT_Emp_WS.tt_AIP_Get_Dept_Emps, Constructed at 01 Oct 2016 09:14:12, written at 09:14:13
===================================================================================================
[Timer timed: Elapsed (per call): 0.03 (0.000034), CPU (per call): 0.03 (0.000030), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Setup          0.08        0.05             5        0.01660        0.01000
Caller         0.03        0.01             5        0.00680        0.00200
(Other)        0.10        0.10             1        0.09500        0.10000
------- ---------- ---------- ------------ ------------- -------------
Total          0.21        0.16            11        0.01927        0.01455
------- ---------- ---------- ------------ ------------- -------------
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
