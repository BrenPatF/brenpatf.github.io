---
layout: post
title: "Data Modelling of XML SOAP Documents"
date: 2012-01-29
categories: 
  - "data-model"
  - "oracle"
  - "web-services"
  - "xml"
tags: 
  - "oracle"
  - "soap"
  - "web-service"
  - "xml-2"
---

I have been involved in a number of projects where web services are used for interfacing, and have generally found a high level of complexity in their use compared with traditional interfacing methods. One of the areas of complexity lies in converting between the XML messages and the fields and arrays used in conventional programming languages such as Oracle PL/SQL. Often this is handled in an unmodular way with request messages being manually built, and the response message being parsed by individual xpath searches for specific strings.

My approach is to handle these conversions in a generic layer that lies between the client applications and the low-level APIs provided by the programming language for HTTP calls and XML processing. This post deals with a data model and the data structures used in the PL/SQL package that I wrote for calling web services from PL/SQL. I have posted on the program itself here: [A Layered Approach To Processing XML Web Services](https://brenpatf.github.io/migrated/a-layered-approach-to-processing-xml-web-services/ "A Layered Approach To Processing XML Web Services") and expect to post on examples of use at a later date.

## Web Services
 XML web services have become a standard mechanism for interfacing data between applications across the internet. The advantage that they have is that a standard transfer mechanism (HTTP, Hypertext Transfer Protocol) is used, and the data are formatted according to a standard specification, regardless of the technologies in which the applications are implemented. The data formats are described in a Web Services Description Language (WSDL) file, and interfacing is effected using SOAP (Simple Object Access Protocol) messages, as specified by the World Wide Web Consortium (W3C).

Web services form the cornerstone of Service Oriented Architecture (SOA), which Oracle uses in its Application Integration Architecture (AIA). A good acronym dictionary is vital for working in these areas 🙂.

Interfacing by web services consists of sending input data as a text string in XML format by an HTTP request, and receiving an HTTP response, also as a text string in XML format.

## My Web Service Interface Program
The layer package is intended to handle any data structures that can be represented in XML. On the request side, the client application will call layer APIs to add records and elements to build the request structure without having to write any XML directly, and the layer will construct the XML SOAP message and call the web service. The response side relies on a generic data structure comprising two hierarchical arrays: a structure array that specifies the group structure of the response XML message (or a subset of it), and a data array that holds the data with pointers to the structure array. The structure array forms an input, and is used by the layer to call standard XML APIs (Oracle XML APIs in my implementation) to retrieve the data from the response. The data modelling and conceptual framework are not language-specific, while my implementation is in Oracle PL/SQL.

## SOAP Data Model
Both input and output of a web service call include an XML SOAP message, which is a text string consisting of the data in the form of an XML hierarchy. The elements in the hierarchy contain a mandatory name field, plus optional namespace and value, optional list of child elements, and an optional list of attributes. The hierarchy must contain certain standard elements and attributes, within a structure shown in this skeleton SOAP message (modified from an example here [XML Soap, by W3Schools.com](https://www.w3schools.com/xml/xml_soap.asp "XML Soap, by W3Schools.com") ):

<img src="/migrated_images/2012/01/WS-Blog-Soap-Doc.jpg" alt="WS Blog Soap Doc" title="WS Blog Soap Doc" />

In addition to the standard elements and attributes, there may be application specific elements, attributes and values as indicated by the ellipses.The hierarchy of elements can be represented as below, where the group entity on the left represents the fact that a number of elements at a given level may be implicitly grouped, although without this grouping being explicit in the SOAP document. We use this grouping in the data structures on the response side but not on the request side:

<img src="/migrated_images/2012/01/WS-Blog-Data-Model.jpg" alt="WS Blog Data Model" title="WS Blog Data Model" />

Each element in an XML document has a name, an optional value, optional attribute name/value pairs, with the names being optionally name space-qualified, and the element may contain child elements. Please refer to widely available documentation on the SOAP protocol for further information.

## Request Side Data Structures

<img src="/migrated_images/2012/01/WS-Blog-Request.jpg" alt="WS Blog Request" title="WS Blog Request" />

The diagram shows the main data structures that we use for building the request. Boxes with double borders represent arrays, and a solid arrow represents a pointer from a field to an array element.

The XML Tree List structure is derived from the general model above by treating the list of attributes as a string to be included in the element entity, and corresponds to the array specified by data type xml\_tree\_list\_type in the following table. This array is built from procedure calls by the client application, and is processed internally by a recursive procedure to construct the SOAP request message. Note that as well as the XML tree types, we have defined two additional list structures for convenience of parameter passing:

- Name Space List for passing name spaces
- XML Field List as it is often convenient to treat a group of fields as a record, consisting of a parent element and a list of child fields

### Name Space List Data Types

| **Type Name** | **Element** | **Category** | **Description** |
| --- | --- | --- | --- |
| name\_space\_type |  | Object | Name space |
| | ns\_label | Character | Name space label |
| | ns\_address | Character | Name space address |
| name\_space\_list\_type |  | Array | List of name spaces |
| | _(unnamed element)_ | name\_space\_type\* | Name space |

### Field List Data Types

| **Type Name** | **Element** | **Category** | **Description** |
| --- | --- | --- | --- |
| field\_type |  | Object | XML field |
| | field\_name | Character | Field name |
| | field\_value | Character | Field value (optional) |
| field\_list\_type |  | Array | List of XML fields |
| | _(unnamed element)_ | field\_type\* | XML field |

### XML Tree List Data Types

| **Type Name** | **Element** | **Category** | **Description** |
| --- | --- | --- | --- |
| xml\_tree\_type |  | Object | Record in the XML tree holding the data for an XML element |
| | parent\_id | Integer | Index to parent record in the XML tree |
| | ns\_label | Character | Name space label (optional) |
| | field\_name | Character | Field name |
| | field\_value | Character | Field value (optional) |
| | attr\_string | Character | Attribute string, including name and value (optional) |
| xml\_tree\_list\_type |  | Array | XML tree |
| | _(unnamed element)_ | xml\_tree\_type\* | Record in the XML tree |

## Response Side Data Structures

<img src="/migrated_images/2012/01/WS-Blog-Response1.jpg" alt= "WS Blog - Response" title= "WS Blog - Response" />

The diagram shows the main data structures that we use for processing the response. The broken arrow between the two arrays signifies that each nested child list in the data tree corresponds to a child group record in the structure tree.

The Group Structure Tree List array defines the group structure of the output data structure. To simplify input, this is first set up in an unnested structure, where a field points to its parent, by procedure calls from the client application. The program then converts this into the nested structure shown below it, where the children are included in the parent record, which is more suitable for the later processing. Note that the hierarchy may be, and usually is, a subset of the actual SOAP response, since typically layers are present in a SOAP response message that are not useful for the client application. Also note that in the event of a standard error response being returned from the web service, the group structure is replaced by that for the error response by the program.

- Each of the arrays (at the top level) has a root element without a parent, and these records have null values other than for their respective child lists
- The model can represent any number of hierarchy levels

### Group Structure Data Types

| **Type Name** | **Element** | **Category** | **Description** |
| --- | --- | --- | --- |
| int\_list\_type |  | Array | List of integers |
| | _(unnamed element)_ | Integer | Integer |
| structure\_tree\_type |  | Object | Structure tree record |
| | group\_name | Character | The group tag, used as a search string |
| | ns\_prefix | Character | Name space prefix |
| | attr\_string | Character | An attribute string that has to be included in searches where the group name is ambiguous (Oracle JDeveloper uses array as a group tag for arrays, with an attribute to differentiate) |
| | child\_list | int\_list\_type\* |  |
| structure\_tree\_list\_type |  | Array | Structure tree list |
| | _(unnamed element)_ | structure\_tree\_type\* | Structure tree record |

### Data Structure Data Types

| **Type Name** | **Element** | **Category** | **Description** |
| --- | --- | --- | --- |
| child\_group\_list\_type |  | Array | List of indexes in the data tree list of child groups of current data tree record |
| | _(unnamed element)_ | int\_list\_type\* | List of indexes in the data tree list of child records of current group |
| data\_tree\_type |  | Object | Data tree record |
| | field\_list | field\_list\_type\* | Field list (specified in input side) |
| | child\_group\_list | child\_group\_list\_type\* | List of indexes in the data tree list of child groups of current data tree record |
| data\_tree\_list\_type |  | Array | Data tree list |
| | _(unnamed element)_ | data\_tree\_type\* | Data tree record |

## Generic Data Models
In database design it is well known that overly generic data models lead to poor performance and extra application complexity. The web service interfacing model is of course highly generic, and it should be noted that the same problems may indeed offset the acknowledged standardisation advantages. The data model and structures described here necessarily reflect the genericity of the underlying architecture, while the approach taken is intended to reduce application complexity by moving much of it into a callable module.
