---
layout: post
title: "Case Expressions and Ignoring Nulls in Analytic Functions"
date: 2012-01-15
categories: 
  - "analytics"
  - "oracle"
  - "sql"
tags: 
  - "analytics"
  - "first_value"
  - "last_value"
  - "oracle"
  - "sql"
---

IGNORE NULLS: This is not the mission statement of a right-wing political party :), but an optional clause in some of Oracle's analytic functions. Recently I posted a query on OTN that used Last\_Value with this clause to simplify another poster's solution to a grouping problem. It occurred to me then that the clause is much more powerful than is generally appreciated, and I'll try to demonstrate that below.

Oracle describes the analytic function First\_Value in its SQL manual thus:

> 'FIRST\_VALUE is an analytic function. It returns the first value in an ordered set of values. If the first value in the set is null, then the function returns NULL unless you specify IGNORE NULLS. This setting is useful for data densification.'

Although accurate, the reference to data densification possibly undersells it: When used in conjunction with CASE expressions, IGNORE NULLS allows you effectively to include a WHERE condition on the rows processed by the function, in addition to the partitioning and windowing conditions. This is useful because the latter two conditions have to be defined relative to the current row, whereas the new condition is absolute. Let's take an example based on Oracle's demo HR schema.

Suppose that we want a list of employees, and for each employee we want to assign another employee, perhaps as a mentor. We'll take the following rules:

- The mentor has to be in the same department
- The mentor has to earn more than the employee, but not too much more, say up to 1000 more
- The mentor has to have worked at the company since at least a certain date, say 01-JAN-1998

Subject to these rules, we'll take the highest-earning (or maybe the lowest, let's try both) employee as mentor, and won't worry about tie-breaks for this post.

The objective of maximising (or minimising) the mentor's salary subject to the rules implies the use of Last\_Value (or First\_Value) with an ordering on salary (we can't use Max because we don't want to return just the salary). The first two conditions can be implemented as partioning and windowing clauses respectively, and operate relative to the current employee. The third condition is absolute though and can't be implemented within the analytic clause itself, which is where IGNORE NULLS comes in. If we make the operand a CASE expression that returns the required details only for employees that meet the required condition and null otherwise, this will implement the required condition. A possible query would be:

```
SELECT	emp.first_name ||' ' || emp.last_name employee,
	dep.department_name dept, 
        To_Char (emp.hire_date, 'DD-MON-YYYY') hire_date, 
        emp.salary,
	Last_Value (CASE WHEN emp.hire_date < '01-JAN-1998' THEN 
                emp.first_name || ' ' || emp.last_name || ', ' || To_Char (emp.hire_date, 'DD-MON-YYYY') || ', ' || emp.salary
                END IGNORE NULLS)
            OVER (PARTITION BY emp.department_id 
			ORDER BY emp.salary
			RANGE BETWEEN 1 FOLLOWING AND 1000 FOLLOWING) mentor
  FROM employees	    emp
  JOIN departments	    dep
    ON dep.department_id    = emp.department_id
 ORDER BY 2, 4, 1;
```

Of course, there may be employees who don't have a mentor on our rules, but here are the first few records for the Shipping department (note that I deleted the department column to reduce scrolling):

Last\_Value:

```
EMPLOYEE            HIRE_DATE   SALARY MENTOR
------------------ ----------- ------ -----------------------------------
TJ Olson            10-APR-1999   2100 Curtis Davies, 29-JAN-1997, 3100
Hazel Philtanker    06-FEB-2000   2200 Julia Nayer, 16-JUL-1997, 3200
Steven Markle       08-MAR-2000   2200 Julia Nayer, 16-JUL-1997, 3200
James Landry        14-JAN-1999   2400 Laura Bissot, 20-AUG-1997, 3300
Ki Gee              12-DEC-1999   2400 Laura Bissot, 20-AUG-1997, 3300
James Marlow        16-FEB-1997   2500 Trenna Rajs, 17-OCT-1995, 3500
Joshua Patel        06-APR-1998   2500 Trenna Rajs, 17-OCT-1995, 3500
Martha Sullivan     21-JUN-1999   2500 Trenna Rajs, 17-OCT-1995, 3500
Peter Vargas        09-JUL-1998   2500 Trenna Rajs, 17-OCT-1995, 3500
Randall Perkins     19-DEC-1999   2500 Trenna Rajs, 17-OCT-1995, 3500
Donald OConnell     21-JUN-1999   2600 Renske Ladwig, 14-JUL-1995, 3600
```

First\_Value:

```
TJ Olson            10-APR-1999   2100 James Marlow, 16-FEB-1997, 2500
Hazel Philtanker    06-FEB-2000   2200 James Marlow, 16-FEB-1997, 2500
Steven Markle       08-MAR-2000   2200 James Marlow, 16-FEB-1997, 2500
James Landry        14-JAN-1999   2400 James Marlow, 16-FEB-1997, 2500
Ki Gee              12-DEC-1999   2400 James Marlow, 16-FEB-1997, 2500
James Marlow        16-FEB-1997   2500 Mozhe Atkinson, 30-OCT-1997, 2800
Joshua Patel        06-APR-1998   2500 Mozhe Atkinson, 30-OCT-1997, 2800
Martha Sullivan     21-JUN-1999   2500 Mozhe Atkinson, 30-OCT-1997, 2800
Peter Vargas        09-JUL-1998   2500 Mozhe Atkinson, 30-OCT-1997, 2800
Randall Perkins     19-DEC-1999   2500 Mozhe Atkinson, 30-OCT-1997, 2800
Donald OConnell     21-JUN-1999   2600 Mozhe Atkinson, 30-OCT-1997, 2800
```

In general, to find the last value of an expression in a record set ordered by a possibly different expression, with an absolute condition on the records to be considered, use the following form of the function:

```
Last_Value (CASE WHEN absolute_condition THEN return_expression END IGNORE NULLS)
	OVER (partitioning_clause ORDER BY order_expression windowing_clause)
```

Note that there is an interesting special case that arises when forming break groups defined by changes in sequential records in an ordered set. The break points can often be obtained by the Lag and Lead analytic functions, and the groups that other records belong to can then be found through expressions of the above type. However, analytic functions can't be nested, so the first step needs to be performed in a separate subquery (inline view or subfactor) -see the first embedded scribd document below for further details on the SQL for this common requirement.

I stated above that we wouldn't worry about tie-breaks in this post, but it's worth mentioning that Oracle allows multiple columns in the ORDER BY only if the windowing clause includes only UNBOUNDED and CURRENT ROW terms. However, you can often pack multiple columns into a single expression by formatting numbers with fixed size and zero-padding etc.

## Other Analytic Functions and Null Values
IGNORE NULLS can also be used with Lead and Lag and the new 11.2 function Nth\_Value, which extends First\_Value, Last\_Value to specific ranked values. It is interesting to note that some of the other functions, such as Sum, ignore nulls implicitly:

```
SELECT 1 + NULL added, Sum (x) summed
  FROM (
SELECT 1 X FROM DUAL
 UNION 
SELECT NULL FROM DUAL);

     ADDED     SUMMED
---------- ----------
                    1
```

In Oracle null signifies an unknown value and therefore adding null to any number, for example, results in null. Technically, you would therefore expect a sum that includes a null value to result in null, but in fact it does not as the SQL above shows. No doubt practicality won out over theory here.

Again, with other functions such as Sum we can apply a condition by using a CASE expression that returns null or zero if the condition is not met, although not with certain functions such as Avg (but where we could sum and count separately and then calculate the average ourselves).

## Other Examples with IGNORE NULLS
The OTN thread mentioned earlier, "Custom ranking OTN Thread", is no longer available. The table temp3 contains transactions, some of which are defined to be interest-only transactions based on a condition on two fields. The requirement is to list all non-interest transactions but to summarise interest-only transactions beneath the previous non-interest transaction. My solution, simplifying an earlier proposed solution, involved using Last\_Value with IGNORE NULLS in a subfactor to associate the prior non-interest transaction with all transactions, and then doing a GROUP BY in the main query.

```
BREAK ON trx_grp
WITH grp AS (
SELECT  Last_Value (CASE WHEN tran_id != 'SHD' OR flg = 'N' THEN tran_code END IGNORE NULLS)
            OVER (ORDER BY tran_code) trx_grp,
        tran_id, flg, tran_date, tran_code, amt
 FROM temp3
)
SELECT tran_id, flg, Min (tran_date) "From", Max (tran_date) "To", trx_grp, Sum (amt)
  FROM grp
 GROUP BY tran_id, flg, trx_grp
 ORDER BY trx_grp, flg
/

TRA FLG From      To           TRX_GRP   SUM(AMT)
--- --- --------- --------- ---------- ----------
ADV N   31-OCT-11 31-OCT-11   59586455         50
SHD Y   01-NOV-11 02-NOV-11                    10
PAY N   03-NOV-11 03-NOV-11   59587854         50
PAY N   03-NOV-11 03-NOV-11   59587855         50
SHD Y   03-NOV-11 05-NOV-11                     9
PAY N   06-NOV-11 06-NOV-11   59588286         50
SHD N   06-NOV-11 06-NOV-11   59590668         50
PAY N   07-NOV-11 07-NOV-11   59590669         50

8 rows selected.
```

I have also used First\_Value, Last\_Value to help form range-based groups, here (if you can't see the document, 'Forming Range-Based Break Groups with Advanced SQL', it is also in the previous post, up the page):

<iframe class="scribd_iframe_embed" title="Forming Range-Based Break Groups With Advanced SQL" src="https://www.scribd.com/embeds/57696875/content?start_page=1&view_mode=scroll&access_key=key-1exh47ppyi1yklevkz9p" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View Forming Range-Based Break Groups With Advanced SQL on Scribd" href="https://www.scribd.com/document/57696875/Forming-Range-Based-Break-Groups-With-Advanced-SQL#from_embed" style="color: #098642; text-decoration: underline;"> Forming Range-Based Break Groups With Advanced SQL </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

## Using KEEP with the First and Last Functions
Oracle says:

> FIRST and LAST are very similar functions. Both are aggregate and analytic functions that operate on a set of values from a set of rows that rank as the FIRST or LAST with respect to a given sorting specification. If only one row ranks as FIRST or LAST, then the aggregate operates on the set with only one element.

and describes their value thus:

> When you need a value from the first or last row of a sorted group, but the needed value is not the sort key, the FIRST and LAST functions eliminate the need for self-joins or views and enable better performance.

This seems at first pretty similar to First\_Value and Last\_Value, so we might ask what they could do in relation to our requirements above. The problem for us is that we can't include a windowing clause as it's not allowed in this case, so we'd have to accept the maximum salary within the allowed date range:

```
SELECT	emp.first_name ||' ' || emp.last_name employee,
	dep.department_name dept, 
        To_Char (emp.hire_date, 'DD-MON-YYYY') hire_date, 
        emp.salary,
	Max (CASE WHEN emp.hire_date < '01-JAN-1998' THEN 
                emp.first_name || ' ' || emp.last_name || ', ' || To_Char (emp.hire_date, 'DD-MON-YYYY') || ', ' || emp.salary
                END)
	 KEEP (DENSE_RANK LAST ORDER BY emp.salary) 
            OVER (PARTITION BY emp.department_id) mentor
  FROM employees	        emp
  JOIN departments	        dep
    ON dep.department_id    = emp.department_id
 ORDER BY 2, 4, 1;

[dept deleted from output]
EMPLOYEE            HIRE_DATE   SALARY MENTOR
------------------ ----------- ------ -----------------------------------
TJ Olson            10-APR-1999   2100 Adam Fripp, 10-APR-1997, 8200
Hazel Philtanker    06-FEB-2000   2200 Adam Fripp, 10-APR-1997, 8200
Steven Markle       08-MAR-2000   2200 Adam Fripp, 10-APR-1997, 8200
James Landry        14-JAN-1999   2400 Adam Fripp, 10-APR-1997, 8200
Ki Gee              12-DEC-1999   2400 Adam Fripp, 10-APR-1997, 8200
James Marlow        16-FEB-1997   2500 Adam Fripp, 10-APR-1997, 8200
Joshua Patel        06-APR-1998   2500 Adam Fripp, 10-APR-1997, 8200
Martha Sullivan     21-JUN-1999   2500 Adam Fripp, 10-APR-1997, 8200
Peter Vargas        09-JUL-1998   2500 Adam Fripp, 10-APR-1997, 8200
Randall Perkins     19-DEC-1999   2500 Adam Fripp, 10-APR-1997, 8200
Donald OConnell     21-JUN-1999   2600 Adam Fripp, 10-APR-1997, 8200
```

However, I thought these functions worth mentioning in this post because they can be very useful but seem to be not very well known. People often simulate the functions, in aggregate form anyway, by means of another analytic function, Row\_Number, within an inline view but, as is generally the case, the native constructs are simpler and more efficient. I benchmarked various approaches for the aggregation case here (if you can't see the document, 'SQL Pivot and Prune Queries - Keeping an Eye on Performance', it is also in the previous post, up the page):

<iframe class="scribd_iframe_embed" title="SQL Pivot and Prune Queries - Keeping an Eye on Performance" src="https://www.scribd.com/embeds/54433084/content?start_page=1&view_mode=scroll&access_key=key-osozbbqczb7m4qadwiy" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View SQL Pivot and Prune Queries - Keeping an Eye on Performance on Scribd" href="https://www.scribd.com/document/54433084/SQL-Pivot-and-Prune-Queries-Keeping-an-Eye-on-Performance#from_embed" style="color: #098642; text-decoration: underline;"> SQL Pivot and Prune Queries - Keeping an Eye on Performance </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 
