---
title: "Design Patterns for Database API Testing 1: Web Service Saving 2 - Code"
date: 2016-05-08
categories: 
  - "oracle"
  - "plsql"
  - "sql"
  - "testing"
  - "utplsql"
  - "web-services"
tags: 
  - "oracle"
  - "plsql"
  - "sql"
  - "utplsql"
  - "web-service"
---

Last October I gave a presentation on database unit testing with utPLSQL, [Oracle Unit Testing with utPLSQL](http://aprogrammerwrites.eu/?p=1545). I mentioned design patterns as a way of reducing the effort of building unit tests and outlined some strategies for coding them effectively.

In the current set of articles, I develop the ideas further, starting from the idea that all database APIs can be considered in terms of the axes:

- direction (i.e. getter or setter, noting that setters can also 'get')
- mode (i.e. real time or batch)

For each cell in the implied matrix, I construct an example API (or view) with specified requirements against Oracle's HR demo schema, and use this example to construct a testing program with appropriate scenarios as a design pattern. Concepts and common patterns and anti-patterns in automated API testing are discussed throughout, and these are largely independent of testing framework used. However, the examples use my own lightweight independent framework that is designed to help avoid many API testing anti-patterns. The code is available on GitHub here, [BrenPatF/trapit\_oracle\_tester](https://github.com/BrenPatF/trapit_oracle_tester), and includes both framework and design pattern examples.

Behind the four examples, there is an underlying design pattern that involves wrapping the API call in a 'pure' procedure, called once per scenario, with the output 'actuals' array including everything affected by the API, whether as output parameters, or on database tables, etc. The inputs are also extended from the API parameters to include any other effective inputs. Assertion takes place after all scenarios and is against the extended outputs, with extended inputs also listed. This concept of the 'pure' function, central to [Functional Programming](https://en.wikipedia.org/wiki/Functional_programming), has important advantages in automated testing. I explained the concepts involved in a presentation at the Oracle User Group Ireland Conference in March 2018:

[The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing)

[![](images/Mode-R_S2.png)](http://aprogrammerwrites.eu/?attachment_id=2363) In this first example, I present a design pattern for web service 'save' procedures by means of a conceptual discussion, together with a working example of base code and test code for a procedure to save new employees in Oracle's well-known HR demonstration schema.

In part 1, [Design Patterns for Database API Testing 1: Web Service Saving 1 - Design](http://aprogrammerwrites.eu/?p=1614) I began with an abstract discussion, and progressed to an example based on Oracle's HR demo schema. That article presented the test scenarios, logical data structures, and the example testing output. Here, I start by listing a number of extremely prevalent antipattern approaches to database API testing, and how to avoid them.

Following this, I list the code for the base procedure and the test package with notes against each section. There are three utility packages used, and I list the code for one procedure from those, that does all of the assertion and reporting from a single call. After the code listings, I provide some design material on the code packages used.

[Design Patterns for Database API Testing 2: Views 1 - Design](http://aprogrammerwrites.eu/?p=1689) presents another design pattern following the same ideas.

**API Testing Antipatterns**

Automated unit testing of database APIs is often considered difficult and time-consuming. Unfortunately I believe it is made much worse by widespread following of antipattern approaches, some of which appear on popular database testing-related sites as examples to follow.

Here are a few antipatterns that I have identified, and which my code attached below avoids.

_Test this then test this then..._

Antipattern description

This occurs when the unit test code is written as a long sequence of method calls, each followed by assertions, and each usually having hard-coded data values in both method calls and assertions. It is a classic antipattern because it is widespread and developers often follow it thinking they are doing the right thing.

Antipattern consequence

The testing code becomes very long-winded and hard to follow, and tends to result in less rigorous testing.

Pattern alternative

Store all input data in arrays at the start, loop over the input scenarios array, accumulating the outputs in arrays, and loop over the output arrays for the assertions.

_Method-based testing_

Antipattern description

This occurs when the test suite is based on testing all the methods in a package, rather than units of behaviour, such as a procedure serving a web service.

Antipattern consequence

Ian Cooper explains very well in the video link below the adverse consequences of this antipattern from a Java perspective, but it applies equally to database testing.

- It results in a great deal more testing code, with a lot of redundancy, which deters people from the whole concept of test-driven development
- Re-factoring is more difficult because the unit test code tests the implementation
- Shifting the focus of testing from the behavioural side is also unlikely to improve testing quality

[TDD: Where Did It All Go Wrong?](http://www.infoq.com/presentations/tdd-original)

Pattern alternative

Include in your test suite only tests of well-defined units of behaviour such as a web service entry point procedure, ignoring helper methods.

_Field-level assertion_

Antipattern description

This occurs when the individual fields written to the database or in output arrays have their own assertions.

Antipattern consequence

Real database applications often have tables with large numbers of fields, and the numbers of assertions can consequently become very large.

Pattern alternative

Assert at the record level by concatenating the fields in a record into one string.

_Coupled tests_

Antipattern description

This occurs when testing one scenario affects another; for example, when all the test data are created at once and not rolled back after each scenario and re-created as needed for the next.

This antipattern is strongly promoted by some popular testing frameworks, where package level setup and teardown are mandatory, at least by default. My own framework deliberately does not support these, preferring the test program to call its own private procedures as necessary at the appropriate level.

Antipattern consequence

The coding becomes more complex as it is necessary to distentangle what results from earlier test scenarios from that of the current scenario.

Pattern alternative

Set up test data at the scenario level, not at the procedure level, and roll it back at the end of the scenario testing.

_Opaque output_

Antipattern description

This occurs when the output does not show what was tested.

Antipattern consequence

It is harder to review the testing, especially when combined, as it usually is, with the _Test this then test this then..._ antipattern. This results in lower quality.

Pattern alternative

Make the output self-documenting with clear scenario descriptions and listings of entities and fields that are being tested.

**Emp\_WS.AIP\_Save\_Emps - Base Procedure**

```
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

_Notes on base procedure_

- There is a FORALL statement to insert a record into employees for each record in the input list in one batch
- The RETURNING clause uses a BULK COLLECT to return the new id and its value in words to the output list
- SAVE EXCEPTIONS means valid records will be inserted, while invalid ones will fall to the EXCEPTION block
- Invalid records cause a zero and the error message to be interpolated into the output array
- I tried to follow a similar principle here to that in Connor McDonald's article, [Assume the Best; Plan for the Worst - Oracle Magazine Article](http://www.oracle.com/technetwork/issue-archive/2016/16-jan/o16performance-2882216.html)

**TT\_Emp\_WS.tt\_AIP\_Save\_Emps - Testing Package Procedure**

_Declaration section_

```
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
  g_exp_3lis                   L3_chr_arr;

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

  l_act_3lis         L3_chr_arr := L3_chr_arr();

```

_Notes on declaration section_

- Observe that the declarations of l\_inp\_3lis and g\_exp\_3lis as 3-level generic lists map directly to Scenario Inputs and Scenario Outputs in the array structure diagram in [Design Patterns for Database API Testing 1: Web Service Saving 1 - Design](http://aprogrammerwrites.eu/?p=1614)
- c\_scenario\_lis, as a generic list maps directly to Scenario Descriptions
- Adding or subtracting fields in the input object means simply adding or subtracting fields in the inner level of the list
- Adding test scenarios (i.e. calls) amounts to adding outer level records in each list
- Using the program as a template for other unit test programs is therefore very easy
- All inputs and outputs are printed in the output, making the testing self-documenting

_Setup_

```
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

```

_Notes on Setup_

- Setup gets a value from the sequence in order to determine what values are expected in the outputs
- Any concurrent calls could invalidate this expectation, but scheduling the test runs may avoid this; if not, one could change the expectation to a category, of say positive integer in a certain range
- The example has two output groups, corresponding to a table and an output array
- Observe how easy it would be to extend to extra tables or arrays, just by adding records in the middle level of the list
- c\_empty\_list is a 1-record list containing a string 'EMPTY', which is used to simplify assertion of empty lists
- Setup in this case has no need to create test data, as we have chosen to reference pre-existing departments
- There is no teardown as the inserts from the procedure call are rolled back after each call
- The call to Write\_Times writes out the report from the package level timer set, including setup timings (if any)

_'Pure' API wrapper procedure_

```
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

```

_Notes on main testing procedure_

- The call converts the generic input data into the objects required by the base procedure, then calls the procedure and sets the resulting records for each output group
- Purely\_Wrap\_API has a very simple main section that calls local procedures to split the logic into three sections
- Its EXCEPTION clause captures the scenario where an invalid type conversion is attempted
- Its output is a generic 2-level list that is mapped to a record in the outer-level output list
- Empty lists are converted to c\_empty\_list, the 1-record list mentioned earlier
- The procedure ends with a rollback

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

_Notes on main section_

- The main section is very simple: a local procedure Purely\_Wrap\_API is called for each input scenario
- The call converts the generic input data into the objects required by the base procedure, then calls the procedure and sets the resulting records for each output group
- Is\_Deeply is a generic utility procedure that does all the assertions and calls the results reporting utility
- This can be used without change for any number of output groups with any numbers of fields, of any types

**Utils\_TT - API Test Utilities Package** We will include only one procedure from this package in the body of the article. See gitHub link for the full code. _Is\_Deeply - to check results from testing_

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

**Package Usages Diagram** [![](images/TRAPIT-Testing-csd-1-2.png)](http://aprogrammerwrites.eu/?attachment_id=2392)

- The code timing package was originally written in 2010, and is described here: [Code Timing and Object Orientation and Zombies](http://aprogrammerwrites.eu/?p=1632), although in an older version without the dependence on the newer Utils package

**Call Structure Table - TT\_Emp\_WS.tt\_AIP\_Save\_Emps** [![](images/TRAPIT-CST-1.png)](http://aprogrammerwrites.eu/?attachment_id=2412)**Installation Instructions**

See [TRAPIT - TRansactional API Testing in Oracle](http://aprogrammerwrites.eu/?p=1723) for a link to the installation code for both the framework and the demo code on gitHub, and links to related articles.

_Exercise_ If you are interested in following a similar approach to testing, you might find this exercise an easy way to get going: Revise the base procedure to return error records as a second output array, with the first output returning only valid records. You do not need to define any new types for this. Update the  unit test code for the new signature.

**Conclusions**

- A design pattern has been presented for database web service save procedures, with scenarios and output results in part 1
- The implementation, presented here, was against an Oracle database publicly available demonstration schema, and used the [TRAPIT - TRansactional API Testing in Oracle](http://aprogrammerwrites.eu/?p=1723) framework
- See also [A Template Script for JDBC Integration Testing of Oracle Procedures](http://aprogrammerwrites.eu/?p=1676)
- See also [Design Patterns for Database Unit Testing 2: Views 1 - Design](http://aprogrammerwrites.eu/?p=1689)

<script type="text/javascript">var addthis_config = {"data_track_addressbar":true};</script>

<script type="text/javascript" src="//s7.addthis.com/js/300/addthis_widget.js#pubid=ra-50fd09b73778290b"></script>
