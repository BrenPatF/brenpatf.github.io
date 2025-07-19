---
layout: post
title: "A Layered Approach To Processing XML Web Services"
date: 2012-02-05
migrated: true
categories: 
  - "oracle"
  - "recursive"
  - "web-services"
  - "xml"
tags: 
  - "oracle"
  - "plsql"
  - "recursive"
  - "soap"
  - "web-services-2"
  - "xml-2"
---

As explained in an earlier post, [Data Modelling XML SOAP Documents](https://brenpatf.github.io/migrated/data-modelling-of-xml-soap-documents/ "Data Modelling XML SOAP Documents"), I have an approach to calling web services that involves the use of a generic layer that lies between the client applications and low-level APIs for HTTP calls and XML processing. The earlier post introduces the subject, and deals with the data modelling aspects. This post gives high-level, largely diagrammatic, design information for my PL/SQL implementation of the approach. I expect to post on examples of use and results at a later date.

## Layer Diagram
 <img src="/migrated_images/2012/02/WS-Program-Layers.jpg" alt="WS Program - Layers" title="WS Program - Layers" />

## External Call Structure

### External Call Structure Diagram

<img src="/migrated_images/2012/02/WS-Program-Entry-Point3.jpg" alt="WS Program - Entry Point" title="WS Program - Entry Point" />

### External Call Procedures


#### Client Side

| **Procedure** | **Description** |
| --- | --- |
| _Client Program_ | Any client program that needs to access web services |
| _Client Converter_ | A program specific to the client to convert between the generic data models of the package and the formats of the client. If there is only one client program then the converter need not be a separate program. |

#### WS Package

| **Procedure** | **Description** |
| --- | --- |
| Set Header | Adds header level nodes into the XML Tree array |
| Format Attribute | Formats an attribute string from name, value and name-space prefix |
| Add Element (Request) | Adds an element node into the XML Tree array |
| Add Record (Request) | Adds a record, consisting of a record header element and child element nodes, into the XML Tree array |
| Add Element (Response) | Adds an element node into the Structure Tree array |
| Add Record (Response) | Adds a record, consisting of a record header element and child element nodes, into the Structure Tree array |
| Process Web Service | Converts the XML Tree array into the SOAP request message, calls the web service and transforms the SOAP response message into the output Data Tree array |
| Write Output | Writes the output Data Tree array |

## Web Service Call

### Web Service Call Structure Diagram
 <img src="/migrated_images/2012/02/WS-Program-Call-WS1.jpg" alt="WS Program - Call WS" title="WS Program - Call WS" />

### Web Service Call Structure Procedures

#### Custom Procedures

| **Procedure** | **Description** |
| --- | --- |
| Call Web Service | Coordinating procedure for the web service call. Note that both request and response writing and reading calls are within loops as the messages can be more than the HTTP maximum chunk size of 32767 bytes |
| Expand Element (Request) | Recursive procedure to create the XML SOAP request from the XML Tree array and other inputs |
| Delim Field | Formats an XML element within its tags |
| Expand Element (Response) | Recursive procedure to convert the initial form of Group Structure Tree List by Parent into the nested form, Group Structure Tree List, used by later processing |

#### Oracle Built-in Packages

| **Procedure** | **Description** |
| --- | --- |
| UTL\_HTTP | Oracle HTTP package used to make the HTTP request and read the response |
| DBMS\_LOB | Oracle ‘large object’ package used for processing CLOB variables for the full request and response, passed in 32767-byte chunks in the HTTP calls |
| DBMS\_XMLDOM | Oracle XML package used to create an XML document and node from the response |
| XMLTYPE | Oracle XML package used to create a variable of XML type for passing to the above package |

## Populate Tree Call

### Populate Tree Call Structure Diagram

<img src="/migrated_images/2012/02/WS-Program-Pop-Tree1.jpg" alt="WS Program - Pop Tree" title="WS Program - Pop Tree" />

### Populate Tree Procedures

#### Custom Procedures

| **Procedure** | **Description** |
| --- | --- |
| Populate Tree | Main procedure for populating Data Tree List array. First an attempt is made to populate the output tree specified by the input Group Structure Tree List array; if this returns an error, then a second call is made to populate the output tree specified by the standard error group structure; sometimes this too can fail, if the HHTP response is not the expected SOAP message, and this will also be trapped and returned as an error message variable |
| Check Fault | Resets the input Group Structure Tree List array to match the standard SOAP error structure and calls the next procedure to populate the corresponding output tree |
| Populate Specific Tree | Populates the output tree specified by the current Group Structure Tree List array: this may be either that specified by the client application, or the standard error structure |
| Populate Tree Record | Recursive procedure to build the output Data Tree List array using Oracle’s XML APIs |

#### Oracle Built-in Packages

| **Procedure** | **Description** |
| --- | --- |
| DBMS\_XMLDOM | _‘The DBMS\_XMLDOM package is used to access XMLType objects, and implements the Document Object Model (DOM), an application programming interface for HTML and XML documents’_ \- Oracle® Database PL/SQL Packages and Types Reference, v11.2 |
| DBMS\_XMLProcessor | _‘The DBMS\_XSLPROCESSOR package provides an interface to manage the contents and structure of XML documents’_ - Oracle® Database PL/SQL Packages and Types Reference, v11.2 |
