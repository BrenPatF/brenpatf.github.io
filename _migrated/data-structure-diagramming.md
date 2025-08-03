---
layout: post
title: "Data Structure Diagramming"
date: 2012-01-21
migrated: true
group: design
categories: 
  - "analytics"
  - "erd"
  - "object"
  - "oracle"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "erd"
  - "object"
  - "recursive"
  - "sql"
  - "subquery-factor"
---

Like many SQL developers I have always used entity-relationship diagrams to help in writing queries, and would extract sections to document them. Some years ago, however, I realised that having a single static diagram was not sufficient for complex queries with large numbers of tables, structures such as inline views, and multiple table instances. I therefore developed a diagram-based design methodology that I published in May 2009 on scribd. Since then I have extended the ideas in that approach to develop diagrams to cover various additional structures in SQL and in other areas. These diagrams were developed as needed for particular scenarios and have been published in several documents on scribd. I thought it would be a good idea to bring them together in one place, namely here, with example diagrams and the scribd document embedded thereafter. _\[Incidentally, I wonder what readers make of this 8-dimensional document structure?\]_

I would categorise them under four headings:

- Entity-Relationship Diagrams
- Structured Design Methodology
- SQL Special Structures
- Object Structures

## Entity-Relationship Diagrams

### Oracle Spatial Schema
 The embedded document below also includes an ERD of the much simpler HR schema, but this one is more interesting as it shows extensive use of subtypes. The document is concerned with networks and I superimposed tree and non-tree network links on the diagram.

<img src="/migrated_images/2012/01/HR-and-Network-Diagrams-SDO-ERD1.jpg" 
       alt="HR and Network Diagrams - SDO ERD" 
       title="HR and Network Diagrams - SDO ERD" />

 <iframe class="scribd_iframe_embed" title="An Oracle Network Traversal PL SQL Program" src="https://www.scribd.com/embeds/32976987/content?start_page=1&view_mode=scroll&access_key=key-1419uez1s7vnasb6e05i" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View An Oracle Network Traversal PL SQL Program on Scribd" href="https://www.scribd.com/document/32976987/An-Oracle-Network-Traversal-PL-SQL-Program#from_embed" style="color: #098642; text-decoration: underline;"> An Oracle Network Traversal PL SQL Program </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

### Oracle Customer Model and Multi-Org
Here I used shading to distinguish between org-striped, org-linked (my term) and other entities.
<img src="/migrated_images/2012/01/Xilinx-ERDs-TCA-Key.jpg" alt="Xilinx ERDs TCA Key" title="Xilinx ERDs TCA Key" />

<iframe class="scribd_iframe_embed" title="Oracle Multi-Org - AR and OM Data Structure Changes" src="https://www.scribd.com/embeds/16996133/content?start_page=1&view_mode=scroll&access_key=key-17xmad7xp6rfrlidkym4" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View Oracle Multi-Org - AR and OM Data Structure Changes on Scribd" href="https://www.scribd.com/document/16996133/Oracle-Multi-Org-AR-and-OM-Data-Structure-Changes#from_embed" style="color: #098642; text-decoration: underline;"> Oracle Multi-Org - AR and OM Data Structure Changes </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

## Structured Design Methodology
The methodology involves a sequence of diagrams and tables, so I have not extracted a diagram in this case.
<iframe class="scribd_iframe_embed" title="A Structured Approach to SQL Query Design" src="https://www.scribd.com/embeds/15723877/content?start_page=1&view_mode=scroll&access_key=key-1wlb575hb2xc5lr97sb8" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View A Structured Approach to SQL Query Design on Scribd" href="https://www.scribd.com/document/15723877/A-Structured-Approach-to-SQL-Query-Design#from_embed" style="color: #098642; text-decoration: underline;">A Structured Approach to SQL Query Design</a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

## SQL Special Structures

### Multiple Table Instances with Scalar Subqueries in Where Clause
<img src="/migrated_images/2012/01/JNSQ-Join-Subquery.jpg" alt="JNSQ Join Subquery" title="JNSQ Join Subquery" />

### Subquery Factor
<img src="/migrated_images/2012/01/WJSQ-With-Subquery.jpg" alt="WJSQ With Subquery" title="WJSQ With Subquery" />

### Selecting Database Function
<img src="/migrated_images/2012/01/FNSC-Database-Function.jpg" alt="FNSC Database Function" title="FNSC Database Function" />

### Selecting Scalar Subqueries
<img src="/migrated_images/2012/01/WJKP-Select-Scalar-Subquery-Keep.jpg" alt="WJKP Select Scalar Subquery Keep" title="WJKP Select Scalar Subquery Keep" />

<iframe class="scribd_iframe_embed" title="SQL Pivot and Prune Queries - Keeping an Eye on Performance" src="https://www.scribd.com/embeds/54433084/content?start_page=1&view_mode=scroll&access_key=key-osozbbqczb7m4qadwiy" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View SQL Pivot and Prune Queries - Keeping an Eye on Performance on Scribd" href="https://www.scribd.com/document/54433084/SQL-Pivot-and-Prune-Queries-Keeping-an-Eye-on-Performance#from_embed" style="color: #098642; text-decoration: underline;"> SQL Pivot and Prune Queries - Keeping an Eye on Performance </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

### Nested Analytics Subqueries
<img src="/migrated_images/2012/01/Ranges-Analytic-Key.jpg" alt="Ranges Analytic Key" title="Ranges Analytic Key" />

### Model Clause
<img src="/migrated_images/2012/01/MOD-SPT.jpg" alt="MOD SPT" title="MOD SPT" />

### Recursive Subquery Factor
<img src="/migrated_images/2012/01/RSQ-NOV.jpg" alt="RSQ NOV" title="RSQ NOV" />

<iframe class="scribd_iframe_embed" title="Forming Range-Based Break Groups With Advanced SQL" src="https://www.scribd.com/embeds/57696875/content?start_page=1&view_mode=scroll&access_key=key-1exh47ppyi1yklevkz9p" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View Forming Range-Based Break Groups With Advanced SQL on Scribd" href="https://www.scribd.com/document/57696875/Forming-Range-Based-Break-Groups-With-Advanced-SQL#from_embed" style="color: #098642; text-decoration: underline;"> Forming Range-Based Break Groups With Advanced SQL </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

## Object Structures
I use a different type of diagram for object structures from those for SQL and ERDs, and it's intended to be very general, being independent of programming language and applicable to any object structure, allowing arbitrary nesting of array and record types. 
### Code Timer Object
This object was implemented in three languages: Oracle, Perl and Java. 

<img src="/migrated_images/2012/01/Code-Timer-object.jpg" alt="Code Timer object" title="Code Timer object" />


<iframe class="scribd_iframe_embed" title="Code Timing and Object Orientation and Zombies" src="https://www.scribd.com/embeds/43588788/content?start_page=1&view_mode=scroll&access_key=key-23rlpqinxuz4npjcq9l3" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View Code Timing and Object Orientation and Zombies on Scribd" href="https://www.scribd.com/document/43588788/Code-Timing-and-Object-Orientation-and-Zombies#from_embed" style="color: #098642; text-decoration: underline;"> Code Timing and Object Orientation and Zombies </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 

### Excel Array Object
This object was implemented in Perl.

<img src="/migrated_images/2012/01/Perl-XL.jpg" alt="Perl XL" title="Perl XL" />

<iframe class="scribd_iframe_embed" title="A Perl Object for Flattened Master-Detail Data in Excel" src="https://www.scribd.com/embeds/61306184/content?start_page=1&view_mode=scroll&access_key=key-fjpchozunhgqrd859mk" tabindex="0" data-auto-height="true" data-aspect-ratio="null" scrolling="no" width="100%" height="600" frameborder="0" ></iframe> <p style="margin: 12px auto 6px auto; font-family: Helvetica,Arial,Sans-serif; font-size: 14px; line-height: normal; display: block;"> <a title="View A Perl Object for Flattened Master-Detail Data in Excel on Scribd" href="https://www.scribd.com/doc/61306184/A-Perl-Object-for-Flattened-Master-Detail-Data-in-Excel#from_embed" style="color: #098642; text-decoration: underline;"> A Perl Object for Flattened Master-Detail Data in Excel </a> by <a title="View Brendan Furey's profile on Scribd" href="https://www.scribd.com/user/10226568/Brendan-Furey#from_embed" style="color: #098642; text-decoration: underline;" > Brendan Furey </a> </p> 