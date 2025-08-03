---
title: "Design Patterns for Database API Testing 2: Views 2 - Code"
date: 2016-05-21
categories: 
  - "design"
  - "oracle"
  - "plsql"
  - "sql"
  - "testing"
  - "utplsql"
  - "web-services"
tags: 
  - "design-pattern"
  - "oracle"
  - "plsql"
  - "sql"
  - "unit-test"
  - "utplsql"
  - "web-services-2"
---

Last October I gave a presentation on database unit testing with utPLSQL, [Oracle Unit Testing with utPLSQL](http://aprogrammerwrites.eu/?p=1545). I mentioned design patterns as a way of reducing the effort of building unit tests and outlined some strategies for coding them effectively.

In the current set of articles, I develop the ideas further, starting from the idea that all database APIs can be considered in terms of the axes:

- direction (i.e. getter or setter, noting that setters can also 'get')
- mode (i.e. real time or batch)

For each cell in the implied matrix, I construct an example API (or view) with specified requirements against Oracle's HR demo schema, and use this example to construct a testing program with appropriate scenarios as a design pattern. Concepts and common patterns and anti-patterns in automated API testing are discussed throughout, and these are largely independent of testing framework used. However, the examples use my own lightweight independent framework that is designed to help avoid many API testing anti-patterns. The code is available on GitHub here, [BrenPatF/trapit\_oracle\_tester](https://github.com/BrenPatF/trapit_oracle_tester), and includes both framework and design pattern examples.

Behind the four examples, there is an underlying design pattern that involves wrapping the API call in a 'pure' procedure, called once per scenario, with the output 'actuals' array including everything affected by the API, whether as output parameters, or on database tables, etc. The inputs are also extended from the API parameters to include any other effective inputs. Assertion takes place after all scenarios and is against the extended outputs, with extended inputs also listed. This concept of the 'pure' function, central to [Functional Programming](https://en.wikipedia.org/wiki/Functional_programming), has important advantages in automated testing. I explained the concepts involved in a presentation at the Oracle User Group Ireland Conference in March 2018:

[The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing)

[![](images/Mode-B_G.png)](http://aprogrammerwrites.eu/?attachment_id=2357) In the first part of this two part article,[Design Patterns for Database API Testing 2: Views 1 - Design](http://aprogrammerwrites.eu/?p=1689) I presented a design pattern for unit testing views, using an example based on Oracle's HR demo schema, and here I list the code for the main test procedure and a couple of the utility procedures, with notes.

A structure diagram shows how the PL/SQL packages relate to each other, and sections of the code are listed with notes.

**Package Structure Diagram** [![](images/TRAPIT-Testing-csd-1-2.png)](http://aprogrammerwrites.eu/?attachment_id=2392)

**Call Structure Table** [![](images/TRAPIT-CST-2.png)](http://aprogrammerwrites.eu/?attachment_id=2417) **TT\_View\_Drivers.tt\_HR\_Test\_View\_V - View Test Procedure**

_Declare section_

```
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

```

_Notes on declare section_

- Data sets, scenarios, expected values etc. are stored in generic arrays, where:
    - L1\_chr\_arr is type of array of VARCHAR2(4000), same as standard type SYS.ODCIVarchar2List
    - L2\_chr\_arr is a type of array of L1\_chr\_arr
    - L3\_chr\_arr is a type of array of L2\_chr\_arr

_Setup section_

```
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

_Notes on setup section_

- c\_dataset\_3lis contains the data for all data sets indexed by (data set, record, field)
- Setup\_Array is called once to do some setup on the arrays
- Setup\_DB is called for a single data set at a time in each scenario
- Description of the data set is contained in the array and printed out
- Data set is printed out in tabular format. In the most recent version of the utility code, this is not strictly necessary, because all the input data is printed out before the outputs

_'Pure' API wrapper procedure_

```
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

_Notes on 'Pure' API wrapper procedure_

- Setup\_DB is called to create the data set for the scenario
- Get\_View returns the results of the query on the view as 2-level array
- Get\_View rolls back after getting the results, so the inserted test records are removed from the database

_Main section_

```
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

_Notes on main section_

- It's quite short isn't it :)
- Setup is called to do array setup
- Main section loops over the scenarios calling Purely\_Wrap\_API
- Is\_Deeply is called to do all the assertions within nested loops, then print the results

**Utils\_TT - Test Utility Procedures** We will include only one procedure from this package in the body of the article. See gitHub link for the full code. _Is\_Deeply - to check results from testing_

```
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

_Notes on Is\_Deeply_

- This is the base version of Is\_Deeply with 3-level arrays of expected and actuals
- An inner loop asserts actual values (which are records) against expected
- The final assertion is against average call time
- It is expected that all assertion within a test procedure will be via a single call to one of the versions of this procedure, making a big reduction in code compared with traditional unit testing approaches
- After final assertion a call is made to write out all the results, by scenario, with all inputs printed first, followed by actuals (and expected, where they differ); this means that the test outputs now become precise and accurate documents of what the program does

_Get\_View - run a query dynamically on a view and return result set as array of strings_

```
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

_Notes on Get\_View_

- A query string is constructed from the input list of fields and optional where clause
- The fields are concatenated with delimiters and returned into an array of strings
- A rollback occurs to remove any test data created, so as not to interfere with any subsequent call
- If no data was returned from the query, we return a default 1-record listing containing the string 'EMPTY'

_Package block section_

```
BEGIN

  DBMS_Application_Info.Set_Client_Info (client_info => 'TT');
  Utils.c_session_id_if_TT := SYS_Context ('userenv', 'sessionid');

END Utils_TT;
```

_Notes on package block section_

- client\_info is set to 'TT', meaning the session operates in test mode
- The session id is stored in a package variable
- This id is referenced in the testing views and in insertion of test records with utid column

**Conclusions**

- A design pattern has been presented for testing views, with scenarios and output results in part 1
- The implementation, presented here, was against an Oracle database publicly available demonstration schema, and used the [TRAPIT - TRansactional API Testing in Oracle](http://aprogrammerwrites.eu/?p=1723) framework
- See also [A Template Script for JDBC Integration Testing of Oracle Procedures](http://aprogrammerwrites.eu/?p=1676)
- See also [Design Patterns for Database API Testing 3: Batch Loading of Flat Files](http://aprogrammerwrites.eu/?p=1791)

<script type="text/javascript">var addthis_config = {"data_track_addressbar":true};</script>

<script type="text/javascript" src="//s7.addthis.com/js/300/addthis_widget.js#pubid=ra-50fd09b73778290b"></script>
