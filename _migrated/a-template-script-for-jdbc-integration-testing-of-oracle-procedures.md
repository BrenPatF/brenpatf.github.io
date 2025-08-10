---
layout: post
migrated: true
title: "A Template Script for JDBC Integration Testing of Oracle Procedures"
date: 2016-05-14
group: func-testing
categories: 
  - "java"
  - "object"
  - "oracle"
  - "plsql"
  - "testing"
  - "v12"
  - "web-services"
tags: 
  - "java"
  - "jdbc"
  - "object"
  - "oracle"
  - "plsql"
  - "web-service"
---
Some time ago I wrote an Oracle database package for a web service. The Java developer for the service told me that it was throwing an error when called from Java, although I had unit tested it from PL/SQL. He gave me a small Java driver script to demonstrate the issue, and this allowed the issue to be quickly identified: As both Java and PL/SQL have boolean data types I had considered that a boolean parameter would make sense to pass a boolean value. However, it turns out that this does not work in JDBC, and so I replaced it with an integer parameter.

It occurred to me then that it would be nice if the database developer was able in general to test JDBC compatibility of his or her procedure as a final step after unit testing. To this end I created a more generic example script based on a simple procedure that I wrote against Oracle's HR demo schema, the same procedure that I used as an example of a unit testing design pattern, [DBAPIT 1: Web Service Saving](/dbapit/dbapit-1/).

**Update, 4 November 2017:** I have made a self-contained project on GitHub with both Java and Oracle code, avoiding dependency on my testing project. [JDBC Calling of Oracle Procedures with Object Array Parameters on GitHub](https://github.com/BrenPatF/oracle_jdbc_demo). I have also added the Oracle code below.

The code below runs against any Oracle instance in which the standard Oracle HR demo schema has been installed. There is a video demonstration of how to install and run it at the end of this article. The procedure has one input and one output object array parameter, and can easily be extended as desired. It requires one jar file in the classpath, ojdbc6.jar, which is available in Oracle client or database installations, and can be run from an IDE such as Eclipse.

## Java Code

<div class="scrollbox">
<pre>
package jdbcdemo;
/***************************************************************************************************
Name:        Driver.java
Description: This is a Java driver script for Brendan's HR demo web service procedure. It is
             designed to serve as a template for other web service procedures to allow a database
             developer to do a JDBC integration test easily, and can also be used as a starting point
             for Java development.

             The template procedure takes an input array of objects and has an output array of 
             objects. It is easy to update for any named object and array types, procedure and
             Oracle connection. Any other signature types would need additional changes.

	     See 'A Template Script for JDBC Calling of Oracle Procedures with Object Array Parameters'
             https://brenpatf.github.io/migrated/a-template-script-for-jdbc-integration-testing-of-oracle-procedures/
                                                                               
Modification History
Who                  When        Which What
-------------------- ----------- ----- -------------------------------------------------------------
Brendan Furey        14-May-2016 1.0   Created                       
Brendan Furey        04-Nov-2017 1.1   Put into new GitHub project along with Oracle code

***************************************************************************************************/
import java.sql.DriverManager;
import java.sql.SQLException;
import java.sql.Array;
import java.sql.Struct;

import oracle.jdbc.OracleTypes;
import oracle.jdbc.OracleCallableStatement;
import oracle.jdbc.OracleConnection;

public class Driver {

// Change section 1/2: Replace these constants with your own values

  private static final String DB_CONNECTION = "jdbc:oracle:thin:hr/hr@localhost:1521/orclpdb";
  private static final String TY_IN_OBJ     = "TY_EMP_IN_OBJ";
  private static final String TY_IN_ARR     = "TY_EMP_IN_ARR";
  private static final String TY_OUT_ARR    = "TY_EMP_OUT_ARR";
  private static final String PROC_NAME     = "Emp_WS.AIP_Save_Emps";

  private static OracleConnection conn;

  public static void main(String[] argv) {
    try {
      getDBConnection ();
      prOutArray (callProc (inArray ()));
    }
    catch (SQLException e) {
      System.out.println(e.getMessage());
    }
  }
  private static Array inArray () throws SQLException {

// Change section 2/2: Replace [2] with number of test records, and the arrays with their values

    Struct[] struct = new Struct[2];
    struct[0] = conn.createStruct (TY_IN_OBJ, new Object[] {"LN 1", "EM 1", "IT_PROG", 1000});
    struct[1] = conn.createStruct (TY_IN_OBJ, new Object[] {"LN 2", "EM 2", "IT_PROG", 2000});
    return conn.createARRAY (TY_IN_ARR, struct);
  }
  private static Array callProc (Array objArray) throws SQLException {
    OracleCallableStatement ocs = (OracleCallableStatement) conn.prepareCall ("BEGIN "+PROC_NAME+"(:1, :2); END;");
    ocs.setArray (1, objArray);
    ocs.registerOutParameter (2, OracleTypes.ARRAY, TY_OUT_ARR);
    ocs.execute ();
    return ocs.getARRAY (2);
  }
  private static void prOutArray (Array arr) throws SQLException {
    Object[] objArr = (Object[]) arr.getArray();
    int j = 0;
    for (Object rec : objArr) {
      Object[] objLis = ((Struct)rec).getAttributes ();
      int i = 0;
      String recStr = "";
      for (Object fld : objLis) {
        if (i++ > 0) recStr = recStr + '/';
        recStr = recStr + fld.toString();
      }
      System.out.println ("Record "+(++j)+": "+recStr);
    }
  }
  private static void getDBConnection () throws SQLException {
    conn = (OracleConnection) DriverManager.getConnection (DB_CONNECTION);
    conn.setAutoCommit (false);
    System.out.println ("Connected...");
  }
}
</pre>
</div>

## Example output

```
Connected...
Record 1: 239/TWO HUNDRED THIRTY-NINE
Record 2: 240/TWO HUNDRED FORTY

```

[Google Java Style](https://google.github.io/styleguide/javaguide.html)

## Oracle Code

<div class="scrollbox">
<pre>
/***************************************************************************************************

Author:      Brendan Furey
Description: Script to create objects to demo JDBC procedure calls with object array parameters

         See 'A Template Script for JDBC Calling of Oracle Procedures with Object Array Parameters'
             https://brenpatf.github.io/migrated/a-template-script-for-jdbc-integration-testing-of-oracle-procedures/

Modification History
Who                  When        Which What
-------------------- ----------- ----- -------------------------------------------------------------
Brendan Furey        04-May-2016 1.0   Created
Brendan Furey        04-Nov-2017 1.1   Extracted the JDBC demo code from the unit testing project, 
                                       and put into new GitHub project along with Java code

***************************************************************************************************/

REM Run this script from Oracle's standard HR schema to create objects to demo JDBC procedure calls

COLUMN "Database"	FORMAT A20
COLUMN "Time"		FORMAT A20
COLUMN "Version"	FORMAT A30
COLUMN "Session"	FORMAT 9999990
COLUMN "OS User"	FORMAT A10
COLUMN "Machine"	FORMAT A20
SELECT 'Start: ' || dbs.name "Database", To_Char (SYSDATE,'DD-MON-YYYY HH24:MI:SS') "Time", 
	Replace (Substr(ver.banner, 1, Instr(ver.banner, '64')-4), 'Enterprise Edition Release ', '') "Version"
  FROM v$database dbs,  v$version ver
 WHERE ver.banner LIKE 'Oracle%';

PROMPT Input types creation
DROP TYPE ty_emp_in_arr
/
CREATE OR REPLACE TYPE ty_emp_in_obj AS OBJECT (
        last_name       VARCHAR2(25),
        email           VARCHAR2(25),
        job_id          VARCHAR2(10),
        salary          NUMBER
)
/
CREATE TYPE ty_emp_in_arr AS TABLE OF ty_emp_in_obj
/
PROMPT Output types creation
DROP TYPE ty_emp_out_arr
/
CREATE OR REPLACE TYPE ty_emp_out_obj AS OBJECT (
        employee_id     NUMBER,
        description     VARCHAR2(500)
)
/
CREATE TYPE ty_emp_out_arr AS TABLE OF ty_emp_out_obj
/
CREATE OR REPLACE PACKAGE Emp_WS AS
/***************************************************************************************************
Description: HR demo web service code. Procedure saves new employees list and returns primary key
             plus same in words, or zero plus error message in output list

***************************************************************************************************/

PROCEDURE AIP_Save_Emps (p_ty_emp_in_lis ty_emp_in_arr, x_ty_emp_out_lis OUT ty_emp_out_arr);

END Emp_WS;
/
CREATE OR REPLACE PACKAGE BODY Emp_WS AS

PROCEDURE AIP_Save_Emps (p_ty_emp_in_lis           ty_emp_in_arr,     -- list of employees to insert
                         x_ty_emp_out_lis      OUT ty_emp_out_arr) IS -- list of employee results

  l_ty_emp_out_lis        ty_emp_out_arr;
  bulk_errors          EXCEPTION;
  PRAGMA               EXCEPTION_INIT (bulk_errors, -24381);
  n_err PLS_INTEGER := 0;

BEGIN

  FORALL i IN 1..p_ty_emp_in_lis.COUNT
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
        p_ty_emp_in_lis(i).last_name,
        p_ty_emp_in_lis(i).email,
        SYSDATE,
        p_ty_emp_in_lis(i).job_id,
        p_ty_emp_in_lis(i).salary
    )
    RETURNING ty_emp_out_obj (employee_id, To_Char(To_Date(employee_id,'J'),'JSP')) BULK COLLECT INTO x_ty_emp_out_lis;

EXCEPTION
  WHEN bulk_errors THEN

    l_ty_emp_out_lis := x_ty_emp_out_lis;

    FOR i IN 1 .. sql%BULK_EXCEPTIONS.COUNT LOOP
      IF i > x_ty_emp_out_lis.COUNT THEN
        x_ty_emp_out_lis.Extend;
      END IF;
      x_ty_emp_out_lis (SQL%Bulk_Exceptions (i).Error_Index) := ty_emp_out_obj (0, SQLERRM (- (SQL%Bulk_Exceptions (i).Error_Code)));
    END LOOP;

    FOR i IN 1..p_ty_emp_in_lis.COUNT LOOP
      IF i > x_ty_emp_out_lis.COUNT THEN
        x_ty_emp_out_lis.Extend;
      END IF;
      IF x_ty_emp_out_lis(i).employee_id = 0 THEN
        n_err := n_err + 1;
      ELSE
        x_ty_emp_out_lis(i) := l_ty_emp_out_lis(i - n_err);
      END IF;
    END LOOP;

END AIP_Save_Emps;

END Emp_WS;
/
</pre>
</div>

There is a demo recording, jdbc_demo_z4Mc7R7V4f.mp4, of installing and running the code, Oracle and Java, in the [GitHub container folder](https://github.com/BrenPatF/wp_ghp_migration/tree/master/a-template-script-for-jdbc-integration-testing-of-oracle-procedures)