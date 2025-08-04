---
layout: post
title: "DIMBEN 6: String Splitting SQL"
date: 2017-03-12
migrated: true
dimben_prev: /dimben/dimben-5/
dimben_next: /dimben/dimben-7/
categories: 
  - "performance"
  - "plsql"
  - "sql"
  - "testing"
tags: 
  - "cpu"
  - "match_recognize"
  - "model-2"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "tuning"
---
#### Part 6 in a series on: Dimensional Benchmarking of SQL Performance

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

I noticed a question on AskTom last November concerning SQL for splitting delimited strings, [Extract domain names from a column having multiple email addresses](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:9532080700346185258), a kind of question that arises frequently on the forums. There was some debate between reviewers Rajeshwaran Jeyabal, Stew Ashton and the AskTom guys on whether an XML-based solution performs better or worse than a more 'classic' solution based on the Substr and Instr functions and collections. AskTom's Chris Saxon noted:

> For me this just highlights the importance of testing in your own environment with your own data. Just because someone produced a benchmark showing X is faster, doesn't mean it will be for you.

For me, relative performance is indeed frequently dependent on the size and 'shape' of the data used for testing. As I have my own dimensional benchmarking framework, available on GitHub and described on the [index page](/migrated/dimben-series-index/) for this series of articles, I was able to very quickly adapt Rajesh's test data to benchmark across numbers of records and numbers of delimiters, and I put the results on the thread. I then decided to take the time to expand the scope to include other solutions, and to use more general data sets, where the token lengths vary as well as the number of tokens per record.

In fact the scope expanded quite a bit, as I found more and more ways to solve the problem, and I have only now found the time to write it up. Here is a list of all the queries considered:

### Queries using Connect By for row generation

- MUL\_QRY - Cast/Multiset to correlate Connect By
- LAT\_QRY - v12 Lateral correlated Connect By
- UNH\_QRY - Uncorrelated Connect By unhinted
- RGN\_QRY - Uncorrelated Connect By with leading hint
- GUI\_QRY - Connect By in main query using sys\_guid trick
- RGX\_QRY - Regular expression function, Regexp\_Substr

### Queries not using Connect By for row generation

- XML\_QRY - XMLTABLE
- MOD\_QRY - Model clause
- PLF\_QRY - database pipelined function
- WFN\_QRY - 'WITH' PL/SQL function directly in query
- RSF\_QRY - Recursive Subquery Factor
- RMR\_QRY - Match\_Recognize

# Test Problem

### DELIMITED\_LISTS Table

```
CREATE TABLE delimited_lists(id INT, list_col VARCHAR2(4000))
/
```

### Functional Test Data

The test data consist of pipe-delimited tokens ('\|') in a VARCHAR2(4000) column in a table with a separate integer unique identifier. For functional testing we will add a single 'normal' record with two tokens, plus four more records designed to validate null-token edge cases as follows:

1. Normal case, two tokens
2. Leading null token, then two not null tokens
3. Trailing null token, after two not null tokens
4. Two not null tokens, with a null token in the middle
5. Two null tokens only

```
     ID LIST_COL
------- ------------------------------------------------------------
      1 token_11|token_12
      2 |token_21|token_22
      3 token_31|token_32|
      4 token_41||token_42
      5 |
```

### Functional Test Results

```
     ID TOKEN
------- ----------
      1 token_11
      1 token_12
      2
      2 token_21
      2 token_22
      3 token_31
      3 token_32
      3
      4 token_41
      4
      4 token_42
      5
      5

13 rows selected.
```

All queries returned the expected results above, except that the XML query returned 12 rows with only a single null token returned for record 5. In the performance testing, no null tokens were included, and all queries returned the same results.

### Performance Test Data

Each test set consisted of 3,000 records with the list\_col column containing the delimited string dependent on width (_w_) and depth (_d_) parameters, as follows:

- Each record contains _w_ tokens
- Each token contains _d_ characters from the sequence _1234567890_ repeated as necessary

The output from the test queries therefore consists of 3,000\*_w_ records with a unique identifier and a token of length _d_. For performance testing purposes the benchmarking framework writes the results to a file in csv format, while counting only the query steps in the query timing results.

In Oracle upto version 11.2 VARCHAR2 expressions cannot be longer than 4,000 characters, so I decided to run the framework for four sets of parameters, as follows:

- Depth fixed, high; width range low: d=18, w=(50,100,150,200)
- Depth fixed, low; width range high: d=1, w=(450,900,1350,1800)
- Width fixed, low; depth range high: w=5, d=(195,390,585,780)
- Width fixed, high; depth range low: w=150, d=(6,12,18,24)

All the queries showed strong time correlation with width, while a few also showed strong correlation with depth.

# Queries

All execution plans are from the data point with Width=1800, Depth=1, which has the largest number of tokens per record.

## Multiset Query (MUL\_QRY)

```sql
SELECT d.id   id,
       Substr (d.list_col, Instr ('|' || d.list_col, '|', 1, t.COLUMN_VALUE), Instr (d.list_col || '|', '|', 1, t.COLUMN_VALUE) - Instr ('|' || d.list_col, '|', 1, t.COLUMN_VALUE)) token
  FROM delimited_lists d, 
       TABLE (CAST (MULTISET (SELECT LEVEL FROM DUAL CONNECT BY LEVEL <= Nvl (Length(d.list_col), 0) - Nvl (Length (Replace (d.list_col, '|')), 0) + 1) AS SYS.ODCINumberlist)) t

Plan hash value: 462687286
```
### Execution Plan
```
--------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                 |      1 |        |   5400K|00:01:52.86 |    3009 |       |       |          |
|   1 |  NESTED LOOPS                       |                 |      1 |     24M|   5400K|00:01:52.86 |    3009 |       |       |          |
|   2 |   TABLE ACCESS FULL                 | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    3009 |       |       |          |
|   3 |   COLLECTION ITERATOR SUBQUERY FETCH|                 |   3000 |   8168 |   5400K|00:01:49.96 |       0 |       |       |          |
|   4 |    CONNECT BY WITHOUT FILTERING     |                 |   3000 |        |   5400K|00:01:48.83 |       0 |  2048 |  2048 | 2048  (0)|
|   5 |     FAST DUAL                       |                 |   3000 |      1 |   3000 |00:00:00.01 |       0 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------
```

### Notes on MUL\_QRY

This is the 'classic' CONNECT BY solution referred to above, which appears frequently on AskTom and elsewhere, and I copied the version used by Jayesh. The somewhat convoluted casting between subquery and array and back to SQL record via multiset allows the prior table in the from list to be referenced within the inline view, which is otherwise not permitted in versions earlier than 12.1, where the LATERAL keyword was introduced.

Despite this query being regarded as the 'classic' CONNECT BY solution to string-splitting, we will find that it is inferior in performance to a query I wrote myself across all data points considered. The new query is also simpler, but is not the most efficient of all methods, as we see later.

## Lateral Query (LAT\_QRY)

```sql
SELECT d.id                id,
       l.subs              token
FROM delimited_lists d
CROSS APPLY (
  SELECT Substr (d.list_col, pos + 1, Lead (pos, 1, 4000) OVER (ORDER BY pos) - pos - 1) subs, pos
    FROM (
    SELECT Instr (d.list_col, '|', 1, LEVEL) pos
      FROM DUAL
    CONNECT BY
      LEVEL <= Length (d.list_col) - Nvl (Length (Replace (d.list_col, '|')), 0) + 1
    )
) l
```
### Execution Plan
```
Plan hash value: 631504984

-----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                 |      1 |        |   5400K|00:15:41.97 |    3009 |       |       |          |
|   1 |  NESTED LOOPS                    |                 |      1 |   3000 |   5400K|00:15:41.97 |    3009 |       |       |          |
|   2 |   TABLE ACCESS FULL              | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    3009 |       |       |          |
|   3 |   VIEW                           | VW_LAT_2D0B8FC8 |   3000 |      1 |   5400K|00:02:02.67 |       0 |       |       |          |
|   4 |    WINDOW SORT                   |                 |   3000 |      1 |   5400K|00:02:00.59 |       0 | 43008 | 43008 |38912  (0)|
|   5 |     VIEW                         |                 |   3000 |      1 |   5400K|00:01:58.78 |       0 |       |       |          |
|   6 |      CONNECT BY WITHOUT FILTERING|                 |   3000 |        |   5400K|00:01:48.53 |       0 |  2048 |  2048 | 2048  (0)|
|   7 |       FAST DUAL                  |                 |   3000 |      1 |   3000 |00:00:00.01 |       0 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------------------
```

### Notes on LAT\_QRY

This query is taken from [Splitting Strings: Proof!](https://stewashton.wordpress.com/2016/08/02/splitting-strings-proof/), and uses a v12.1 new feature, described with examples in [LATERAL Inline Views](https://oracle-base.com/articles/12c/lateral-inline-views-cross-apply-and-outer-apply-joins-12cr1). The new feature allows you to correlate an inline view directly without the convoluted Multiset code, and can also be used with the keywords CROSS APPLY instead of LATERAL. It's sometimes regarded as having peformance advantages, but in this context we will see that avoiding this correlation altogether is best for performance.

## Row-generator Query, Unhinted and Hinted (UNH\_QRY and RGN\_QRY)

### Unhinted Query

```sql
WITH row_gen AS (
        SELECT LEVEL rn FROM DUAL CONNECT BY LEVEL <= 
            (SELECT Max (Nvl (Length(list_col), 0) - Nvl (Length (Replace (list_col,'|')), 0) + 1)
               FROM delimited_lists)
)
SELECT d.id   id,
       Substr (d.list_col, Instr ('|' || d.list_col, '|', 1, r.rn), Instr (d.list_col || '|', '|', 1, r.rn) - Instr ('|' || d.list_col, '|', 1, r.rn)) token
  FROM delimited_lists d
  JOIN row_gen r
    ON r.rn <= Nvl (Length(d.list_col), 0) - Nvl (Length (Replace (d.list_col,'|')), 0) + 1
```
### Execution Plan
```
Plan hash value: 747926158

---------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                      | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                 |      1 |        |   5400K|00:01:55.35 |    2717K|       |       |          |
|   1 |  NESTED LOOPS                  |                 |      1 |    150 |   5400K|00:01:55.35 |    2717K|       |       |          |
|   2 |   VIEW                         |                 |      1 |      1 |   1800 |00:00:07.39 |    1509 |       |       |          |
|   3 |    CONNECT BY WITHOUT FILTERING|                 |      1 |        |   1800 |00:00:07.39 |    1509 |  2048 |  2048 | 2048  (0)|
|   4 |     FAST DUAL                  |                 |      1 |      1 |      1 |00:00:00.01 |       0 |       |       |          |
|   5 |     SORT AGGREGATE             |                 |      1 |      1 |      1 |00:00:00.06 |    1509 |       |       |          |
|   6 |      TABLE ACCESS FULL         | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |       |       |          |
|*  7 |   TABLE ACCESS FULL            | DELIMITED_LISTS |   1800 |    150 |   5400K|00:01:53.61 |    2716K|       |       |          |
---------------------------------------------------------------------------------------------------------------------------------------
```

### Execution Plan with Hint /\*+ leading (d) \*/

```
Plan hash value: 1241630378

----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                       | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |                 |      1 |        |   5400K|00:00:02.37 |    3018 |       |       |          |
|   1 |  MERGE JOIN                     |                 |      1 |    150 |   5400K|00:00:02.37 |    3018 |       |       |          |
|   2 |   SORT JOIN                     |                 |      1 |   3000 |   3000 |00:00:00.07 |    1509 |    11M|  1318K|   10M (0)|
|   3 |    TABLE ACCESS FULL            | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |       |       |          |
|*  4 |   SORT JOIN                     |                 |   3000 |      1 |   5400K|00:00:01.42 |    1509 |   160K|   160K|  142K (0)|
|   5 |    VIEW                         |                 |      1 |      1 |   1800 |00:00:07.37 |    1509 |       |       |          |
|   6 |     CONNECT BY WITHOUT FILTERING|                 |      1 |        |   1800 |00:00:07.37 |    1509 |  2048 |  2048 | 2048  (0)|
|   7 |      FAST DUAL                  |                 |      1 |      1 |      1 |00:00:00.01 |       0 |       |       |          |
|   8 |      SORT AGGREGATE             |                 |      1 |      1 |      1 |00:00:00.06 |    1509 |       |       |          |
|   9 |       TABLE ACCESS FULL         | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   4 - access(INTERNAL_FUNCTION("R"."RN")<=NVL(LENGTH("D"."LIST_COL"),0)-NVL(LENGTH(REPLACE("D"."LIST_COL",'|')),0)+1)
       filter(INTERNAL_FUNCTION("R"."RN")<=NVL(LENGTH("D"."LIST_COL"),0)-NVL(LENGTH(REPLACE("D"."LIST_COL",'|')),0)+1)
```

### Notes on UNH\_QRY and RGN\_QRY

I wrote the UNH\_QRY query in an attempt to avoid the convoluted Multiset approach of the 'classic' solution. The reason for the use of arrays and Multiset seems to be that, while we need to 'generate' multiple rows for each source row, the number of rows generated has to vary by source record and so the row-generating inline view computes the number of tokens for each record in its where clause.

The use of row-generating subqueries is quite common, but in other cases one often has a fixed number of rows to generate, as in data densification scenarios for example. It occurred to me that, although we don't know the number to generate, we do have an upper bound, dependent on the maximum number of characters, and we could generate that many in a subquery, then join only as many as are required to the source record.

This approach resulted in a simpler and more straightforward query, but it turned out in its initial form to be very slow. The execution plan above shows that the row generator is driving a nested loops join within which a full scan is performed on the table. The CBO is not designed to optimise this type of algorithmic query, so I added a leading hint to reverse the join order, and this resulted in much better performance. In fact, as we see later the hinted query outperforms the other CONNECT BY queries, including the v12.1 LAT\_QRY query at all data points considered.

## sys\_guid Query (GUI\_QRY)

```sql
WITH guid_cby AS (
  SELECT id, level rn, list_col,Instr ('|' || d.list_col, '|', 1, LEVEL) pos
    FROM delimited_lists d
  CONNECT BY prior id = id and prior sys_guid() is not null and
    LEVEL <= Length (d.list_col) - Nvl (Length (Replace (d.list_col, '|')), 0) + 1
)
SELECT id  id, 
       Substr (list_col, pos, Lead (pos, 1, 4000) OVER (partition by id ORDER BY pos) - pos - 1) token
  FROM guid_cby
```
### Execution Plan
```
Plan hash value: 240527573

-------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                      | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
-------------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |                 |      1 |        |   5400K|00:14:12.07 |   77412 |   2404K|   2404K|       |       |          |         |
|   1 |  WINDOW SORT                   |                 |      1 |   3000 |   5400K|00:14:12.07 |   77412 |   2404K|   2404K|    20G|    45M|  163M (0)|      18M|
|   2 |   VIEW                         |                 |      1 |   3000 |   5400K|00:04:07.47 |    1509 |      0 |      0 |       |       |          |         |
|*  3 |    CONNECT BY WITHOUT FILTERING|                 |      1 |        |   5400K|00:03:55.99 |    1509 |      0 |      0 |    12M|  1343K|   10M (0)|         |
|   4 |     TABLE ACCESS FULL          | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |      0 |      0 |       |       |          |         |
-------------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   3 - access("ID"=PRIOR NULL)

```

### Notes on GUI\_QRY

This query also generates rows using CONNECT BY, but differs from the others shown by integrating the row-generation code with the main rowset and avoiding a separate subquery against DUAL. This seems to be a more recent approach than the traditional Multiset solution. It uses a trick involving the system function sys\_guid() to avoid the 'connect by cycle' error that you would otherwise get, as explained in this OTN thread: [Reg : sys\_guid()](https://community.oracle.com/thread/2526535?start=0&tstart=0) .

Unfortunately, and despite its current popularity on OTN, it turns out to be even less efficient than the earlier approaches, by quite a margin.

## Regex Query (RGX\_QRY)

```
WITH row_gen AS (
        SELECT LEVEL rn FROM DUAL CONNECT BY LEVEL < 2000
)
SELECT d.id   id,
       RTrim (Regexp_Substr (d.list_col || '|', '(.*?)\|', 1, r.rn), '|') token
  FROM delimited_lists d
  JOIN row_gen r
    ON r.rn <= Nvl (Length(d.list_col), 0) - Nvl (Length (Replace (d.list_col,'|')), 0) + 1
```
### Execution Plan
```
Plan hash value: 1537360357

----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                       | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |                 |      1 |        |   5400K|00:00:03.35 |    1509 |       |       |          |
|   1 |  MERGE JOIN                     |                 |      1 |    150 |   5400K|00:00:03.35 |    1509 |       |       |          |
|   2 |   SORT JOIN                     |                 |      1 |   3000 |   3000 |00:00:00.07 |    1509 |    11M|  1318K|   10M (0)|
|   3 |    TABLE ACCESS FULL            | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |       |       |          |
|*  4 |   SORT JOIN                     |                 |   3000 |      1 |   5400K|00:00:01.75 |       0 |   160K|   160K|  142K (0)|
|   5 |    VIEW                         |                 |      1 |      1 |   1999 |00:00:00.01 |       0 |       |       |          |
|   6 |     CONNECT BY WITHOUT FILTERING|                 |      1 |        |   1999 |00:00:00.01 |       0 |  2048 |  2048 | 2048  (0)|
|   7 |      FAST DUAL                  |                 |      1 |      1 |      1 |00:00:00.01 |       0 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   4 - access(INTERNAL_FUNCTION("R"."RN")<=NVL(LENGTH("D"."LIST_COL"),0)-NVL(LENGTH(REPLACE("D"."LIST_COL",'|')),0)+1)
       filter(INTERNAL_FUNCTION("R"."RN")<=NVL(LENGTH("D"."LIST_COL"),0)-NVL(LENGTH(REPLACE("D"."LIST_COL",'|')),0)+1)
```

### Notes on RGX\_QRY

This query also generates rows using CONNECT BY, but differs from the others shown by using regular expressions to do the token parsing, which is simpler than the Substr/Instr approaches.

This seems to be quite popular, but it's well known that regular expression processing can be bad for performance, and so it proves here, with CPU time increasing quadratically with number of tokens.

## XML Query (XML\_QRY)

```
SELECT id   id,
       x2   token
  FROM delimited_lists, XMLTable(
    'if (contains($X2,"|")) then ora:tokenize($X2,"\|") else $X2'
  PASSING list_col AS x2
  COLUMNS x2 VARCHAR2(4000) PATH '.'
)
```
### Execution Plan
```
Plan hash value: 2423482301

----------------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name                  | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                       |      1 |        |   5400K|00:00:10.85 |    3009 |
|   1 |  NESTED LOOPS                      |                       |      1 |     24M|   5400K|00:00:10.85 |    3009 |
|   2 |   TABLE ACCESS FULL                | DELIMITED_LISTS       |      1 |   3000 |   3000 |00:00:00.01 |    3009 |
|   3 |   COLLECTION ITERATOR PICKLER FETCH| XQSEQUENCEFROMXMLTYPE |   3000 |   8168 |   5400K|00:00:03.49 |       0 |
----------------------------------------------------------------------------------------------------------------------
```

### Notes on XML\_QRY

This query using XMLTable is copied from the version used by Jayesh in the AskTom thread above.

## Model Query (MOD\_QRY)

```
SELECT id      id,
       token   token
  FROM delimited_lists
 MODEL
    PARTITION BY (id)
    DIMENSION BY (1 rn)
    MEASURES (CAST('' AS VARCHAR2(4000)) AS token, '|' || list_col || '|' list_col, 2 pos, 0 nxtpos, Length(list_col) + 2 len)
    RULES ITERATE (2000) UNTIL pos[1] > len[1] (
       nxtpos[1] = Instr (list_col[1], '|', pos[1], 1),
       token[iteration_number+1] = Substr (list_col[1], pos[1], nxtpos[1] - pos[1]),
       pos[1] = nxtpos[1] + 1
    )
```
### Execution Plan
```
Plan hash value: 1656081500

-------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation              | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT       |                 |      1 |        |   5400K|03:10:43.97 |    1509 |   2883 |   2883 |       |       |          |
|   1 |  SQL MODEL ORDERED FAST|                 |      1 |   3000 |   5400K|03:10:43.97 |    1509 |   2883 |   2883 |  2047M|   112M| 2844M (1)|
|   2 |   TABLE ACCESS FULL    | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |      0 |      0 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------------------------
```

### Notes on MOD\_QRY

I wrote this query using the Model clause, available since Oracle Database version 10, for this article.

The Model clause has something of a reputation for poor performance, and this was one of the slower methods, with CPU time increasing quadratically with number of tokens.

## Pipelined Function Query (PLF\_QRY)

### Pipelined Function Strings.Split

```sql
FUNCTION Split (p_string VARCHAR2, p_delim VARCHAR2) RETURN L1_chr_db_arr PIPELINED IS

  c_delim_len   CONSTANT SIMPLE_INTEGER := Length(p_delim);
  l_token_start          SIMPLE_INTEGER := 1;
  l_next_delim           SIMPLE_INTEGER := Instr (p_string, p_delim, l_token_start, 1);

BEGIN

  WHILE l_next_delim > 0 LOOP
    PIPE ROW (Substr (p_string, l_token_start, l_next_delim - l_token_start));
    l_token_start := l_next_delim + c_delim_len;
    l_next_delim := Instr (p_string || p_delim, p_delim, l_token_start, 1);
  END LOOP;

END Split;
```

### Query Using Pipelined Function

```sql
SELECT d.id                id,
       s.COLUMN_VALUE      token
  FROM delimited_lists d
 CROSS JOIN TABLE (Strings.Split(d.list_col, '|')) s
```
### Execution Plan
```
Plan hash value: 2608399241

----------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                 |      1 |        |   5400K|00:00:03.08 |    3009 |
|   1 |  NESTED LOOPS                      |                 |      1 |     24M|   5400K|00:00:03.08 |    3009 |
|   2 |   TABLE ACCESS FULL                | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    3009 |
|   3 |   COLLECTION ITERATOR PICKLER FETCH| SPLIT           |   3000 |   8168 |   5400K|00:00:01.87 |       0 |
----------------------------------------------------------------------------------------------------------------
```

### Notes on PLF\_QRY

This is a fairly well known approach to the problem that involves doing the string splitting within a pipelined database function that is passed the delimited string as a parameter. I wrote my own version for this article, taking care to make only one call to each of the oracle functions Instr and Substr within a loop over the tokens.

The results confirm that it is in fact the fastest approach over all data points considered, and CPU time increased approximately linearly with number of tokens.

## With Function v12.1 Query (WFN\_QRY)

```sql
WITH FUNCTION Split (p_string VARCHAR2, p_delim VARCHAR2) RETURN L1_chr_db_arr IS
  c_delim_len   CONSTANT SIMPLE_INTEGER := Length(p_delim);
  l_token_start          SIMPLE_INTEGER := 1;
  l_next_delim           SIMPLE_INTEGER := Instr (p_string, p_delim, l_token_start, 1);
  l_ret_arr              L1_chr_db_arr := L1_chr_db_arr();

BEGIN

  WHILE l_next_delim > 0 LOOP
    l_ret_arr.EXTEND;
    l_ret_arr(l_ret_arr.COUNT) := Substr (p_string, l_token_start, l_next_delim - l_token_start);
    l_token_start := l_next_delim + c_delim_len;
    l_next_delim := Instr (p_string || p_delim, p_delim, l_token_start, 1);
  END LOOP;
  RETURN l_ret_arr;

END Split;
SELECT d.id                id,
       s.COLUMN_VALUE      token
  FROM delimited_lists d
 CROSS JOIN TABLE (Split(d.list_col, '|')) s
```
### Execution Plan
```
Plan hash value: 2608399241

----------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                 |      1 |        |   5400K|00:00:17.56 |    3009 |
|   1 |  NESTED LOOPS                      |                 |      1 |     24M|   5400K|00:00:17.56 |    3009 |
|   2 |   TABLE ACCESS FULL                | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    3009 |
|   3 |   COLLECTION ITERATOR PICKLER FETCH| SPLIT           |   3000 |   8168 |   5400K|00:00:02.57 |       0 |
----------------------------------------------------------------------------------------------------------------

```

### Notes on WFN\_QRY

Oracle introduced the ability to include a PL/SQL function definition directly in a query in version 12.1. I converted my pipelined function into a function within a query, returning an array of character strings.

As we would expect from the results of the similar pipelined function approach, this also turns out to be a very efficient solution. However, it may be surprising to many that it is significantly slower (20-30%) than using the separate database function, given the prominence that is usually assigned to context-switching.

## Recursive Subquery Factor Query (RSF\_QRY)

```sql
WITH rsf (id, token, nxtpos, nxtpos2, list_col, len, iter) AS
(
SELECT id,
       Substr (list_col, 1, Instr (list_col || '|', '|', 1, 1) - 1),
       Instr (list_col || '|', '|', 1, 1) + 1,
       Instr (list_col || '|', '|', 1, 2),
       list_col || '|',
       Length (list_col) + 1,
       1
  FROM delimited_lists
UNION ALL
SELECT id,
       Substr (list_col, nxtpos, nxtpos2 - nxtpos),
       nxtpos2 + 1,
       Instr (list_col, '|', nxtpos2 + 1, 1),
       list_col,
       len,
       iter + 1
  FROM rsf r
 WHERE nxtpos <= len
)
SELECT
       id       id,
       token    token
  FROM rsf
```
### Execution Plan
```
Plan hash value: 2159872273

--------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                                 | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
--------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                          |                 |      1 |        |   5400K|03:27:56.79 |    1434M|       |       |          |
|   1 |  VIEW                                     |                 |      1 |   6000 |   5400K|03:27:56.79 |    1434M|       |       |          |
|   2 |   UNION ALL (RECURSIVE WITH) BREADTH FIRST|                 |      1 |        |   5400K|03:27:45.02 |    1434M|    12M|  1343K|   10M (0)|
|   3 |    TABLE ACCESS FULL                      | DELIMITED_LISTS |      1 |   3000 |   3000 |00:00:00.01 |    1509 |       |       |          |
|   4 |    RECURSIVE WITH PUMP                    |                 |   1800 |        |   5397K|00:00:04.48 |       0 |       |       |          |
--------------------------------------------------------------------------------------------------------------------------------------------------
```

### Notes on RSF\_QRY

Oracle introduced recursive subquery factoring in v11.2, as an Ansi standard approach to SQL recursion, and with greater power than the older CONNECT BY recursion. I wrote the query for this article.

The query turned out to be surprisingly simple in structure, but for large numbers of tokens it was by far the slowest, and CPU time increased quadratically with number of tokens.

## Match Recognize Query (RMR\_QRY)

```sql
WITH row_gen AS (
        SELECT LEVEL rn FROM DUAL CONNECT BY LEVEL <= 4000
), char_streams AS (
SELECT d.id, r.rn, Substr (d.list_col || '|', r.rn, 1) chr
  FROM delimited_lists d
  JOIN row_gen r
    ON r.rn <= Nvl (Length(d.list_col), 0) + 2
), chars_grouped AS (
SELECT *
  FROM char_streams
 MATCH_RECOGNIZE (
   PARTITION BY id
   ORDER BY rn
   MEASURES chr mchr,
            FINAL COUNT(*) n_chrs,
            MATCH_NUMBER() mno
      ALL ROWS PER MATCH
  PATTERN (c*? d)
   DEFINE d AS d.chr = '|'
  ) m
)
SELECT id   id, 
       RTrim (Listagg (chr, '') WITHIN GROUP (ORDER BY rn), '|') token
  FROM chars_grouped
GROUP BY id, mno
```
### Execution Plan
```
Plan hash value: 2782416907

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                          | Name            | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  | Writes |  OMem |  1Mem | Used-Mem | Used-Tmp|
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                   |                 |      1 |        |   5400K|00:00:52.53 |    6036K|    103K|    103K|       |       |          |         |
|   1 |  SORT GROUP BY                     |                 |      1 |    150 |   5400K|00:00:52.53 |    6036K|    103K|    103K|   330M|  6949K|   97M (1)|     395K|
|   2 |   VIEW                             |                 |      1 |    150 |     10M|00:00:53.55 |    6036K|  52539 |  52539 |       |       |          |         |
|   3 |    MATCH RECOGNIZE SORT            |                 |      1 |    150 |     10M|00:00:51.47 |    6036K|  52539 |  52539 |   231M|  5084K|  163M (1)|         |
|   4 |     VIEW                           |                 |      1 |    150 |     10M|00:00:18.75 |    6036K|      0 |      0 |       |       |          |         |
|   5 |      NESTED LOOPS                  |                 |      1 |    150 |     10M|00:00:11.76 |    6036K|      0 |      0 |       |       |          |         |
|   6 |       VIEW                         |                 |      1 |      1 |   4000 |00:00:00.01 |       0 |      0 |      0 |       |       |          |         |
|   7 |        CONNECT BY WITHOUT FILTERING|                 |      1 |        |   4000 |00:00:00.01 |       0 |      0 |      0 |  2048 |  2048 | 2048  (0)|         |
|   8 |         FAST DUAL                  |                 |      1 |      1 |      1 |00:00:00.01 |       0 |      0 |      0 |       |       |          |         |
|*  9 |       TABLE ACCESS FULL            | DELIMITED_LISTS |   4000 |    150 |     10M|00:00:10.12 |    6036K|      0 |      0 |       |       |          |         |
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   9 - filter("R"."RN"<=NVL(LENGTH("D"."LIST_COL"),0)+2)
```

### Notes on RMR\_QRY

Oracle introduced Match\_Recognize in v12.1, as a mechanism for pattern matching along the lines of regular expressions for strings, but for matching patterns across records. I wrote the query for this article, converting each character in the input strings into a separate record to allow for its use.

This approach might seems somewhat convoluted, and one might expect it to be correspondingly slow. As it turns out though, for most datasets it is faster than many of the other methods, the ones with very long tokens being the exception, and CPU time increased linearly with both number of tokens and number of characters per token. It is notable that, apart from the exception mentioned, it outperformed the regular expression query.

# Performance Results

In the tables below, we will include some expressions used in [Dimensional Benchmarking of Bracket Parsing SQL](http://aprogrammerwrites.eu/?p=1913):

- LRTB = Ratio to best time at high data point
- RTP\_L = Linear ratio as defined in the link above, averaged over successive data points
- RTP\_Q = Quadratic ratio as defined in the link above, averaged over successive data points

The CPU times are listed but elapsed times are much the same. Each table has columns in order of increasing last CPU time by query.

## Depth fixed, high; width range low: d=18, w=(50,100,150,200)
<img src="/migrated_images/2017/03/D18-1.png" alt="Depth High, Best" title="Depth High, Best" />

<img src="/migrated_images/2017/03/D18-2.png" alt="Depth High, Worst" title="Depth High, Worst" />

## Depth fixed, low; width range high: d=1, w=(450,900,1350,1800)
<img src="/migrated_images/2017/03/D1-1.png" alt="Depth Low, Best" title="Depth Low, Best" />

<img src="/migrated_images/2017/03/D1-2.png" alt="Depth Low, Worst" title="Depth Low, Worst" />

## Width fixed, low; depth range high: w=5, d=(195,390,585,780)
<img src="/migrated_images/2017/03/W5-1.png" alt="Width Low, Best" title="Width Low, Best" />

<img src="/migrated_images/2017/03/W5-2.png" alt="Width Low, Middle" title="Width Low, Middle" />

<img src="/migrated_images/2017/03/W5-3.png" alt="Width Low, Worst" title="Width Low, Worst" />

## Width fixed, high; depth range low: w=150, d=(6,12,18,24)
<img src="/migrated_images/2017/03/W150-1.png" alt="Width High, Best" title="Width High, Best" />

<img src="/migrated_images/2017/03/W150-2.png" alt="Width High, Worst" title="Width High, Worst" />

# A Note on the Row Generation by Connect By Results

It is interesting to observe that the 'classical' mechanism for row-generation in string-splitting and similar scenarios turns out to be much slower than a simpler approach that removes the correlation of the row-generating subquery. This 'classical' mechanism has been proposed on Oracle forums over many years, while a simpler and faster approach seems to have gone undiscovered. The reason for its performance deficit is simply that running a Connect By query for every master row is unsurprisingly inefficient. The Use of the v12.1 LATERAL correlation syntax simplifies but doesn't improve performance by much.

The more recent approach to Connect By row-generation is to use the sys\_guid 'trick' to embed the Connect By in the main query rather than in a correlated subquery, and this has become very popular on forums such as OTN. As we have seen, although simpler, this is even worse for performance: Turning the whole query into a tree-walk isn't good for performance either. It's better to isolate the tree-walk, execute it once, and then just join its result set as in RGN\_QRY.

# Conclusions

- The database pipelined function solution (PLF\_QRY) is generally the fastest across all data points
- Using the v12.1 feature of a PL/SQL function embedded within the query is almost always next best, although slower by up to about a third; its being slower than a database function may surprise some
- Times generally increased uniformly with numbers of tokens, usually either linearly or quadratically
- Times did not seem to increase so uniformly with token size, except for XML (XML\_QRY), Match\_Recognize (RMR\_QRY) and regular expression (RGX\_QRY)
- For larger numbers of tokens, three methods all showed quadratic variation and were very inefficient: Model (MOD\_QRY), regular expression (RGX\_QRY), and recursive subquery factors (RSF\_QRY)
- We have highlighted two inefficient but widespread approaches to row-generation by Connect By SQL, and pointed out a better method

These conclusions are based on the string-splitting problem considered, but no doubt would apply to other scenarios involving requirements to split rows into multiple rows based on some form of string-parsing.

The output log is in [GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/dimensional-benchmarking-of-string-splitting-sql).

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
