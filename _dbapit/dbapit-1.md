---
layout: post
title: "DBAPIT 1: Web Service Saving"
date: 2016-05-08
migrated: true
dbapit_next: /dbapit/dbapit-2/
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
#### Part 1 in a series on: Design Patterns for Database API Testing

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

<img src="/migrated_images/2016/05/Mode-R_S.png" alt="Mode: Batch Save" title="Mode: Batch Save" />

In the first article in the series a design pattern is presented for testing data saving procedures, such as might be used by a web service.

There is a working example of base code and test code for a procedure to save new employees in Oracle's well-known HR demonstration schema.

## Use Case for Web Service Save Procedure

- Purpose of procedure is to save a set of new records to a database table
- Surrogate primary key is generated
- Input is an array of objects with the records to be saved
- Output is an array of objects containing the new primary key plus a description
- For records failing validation, zero is returned, plus the error message, and the valid records will still be saved


## Scenarios

<img src="/migrated_images/2016/05/Unit-Testing-HR.png" alt="Unit Testing - HR" title="Unit Testing - HR" />

The procedure inserts records in Oracle's HR employees table, and we identify four test scenarios:

1. Passing a single valid record
2. Passing a single invalid record
3. Trying to pass a record with an invalid type
4. Passing multiple valid records, and one invalid record

**2025 Note:** In October 2021 I published [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html), which proposes a new approach to scenario selection that tends to result in a larger number of scenarios, with clearer descriptions.

## Output Groups

- Records inserted into table employees
- Records returned in output parameter array
- Timing of average call

## Test Results Output

The output below is for a failing run, where the time limit is breached, and I also have deliberately entered incorrect expected values for two records, to show the difference in formatting between success and failure output group records. I like to include the output tables on completion of development in my technical design document.

<div class="scrollbox">
<pre>
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
</pre>
</div>

### Notes on output

- In the event of the suite failing, as here, the utility code ensures that an Oracle error is generated. This can then be trapped by a calling Unix script from a scheduled Jenkins job to send out emails

## Emp\_WS.AIP\_Save\_Emps - Base Procedure

```sql
PROCEDURE AIP_Save_Emps (p_emp_in_lis           ty_emp_in_arr,     -- list of employees to insert
                         x_emp_out_lis      OUT ty_emp_out_arr) IS -- list of employee results
   l_emp_out_lis        ty_emp_out_arr;
  bulk_errors          EXCEPTION;
  PRAGMA               EXCEPTION_INIT (bulk_errors, -24381);
  n_err PLS_INTEGER := 0;
BEGIN
  FORALL i IN 1..p_emp_in_lis.COUNT
    SAVE EXCEPTIONS
    INSERT INTO employees (
        employee_id,
        last_name,
        email,
        hire_date,
        job_id,
        salary
    ) VALUES (
        employees_seq.NEXTVAL,
        p_emp_in_lis(i).last_name,
        p_emp_in_lis(i).email,
        SYSDATE,
        p_emp_in_lis(i).job_id,
        p_emp_in_lis(i).salary
    )
    RETURNING ty_emp_out_obj (employee_id, To_Char(To_Date(employee_id,'J'),'JSP')) BULK COLLECT INTO x_emp_out_lis;
EXCEPTION
  WHEN bulk_errors THEN
    l_emp_out_lis := x_emp_out_lis;
    FOR i IN 1 .. sql%BULK_EXCEPTIONS.COUNT LOOP
      IF i > x_emp_out_lis.COUNT THEN
        x_emp_out_lis.Extend;
      END IF;
      x_emp_out_lis (SQL%Bulk_Exceptions (i).Error_Index) := ty_emp_out_obj (0, SQLERRM (- (SQL%Bulk_Exceptions (i).Error_Code)));
    END LOOP;
    FOR i IN 1..p_emp_in_lis.COUNT LOOP
      IF i > x_emp_out_lis.COUNT THEN
        x_emp_out_lis.Extend;
      END IF;
      IF x_emp_out_lis(i).employee_id = 0 THEN
        n_err := n_err + 1;
      ELSE
        x_emp_out_lis(i) := l_emp_out_lis(i - n_err);
      END IF;
    END LOOP;
END AIP_Save_Emps;
```

### Notes on base procedure

- There is a FORALL statement to insert a record into employees for each record in the input list in one batch
- The RETURNING clause uses a BULK COLLECT to return the new id and its value in words to the output list
- SAVE EXCEPTIONS means valid records will be inserted, while invalid ones will fall to the EXCEPTION block
- Invalid records cause a zero and the error message to be interpolated into the output array
- I tried to follow a similar principle here to that in Connor McDonald's article, [Assume the Best; Plan for the Worst - Oracle Magazine Article](http://www.oracle.com/technetwork/issue-archive/2016/16-jan/o16performance-2882216.html)

## TT\_Emp\_WS.tt\_AIP\_Save\_Emps - Testing Package Procedure

### Declaration section

<div class="scrollbox">
<pre>
PROCEDURE tt_AIP_Save_Emps IS

  c_proc_name CONSTANT  VARCHAR2(61) := 'TT_Emp_WS.tt_AIP_Save_Emps';
  c_ln_prefix             CONSTANT VARCHAR2(20) := 'LN ';
  c_em_prefix             CONSTANT VARCHAR2(20) := 'EM ';
  c_ln                    CONSTANT L1_chr_arr := L1_chr_arr (
        c_ln_prefix || '1', c_ln_prefix || '2', c_ln_prefix || '3', c_ln_prefix || '4', c_ln_prefix || '5', c_ln_prefix || '6');
  c_em                    CONSTANT L1_chr_arr := L1_chr_arr (
        c_em_prefix || '1', c_em_prefix || '2', c_em_prefix || '3', c_em_prefix || '4', c_em_prefix || '5', c_em_prefix || '6');
  c_job_id                CONSTANT VARCHAR2(20) := 'IT_PROG';
  c_job_id_invalid        CONSTANT VARCHAR2(20) := 'NON_JOB';

  c_salary                CONSTANT L1_chr_arr := L1_chr_arr ('1000', '1500', '2000x', '3000', '4000', '5000');

  c_params_3lis           CONSTANT L3_chr_arr := L3_chr_arr (
          L2_chr_arr (L1_chr_arr (c_ln (1),    c_em (1),        c_job_id,         c_salary (1))), -- valid
          L2_chr_arr (L1_chr_arr (c_ln (2),    c_em (2),        c_job_id_invalid, c_salary (2))), -- invalid
          L2_chr_arr (L1_chr_arr (c_ln (3),    c_em (3),        c_job_id,         c_salary (3))), -- invalid salary, nan
          L2_chr_arr (L1_chr_arr (c_ln (4),    c_em (4),        c_job_id,         c_salary (4)),  -- valid
                      L1_chr_arr (c_ln (5),    c_em (5),        c_job_id_invalid, c_salary (5)),  -- invalid job id
                      L1_chr_arr (c_ln (6),    c_em (6),        c_job_id,         c_salary (6)))  -- valid
  );
  g_exp_3lis             L3_chr_arr;
  c_ws_ms_limit          CONSTANT PLS_INTEGER := 2;
  c_scenario_lis         CONSTANT L1_chr_arr := L1_chr_arr (
                               '1 valid record',
                               '1 invalid job id',
                               '1 invalid number',
                               '2 valid records, 1 invalid job id (2 deliberate errors)'
  );
  c_inp_group_lis       CONSTANT L1_chr_arr := L1_chr_arr ('Employee');
  c_inp_field_2lis      CONSTANT L2_chr_arr := L2_chr_arr (
                                                        L1_chr_arr (
                                                                'Name',
                                                                'Email',
                                                                'Job',
                                                                '*Salary')
  );
  c_out_group_lis       CONSTANT L1_chr_arr := L1_chr_arr ('Employee', 'Output array', 'Exception');
  c_out_field_2lis      CONSTANT L2_chr_arr :=  L2_chr_arr (
                                    L1_chr_arr ('*Employee id', 'Name', 'Email', 'Job', '*Salary'),
                                    L1_chr_arr ('*Employee id', 'Description'),
                                    L1_chr_arr ('Error message')
  );
  l_timer_set           PLS_INTEGER;
  l_inp_3lis            L3_chr_arr := L3_chr_arr();
  l_act_3lis            L3_chr_arr := L3_chr_arr();
</pre>
</div>

### Notes on declaration section

- Observe that the declarations of l\_inp\_3lis and g\_exp\_3lis as 3-level generic lists map directly to Scenario Inputs and Scenario Outputs in the array structure diagram above
- c\_scenario\_lis, as a generic list maps directly to Scenario Descriptions
- Adding or subtracting fields in the input object means simply adding or subtracting fields in the inner level of the list
- Adding test scenarios (i.e. calls) amounts to adding outer level records in each list
- Using the program as a template for other unit test programs is therefore very easy
- All inputs and outputs are printed in the output, making the testing self-documenting

### Setup

<div class="scrollbox">
<pre>
  PROCEDURE Setup_Array IS
    l_last_seq_val         PLS_INTEGER;
  BEGIN

    SELECT employees_seq.NEXTVAL
      INTO l_last_seq_val
      FROM DUAL;
    g_exp_3lis := L3_chr_arr ( -- each call results in a list of 2 output lists: first is the table records; second is the out array
                        L2_chr_arr (L1_chr_arr (Utils.List_Delim (To_Char(l_last_seq_val+1), c_ln (1), c_em (1), c_job_id, c_salary (1))), -- valid char, num pair
                                    L1_chr_arr (Utils.List_Delim (To_Char(l_last_seq_val+1), To_Char(To_Date(l_last_seq_val+1,'J'),'JSP'))),
                                    Utils_TT.c_empty_list
                        ),
                        L2_chr_arr (Utils_TT.c_empty_list,
                                    L1_chr_arr (Utils.List_Delim (0, 'ORA-02291: integrity constraint (.) violated - parent key not found')),
                                    Utils_TT.c_empty_list
                        ),
                        L2_chr_arr (Utils_TT.c_empty_list,
                                    Utils_TT.c_empty_list,
                                    L1_chr_arr ('ORA-06502: PL/SQL: numeric or value error: character to number conversion error')
                        ),
                        L2_chr_arr (L1_chr_arr (Utils.List_Delim (To_Char(l_last_seq_val+3), c_ln (4), c_em (4), c_job_id, c_salary (1)), -- c_salary (1) should be c_salary (4)
                                                Utils.List_Delim (To_Char(l_last_seq_val+5), c_ln (6), c_em (6), c_job_id, c_salary (6)),
                                                Utils.List_Delim (To_Char(l_last_seq_val+5), c_ln (6), c_em (6), c_job_id, c_salary (6))), -- duplicate record to generate error
                                    L1_chr_arr (Utils.List_Delim (To_Char(l_last_seq_val+3), To_Char(To_Date(l_last_seq_val+3,'J'),'JSP')),
                                                Utils.List_Delim (0, 'ORA-02291: integrity constraint (.) violated - parent key not found'),
                                                Utils.List_Delim (To_Char(l_last_seq_val+5), To_Char(To_Date(l_last_seq_val+5,'J'),'JSP'))),
                                    Utils_TT.c_empty_list
                        )
                     );
    l_act_3lis.EXTEND (c_params_3lis.COUNT);
    l_inp_3lis.EXTEND (c_params_3lis.COUNT);

    FOR i IN 1..c_params_3lis.COUNT LOOP
      l_inp_3lis(i) := L2_chr_arr();
      l_inp_3lis(i).EXTEND(1);
      l_inp_3lis(i)(1) := L1_chr_arr();
      l_inp_3lis(i)(1).EXTEND(c_params_3lis(i).COUNT);
      FOR j IN 1..c_params_3lis(i).COUNT LOOP
        l_inp_3lis(i)(1)(j) := Utils.List_Delim (c_params_3lis(i)(j)(1), c_params_3lis(i)(j)(2), c_params_3lis(i)(j)(3), c_params_3lis(i)(j)(4));
      END LOOP;
    END LOOP;

  END Setup_Array;
</pre>
</div>

### Notes on Setup

- Setup gets a value from the sequence in order to determine what values are expected in the outputs
- Any concurrent calls could invalidate this expectation, but scheduling the test runs may avoid this; if not, one could change the expectation to a category, of say positive integer in a certain range
- The example has two output groups, corresponding to a table and an output array
- Observe how easy it would be to extend to extra tables or arrays, just by adding records in the middle level of the list
- c\_empty\_list is a 1-record list containing a string 'EMPTY', which is used to simplify assertion of empty lists
- Setup in this case has no need to create test data, as we have chosen to reference pre-existing departments
- There is no teardown as the inserts from the procedure call are rolled back after each call
- The call to Write\_Times writes out the report from the package level timer set, including setup timings (if any)

### 'Pure' API wrapper procedure

<div class="scrollbox">
<pre>
PROCEDURE Purely_Wrap_API (p_inp_2lis        L2_chr_arr,       -- input list of lists (record, field)
                           x_act_2lis    OUT L2_chr_arr) IS    -- output list of lists (group, record)

  l_emp_out_lis       emp_out_arr;
  l_tab_lis           L1_chr_arr;
  l_arr_lis           L1_chr_arr;
  l_err_lis           L1_chr_arr;

  -- Do_Save makes the ws call and returns o/p array
  PROCEDURE Do_Save (x_emp_out_lis OUT emp_out_arr) IS
    l_emp_in_lis        emp_in_arr := emp_in_arr();
  BEGIN

    FOR i IN 1..p_inp_2lis.COUNT LOOP
      l_emp_in_lis.EXTEND;
      l_emp_in_lis (l_emp_in_lis.COUNT) := emp_in_rec (p_inp_2lis(i)(1), p_inp_2lis(i)(2), p_inp_2lis(i)(3), p_inp_2lis(i)(4));
    END LOOP;

    Timer_Set.Init_Time (p_timer_set_ind => l_timer_set);
    Emp_WS.AIP_Save_Emps (
              p_emp_in_lis        => l_emp_in_lis,
              x_emp_out_lis       => x_emp_out_lis);
    Timer_Set.Increment_Time (p_timer_set_ind => l_timer_set, p_timer_name => Utils_TT.c_call_timer);

  END Do_Save;

  -- Get_Tab_Lis: gets the database records inserted into a generic list of strings
  PROCEDURE Get_Tab_Lis (x_tab_lis OUT L1_chr_arr) IS
  BEGIN

    SELECT Utils.List_Delim (employee_id, last_name, email, job_id, salary)
      BULK COLLECT INTO x_tab_lis
      FROM employees
     WHERE ttid = Utils.c_session_id_if_TT
     ORDER BY employee_id;
    Timer_Set.Increment_Time (p_timer_set_ind => l_timer_set, p_timer_name => 'SELECT');

  EXCEPTION
    WHEN NO_DATA_FOUND THEN NULL;
  END Get_Tab_Lis;

  -- Get_Arr_Lis converts the ws output array into a generic list of strings
  PROCEDURE Get_Arr_Lis (p_emp_out_lis emp_out_arr, x_arr_lis OUT L1_chr_arr) IS
  BEGIN

    IF p_emp_out_lis IS NOT NULL THEN
      x_arr_lis := L1_chr_arr();
      x_arr_lis.EXTEND (p_emp_out_lis.COUNT);
      FOR i IN 1..p_emp_out_lis.COUNT LOOP
        x_arr_lis (i) := Utils.List_Delim (p_emp_out_lis(i).employee_id, p_emp_out_lis(i).description);
      END LOOP;
    END IF;

  END Get_Arr_Lis;
BEGIN

  BEGIN
    Do_Save (x_emp_out_lis => l_emp_out_lis);
    Get_Tab_Lis (x_tab_lis => l_tab_lis);
    Get_Arr_Lis (p_emp_out_lis => l_emp_out_lis, x_arr_lis => l_arr_lis);
  EXCEPTION
    WHEN OTHERS THEN
      l_err_lis := L1_chr_arr (SQLERRM);
  END;

  x_act_2lis := L2_chr_arr (Utils_TT.List_or_Empty (l_tab_lis), Utils_TT.List_or_Empty (l_arr_lis), Utils_TT.List_or_Empty (l_err_lis));
  ROLLBACK;
END Purely_Wrap_API;
</pre>
</div>

### Notes on main testing procedure

- The call converts the generic input data into the objects required by the base procedure, then calls the procedure and sets the resulting records for each output group
- Purely\_Wrap\_API has a very simple main section that calls local procedures to split the logic into three sections
- Its EXCEPTION clause captures the scenario where an invalid type conversion is attempted
- Its output is a generic 2-level list that is mapped to a record in the outer-level output list
- Empty lists are converted to c\_empty\_list, the 1-record list mentioned earlier
- The procedure ends with a rollback

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
  Timer_Set.Increment_Time (l_timer_set, 'Setup_Array');

  FOR i IN 1..c_params_3lis.COUNT LOOP
    Purely_Wrap_API (c_params_3lis(i), l_act_3lis(i));
  END LOOP;
  Utils_TT.Is_Deeply (c_proc_name, c_scenario_lis, l_inp_3lis, l_act_3lis, g_exp_3lis, l_timer_set, c_ws_ms_limit,
                      c_inp_group_lis, c_inp_field_2lis, c_out_group_lis, c_out_field_2lis);
EXCEPTION
  WHEN OTHERS THEN
    Utils.Write_Other_Error;
    RAISE;
END tt_AIP_Save_Emps;
```

### Notes on main section

- The main section is very simple: a local procedure Purely\_Wrap\_API is called for each input scenario
- The call converts the generic input data into the objects required by the base procedure, then calls the procedure and sets the resulting records for each output group
- Is\_Deeply is a generic utility procedure that does all the assertions and calls the results reporting utility
- This can be used without change for any number of output groups with any numbers of fields, of any types

**2025 Note:** In the latest version of the unit testing framework, [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester), the main section is in a library module.

## Utils\_TT - API Test Utilities Package

We will include only one procedure from this package in the body of the article. See gitHub link for the full code.

**2025 Note:** In the latest version of the unit testing framework, [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester), the example code is not present, but may be found in the git history.

### Is\_Deeply - to check results from testing

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

## Package Usages Diagram
<img src="/migrated_images/2016/05/TRAPIT-Testing-csd-1-2.png" alt="Package Usages Diagram" title="Package Usages Diagram" />

- The code timing package was originally written in 2010, and is described here: [Code Timing and Object Orientation and Zombies](https://brenpatf.github.io/migrated/code-timing-and-object-orientation-and-zombies/), although in an older version without the dependence on the newer Utils package

**2025 Note:** The latest version of the code timing package is here, [Oracle PL/SQL Code Timing Module](https://github.com/BrenPatF/timer_set_oracle).

## Call Structure Table - TT\_Emp\_WS.tt\_AIP\_Save\_Emps

<img src="/migrated_images/2016/05/TRAPIT-CST-1.png" alt="Call Structure Table" title="Call Structure Table" />

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
