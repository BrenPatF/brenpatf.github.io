---
layout: post
title: "SQL for Length-Controlled List Aggregation"
date: 2013-08-17
migrated: true
group: recursive-sql
categories: 
  - "analytics"
  - "match_recognize"
  - "model"
  - "oracle"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "v12"
tags: 
  - "analytics"
  - "listagg"
  - "match_recognize"
  - "oracle"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "v12"
---

Recently an OTN poster asked how to return values from a column in multiple records into a single output record and column in SQL, [Multiple Rows Into One Column Field](https://forums.oracle.com/ords/apexds/post/multiple-rows-into-one-column-field-9401). In the usual version of this _list aggregation_ problem, one record is required for each distinct combination of grouping columns, with the aggregation fields delimited within the output field, and from Oracle v11.2 there is a built-in function for it, ListAgg. However, in this case the poster wanted a maximum of three values in the output field, with overflow records as necessary. Tom Kyte solved the problem in that thread essentially by adding in a calculated row number to the grouping columns, and concatenating the aggregation fields directly in the code.

Tim Hall has compiled a list of the main techniques available for the standard problem for different versions of the database, up to v11.2, here: [String Aggregation Techniques](http://www.oracle-base.com/articles/misc/string-aggregation-techniques.php)

I thought it would be an interesting and useful variation on the problem to base record overflow on concatenated length rather than on number of values. This would provide an alternative to the CLOB-based variations for cases where the length exceeds 4KB in v11.2 or earlier (v12.1 raises the string limit to 32KB), and cases where the records just need to be of limited length for display or other purposes. Here's an example on another forum of a requirement to handle long strings: [Ordering by list of strings in Oracle SQL without LISTAGG](http://stackoverflow.com/questions/11541383/ordering-by-list-of-strings-in-oracle-sql-without-listagg).

On the OTN thread, I provided an SQL solution for this variation, using recursion and the new v12.1 MATCH\_RECOGNIZE syntax for row pattern matching, and an alternative using the MODEL clause. In this article I provide modified versions with explanations and execution plans.

## SQL Analytics and Recursion

The problem variation as I have defined it is harder than it may at first appear, and in fact, can't be solved by SQL grouping and analytics alone: Recursion is required. The reason is that when considering whether a source record needs to start a new grouping record, or can be included on the prior record, the answer depends on the lengths of the fields of an unknown number of prior records. Analytic functions can only sum over a known number of prior records, and, although 'known' includes values that can be computed via a prior subquery, this is not possible here.

The approach we take involves two logical steps. In the first step, the records are processed in sequence within the partitions, and aggregate strings are accumulated for each record. When the string would exceed the maximum length a new aggregate is started and a flag is set.

Now, from the first step we have the input source number of records, with the desired aggregate strings being on the last records before any new aggregate, marked by the flag. We then just need to filter out the intermediate records.

## Test Example

We will use Oracle's standard HR schema for test data, and will aggregate employee names by department with the name list field having maximum length 80. There are 107 employees, and the desired output is:

```
      DEPT ENAME_LIST
---------- --------------------------------------------------------------------------------
        10 Whalen, Jennifer
        20 Fay, Pat; Hartstein, Michael
        30 Baida, Shelli; Colmenares, Karen; Himuro, Guy; Khoo, Alexander; Raphaely, Den
        30 Tobias, Sigal
        40 Mavris, Susan
        50 Atkinson, Mozhe; Bell, Sarah; Bissot, Laura; Bull, Alexis; Cabrio, Anthony
        50 Chung, Kelly; Davies, Curtis; Dellinger, Julia; Dilly, Jennifer
        50 Everett, Britney; Feeney, Kevin; Fleaur, Jean; Fripp, Adam; Gates, Timothy
        50 Gee, Ki; Geoni, Girard; Grant, Douglas; Jones, Vance; Kaufling, Payam
        50 Ladwig, Renske; Landry, James; Mallin, Jason; Markle, Steven; Marlow, James
        50 Matos, Randall; McCain, Samuel; Mikkilineni, Irene; Mourgos, Kevin; Nayer, Julia
        50 OConnell, Donald; Olson, TJ; Patel, Joshua; Perkins, Randall; Philtanker, Hazel
        50 Rajs, Trenna; Rogers, Michael; Sarchand, Nandita; Seo, John; Stiles, Stephen
        50 Sullivan, Martha; Taylor, Winston; Vargas, Peter; Vollman, Shanta; Walsh, Alana
        50 Weiss, Matthew
        60 Austin, David; Ernst, Bruce; Hunold, Alexander; Lorentz, Diana; Pataballa, Valli
        70 Baer, Hermann
        80 Abel, Ellen; Ande, Sundar; Banda, Amit; Bates, Elizabeth; Bernstein, David
        80 Bloom, Harrison; Cambrault, Gerald; Cambrault, Nanette; Doran, Louise
        80 Errazuriz, Alberto; Fox, Tayler; Greene, Danielle; Hall, Peter; Hutton, Alyssa
        80 Johnson, Charles; King, Janette; Kumar, Sundita; Lee, David; Livingston, Jack
        80 Marvins, Mattea; McEwen, Allan; Olsen, Christopher; Ozer, Lisa; Partners, Karen
        80 Russell, John; Sewall, Sarath; Smith, Lindsey; Smith, William; Sully, Patrick
        80 Taylor, Jonathon; Tucker, Peter; Tuvault, Oliver; Vishney, Clara; Zlotkey, Eleni
        90 De Haan, Lex; King, Steven; Kochhar, Neena
       100 Chen, John; Faviet, Daniel; Greenberg, Nancy; Popp, Luis; Sciarra, Ismael
       100 Urman, Jose Manuel
       110 Gietz, William; Higgins, Shelley
           Grant, Kimberely

29 rows selected.
```

## Recursive Subquery Factor Solutions

Recursive subquery factors are available from Oracle v11.2 up, and can be used to implement the required recursion. In this solution the flag denoting an overspill line has to be set initially on that overspill line, rather than on the preceding line, and on the last line in the partition, which are the lines we want to display. We therefore need another step.

### Setting print flag via analytic function

In v11.2, an additional subquery can be added that uses the analytic function Lead to set a flag on the required lines. This works by looking at the previously set flag on the next record (if the record exists), and setting the desired line print flag accordingly on the current record. The query is:

```sql
WITH emps_ordered AS (
SELECT department_id dept,
       CAST (last_name || ', ' || first_name AS VARCHAR2(4000)) ename,
       Row_Number () OVER (PARTITION BY department_id ORDER BY last_name, first_name) rn
  FROM hr.employees
), rsf (dept, ename_list, new_line, rn) AS (
SELECT dept, ename, 1, rn
  FROM emps_ordered
 WHERE rn = 1
 UNION ALL
SELECT r.dept,
       CASE WHEN Length (r.ename_list || '; ' || e.ename) > :rec_len THEN e.ename ELSE r.ename_list || '; ' || e.ename END,
       CASE WHEN Length (r.ename_list || '; ' || e.ename) > :rec_len THEN 1 ELSE 0 END,
       r.rn + 1
  FROM rsf r
  JOIN emps_ordered e
    ON e.dept = r.dept
   AND e.rn = r.rn + 1
), leads_v AS (
SELECT dept, ename_list, 
       new_line, Lead (new_line, 1, 1) OVER (PARTITION BY dept ORDER BY rn) line_print, rn
  FROM rsf
)
SELECT *
  FROM leads_v
 WHERE line_print = 1
 ORDER BY dept, ename_list
```

### How it works

- emps\_ordered subquery: Formats the name field and gets a row number within department ordering by the name
- rsf recursive subquery: Anchor branch selects first employee in each department; recursive branch joins the next employee based on the row number, and accumulates the name list, resetting and flagging when length dictates a new overspill line
- leads\_v subquery: Use Lead analytic function to set the print flag
- Main query: Selects rows where the print flag = 1

The output from the leads\_v subquery, before filtering, illustrates how it works:

<div class="scrollbox">

<pre>
DEPT ENAME_LIST                                                                       NEW_LINE LINE_PRINT  RN
---- -------------------------------------------------------------------------------- -------- ---------- ---
  10 Whalen, Jennifer                                                                        1          1   1
  20 Fay, Pat                                                                                1          0   1
  20 Fay, Pat; Hartstein, Michael                                                            0          1   2
  30 Baida, Shelli                                                                           1          0   1
  30 Baida, Shelli; Colmenares, Karen                                                        0          0   2
  30 Baida, Shelli; Colmenares, Karen; Himuro, Guy                                           0          0   3
  30 Baida, Shelli; Colmenares, Karen; Himuro, Guy; Khoo, Alexander                          0          0   4
  30 Baida, Shelli; Colmenares, Karen; Himuro, Guy; Khoo, Alexander; Raphaely, Den           0          1   5
  30 Tobias, Sigal                                                                           1          1   6
  40 Mavris, Susan                                                                           1          1   1
  50 Atkinson, Mozhe                                                                         1          0   1
  50 Atkinson, Mozhe; Bell, Sarah                                                            0          0   2
  50 Atkinson, Mozhe; Bell, Sarah; Bissot, Laura                                             0          0   3
  50 Atkinson, Mozhe; Bell, Sarah; Bissot, Laura; Bull, Alexis                               0          0   4
  50 Atkinson, Mozhe; Bell, Sarah; Bissot, Laura; Bull, Alexis; Cabrio, Anthony              0          1   5
  50 Chung, Kelly                                                                            1          0   6
  50 Chung, Kelly; Davies, Curtis                                                            0          0   7
  50 Chung, Kelly; Davies, Curtis; Dellinger, Julia                                          0          0   8
  50 Chung, Kelly; Davies, Curtis; Dellinger, Julia; Dilly, Jennifer                         0          1   9
  50 Everett, Britney                                                                        1          0  10
  50 Everett, Britney; Feeney, Kevin                                                         0          0  11
  50 Everett, Britney; Feeney, Kevin; Fleaur, Jean                                           0          0  12
  50 Everett, Britney; Feeney, Kevin; Fleaur, Jean; Fripp, Adam                              0          0  13
  50 Everett, Britney; Feeney, Kevin; Fleaur, Jean; Fripp, Adam; Gates, Timothy              0          1  14
  50 Gee, Ki                                                                                 1          0  15
  50 Gee, Ki; Geoni, Girard                                                                  0          0  16
  50 Gee, Ki; Geoni, Girard; Grant, Douglas                                                  0          0  17
  50 Gee, Ki; Geoni, Girard; Grant, Douglas; Jones, Vance                                    0          0  18
  50 Gee, Ki; Geoni, Girard; Grant, Douglas; Jones, Vance; Kaufling, Payam                   0          1  19
  50 Ladwig, Renske                                                                          1          0  20
  50 Ladwig, Renske; Landry, James                                                           0          0  21
  50 Ladwig, Renske; Landry, James; Mallin, Jason                                            0          0  22
  50 Ladwig, Renske; Landry, James; Mallin, Jason; Markle, Steven                            0          0  23
  50 Ladwig, Renske; Landry, James; Mallin, Jason; Markle, Steven; Marlow, James             0          1  24
  50 Matos, Randall                                                                          1          0  25
  50 Matos, Randall; McCain, Samuel                                                          0          0  26
  50 Matos, Randall; McCain, Samuel; Mikkilineni, Irene                                      0          0  27
  50 Matos, Randall; McCain, Samuel; Mikkilineni, Irene; Mourgos, Kevin                      0          0  28
  50 Matos, Randall; McCain, Samuel; Mikkilineni, Irene; Mourgos, Kevin; Nayer, Julia        0          1  29
  50 OConnell, Donald                                                                        1          0  30
  50 OConnell, Donald; Olson, TJ                                                             0          0  31
  50 OConnell, Donald; Olson, TJ; Patel, Joshua                                              0          0  32
  50 OConnell, Donald; Olson, TJ; Patel, Joshua; Perkins, Randall                            0          0  33
  50 OConnell, Donald; Olson, TJ; Patel, Joshua; Perkins, Randall; Philtanker, Hazel         0          1  34
  50 Rajs, Trenna                                                                            1          0  35
  50 Rajs, Trenna; Rogers, Michael                                                           0          0  36
  50 Rajs, Trenna; Rogers, Michael; Sarchand, Nandita                                        0          0  37
  50 Rajs, Trenna; Rogers, Michael; Sarchand, Nandita; Seo, John                             0          0  38
  50 Rajs, Trenna; Rogers, Michael; Sarchand, Nandita; Seo, John; Stiles, Stephen            0          1  39
  50 Sullivan, Martha                                                                        1          0  40
  50 Sullivan, Martha; Taylor, Winston                                                       0          0  41
  50 Sullivan, Martha; Taylor, Winston; Vargas, Peter                                        0          0  42
  50 Sullivan, Martha; Taylor, Winston; Vargas, Peter; Vollman, Shanta                       0          0  43
  50 Sullivan, Martha; Taylor, Winston; Vargas, Peter; Vollman, Shanta; Walsh, Alana         0          1  44
  50 Weiss, Matthew                                                                          1          1  45
  60 Austin, David                                                                           1          0   1
  60 Austin, David; Ernst, Bruce                                                             0          0   2
  60 Austin, David; Ernst, Bruce; Hunold, Alexander                                          0          0   3
  60 Austin, David; Ernst, Bruce; Hunold, Alexander; Lorentz, Diana                          0          0   4
  60 Austin, David; Ernst, Bruce; Hunold, Alexander; Lorentz, Diana; Pataballa, Valli        0          1   5
  70 Baer, Hermann                                                                           1          1   1
  80 Abel, Ellen                                                                             1          0   1
  80 Abel, Ellen; Ande, Sundar                                                               0          0   2
  80 Abel, Ellen; Ande, Sundar; Banda, Amit                                                  0          0   3
  80 Abel, Ellen; Ande, Sundar; Banda, Amit; Bates, Elizabeth                                0          0   4
  80 Abel, Ellen; Ande, Sundar; Banda, Amit; Bates, Elizabeth; Bernstein, David              0          1   5
  80 Bloom, Harrison                                                                         1          0   6
  80 Bloom, Harrison; Cambrault, Gerald                                                      0          0   7
  80 Bloom, Harrison; Cambrault, Gerald; Cambrault, Nanette                                  0          0   8
  80 Bloom, Harrison; Cambrault, Gerald; Cambrault, Nanette; Doran, Louise                   0          1   9
  80 Errazuriz, Alberto                                                                      1          0  10
  80 Errazuriz, Alberto; Fox, Tayler                                                         0          0  11
  80 Errazuriz, Alberto; Fox, Tayler; Greene, Danielle                                       0          0  12
  80 Errazuriz, Alberto; Fox, Tayler; Greene, Danielle; Hall, Peter                          0          0  13
  80 Errazuriz, Alberto; Fox, Tayler; Greene, Danielle; Hall, Peter; Hutton, Alyssa          0          1  14
  80 Johnson, Charles                                                                        1          0  15
  80 Johnson, Charles; King, Janette                                                         0          0  16
  80 Johnson, Charles; King, Janette; Kumar, Sundita                                         0          0  17
  80 Johnson, Charles; King, Janette; Kumar, Sundita; Lee, David                             0          0  18
  80 Johnson, Charles; King, Janette; Kumar, Sundita; Lee, David; Livingston, Jack           0          1  19
  80 Marvins, Mattea                                                                         1          0  20
  80 Marvins, Mattea; McEwen, Allan                                                          0          0  21
  80 Marvins, Mattea; McEwen, Allan; Olsen, Christopher                                      0          0  22
  80 Marvins, Mattea; McEwen, Allan; Olsen, Christopher; Ozer, Lisa                          0          0  23
  80 Marvins, Mattea; McEwen, Allan; Olsen, Christopher; Ozer, Lisa; Partners, Karen         0          1  24
  80 Russell, John                                                                           1          0  25
  80 Russell, John; Sewall, Sarath                                                           0          0  26
  80 Russell, John; Sewall, Sarath; Smith, Lindsey                                           0          0  27
  80 Russell, John; Sewall, Sarath; Smith, Lindsey; Smith, William                           0          0  28
  80 Russell, John; Sewall, Sarath; Smith, Lindsey; Smith, William; Sully, Patrick           0          1  29
  80 Taylor, Jonathon                                                                        1          0  30
  80 Taylor, Jonathon; Tucker, Peter                                                         0          0  31
  80 Taylor, Jonathon; Tucker, Peter; Tuvault, Oliver                                        0          0  32
  80 Taylor, Jonathon; Tucker, Peter; Tuvault, Oliver; Vishney, Clara                        0          0  33
  80 Taylor, Jonathon; Tucker, Peter; Tuvault, Oliver; Vishney, Clara; Zlotkey, Eleni        0          1  34
  90 De Haan, Lex                                                                            1          0   1
  90 De Haan, Lex; King, Steven                                                              0          0   2
  90 De Haan, Lex; King, Steven; Kochhar, Neena                                              0          1   3
 100 Chen, John                                                                              1          0   1
 100 Chen, John; Faviet, Daniel                                                              0          0   2
 100 Chen, John; Faviet, Daniel; Greenberg, Nancy                                            0          0   3
 100 Chen, John; Faviet, Daniel; Greenberg, Nancy; Popp, Luis                                0          0   4
 100 Chen, John; Faviet, Daniel; Greenberg, Nancy; Popp, Luis; Sciarra, Ismael               0          1   5
 100 Urman, Jose Manuel                                                                      1          1   6
 110 Gietz, William                                                                          1          0   1
 110 Gietz, William; Higgins, Shelley                                                        0          1   2
     Grant, Kimberely                                                                        1          1   1

107 rows selected.
</pre>
</div>

### Execution Plan (with filtering)

```
-----------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                    | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |           |      1 |        |     29 |00:00:00.01 |     322 |       |       |          |
|   1 |  SORT ORDER BY                               |           |      1 |    117 |     29 |00:00:00.01 |     322 |  9216 |  9216 | 8192  (0)|
|*  2 |   VIEW                                       |           |      1 |    117 |     29 |00:00:00.01 |     322 |       |       |          |
|   3 |    WINDOW SORT                               |           |      1 |    117 |    107 |00:00:00.01 |     322 | 13312 | 13312 |12288  (0)|
|   4 |     VIEW                                     |           |      1 |    117 |    107 |00:00:00.01 |     322 |       |       |          |
|   5 |      UNION ALL (RECURSIVE WITH) BREADTH FIRST|           |      1 |        |    107 |00:00:00.01 |     322 |  2048 |  2048 | 2048  (0)|
|*  6 |       VIEW                                   |           |      1 |    107 |     12 |00:00:00.01 |       7 |       |       |          |
|*  7 |        WINDOW SORT PUSHED RANK               |           |      1 |    107 |     12 |00:00:00.01 |       7 |  9216 |  9216 | 8192  (0)|
|   8 |         TABLE ACCESS FULL                    | EMPLOYEES |      1 |    107 |    107 |00:00:00.01 |       7 |       |       |          |
|*  9 |       HASH JOIN                              |           |     45 |     10 |     95 |00:00:00.01 |     315 |  1160K|  1160K| 1168K (0)|
|  10 |        RECURSIVE WITH PUMP                   |           |     45 |        |    107 |00:00:00.01 |       0 |       |       |          |
|  11 |        VIEW                                  |           |     45 |    107 |   4815 |00:00:00.01 |     315 |       |       |          |
|  12 |         WINDOW SORT                          |           |     45 |    107 |   4815 |00:00:00.01 |     315 | 18432 | 18432 |16384  (0)|
|  13 |          TABLE ACCESS FULL                   | EMPLOYEES |     45 |    107 |   4815 |00:00:00.01 |     315 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("LINE_PRINT"=1)
   6 - filter("RN"=1)
   7 - filter(ROW_NUMBER() OVER ( PARTITION BY "DEPARTMENT_ID" ORDER BY "LAST_NAME","FIRST_NAME")<=1)
   9 - access("E"."DEPT"="R"."DEPT" AND "E"."RN"="R"."RN"+1)

```

Note that the employees table is accessed 46 times, indicating an obvious performance issue.

### Using MATCH\_RECOGNIZE for filtering

In Oracle v12.1 row pattern matching was introduced, with the new keyword MATCH\_RECOGNIZE, and this provides an alternative where the filtering does not require an additional subquery. The query is:

```sql
WITH emp_ordered AS (
SELECT department_id dept,
       Row_Number() OVER (PARTITION BY department_id ORDER BY last_name || ', ' || first_name) rn, 
       last_name || ', ' || first_name ename
  FROM hr.employees e
), emp_rsf (dept, rn, ename_list, new_line) AS (
SELECT dept, rn, ename, 1
  FROM emp_ordered
 WHERE rn = 1
UNION ALL
SELECT r.dept, 
       e.rn,
       CASE WHEN Length (r.ename_list || '; ' || e.ename) > :rec_len THEN e.ename
            ELSE r.ename_list || '; ' || e.ename END,
       CASE WHEN Length (r.ename_list || '; ' || e.ename) > :rec_len THEN 1
            ELSE 0 END
  FROM emp_rsf r
  JOIN emp_ordered e
    ON e.dept = r.dept
   AND e.rn = r.rn + 1
)
SELECT dept, ename_list, cls, mtc
  FROM emp_rsf
 MATCH_RECOGNIZE (
 PARTITION BY dept
 ORDER BY rn
 MEASURES ename_list AS ename_list,
          new_line AS new_line,
          Classifier() AS cls,
          Match_Number() AS mtc
 PATTERN ( strt sm* )
 DEFINE
   sm AS sm.new_line = 0
 )
 ORDER BY 1
```

### How it works


- emps\_ordered and rsf subqueries as before
- MATCH\_RECOGNIZE clause: defines matching row sets that end in the lines to print
- MEASURES section: includes two built-in functions for illustration purposes only: Classifier() = grouping, and Match\_Number() = match number of the record
- DEFINE section: defines a grouping, sm, based on the previously set flag, new\_line, that applies if the flag = 0
- PATTERN section: ( strt sm\* ) includes an undefined grouping strt, and means to match an ordered set of records beginning with a record that does not fall into any defined grouping and continuing with zero or more records (but as many as possible) that are in the sm grouping

The output, with the extra built-in fields, helps to show how it works:

```
DEPT ENAME_LIST                                                                       CLS   MTC
---- -------------------------------------------------------------------------------- ---- ----
  10 Whalen, Jennifer                                                                 STRT    1
  20 Fay, Pat; Hartstein, Michael                                                     SM      1
  30 Baida, Shelli; Colmenares, Karen; Himuro, Guy; Khoo, Alexander; Raphaely, Den    SM      1
  30 Tobias, Sigal                                                                    STRT    2
  40 Mavris, Susan                                                                    STRT    1
  50 Atkinson, Mozhe; Bell, Sarah; Bissot, Laura; Bull, Alexis; Cabrio, Anthony       SM      1
  50 Chung, Kelly; Davies, Curtis; Dellinger, Julia; Dilly, Jennifer                  SM      2
  50 Everett, Britney; Feeney, Kevin; Fleaur, Jean; Fripp, Adam; Gates, Timothy       SM      3
  50 Gee, Ki; Geoni, Girard; Grant, Douglas; Jones, Vance; Kaufling, Payam            SM      4
  50 Ladwig, Renske; Landry, James; Mallin, Jason; Markle, Steven; Marlow, James      SM      5
  50 Matos, Randall; McCain, Samuel; Mikkilineni, Irene; Mourgos, Kevin; Nayer, Julia SM      6
  50 OConnell, Donald; Olson, TJ; Patel, Joshua; Perkins, Randall; Philtanker, Hazel  SM      7
  50 Rajs, Trenna; Rogers, Michael; Sarchand, Nandita; Seo, John; Stiles, Stephen     SM      8
  50 Sullivan, Martha; Taylor, Winston; Vargas, Peter; Vollman, Shanta; Walsh, Alana  SM      9
  50 Weiss, Matthew                                                                   STRT   10
  60 Austin, David; Ernst, Bruce; Hunold, Alexander; Lorentz, Diana; Pataballa, Valli SM      1
  70 Baer, Hermann                                                                    STRT    1
  80 Abel, Ellen; Ande, Sundar; Banda, Amit; Bates, Elizabeth; Bernstein, David       SM      1
  80 Bloom, Harrison; Cambrault, Gerald; Cambrault, Nanette; Doran, Louise            SM      2
  80 Errazuriz, Alberto; Fox, Tayler; Greene, Danielle; Hall, Peter; Hutton, Alyssa   SM      3
  80 Johnson, Charles; King, Janette; Kumar, Sundita; Lee, David; Livingston, Jack    SM      4
  80 Marvins, Mattea; McEwen, Allan; Olsen, Christopher; Ozer, Lisa; Partners, Karen  SM      5
  80 Russell, John; Sewall, Sarath; Smith, Lindsey; Smith, William; Sully, Patrick    SM      6
  80 Taylor, Jonathon; Tucker, Peter; Tuvault, Oliver; Vishney, Clara; Zlotkey, Eleni SM      7
  90 De Haan, Lex; King, Steven; Kochhar, Neena                                       SM      1
 100 Chen, John; Faviet, Daniel; Greenberg, Nancy; Popp, Luis; Sciarra, Ismael        SM      1
 100 Urman, Jose Manuel                                                               STRT    2
 110 Gietz, William; Higgins, Shelley                                                 SM      1
     Grant, Kimberely                                                                 STRT    1

29 rows selected.

```

### Execution Plan

```
--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                       | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                                |           |      1 |        |     29 |00:00:00.02 |     322 |       |       |          |
|   1 |  VIEW                                           |           |      1 |    117 |     29 |00:00:00.02 |     322 |       |       |          |
|   2 |   MATCH RECOGNIZE SORT DETERMINISTIC FINITE AUTO|           |      1 |    117 |     29 |00:00:00.02 |     322 | 13312 | 13312 |12288  (0)|
|   3 |    VIEW                                         |           |      1 |    117 |    107 |00:00:00.01 |     322 |       |       |          |
|   4 |     UNION ALL (RECURSIVE WITH) BREADTH FIRST    |           |      1 |        |    107 |00:00:00.01 |     322 |  2048 |  2048 | 2048  (0)|
|*  5 |      VIEW                                       |           |      1 |    107 |     12 |00:00:00.01 |       7 |       |       |          |
|*  6 |       WINDOW SORT PUSHED RANK                   |           |      1 |    107 |     12 |00:00:00.01 |       7 | 11264 | 11264 |10240  (0)|
|   7 |        TABLE ACCESS FULL                        | EMPLOYEES |      1 |    107 |    107 |00:00:00.01 |       7 |       |       |          |
|*  8 |      HASH JOIN                                  |           |     45 |     10 |     95 |00:00:00.02 |     315 |  1301K|  1301K| 1464K (0)|
|   9 |       VIEW                                      |           |     45 |    107 |   4815 |00:00:00.01 |     315 |       |       |          |
|  10 |        WINDOW SORT                              |           |     45 |    107 |   4815 |00:00:00.01 |     315 | 20480 | 20480 |18432  (0)|
|  11 |         TABLE ACCESS FULL                       | EMPLOYEES |     45 |    107 |   4815 |00:00:00.01 |     315 |       |       |          |
|  12 |       RECURSIVE WITH PUMP                       |           |     45 |        |    107 |00:00:00.01 |       0 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - filter("RN"=1)
   6 - filter(ROW_NUMBER() OVER ( PARTITION BY "DEPARTMENT_ID" ORDER BY "LAST_NAME"||', '||"FIRST_NAME")<=1)
   8 - access("E"."DEPT"="R"."DEPT" AND "E"."RN"="R"."RN"+1)
```

As in the earlier query, the employees table is accessed 46 times, indicating an obvious performance issue.

## MODEL Solution

Oracle introduced the MODEL clause to its SQL syntax in v10. It has a reputation for often leading to SQL that is difficult to understand and sometimes inefficient, but it is well suited to this problem.

The query is:

```sql
WITH mod AS (
SELECT *
  FROM hr.employees
MODEL
  PARTITION BY (department_id)
  DIMENSION BY (Row_Number () OVER (PARTITION BY department_id ORDER BY last_name, first_name) rn)
  MEASURES (
    last_name || ', ' || first_name ename,
    CAST (NULL AS VARCHAR2(4000)) ename_list,
    0 line_print
  )
  RULES (
    ename_list[ANY] = CASE WHEN ename_list[CV()-1] IS NULL OR Length (ename_list[CV()-1] ||  '; ' || ename[CV()]) > :rec_len THEN ename[CV()]
                           ELSE ename_list[CV()-1] ||  '; ' || ename[CV()]
                      END,
    line_print[ANY] = CASE WHEN ename[CV()+1] IS NULL OR Length (ename_list[CV()] ||  '; ' || ename[CV()+1]) > :rec_len THEN 1 END
  )
)
SELECT department_id dept, ename_list
  FROM mod
 ORDER BY department_id, ename_list
```

### How it works

- mod subquery: the aggregation and line pirint flags are calculated in a single subquery using the MODEL clause
- RULES section: first rule accumulates the aggregated lines, resetting when overspill occurs, with similar logic to that in the recursive subquery solution; the rule relies on the calculation occurring in the default order, by ascending dimension value; second rule relies on all the aggregates being calculated first by the first rule, then looks one row ahead within the partition to set the print flag
- Main query: Selects rows where the print flag = 1

### Execution Plan

```
------------------------------------------------------------------------------------------------------------------------
| Id  | Operation             | Name      | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |           |      1 |        |     29 |00:00:00.01 |       7 |       |       |          |
|   1 |  SORT ORDER BY        |           |      1 |    107 |     29 |00:00:00.01 |       7 |  9216 |  9216 | 8192  (0)|
|*  2 |   VIEW                |           |      1 |    107 |     29 |00:00:00.01 |       7 |       |       |          |
|   3 |    SQL MODEL ORDERED  |           |      1 |    107 |    107 |00:00:00.01 |       7 |   962K|   905K| 1165K (0)|
|   4 |     WINDOW SORT       |           |      1 |    107 |    107 |00:00:00.01 |       7 | 18432 | 18432 |16384  (0)|
|   5 |      TABLE ACCESS FULL| EMPLOYEES |      1 |    107 |    107 |00:00:00.01 |       7 |       |       |          |
------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("LINE_PRINT"=1)

```

Note that the employees table is accessed only once, suggesting that this may be a mmore efficient query than the recursive subquery solutions.

## Conclusion

- We have presented three SQL solutions for length-controlled list aggregation
- Two are based around the v11.2 feature recursive subquery factoring, with one also using the v12.1 feature, match\_recognise
- The third solution uses the v10.1 model clause feature, and this appears to be the simplest and fastest of the three, although no volume testing has been performed
