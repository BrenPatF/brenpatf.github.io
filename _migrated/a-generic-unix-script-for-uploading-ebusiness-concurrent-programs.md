---
layout: post
title: "A Generic Unix Script for Uploading eBusiness Concurrent Programs"
date: 2014-09-28
migrated: true
categories: 
  - "design"
  - "ebusiness-r12"
  - "qsd"
  - "reports"
  - "xml-publisher"
tags: 
  - "fndload"
  - "ksh"
  - "oracle"
  - "plsql"
  - "qsd"
  - "reports"
  - "sql"
  - "unix"
  - "xdoloader"
  - "xml-publisher"
---

I have posted a couple of articles recently on XML Publisher report development within Oracle eBusiness applications ([A Design Pattern for Oracle eBusiness Audit Trail Reports with XML Publisher](https://brenpatf.github.io/migrated/a-design-pattern-for-oracle-ebusiness-audit-trail-reports-with-xml-publisher) and [Design Patterns for Database Reports with XML Publisher and Email Bursting](https://brenpatf.github.io/migrated/a-design-pattern-for-oracle-ebusiness-audit-trail-reports-with-xml-publisher)). These reports are of one of several types of batch program, or _concurrent program_ that can be defined within Oracle eBusiness.

Oracle eBusiness uses a number of metadata tables to store information on the programs, and in release 11.5 Oracle introduced a Unix utility called FNDLOAD to download the metadata to formatted text files to allow them to be uploaded via the same utility into downstream environments. At that time batch reports were generally developed in the Oracle Reports tool and typically there might only be two text files (known as _LDT_ files after their extension), for the program and for associated _value sets_, and maybe one for _request groups_ (which control access to the reports). The executable report file, the _RDF_, would just be copied to the target environment server directory. I wrote a set of wrapper scripts for the utility to streamline its use and to deal with a number of issues, including handling of audit fields, with a structure consisting of a separate pair of scripts for download and upload of each type of LDT file. I published these on Scribd in July 2009.

In working more recently with Oracle's successor reporting tool, XML Publisher, I found that the number of objects involved in installation has increased substantially. As well as the LDT files there are also now XML configuration files and RTF (usually) templates, uploaded via a Java utility, XDOLoader. Installation also involves at least one PL/SQL package. For example the email version of my model reports (see the second link above) had 11 configuration files. For this reason, I decided to create a single script that would copy, upload and install all objects required for a concurrent program of any type, and I describe the script in this article.

Here is my original Scribd article, describing the issues mentioned, and with the original individual upload and download scripts.

<iframe class="scribd_iframe_embed" title="Oracle Applications FNDLOAD Unix Scripts" src="https://www.scribd.com/embeds/17029415/content?start_page=1&view_mode=scroll&access_key=key-1c3o2jldmu8zf33gdm93" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View Oracle Applications FNDLOAD Unix Scripts on Scribd" href="https://www.scribd.com/document/17029415/Oracle-Applications-FNDLOAD-Unix-Scripts#from_embed" style="color: #098642; text-decoration: underline;"> Oracle Applications FNDLOAD Unix Scripts </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

The new script is here, in a GitHub container project subfolder, [XX\_Install\_XMLCP.ksh](https://github.com/BrenPatF/wp_ghp_migration/tree/master/a-generic-unix-script-for-uploading-ebusiness-concurrent-programs), and here is the MD120 from the model article that uses it. 

<iframe class="scribd_iframe_embed" title="MD120 - XX Example XML CP (Email)" src="https://www.scribd.com/embeds/241235110/content?start_page=1&view_mode=scroll&access_key=key-SFfYhJzWyXm9RWhuVLp7" tabindex="0" data-auto-height="true" data-aspect-ratio="0.7068965517241379" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View MD120 - XX Example XML CP (Email) on Scribd" href="https://www.scribd.com/document/241235110/MD120-XX-Example-XML-CP-Email#from_embed" style="color: #098642; text-decoration: underline;"> MD120 - XX Example XML CP (Email) </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

## Directory Structure

The script operates on an input TAR file containing all the necessary files. The TAR file is called _prog_.tar after the program short name _prog_, and contains a directory also called _prog_ with all the installation files. The installer uses relative paths and assumes that the TAR file, with the installer, is placed in a directory below the custom top directory, say $XX\_TOP/rel. All loader files are copied to $XX\_TOP/import and SQL files to $XX\_TOP/install/sql.

After installation, a new directory will remain with all the installation files for reference, $XX\_TOP/rel/_prog_.

## Program Structure

The internal call structure of the Unix script is shown below.

<img src="/migrated_images/2014/09/Install-CP-CSD.jpg" alt="Install CP - CSD" title="Install CP - CSD" width="800" />

The script operates on an input TAR file containing all the necessary files.

After extracting the TAR file, the script has local subroutines that can be divided into four categories, as above:

1. Preliminaries - parameter processing, file moving and validation
2. FNDLOAD uploads - uploading the LDT files
3. XDOLoader uploads - uploading the data and any bursting XML files, and all layout templates present
4. SQL installations - installing any SQL script present, and the package spec and body files

The following table gives a few notes on the main program and preliminary subroutines. 

### Preliminaries Summary

<img src="/migrated_images/2014/09/Prelims-Table.png" alt="Prelims Table" title="Prelims Table" />

The following table gives a few notes on the upload and SQL subroutines.

### Upload and SQL Summary

<img src="/migrated_images/2014/09/Install-Uploads-Table.png" alt="Install Uploads Table" title="Install Uploads Table" />

The upload subroutines all have SQL queries for verification, and here is a sample log file from the script, in a GitHub container project subfolder, [XX\_ERPXMLCP\_EM.log](https://github.com/BrenPatF/wp_ghp_migration/tree/master/a-generic-unix-script-for-uploading-ebusiness-concurrent-programs)

The remainder of the article lists the queries with diagrams and examples of output.

## Upload Subroutines

### upload\_ag

#### Validation Query

```sql
SELECT app_g.application_short_name "App G", fag.group_name "Group",
      fag.description "Description",
      app_t.application_short_name "App T", ftb.table_name "Table",
      fcl.column_name "Column"
  FROM fnd_audit_groups fag
  JOIN fnd_application app_g
    ON app_g.application_id = fag.application_id
  JOIN fnd_audit_tables fat
    ON fat.audit_group_app_id = fag.application_id
   AND fat.audit_group_id = fag.audit_group_id
  JOIN fnd_application app_t
    ON app_t.application_id = fat.table_app_id
  JOIN fnd_tables ftb
    ON ftb.application_id = fat.table_app_id
   AND ftb.table_id = fat.table_id
  JOIN fnd_audit_columns fac
    ON fac.table_app_id = fat.table_app_id
   AND fac.table_id = fat.table_id
  JOIN fnd_columns fcl
    ON fcl.application_id = fac.table_app_id
   AND fcl.table_id = fac.table_id
   AND fcl.column_id = fac.column_id
 WHERE fag.last_update_date     = To_Date ('$sysdate', 'YYYY/MM/DD')
   AND fac.schema_id            = 900
ORDER BY app_g.application_short_name, fag.group_name, 
         app_t.application_short_name, ftb.table_name,
         fcl.column_name;
```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-AG-1.jpg" alt="Install CP - AG - 1" title="Install CP - AG - 1" width="400" />

#### Example Output

No example available here, but the headings are: "App G", "Group", "Description", "App T", "Table", "Column"

### upload\_ms

#### Validation Query

```sql
SELECT mes.message_name "Name", mes.message_text "Text"
  FROM fnd_new_messages mes
 WHERE mes.last_update_date     = To_Date ('$sysdate', 'YYYY/MM/DD')
 ORDER BY 1;

```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-Message.jpg" alt="Install CP - Message" title="Install CP - Message" width="400" />

#### Example Output

```
Name                     Text
--------------------- ---------------------------------------------
XX_ERPXMLCP_EM_NONXML Non-XML Concurrent Program &PROGAPP
```

### upload\_vs

#### Validation Query

```sql
SELECT fvs.flex_value_set_name "Value Set", Count(fvl.flex_value_set_id) "Values"
  FROM fnd_flex_value_sets fvs, fnd_flex_values fvl
 WHERE fvs.last_update_date     = To_Date ('$sysdate', 'YYYY/MM/DD')
   AND fvl.flex_value_set_id(+) = fvs.flex_value_set_id
 GROUP BY fvs.flex_value_set_name;
```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-VS.jpg" alt="Install CP - VS" title="Install CP - VS" width="400" />

#### Example Output

```
Value Set             Values
--------------------- ------
XX_PROGS                   0
XX_APPNAME_ID              0
```

### upload\_cp

#### Validation Query

```sql
SELECT prg.user_concurrent_program_name || ': ' || prg.concurrent_program_name "Program", fcu.column_seq_num || ': ' || fcu.end_user_column_name "Parameter"
  FROM fnd_concurrent_programs_vl               prg
  LEFT JOIN fnd_descr_flex_column_usages      fcu
    ON fcu.descriptive_flexfield_name         = '\$SRS\$.' || prg.concurrent_program_name
   AND fcu.descriptive_flex_context_code      = 'Global Data Elements'
 WHERE prg.concurrent_program_name              = '$cp_name'
 ORDER BY 1, 2;
```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-CP.jpg" alt="Install CP - CP" title="Install CP - CP" width="400" />

#### Example Output

```
Program                                    Parameter
------------------------------------------ ------------------------
XX Example XML CP (Email): XX_ERPXMLCP_EM  100: Cc Email
                                           10: Application
                                           20: From Program
                                           30: To Program
                                           40: From Date
                                           50: To Date
                                           60: From Parameter Count
                                           70: To Parameter Count
                                           80: Override Email
                                           90: From Email
```

### upload\_rga

#### Validation Query

```sql
SELECT rgp.request_group_name "Request Group",
       app.application_short_name "App"
  FROM fnd_concurrent_programs          cpr
  JOIN fnd_request_group_units          rgu
    ON rgu.unit_application_id          = cpr.application_id
   AND rgu.request_unit_id              = cpr.concurrent_program_id
  JOIN fnd_request_groups               rgp
    ON rgp.application_id               = rgu.application_id
   AND rgp.request_group_id             = rgu.request_group_id
  JOIN fnd_application                  app
    ON app.application_id               = rgp.application_id
 WHERE cpr.concurrent_program_name      = '$cp_name'
 ORDER BY 1;
```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-RGA.jpg" alt="Install CP - RGA" title="Install CP - RGA" width="400" />

#### Example Output

```
Request Group                  App
------------------------------ ----------
System Administrator Reports   FND
```

### upload\_dd

#### Validation Query

```sql
SELECT xdd.data_source_code "Code", xtm.default_language "Lang", xtm.default_territory "Terr"
  FROM xdo_ds_definitions_b xdd
  LEFT JOIN xdo_templates_b xtm
    ON xtm.application_short_name       = xdd.application_short_name
   AND xtm.data_source_code             = xdd.data_source_code
 WHERE xdd.data_source_code             = '$cp_name'
 ORDER BY 1, 2, 3;
```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-DD.jpg" alt="Install CP - DD" title="Install CP - DD" width="400" />

#### Example Output

```
Code                 Lang Terr
-------------------- ---- ----
XX_ERPXMLCP_EM       en   US
```

### upload\_all\_temps

#### Validation Query

```sql
SELECT xdd.data_source_code "Code", 
        xlb_d.file_name "Data Template",
        xlb_b.file_name "Bursting File",
        xtm.template_code "Template",
        xlb.language "Lang", 
        xlb.territory "Terr",
        xlb.file_name || 
               CASE
               WHEN xlb.language = xtm.default_language AND
                    xlb.territory = xtm.default_territory
               THEN '*' END "File"
  FROM xdo_ds_definitions_b             xdd
  LEFT JOIN xdo_lobs                    xlb_d
    ON xlb_d.application_short_name     = xdd.application_short_name
   AND xlb_d.lob_code                   = xdd.data_source_code
   AND xlb_d.lob_type                   = 'DATA_TEMPLATE'
  LEFT JOIN xdo_lobs                    xlb_b
    ON xlb_b.application_short_name     = xdd.application_short_name
   AND xlb_b.lob_code                   = xdd.data_source_code
   AND xlb_b.lob_type                   = 'BURSTING_FILE'
  LEFT JOIN xdo_templates_b             xtm
    ON xtm.application_short_name       = xdd.application_short_name
   AND xtm.data_source_code             = xdd.data_source_code
  LEFT JOIN xdo_lobs                    xlb
    ON xlb.application_short_name       = xtm.application_short_name
   AND xlb.lob_code                     = xtm.template_code
   AND xlb.lob_type                     LIKE 'TEMPLATE%'
 WHERE xdd.data_source_code             = '$cp_name'
   AND xdd.application_short_name       = '$app'
 ORDER BY 1, 2, 3, 4;
```

#### QSD

<img src="/migrated_images/2014/09/Install-CP-Template.jpg" alt="Install CP - Template" title="Install CP - Template" width="400" />

#### Example Output

```
Code            Data Template        Bursting File          Template             Lang Terr File
--------------- -------------------- ---------------------- -------------------- ---- ---- -------------------------
XX_ERPXMLCP_EM  XX_ERPXMLCP_EM.xml   XX_ERPXMLCP_EM_BUR.xml XX_ERPXMLCP_EM       en   US   XX_ERPXMLCP_EM.xsl*
                                                                                           XX_ERPXMLCP_EM.rtf*
                                                            XX_ERPXMLCP_EM_XML   en   US   XX_ERPXMLCP_EM_XML.xsl*
                                                                                           XX_ERPXMLCP_EM_XML.rtf*
```
