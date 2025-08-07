---
layout: post
title: "A Utility for Reading REF Cursors into a List of Delimited Strings"
date: 2016-08-09
migrated: true
group: general-sql
categories: 
  - "oracle"
  - "plsql"
  - "testing"
  - "utplsql"
tags: 
  - "dbms_sql"
  - "oracle"
  - "ref-cursor"
  - "unit-test"
  - "utplsql"
  - "web-services"
---

**Note, 2025:** This function is now included in [Oracle PL/SQL General Utilities Module](https://github.com/BrenPatF/oracle_plsql_utils).

This is a generic utility function that I wrote for use in unit testing web services that return REF cursors. It takes as input an open REF cursor and an optional filter string and returns the output as an array of delimited strings matching the filter. This can then be used to assert against expected output records.

The initial technical difficulty I had with making this generic was in getting the structure of the cursor. This is resolved by translating the REF cursor into a cursor that can be managed by the package DBMS\_SQL, using DBMS\_SQL.To\_Cursor\_Number.

As I am using it for unit testing the function does not need to be scalable. For other uses involving large volumes it could be made scalable by, for example, converting to a pipelined function.

### Data Types

The cursor needs to return a flat projection, and columns accounted for currently are:

- NUMBER
- VARCHAR2
- DATE
- TIMESTAMP

### Custom Dependencies

- L1\_chr\_arr - custom array type, equivalent to SYS.ODCIVarchar2List
- Utils.List\_Delim - function that turns an array of strings into one delimited string

## Code

The function wiil (shortly) be packaged within my unit testing package UT\_Utils, but for demo purposes, here is a script that has a driving anonymous block with the function declared locally within that block.

```sql
DECLARE
  l_csr         SYS_REFCURSOR;
  l_res_lis     L1_chr_arr;
  c_query_1     CONSTANT VARCHAR2(4000) := 'SELECT * FROM employees ORDER BY employee_id';
   PROCEDURE Write_Log (p_line VARCHAR2) IS
  BEGIN
    DBMS_Output.Put_Line (p_line);
  END Write_Log;

FUNCTION Cursor_to_Array (p_csr IN OUT SYS_REFCURSOR, p_filter VARCHAR2 DEFAULT NULL) RETURN L1_chr_arr IS
  c_chr_type    CONSTANT PLS_INTEGER := 1; --DBMS_Types.TYPECODE_* do not seem to quite work
  c_num_type    CONSTANT PLS_INTEGER := 2;
  c_dat_type    CONSTANT PLS_INTEGER := 12;
  c_stp_type    CONSTANT PLS_INTEGER := 180;
  l_csr_id      PLS_INTEGER;
  l_n_cols      PLS_INTEGER;
  l_desctab     DBMS_SQL.DESC_TAB;
  l_chr_val     VARCHAR2(4000);
  l_num_val     NUMBER;
  l_dat_val     DATE;
  l_stp_val     TIMESTAMP;
  l_val_lis     L1_chr_arr;
  l_res_lis     L1_chr_arr := L1_chr_arr();
  l_rec         VARCHAR2(4000);
BEGIN
  l_csr_id := DBMS_SQL.To_Cursor_Number (p_csr);
  DBMS_SQL.Describe_Columns (l_csr_id, l_n_cols, l_desctab);
  FOR i IN 1..l_n_cols LOOP
    CASE l_desctab(i).col_type
      WHEN c_chr_type THEN
        DBMS_SQL.Define_Column (l_csr_id, i, l_chr_val, 4000);
      WHEN c_num_type THEN
        DBMS_SQL.Define_Column (l_csr_id, i, l_num_val);
      WHEN c_dat_type THEN
        DBMS_SQL.Define_Column (l_csr_id, i, l_dat_val);
      WHEN c_stp_type THEN
         DBMS_SQL.Define_Column (l_csr_id, i, l_stp_val);
     ELSE
        Write_Log ('Col type ' || l_desctab(i).col_type || ' not accounted for!');
    END CASE;
  END LOOP;
  WHILE DBMS_SQL.Fetch_Rows (l_csr_id) > 0 LOOP
    l_val_lis := L1_chr_arr();
    l_val_lis.EXTEND (l_n_cols);
    FOR i IN 1 .. l_n_cols LOOP
      CASE l_desctab(i).col_type
        WHEN c_chr_type THEN
          DBMS_SQL.Column_Value (l_csr_id, i, l_chr_val);
          l_val_lis(i) := l_chr_val;
        WHEN c_num_type THEN
          DBMS_SQL.Column_Value (l_csr_id, i, l_num_val);
          l_val_lis(i) := l_num_val;
        WHEN c_dat_type THEN
          DBMS_SQL.Column_Value (l_csr_id, i, l_dat_val);
          l_val_lis(i) := l_dat_val;
        WHEN c_stp_type THEN
          DBMS_SQL.Column_Value (l_csr_id, i, l_stp_val);
          l_val_lis(i) := l_stp_val;
      END CASE;
    END LOOP;
    l_rec := Utils.List_Delim (l_val_lis);
    IF l_rec LIKE '%' || p_filter || '%' THEN
      l_res_lis.EXTEND;
      l_res_lis (l_res_lis.COUNT) := l_rec;
    END IF;
  END LOOP;
  DBMS_SQL.Close_Cursor (l_csr_id);
  RETURN l_res_lis;
END Cursor_to_Array;
BEGIN
  OPEN l_csr FOR c_query_1;
  l_res_lis := Cursor_to_Array (l_csr, 'SA_REP');
  FOR i IN 1..l_res_lis.COUNT LOOP
    Write_Log (i || ': ' || l_res_lis(i));
  END LOOP;
END;
/
```

## Output for Cursor against HR.employees

```
1: 150|Peter|Tucker|PTUCKER|011.44.1344.129268|30-JAN-05|SA_REP|10000|.3|145|80|
2: 151|David|Bernstein|DBERNSTE|011.44.1344.345268|24-MAR-05|SA_REP|9500|.25|145|80|
3: 152|Peter|Hall|PHALL|011.44.1344.478968|20-AUG-05|SA_REP|9000|.25|145|80|
4: 153|Christopher|Olsen|COLSEN|011.44.1344.498718|30-MAR-06|SA_REP|8000|.2|145|80|
5: 154|Nanette|Cambrault|NCAMBRAU|011.44.1344.987668|09-DEC-06|SA_REP|7500|.2|145|80|
6: 155|Oliver|Tuvault|OTUVAULT|011.44.1344.486508|23-NOV-07|SA_REP|7000|.15|145|80|
7: 156|Janette|King|JKING|011.44.1345.429268|30-JAN-04|SA_REP|10000|.35|146|80|
8: 157|Patrick|Sully|PSULLY|011.44.1345.929268|04-MAR-04|SA_REP|9500|.35|146|80|
9: 158|Allan|McEwen|AMCEWEN|011.44.1345.829268|01-AUG-04|SA_REP|9000|.35|146|80|
10: 159|Lindsey|Smith|LSMITH|011.44.1345.729268|10-MAR-05|SA_REP|8000|.3|146|80|
11: 160|Louise|Doran|LDORAN|011.44.1345.629268|15-DEC-05|SA_REP|7500|.3|146|80|
12: 161|Sarath|Sewall|SSEWALL|011.44.1345.529268|03-NOV-06|SA_REP|7000|.25|146|80|
13: 162|Clara|Vishney|CVISHNEY|011.44.1346.129268|11-NOV-05|SA_REP|10500|.25|147|80|
14: 163|Danielle|Greene|DGREENE|011.44.1346.229268|19-MAR-07|SA_REP|9500|.15|147|80|
15: 164|Mattea|Marvins|MMARVINS|011.44.1346.329268|24-JAN-08|SA_REP|7200|.1|147|80|
16: 165|David|Lee|DLEE|011.44.1346.529268|23-FEB-08|SA_REP|6800|.1|147|80|
17: 166|Sundar|Ande|SANDE|011.44.1346.629268|24-MAR-08|SA_REP|6400|.1|147|80|
18: 167|Amit|Banda|ABANDA|011.44.1346.729268|21-APR-08|SA_REP|6200|.1|147|80|
19: 168|Lisa|Ozer|LOZER|011.44.1343.929268|11-MAR-05|SA_REP|11500|.25|148|80|
20: 169|Harrison|Bloom|HBLOOM|011.44.1343.829268|23-MAR-06|SA_REP|10000|.2|148|80|
21: 170|Tayler|Fox|TFOX|011.44.1343.729268|24-JAN-06|SA_REP|9600|.2|148|80|
22: 171|William|Smith|WSMITH|011.44.1343.629268|23-FEB-07|SA_REP|7400|.15|148|80|
23: 172|Elizabeth|Bates|EBATES|011.44.1343.529268|24-MAR-07|SA_REP|7300|.15|148|80|
24: 173|Sundita|Kumar|SKUMAR|011.44.1343.329268|21-APR-08|SA_REP|6100|.1|148|80|
25: 174|Ellen|Abel|EABEL|011.44.1644.429267|11-MAY-04|SA_REP|11000|.3|149|80|
26: 175|Alyssa|Hutton|AHUTTON|011.44.1644.429266|19-MAR-05|SA_REP|8800|.25|149|80|
27: 176|Jonathon|Taylor|JTAYLOR|011.44.1644.429265|24-MAR-06|SA_REP|8600|.2|149|80|
28: 177|Jack|Livingston|JLIVINGS|011.44.1644.429264|23-APR-06|SA_REP|8400|.2|149|80|
29: 178|Kimberely|Grant|KGRANT|011.44.1644.429263|24-MAY-07|SA_REP|7000|.15|149||
30: 179|Charles|Johnson|CJOHNSON|011.44.1644.429262|04-JAN-08|SA_REP|6200|.1|149|80|
 
PL/SQL procedure successfully completed.
```
