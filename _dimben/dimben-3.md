---
layout: post
title: "DIMBEN 3: Bracket Parsing SQL"
date: 2017-02-05
migrated: true
dimben_prev: /dimben/dimben-2/
dimben_next: /dimben/dimben-4/
categories: 
  - "analytics"
  - "match_recognize"
  - "oracle"
  - "performance"
  - "pipelined"
  - "plsql"
  - "sql"
  - "v12"
tags: 
  - "analytics"
  - "benchmarking"
  - "match_recognize"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "sql"
---
#### Part 3 in a series on: Dimensional Benchmarking of SQL Performance

<div class="dimben-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 DIMBEN Series:</strong>
  <a href="/migrated/dimben-series-index/">Index</a>
  {% if page.dimben_prev %}
    | <a href="{{ page.dimben_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.dimben_next %}
    | <a href="{{ page.dimben_next }}">Next ▶</a>
  {% endif %}
</div>

I noticed an interesting thread on OTN recently, [Matching ( in a string](https://forums.oracle.com/ords/apexds/post/matching-in-a-string-4165). It's about using SQL to find matching bracket pairs (technically 'parentheses' but 'brackets' is shorter, and it makes no difference to the SQL). Incidentally, I found recently that nested bracket expressions are a nice way of compactly representing the structure of complex hash join execution plans, [A Note on Oracle Join Orders and Hints](https://brenpatf.github.io/migrated/a-note-on-oracle-join-orders-and-hints/).

I thought it would be interesting to run some of the solution queries through my benchmarking package to test performance, available on GitHub and described on the [index page](/migrated/dimben-series-index/) for this series of articles. I decided to consider only the queries that addressed multiple records, and the form of the problem that requires returning record uid, opening and closing bracket positions, plus the substring enclosed. These were by 'mathguy' and 'Peter vd Zwan', and I made very minor tweaks for consistency. I also wrote a query myself using PL/SQL in an inline SQL function using the new (v12.1) 'WITH Function' functionality, and copied this to a version using a pipelined database function to check for any performance differences. The four queries tested were then:

- CBL\_QRY, mathguy: Connect By, Analytics, Regex
- MRB\_QRY, Peter vd Zwan: Connect By, Match\_Recognize
- WFB\_QRY, me: With PL/SQL Function, Arrays
- PFB\_QRY, me: Pipelined PL/SQL Function, Arrays

## Bracket Pair Definition

Consider a function (BrDiff) defined at each character position as the difference between the number of opening and closing brackets to the left of, or at, that position.

A closing bracket closes an opening bracket if it is the first closing bracket where BrDiff is 1 less than BrDiff at the opening bracket. If all brackets are in some pair, then the expression can be considered well-formed.

This can be illustrated with a diagram for the fourth functional example below.

<img src="/migrated_images/2017/02/Diff_Map.png" alt="" title="" />

## Test Problem

### BRACKET\_STRINGS Table

```sql
CREATE TABLE bracket_strings (id NUMBER, str VARCHAR2(4000))
/
```

### Functional Test Data

I took four of mathguy's test records, excluding the (deliberately) badly-formed strings, and which included some embedded returns:

```
Test Data

     ID STR
------- ----------------------------------------
      1  ((Hello ( Hi Hi hi ( A B C ( D)) (EF)
        why Whwy whyhhh )
        )
        )

      2 (1+3*(3-1) + 3*(2+1))
      3 ()()*(())a()(())
      4 b0(b1(b2(b3(x))(xy)))
```

Result

```
     ID      O_POS      C_POS STR
------- ---------- ---------- ----------------------------------------
      1          2         60 ((Hello ( Hi Hi hi ( A B C ( D)) (EF)
                              why Whwy whyhhh )
                              )
                              )

      1          3         58 (Hello ( Hi Hi hi ( A B C ( D)) (EF)
                              why Whwy whyhhh )
                              )

      1         10         56 ( Hi Hi hi ( A B C ( D)) (EF)
                              why Whwy whyhhh )

      1         21         33 ( A B C ( D))
      1         29         32 ( D)
      1         35         38 (EF)
      2          1         21 (1+3*(3-1) + 3*(2+1))
      2          6         10 (3-1)
      2         16         20 (2+1)
      3          1          2 ()
      3          3          4 ()
      3          6          9 (())
      3          7          8 ()
      3         11         12 ()
      3         13         16 (())
      3         14         15 ()
      4          3         21 (b1(b2(b3(x))(xy)))
      4          6         20 (b2(b3(x))(xy))
      4          9         15 (b3(x))
      4         12         14 (x)
      4         16         19 (xy)

21 rows selected.
```

All queries returned the expected results above.

### Performance Test Data

Each test set consisted of 100 records with the str column containing the brackets expression dependent on width (_w_) and depth (_d_) parameters, as follows:

- Each str column contains _w_ bracket pairs
- The str column begins with a 3-character record number
- After the record number, the str column begins with _d_ opening brackets with 3 characters of text, like: '(001', etc., followed by the _d_ closing brackets, then the remaining _w_\-_d_ pairs in an unnested sequence, like '(001)'

When _w_\=_d_ the pairs are fully nested, and when _d_\=0 there is no nesting, just a sequence of '(123)' stringss.

This choice of test data sets allows us to see if both number of brackets, and bracket nesting have any effect on performance.

- Depth fixed, at small width point; width varies: d=100, w=(100, 200, 300, 400)
- Width fixed at high depth point; depth varies: w=400, d=(0, 100, 200, 300, 400)

The output from the test queries therefore consists of 100\*_w_ records with a record identifier and a bracketed string. For performance testing purposes the benchmarking framework writes the results to a file in csv format, while counting only the query steps in the query timing results.

All the queries showed strong time correlation with width, with smaller correlation with depth.

## Connect By, Analytics, Regex Query (CBL\_QRY)

```sql
WITH    d ( id, str, pos ) as (
      select id, str, regexp_instr(str, '\(|\)', 1, level)
      from   bracket_strings
      connect by level <= length(str) - length(translate(str, 'x()', 'x'))
             and prior id = id
             and prior sys_guid() is not null
    ),
    p ( id, str, pos, flag, l_ct, ct ) as (
      select id, str, pos, case substr(str, pos, 1) when '(' then 1 else -1 end,
             sum(case substr(str, pos, 1) when '(' then 1         end) over (partition by id order by pos),
             sum(case substr(str, pos, 1) when '(' then 1 else -1 end) over (partition by id order by pos)
      from   d
    ),
    f ( id, str, pos, flag, l_ct, ct, o_ct ) as (
      select id, str, pos, flag, l_ct, ct + case flag when 1 then 0 else 1 end as ct,
             row_number() over (partition by id, flag, ct order by pos)
      from   p
    )
select   /*+ CBL_QRY gather_plan_statistics */ id,
        min(case when flag =  1 then pos end) as o_pos,
        min(case when flag = -1 then pos end) as c_pos,
                                Substr (str, min(case when flag =  1 then pos end), min(case when flag = -1 then pos end) - min(case when flag =  1 then pos end) + 1) str
from    f
group by id, str, ct, o_ct
order by 1, 2
```
### Execution Plan
```
Plan hash value: 2736674058

------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                 |      1 |        |  40000 |00:00:16.30 |      43 |  40000 |  40000 |       |       |          |         |
|   1 |  SORT ORDER BY                      |                 |      1 |    100 |  40000 |00:00:16.30 |      43 |  40000 |  40000 |    21M|  1702K|   19M (0)|         |
|   2 |   HASH GROUP BY                     |                 |      1 |    100 |  40000 |00:00:16.24 |      43 |  40000 |  40000 |    85M|  7293K|   85M (0)|         |
|   3 |    VIEW                             |                 |      1 |    100 |  80000 |00:01:19.35 |      43 |  40000 |  40000 |       |       |          |         |
|   4 |     WINDOW SORT                     |                 |      1 |    100 |  80000 |00:01:19.27 |      43 |  40000 |  40000 |   175M|  4458K|   97M (1)|     157K|
|   5 |      VIEW                           |                 |      1 |    100 |  80000 |00:01:05.90 |      40 |  20000 |  20000 |       |       |          |         |
|   6 |       WINDOW SORT                   |                 |      1 |    100 |  80000 |00:01:05.86 |      40 |  20000 |  20000 |   175M|  4457K|   97M (0)|     157K|
|   7 |        VIEW                         |                 |      1 |    100 |  80000 |00:00:09.77 |      38 |      0 |      0 |       |       |          |         |
|*  8 |         CONNECT BY WITHOUT FILTERING|                 |      1 |        |  80000 |00:00:02.26 |      38 |      0 |      0 |   267K|   267K|  237K (0)|         |
|   9 |          TABLE ACCESS FULL          | BRACKET_STRINGS |      1 |    100 |    100 |00:00:00.01 |      38 |      0 |      0 |       |       |          |         |
------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   8 - access("ID"=PRIOR NULL)
```

### Notes on CBL\_QRY

Subquery d uses regexp\_instr with connect by to generate rows for each bracket.

## Connect By, Match\_Recognize Query (MRB\_QRY)

```sql
WITH b as
(
select
  substr(str,level,1) s
  ,level n
  ,id
  ,str
from
  bracket_strings
connect by
id =  prior id
and substr(str,level,1) is not null
and prior sys_guid() is not null
)
select  /*+ MRB_QRY gather_plan_statistics */ 
  id
  ,o_pos
  ,c_pos
  ,substr(str,o_pos,c_pos - o_pos + 1) str
from
b
MATCH_RECOGNIZE (
partition by id
ORDER BY n
MEASURES 
  str as str
  ,FIRST( N) AS o_pos
  ,LAST( N) AS c_pos
one ROW PER MATCH
AFTER MATCH SKIP to next row
PATTERN (ob (ob | nb | cb)*? lcb)
DEFINE
  ob as ob.s = '('
  ,cb as cb.s = ')'
  ,nb as nb.s not in ('(',')')
  ,lcb as lcb.s = ')' and (count(ob.s) = count(cb.s) + 1)
) MR
```
### Execution Plan
```
Plan hash value: 2214599770

-----------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                 |      1 |        |  40000 |00:03:38.26 |      40 |   4473K|  50075 |       |       |          |
|   1 |  SORT ORDER BY                   |                 |      1 |    100 |  40000 |00:03:38.26 |      40 |   4473K|  50075 |    21M|  1702K|   19M (0)|
|   2 |   VIEW                           |                 |      1 |    100 |  40000 |00:04:48.17 |      40 |   4473K|  50075 |       |       |          |
|   3 |    MATCH RECOGNIZE SORT          |                 |      1 |    100 |  40000 |00:04:48.16 |      40 |   4473K|  50075 |   440M|  6922K|  163M (0)|
|   4 |     VIEW                         |                 |      1 |    100 |    200K|00:00:04.65 |      38 |      0 |      0 |       |       |          |
|*  5 |      CONNECT BY WITHOUT FILTERING|                 |      1 |        |    200K|00:00:04.54 |      38 |      0 |      0 |   267K|   267K|  237K (0)|
|   6 |       TABLE ACCESS FULL          | BRACKET_STRINGS |      1 |    100 |    100 |00:00:00.01 |      38 |      0 |      0 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   5 - access("ID"=PRIOR NULL)
```

### Notes on MRB\_QRY

This uses the v12.1 feature Match\_Recognize operating on the charcaters in the string after conversion to rows.

## With PL/SQL Function, Arrays Query (WFB\_QRY)

```sql
WITH FUNCTION Parse_Brackets (p_str VARCHAR2) RETURN bra_lis_type IS /* WFB_QRY */ 
  c_n_ob       CONSTANT PLS_INTEGER := Length (p_str) - Length (Replace (p_str, '(', ''));
  l_ob_lis              SYS.ODCINumberList := SYS.ODCINumberList();
  l_cb_lis              SYS.ODCINumberList := SYS.ODCINumberList();
  TYPE b_rec_type   IS  RECORD (pos INTEGER, diff INTEGER);
  TYPE b_lis_type   IS  VARRAY(32767) OF b_rec_type;
  l_b_lis               b_lis_type := b_lis_type(NULL);
  l_bra_lis             bra_lis_type := bra_lis_type();
  n_b                   PLS_INTEGER := 0;
  n_ob                  PLS_INTEGER := 0;
  n_cb                  PLS_INTEGER := 0;
  l_chr                 VARCHAR2(1);
  l_o_diff              PLS_INTEGER;
BEGIN
  IF c_n_ob = 0 THEN
    RETURN NULL;
  END IF;
  l_ob_lis.EXTEND (c_n_ob);
  l_bra_lis.EXTEND (c_n_ob);
  l_cb_lis.EXTEND (c_n_ob);
  l_b_lis.EXTEND (c_n_ob + c_n_ob);
   FOR i IN 1..Length (p_str) LOOP
     l_chr := Substr (p_str, i, 1);
    IF l_chr NOT IN ('(', ')') THEN CONTINUE; END IF;
    n_b := n_b + 1;
    l_b_lis(n_b).pos := i;
     IF l_chr = '(' THEN
      n_ob := n_ob + 1;
      l_ob_lis(n_ob) := n_b;
    ELSE
      n_cb := n_cb + 1;
      l_cb_lis(n_cb) := n_b;
    END IF;
    l_b_lis(n_b).diff := n_ob - n_cb;
   END LOOP;
  FOR i IN 1..n_ob LOOP
    l_o_diff := l_b_lis (l_ob_lis(i)).diff;
    FOR j IN 1..n_cb LOOP
      IF l_b_lis (l_cb_lis(j)).pos < l_b_lis (l_ob_lis(i)).pos THEN CONTINUE; END IF;
      IF l_o_diff = l_b_lis (l_cb_lis(j)).diff + 1 THEN
        l_bra_lis(i) := bra_rec_type (l_b_lis(l_ob_lis(i)).pos, l_b_lis(l_cb_lis(j)).pos, Substr (p_str, l_b_lis(l_ob_lis(i)).pos, l_b_lis(l_cb_lis(j)).pos - l_b_lis(l_ob_lis(i)).pos + 1));
        EXIT;
      END IF;
    END LOOP;
   END LOOP;
  RETURN l_bra_lis;
END;
SELECT
    b.id, t.o_pos, t.c_pos, t.str
  FROM bracket_strings b
  OUTER APPLY TABLE (Parse_Brackets (b.str)) t
 ORDER BY 1, 2
```

### Notes on WFB\_QRY

This uses the v12.1 feature whereby a PL/SQL function can be included directly in a query.

## Pipelined PL/SQL Function, Arrays Query (PFB\_QRY)

### WFB\_QRY - Pipelined Function

```sql
CREATE OR REPLACE TYPE bra_rec_type IS OBJECT (o_pos INTEGER, c_pos INTEGER, str VARCHAR2(4000));
/
CREATE TYPE bra_lis_type IS VARRAY(4000) OF bra_rec_type;
/
FUNCTION Parse_Brackets (p_str VARCHAR2) RETURN bra_lis_type PIPELINED IS
  c_n_ob       CONSTANT PLS_INTEGER := Length (p_str) - Length (Replace (p_str, '(', ''));
  l_ob_lis              SYS.ODCINumberList := SYS.ODCINumberList();
  l_cb_lis              SYS.ODCINumberList := SYS.ODCINumberList();
  TYPE b_rec_type   IS  RECORD (pos INTEGER, diff INTEGER);
  TYPE b_lis_type   IS  VARRAY(32767) OF b_rec_type;
  l_b_lis               b_lis_type := b_lis_type(NULL);
  l_bra_lis             bra_lis_type := bra_lis_type();
  n_b                   PLS_INTEGER := 0;
  n_ob                  PLS_INTEGER := 0;
  n_cb                  PLS_INTEGER := 0;
  l_chr                 VARCHAR2(1);
  l_o_diff              PLS_INTEGER;
BEGIN
  IF c_n_ob = 0 THEN
    RETURN;
  END IF;
  l_ob_lis.EXTEND (c_n_ob);
  l_bra_lis.EXTEND (c_n_ob);
  l_cb_lis.EXTEND (c_n_ob);
  l_b_lis.EXTEND (c_n_ob + c_n_ob);
  FOR i IN 1..Length (p_str) LOOP
    l_chr := Substr (p_str, i, 1);
    IF l_chr NOT IN ('(', ')') THEN CONTINUE; END IF;
    n_b := n_b + 1;
    l_b_lis(n_b).pos := i;
    IF l_chr = '(' THEN
      n_ob := n_ob + 1;
      l_ob_lis(n_ob) := n_b;
    ELSE
      n_cb := n_cb + 1;
      l_cb_lis(n_cb) := n_b;
    END IF;
    l_b_lis(n_b).diff := n_ob - n_cb;
  END LOOP;
  FOR i IN 1..n_ob LOOP
    l_o_diff := l_b_lis (l_ob_lis(i)).diff;
    FOR j IN 1..n_cb LOOP
      IF l_b_lis (l_cb_lis(j)).pos < l_b_lis (l_ob_lis(i)).pos THEN CONTINUE; END IF;
      IF l_o_diff = l_b_lis (l_cb_lis(j)).diff + 1 THEN
        PIPE ROW (bra_rec_type (l_b_lis(l_ob_lis(i)).pos, l_b_lis(l_cb_lis(j)).pos, Substr (p_str, l_b_lis(l_ob_lis(i)).pos, l_b_lis(l_cb_lis(j)).pos - l_b_lis(l_ob_lis(i)).pos + 1)));
        EXIT;
      END IF;
    END LOOP;
  END LOOP;
END Parse_Brackets;
```
### Execution Plan
```
Plan hash value: 3367347570

---------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                 |      1 |        |  40000 |00:00:01.17 |      38 |       |       |          |
|   1 |  SORT ORDER BY                       |                 |      1 |    816K|  40000 |00:00:01.17 |      38 |    21M|  1702K|   19M (0)|
|   2 |   NESTED LOOPS OUTER                 |                 |      1 |    816K|  40000 |00:00:16.80 |      38 |       |       |          |
|   3 |    TABLE ACCESS FULL                 | BRACKET_STRINGS |      1 |    100 |    100 |00:00:00.01 |      38 |       |       |          |
|   4 |    VIEW                              | VW_LAT_D4FD8C38 |    100 |   8168 |  40000 |00:00:01.13 |       0 |       |       |          |
|   5 |     COLLECTION ITERATOR PICKLER FETCH| PARSE_BRACKETS  |    100 |   8168 |  40000 |00:00:01.10 |       0 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------------------
```

### PFB\_QRY - Query

```sql
SELECT  /*+ PFB_QRY gather_plan_statistics */
        b.id        id, 
        t.o_pos     o_pos, 
        t.c_pos     c_pos,
        t.str       str
  FROM bracket_strings b
  OUTER APPLY TABLE (Strings.Parse_Brackets (b.str)) t
 ORDER BY b.id, t.o_pos
```
### Execution Plan
```
Plan hash value: 3367347570

---------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                 |      1 |        |  40000 |00:00:01.15 |      38 |       |       |          |
|   1 |  SORT ORDER BY                       |                 |      1 |    816K|  40000 |00:00:01.15 |      38 |    21M|  1702K|   19M (0)|
|   2 |   NESTED LOOPS OUTER                 |                 |      1 |    816K|  40000 |00:00:01.95 |      38 |       |       |          |
|   3 |    TABLE ACCESS FULL                 | BRACKET_STRINGS |      1 |    100 |    100 |00:00:00.01 |      38 |       |       |          |
|   4 |    VIEW                              | VW_LAT_D4FD8C38 |    100 |   8168 |  40000 |00:00:01.51 |       0 |       |       |          |
|   5 |     COLLECTION ITERATOR PICKLER FETCH| PARSE_BRACKETS  |    100 |   8168 |  40000 |00:00:01.50 |       0 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------------------
```

### Notes on PFB\_QRY

This uses PL/SQL pipelined database function which is called in the query.

## Performance Testing Results

### Key

If CPU time (y) is proportional to the varying dimension (x), for data points 1 and 2, we would have:

y = kx

and so, for any two data points 1 and 2:

y2.x1/(y1.x2) = 1

We can take the actual value of the linear ratio as a marker for linear proportionality or not.

If CPU time (y) is proportional to the varying dimension (x), for data points 1 and 2, we would have:

y = kx.x

and so, for any two data points 1 and 2:

y2.x1.x1/(y1.x2.x2) = 1

We can take the actual value of the quadratic ratio as a marker for quadratic proportionality or not.

- LRTB = Ratio to best time at high data point
- RTP\_L = Linear ratio as defined above, averaged over successive data points
- RTP\_Q = Quadratic ratio as defined above, averaged over successive data points

### Depth fixed, at small width point; width varies: d=100, w=(100, 200, 300, 400)

<img src="/migrated_images/2017/02/Bracket_D100-2.png" alt="" title="" />

### Notes on Results for Fixed Depth

- CPU time increases with width for all queries at above a linear rate
- CBL\_QRY is significantly faster than MRB\_QRY, which appears to be rising quadratically
- Both WFB\_QRY and PFB\_QRY are much faster than the non-PL/SQL queries
- PFB\_QRY is slightly faster than WFB\_QRY. This could be regarded as too small a difference to be significant, but is consistent across the data points, despite the context switching with database function calls

_Width fixed at high depth point; depth varies: w=400, d=(0, 100, 200, 300, 400)_

<img src="/migrated_images/2017/02/Bracket_W400.png" alt="" title="" />

### Notes on Results for Fixed Width

- CPU time increases with depth for all queries although at a sublinear rate
- CBL\_QRY has a big jump in CPU time between 0 and 100, where nesting starts to come in, and was actually faster than CBL\_QRY without nesting

<div class="dimben-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 DIMBEN Series:</strong>
  <a href="/migrated/dimben-series-index/">Index</a>
  {% if page.dimben_prev %}
    | <a href="{{ page.dimben_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.dimben_next %}
    | <a href="{{ page.dimben_next }}">Next ▶</a>
  {% endif %}
</div>
