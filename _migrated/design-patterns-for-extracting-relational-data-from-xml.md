---
layout: post
title: "Design Patterns for Extracting Relational Data from XML"
date: 2013-04-21
migrated: true
categories: 
  - "data-model"
  - "object"
  - "oracle"
  - "sql"
  - "xml"
tags: 
  - "oracle"
  - "sql"
  - "xml-2"
---

I recently replicated a very complicated Informatica data load from XML files into two Oracle staging tables in a much simpler way using just three SQL insert statements. I was able to load over 13,000 XML files into the tables in around 3 minutes on a windows PC running Oracle 11.2 XE, populating about 2 million records. This article will focus on the general design patterns involved in selecting relational data from XML using SQL in Oracle 11.2.

## XML Selection Scenarios

The hierarchical structure of XML files means that the data may be stored at various different levels, but a relational SELECT statement returns data in a flat format. One way of converting between hierarchical and flat formats is to have separate SELECT statements for each level of data, and have any additional post-processing performed after inserting into staging tables. However, it is also possible to flatten the hierarchy within a single SELECT query clause in the case of a strict linear hierarchy.

It is instructive to consider the general case where any number of entities can occur within a linear hierarchy. If we understand how to deal with the case of two entities in a master-detail relationship, along with global values, then we will know how to deal with the general case, and so I will construct just such a test example.

## XML Test Data

Here is the test XML file
```
<elem-0-body>
    <elem-1-global attr-1-global="id_1_global">
        <field-1-global>val_global</field-1-global>
    </elem-1-global>
    <list-1-master>
        <elem-1-master>
            <field-1-master>val_master_m1</field-1-master>
            <list-2-detail>
                  <elem-2-detail>
                       <field-2-detail>val_detail_m1_d1</field-2-detail>
                  </elem-2-detail>
                  <elem-2-detail>
                       <field-2-detail>val_detail_m1_d2</field-2-detail>
                  </elem-2-detail>
            </list-2-detail>
        </elem-1-master>
        <elem-1-master>
            <field-1-master>val_master_m2</field-1-master>
            <list-2-detail>
                  <elem-2-detail>
                       <field-2-detail>val_detail_m2_d1</field-2-detail>
                  </elem-2-detail>
                  <elem-2-detail>
                       <field-2-detail>val_detail_m2_d2</field-2-detail>
                  </elem-2-detail>
            </list-2-detail>
        </elem-1-master>
        <elem-1-master>
            <field-1-master>val_master_m3</field-1-master>
        </elem-1-master>
    </list-1-master>
</elem-0-body>
```

The file represents data in a master-detail relationship, plus data that are global to the file. As can be seen, there is a global entity, _elem-1-global_, that occurs exactly once, a master entity, _elem-1-master_, that may occur multiple times and that contains a child entity, _elem-2-detail_, that may occur multiple times. Each entity has been assigned a single sample field. One entity has an XML attribute.

There are three master records, having two, two and zero detail records respectively. We will filter out one of the second master's detail records.

In an article from January 2012, [Data Modelling of XML SOAP Documents](https://brenpatf.github.io/migrated/data-modelling-of-xml-soap-documents/ "Data Modelling of XML SOAP Documents"), I introduced my own diagram notation for hierarchical data structures, and re-use it below, showing the links to entities in relational form. I have added rounded corners to distinguish attributes from fields.

<img src="/migrated_images/2013/04/XMLSQL.jpg" alt="XMLSQL" title="XMLSQL" />

## SQL

### Reading the XML File

```sql
CREATE TABLE xml_design_pattern (
       xml_field   XMLTYPE) 
       XMLTYPE COLUMN xml_field STORE AS BINARY XML
/
INSERT INTO xml_design_pattern VALUES (
       XMLTYPE (BFilename ('BRENDAN_IN_DIR', 'DESIGN_PATTERN.xml'),                      
                           NLS_Charset_id ('AL32UTF8'))
)
/
```

### Unfiltered, Outer Join on Detail

#### SQL
```sql
SELECT /*+ GATHER_PLAN_STATISTICS XMLOJA */ 
       x1.global_id, x1.global_val, x2.master, x3.detail
  FROM xml_design_pattern t,
     XMLTable ('/elem-0-body'
                   PASSING t.xml_field
                   COLUMNS global_id           VARCHAR2(100) PATH 'elem-1-global/@attr-1-global',
                           global_val          VARCHAR2(100) PATH 'elem-1-global/field-1-global',
                           l2_xml              XMLTYPE       PATH 'list-1-master') x1,
     XMLTable ('/list-1-master/elem-1-master'
                   PASSING x1.l2_xml
                   COLUMNS master              VARCHAR2(100) PATH 'field-1-master',
                           l3_xml              XMLTYPE       PATH 'list-2-detail') x2,
     XMLTable ('/list-2-detail/elem-2-detail'
                   PASSING x2.l3_xml
                   COLUMNS detail              VARCHAR2(100) PATH 'field-2-detail') (+) x3
ORDER BY 1, 2, 3, 4
```

#### Output
```
GLOBAL_ID       GLOBAL_VAL      MASTER          DETAIL
--------------- --------------- --------------- --------------------
id_1_global     val_global      val_master_m1
id_1_global     val_global      val_master_m2
id_1_global     val_global      val_master_m3
```

The result shows that none of the detail records have been joined, while the expected result would be that all detail records would be joined with null detail only for m3.

### Filtered, Outer Join on Detail

#### SQL
```sql
SELECT x1.global_id, x1.global_val, x2.master, x3.detail
  FROM xml_design_pattern t,
     XMLTable ('/elem-0-body'
                   PASSING t.xml_field
                   COLUMNS global_id           VARCHAR2(100) PATH 'elem-1-global/@attr-1-global',
                           global_val          VARCHAR2(100) PATH 'elem-1-global/field-1-global',
                           l2_xml              XMLTYPE       PATH 'list-1-master') x1,
     XMLTable ('/list-1-master/elem-1-master'
                   PASSING x1.l2_xml
                   COLUMNS master              VARCHAR2(100) PATH 'field-1-master',
                           l3_xml              XMLTYPE       PATH 'list-2-detail') x2,
     XMLTable ('/list-2-detail/elem-2-detail[field-2-detail != "val_detail_m2_d1"]'
                   PASSING x2.l3_xml
                   COLUMNS detail              VARCHAR2(100) PATH 'field-2-detail') (+) x3
ORDER BY 1, 2, 3, 4
```

#### Output
```
no rows selected
```

The result shows that no records have been returned, while the expected result would be that all master records would be returned, with detail records joined for all except the filtered out detail, with null detail only for m3.

### Unfiltered, Outer Joins on all

#### SQL
```sql
SELECT x1.global_id, x1.global_val, x2.master, x3.detail
  FROM xml_design_pattern t,
     XMLTable ('/elem-0-body'
                   PASSING t.xml_field
                   COLUMNS global_id           VARCHAR2(100) PATH 'elem-1-global/@attr-1-global',
                           global_val          VARCHAR2(100) PATH 'elem-1-global/field-1-global',
                           l2_xml              XMLTYPE       PATH 'list-1-master') (+) x1,
    XMLTable ('/list-1-master/elem-1-master'
                   PASSING x1.l2_xml
                   COLUMNS master              VARCHAR2(100) PATH 'field-1-master',
                           l3_xml              XMLTYPE       PATH 'list-2-detail') (+) x2,
     XMLTable ('/list-2-detail/elem-2-detail'
                   PASSING x2.l3_xml
                   COLUMNS detail              VARCHAR2(100) PATH 'field-2-detail') (+) x3
ORDER BY 1, 2, 3, 4
```

#### Output
```
GLOBAL_ID       GLOBAL_VAL      MASTER          DETAIL
--------------- --------------- --------------- --------------------
id_1_global     val_global      val_master_m1   val_detail_m1_d1
id_1_global     val_global      val_master_m1   val_detail_m1_d2
id_1_global     val_global      val_master_m2   val_detail_m2_d1
id_1_global     val_global      val_master_m2   val_detail_m2_d2
id_1_global     val_global      val_master_m3
```

The result shows that all records are returned, as expected.

### Unfiltered, Inner Joins

The attached file shows that, without the outer joins, records are returned only for the first two master records that have detail records, as expected.

### Filtered, Outer Joins on all

#### SQL
```sql
SELECT x1.global_id, x1.global_val, x2.master, x3.detail
  FROM xml_design_pattern t,
     XMLTable ('/elem-0-body'
                   PASSING t.xml_field
                   COLUMNS global_id           VARCHAR2(100) PATH 'elem-1-global/@attr-1-global',
                           global_val          VARCHAR2(100) PATH 'elem-1-global/field-1-global',
                           l2_xml              XMLTYPE       PATH 'list-1-master') (+) x1,
     XMLTable ('/list-1-master/elem-1-master'
                   PASSING x1.l2_xml
                   COLUMNS master              VARCHAR2(100) PATH 'field-1-master',
                           l3_xml              XMLTYPE       PATH 'list-2-detail') (+) x2,
     XMLTable ('/list-2-detail/elem-2-detail[field-2-detail != "val_detail_m2_d1"]'
                   PASSING x2.l3_xml
                   COLUMNS detail              VARCHAR2(100) PATH 'field-2-detail') (+) x3
ORDER BY 1, 2, 3, 4
```

#### Output
```
GLOBAL_ID       GLOBAL_VAL      MASTER          DETAIL
--------------- --------------- --------------- --------------------
id_1_global     val_global      val_master_m1   val_detail_m1_d1
id_1_global     val_global      val_master_m1   val_detail_m1_d2
id_1_global     val_global      val_master_m2   val_detail_m2_d2
id_1_global     val_global      val_master_m3
```

The result shows that, with all joins outer joins, all records are returned except for the filtered-out detail record, as expected.

### Filtered, Ansi Outer Joins on all

#### SQL
```sql
SELECT x1.global_id, x1.global_val, x2.master, x3.detail
  FROM xml_design_pattern t
    LEFT JOIN
     XMLTable ('/elem-0-body'
                   PASSING t.xml_field
                   COLUMNS global_id           VARCHAR2(100) PATH 'elem-1-global/@attr-1-global',
                           global_val          VARCHAR2(100) PATH 'elem-1-global/field-1-global',
                           l2_xml              XMLTYPE       PATH 'list-1-master') x1 ON 1=1
    LEFT JOIN
     XMLTable ('/list-1-master/elem-1-master'
                   PASSING x1.l2_xml
                   COLUMNS master              VARCHAR2(100) PATH 'field-1-master',
                           l3_xml              XMLTYPE       PATH 'list-2-detail') x2 ON 1=1
    LEFT JOIN
     XMLTable ('/list-2-detail/elem-2-detail[field-2-detail != "val_detail_m2_d1"]'
                   PASSING x2.l3_xml
                   COLUMNS detail              VARCHAR2(100) PATH 'field-2-detail') x3 ON 1=1
ORDER BY 1, 2, 3, 4
```

Output: The result in the attached file shows that, with all joins Ansi outer joins, all records are returned except for the filtered-out detail record, as expected.

## Notes on SQL

### XMLTable and XPath Syntax

Thh XMLTable function represents a rowset retrieved from an XML fragment. The first string within the call represents an XPath expression for the root element defining the rowset in the XML fragment passed. The number of occurrences in the fragment is the rowset cardinality.

The PASSING clause passes in the XMLTYPE field from a prior table or XMLTable instance from which the XML fragment is extracted. For a detail entity, the field will be passed from a master record in a a prior instance, and the instance order matters.

The COLUMNS clause defines the columns that can be included in the select list, and specifies a field for each column by an XPath expression. The expression is relative to the XMLTable root and must specify a single element instance.

### XQuery Error (ORA-19279)

A common error in development is: _'ORA-19279: XPTY0004 - XQuery dynamic type mismatch: expected singleton sequence - got multi-item sequence'_ The error occurs when the XPath expression for a column selects more than one instance.

### Filtering

Filtering of the XNL elements is effected using standard XPath notation, with conditions placed in square brackets after the element name.

### Fields and Attributes

Elements may contain attributes, which are denoted in standard XML syntax by preceding the attribute name with '@'. Further details on XML syntax for Oracle, including additional features such as namespaces, can be found in [Oracle XML DB Developer's Guide](http://docs.oracle.com/cd/E11882_01/appdev.112/e23094/toc.htm)

### Ansi and Oracle Join Syntaxes

Oracle introduced the Ansi standard join syntax in version 9, and it is generally to be preferred to its older proprietary syntax. However, this is debatable in the special case of SQL incorporating XMLTable because joining occurs in a non-standard way within the PASSING and other clauses of XMLTable, and not in the mandatory (except for cross joins) ON clause. In my examples, I have had to join ON 1=1 as a work-around.

### Outer Joins

The examples show that outer-joining only the detail table does not work correctly. This appears to be a bug in Oracle 11.2. We have worked around it by outer-joining all tables.

Also note that the outer-join (+) needs to be after the table instance, unlike in standard SQL, where it goes after the join columns.

### XMLTYPE Storage Clause

The XMLTYPE column in the table used to load the XML file has a storage clause that can specify either CLOB or BINARY XML (and maybe others) as the storage mode. It is vital for performance to choose BINARY XML if the table is going to be queried using XMLTable.

### XPlan Statistics

I included calls to DBMS\_XPlan in some of the queries in the attached files, and it can be seen that the CBO cardinalities are extremely inaccurate. This is perhaps not surprising as there is only one record in the table, which is exploded within the SQL, and the CBO is obviously not designed to optimise this. 

[SQL XML Design Patterns Files, in GitHub container](https://github.com/BrenPatF/wp_ghp_migration/tree/master/design-patterns-for-extracting-relational-data-from-xml)
