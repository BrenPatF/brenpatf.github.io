---
layout: post
title: "Query Structure Diagramming"
date: 2012-08-11
migrated: true
categories: 
  - "data-model"
  - "erd"
  - "oracle"
  - "sql"
  - "subquery-factor"
tags: 
  - "erd"
  - "oracle"
  - "qsd"
  - "sql"
  - "subquery-factor"
---

Last bank holiday Monday I posted a solution to an SQL problem on OTN, and I later thought that the SQL would make a nice example to illustrate my Query Structure Diagramming (QSD) technique. I published my first example of this in May 2009 on scribd,
<iframe class="scribd_iframe_embed" title="A Structured Approach to SQL Query Design" src="https://www.scribd.com/embeds/15723877/content?start_page=1&view_mode=scroll&access_key=key-1wlb575hb2xc5lr97sb8" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View A Structured Approach to SQL Query Design on Scribd" href="https://www.scribd.com/document/15723877/A-Structured-Approach-to-SQL-Query-Design#from_embed" style="color: #098642; text-decoration: underline;">A Structured Approach to SQL Query Design</a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p>
and have continued to develop it in subsequent articles. I use the technique here to illustrate the SQL structure for the OTN example mentioned and also for a second OTN example that I posted shortly after. Both examples structure the queries using subquery factors.

## SQL Subquery Factors


Subquery factors were introduced in Oracle Database v9.2 and have since become a key technique in developing queries of any complexity. They generalise the v8 inline view technique, allowing subqueries now to be declared with an alias (using the 'WITH' clause), then referenced as often as desired later in the query. When referenced multiple times, Oracle's Cost Based Optimiser (CBO) normally executes the subquery once and writes the results to temporary space to aid performance (this can be seen in the Explain Plan as a LOAD AS SELECT action). Using subquery factors can make queries much easier to read, even when they are referenced only once, in which case CBO normally restructures them internally to incorporate them within another subquery or the main query. It is important to note, though, that subquery factors, when retained by CBO, will be joined by full scans when referenced later in the query and in some cases it is more efficient to retain the table references to allow indexed joins (my next blog post will include an example of this).

Subquery factors, along with inline views, are the building blocks of modern SQL, as subroutines are of other languages. My QSDs are intended to show how they allow a procedural flow at the structural level, while retaining the set-based logic within the subqueries.

## Leave and Attendance Query


The problem here is that the poster has a daily attendance table and a leave table, where leave is stored as date ranges, but wants a single query that outputs data in daily form. The keys to this are:

- Realising that you cannot drive from the event tables but must generate a continuous set of days to drive from, for each employee (assuming you don't have them in a separate reference table)
- Converting the leave ranges to leave days by joining to the generated days rowset
- Joining the leave daily rowset with the attendance table by a union

The original post is [Attendance and Leave table Join](https://forums.oracle.com/ords/apexds/post/attendance-and-leave-table-join-7216).

### ERD

<img src="/migrated_images/2012/08/OTN-V1.1-ERD-Leave.jpg" alt="ERD Leave" title="ERD Leave" />

Note that tables were not provided for Employee and Day in the OTN post, but it is useful to include them as entities nonetheless.

### SQL

Note that in the query below, both the date range and the employee set are generated from the transactional data, which is obviously an artificial feature arising from this being for a problem on a forum, but it's no harm in terms of the purpose of this article.


```sql
WITH ext AS (
SELECT Min (att_date) min_date, Max (att_date) - Min (att_date) + 1 n_days
  FROM attendance
), dys AS (
SELECT min_date + LEVEL - 1 day
  FROM ext
CONNECT BY LEVEL < n_days + 1
), ems AS (
SELECT emp_id
  FROM attendance
 UNION
SELECT employee_number
  FROM leave
), edy AS (
SELECT
    dys.day,
    ems.emp_id
  FROM dys
 CROSS JOIN ems
), ldy AS (
SELECT
    edy.emp_id,
    edy.day,
    lve.leave_reason
  FROM edy
  JOIN leave                lve
    ON edy.day              BETWEEN lve.date_start AND lve.date_end
   AND lve.employee_number  = edy.emp_id
), uni AS (
 SELECT
    emp_id,
    att_date,
    timein,
    timeout,
    late_in,
    early_out,
    reason
  FROM attendance
UNION
SELECT
    emp_id,
    day,
    NULL,
    NULL,
    NULL,
    NULL,
    leave_reason
  FROM ldy
)
 SELECT
    edy.emp_id,
    edy.day,
    uni.timein,
    uni.timeout,
    uni.late_in,
    uni.early_out,
    uni.reason
  FROM edy
  LEFT JOIN uni
    ON uni.att_date     = edy.day
   AND uni.emp_id       = edy.emp_id
 ORDER BY 1, 2
```

### QSD

<img src="/migrated_images/2012/08/OTN-V1.1-QSD-Leave.jpg" alt="QSD Leave" title="QSD Leave" />

## Counting Flight Statistics Query


The problem here is that the poster has three tables with data on events for frequent fliers and wants to show aggregate counts by year, but wants all years within a range to be included in the output, including years with no events. The key to this is realising that you cannot drive from the event tables but must generate a continuous set of years to drive from (assuming you don't have them in a separate reference table). The original post is [Multiple Count aggregates from different sources grouped by Year](https://forums.oracle.com/ords/apexds/post/multiple-count-aggregates-from-different-sources-grouped-by-4155).

### ERD

<img src="/migrated_images/2012/08/OTN-V1.1-ERD-Flight.jpg" alt="ERD Flight" title="ERD Flight" />

Note that a table was not provided for Year in the OTN post, but it is useful to include it as an entity nonetheless.

### SQL

```sql
WITH yrs AS (
SELECT Add_Months (To_Date ('01011989', 'ddmmyyyy'), 12*LEVEL) YEAR
  FROM DUAL
CONNECT BY LEVEL < 24
), ffe AS (
SELECT Trunc (start_date, 'YEAR') year, Count(*) n_enr
  FROM freq_flyer_enrollment
 GROUP BY Trunc (start_date, 'YEAR')
), ffs AS (
SELECT Trunc (survey_date, 'YEAR') year, Count(*) n_sur
  FROM freq_flyer_survey
 GROUP BY Trunc (survey_date, 'YEAR')
), fff AS (
SELECT Trunc (flt_date, 'YEAR') year, Count(*) n_fly
  FROM freq_flyer_flights
 GROUP BY Trunc (flt_date, 'YEAR')
)
SELECT  yrs.year,
        Nvl (ffe.n_enr, 0) n_enr,
        Nvl (ffs.n_sur, 0) n_sur,
        Nvl (fff.n_fly, 0) n_fly
  FROM yrs
  LEFT JOIN ffe
    ON ffe.year = yrs.year
  LEFT JOIN ffs
    ON ffs.year = yrs.year
  LEFT JOIN fff
    ON fff.year = yrs.year
 ORDER BY 1
```

### QSD

<img src="/migrated_images/2012/08/OTN-V1.1-QSD-Flight.jpg" alt="QSD Flight" title="QSD Flight" />
