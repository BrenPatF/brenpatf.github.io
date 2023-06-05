---
layout: post
title:  "The Math Function Unit Testing Design Pattern"
tags: ["programming", "testing", "unit test"]
date:   2023-06-05 09:00:00 +0100
---

This article provides an overview of a new design pattern for unit testing in any programming language, ‘The Math Function Unit Testing Design Pattern’. The data-driven design pattern involves repeatable, automated testing, and minimises unit test code ([see example](#wrapper-function---purely_wrap_all_nets)) by delegating to library modules the unit test driver, reading/writing of test data and assertion and formatting of results.

After a section on the [background](#background), the article describes the design pattern in two parts: the [first](#systems-transactions-and-apis) deals with the concepts of transactions and APIs and why we should base our unit testing around APIs, while the [second](#api-testing-with-the-math-function-unit-testing-design-pattern) describes the design pattern at high level.

There are modules in Oracle PL/SQL, JavaScript, Powershell and Python implementing utilities to support the design pattern, and demonstrating its use. A final [section](#see-also) provides links to their GitHub projects and some related articles.

<img src="/images/2023/06/05/bauhaus_intro.jpg">

[Infographic: The Bauhaus, Where Form Follows Function](https://www.archdaily.com/225792/the-bauhaus)

<iframe style="border-radius:12px" src="https://open.spotify.com/embed/track/1wyVyr8OhYsC9l0WgPPbh8?utm_source=generator" width="100%" height="352" frameBorder="0" allowfullscreen="" allow="autoplay; clipboard-write; encrypted-media; fullscreen; picture-in-picture" loading="lazy"></iframe>

# Contents
[&darr; Background](#background)<br />
[&darr; Systems, Transactions and APIs](#systems-transactions-and-apis)<br />
[&darr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />
[&darr; See Also](#see-also)<br />

## Background
[&uarr; Contents](#contents)<br />

Some years ago, I became involved in automated testing within Oracle database systems for the first time, after many years working on projects that used only manual testing. The potential benefits of repeatable, automated testing were clear, but I soon realised that there was one big problem: Almost all the material available on how to implement such testing was based on concepts developed for object oriented programming in Java in the early 2000s.

These concepts generally involved splitting code into small units and writing large numbers of short tests (later often called 'microtests'), with the proviso that the tests should run very quickly and in isolation, without any database connection.

Some database developers tried to adopt these ideas and apply them to database programs by simply copying the approach, while dropping the isolation proviso. The main problem with this is that most of the code in database systems tends to be in the form of SQL statements, including complex logic and table joins. SQL is sometimes described as the only widely successful 4GL (4'th generation language), specifying joins and logic declaratively, and it is generally considered best practice to maximise its use, with procedural code (PL/SQL in Oracle) largely used to wrap the SQL into callable subprograms.

It seemed clear that microtesting was not well suited to testing programs written mostly in declarative languages such as SQL, so I felt I had to develop new ideas. These ideas were first expressed in a presentation at the Oracle User Group Ireland Conference in March 2018, [The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing).

Since then I have developed what I now call 'The Math Function Unit Testing Design Pattern', and have written modules in Oracle PL/SQL, JavaScript, Powershell and Python implementing utilities to support it, and demonstrating its use.
## Systems, Transactions and APIs
[&uarr; Contents](#contents)<br />
[&darr; The Data Processing Module](#the-data-processing-module)<br />
[&darr; Client Transactions](#client-transactions)<br />
[&darr; Types of Testing](#types-of-testing)<br />
[&darr; Non-Transactional Modules](#non-transactional-modules)<br />

### The Data Processing Module
[&uarr; Systems, Transactions and APIs](#systems-transactions-and-apis)<br />

Application systems consist of different types of component, including user interfaces, databases and code. In order to facilitate automated unit testing it is convenient to separate the data processing module from components such as user interfaces, where testing is harder to automate.

<img src="/images/2023/06/05/api-app.png">

The diagram shows a schematic representation of a data processing module consisting of a database (where necessary) and a set of API (*Application Programming Interface*) functions called by external client applications. The APIs may take inputs from various possible sources, not only parameters but potentially also, for example, database tables and operating system files; similarly they may output in various possible ways, not only via return values, but potentially writing also, for example, to database tables and operating system files. The APIs may be structured in many ways including public entry point subprograms in packages, or database views.

In an API-centred module all interfacing between the module and external applications are through these 'transactional' APIs. Each call effects a single transaction, where a transaction is defined simply as a complete logical unit of data activity. In this context, other components that belong to the same overall system as the data processing module, such as web pages, would be considered as external.

Where a database is present, the transaction may move the database from one logically consistent state to another consistent state, or leave it unaltered.

The transaction may also transfer a logically complete set of data to the client application, in response to the inputs supplied. The data transferred may or may not depend on data in the database, and may involve data transformations.

### Client Transactions
[&uarr; Systems, Transactions and APIs](#systems-transactions-and-apis)<br />

<img src="/images/2023/06/05/trx-db.png">

This diagram shows a schematic representation of a single transaction triggered by a call from a client application to a single API, with database access.

<img src="/images/2023/06/05/trx-nondb.png">

This diagram shows a schematic representation of a single transaction triggered by a call from a client application to a single API, without database access.

### Types of Testing
[&uarr; Systems, Transactions and APIs](#systems-transactions-and-apis)<br />
[&darr; Microtesting](#microtesting)<br />
[&darr; System Testing](#system-testing)<br />
[&darr; API-Based Unit Testing](#api-based-unit-testing)<br />

#### Microtesting
[&uarr; Types of Testing](#types-of-testing)<br />

Microtesting is a style of testing based on writing lots of short tests of small subprograms, focussed around internal branching logic, with dependencies excluded.

One of its proponents describes it this way in [The Technical Meaning Of Microtest](https://www.geepawhill.org/2018/04/16/the-technical-meaning-of-microtest/):

> "A single microtest is a small fast chunk of code that we run, outside of our shipping source but depending on it, to confirm or deny simple statements about how that shipping source works, with a particular, but not exclusive, focus on the branching logic within it. "

As mentioned in the background section it seems to have major shortcomings when applied to higher-level programming.

However, even for application to lower-level programming it seems questionable whether the limited scope of its aims justifies the effort required: It's quite hard in practice to test only the internal logic in isolation from all dependencies.

#### System Testing
[&uarr; Types of Testing](#types-of-testing)<br />

In order for the system to be correct it is necessary for the APIs implementing the transactions to work consistently with each other, and testing that they do may be considered as system testing, along with testing of other components such as user interfaces.

This higher level testing is usually harder to automate.

#### API-Based Unit Testing
[&uarr; Types of Testing](#types-of-testing)<br />

Once we view the data processing module as a set of APIs interfacing with external systems, it seems obvious that we would want to test these APIs individually before system testing. It also seems clear that we would want to test them in the common sense of testing, that for any given input the API should produce the expected output for that input, and that we should not in general exclude, or mock, subprogram calls except in special circumstances.

In well designed systems the complexity of a transaction would remain manageable even as systems increase in overall size, so that testing them would also be manageable.

Ideally, we would automate unit testing to bring all the well known benefits of automation in terms of quality and overall effort. In order to achieve this, we need to think carefully about the design of our unit test programs; in particular, we want to design so that as much testing code is reusable as possible. This is what 'The Math Function Unit Testing Design Pattern', described in overview in the remainder of this article, is about.

### Non-Transactional Modules
[&uarr; Systems, Transactions and APIs](#systems-transactions-and-apis)<br />

In certain situations there may be a need for a module that maintains temporary internal state between API calls. In these cases there will be an underlying transaction comprised of a number of API calls that are not themselves transactional. In order to unit test such a module we can construct a wrapper function around the calls that represents the underlying transaction. Here is an example:

- [Timer_Set - Oracle PL/SQL Code Timing Module](https://github.com/BrenPatF/timer_set_oracle)



## API Testing with The Math Function Unit Testing Design Pattern
[&uarr; Contents](#contents)<br />
[&darr; Unit Test Steps](#unit-test-steps)<br />
[&darr; Wrapper Function](#wrapper-function)<br />
[&darr; Scenarios and Categories](#scenarios-and-categories)<br />
[&darr; Generic Data Model](#generic-data-model)<br />
[&darr; Design Pattern Components](#design-pattern-components)<br />
[&darr; Example of Wrapper Function](#example-of-wrapper-function)<br />

The design pattern is intended for fully automated, repeatable unit testing of transactional APIs, and its main features are:

- The unit under test, the API, is viewed from the perspective of a mathematical function having an 'extended signature', comprising any actual parameters and return value, together with other inputs and outputs of any kind
- A wrapper function is constructed based on this conceptual function, and this wrapper function is 'externally pure', in the sense that any data changes made are rolled back before returning, and it is essentially deterministic
- The wrapper function performs the steps necessary to test the API in a single scenario
- It takes all inputs of the extended signature as a parameter, creates any test data needed from them, effects a transaction with the API, and returns all outputs as a return value
- Any test data, and any data changes made by the API, are reverted before return
- The wrapper function has a fixed signature with input as a set of input groups containing arrays of records, and return value a set of output groups containing arrays of records
- A library test driver module reads data for all scenarios, with both inputs to the API and the expected outputs, and metadata records describing the specific data structure
- The module takes the actual outputs from the wrapper function and merges them in alongside the expected outputs to create an output results object
- The output results object is processed to produce summary and detailed results in HTML or other formats

### Unit Test Steps
[&uarr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />

At a high level the Math Function Unit Testing design pattern involves three main steps:

1. Create an input file containing all test scenarios with input data and expected output data for each scenario
2. Create a results object based on the input file, but with actual outputs merged in, based on calls to the unit under test, via the wrapper function
3. Use the results object to generate unit test results files formatted in HTML or text, or other format

<img src="/images/2023/06/05/HLS.png">

### Wrapper Function
[&uarr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />

This diagram shows a schematic representation of the data flows involved in a single call to the API wrapper function.

<img src="/images/2023/06/05/wrapper.png">

The wrapper function takes all inputs in a single, composite parameter and it outputs via a composite return value. Where the API reads from an external source the wrapper writes those data items to the source ahead of its call, and where the API writes outputs to an external source the wrapper reads from the source in order to include them in its return value.

Any external writing is reverted before the wrapper returns. If there is any indeterminacy in the API outputs the wrapper should substitute a deterministic value. In this way the wrapper function can be described as 'externally' pure.

### Scenarios and Categories
[&uarr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />

The art of unit testing lies in choosing a set of scenarios that will produce a high degree of confidence in the functioning of the unit under test across the often very large range of possible inputs.

A useful approach to this can be to think in terms of categories of inputs, where we reduce large ranges to representative categories.  I explore this approach further in this article:

- [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/jekyll/update/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html)

The diagram shows a schematic representation of a sequence of calls to the API wrapper function with input data points corresponding to categories within category sets, resulting in actual output data points that can be compared with expected outputs for the scenario.

<img src="/images/2023/06/05/cat-api.png">

The wrapper function specific to the API is called within a loop over scenarios by a library test driver module. The library module creates an output object combining the expected and actual outputs for all scenarios.

If we can use a generic data model for inputs and outputs of the wrapper function, this would allow us to store scenario data in a standard format, centralise the code for reading and writing scenario data from/to file, and centralise the code for looping over the scenarios calling the wrapper function.

This would mean that the only API-specific test code needed would be for converting between specific and generic inputs and outputs of the extended signature, and calling the API.

#### Example of a Category Structure Diagram

In determining the categories to include in scenarios it may be a good approach to draw a diagram, such as this example, taken from [Net_Pipe - Oracle PL/SQL Network Analysis Module](https://github.com/BrenPatF/plsql_network):

<img src="/images/2023/06/05/CSD-plsnet.png">

### Generic Data Model
[&uarr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />

The diagram shows a generic data model consisting of a 3-level nested array for both inputs and return value for the wrapper function.

<img src="/images/2023/06/05/gdm.png">

With a generic model of this kind we can use the standard JSON (Jason Object Notation) format for storing scenario data, which can be read and written in a single library module.

#### Example of a JSON Structure Diagram

A good start in determining the extended signature of the wrapper function may be to draw a diagram, such as this example, taken from [Net_Pipe - Oracle PL/SQL Network Analysis Module](https://github.com/BrenPatF/plsql_network):

<img src="/images/2023/06/05/JSD-plsnet.png">

In this example, the input Link group correspondes to a table (External Inputs in the earlier diagram), while the output group corresponds to a return value (Program Outputs in the earlier diagram)

### Design Pattern Components
[&uarr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />

The diagram shows how the various code components work together and exchange data to go from an input scenarios JSON file to results files formatted in HTML or other formats.

<img src="/images/2023/06/05/mfutdp-flow-ext.png">

Note that the green library packages do not need to be written for each API: One instance of the external package is needed in the language of the API, and only one instance of the formatting package is needed globally since it uses JSON files for input (I wrote this in JavaScript).

### Example of Wrapper Function
[&uarr; API Testing with The Math Function Unit Testing Design Pattern](#api-testing-with-the-math-function-unit-testing-design-pattern)<br />
[&darr; Scenario 6 - Network Diagram](#scenario-6---network-diagram)<br />
[&darr; Wrapper Function - Purely_Wrap_All_Nets](#wrapper-function---purely_wrap_all_nets)<br />

This article is intended as a high level overview of the design pattern, but it is worth including one example of the wrapper function code. This example is taken from a GitHub project:

- [Oracle PL/SQL Network Analysis Module](https://github.com/BrenPatF/plsql_network)

> "The module contains a PL/SQL package for the efficient analysis of networks that can be specified by a view representing their node pair links. The package has a pipelined function that returns a record for each link in all connected subnetworks, with the root node id used to identify the subnetwork that a link belongs to."

We'll show the network used as scenario 6 in testing first, followed by the wrapper function code.

Although there is a certain amount of internal complexity in the algorithm used in the API, the inputs and outputs are simple flat arrays of records, and the unit test code is quite simple.

#### Scenario 6 - Network Diagram
[&uarr; Example of Wrapper Function](#example-of-wrapper-function)<br />

<img src="/images/2023/06/05/plsql_network - sce_6.png">

#### Wrapper Function - Purely_Wrap_All_Nets
[&uarr; Example of Wrapper Function](#example-of-wrapper-function)<br />

Here is the complete function:
```sql
FUNCTION Purely_Wrap_All_Nets(
            p_inp_3lis                     L3_chr_arr)   -- input list of lists (group, record, field)
            RETURN                         L2_chr_arr IS -- output list of lists (group, record)
  l_act_2lis        L2_chr_arr := L2_chr_arr();
  l_csr             SYS_REFCURSOR;
BEGIN
  FOR i IN 1..p_inp_3lis(1).COUNT LOOP
    INSERT INTO network_links VALUES (p_inp_3lis(1)(i)(1), p_inp_3lis(1)(i)(2), p_inp_3lis(1)(i)(3));
  END LOOP;
  l_act_2lis.EXTEND;
  OPEN l_csr FOR SELECT * FROM TABLE(Net_Pipe.All_Nets);
  l_act_2lis(1) := Utils.Cursor_To_List(x_csr    => l_csr);
  ROLLBACK;
  RETURN l_act_2lis;
END Purely_Wrap_All_Nets;
```
This is a good example of how little code may be necessary when following the Math Function Unit Testing design pattern: The complexity of the code reflects only the complexity of the external interface. Here, there are single input and output groups, and so the code is equally simple, despite the unit under test having a larger degree of internal complexity.
## See Also
[&uarr; Contents](#contents)<br />
[&darr; Articles and Presentations](#articles-and-presentations)<br />
[&darr; JavaScript](#javascript)<br />
[&darr; Powershell](#powershell)<br />
[&darr; Oracle PL/SQL](#oracle-plsql)<br />
[&darr; Python](#python)<br />

This section has links to other articles and projects relevant to the Math Function Unit Testing Design Pattern.

### Articles and Presentations
[&uarr; See Also](#see-also)<br />

This links to the powerpoint slides I presented at the Ireland Oracle User Group conference in Dublin of March 2018:
- [The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing)

These are blog articles from October 2021 and August 2016 respectively:
- [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/jekyll/update/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html)
- [A Note on Dependencies and Database Unit Testing](http://aprogrammerwrites.eu/?p=1769)

### JavaScript
[&uarr; See Also](#see-also)<br />

This GitHub project has the JavaScript version of the design pattern test driver utility:
- [Trapit - JavaScript Unit Tester/Formatter](https://github.com/BrenPatF/trapit_nodejs_tester)

It also has the APIs used to format unit test results in HTML and text from JSON results files created in any language, and the core formatting API is itself unit tested using the design pattern.

This GitHub project has the JavaScript version of a code timing utility that is unit tested using the design pattern:
- [Timer_Set - JavaScript Code Timing Module](https://github.com/BrenPatF/timer_set_nodejs)

It is an example of unit testing where the module has non-transactional entry points, with the unit test wrapper function implementing the underlying transactions.

### Powershell
[&uarr; See Also](#see-also)<br />

This GitHub project has the Powershell version of the design pattern test driver utility:
- [Powershell Utilities Module](https://github.com/BrenPatF/powershell_utils)

It also has the APIs used to generate a template for the design pattern input JSON file based on input CSV files, and a range of other utilities, which are unit tested using the design pattern.

### Oracle PL/SQL
[&uarr; See Also](#see-also)<br />

This GitHub project has the Oracle PL/SQL version of the design pattern test driver utility:
- [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester)

These GitHub projects have Oracle PL/SQL utilities that are unit tested using the design pattern, including the latest expression of the pattern, with the category structure diagram and use of the Powershell template generator:
- [Utils - Oracle PL/SQL General Utilities Module](https://github.com/BrenPatF/oracle_plsql_utils)
- [Net_Pipe - Oracle PL/SQL Network Analysis Module](https://github.com/BrenPatF/plsql_network)

These GitHub projects have Oracle PL/SQL utilities that are unit tested using the design pattern, with earlier expressions of the pattern, without the category structure diagram and use of the Powershell template generator:
- [Log_Set - Oracle PL/SQL Logging Module](https://github.com/BrenPatF/log_set_oracle)
- [Timer_Set - Oracle PL/SQL Code Timing Module](https://github.com/BrenPatF/timer_set_oracle)
- [Oracle PL/SQL API Demos - demonstrating instrumentation and logging, code timing and unit testing of Oracle PL/SQL APIs](https://github.com/BrenPatF/oracle_plsql_api_demos)

### Python
[&uarr; See Also](#see-also)<br />

This GitHub project has the Python version of the design pattern test driver utility:
- [Trapit - Python Unit Testing Module](https://github.com/BrenPatF/trapit_python_tester)
It is itself unit tested using the design pattern.

This GitHub project has the Python version of a code timing utility that is unit tested using the design pattern:
- [Timer_Set - Python Code Timing Module](https://github.com/BrenPatF/timer_set_python)
