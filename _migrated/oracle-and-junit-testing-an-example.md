---
layout: post
title: "Oracle and JUnit Data Driven Testing: An Example"
date: 2016-11-13
migrated: true
group: func-testing
categories: 
  - "java"
  - "oracle"
  - "plsql"
  - "sql"
  - "testing"
  - "utplsql"
tags: 
  - "java"
  - "junit"
  - "oop"
  - "oracle"
  - "sql"
  - "unit-test"
  - "utplsql"
---
**Note, 2025:** See [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html), from June 2023, for the author's current views on unit testing.

In [Design Patterns for Database API Testing](https://brenpatf.github.io//migrated/dbapit-series-index/) I identified a number of _antipatterns_ commonly seen in database testing.

utPLSQL, originally developed by Steve Feuerstein around 15 years ago, may have been the first Oracle unit testing framework and seems to be the most widely used. Its new [utPLSQL GitHub Page](https://github.com/utPLSQL) describes itself thus:

> The official new home of utPLSQL, unit testing framework for PL/SQL, based on xUnit.

xUnit refers to the family of unit testing frameworks derived from Java's JUnit, and present in most object-oriented languages, such as Ruby, Python etc.

It has occurred to me that some of the problems with unit testing in the database world may arise from translating object-oriented ideas on testing too zealously into the database world, where they may not work so well. It could also impinge on the design of base code, in that in the object-oriented world any complexity in unit tests is usually seen as a 'code smell' indicating the base units are too complex; _testability_ is seen as a key objective in OO module design. To gain some insight into the differences between database and object-oriented testing, it seemed like a good idea to try to test the same functionality in both Java and Oracle. This article gives the results of this experiment.

## Testing Example

In Java unit testing one normally tries to test code in isolation, without database access or other dependencies, whereas in Oracle it is normally database operations that are being tested. As a compromise I chose to implement code that would read a single key column from a CSV file and store counts of the key values in memory, with methods to return these key-value pairs as lists either unordered, ordered by key, or ordered by value (then key).

The constructor method will take three parameters:

- File name
- Delimiter
- Key column position

We'll use the same file name, which in Java will include the full path, whereas Oracle will assume the path from an Oracle directory object. There will be two scenarios that will test different values for the other two parameters simultaneously (as what I call _sub-scenarios_ in [Design Patterns for Database API Testing](https://brenpatf.github.io//migrated/dbapit-series-index/)).

For the unordered method, we will validate only the scalar count of the records returned, while for the ordered methods we will validate the full ordered lists of tuples returned. For illustrative purposes one test method for each scenario will have a deliberate error in the expected values.

### Test Scenario 1 - Tie-break/single-delimiter/interior column

Here the test file has four columns, with the key column being the third, and the delimiter a single character ','. The file contains lines:

```
0,1,Cc,3
00,1,A,9
000,1,B,27
0000,1,A,81
```

Note that keys 'Cc' and 'B' both occur once, and when ordered by value (then key) 'B' should appear before 'Cc'.

### Test Scenario 2 - Two copies/double-delimiter/first column

Here the test file has three columns, with the key column being the first, and the delimiter two characters ';;'. The file contains two identical lines:

```
X;;1;;A
X;;1;;A
```

## Java - JUnit

### JUnit Code

<div class="scrollbox">
<pre>
package colgroup;
/***************************************************************************************************
Name:        TestColGroup.java

Description: Junit testing class for Col_Group class. Uses Parameterized.class to data-drive
                                                                               
Modification History
Who                  When        Which What
-------------------- ----------- ----- -------------------------------------------------------------
B. Furey             22-Oct-2016 1.0   Created                       

***************************************************************************************************/
import static org.junit.Assert.\*;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.AbstractMap;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.List;
import java.util.Map;

import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.Parameterized.Parameters;
import org.junit.runners.Parameterized;

@RunWith(Parameterized.class)
public class TestColGroup {
/***************************************************************************************************

Private instance variables: 2 scenarios, input, and expected records declared here, initially in 
2-level generic arrays, but expected records transferred to List for assertion

***************************************************************************************************/
  private ColGroup colGroup = null;
  private String testFile = "H:/Script/Input/ut_group.csv";
  private String[][] testLines = new String[][] { 
      {"0,1,Cc,3", "00,1,A,9", "000,1,B,27", "0000,1,A,81"}, 
      {"X;;1;;A", "X;;1;;A"}
  };
  private String[] testDelim = new String[] {",", ";;"};
  private int[] testColnum = new int[] {2, 0};
  private List&lt;String&gt; lines;
  private String delim;
  private int colnum;

  private String[][] keysK = new String[][] { 
      {"A", "Bx", "Cc"}, 
      {"X"}
  };
  private int[][] valuesK = new int[][] { 
      {2, 1, 1}, 
      {2}
  };
  private String[][] keysV = new String[][] { 
      {"B", "Cc", "A"},
      {"X"}
  };
  private int[][] valuesV = new int[][] { 
      {1, 1, 2}, 
      {2}
  };
  private int expAsIs;
  private List&lt;Map.Entry&lt;String,Long&gt;&gt; expListK = null;
  private List&lt;Map.Entry&lt;String,Long&gt;&gt; expListV = null;

  private void addMap (int i, String strValK, int lonValK, String strValV, int lonValV) {
    expListK.add (i, new AbstractMap.SimpleEntry&lt;String, Long&gt; (strValK, (long) lonValK));
    expListV.add (i, new AbstractMap.SimpleEntry&lt;String, Long&gt; (strValV, (long) lonValV));
  }

  private int testNum;

  /***************************************************************************************************

  TestColGroup: Constructor function, which sets the instance variables for given scenario (testNum), and
                is called before each test with parameters passed via test_data (see end)

  ***************************************************************************************************/
  public TestColGroup (int   testNum,   // test scenario number
                  int   nGroups) { // number of groups

    System.out.println("Doing TestCG3 before test "+testNum+"...");
    this.lines = Arrays.asList (testLines[testNum]);
    this.delim = testDelim[testNum];
    this.colnum = testColnum[testNum];
    this.expAsIs = nGroups;
    this.testNum = testNum;
    int i = 0;
    expListK = new ArrayList&lt;Map.Entry&lt;String,Long&gt;&gt;(keysK[testNum].length);
    expListV = new ArrayList&lt;Map.Entry&lt;String,Long&gt;&gt;(keysV[testNum].length);
    for (String k : keysK[testNum]) {
      addMap (i, k, valuesK[testNum][i], keysV[testNum][i], valuesV[testNum][i]);
      i++;
    }
  }
  /***************************************************************************************************

  getGroup: Before each test method to write the test file and instantiate base object, using instance
            variables set for the scenario in TestCG3

  ***************************************************************************************************/
  @Before
  public void getGroup() {
    try {
      System.out.println("Doing setup before test "+this.testNum+"...");
      Files.write (Paths.get (testFile), lines, StandardCharsets.UTF_8);
      colGroup = new ColGroup (testFile, delim, colnum);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
  /***************************************************************************************************

  delFile: After each test method to delete the test file

  ***************************************************************************************************/
  @After
  public void delFile() {
    try {
      System.out.println("Doing teardown after test "+this.testNum+"...");
      Files.delete(Paths.get (testFile));
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
  /***************************************************************************************************

  test*: Test method for each base method; each one is run once for each record defined in test_data
         in @Parameters

  ***************************************************************************************************/
  @Test
  public void testAsIs() {

    List&lt;Map.Entry&lt;String,Long&gt;&gt; actList = colGroup.listAsIs();
    assertEquals ("(as is)", expAsIs, actList.size());
    colGroup.prList("(as is)", actList);
  }
  @Test
  public void testKey() {

    List&lt;Map.Entry&lt;String,Long&gt;&gt; actList = colGroup.sortByKey();
    assertEquals ("keys", expListK, actList);
    colGroup.prList("keys", actList);
  }
  @Test
  public void testValue() {
    List&lt;Map.Entry&lt;String,Long&gt;&gt; actList = colGroup.sortByValue();
    assertEquals ("values", expListV, actList);
    colGroup.prList("values", actList);
  }
  /***************************************************************************************************

  test_data: @Parameters section allows passing of data into tests per scenario; neater to pass in a
             pointer to the instance arrays for most of the data

  ***************************************************************************************************/
  @Parameters
  public static Collection&lt;Object[]&gt; test_data() {
    Object[][] data = new Object[][] { {0, 3}, {1, 2} }; // 2 records, columns = scenario #, # groups
    return Arrays.asList(data);
  }
}
</pre>
</div>

### JUnit Output

# Unit Test Results.

|  | Designed for use with [JUnit](http://www.junit.org/) and [Ant](http://ant.apache.org/). |
| :-- | --: |

* * *

## All Tests

| Class | Name | Status | Type | Time(s) |
| --- | --- | --- | --- | --- |
| [TestColGroup](colgroup/0_TestColGroup.html) | [testKey\[0\]](colgroup/0_TestColGroup.html#testKey[0]) | Failure | keys expected:<\[A=2, Bx=1, Cc=1\]> but was:<\[A=2, B=1, Cc=1\]>      `junit.framework.AssertionFailedError: keys expected:<[A=2, Bx=1, Cc=1]> but was:<[A=2, B=1, Cc=1]>   at colgroup.TestColGroup.testKey(TestColGroup.java:150)   ` | 0.055 |
| [TestColGroup](colgroup/0_TestColGroup.html) | [testValue\[0\]](colgroup/0_TestColGroup.html#testValue[0]) | Success |  | 0.004 |
| [TestColGroup](colgroup/0_TestColGroup.html) | [testAsIs\[0\]](colgroup/0_TestColGroup.html#testAsIs[0]) | Success |  | 0.002 |
| [TestColGroup](colgroup/0_TestColGroup.html) | [testKey\[1\]](colgroup/0_TestColGroup.html#testKey[1]) | Success |  | 0.004 |
| [TestColGroup](colgroup/0_TestColGroup.html) | [testValue\[1\]](colgroup/0_TestColGroup.html#testValue[1]) | Success |  | 0.002 |
| [TestColGroup](colgroup/0_TestColGroup.html) | [testAsIs\[1\]](colgroup/0_TestColGroup.html#testAsIs[1]) | Failure | (as is) expected:<2> but was:<1>      `junit.framework.AssertionFailedError: (as is) expected:<2> but was:<1>   at colgroup.TestColGroup.testAsIs(TestColGroup.java:143)   ` | 0.002 |

### JUnit Notes

- JUnit first creates instances of the test class for each test method and starts running the tests after each instance is created
- From JUnit 4 it is possible to data-drive testing by means of the @Parameters annotation, as implemented here, whereas I identified lack of data-driving as a common antipattern
- Test methods are identifiable only by their method names rather than full descriptions
- Scenarios are identifiable only by a number, which is even worse
- Execution of a test method (instance) is aborted on failure of any assertion
- Here there is only one test method per base method, but in general there could be several
- JUnit aborting on assertion failure means that unit tests should have one or very few assertions, with additional unit tests being generated where necessary
- Data-driving allows JUnit to generate additional unit tests from a single method at run-time for each scenario
- A good approach is to start with a single parameterized scenario, then add new scenarios just by adding data records; this is how I usually proceed in Oracle
- On assertion failure JUnit prints expected and actual values for both scalar and complex values passed to the assertion

## Oracle - utPLSQL

### utPLSQL Code

<div class="scrollbox">
<pre>
CREATE OR REPLACE PACKAGE BODY UT_Col_Group AS
/***************************************************************************************************

Description: utPLSQL unit testing for polyglot group-counting module, Col_Group
                                                                               
Modification History
Who                  When        Which What
-------------------- ----------- ----- -------------------------------------------------------------
Brendan Furey        30-Oct-2016 1.0   Created

***************************************************************************************************/

c_proc_name_asis        CONSTANT VARCHAR2(60) := 'Col_Group.ut_AIP_List_Asis';
c_proc_name_key         CONSTANT VARCHAR2(60) := 'Col_Group.ut_AIP_Sort_By_Key';
c_proc_name_value       CONSTANT VARCHAR2(60) := 'Col_Group.ut_AIP_Sort_By_vALUE';

c_file_2lis             CONSTANT L2_chr_arr := L2_chr_arr (
                                      L1_chr_arr ('0,1,Cc,3', '00,1,A,9', '000,1,B,27', '0000,1,A,81'),
                                      L1_chr_arr ('X;;1;;A', 'X;;1;;A')
);
c_prms_2lis             CONSTANT L2_chr_arr := L2_chr_arr (
                                      L1_chr_arr ('lines.csv', ',', '3'), L1_chr_arr ('lines.csv', ';;', '1')
);
c_scenario_lis          CONSTANT L1_chr_arr := L1_chr_arr ('Tie-break/single-delimiter/interior column', 'Two copies/double-delimiter/first column');

/***************************************************************************************************

ut_Setup, ut_Teardown: Mandatory procedures for utPLSQL but don't do anything here

***************************************************************************************************/
PROCEDURE ut_Setup IS
BEGIN
  NULL;
END ut_Setup;

PROCEDURE ut_Teardown IS
BEGIN
  NULL;
END ut_Teardown;

/***************************************************************************************************

Do_Test: Main local procedure for utPLSQL unit testing Col_Group methods

***************************************************************************************************/

PROCEDURE Do_Test (p_proc_name          VARCHAR2,      -- procedure name
                   p_exp_2lis           L2_chr_arr) IS -- expected values 2-d array

  /***************************************************************************************************

  Setup: Setup procedure for unit testing Col_Group package. Writes test file, then calls
         constructor API to store data in an array, line counts grouped by key

  ***************************************************************************************************/
  PROCEDURE Setup (p_file                 VARCHAR2,      -- file name
                   p_delim                VARCHAR2,      -- delimiter
                   p_colnum               PLS_INTEGER,   -- key column number in file
                   p_dat_lis              L1_chr_arr) IS -- lines to write to test file

  BEGIN

    Utils.Delete_File (p_file);
    Utils.Write_File (p_file, p_dat_lis);

    Col_Group.AIP_Load_File (p_file => p_file, p_delim => p_delim, p_colnum => p_colnum);

  END Setup;

  /***************************************************************************************************

  Call_Proc: Calls the base method according to calling procedure, and uses utPLSQL assert procedure
             to assert list counts, and for ordered methods, record lists in delimited form

  ***************************************************************************************************/
  PROCEDURE Call_Proc (p_exp_lis        L1_chr_arr,  -- expected values list (delimited records)
                       p_scenario       VARCHAR2) IS -- scenario description

    l_arr_lis           chr_int_arr;
  BEGIN

    l_arr_lis := CASE p_proc_name
                   WHEN c_proc_name_asis        THEN Col_Group.AIP_List_Asis
                   WHEN c_proc_name_key         THEN Col_Group.AIP_Sort_By_Key
                   WHEN c_proc_name_value       THEN Col_Group.AIP_Sort_By_Value
                 END;

    IF p_proc_name = c_proc_name_asis THEN

      utAssert.Eq (p_scenario || ': List count', l_arr_lis.COUNT, p_exp_lis(1), TRUE);

    ELSE

      utAssert.Eq (p_scenario || ': List count', l_arr_lis.COUNT, p_exp_lis.COUNT, TRUE);
      FOR i IN 1..LEAST (l_arr_lis.COUNT, p_exp_lis.COUNT) LOOP

        utAssert.Eq ('...Record', Utils.List_Delim (l_arr_lis(i).chr_field, l_arr_lis(i).int_field), p_exp_lis(i), TRUE);

      END LOOP;

    END IF;

  END Call_Proc;

BEGIN

  FOR i IN 1..c_file_2lis.COUNT LOOP

    Setup (p_file              => c_prms_2lis(i)(1),
           p_delim             => c_prms_2lis(i)(2),
           p_colnum            => c_prms_2lis(i)(3),
           p_dat_lis           => c_file_2lis(i));

    Call_Proc (p_exp_2lis(i), c_scenario_lis(i));

  END LOOP;

END Do_Test;

/***************************************************************************************************

ut_AIP_List_Asis: Entry procedure for utPLSQL testing Col_Group.AIP_List_Asis

***************************************************************************************************/
PROCEDURE ut_AIP_List_Asis IS

  c_proc_name           CONSTANT VARCHAR2(61) := c_proc_name_asis;
  c_exp_2lis            CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr('3'), L1_chr_arr('2'));

BEGIN

  Do_Test (c_proc_name, c_exp_2lis);

END ut_AIP_List_Asis;

/***************************************************************************************************

ut_AIP_Sort_By_Key: Entry procedure for utPLSQL testing Col_Group.AIP_Sort_By_Key

***************************************************************************************************/
PROCEDURE ut_AIP_Sort_By_Key IS

  c_proc_name           CONSTANT VARCHAR2(61) := c_proc_name_key;
  c_exp_2lis            CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr (Utils.List_Delim ('A','2'),
                                                                         Utils.List_Delim ('Bx','1'),
                                                                         Utils.List_Delim ('Cc','1')),
                                                             L1_chr_arr (Utils.List_Delim ('X','2'))
                                               );
BEGIN

  Do_Test (c_proc_name, c_exp_2lis);

END ut_AIP_Sort_By_Key;

/***************************************************************************************************

ut_AIP_Sort_By_Value: Entry procedure for utPLSQL testing Col_Group.AIP_Sort_By_Value

***************************************************************************************************/
PROCEDURE ut_AIP_Sort_By_Value IS

  c_proc_name           CONSTANT VARCHAR2(61) := c_proc_name_value;
  c_exp_2lis            CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr (Utils.List_Delim ('B','1'),
                                                                         Utils.List_Delim ('Cc','1'),
                                                                         Utils.List_Delim ('A','2')),
                                                             L1_chr_arr (Utils.List_Delim ('X','2'))
                                               );
BEGIN

  Do_Test (c_proc_name, c_exp_2lis);

END ut_AIP_Sort_By_Value;

END UT_Col_Group;
/
</pre>
</div>

### utPLSQL Output

```
.
>  FFFFFFF   AA     III  L      U     U RRRRR   EEEEEEE
>  F        A  A     I   L      U     U R    R  E
>  F       A    A    I   L      U     U R     R E
>  F      A      A   I   L      U     U R     R E
>  FFFF   A      A   I   L      U     U RRRRRR  EEEE
>  F      AAAAAAAA   I   L      U     U R   R   E
>  F      A      A   I   L      U     U R    R  E
>  F      A      A   I   L       U   U  R     R E
>  F      A      A  III  LLLLLLL  UUU   R     R EEEEEEE
.
 FAILURE: ".COL_GROUP"
.
> Individual Test Case Results:
>
SUCCESS - COL_GROUP.UT_AIP_LIST_ASIS: EQ "Tie-break/single-delimiter/interior column: List count" Expected "3" and got "3"
>
FAILURE - COL_GROUP.UT_AIP_LIST_ASIS: EQ "Two copies/double-delimiter/first column: List count" Expected "2" and got "1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_KEY: EQ "Tie-break/single-delimiter/interior column: List count" Expected "3" and got "3"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_KEY: EQ "...Record" Expected "A|2" and got "A|2"
>
FAILURE - COL_GROUP.UT_AIP_SORT_BY_KEY: EQ "...Record" Expected "Bx|1" and got "B|1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_KEY: EQ "...Record" Expected "Cc|1" and got "Cc|1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_KEY: EQ "Two copies/double-delimiter/first column: List count" Expected "1" and got "1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_KEY: EQ "...Record" Expected "X|2" and got "X|2"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_VALUE: EQ "Tie-break/single-delimiter/interior column: List count" Expected "3" and got "3"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_VALUE: EQ "...Record" Expected "B|1" and got "B|1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_VALUE: EQ "...Record" Expected "Cc|1" and got "Cc|1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_VALUE: EQ "...Record" Expected "A|2" and got "A|2"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_VALUE: EQ "Two copies/double-delimiter/first column: List count" Expected "1" and got "1"
>
SUCCESS - COL_GROUP.UT_AIP_SORT_BY_VALUE: EQ "...Record" Expected "X|2" and got "X|2"
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND

PL/SQL procedure successfully completed.
```

### utPLSQL Notes

- Code is shared between the three test methods by means of a common local procedure Do\_Test
- Data-driving is achieved by using generic arrays and looping over scenarios
- Only the simple assertion procedure utAssert.Eq is used; from experience the _complex_ utPLSQL assertions are rarely suitable, and I think it is conceptually simpler to avoid them altogether
- In general the lists of actual and expected values may have different cardinalities, so looping over them I use the minimum cardinality as loop maximum, and explicitly assert the counts; this means you may not see the detail for the unmatched records - in my own TRAPIT framework I handle this case by adding null records to the smaller list
- Delimited records are asserted to limit the number of assertions, which would matter more in realistic cases having larger number of columns
- utPLSQL does not have any data model for scenarios or output groups, with just a single description field available within the assert call to describe the scenario/group/assertion; I recommend bundling scenario and group information into this message for want of anything better
- utPLSQL allows only one test method per base method, unlike JUnit, so multiple assertions may be necessary; fortunately an assertion failure does not abort the test procedure
- utPLSQL test procedures run sequentially, in alphabetical order, unlike JUnit

## Oracle - TRAPIT

This is the framework I described in [Design Patterns for Database API Testing](https://brenpatf.github.io//migrated/dbapit-series-index/). I have renamed the framework, dropping the phrase 'unit testing' because it has connotations from JUnit of testing small, isolated pieces of code, which is not what it is intended for.

**Note, 2025:** The current version of the framework can be found on GitHub, [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester).

### TRAPIT Code

<div class="scrollbox">
<pre>
CREATE OR REPLACE PACKAGE BODY TT_Col_Group AS
/***************************************************************************************************

Description: TRAPIT (TRansactional API Testing) package for Col_Group

Further details: 'TRAPIT - TRansactional API Testing in Oracle'
                 http://aprogrammerwrites.eu/?p=1723

Modification History
Who                  When        Which What
-------------------- ----------- ----- -------------------------------------------------------------
Brendan Furey        22-Oct-2016 1.0   Created
Brendan Furey        13-Nov-2016 1.1   Utils_TT -> Utils_TT

***************************************************************************************************/

c_ms_limit              CONSTANT PLS_INTEGER := 2;
c_proc_name_asis        CONSTANT VARCHAR2(60) := 'Col_Group.tt_AIP_List_Asis';
c_proc_name_key         CONSTANT VARCHAR2(60) := 'Col_Group.tt_AIP_Sort_By_Key';
c_proc_name_value       CONSTANT VARCHAR2(60) := 'Col_Group.tt_AIP_Sort_By_vALUE';

c_file_2lis             CONSTANT L2_chr_arr := L2_chr_arr (
                                      L1_chr_arr ('0,1,Cc,3', '00,1,A,9', '000,1,B,27', '0000,1,A,81'),
                                      L1_chr_arr ('X;;1;;A', 'X;;1;;A')
);
c_prms_2lis             CONSTANT L2_chr_arr := L2_chr_arr (
                                      L1_chr_arr ('lines.csv', ',', '3'), L1_chr_arr ('lines.csv', ';;', '1')
);

c_scenario_lis          CONSTANT L1_chr_arr := L1_chr_arr ('Tie-break/single-delimiter/interior column', 'Two copies/double-delimiter/first column');
c_inp_group_lis         CONSTANT L1_chr_arr := L1_chr_arr ('Parameter', 'File');
c_inp_field_2lis        CONSTANT L2_chr_arr := L2_chr_arr (
                                      L1_chr_arr ('File Name', 'Delimiter', '*Column'),
                                      L1_chr_arr ('Line')
);
c_out_group_lis         CONSTANT L1_chr_arr := L1_chr_arr ('Sorted Array');
c_out_fields_2lis       CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr ('Key', '*Count'));

/***************************************************************************************************

Do_Test: Main local procedure for TRAPIT testing Col_Group methods

***************************************************************************************************/

PROCEDURE Do_Test (p_proc_name VARCHAR2, p_exp_2lis L2_chr_arr, p_out_group_lis L1_chr_arr, p_out_fields_2lis L2_chr_arr) IS

  l_timer_set                      PLS_INTEGER;
  l_inp_3lis                       L3_chr_arr := L3_chr_arr();
  l_act_2lis                       L2_chr_arr := L2_chr_arr();

  /***************************************************************************************************

  Setup: Setup procedure for TRAPIT testing Col_Group package. Writes test file, then calls
         constructor API to store data in an array, line counts grouped by key

  ***************************************************************************************************/
  PROCEDURE Setup (p_file                 VARCHAR2,      -- file name
                   p_delim                VARCHAR2,      -- delimiter
                   p_colnum               PLS_INTEGER,   -- key column number in file
                   p_dat_lis              L1_chr_arr,    -- lines to write to test file
                   x_inp_2lis         OUT L2_chr_arr) IS -- generic inputs list

  BEGIN

    Utils.Delete_File (p_file);
    Utils.Write_File (p_file, p_dat_lis);

    x_inp_2lis := L2_chr_arr (L1_chr_arr (Utils.List_Delim (p_file, p_delim, p_colnum)),
                              p_dat_lis);

    Col_Group.AIP_Load_File (p_file => p_file, p_delim => p_delim, p_colnum => p_colnum);

  END Setup;

  /***************************************************************************************************

  Call_Proc: Calls the base method according to calling procedure, and converts record lists to 
             delimited form, and populates the actual list for later checking

  ***************************************************************************************************/
  PROCEDURE Call_Proc (x_act_lis   OUT L1_chr_arr) IS -- actual values list (delimited records)

    l_arr_lis           chr_int_arr;
    l_act_lis           L1_chr_arr := L1_chr_arr();
  BEGIN

    l_arr_lis := CASE p_proc_name
                   WHEN c_proc_name_asis        THEN Col_Group.AIP_List_Asis
                   WHEN c_proc_name_key         THEN Col_Group.AIP_Sort_By_Key
                   WHEN c_proc_name_value       THEN Col_Group.AIP_Sort_By_Value
                 END;
    Timer_Set.Increment_Time (l_timer_set, Utils_TT.c_call_timer);

    l_act_lis.EXTEND (l_arr_lis.COUNT);
    FOR i IN 1..l_arr_lis.COUNT LOOP

      l_act_lis(i) := Utils.List_Delim (l_arr_lis(i).chr_field, l_arr_lis(i).int_field);

    END LOOP;
    x_act_lis := CASE p_proc_name
                    WHEN c_proc_name_asis        THEN L1_chr_arr(l_arr_lis.COUNT)
                    ELSE                              l_act_lis
                  END;

  END Call_Proc;

BEGIN

  l_timer_set := Utils_TT.Init (p_proc_name);
  l_act_2lis.EXTEND (c_file_2lis.COUNT);
  l_inp_3lis.EXTEND (c_file_2lis.COUNT);

  FOR i IN 1..c_file_2lis.COUNT LOOP

    Setup (p_file              => c_prms_2lis(i)(1),
           p_delim             => c_prms_2lis(i)(2),
           p_colnum            => c_prms_2lis(i)(3),
           p_dat_lis           => c_file_2lis(i),    -- data file inputs 
           x_inp_2lis          => l_inp_3lis(i));
    Timer_Set.Increment_Time (l_timer_set, 'Setup');

    Call_Proc (l_act_2lis(i));

  END LOOP;

  Utils_TT.Check_TT_Results (p_proc_name, c_scenario_lis, l_inp_3lis, l_act_2lis, p_exp_2lis, l_timer_set, c_ms_limit,
                             c_inp_group_lis, c_inp_field_2lis, p_out_group_lis, p_out_fields_2lis);

END Do_Test;

/***************************************************************************************************

tt_AIP_List_Asis: Entry procedure for TRAPIT testing Col_Group.AIP_List_Asis

***************************************************************************************************/
PROCEDURE tt_AIP_List_Asis IS

  c_proc_name           CONSTANT VARCHAR2(61) := c_proc_name_asis;
  c_exp_2lis            CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr('3'), L1_chr_arr('2'));
  c_out_group_lis       CONSTANT L1_chr_arr := L1_chr_arr ('Counts');
  c_out_fields_2lis     CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr ('*#Records'));

BEGIN

  Do_Test (c_proc_name, c_exp_2lis, c_out_group_lis, c_out_fields_2lis);

END tt_AIP_List_Asis;

/***************************************************************************************************

tt_AIP_Sort_By_Key: Entry procedure for TRAPIT testing Col_Group.AIP_Sort_By_Key

***************************************************************************************************/
PROCEDURE tt_AIP_Sort_By_Key IS

  c_proc_name             CONSTANT VARCHAR2(61) := c_proc_name_key;
  c_exp_2lis              CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr (Utils.List_Delim ('A','2'),
                                                                         Utils.List_Delim ('Bx','1'),
                                                                         Utils.List_Delim ('Cc','1')),
                                                             L1_chr_arr (Utils.List_Delim ('X','2'))
                                                 );
BEGIN

  Do_Test (c_proc_name, c_exp_2lis, c_out_group_lis, c_out_fields_2lis);

END tt_AIP_Sort_By_Key;

/***************************************************************************************************

tt_AIP_Sort_By_Value: Entry procedure for TRAPIT testing Col_Group.AIP_Sort_By_Value

***************************************************************************************************/
PROCEDURE tt_AIP_Sort_By_Value IS

  c_proc_name             CONSTANT VARCHAR2(61) := c_proc_name_value;
  c_exp_2lis              CONSTANT L2_chr_arr := L2_chr_arr (L1_chr_arr (Utils.List_Delim ('B','1'),
                                                                         Utils.List_Delim ('Cc','1'),
                                                                         Utils.List_Delim ('A','2')),
                                                             L1_chr_arr (Utils.List_Delim ('X','2'))
                                                 );
BEGIN

  Do_Test (c_proc_name, c_exp_2lis, c_out_group_lis, c_out_fields_2lis);

END tt_AIP_Sort_By_Value;

END TT_Col_Group;
/
</pre>
</div>

### TRAPIT Output

<div class="scrollbox">
<pre>
PL/SQL procedure successfully completed.

TEXT
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

TRAPIT TEST: Col_Group.tt_AIP_List_Asis
=======================================

SCENARIO 1: Tie-break/single-delimiter/interior column {
========================================================

    INPUTS
    ======

        GROUP Parameter {
        =================

            File Name  Delimiter  Column
            --------- --------- ------
            lines.csv  ,               3

        }
        =

        GROUP File {
        ============

            Line
            -----------
            0,1,Cc,3
            00,1,A,9
            000,1,B,27
            0000,1,A,81

        }
        =

    OUTPUTS
    =======

        GROUP Counts: Actual = 1, Expected = 1 {
        ========================================

            F?  #Records
            -- --------
                       3

        } 0 failed, of 1: SUCCESS
        =========================

} 0 failed, of 1: SUCCESS
=========================

SCENARIO 2: Two copies/double-delimiter/first column {
======================================================

    INPUTS
    ======

        GROUP Parameter {
        =================

            File Name  Delimiter  Column
            --------- --------- ------
            lines.csv  ;;              1

        }
        =

        GROUP File {
        ============

            Line
            -------
            X;;1;;A
            X;;1;;A

        }
        =

    OUTPUTS
    =======

        GROUP Counts: Actual = 1, Expected = 1 {
        ========================================

            F?  #Records
            -- --------
            F          1
            >          2

        } 1 failed, of 1: FAILURE
        =========================

} 1 failed, of 1: FAILURE
=========================

TIMING: Actual = 0, Expected <= 2: SUCCESS
==========================================

SUMMARY for Col_Group.tt_AIP_List_Asis
======================================

Scenario                                    # Failed  # Tests  Status
------------------------------------------ -------- ------- -------
Tie-break/single-delimiter/interior column         0        1  SUCCESS
Two copies/double-delimiter/first column           1        1  FAILURE
Timing                                             0        1  SUCCESS
------------------------------------------ -------- ------- -------
Total                                              1        3  FAILURE
------------------------------------------ -------- ------- -------

Timer Set: Col_Group.tt_AIP_List_Asis, Constructed at 13 Nov 2016 09:07:08, written at 09:07:08
===============================================================================================
[Timer timed: Elapsed (per call): 0.01 (0.000006), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Setup          0.07        0.03             2        0.03700        0.01500
Caller         0.00        0.00             2        0.00000        0.00000
(Other)        0.01        0.01             1        0.00900        0.01000
------- ---------- ---------- ------------ ------------- -------------
Total          0.08        0.04             5        0.01660        0.00800
------- ---------- ---------- ------------ ------------- -------------

TRAPIT TEST: Col_Group.tt_AIP_Sort_By_Key
=========================================

SCENARIO 1: Tie-break/single-delimiter/interior column {
========================================================

    INPUTS
    ======

        GROUP Parameter {
        =================

            File Name  Delimiter  Column
            --------- --------- ------
            lines.csv  ,               3

        }
        =

        GROUP File {
        ============

            Line
            -----------
            0,1,Cc,3
            00,1,A,9
            000,1,B,27
            0000,1,A,81

        }
        =

    OUTPUTS
    =======

        GROUP Sorted Array: Actual = 3, Expected = 3 {
        ==============================================

            F?  Key  Count
            -- --- -----
                A        2
            F   B        1
            >   Bx       1
                Cc       1

        } 1 failed, of 3: FAILURE
        =========================

} 1 failed, of 3: FAILURE
=========================

SCENARIO 2: Two copies/double-delimiter/first column {
======================================================

    INPUTS
    ======

        GROUP Parameter {
        =================

            File Name  Delimiter  Column
            --------- --------- ------
            lines.csv  ;;              1

        }
        =

        GROUP File {
        ============

            Line
            -------
            X;;1;;A
            X;;1;;A

        }
        =

    OUTPUTS
    =======

        GROUP Sorted Array: Actual = 1, Expected = 1 {
        ==============================================

            F?  Key  Count
            -- --- -----
                X        2

        } 0 failed, of 1: SUCCESS
        =========================

} 0 failed, of 1: SUCCESS
=========================

TIMING: Actual = 2, Expected <= 2: SUCCESS
==========================================

SUMMARY for Col_Group.tt_AIP_Sort_By_Key
========================================

Scenario                                    # Failed  # Tests  Status
------------------------------------------ -------- ------- -------
Tie-break/single-delimiter/interior column         1        3  FAILURE
Two copies/double-delimiter/first column           0        1  SUCCESS
Timing                                             0        1  SUCCESS
------------------------------------------ -------- ------- -------
Total                                              1        5  FAILURE
------------------------------------------ -------- ------- -------

Timer Set: Col_Group.tt_AIP_Sort_By_Key, Constructed at 13 Nov 2016 09:07:08, written at 09:07:08
=================================================================================================
[Timer timed: Elapsed (per call): 0.01 (0.000006), CPU (per call): 0.00 (0.000000), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Setup          0.03        0.03             2        0.01400        0.01500
Caller         0.00        0.00             2        0.00150        0.00000
(Other)        0.01        0.02             1        0.01000        0.02000
------- ---------- ---------- ------------ ------------- -------------
Total          0.04        0.05             5        0.00820        0.01000
------- ---------- ---------- ------------ ------------- -------------

TRAPIT TEST: Col_Group.tt_AIP_Sort_By_vALUE
===========================================

SCENARIO 1: Tie-break/single-delimiter/interior column {
========================================================

    INPUTS
    ======

        GROUP Parameter {
        =================

            File Name  Delimiter  Column
            --------- --------- ------
            lines.csv  ,               3

        }
        =

        GROUP File {
        ============

            Line
            -----------
            0,1,Cc,3
            00,1,A,9
            000,1,B,27
            0000,1,A,81

        }
        =

    OUTPUTS
    =======

        GROUP Sorted Array: Actual = 3, Expected = 3 {
        ==============================================

            F?  Key  Count
            -- --- -----
                B        1
                Cc       1
                A        2

        } 0 failed, of 3: SUCCESS
        =========================

} 0 failed, of 3: SUCCESS
=========================

SCENARIO 2: Two copies/double-delimiter/first column {
======================================================

    INPUTS
    ======

        GROUP Parameter {
        =================

            File Name  Delimiter  Column
            --------- --------- ------
            lines.csv  ;;              1

        }
        =

        GROUP File {
        ============

            Line
            -------
            X;;1;;A
            X;;1;;A

        }
        =

    OUTPUTS
    =======

        GROUP Sorted Array: Actual = 1, Expected = 1 {
        ==============================================

            F?  Key  Count
            -- --- -----
                X        2

        } 0 failed, of 1: SUCCESS
        =========================

} 0 failed, of 1: SUCCESS
=========================

TIMING: Actual = 1, Expected <= 2: SUCCESS
==========================================

SUMMARY for Col_Group.tt_AIP_Sort_By_vALUE
==========================================

Scenario                                    # Failed  # Tests  Status
------------------------------------------ -------- ------- -------
Tie-break/single-delimiter/interior column         0        3  SUCCESS
Two copies/double-delimiter/first column           0        1  SUCCESS
Timing                                             0        1  SUCCESS
------------------------------------------ -------- ------- -------
Total                                              0        5  SUCCESS
------------------------------------------ -------- ------- -------

Timer Set: Col_Group.tt_AIP_Sort_By_vALUE, Constructed at 13 Nov 2016 09:07:08, written at 09:07:08
===================================================================================================
[Timer timed: Elapsed (per call): 0.01 (0.000008), CPU (per call): 0.01 (0.000010), calls: 1000, '***' denotes corrected line below]

Timer       Elapsed         CPU         Calls       Ela/Call       CPU/Call
------- ---------- ---------- ------------ ------------- -------------
Setup          0.03        0.03             2        0.01450        0.01500
Caller         0.00        0.00             2        0.00050        0.00000
(Other)        0.01        0.02             1        0.01100        0.02000
------- ---------- ---------- ------------ ------------- -------------
Total          0.04        0.05             5        0.00820        0.01000
------- ---------- ---------- ------------ ------------- -------------

Suite Summary
=============

Package.Procedure               Tests  Fails         ELA         CPU
------------------------------ ----- ----- ---------- ----------
Col_Group.tt_AIP_List_Asis          3      1        0.08        0.04
Col_Group.tt_AIP_Sort_By_Key        5      1        0.04        0.05
Col_Group.tt_AIP_Sort_By_vALUE      5      0        0.04        0.05
------------------------------ ----- ----- ---------- ----------
Total                              13      2        0.17        0.14
------------------------------ ----- ----- ---------- ----------
Others error in (): ORA-20001: Suite BRENDAN returned error status: ORA-06512: at "LIB.UTILS_TT", line 152
ORA-06512: at "LIB.UTILS_TT", line 819
ORA-06512: at line 5

376 rows selected.
</pre>
</div>

### TRAPIT Notes

- The approach to code-sharing and data-driving is similar to that used in the utPLSQL version
- No assertions are made at all in the client code; the actual values are collected and passed to the library procedure for assertion
- The famous 'arrange-act-assert' OO pattern is therefore not followed, with no ill effects
- The output displays all inputs and outputs in 3-level format: Scenario/Group/Record with scenario descriptions, group names and column headings passed in

## Oracle - SQL Developer Unit Test

I briefly tried to use this gui framework as well, but soon gave up when I could not see how to handle the object array return values.

## Conclusion

- Some significant differences in the functionality of the frameworks between utPLSQL and JUnit have been noted
- Following design patterns for testing from the OO world may not always be advisable
- It may be best to drop the term 'unit testing' for the database altogether, with the understanding that testing only transactional APIs such as web service procedures is a much more efficient approach
- Consider using a data-driven approach to testing multiple scenarios, in testing both database and Java code, where applicable

All code and output can be seen on [polyglot\_group on GitHub](https://github.com/BrenPatF/polyglot_group), where Python, Ruby, Perl and Scala versions are also included.
