---
layout: post
title: "SQL Developer: Importing Unit Test Repository via Data Modeler"
date: 2015-07-11
migrated: true
group: design
categories: 
  - "data-model"
  - "design"
  - "erd"
  - "oracle"
tags: 
  - "data-modeler"
  - "oracle"
  - "sql-developer"
  - "unit-test"
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
Recently I started looking at the unit testing functionality within Oracle's SQL Developer, as a possible alternative to ut/PLSQL. Oracle's Jeff Smith has a useful starter page on this, [Unit Testing Your PL/SQL with Oracle SQL Developer](http://www.thatjeffsmith.com/archive/2014/04/unit-testing-your-plsql-with-oracle-sql-developer/). As he notes, the first thing you need to do is set up a unit test repository, which is essentially a schema within an Oracle database that SQL Developer will create for you for the unit test metadata. The schema contains 24 tables, and when I set it up I thought it would be nice to see an entity-relationship diagram for it.

I often draw these myself after running an analysis query on the foreign key network, as described in a recent post, [PL/SQL Pipelined Function for Network Analysis](http://aprogrammerwrites.eu/?p=1426). However, in this case I remembered that SQL Developer also has a data modeler component, and I thought it would be a good opportunity to learn how to use this tool to reverse-engineer a diagram from the schema. It's actually very easy, and in this article I will show the result (see another Jeff Smith post for learning resources on the data modeler, [Data Modeling](http://www.thatjeffsmith.com/data-modeling/)). For good measure, I will include outputs from my own metadata queries, and add similar for Oracle's HR demo schema (and linked schemas, in version 12.1).

**Update, 12 July 2015:** I added a section on the Oracle v11.2 Apex schema, APEX\_040000, which has 425 tables, to see how the data modeler handles larger schemas.

## Unit Test Repository Schema

I created the diagram by following the wizard from File/Import/Data Dictionary - see the link above for more information on how to use the Data Modeler.

### Data Modeler Diagram

<img src="/migrated_images/2015/07/UT_REPOS.png" alt="UT_REPOS" title="UT_REPOS" />

### Network Analysis Output

```
Network                      #Links  #Nodes Lev  Node                                                     Link
--------------------------- ------ ------ --- ------------------------------------------------------- --------------------------------------
UT_LIB_DYN_QUERIES|UT_REPOS      30      20   0  UT_LIB_DYN_QUERIES|UT_REPOS                              ROOT
                                              1  < UT_TEST_IMPL|UT_REPOS                                  ut_test_lib_dyn_queries_fk1|ut_repos
                                              2    < UT_TEST_COVERAGE_STATS|UT_REPOS                      ut_test_cov_stats_ut_t_fk1|ut_repos                                               3      > UT_TEST_IMPL_RESULTS|UT_REPOS                      ut_test_cov_stats_ut_t_fk2|ut_repos
                                              4        < UT_TEST_IMPL_VAL_RESULTS|UT_REPOS                ut_test_impl_val_res_fk3|ut_repos                                               5          > UT_TEST_IMPL|UT_REPOS*                         ut_test_impl_val_res_fk2|ut_repos
                                              5          > UT_VALIDATIONS|UT_REPOS                        ut_test_impl_val_res_fk1|ut_repos
                                              6            > UT_LIB_VALIDATIONS|UT_REPOS                  ut_validations_lib_validn_fk1|ut_repos
                                              6            > UT_TEST_IMPL|UT_REPOS*                       ut_validations_test_impl_fk1|ut_repos
                                              4        > UT_TEST_IMPL|UT_REPOS*                           ut_test_impl_results_ut_t_fk1|ut_repos
                                              4        > UT_TEST_RESULTS|UT_REPOS                         ut_test_impl_results_ut_t_fk2|ut_repos
                                              5          < UT_SUITE_ITEM_RESULTS|UT_REPOS                 ut_suite_item_results_fk2|ut_repos                                               6            > UT_SUITE_RESULTS|UT_REPOS                    ut_suite_item_results_fk1|ut_repos
                                              7              < UT_SUITE_ITEM_RESULTS|UT_REPOS*            ut_suite_item_results_fk3|ut_repos                                               7              > UT_SUITE|UT_REPOS                          ut_suite_results_fk1|ut_repos
                                              8                < UT_STARTUPS|UT_REPOS                     ut_startups_suite_fk1|ut_repos                                               9                  > UT_LIB_STARTUPS|UT_REPOS               ut_startups_lib_validn_fk1|ut_repos
                                              9                  > UT_TEST|UT_REPOS                       ut_startups_test_fk1|ut_repos
                                              0                    < UT_SUITE_ITEMS|UT_REPOS              ut_suite_items_fk2|ut_repos                                               1                      > UT_SUITE|UT_REPOS*                 ut_suite_items_fk1|ut_repos
                                              1                      > UT_SUITE|UT_REPOS*                 ut_suite_items_fk3|ut_repos
                                              0                    < UT_TEARDOWNS|UT_REPOS                ut_teardowns_test_fk1|ut_repos                                               1                      > UT_LIB_TEARDOWNS|UT_REPOS          ut_teardowns_lib_validn_fk1|ut_repos
                                              1                      > UT_SUITE|UT_REPOS*                 ut_teardowns_suite_fk1|ut_repos
                                              0                    < UT_TEST_ARGUMENTS|UT_REPOS           ut_test_arguments_fk|ut_repos
                                              1                      < UT_TEST_IMPL_ARGUMENTS|UT_REPOS    ut_test_impl_arguments_ut_fk1|ut_repos                                               2                        > UT_TEST_IMPL|UT_REPOS*           ut_test_impl_arguments_ut_fk2|ut_repos
                                              1                      < UT_TEST_IMPL_ARG_RESULTS|UT_REPOS  ut_test_impl_arg_results__fk2|ut_repos                                               2                        > UT_TEST_IMPL|UT_REPOS*           ut_test_impl_arg_results__fk1|ut_repos
                                              0                    < UT_TEST_IMPL|UT_REPOS*               ut_test_impl_ut_test_fk1|ut_repos
                                              0                    < UT_TEST_RESULTS|UT_REPOS*            ut_test_results_ut_test_fk1|ut_repos
UT_LOOKUP_CATEGORIES|UT_REPOS      2       3  0  UT_LOOKUP_CATEGORIES|UT_REPOS                            ROOT
                                              1  < UT_LOOKUP_DATATYPES|UT_REPOS                           ut_lookup_datatypes_ut_lo_fk1|ut_repos
                                              2    < UT_LOOKUP_VALUES|UT_REPOS                            ut_lookup_values_ut_looku_fk1|ut_repos
```

There is one table not shown as it has no links to any other table.

### Schema Metadata Analysis Report

#### Database Banner
```
SQL> SELECT 'Start: '||dbs.name "Database", ses.sid "Session", ses.osuser "OS User", ses.machine "Machine", To_Char (SYSDATE,'DD-MON-YYYY HH24:MI:SS') "Time",
  2   Replace (Substr(ver.banner, 1, Instr(ver.banner, '64')-4), 'Enterprise Edition Release ', '') "Version"
  3    FROM v$database dbs, v$version ver, v$session ses
  4   WHERE ver.banner LIKE 'Oracle%'
  5     AND audsid = USERENV('sessionid');

Database              Session OS User    Machine              Time                 Version
-------------------- -------- ---------- -------------------- -------------------- ------------------------------
Start: XE                 137 HP-Brendan WORKGROUP\HP-BRENDAN 11-JUL-2015 13:21:25 Oracle Database 11g Express Ed
                              \Brend_000                                           ition Release 11.2.0.2.0
```
#### Tables Summary
<div class="scrollbox">
<pre>
SQL> PROMPT '&owner' "Schema", '&tab' "Prefix"
'UT_REPOS' "Schema", '%' "Prefix"
SQL> PROMPT Tables Summary (excluding like '&nottab1', '&nottab2', '$' )
Tables Summary (excluding like '?', '?', '$' )
SQL> SELECT atc.table_name, Nvl2 (atc_w.column_name, 'Yes', NULL)    "Who?",
  2          CASE att.n_att WHEN 0 THEN To_Number(NULL) ELSE att.n_att END  "Attrs",
  3          ind_tot.n_ind "Indexes", Count(atc.column_name)      "Cols"
  4    FROM all_tab_columns   atc
  5      JOIN all_tables atb
  6        ON atb.table_name      = atc.table_name
  7       AND atb.owner     = atc.owner                             -- Join index count
  8        LEFT JOIN (SELECT i.table_owner, i.table_name, Count (i.index_name) n_ind
  9        FROM all_indexes     i
 10       GROUP BY i.table_owner, i.table_name)     ind_tot
 11          ON ind_tot.table_name    = atb.table_name
 12         AND ind_tot.table_owner   = atb.owner       -- Join attribute count
 13        LEFT JOIN (SELECT atc_att.table_name, atc_att.owner, Count(atc_att.column_name) n_att
 14              FROM all_tab_columns     atc_att
 15             WHERE atc_att.column_name   LIKE '%&notcol%'
 16             GROUP BY atc_att.table_name, atc_att.owner)   att
 17          ON att.table_name    = atb.table_name
 18         AND att.owner     = atb.owner
 19    LEFT JOIN all_tab_columns    atc_w         -- Join the Who column if it exists
 20           ON atc_w.table_name   = atb.table_name
 21          AND atc_w.owner      = atb.owner
 22          AND atc_w.column_name    = 'CREATED_BY'        -- Join index with its sequence for table
 23   WHERE atc.table_name      LIKE Upper('&tab'||'%')
 24     AND atc.table_name      NOT LIKE '%$%'
 25     AND atb.table_name      NOT LIKE '%&nottab1%'
 26     AND atb.table_name      NOT LIKE '%&nottab2%'
 27     AND atb.owner     LIKE Upper('&owner')
 28   GROUP BY atc.table_name,  Nvl2 (atc_w.column_name, 'Yes', NULL),
 29   CASE att.n_att WHEN 0 THEN To_Number(NULL) ELSE att.n_att END,
 30          ind_tot.n_ind
 31   ORDER BY 1;

TABLE_NAME                     Who?  Attrs Indexes Cols
------------------------------ ----- ----- ------- ----
UT_LIB_DYN_QUERIES             Yes               2    8
UT_LIB_STARTUPS                Yes               2    8
UT_LIB_TEARDOWNS               Yes               2    8
UT_LIB_VALIDATIONS             Yes               2    8
UT_LOOKUP_CATEGORIES           Yes               2    6
UT_LOOKUP_DATATYPES            Yes               3    8
UT_LOOKUP_VALUES               Yes               2    7
UT_METADATA                                      1    2
UT_STARTUPS                    Yes               4   10
UT_SUITE                       Yes               1    7
UT_SUITE_ITEMS                 Yes               4   10
UT_SUITE_ITEM_RESULTS          Yes               4    8
UT_SUITE_RESULTS               Yes               2   21
UT_TEARDOWNS                   Yes               4   10
UT_TEST                        Yes               2   12
UT_TEST_ARGUMENTS              Yes               2   30
UT_TEST_COVERAGE_STATS         Yes               3   13
UT_TEST_IMPL                   Yes               4   11
UT_TEST_IMPL_ARGUMENTS         Yes               3    9
UT_TEST_IMPL_ARG_RESULTS       Yes               4   12
UT_TEST_IMPL_RESULTS           Yes               3   29
UT_TEST_IMPL_VAL_RESULTS       Yes               4   15
UT_TEST_RESULTS                Yes               2   13
UT_VALIDATIONS                 Yes               3   10

24 rows selected.
</pre>
</div>

#### Triggers
<div class="scrollbox">
<pre>
SQL> COLUMN "Trigger"  FORMAT A30
SQL> COLUMN "Type"   FORMAT A30
SQL> COLUMN "Event"    FORMAT A30
SQL> 
SQL> SELECT  trg.table_name,
  2   trg.trigger_name    "Trigger",
  3   trg.trigger_type    "Type",
  4   trg.triggering_event    "Event"
  5    FROM all_triggers trg
  6   WHERE trg.table_name      LIKE Upper('&tab'||'%')
  7     AND trg.table_name      NOT LIKE '%$%'
  8     AND trg.table_name      NOT LIKE '%&nottab1%'
  9     AND trg.table_name      NOT LIKE '%&nottab2%'
 10     AND trg.owner     LIKE Upper('&owner')
 11   ORDER BY 1, 2;

TABLE_NAME                     Trigger                        Type                           Event
------------------------------ ------------------------------ ------------------------------ ------------------------------
UT_LIB_DYN_QUERIES             UT_LIB_DYN_QUERIES             BEFORE EACH ROW                INSERT
                               UT_LIB_DYN_QUERIES_UP_TRG      BEFORE EACH ROW                UPDATE
UT_LIB_STARTUPS                UT_LIB_STARTUPS                BEFORE EACH ROW                INSERT
                               UT_LIB_STARTUPS_UP_TRG         BEFORE EACH ROW                UPDATE
UT_LIB_TEARDOWNS               UT_LIB_TEARDOWNS               BEFORE EACH ROW                INSERT
                               UT_LIB_TEARDOWNS_UP_TRG        BEFORE EACH ROW                UPDATE
UT_LIB_VALIDATIONS             UT_LIB_VALIDATIONS             BEFORE EACH ROW                INSERT
                               UT_LIB_VALIDATIONS_UP_TRG      BEFORE EACH ROW                UPDATE
UT_LOOKUP_CATEGORIES           UT_LOOKUP_CAT_TRG              BEFORE EACH ROW                INSERT
                               UT_LOOKUP_CAT_UP_TRG           BEFORE EACH ROW                UPDATE
UT_LOOKUP_DATATYPES            UT_LOOKUP_DATATYPES_UP_TRG     BEFORE EACH ROW                UPDATE
                               UT_LOOKUP_DATA_TRG             BEFORE EACH ROW                INSERT
UT_LOOKUP_VALUES               UT_LOOKUP_VALUES_UP_TRG        BEFORE EACH ROW                UPDATE
                               UT_LOOKUP_VALUE_TRG            BEFORE EACH ROW                INSERT
UT_STARTUPS                    UT_STARTUPS_TRG                BEFORE EACH ROW                INSERT
                               UT_STARTUPS_UP_TRG             BEFORE EACH ROW                UPDATE
UT_SUITE                       UT_SUITE_TRG                   BEFORE EACH ROW                INSERT
                               UT_SUITE_UP_TRG                BEFORE EACH ROW                UPDATE
UT_SUITE_ITEMS                 UT_SUITE_ITEMS_TRG             BEFORE EACH ROW                INSERT
                               UT_SUITE_ITEMS_UP_TRG          BEFORE EACH ROW                UPDATE
UT_SUITE_ITEM_RESULTS          UT_SUITE_ITEM_RESULTS_TRG      BEFORE EACH ROW                INSERT
                               UT_SUITE_ITEM_RESULTS_UP_TRG   BEFORE EACH ROW                UPDATE
UT_SUITE_RESULTS               UT_SUITE_RESULTS_TRG           BEFORE EACH ROW                INSERT
                               UT_SUITE_RESULTS_UP_TRG        BEFORE EACH ROW                UPDATE
UT_TEARDOWNS                   UT_TEARDOWNS_TRG               BEFORE EACH ROW                INSERT
                               UT_TEARDOWNS_UP_TRG            BEFORE EACH ROW                UPDATE
UT_TEST                        UT_TEST_TRG                    BEFORE EACH ROW                INSERT
                               UT_TEST_UP_TRG                 BEFORE EACH ROW                UPDATE
UT_TEST_ARGUMENTS              UT_TEST_ARGUMENTS_TRG          BEFORE EACH ROW                INSERT
                               UT_TEST_ARGUMENTS_UP_TRG       BEFORE EACH ROW                UPDATE
UT_TEST_COVERAGE_STATS         UT_TEST_COVERAGE_STATS_TRG     BEFORE EACH ROW                INSERT
                               UT_TEST_COVERAGE_STATS_UP_TRG  BEFORE EACH ROW                UPDATE
UT_TEST_IMPL                   UT_TEST_IMPL_TRG               BEFORE EACH ROW                INSERT
                               UT_TEST_IMPL_UP_TRG            BEFORE EACH ROW                UPDATE
UT_TEST_IMPL_ARGUMENTS         UT_TEST_IMPL_ARGUMENTS_TRG     BEFORE EACH ROW                INSERT
                               UT_TEST_IMPL_ARGUMENTS_UP_TRG  BEFORE EACH ROW                UPDATE
UT_TEST_IMPL_ARG_RESULTS       UT_TEST_IMPL_ARG_RESULTS_TRG   BEFORE EACH ROW                INSERT
                               UT_TEST_IMPL_ARG_RES_UP_TRG    BEFORE EACH ROW                UPDATE
UT_TEST_IMPL_RESULTS           UT_TEST_IMPL_RESULTS_TRG       BEFORE EACH ROW                INSERT
                               UT_TEST_IMPL_RESULTS_UP_TRG    BEFORE EACH ROW                UPDATE
UT_TEST_IMPL_VAL_RESULTS       UT_TEST_IMPL_VAL_RESULTS_TRG   BEFORE EACH ROW                INSERT
                               UT_TEST_IMPL_VAL_RES_UP_TRG    BEFORE EACH ROW                UPDATE
UT_TEST_RESULTS                UT_TEST_RESULTS_TRG            BEFORE EACH ROW                INSERT
                               UT_TEST_RESULTS_UP_TRG         BEFORE EACH ROW                UPDATE
UT_VALIDATIONS                 UT_VALIDATIONS_TRG             BEFORE EACH ROW                INSERT
                               UT_VALIDATIONS_UP_TRG          BEFORE EACH ROW                UPDATE

46 rows selected.
</pre>
</div>
#### Foreign Keys
<div class="scrollbox">
<pre>
SQL> COLUMN "From Table" FORMAT A30
SQL> COLUMN "To Table" FORMAT A30
SQL> COLUMN "Constraint" FORMAT A30
SQL> COLUMN "Column"   FORMAT A30
SQL> COLUMN "Seq"    FORMAT 990
SQL> 
SQL> BREAK ON "From Table" ON "To Table" ON "Constraint"
SQL> 
SQL> SELECT  con_f.table_name    "From Table",
  2   con_t.table_name    "To Table",
  3   con_f.constraint_name   "Constraint",
  4   col_f.position      "Seq",
  5   col_f.column_name   "Column"
  6    FROM all_constraints     con_f
  7    JOIN all_constraints     con_t
  8      ON con_t.constraint_name   = con_f.r_constraint_name
  9     AND con_t.owner     = con_f.r_owner
 10    JOIN all_cons_columns    col_f
 11      ON con_f.constraint_type   = 'R'
 12     AND col_f.constraint_name   = con_f.constraint_name
 13     AND col_f.owner     = con_f.owner
 14   WHERE
 15      (
 16        (con_f.table_name        LIKE Upper('&tab'||'%')
 17     AND con_f.table_name        NOT LIKE '%$%'
 18     AND con_f.table_name        NOT LIKE '%&nottab1%'
 19     AND con_f.table_name        NOT LIKE '%&nottab2%'
 20     AND con_f.owner       LIKE Upper('&owner')
 21         ) OR
 22        (con_t.table_name        LIKE Upper('&tab'||'%')
 23     AND con_t.table_name        NOT LIKE '%$%'
 24     AND con_t.table_name        NOT LIKE '%&nottab1%'
 25     AND con_t.table_name        NOT LIKE '%&nottab2%'
 26     AND con_t.owner       LIKE Upper('&owner')
 27        ))
 28   ORDER BY 1, 2, 3, 4;

From Table                     To Table                       Constraint                      Seq Column
------------------------------ ------------------------------ ------------------------------ ---- ------------------------------
UT_LOOKUP_DATATYPES            UT_LOOKUP_CATEGORIES           UT_LOOKUP_DATATYPES_UT_LO_FK1     1 CAT_ID
UT_LOOKUP_VALUES               UT_LOOKUP_DATATYPES            UT_LOOKUP_VALUES_UT_LOOKU_FK1     1 DATA_ID
UT_STARTUPS                    UT_LIB_STARTUPS                UT_STARTUPS_LIB_VALIDN_FK1        1 LIB_STARTUP_ID
                               UT_SUITE                       UT_STARTUPS_SUITE_FK1             1 UT_SID
                               UT_TEST                        UT_STARTUPS_TEST_FK1              1 UT_ID
UT_SUITE_ITEMS                 UT_SUITE                       UT_SUITE_ITEMS_FK1                1 UT_SID
                                                              UT_SUITE_ITEMS_FK3                1 UT_NSID
                               UT_TEST                        UT_SUITE_ITEMS_FK2                1 UT_ID
UT_SUITE_ITEM_RESULTS          UT_SUITE_RESULTS               UT_SUITE_ITEM_RESULTS_FK1         1 UTSR_ID
                                                              UT_SUITE_ITEM_RESULTS_FK3         1 UTR_NSID
                               UT_TEST_RESULTS                UT_SUITE_ITEM_RESULTS_FK2         1 UTR_ID
UT_SUITE_RESULTS               UT_SUITE                       UT_SUITE_RESULTS_FK1              1 UT_SID
UT_TEARDOWNS                   UT_LIB_TEARDOWNS               UT_TEARDOWNS_LIB_VALIDN_FK1       1 LIB_TEARDOWN_ID
                               UT_SUITE                       UT_TEARDOWNS_SUITE_FK1            1 UT_SID
                               UT_TEST                        UT_TEARDOWNS_TEST_FK1             1 UT_ID
UT_TEST_ARGUMENTS              UT_TEST                        UT_TEST_ARGUMENTS_FK              1 UT_ID
UT_TEST_COVERAGE_STATS         UT_TEST_IMPL                   UT_TEST_COV_STATS_UT_T_FK1        1 UTI_ID
                               UT_TEST_IMPL_RESULTS           UT_TEST_COV_STATS_UT_T_FK2        1 UTIR_ID
UT_TEST_IMPL                   UT_LIB_DYN_QUERIES             UT_TEST_LIB_DYN_QUERIES_FK1       1 LIB_DYN_QUERY_ID
                               UT_TEST                        UT_TEST_IMPL_UT_TEST_FK1          1 UT_ID
UT_TEST_IMPL_ARGUMENTS         UT_TEST_ARGUMENTS              UT_TEST_IMPL_ARGUMENTS_UT_FK1     1 ARG_ID
                               UT_TEST_IMPL                   UT_TEST_IMPL_ARGUMENTS_UT_FK2     1 UTI_ID
UT_TEST_IMPL_ARG_RESULTS       UT_TEST_ARGUMENTS              UT_TEST_IMPL_ARG_RESULTS__FK2     1 ARG_ID
                               UT_TEST_IMPL                   UT_TEST_IMPL_ARG_RESULTS__FK1     1 UTI_ID
UT_TEST_IMPL_RESULTS           UT_TEST_IMPL                   UT_TEST_IMPL_RESULTS_UT_T_FK1     1 UTI_ID
                               UT_TEST_RESULTS                UT_TEST_IMPL_RESULTS_UT_T_FK2     1 UTR_ID
UT_TEST_IMPL_VAL_RESULTS       UT_TEST_IMPL                   UT_TEST_IMPL_VAL_RES_FK2          1 UTI_ID
                               UT_TEST_IMPL_RESULTS           UT_TEST_IMPL_VAL_RES_FK3          1 UTIR_ID
                               UT_VALIDATIONS                 UT_TEST_IMPL_VAL_RES_FK1          1 VAL_ID
UT_TEST_RESULTS                UT_TEST                        UT_TEST_RESULTS_UT_TEST_FK1       1 UT_ID
UT_VALIDATIONS                 UT_LIB_VALIDATIONS             UT_VALIDATIONS_LIB_VALIDN_FK1     1 LIB_VALIDATION_ID
                               UT_TEST_IMPL                   UT_VALIDATIONS_TEST_IMPL_FK1      1 UTI_ID

32 rows selected.
</pre>
</div>

#### Tables and Columns
<div class="scrollbox">
<pre>
SQL> BREAK ON table_name ON "Who?" ON "Attrs" ON "Indexes"
SQL> PROMPT Tables and Columns (omitting Who and %&notcol%)
Tables and Columns (omitting Who and %?%)
SQL> SELECT atc.table_name, Nvl2 (atc_w.column_name, 'Yes', NULL)      "Who?",
  2          CASE att.n_att WHEN 0 THEN To_Number(NULL) ELSE att.n_att END    "Attrs",
  3          ind_tot.n_ind                "Indexes",
  4          Lower (atc.column_name)|| CASE atc.nullable WHEN 'N' THEN '*' END  column_name,
  5   atc.data_type,
  6   atc.data_length               "Length",
  7                  Max(CASE ind.rn WHEN 1 THEN To_Char(aic.column_position) ELSE ' ' END) ||
  8                  Max(CASE ind.rn WHEN 2 THEN To_Char(aic.column_position) ELSE ' ' END) ||
  9                  Max(CASE ind.rn WHEN 3 THEN To_Char(aic.column_position) ELSE ' ' END) ||
 10                  Max(CASE ind.rn WHEN 4 THEN To_Char(aic.column_position) ELSE ' ' END) ||
 11                  Max(CASE ind.rn WHEN 5 THEN To_Char(aic.column_position) ELSE ' ' END) ||
 12                  Max(CASE ind.rn WHEN 6 THEN To_Char(aic.column_position) ELSE ' ' END) ||
 13                  Max(CASE ind.rn WHEN 7 THEN To_Char(aic.column_position) ELSE ' ' END) ||
 14                  Max(CASE ind.rn WHEN 8 THEN To_Char(aic.column_position) ELSE ' ' END) ||
 15                  Max(CASE ind.rn WHEN 9 THEN To_Char(aic.column_position) ELSE ' ' END) "Index Pos"
 16    FROM all_tab_columns     atc
 17      JOIN all_tables      atb
 18        ON atb.table_name      = atc.table_name
 19       AND atb.owner     = atc.owner                             -- Join index count
 20      LEFT JOIN (SELECT i.table_owner, i.table_name, Count (i.index_name) n_ind
 21              FROM all_indexes       i
 22           GROUP BY i.table_owner, i.table_name)   ind_tot
 23           ON ind_tot.table_name   = atb.table_name
 24          AND ind_tot.table_owner    = atb.owner       -- Join attribute count
 25    LEFT JOIN (SELECT atc_att.table_name, atc_att.owner, Count(atc_att.column_name) n_att
 26          FROM all_tab_columns       atc_att
 27         WHERE atc_att.column_name LIKE '%&notcol%'
 28         GROUP BY atc_att.table_name, atc_att.owner) att
 29           ON att.table_name   = atb.table_name
 30          AND att.owner      = atb.owner
 31    LEFT JOIN all_tab_columns atc_w            -- Join the Who column if it exists
 32           ON atc_w.table_name   = atb.table_name
 33          AND atc_w.owner      = atb.owner
 34          AND atc_w.column_name    = 'CREATED_BY'        -- Join index with its sequence for table
 35    LEFT JOIN (SELECT Row_Number () OVER (PARTITION BY i.table_name, i.table_owner
 36           ORDER BY i.table_name, i.table_owner, i.index_name) rn,
 37         i.table_name, i.table_owner, i.owner, i.index_name
 38          FROM all_indexes i)        ind
 39      ON ind.table_name      = atb.table_name
 40     AND ind.table_owner     = atb.owner
 41          LEFT JOIN all_ind_columns  aic         -- Join columns for the index
 42            ON aic.index_owner   = ind.owner
 43           AND aic.index_name    = ind.index_name
 44           AND aic.column_name   = atc.column_name
 45   WHERE atc.column_name     NOT LIKE '%&notcol%'
 46     AND atc.table_name      LIKE Upper('&tab'||'%')
 47     AND atc.table_name      NOT LIKE '%$%'
 48     AND atb.table_name      NOT LIKE '%&nottab1%'
 49     AND atb.table_name      NOT LIKE '%&nottab2%'
 50     AND atb.owner     LIKE Upper('&owner')
 51     AND atc.column_name     NOT IN ('CREATED_BY', 'CREATION_DATE', 'LAST_UPDATED_BY', 'LAST_UPDATE_DATE')
 52   GROUP BY atc.table_name, Nvl2 (atc_w.column_name, 'Yes', NULL), Lower (atc.column_name)||CASE atc.nullable WHEN 'N' THEN '*' END,
 53          CASE att.n_att WHEN 0 THEN To_Number(NULL) ELSE att.n_att END,
 54          ind_tot.n_ind, atc.data_type, atc.data_length
 55   ORDER BY 1, 2;

TABLE_NAME                     Who?  Attrs Indexes COLUMN_NAME                    DATA_TYPE     Length Index Pos
------------------------------ ----- ----- ------- ------------------------------ ------------- ------ ----------
UT_LIB_DYN_QUERIES             Yes               2 created_on*                    TIMESTAMP(6)      11
                                                   lib_dyn_query*                 CLOB            4000
                                                   lib_dyn_query_class*           VARCHAR2         120
                                                   lib_dyn_query_id*              VARCHAR2          40  1
                                                   lib_dyn_query_name*            VARCHAR2         120 1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
UT_LIB_STARTUPS                Yes               2 created_on*                    TIMESTAMP(6)      11
                                                   lib_startup*                   CLOB            4000
                                                   lib_startup_class*             VARCHAR2         120
                                                   lib_startup_id*                VARCHAR2          40  1
                                                   lib_startup_name*              VARCHAR2         120 1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
UT_LIB_TEARDOWNS               Yes               2 created_on*                    TIMESTAMP(6)      11
                                                   lib_teardown*                  CLOB            4000
                                                   lib_teardown_class*            VARCHAR2         120
                                                   lib_teardown_id*               VARCHAR2          40  1
                                                   lib_teardown_name*             VARCHAR2         120 1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
UT_LIB_VALIDATIONS             Yes               2 created_on*                    TIMESTAMP(6)      11
                                                   lib_validation*                CLOB            4000
                                                   lib_validation_class*          VARCHAR2         120
                                                   lib_validation_id*             VARCHAR2          40  1
                                                   lib_validation_name*           VARCHAR2         120 1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
UT_LOOKUP_CATEGORIES           Yes               2 created_on*                    TIMESTAMP(6)      11
                                                   id*                            VARCHAR2          40 1
                                                   name                           VARCHAR2         120  1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
UT_LOOKUP_DATATYPES            Yes               3 cat_id                         VARCHAR2          40  11
                                                   created_on*                    TIMESTAMP(6)      11
                                                   id*                            VARCHAR2          40 1
                                                   type_id                        NUMBER            22
                                                   type_string                    VARCHAR2         120  2
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
UT_LOOKUP_VALUES               Yes               2 created_on*                    TIMESTAMP(6)      11
                                                   data_id                        VARCHAR2          40  1
                                                   id*                            VARCHAR2          40 1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   value                          CLOB            4000
UT_METADATA                                      1 name*                          VARCHAR2         120 1
                                                   value*                         VARCHAR2        2000
UT_STARTUPS                    Yes               4 created_on*                    TIMESTAMP(6)      11
                                                   index_no*                      NUMBER            22
                                                   lib_startup_id                 VARCHAR2          40 1
                                                   startup                        CLOB            4000
                                                   startup_id*                    VARCHAR2          40  1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id                          VARCHAR2          40    1
                                                   ut_sid                         VARCHAR2          40   1
UT_SUITE                       Yes               1 coverage                       NUMBER            22
                                                   created_on*                    TIMESTAMP(6)      11
                                                   name*                          VARCHAR2         120
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_sid*                        VARCHAR2          40 1
UT_SUITE_ITEMS                 Yes               4 created_on*                    TIMESTAMP(6)      11
                                                   run_start*                     VARCHAR2           1
                                                   run_tear*                      VARCHAR2           1
                                                   sequence*                      NUMBER            22
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id                          VARCHAR2          40   12
                                                   ut_nsid                        VARCHAR2          40  1 3
                                                   ut_sid*                        VARCHAR2          40 1  1
UT_SUITE_ITEM_RESULTS          Yes               4 created_on*                    TIMESTAMP(6)      11
                                                   sequence*                      NUMBER            22
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   utr_id                         VARCHAR2          40  1 2
                                                   utr_nsid                       VARCHAR2          40   13
                                                   utsr_id*                       VARCHAR2          40 1  1
UT_SUITE_RESULTS               Yes               2 coverage*                      NUMBER            22
                                                   created_on*                    TIMESTAMP(6)      11
                                                   message                        VARCHAR2        2000
                                                   name*                          VARCHAR2         120
                                                   run_date*                      TIMESTAMP(6)      11
                                                   startup_duration               NUMBER            22
                                                   startup_end_time               VARCHAR2          20
                                                   startup_message                VARCHAR2        2000
                                                   startup_start_time             VARCHAR2          20
                                                   startup_status                 VARCHAR2          20
                                                   status*                        VARCHAR2          20
                                                   teardown_duration              NUMBER            22
                                                   teardown_end_time              VARCHAR2          20
                                                   teardown_message               VARCHAR2        2000
                                                   teardown_start_time            VARCHAR2          20
                                                   teardown_status                VARCHAR2          20
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_sid*                        VARCHAR2          40 1
                                                   utsr_id*                       VARCHAR2          40  1
UT_TEARDOWNS                   Yes               4 created_on*                    TIMESTAMP(6)      11
                                                   index_no*                      NUMBER            22
                                                   lib_teardown_id                VARCHAR2          40 1
                                                   teardown                       CLOB            4000
                                                   teardown_id*                   VARCHAR2          40  1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id                          VARCHAR2          40    1
                                                   ut_sid                         VARCHAR2          40   1
UT_TEST                        Yes               2 connection_name                VARCHAR2         120
                                                   coverage                       NUMBER            22
                                                   created_on*                    TIMESTAMP(6)      11
                                                   name*                          VARCHAR2         120  1
                                                   object_call                    VARCHAR2         120
                                                   object_name*                   VARCHAR2         120
                                                   object_owner*                  VARCHAR2         120
                                                   object_type*                   VARCHAR2         120
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id*                         VARCHAR2          40 1
UT_TEST_ARGUMENTS              Yes               2 arg_id*                        VARCHAR2          40  1
                                                   argument_name                  VARCHAR2          30
                                                   char_length                    NUMBER            22
                                                   char_used                      VARCHAR2           1
                                                   character_set_name             VARCHAR2          44
                                                   created_on*                    TIMESTAMP(6)      11
                                                   data_length                    NUMBER            22
                                                   data_level*                    NUMBER            22
                                                   data_precision                 NUMBER            22
                                                   data_scale                     NUMBER            22
                                                   data_type                      VARCHAR2          30
                                                   default_length                 NUMBER            22
                                                   in_out                         VARCHAR2           9
                                                   object_id*                     NUMBER            22
                                                   object_name                    VARCHAR2          30
                                                   overload                       VARCHAR2          40
                                                   owner*                         VARCHAR2          30
                                                   package_name                   VARCHAR2          30
                                                   pls_type                       VARCHAR2          30
                                                   position*                      NUMBER            22
                                                   radix                          NUMBER            22
                                                   sequence*                      NUMBER            22
                                                   type_link                      VARCHAR2         128
                                                   type_name                      VARCHAR2          30
                                                   type_owner                     VARCHAR2          30
                                                   type_subname                   VARCHAR2          30
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id                          VARCHAR2          40 1
UT_TEST_COVERAGE_STATS         Yes               3 created_on*                    TIMESTAMP(6)      11
                                                   line                           NUMBER            22
                                                   text                           VARCHAR2        4000
                                                   total_occur                    NUMBER            22
                                                   total_time                     NUMBER            22
                                                   unit_name                      VARCHAR2          30
                                                   unit_owner                     VARCHAR2          30
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   utc_id*                        VARCHAR2          40 1
                                                   uti_id                         VARCHAR2          40  1
                                                   utir_id                        VARCHAR2          40   1
UT_TEST_IMPL                   Yes               4 created_on*                    TIMESTAMP(6)      11
                                                   dynamic_value_query            CLOB            4000
                                                   expected_return                VARCHAR2          20
                                                   expected_return_error          VARCHAR2         200
                                                   lib_dyn_query_id               VARCHAR2          40    1
                                                   name*                          VARCHAR2         120  2
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id*                         VARCHAR2          40  11
                                                   uti_id*                        VARCHAR2          40 1
UT_TEST_IMPL_ARGUMENTS         Yes               3 arg_id*                        VARCHAR2          40 21
                                                   created_on*                    TIMESTAMP(6)      11
                                                   input_value                    CLOB            4000
                                                   output_value                   CLOB            4000
                                                   test_outval*                   NUMBER            22
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   uti_id*                        VARCHAR2          40 1 1
UT_TEST_IMPL_ARG_RESULTS       Yes               4 arg_id*                        VARCHAR2          40  1
                                                   created_on*                    TIMESTAMP(6)      11
                                                   message                        VARCHAR2        2000
                                                   name                           VARCHAR2         120
                                                   run_date*                      TIMESTAMP(6)      11
                                                   status*                        VARCHAR2          20
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   uti_id*                        VARCHAR2          40 1
                                                   utiar_id*                      VARCHAR2          40    1
                                                   utir_id                        VARCHAR2          40   1
UT_TEST_IMPL_RESULTS           Yes               3 created_on*                    TIMESTAMP(6)      11
                                                   duration                       NUMBER            22
                                                   end_time                       VARCHAR2          20
                                                   message                        VARCHAR2        2000
                                                   name*                          VARCHAR2         120
                                                   op_call_duration               NUMBER            22
                                                   op_call_end_time               VARCHAR2          20
                                                   op_call_message                VARCHAR2        2000
                                                   op_call_start_time             VARCHAR2          20
                                                   op_call_status                 VARCHAR2          20
                                                   run_date*                      TIMESTAMP(6)      11
                                                   start_time                     VARCHAR2          20
                                                   startup_duration               NUMBER            22
                                                   startup_end_time               VARCHAR2          20
                                                   startup_message                VARCHAR2        2000
                                                   startup_start_time             VARCHAR2          20
                                                   startup_status                 VARCHAR2          20
                                                   status*                        VARCHAR2          20
                                                   teardown_duration              NUMBER            22
                                                   teardown_end_time              VARCHAR2          20
                                                   teardown_message               VARCHAR2        2000
                                                   teardown_start_time            VARCHAR2          20
                                                   teardown_status                VARCHAR2          20
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   uti_id*                        VARCHAR2          40  1
                                                   utir_id*                       VARCHAR2          40 1
                                                   utr_id*                        VARCHAR2          40   1
UT_TEST_IMPL_VAL_RESULTS       Yes               4 created_on*                    TIMESTAMP(6)      11
                                                   message                        VARCHAR2        2000
                                                   run_date                       TIMESTAMP(6)      11
                                                   status*                        VARCHAR2          20
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   uti_id*                        VARCHAR2          40  1
                                                   utir_id                        VARCHAR2          40   1
                                                   utivr_id*                      VARCHAR2          40    1
                                                   val_duration                   NUMBER            22
                                                   val_end_time                   VARCHAR2          20
                                                   val_id*                        VARCHAR2          40 1
                                                   val_start_time                 VARCHAR2          20
                                                   val_type*                      VARCHAR2          40
UT_TEST_RESULTS                Yes               2 connection_name                VARCHAR2         120
                                                   coverage*                      NUMBER            22
                                                   created_on*                    TIMESTAMP(6)      11
                                                   message                        VARCHAR2        2000
                                                   name*                          VARCHAR2         120
                                                   run_date*                      TIMESTAMP(6)      11
                                                   status*                        VARCHAR2          20
                                                   test_user_name                 VARCHAR2         120
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   ut_id*                         VARCHAR2          40  1
                                                   utr_id*                        VARCHAR2          40 1
UT_VALIDATIONS                 Yes               3 apply_validation*              NUMBER            22
                                                   created_on*                    TIMESTAMP(6)      11
                                                   index_no*                      NUMBER            22
                                                   lib_validation_id              VARCHAR2          40 1
                                                   updated_by*                    VARCHAR2         120
                                                   updated_on*                    TIMESTAMP(6)      11
                                                   uti_id*                        VARCHAR2          40   1
                                                   validation                     CLOB            4000
                                                   validation_id*                 VARCHAR2          40  1

252 rows selected.
</pre>
</div>
#### Indexes
<div class="scrollbox">
<pre>
SQL> BREAK ON table_name ON index_name
SQL> SELECT  atb.table_name, ind.index_name||CASE ind.uniqueness WHEN 'UNIQUE' THEN '*' END index_name,
  2   aic.column_position "Seq", Lower(aic.column_name) column_name
  3    FROM all_tables      atb
  4    LEFT JOIN all_indexes      ind
  5           ON ind.table_name   = atb.table_name
  6          AND ind.table_owner    = atb.owner
  7         LEFT JOIN all_ind_columns aic
  8                ON aic.index_name    = ind.index_name
  9               AND aic.index_owner = ind.owner
 10   WHERE atb.table_name      LIKE Upper('&tab'||'%')
 11     AND atb.table_name      NOT LIKE '%$%'
 12     AND atb.table_name      NOT LIKE '%&nottab1%'
 13     AND atb.table_name      NOT LIKE '%&nottab2%'
 14     AND atb.owner     LIKE Upper('&owner')
 15   ORDER BY 1, 2, 3;

TABLE_NAME                     INDEX_NAME                       Seq COLUMN_NAME
------------------------------ ------------------------------- ---- ------------------------------
UT_LIB_DYN_QUERIES             UT_LIB_DYN_QUERIES_NAME*           1 lib_dyn_query_name
                               UT_LIB_DYN_QUERIES_PK*             1 lib_dyn_query_id
UT_LIB_STARTUPS                UT_LIB_STARTUPS_NAME*              1 lib_startup_name
                               UT_LIB_STARTUPS_PK*                1 lib_startup_id
UT_LIB_TEARDOWNS               UT_LIB_TEARDOWNS_NAME*             1 lib_teardown_name
                               UT_LIB_TEARDOWNS_PK*               1 lib_teardown_id
UT_LIB_VALIDATIONS             UT_LIB_VALIDATIONS_NAME*           1 lib_validation_name
                               UT_LIB_VALIDATIONS_PK*             1 lib_validation_id
UT_LOOKUP_CATEGORIES           UT_LOOKUP_CATEGORIES_PK*           1 id
                               UT_LOOKUP_CATEGORIES_UK1*          1 name
UT_LOOKUP_DATATYPES            UT_LOOKUP_DATATYPES_PK*            1 id
                               UT_LOOKUP_DATATYPES_UK1*           1 cat_id
                                                                  2 type_string
                               UT_LOOKUP_DTS_CAT_ID_IX            1 cat_id
UT_LOOKUP_VALUES               UT_LOOKUP_VALUES_PK*               1 id
                               UT_LOOKUP_VALUES_UT_LOOKU_IX       1 data_id
UT_METADATA                    UT_METADATA_PK*                    1 name
UT_STARTUPS                    UT_STARTUPS_LIB_VALIDN_IX          1 lib_startup_id
                               UT_STARTUPS_PK*                    1 startup_id
                               UT_STARTUPS_SUITE_IX               1 ut_sid
                               UT_STARTUPS_TEST_IX                1 ut_id
UT_SUITE                       UT_SUITE_PK*                       1 ut_sid
UT_SUITE_ITEMS                 UT_SUITE_ITEMS_IX1                 1 ut_sid
                               UT_SUITE_ITEMS_IX2                 1 ut_nsid
                               UT_SUITE_ITEMS_IX3                 1 ut_id
                               UT_SUITE_ITEMS_UK*                 1 ut_sid
                                                                  2 ut_id
                                                                  3 ut_nsid
UT_SUITE_ITEM_RESULTS          UT_SUITE_ITEM_RESULTS_FK1          1 utsr_id
                               UT_SUITE_ITEM_RESULTS_FK2          1 utr_id
                               UT_SUITE_ITEM_RESULTS_FK3          1 utr_nsid
                               UT_SUITE_ITEM_RESULTS_UK*          1 utsr_id
                                                                  2 utr_id
                                                                  3 utr_nsid
UT_SUITE_RESULTS               UT_SUITE_RESULTS_IX                1 ut_sid
                               UT_SUITE_RESULTS_PK*               1 utsr_id
UT_TEARDOWNS                   UT_TEARDOWNS_LIB_VALIDN_IX         1 lib_teardown_id
                               UT_TEARDOWNS_PK*                   1 teardown_id
                               UT_TEARDOWNS_SUITE_IX              1 ut_sid
                               UT_TEARDOWNS_TEST_IX               1 ut_id
UT_TEST                        UT_TEST_PK*                        1 ut_id
                               UT_TEST_UK1*                       1 name
UT_TEST_ARGUMENTS              UT_TEST_ARGUMENTS_IX               1 ut_id
                               UT_TEST_ARGUMENTS_PK*              1 arg_id
UT_TEST_COVERAGE_STATS         UT_TEST_COVERAGE_STATS_PK*         1 utc_id
                               UT_TEST_COV_STATS_UT_T_IX1         1 uti_id
                               UT_TEST_COV_STATS_UT_T_IX2         1 utir_id
UT_TEST_IMPL                   UT_TEST_IMPL_PK*                   1 uti_id
                               UT_TEST_IMPL_UK1*                  1 ut_id
                                                                  2 name
                               UT_TEST_IMPL_UT_TEST_IX            1 ut_id
                               UT_TEST_LIB_DYN_QUERIES_IX         1 lib_dyn_query_id
UT_TEST_IMPL_ARGUMENTS         UT_TEST_IMPL_ARGUMENTS_PK*         1 uti_id
                                                                  2 arg_id
                               UT_TEST_IMPL_ARGUMENTS_UT_IX1      1 arg_id
                               UT_TEST_IMPL_ARGUMENTS_UT_IX2      1 uti_id
UT_TEST_IMPL_ARG_RESULTS       UT_TEST_IMPL_ARG_RESULTS_IX1       1 uti_id
                               UT_TEST_IMPL_ARG_RESULTS_IX2       1 arg_id
                               UT_TEST_IMPL_ARG_RESULTS_IX3       1 utir_id
                               UT_TEST_IMPL_ARG_RESULTS_PK*       1 utiar_id
UT_TEST_IMPL_RESULTS           UT_TEST_IMPL_RESULTS_PK*           1 utir_id
                               UT_TEST_IMPL_RESULTS_UT_T_IX1      1 uti_id
                               UT_TEST_IMPL_RESULTS_UT_T_IX2      1 utr_id
UT_TEST_IMPL_VAL_RESULTS       UT_TEST_IMPL_VAL_RES_IX1           1 val_id
                               UT_TEST_IMPL_VAL_RES_IX2           1 uti_id
                               UT_TEST_IMPL_VAL_RES_IX3           1 utir_id
                               UT_TEST_IMPL_VAL_RES_PK*           1 utivr_id
UT_TEST_RESULTS                UT_TEST_RESULTS_PK*                1 utr_id
                               UT_TEST_RESULTS_UT_TEST_IX1        1 ut_id
UT_VALIDATIONS                 UT_VALIDATIONS_LIB_VALIDN_IX       1 lib_validation_id
                               UT_VALIDATIONS_PK*                 1 validation_id
                               UT_VALIDATIONS_TEST_IMPL_IX        1 uti_id

72 rows selected.
</pre>
</div>

#### Tables with no Who columns / No unique indexes/ Id only unique indexes
<div class="scrollbox">
<pre>
Tables with no Who columns / No unique indexes/ Id only unique indexes
SQL> SELECT atb.table_name,  CASE WHEN atc_w.column_name IS NULL THEN 'X' END "Who?",
  2                          CASE WHEN uni.maxind IS NULL THEN 'X' END "No UID?",
  3                          CASE WHEN uni.maxind = 1 THEN 'X' END "Id UID?"
  4    FROM all_tables      atb
  5    LEFT JOIN all_tab_columns    atc_w
  6           ON atc_w.table_name   = atb.table_name
  7          AND atc_w.owner      = atb.owner
  8          AND atc_w.column_name    = 'CREATED_BY'
  9    LEFT JOIN (SELECT ind.table_name, ind.table_owner, Max(CASE WHEN Substr(aic.column_name, Length(aic.column_name)-2) IS NULL THEN 0
 10                     WHEN Substr(aic.column_name, Length(aic.column_name)-2) = '_ID' THEN 1
 11                     ELSE 2 END) maxind
 12                 FROM all_indexes ind
 13                 LEFT JOIN all_ind_columns aic
 14                        ON aic.index_name         = ind.index_name
 15                       AND aic.index_owner        = ind.owner
 16                       AND aic.column_position    = 1
 17                WHERE ind.uniqueness   = 'UNIQUE'
 18                GROUP BY ind.table_name, ind.table_owner) uni
 19           ON uni.table_name     = atb.table_name
 20          AND uni.table_owner      = atb.owner
 21   WHERE atb.table_name      LIKE Upper('&tab'||'%')
 22     AND atb.table_name      NOT LIKE '%$%'
 23     AND atb.table_name      NOT LIKE '%&nottab1%'
 24     AND atb.table_name      NOT LIKE '%&nottab2%'
 25     AND atb.owner     LIKE Upper('&owner')
 26     AND (atc_w.column_name    IS NULL OR uni.maxind IN (0, 1))
 27   ORDER BY 1;

TABLE_NAME                     Who?  No UID? Id UID?
------------------------------ ----- ------- -------
UT_METADATA                    X
UT_STARTUPS                                  X
UT_SUITE_ITEM_RESULTS                        X
UT_SUITE_RESULTS                             X
UT_TEARDOWNS                                 X
UT_TEST_ARGUMENTS                            X
UT_TEST_COVERAGE_STATS                       X
UT_TEST_IMPL                                 X
UT_TEST_IMPL_ARGUMENTS                       X
UT_TEST_IMPL_ARG_RESULTS                     X
UT_TEST_IMPL_RESULTS                         X
UT_TEST_IMPL_VAL_RESULTS                     X
UT_TEST_RESULTS                              X
UT_VALIDATIONS                               X

14 rows selected.

SQL> SELECT 'End: '||name "Database", '&owner' "Schema", '&tab' "Prefix", To_Char(SYSDATE,'DD-MON-YYYY HH24:MI:SS') FROM v$database;

Database             Schema          Prefix     TO_CHAR(SYSDATE,'DD-MON-YYYYH
-------------------- --------------- ---------- -----------------------------
End: XE              UT_REPOS        %          11-JUL-2015 13:21:35
</pre>
</div>

## HR Demo Schemas (v12)

I had to import the three schemas (HR, OE and PM) one at a time, merging with the previous one, am not sure if you can do it in one go, or if the diagram is affected by the order of import.

### Data Modeler Diagram

<img src="/migrated_images/2015/07/HR.png" alt="HR" title="HR" />

### Manual Visio Diagram

For comparison, here is a manual diagram, deliberately omitting column and other detail. It was done as an illustration of the network analysis program output, and I therefore did not include useful information such as on relationship optionality.

<img src="/migrated_images/2015/07/Networks-HR-v1.0.png" alt="Networks - HR, v1.0" title="Networks - HR, v1.0" />

### Network Analysis Output

I copied this from my article above.

```
Network       #Links  #Nodes Lev  Node                                     Link
------------ ------ ------ --- --------------------------------------- -------------------------------
COUNTRIES|HR      21      16   0  COUNTRIES|HR                             ROOT
                               1  < LOCATIONS|HR                           loc_c_id_fk|hr
                               2    < DEPARTMENTS|HR                       dept_loc_fk|hr                                3      > EMPLOYEES|HR                       dept_mgr_fk|hr
                               4        < CUSTOMERS|OE                     customers_account_manager_fk|oe
                               5          < ORDERS|OE                      orders_customer_id_fk|oe                                6            > EMPLOYEES|HR*                orders_sales_rep_fk|oe
                               6            < ORDER_ITEMS|OE               order_items_order_id_fk|oe                                7              > PRODUCT_INFORMATION|OE     order_items_product_id_fk|oe
                               8                < INVENTORIES|OE           inventories_product_id_fk|oe                                9                  > WAREHOUSES|OE          inventories_warehouses_fk|oe
                              10                    > LOCATIONS|HR*        warehouses_location_fk|oe
                               8                < ONLINE_MEDIA|PM          loc_c_id_fk|pm
                               8                < PRINT_MEDIA|PM           printmedia_fk|pm
                               8                < PRODUCT_DESCRIPTIONS|OE  pd_product_id_fk|oe                                4        > DEPARTMENTS|HR*                  emp_dept_fk|hr
                               4        = EMPLOYEES|HR*                    emp_manager_fk|hr
                               4        > JOBS|HR                          emp_job_fk|hr
                               5          < JOB_HISTORY|HR                 jhist_job_fk|hr                                6            > DEPARTMENTS|HR*              jhist_dept_fk|hr
                               6            > EMPLOYEES|HR*                jhist_emp_fk|hr
                               1  > REGIONS|HR                             countr_reg_fk|hr
```

### Schema Metadata Analysis Report - not included

## Apex Schema - APEX\_040000 (v11.22)

This imported quickly, with 425 tables. However, the diagram was not so useful. Trying to print it to .png or .jpg (via File/Print Diagram/To Image File) silently failed; printing to .pdf worked, but the reader opens at a zoom level of 1.41% and it's not practical to navigate the links. Maybe text-based analysis reports are more useful for the larger schemas.

### Data Modeler Diagram

Here is a screenshot of the modeler diagram:
<img src="/migrated_images/2015/07/APEX_040000_DM-Shot.png" alt="APEX_040000_DM-Shot" title="APEX_040000_DM-Shot" />

### Network Analysis Output

I include my standard summary listings in this case, showing that the schema splits into 21 sub-networks, with 298 tables having foreign key links, the remaining 127 being thereby excluded from this report.

#### Network detail
<div class="scrollbox">
<pre>
SQL> @..\sql\R_Net

Network                                     #Links  #Nodes    Lev Node                                                                   Link
------------------------------------------ ------- ------- ------ ---------------------------------------------------------------------- ------------------------------------------
APEX$_WS_FILES|APEX_040000                       4       5      0 APEX$_WS_FILES|APEX_040000                                             ROOT
                                                                1 > APEX$_WS_ROWS|APEX_040000                                            apex$_ws_files_fk|apex_040000
                                                                2   < APEX$_WS_LINKS|APEX_040000                                         apex$_ws_links_fk|apex_040000
                                                                2   < APEX$_WS_NOTES|APEX_040000                                         apex$_ws_notes_fk|apex_040000
                                                                2   < APEX$_WS_TAGS|APEX_040000                                          apex$_ws_tags_fk|apex_040000
WWV_FLOWS|APEX_040000                          286     236      0 WWV_FLOWS|APEX_040000                                                  ROOT
                                                                1 < WWV_FLOW_ALTERNATE_CONFIG|APEX_040000                                wwv_flow_alt_cfg_fk|apex_040000
                                                                2   < WWV_FLOW_ALT_CONFIG_DETAIL|APEX_040000                             wwv_flow_alt_cfg_d_fk|apex_040000
                                                                1 < WWV_FLOW_APP_COMMENTS|APEX_040000                                    wwv_flow_app_comments_fk|apex_040000
                                                                1 < WWV_FLOW_BANNER|APEX_040000                                          wwv_flow_banner_fk|apex_040000
                                                                1 < WWV_FLOW_BUTTON_TEMPLATES|APEX_040000                                wwv_flow_buttont_fk|apex_040000
                                                                1 < WWV_FLOW_CALS|APEX_040000                                            wwv_flow_cal_to_flow_fk|apex_040000
                                                                2   > WWV_FLOW_PAGE_PLUGS|APEX_040000                                    wwv_flow_plug_calendar_fk|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_plug_to_flow_fk|apex_040000
                                                                3     < WWV_FLOW_FLASH_CHARTS_5|APEX_040000                              wwv_flow_flash_charts_5_fk2|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_flash_charts_5_fk|apex_040000
                                                                4       < WWV_FLOW_FLASH_CHART5_SERIES|APEX_040000                       wwv_flow_flash_5_series_fk|apex_040000
                                                                4       < WWV_FLOW_FLASH_CHARTS_5_DASH|APEX_040000                       wwv_flow_flash_charts5_dash_fk|apex_040000
                                                                3     < WWV_FLOW_FLASH_CHARTS|APEX_040000                                wwv_flow_flash_charts_fk2|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_flash_charts_fk|apex_040000
                                                                4       < WWV_FLOW_FLASH_CHART_SERIES|APEX_040000                        wwv_flow_flash_chart_series_fk|apex_040000
                                                                3     < WWV_FLOW_PAGE_DA_ACTIONS|APEX_040000                             wwv_flow_page_da_a_ar_fk|apex_040000
                                                                4       > WWV_FLOW_PAGE_DA_EVENTS|APEX_040000                            wwv_flow_page_da_a_evnt_fk|apex_040000
                                                                5         > WWV_FLOWS|APEX_040000*                                       wwv_flow_page_da_e_flow_fk|apex_040000
                                                                5         > WWV_FLOW_PAGE_PLUGS|APEX_040000*                             wwv_flow_page_da_e_tr_fk|apex_040000
                                                                5         > WWV_FLOW_STEPS|APEX_040000                                   wwv_flow_page_da_e_page_fk|apex_040000
                                                                6           > WWV_FLOWS|APEX_040000*                                     wwv_flow_steps_fk|apex_040000
                                                                6           < WWV_FLOW_PAGE_DA_ACTIONS|APEX_040000*                      wwv_flow_page_da_a_page_fk|apex_040000
                                                                6           < WWV_FLOW_PAGE_PLUGS|APEX_040000*                           wwv_flow_plug_to_page_fk|apex_040000
                                                                6           < WWV_FLOW_STEP_BRANCHES|APEX_040000                         wwv_flow_step_branches_fk2|apex_040000
                                                                7             > WWV_FLOWS|APEX_040000*                                   wwv_flow_step_branches_fk|apex_040000
                                                                7             < WWV_FLOW_STEP_BRANCH_ARGS|APEX_040000                    wwv_flow_step_branch_args_fk|apex_040000
                                                                6           < WWV_FLOW_STEP_BUTTONS|APEX_040000                          wwv_flow_step_buttons_fk2|apex_040000
                                                                7             > WWV_FLOWS|APEX_040000*                                   wwv_flow_step_buttons_fk1|apex_040000
                                                                7             > WWV_FLOW_PAGE_PLUGS|APEX_040000*                         wwv_flow_step_buttons_plug_fk|apex_040000
                                                                6           < WWV_FLOW_STEP_COMPUTATIONS|APEX_040000                     wwv_flow_step_comp_fk2|apex_040000
                                                                7             > WWV_FLOWS|APEX_040000*                                   wwv_flow_step_comp_fk|apex_040000
                                                                6           < WWV_FLOW_STEP_ITEMS|APEX_040000                            wwv_flow_step_items_fk2|apex_040000
                                                                7             > WWV_FLOWS|APEX_040000*                                   wwv_flow_step_items_fk|apex_040000
                                                                7             > WWV_FLOW_PAGE_PLUGS|APEX_040000*                         wwv_flow_step_items_plug_fk|apex_040000
                                                                7             < WWV_FLOW_STEP_ITEM_HELP|APEX_040000                      wwv_flow_item_helptext_fk|apex_040000
                                                                8               > WWV_FLOWS|APEX_040000*                                 wwv_flow_page_helptext_fk|apex_040000
                                                                6           < WWV_FLOW_STEP_PROCESSING|APEX_040000                       wwv_flow_step_proc_fk2|apex_040000
                                                                7             > WWV_FLOWS|APEX_040000*                                   wwv_flow_step_proc_fk|apex_040000
                                                                7             < WWV_FLOW_WS_PROCESS_PARMS_MAP|APEX_040000                wwv_flow_ws_map_fk2|apex_040000
                                                                8               > WWV_FLOW_WS_PARAMETERS|APEX_040000                     wwv_flows_ws_map_fk1|apex_040000
                                                                9                 > WWV_FLOW_WS_OPERATIONS|APEX_040000                   wwv_flow_ws_parms_fk|apex_040000
                                                               10                   > WWV_FLOW_SHARED_WEB_SERVICES|APEX_040000           wwv_flow_ws_opers_fk|apex_040000
                                                               11                     > WWV_FLOWS|APEX_040000*                           wwv_flow_ws_fk|apex_040000
                                                                6           < WWV_FLOW_STEP_VALIDATIONS|APEX_040000                      wwv_flow_step_val_fk2|apex_040000
                                                                7             > WWV_FLOWS|APEX_040000*                                   wwv_flow_step_val_fk|apex_040000
                                                                7             > WWV_FLOW_PAGE_PLUGS|APEX_040000*                         wwv_flow_step_val_to_reg_fk|apex_040000
                                                                3     < WWV_FLOW_PAGE_GENERIC_ATTR|APEX_040000                           wwv_flow_genattr_to_region_fk|apex_040000
                                                                3     = WWV_FLOW_PAGE_PLUGS|APEX_040000*                                 wwv_flow_plug_parent_fk|apex_040000
                                                                3     < WWV_FLOW_QUERY_DEFINITION|APEX_040000                            query_def_to_region_fk|apex_040000
                                                                4       < WWV_FLOW_QUERY_COLUMN|APEX_040000                              query_column_to_query_fk|apex_040000
                                                                5         > WWV_FLOW_QUERY_OBJECT|APEX_040000                            query_column_to_qry_object_fk|apex_040000
                                                                6           > WWV_FLOW_QUERY_DEFINITION|APEX_040000*                     query_object_to_query_fk|apex_040000
                                                                4       < WWV_FLOW_QUERY_CONDITION|APEX_040000                           query_condition_to_query_fk|apex_040000
                                                                3     < WWV_FLOW_REGION_CHART_SER_ATTR|APEX_040000                       wwv_flow_seattr_to_region_fk|apex_040000
                                                                3     < WWV_FLOW_REGION_REPORT_COLUMN|APEX_040000                        report_column_to_region_fk|apex_040000
                                                                3     < WWV_FLOW_REGION_REPORT_FILTER|APEX_040000                        sys_c004963|apex_040000
                                                                3     < WWV_FLOW_REGION_UPD_RPT_COLS|APEX_040000                         wwv_flow_urc_to_plug_fk|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_urc_to_flow_fk|apex_040000
                                                                3     < WWV_FLOW_TREE_REGIONS|APEX_040000                                wwv_flow_treeregion_fk2|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_treeregion_fk|apex_040000
                                                                3     < WWV_FLOW_WORKSHEETS|APEX_040000                                  wwv_flow_worksheets_reg_fk|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_worksheets_flow_fk|apex_040000
                                                                4       < WWV_FLOW_WORKSHEET_COLUMNS|APEX_040000                         wwv_flow_worksheet_columns_fk|apex_040000
                                                                5         > WWV_FLOWS|APEX_040000*                                       wwv_flow_worksheet_col_fk|apex_040000
                                                                5         > WWV_FLOW_WORKSHEET_COL_GROUPS|APEX_040000                    wwv_flow_worksheet_col_grps_fk|apex_040000
                                                                6           > WWV_FLOWS|APEX_040000*                                     wwv_flow_worksheet_col_grp_fk|apex_040000
                                                                6           > WWV_FLOW_WORKSHEETS|APEX_040000*                           wwv_flow_worksheet_col_grws_fk|apex_040000
                                                                6           > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000                      wwv_flow_worksheet_col_grp_fk2|apex_040000
                                                                7             > WWV_FLOW_WORKSHEETS|APEX_040000*                         wwwv_flow_ws_websheet_attr_fk|apex_040000
                                                                7             < WWV_FLOW_WORKSHEET_COLUMNS|APEX_040000*                  wwv_flow_worksheet_col_fk2|apex_040000
                                                                7             < WWV_FLOW_WORKSHEET_COMPUTATION|APEX_040000               wwv_flow_ws_computation_fk|apex_040000
                                                                8               > WWV_FLOW_WORKSHEET_RPTS|APEX_040000                    wwv_flow_ws_comp_cols_fk|apex_040000
                                                                9                 > WWV_FLOW_WORKSHEET_CATEGORIES|APEX_040000            wwv_flow_worksheet_rpts_fk|apex_040000
                                                                9                 < WWV_FLOW_WORKSHEET_CONDITIONS|APEX_040000            wwv_flow_worksheet_cond_fk|apex_040000
                                                               10                   > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000*             wwv_flow_ws_condition_fk|apex_040000
                                                                9                 < WWV_FLOW_WORKSHEET_GROUP_BY|APEX_040000              wwv_flow_ws_groupby_fk2|apex_040000
                                                               10                   > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000*             wwv_flow_ws_groupby_fk|apex_040000
                                                                9                 < WWV_FLOW_WORKSHEET_NOTIFY|APEX_040000                wwv_flow_worksheet_notify_fk2|apex_040000
                                                               10                   > WWV_FLOW_WORKSHEETS|APEX_040000*                   wwv_flow_worksheet_notify_fk|apex_040000
                                                               10                   > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000*             wwv_flow_worksheet_notify_fk4|apex_040000
                                                                9                 > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000*               wwv_flow_ws_rpt_fk|apex_040000
                                                                7             < WWV_FLOW_WORKSHEET_LOVS|APEX_040000                      wwv_flow_worksheet_lovs_fk2|apex_040000
                                                                8               > WWV_FLOW_WORKSHEETS|APEX_040000*                       wwv_flow_worksheet_lovs_fk|apex_040000
                                                                8               < WWV_FLOW_WORKSHEET_LOV_ENTRIES|APEX_040000             wwv_flow_worksheet_lov_ent_fk2|apex_040000
                                                                9                 > WWV_FLOW_WORKSHEETS|APEX_040000*                     wwv_flow_worksheet_lov_ent_fk|apex_040000
                                                                9                 > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000*               wwv_flow_worksheet_lov_ent_fk3|apex_040000
                                                                7             > WWV_FLOW_WS_APPLICATIONS|APEX_040000                     wwv_flow_ws_websheet_attr_fk2|apex_040000
                                                                8               < WWV_FLOW_WS_APP_SUG_OBJECTS|APEX_040000                wwv_flow_ws_app_so_fk1|apex_040000
                                                                8               < WWV_FLOW_WS_COL_VALIDATIONS|APEX_040000                wwv_flow_ws_col_val_fk3|apex_040000
                                                                9                 > WWV_FLOW_WORKSHEETS|APEX_040000*                     wwv_flow_ws_col_val_fk|apex_040000
                                                                9                 > WWV_FLOW_WS_WEBSHEET_ATTR|APEX_040000*               wwv_flow_ws_col_val_fk2|apex_040000
                                                                8               < WWV_FLOW_WS_CUSTOM_AUTH_SETUPS|APEX_040000             wwv_flow_ws_auth_setups_fk|apex_040000
                                                                8               < WWV_FLOW_WS_WEBPAGES|APEX_040000                       wwv_flow_ws_webpages_fk|apex_040000
                                                                9                 = WWV_FLOW_WS_WEBPAGES|APEX_040000*                    wwv_flow_ws_webpages_fk2|apex_040000
                                                                1 < WWV_FLOW_CAL_TEMPLATES|APEX_040000                                   wwv_flow_cal_templ_to_flow_fk|apex_040000
                                                                1 < WWV_FLOW_COMPOUND_CONDITIONS|APEX_040000                             wwv_flow_comp_cond_fk|apex_040000
                                                                1 < WWV_FLOW_COMPUTATIONS|APEX_040000                                    wwv_flow_computations_fk|apex_040000
                                                                1 < WWV_FLOW_CUSTOM_AUTH_SETUPS|APEX_040000                              wwv_flow_auth_setups_fk|apex_040000
                                                                1 < WWV_FLOW_ENTRY_POINTS|APEX_040000                                    wwv_flow_entry_points_fk|apex_040000
                                                                2   < WWV_FLOW_ENTRY_POINT_ARGS|APEX_040000                              wwv_flow_entry_point_args_fk|apex_040000
                                                                1 < WWV_FLOW_FIELD_TEMPLATES|APEX_040000                                 wwv_flow_field_temp_f_fk|apex_040000
                                                                1 < WWV_FLOW_ICON_BAR_ATTRIBUTES|APEX_040000                             wwv_flow_iconbarattr_fk|apex_040000
                                                                1 < WWV_FLOW_ICON_BAR|APEX_040000                                        wwv_flow_icon_bar_fk|apex_040000
                                                                1 < WWV_FLOW_INSTALL_BUILD_OPT|APEX_040000                               wwv_flow_install_build_opt_fk|apex_040000
                                                                2   > WWV_FLOW_INSTALL|APEX_040000                                       wwv_flow_install_build_opt_fk3|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_install_fk|apex_040000
                                                                3     < WWV_FLOW_INSTALL_CHECKS|APEX_040000                              wwv_flow_install_checks_fk3|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_install_checks_fk|apex_040000
                                                                3     < WWV_FLOW_INSTALL_SCRIPTS|APEX_040000                             wwv_flow_install_scripts_fk3|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_install_scripts_fk|apex_040000
                                                                2   > WWV_FLOW_PATCHES|APEX_040000                                       wwv_flow_install_build_opt_fk4|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_patches_fk|apex_040000
                                                                1 < WWV_FLOW_ITEMS|APEX_040000                                           wwv_flow_items_fk|apex_040000
                                                                1 < WWV_FLOW_LANGUAGE_MAP|APEX_040000                                    wwv_flow_lang_flow_id_fk|apex_040000
                                                                1 < WWV_FLOW_LISTS_OF_VALUES$|APEX_040000                                wwv_flow_lov_fk|apex_040000
                                                                2   < WWV_FLOW_LIST_OF_VALUES_DATA|APEX_040000                           wwv_flow_lov_data_fk|apex_040000
                                                                1 < WWV_FLOW_LISTS|APEX_040000                                           wwv_flow_lists_flow_fk|apex_040000
                                                                2   < WWV_FLOW_LIST_ITEMS|APEX_040000                                    wwv_flow_list_items_fk|apex_040000
                                                                3     = WWV_FLOW_LIST_ITEMS|APEX_040000*                                 parent_list_item_fk|apex_040000
                                                                1 < WWV_FLOW_LIST_TEMPLATES|APEX_040000                                  wwv_flow_list_template_fk|apex_040000
                                                                1 < WWV_FLOW_LOCK_PAGE|APEX_040000                                       sys_c004810|apex_040000
                                                                1 < WWV_FLOW_MENUS|APEX_040000                                           wwv_flow_menus_flow_fk|apex_040000
                                                                2   < WWV_FLOW_MENU_OPTIONS|APEX_040000                                  wwv_flow_opt_menus_fk|apex_040000
                                                                1 < WWV_FLOW_MENU_TEMPLATES|APEX_040000                                  wwv_flow_menus_t_flow_fk|apex_040000
                                                                1 < WWV_FLOW_MESSAGES$|APEX_040000                                       wwv_flow_messages_fk|apex_040000
                                                                1 < WWV_FLOW_PAGES_RESERVED|APEX_040000                                  wwv_flow_pages_reserved_fk|apex_040000
                                                                1 < WWV_FLOW_PAGE_CACHE|APEX_040000                                      wwv_flow_page_cache_fk|apex_040000
                                                                2   < WWV_FLOW_PAGE_CODE_CACHE|APEX_040000                               wwv_flow_page_code_cache_fk|apex_040000
                                                                1 < WWV_FLOW_PAGE_GROUPS|APEX_040000                                     sys_c004993|apex_040000
                                                                1 < WWV_FLOW_PAGE_PLUG_TEMPLATES|APEX_040000                             wwv_flow_plug_temp_fk|apex_040000
                                                                1 < WWV_FLOW_PAGE_SUBMISSIONS|APEX_040000                                wwv_flow_page_sub_fk|apex_040000
                                                                2   > WWV_FLOW_SESSIONS$|APEX_040000                                     wwv_flow_page_sub_sess_fk|apex_040000
                                                                3     < WWV_FLOW_COLLECTIONS$|APEX_040000                                wwv_flow_collection_fk|apex_040000
                                                                4       < WWV_FLOW_COLLECTION_MEMBERS$|APEX_040000                       wwv_flow_collection_membes_fk|apex_040000
                                                                3     < WWV_FLOW_DATA|APEX_040000                                        wwv_flow_data_fk|apex_040000
                                                                3     < WWV_FLOW_REQUEST_VERIFICATIONS|APEX_040000                       wwv_flow_request_verif_fk|apex_040000
                                                                3     < WWV_FLOW_SC_TRANS|APEX_040000                                    wwv_flow_sc_trans_fk2|apex_040000
                                                                3     < WWV_FLOW_TREE_STATE|APEX_040000                                  wwv_flow_tree_state$fk|apex_040000
                                                                1 < WWV_FLOW_PLUGINS|APEX_040000                                         wwv_flow_plugin_flow_fk|apex_040000
                                                                2   < WWV_FLOW_PLUGIN_ATTRIBUTES|APEX_040000                             wwv_flow_plugin_attr_parent_fk|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_plugin_attr_flow_fk|apex_040000
                                                                3     = WWV_FLOW_PLUGIN_ATTRIBUTES|APEX_040000*                          wwv_flow_plugin_attr_depend_fk|apex_040000
                                                                3     < WWV_FLOW_PLUGIN_ATTR_VALUES|APEX_040000                          wwv_flow_plugin_attrv_attr_fk|apex_040000
                                                                4       > WWV_FLOWS|APEX_040000*                                         wwv_flow_plugin_attrv_flow_fk|apex_040000
                                                                2   < WWV_FLOW_PLUGIN_EVENTS|APEX_040000                                 wwv_flow_plugin_evnt_parent_fk|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_plugin_evnt_flow_fk|apex_040000
                                                                2   < WWV_FLOW_PLUGIN_FILES|APEX_040000                                  wwv_flow_plugin_file_parent_fk|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_plugin_file_flow_fk|apex_040000
                                                                1 < WWV_FLOW_POPUP_LOV_TEMPLATE|APEX_040000                              wwv_flow_fk_poplov_temp|apex_040000
                                                                1 < WWV_FLOW_PROCESSING|APEX_040000                                      wwv_flow_processing_fk|apex_040000
                                                                1 < WWV_FLOW_REPORT_LAYOUTS|APEX_040000                                  wwv_flow_report_layoutse_fk|apex_040000
                                                                1 < WWV_FLOW_REQUIRED_ROLES|APEX_040000                                  wwv_flow_req_roles_fk|apex_040000
                                                                1 < WWV_FLOW_ROW_TEMPLATES|APEX_040000                                   wwv_flow_row_template_fk|apex_040000
                                                                1 < WWV_FLOW_SECURITY_SCHEMES|APEX_040000                                wwv_flow_sec_schemes_fk|apex_040000
                                                                1 < WWV_FLOW_SHARED_QRY_SQL_STMTS|APEX_040000                            wwv_flow_sqry_sql_flow_fk|apex_040000
                                                                2   > WWV_FLOW_SHARED_QUERIES|APEX_040000                                wwv_flow_sqry_sql_sqry_fk|apex_040000
                                                                3     > WWV_FLOWS|APEX_040000*                                           wwv_flow_shdqry_flow_fk|apex_040000
                                                                1 < WWV_FLOW_SHORTCUTS|APEX_040000                                       wwv_flow_shortcuts_to_flow_fk|apex_040000
                                                                1 < WWV_FLOW_TABS|APEX_040000                                            wwv_flow_tabs_fk|apex_040000
                                                                1 < WWV_FLOW_TEMPLATES|APEX_040000                                       wwv_flow_templates_fk|apex_040000
                                                                1 < WWV_FLOW_TEMPLATE_PREFERENCES|APEX_040000                            wwv_flow_templ_pref_fk|apex_040000
                                                                1 < WWV_FLOW_THEMES|APEX_040000                                          wwv_flow_themes_2f_fk|apex_040000
                                                                1 < WWV_FLOW_TOPLEVEL_TABS|APEX_040000                                   wwv_flow_toplev_tab_fk|apex_040000
                                                                1 < WWV_FLOW_TRANSLATABLE_TEXT$|APEX_040000                              wwv_flow_trans_text_fk|apex_040000
                                                                1 < WWV_FLOW_TREES|APEX_040000                                           wwv_flow_tree_fk|apex_040000
                                                                1 < WWV_FLOW_VALIDATIONS|APEX_040000                                     wwv_flow_val_fk|apex_040000
                                                                1 < WWV_MIG_GENERATED_APPLICATIONS|APEX_040000                           wwv_mig_gen_app_flow_id_fk|apex_040000
                                                                2   > WWV_MIG_PROJECTS|APEX_040000                                       wwv_mig_gen_app_proj_id_fk|apex_040000
                                                                3     < WWV_MIG_ACCESS|APEX_040000                                       wwv_mig_acc_fk|apex_040000
                                                                3     < WWV_MIG_FORMS|APEX_040000                                        wwv_mig_forms_project_id_fk|apex_040000
                                                                4       < WWV_MIG_FRM_MODULES|APEX_040000                                wwv_mig_frm_modules_file_id_fk|apex_040000
                                                                5         < WWV_MIG_FRM_FORMMODULES|APEX_040000                          wwv_mig_frm_frmmdl_mdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_ALERTS|APEX_040000                             wwv_mig_frm_alrt_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_ATTACHEDLIBRARY|APEX_040000                    wwv_mig_frm_atlib_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_BLOCKS|APEX_040000                             wwv_mig_frm_blk_frmmdl_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_BLK_DSA|APEX_040000                          wwv_mig_frm_blk_dsa_blk_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_BLK_DSC|APEX_040000                          wwv_mig_frm_blk_dsc_blk_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_BLK_ITEMS|APEX_040000                        wwv_mig_frm_bi_blk_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_BLK_ITEM_LIE|APEX_040000                   wwv_mig_frm_bi_lie_item_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_BLK_ITEM_RADIO|APEX_040000                 wwv_mig_frm_bir_item_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_BLK_ITEM_TRIGGERS|APEX_040000              wwv_mig_frm_bi_trg_item_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_REV_BLK_ITEMS|APEX_040000                  wwv_mig_frm_rev_bi_item_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_BLK_RELATIONS|APEX_040000                    wwv_mig_frm_blk_rel_blk_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_BLK_TRIGGERS|APEX_040000                     wwv_mig_frm_blk_trg_blk_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_REV_BLOCKS|APEX_040000                       wwv_mig_frm_rev_blocks_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_CANVAS|APEX_040000                             wwv_mig_frm_canvs_frmmdl_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_CNVS_GRAPHICS|APEX_040000                    wwv_mig_frm_cg_cnvs_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_CNVG_COMPOUNDTEXT|APEX_040000              wwv_mig_frm_cpdtxt_grphs_id_fk|apex_040000
                                                                9                 < WWV_MIG_FRM_CPDTXT_TEXTSEGMENT|APEX_040000           wwv_mig_frm_txtsgmt_cpd_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_CNVS_TABPAGE|APEX_040000                     wwv_mig_frm_ctp_cnvs_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_COORDINATES|APEX_040000                        wwv_mig_frm_crdnt_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_EDITOR|APEX_040000                             wwv_mig_frm_edtr_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_FMB_MENU|APEX_040000                           wwv_mig_frm_menu_frmmdl_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_FMB_MENU_MENUITEM|APEX_040000                wwv_mig_fmb_menuitem_menuid_fk|apex_040000
                                                                8               < WWV_MIG_FRM_FMB_MENUITEM_ROLE|APEX_040000              wwv_mig_fmb_mnuitemrl_mitm_fk|apex_040000
                                                                6           < WWV_MIG_FRM_LOV|APEX_040000                                wwv_mig_frm_lov_frmmdl_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_LOVCOLUMNMAPPING|APEX_040000                 wwv_mig_frm_lvcm_frmmdl_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_REV_LOVCOLMAPS|APEX_040000                 wwv_mig_frm_rev_lcm_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_REV_LOV|APEX_040000                          wwv_mig_frm_rev_lov_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_MODULEPARAMETER|APEX_040000                    wwv_mig_frm_mdlpr_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_OBJECTGROUP|APEX_040000                        wwv_mig_frm_objgp_frmmdl_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_OBJECTGROUPCHILD|APEX_040000                 wwv_mig_frm_objgpc_objgp_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_PROGRAMUNIT|APEX_040000                        wwv_mig_frm_pgut_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_PROPERTYCLASS|APEX_040000                      wwv_mig_frm_ppcl_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_RECORDGROUPS|APEX_040000                       wwv_mig_frm_recgp_frmmdl_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_RECORDGROUPCOLUMN|APEX_040000                wwv_mig_frm_rgc_recgp_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_REPORT|APEX_040000                             wwv_mig_frm_rpt_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_REV_FORMMODULES|APEX_040000                    wwv_mig_frm_rev_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_TRIGGERS|APEX_040000                           wwv_mig_frm_trg_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_VISUALATTRIBUTES|APEX_040000                   wwv_mig_frm_visat_frmmdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_WINDOWS|APEX_040000                            wwv_mig_frm_wndow_frmmdl_id_fk|apex_040000
                                                                3     < WWV_MIG_FRM_MENUS|APEX_040000                                    wwv_mig_menus_project_id_fk|apex_040000
                                                                4       < WWV_MIG_FRM_MENUS_MODULES|APEX_040000                          wwv_mig_mnu_modules_file_id_fk|apex_040000
                                                                5         < WWV_MIG_FRM_MENUS_MENUMODULES|APEX_040000                    wwv_mig_mnu_mnumdl_mdl_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_MENUSMODULEROLES|APEX_040000                   wwv_mig_mmodrole_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_MENUS_PROGRAMUNIT|APEX_040000                  wwv_mig_mnu_progunit_id_fk|apex_040000
                                                                6           < WWV_MIG_FRM_MENU|APEX_040000                               wwv_mig_mnu_id_fk|apex_040000
                                                                7             < WWV_MIG_FRM_MENU_MENUITEM|APEX_040000                    wwv_mig_mnuitem_id_fk|apex_040000
                                                                8               < WWV_MIG_FRM_MENUITEM_ROLE|APEX_040000                  wwv_mig_mnuitemrole_id_fk|apex_040000
                                                                3     < WWV_MIG_FRM_REV_APEX_APP|APEX_040000                             wwv_mig_frm_rev_apex_app_fk|apex_040000
                                                                3     < WWV_MIG_OLB|APEX_040000                                          wwv_mig_olb_project_id_fk|apex_040000
                                                                4       < WWV_MIG_OLB_MODULES|APEX_040000                                wwv_mig_olb_modules_file_id_fk|apex_040000
                                                                5         < WWV_MIG_OLB_OBJECTLIBRARY|APEX_040000                        wwv_mig_olb_objlib_mdl_id_fk|apex_040000
                                                                6           < WWV_MIG_OLB_BLOCK|APEX_040000                              wwv_mig_olb_block_objlib_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_BLK_DATASOURCECOL|APEX_040000                wwv_mig_olb_blk_dsc_blk_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_BLK_ITEM|APEX_040000                         wwv_mig_olb_blk_item_blk_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_BLK_ITEM_LIE|APEX_040000                   wwv_mig_olb_bil_item_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_BLK_ITEM_TRIGGER|APEX_040000               wwv_mig_olb_bit_item_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_BLK_TRIGGER|APEX_040000                      wwv_mig_olb_blk_trgr_blk_id_fk|apex_040000
                                                                6           < WWV_MIG_OLB_CANVAS|APEX_040000                             wwv_mig_olb_canvs_objlib_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_CNVS_GRAPHICS|APEX_040000                    wwv_mig_olb_cg_cnvs_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_CG_COMPOUNDTEXT|APEX_040000                wwv_mig_olb_cg_ct_grphs_id_fk|apex_040000
                                                                9                 < WWV_MIG_OLB_CG_CT_TEXTSEGMENT|APEX_040000            wwv_mig_olb_cg_ct_ts_ct_id_fk|apex_040000
                                                                6           < WWV_MIG_OLB_OBJECTLIBRARYTAB|APEX_040000                   wwv_mig_olb_olt_objlib_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_ALERT|APEX_040000                        wwv_mig_olb_olt_alrt_olt_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_BLOCK|APEX_040000                        wwv_mig_olb_t_block_olt_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_OLT_BLK_ITEM|APEX_040000                   wwv_mig_olb_olt_bi_blk_id_fk|apex_040000
                                                                9                 < WWV_MIG_OLB_OLT_BLK_ITEM_TRIGR|APEX_040000           wwv_mig_olb_olt_bit_item_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_CANVAS|APEX_040000                       wwv_mig_olb_t_canvas_olt_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_OLT_CNVS_GRAPHICS|APEX_040000              wwv_mig_olb_olt_cg_cnvs_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_GRAPHICS|APEX_040000                     wwv_mig_olb_t_grphcs_olt_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_ITEM|APEX_040000                         wwv_mig_olb_olt_item_olt_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_MENU|APEX_040000                         wwv_mig_olb_olt_menu_olt_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_OLT_MENU_MENUITEM|APEX_040000              wwv_mig_olb_olt_mmi_menu_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_OBJECTGROUP|APEX_040000                  wwv_mig_olb_t_objgrp_olt_id_fk|apex_040000
                                                                8               < WWV_MIG_OLB_OLT_OB_OBJGRPCHILD|APEX_040000             wwv_mig_olb_olt_ob_ogc_obid_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_REPORT|APEX_040000                       wwv_mig_olb_t_report_olt_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_TABPAGE|APEX_040000                      wwv_mig_olb_t_tabpage_oltid_fk|apex_040000
                                                                8               < WWV_MIG_OLB_OLT_TABPG_GRAPHICS|APEX_040000             wwv_mig_olb_olt_tpg_tp_id_fk|apex_040000
                                                                9                 < WWV_MIG_OLB_T_TP_G_GRAPHICS|APEX_040000              wwv_mig_olb_t_tp_gg_g_id_fk|apex_040000
                                                               10                   < WWV_MIG_OLB_T_TP_GG_CPDTXT|APEX_040000             wwv_mig_olb_t_tp_gg_ct_g_id_fk|apex_040000
                                                               11                     < WWV_MIG_OLB_T_TP_GG_CT_TXTSGT|APEX_040000        wwv_mig_olb_ttpggctts_ctid_fk|apex_040000
                                                               10                   < WWV_MIG_OLB_T_TP_GG_GRAPHICS|APEX_040000           wwv_mig_olb_t_tp_ggg_g_id_fk|apex_040000
                                                               11                     < WWV_MIG_OLB_T_TP_GGG_CPDTXT|APEX_040000          wwv_mig_olb_ttp_ggg_ct_gid_fk|apex_040000
                                                               12                       < WWV_MIG_OLB_T_TP_GGG_CT_TXTSGT|APEX_040000     wwv_mig_olb_ttpgggctts_ctid_fk|apex_040000
                                                               11                     < WWV_MIG_OLB_T_TP_GGG_GRAPHICS|APEX_040000        wwv_mig_olb_t_tp_gggg_g_id_fk|apex_040000
                                                               12                       < WWV_MIG_OLB_T_TP_GGGG_CPDTXT|APEX_040000       wwv_mig_olb_ttpggggct_g_id_fk|apex_040000
                                                               13                         < WWV_MIG_OLB_T_TP_GGGG_CT_TXSGT|APEX_040000   wwv_mig_olb_ttpggggcts_ctid_fk|apex_040000
                                                               12                       < WWV_MIG_OLB_T_TP_GGGG_GRAPHICS|APEX_040000     wwv_mig_olb_ttpggggg_g_id_fk|apex_040000
                                                               13                         < WWV_MIG_OLB_T_TP_GGGGG_CPDTXT|APEX_040000    wwv_mig_olb_ttpgggggct_g_id_fk|apex_040000
                                                               14                           < WWV_MIG_OLB_T_TP_GGGGG_CT_TXST|APEX_040000 wwv_mig_olb_ttp5gcts_ct_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_VISUALATTRBUTE|APEX_040000               wwv_mig_olb_olt_va_olt_id_fk|apex_040000
                                                                7             < WWV_MIG_OLB_OLT_WINDOW|APEX_040000                       wwv_mig_olb_olt_wndow_oltid_fk|apex_040000
                                                                6           < WWV_MIG_OLB_PROGRAMUNIT|APEX_040000                        wwv_mig_olb_pu_objlib_id_fk|apex_040000
                                                                6           < WWV_MIG_OLB_PROPERTYCLASS|APEX_040000                      wwv_mig_olb_pc_objlib_id_fk|apex_040000
                                                                6           < WWV_MIG_OLB_VISUALATTRIBUTE|APEX_040000                    wwv_mig_olb_va_objlib_id_fk|apex_040000
                                                                6           < WWV_MIG_OLB_WINDOW|APEX_040000                             wwv_mig_olb_wndow_objlib_id_fk|apex_040000
                                                                3     < WWV_MIG_PLSQL_LIBS|APEX_040000                                   wwv_mig_plls_project_id_fk|apex_040000
                                                                3     < WWV_MIG_PROJECT_COMPONENTS|APEX_040000                           wwv_mig_proj_comp_fk|apex_040000
                                                                3     < WWV_MIG_PROJECT_TRIGGERS|APEX_040000                             wwv_mig_proj_trig_fk|apex_040000
                                                                3     < WWV_MIG_RPTS|APEX_040000                                         wwv_mig_rpts_project_id_fk|apex_040000
                                                                4       < WWV_MIG_REPORT|APEX_040000                                     wwv_mig_rep_file_id_fk|apex_040000
                                                                5         < WWV_MIG_RPT_DATA|APEX_040000                                 wwv_mig_repdata_id_fk|apex_040000
                                                                6           < WWV_MIG_RPT_DATASRC|APEX_040000                            wwv_mig_repsrc_id_fk|apex_040000
                                                                7             < WWV_MIG_RPT_DATASRC_GRP|APEX_040000                      wwv_mig_grp_id_fk|apex_040000
                                                                8               < WWV_MIG_RPT_GRP_DATAITEM|APEX_040000                   wwv_mig_grp_dataitem_id_fk|apex_040000
                                                                9                 < WWV_MIG_RPT_GRP_DATAITEM_DESC|APEX_040000            wwv_mig_grp_itemdesc_id_fk|apex_040000
                                                                9                 < WWV_MIG_RPT_GRP_DATAITEM_PRIV|APEX_040000            wwv_mig_grp_itempriv_id_fk|apex_040000
                                                                8               < WWV_MIG_RPT_GRP_FIELD|APEX_040000                      wwv_mig_grp_fld_id_fk|apex_040000
                                                                8               < WWV_MIG_RPT_GRP_FILTER|APEX_040000                     wwv_mig_grp_fltr_id_fk|apex_040000
                                                                8               < WWV_MIG_RPT_GRP_FORMULA|APEX_040000                    wwv_mig_grp_form_id_fk|apex_040000
                                                                8               < WWV_MIG_RPT_GRP_ROWDELIM|APEX_040000                   wwv_mig_grp_row_id_fk|apex_040000
                                                                8               < WWV_MIG_RPT_GRP_SUMMARY|APEX_040000                    wwv_mig_grp_sum_id_fk|apex_040000
                                                                7             < WWV_MIG_RPT_DATASRC_SELECT|APEX_040000                   wwv_mig_select_id_fk|apex_040000
                                                                6           < WWV_MIG_RPT_DATA_SUMMARY|APEX_040000                       wwv_mig_repsum_id_fk|apex_040000
                                                                5         < WWV_MIG_RPT_REPORTPRIVATE|APEX_040000                        wwv_mig_rptpriv_id_fk|apex_040000
WWV_FLOW_ADVISOR_CATEGORIES|APEX_040000          2       3      0 WWV_FLOW_ADVISOR_CATEGORIES|APEX_040000                                ROOT
                                                                1 < WWV_FLOW_ADVISOR_CHECKS|APEX_040000                                  wwv_flow_adv_chk_cat_fk|apex_040000
                                                                2   < WWV_FLOW_ADVISOR_CHECK_MSGS|APEX_040000                            wwv_flow_adv_chk_msg_check_fk|apex_040000
WWV_FLOW_BUGS|APEX_040000                       13      11      0 WWV_FLOW_BUGS|APEX_040000                                              ROOT
                                                                1 < WWV_FLOW_TEAMDEV_TAG_CLOUD|APEX_040000                               wwv_flow_teamdev_tc_b|apex_040000
                                                                2   > WWV_FLOW_FEATURES|APEX_040000                                      wwv_flow_teamdev_tc_f|apex_040000
                                                                3     = WWV_FLOW_FEATURES|APEX_040000*                                   wwv_flow_features_par_feat_fk|apex_040000
                                                                3     < WWV_FLOW_FEATURE_HISTORY|APEX_040000                             wwv_flow_feature_hist_fk|apex_040000
                                                                3     < WWV_FLOW_FEATURE_PROGRESS|APEX_040000                            wwv_flow_feature_prog_fk|apex_040000
                                                                3     < WWV_FLOW_TEAM_FILES|APEX_040000                                  wwv_flow_team_files_fk1|apex_040000
                                                                4       > WWV_FLOW_EVENTS|APEX_040000                                    wwv_flow_team_files_fk3|apex_040000
                                                                4       > WWV_FLOW_FEEDBACK|APEX_040000                                  wwv_flow_team_files_fk4|apex_040000
                                                                5         < WWV_FLOW_FEEDBACK_FOLLOWUP|APEX_040000                       wwv_flow_feedback_fup_fk|apex_040000
                                                                5         < WWV_FLOW_TEAM_FILES|APEX_040000*                             wwv_flow_team_files_fk5|apex_040000
                                                                4       > WWV_FLOW_TASKS|APEX_040000                                     wwv_flow_team_files_fk2|apex_040000
                                                                5         < WWV_FLOW_TASK_PROGRESS|APEX_040000                           wwv_flow_task_prog_fk|apex_040000
                                                                5         < WWV_FLOW_TEAMDEV_TAG_CLOUD|APEX_040000*                      wwv_flow_teamdev_tc_t|apex_040000
WWV_FLOW_DATA_LOAD_BAD_LOG|APEX_040000           1       2      0 WWV_FLOW_DATA_LOAD_BAD_LOG|APEX_040000                                 ROOT
                                                                1 > WWV_FLOW_DATA_LOAD_UNLOAD|APEX_040000                                wwv_flow_data_load_bad_log_fk1|apex_040000
WWV_FLOW_DICTIONARY_VIEWS|APEX_040000            1       1      0 WWV_FLOW_DICTIONARY_VIEWS|APEX_040000                                  ROOT
                                                                1 = WWV_FLOW_DICTIONARY_VIEWS|APEX_040000*                               wwv_flow_dict_view_parent_fk|apex_040000
WWV_FLOW_FILE_OBJECTS$|FLOWS_FILES               5       6      0 WWV_FLOW_FILE_OBJECTS$|FLOWS_FILES                                     ROOT
                                                                1 < WWV_FLOW_IMPORT_EXPORT|APEX_040000                                   wwv_flow_import_export_fk|apex_040000
                                                                1 < WWV_FLOW_SW_BINDS|APEX_040000                                        wwv_flow_sw_bind_fk|apex_040000
                                                                1 < WWV_FLOW_SW_RESULTS|APEX_040000                                      wwv_flow_sw_result_fk|apex_040000
                                                                2   < WWV_FLOW_SW_DETAIL_RESULTS|APEX_040000                             wwv_flow_sw_d_result_fk|apex_040000
                                                                1 < WWV_FLOW_SW_STMTS|APEX_040000                                        wwv_flow_sw_stmts_fk|apex_040000
WWV_FLOW_FLASH_MAP_FILES|APEX_040000             2       3      0 WWV_FLOW_FLASH_MAP_FILES|APEX_040000                                   ROOT
                                                                1 > WWV_FLOW_FLASH_MAP_FOLDERS|APEX_040000                               wwv_flow_flash_map_files_fk|apex_040000
                                                                1 < WWV_FLOW_FLASH_MAP_REGIONS|APEX_040000                               wwv_flow_flash_map_reg_fk|apex_040000
WWV_FLOW_FND_GROUP_USERS|APEX_040000             1       2      0 WWV_FLOW_FND_GROUP_USERS|APEX_040000                                   ROOT
                                                                1 > WWV_FLOW_FND_USER_GROUPS|APEX_040000                                 wwv_flow_fnd_gu_int_g_fk|apex_040000
WWV_FLOW_HNT_ARGUMENT_INFO|APEX_040000           1       2      0 WWV_FLOW_HNT_ARGUMENT_INFO|APEX_040000                                 ROOT
                                                                1 > WWV_FLOW_HNT_PROCEDURE_INFO|APEX_040000                              wwv_flow_hnt_arg_info_proc_fk|apex_040000
WWV_FLOW_HNT_COLUMN_DICT|APEX_040000             1       2      0 WWV_FLOW_HNT_COLUMN_DICT|APEX_040000                                   ROOT
                                                                1 < WWV_FLOW_HNT_COL_DICT_SYN|APEX_040000                                wwv_flow_hnt_col_dict_syn_fk|apex_040000
WWV_FLOW_HNT_COLUMN_INFO|APEX_040000             4       4      0 WWV_FLOW_HNT_COLUMN_INFO|APEX_040000                                   ROOT
                                                                1 > WWV_FLOW_HNT_GROUPS|APEX_040000                                      wwv_flow_hnt_col_info_grp_fk|apex_040000
                                                                2   > WWV_FLOW_HNT_TABLE_INFO|APEX_040000                                wwv_flow_hnt_groups_tab_fk|apex_040000
                                                                3     < WWV_FLOW_HNT_COLUMN_INFO|APEX_040000*                            wwv_flow_hnt_col_info_tab_fk|apex_040000
                                                                1 < WWV_FLOW_HNT_LOV_DATA|APEX_040000                                    wwv_flow_hnt_lov_data_col_fk|apex_040000
WWV_FLOW_MAIL_ATTACHMENTS|APEX_040000            1       2      0 WWV_FLOW_MAIL_ATTACHMENTS|APEX_040000                                  ROOT
                                                                1 > WWV_FLOW_MAIL_QUEUE|APEX_040000                                      wwv_flow_mail_attachments_fk1|apex_040000
WWV_FLOW_MODELS|APEX_040000                      4       4      0 WWV_FLOW_MODELS|APEX_040000                                            ROOT
                                                                1 < WWV_FLOW_MODEL_PAGES|APEX_040000                                     wwv_flow_model_pages_fk|apex_040000
                                                                2   = WWV_FLOW_MODEL_PAGES|APEX_040000*                                  wwv_flow_model_pages_fk2|apex_040000
                                                                2   < WWV_FLOW_MODEL_PAGE_REGIONS|APEX_040000                            wwv_flow_mpr_fk|apex_040000
                                                                3     < WWV_FLOW_MODEL_PAGE_COLS|APEX_040000                             wwv_flow_model_page_cols_fk|apex_040000
WWV_FLOW_QB_SAVED_COND|APEX_040000               3       4      0 WWV_FLOW_QB_SAVED_COND|APEX_040000                                     ROOT
                                                                1 > WWV_FLOW_QB_SAVED_QUERY|APEX_040000                                  sys_c005031|apex_040000
                                                                2   < WWV_FLOW_QB_SAVED_JOIN|APEX_040000                                 sys_c005038|apex_040000
                                                                2   < WWV_FLOW_QB_SAVED_TABS|APEX_040000                                 sys_c005045|apex_040000
WWV_FLOW_RESTRICTED_SCHEMAS|APEX_040000          1       2      0 WWV_FLOW_RESTRICTED_SCHEMAS|APEX_040000                                ROOT
                                                                1 < WWV_FLOW_RSCHEMA_EXCEPTIONS|APEX_040000                              wwv_flow_rschema_exceptions_fk|apex_040000
WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040000       1       1      0 WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040000                             ROOT
                                                                1 = WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040000*                          wwv_mig_olb_xmltagtablemap_fk|apex_040000
WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040000           1       1      0 WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040000                                 ROOT
                                                                1 = WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040000*                              wwv_mig_frm_xmltagtablemap_fk|apex_040000
WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040000          1       1      0 WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040000                                ROOT
                                                                1 = WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040000*                             wwv_mig_mnu_xmltagtablemap_fk|apex_040000
WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040000           1       1      0 WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040000                                 ROOT
                                                                1 = WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040000*                              wwv_mig_rpt_xmltagtablemap_fk|apex_040000
WWV_PURGE_DATAFILES|APEX_040000                  4       5      0 WWV_PURGE_DATAFILES|APEX_040000                                        ROOT
                                                                1 > WWV_PURGE_WORKSPACES|APEX_040000                                     wwv_purge_datafiles_fk1|apex_040000
                                                                2   < WWV_PURGE_EMAILS|APEX_040000                                       wwv_purge_emails_fk1|apex_040000
                                                                3     < WWV_PURGE_WORKSPACE_RESPONSES|APEX_040000                        wwv_purge_workspace_resp_fk1|apex_040000
                                                                2   < WWV_PURGE_SCHEMAS|APEX_040000                                      wwv_purge_schemas_fk1|apex_040000

359 rows selected.

Elapsed: 00:00:01.81
</pre>
</div>

#### Network Summaries
```
Network summary 1 - by network

Network                                     #Links  #Nodes    Max Lev
------------------------------------------ ------- ------- ----------
WWV_FLOW_HNT_COLUMN_DICT|APEX_040000             2       2          1
WWV_MIG_RPT_XMLTAGTABLEMAP|APEX_040000           2       1          1
WWV_MIG_MENU_XMLTAGTABLEMAP|APEX_040000          2       1          1
WWV_MIG_FRM_XMLTAGTABLEMAP|APEX_040000           2       1          1
WWV_MIG_FRM_OLB_XMLTAGTABLEMAP|APEX_040000       2       1          1
WWV_FLOW_RESTRICTED_SCHEMAS|APEX_040000          2       2          1
WWV_FLOW_MAIL_ATTACHMENTS|APEX_040000            2       2          1
WWV_FLOW_HNT_ARGUMENT_INFO|APEX_040000           2       2          1
WWV_FLOW_FND_GROUP_USERS|APEX_040000             2       2          1
WWV_FLOW_DICTIONARY_VIEWS|APEX_040000            2       1          1
WWV_FLOW_DATA_LOAD_BAD_LOG|APEX_040000           2       2          1
WWV_FLOW_FLASH_MAP_FILES|APEX_040000             3       3          1
WWV_FLOW_ADVISOR_CATEGORIES|APEX_040000          3       3          2
WWV_FLOW_QB_SAVED_COND|APEX_040000               4       4          2
WWV_PURGE_DATAFILES|APEX_040000                  5       5          3
APEX$_WS_FILES|APEX_040000                       5       5          2
WWV_FLOW_MODELS|APEX_040000                      5       4          3
WWV_FLOW_HNT_COLUMN_INFO|APEX_040000             5       4          3
WWV_FLOW_FILE_OBJECTS$|FLOWS_FILES               6       6          2
WWV_FLOW_BUGS|APEX_040000                       14      11          5
WWV_FLOWS|APEX_040000                          287     236         14

21 rows selected.

Elapsed: 00:00:00.01
Network summary 2 - grouped by numbers of nodes

 #Nodes  #Networks
------- ----------
      1          5
      2          6
      3          2
      4          3
      5          2
      6          1
     11          1
    236          1

8 rows selected.

Elapsed: 00:00:00.01
```

### Schema Metadata Analysis Report

This, L\_APEX\_040000.txt, is too long to embed so is included in [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/sql-developer-importing-unit-test-repository-via-data-modeler).
