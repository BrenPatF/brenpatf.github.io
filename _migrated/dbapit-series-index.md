---
layout: post
title: Design Patterns for Database API Testing
date: 2016-05-08
group: func-testing
migrated: true
permalink: /migrated/dbapit-series-index/
---

**2025 Note:** This is the overview/index page for a 4-part series of articles originally published (in 6 parts) on my Wordpress blog, starting in May 2016, [Design Patterns for Database API Testing 1: Web Service Saving 1 – Design](http://aprogrammerwrites.eu/?p=1614). Since then I have further developed the Oracle unit testing framework, and generalised it into a design pattern for any programming language, as described in June 2023, [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html). The example code in the series can still be found in the git history at the link below, although it's not in the latest commit.

<ul>
  <li><a href="/dbapit/dbapit-1/">DBAPIT 1: Web Service Saving</a></li>
  <li><a href="/dbapit/dbapit-2/">DBAPIT 2: Views</a></li>
  <li><a href="/dbapit/dbapit-3/">DBAPIT 3: Batch Loading of Flat Files</a></li>
  <li><a href="/dbapit/dbapit-4/">DBAPIT 4: REF Cursor Getter</a></li>
</ul>

#### GitHub <img src="/images/common/github-mark.png" style="width: 10%; max-width: 5%;"/><br />

- [Trapit - Oracle PL/SQL Unit Testing Module](https://github.com/BrenPatF/trapit_oracle_tester)<br />

Last October I gave a presentation on database unit testing with utPLSQL, [Oracle Unit Testing with utPLSQL](http://www.slideshare.net/brendanfurey7/oracle-unit-testing-with-utplsql). I mentioned design patterns as a way of reducing the effort of building unit tests and outlined some strategies for coding them effectively.

In the current set of articles, the ideas are developed further, starting from the idea that all database APIs can be considered in terms of two axes:

- Direction (i.e. getter or setter, noting that setters can also 'get')
- Mode (i.e. real time or batch)

<img src="/migrated_images/2016/05/Mode-All.png" alt="Mode: View" title="Mode: View" />

For each cell in the matrix, I construct an example API (or view) with specified requirements against Oracle's HR demo schema, and use this example to construct a testing program with appropriate scenarios as a design pattern. Concepts and common patterns and anti-patterns in automated API testing are discussed throughout, and these are largely independent of testing framework used. However, the examples use my own lightweight independent framework that is designed to help avoid many API testing anti-patterns.

Behind the four examples, there is an underlying design pattern that involves wrapping the API call in a 'pure' procedure, called once per scenario, with the output 'actuals' array including everything affected by the API, whether as output parameters, or on database tables, etc. The inputs are also extended from the API parameters to include any other effective inputs. Assertion takes place after all scenarios and is against the extended outputs, with extended inputs also listed. This concept of the 'pure' function, central to [Functional Programming](https://en.wikipedia.org/wiki/Functional_programming), has important advantages in automated testing. I explained the concepts involved in a presentation at the Oracle User Group Ireland Conference in March 2018:

[The Database API Viewed As A Mathematical Function: Insights into Testing](https://www.slideshare.net/brendanfurey7/database-api-viewed-as-a-mathematical-function-insights-into-testing)

Design patterns involve abstraction and conceptual separation of general features of a situation from the particular. Therefore we will start with a fairly abstract discussion of automated API testing for the database here, followed by a listing of a number of extremely prevalent antipattern approaches to database API testing, with ways avoid them.

We discuss the use cases in question, describe the test cases, and show the results in the individual articles. The code itself centralises as much as possible in order to make specific test code as small as possible, and is structured very differently from most unit testing code that I have seen.

## General Discussion of Database Unit Testing

The underlying functionality for unit testing could be described logically as:

- Given a list of test inputs, X and a list of expected outputs, E, for function F:
- For each x in X, with e in E:
    - Apply y = F(x)
    - Assert y = e

As the Functional Programming community knows well, functions having well-defined parameter inputs, returning values as outputs, and with no 'side-effects', are the easiest to test reliably. The difficulty with database unit testing is that most use cases do not fall into that category; instead, database procedures can read from and write to the database as well as using input and output parameters. This means that theoretically the inputs and outputs could include the whole database (at least); furthermore the database is a shared resource, so other parties can alter the data we are dealing with. One important consequence of these facts is that much of the thinking on best practices for unit testing, coming as it does from the non-database world, is not applicable here. So what to do?

### Pragmatic testing

To make progress, we note that our purpose with unit testing is not to formally prove that our programs work, but rather includes the following aims:

- Improve code quality within a Test Driven Development (TDD) approach
- Provide regression tests to allow safe code-refactoring
- Detect quickly changes external to the code that cause it to fail, such as reference data changes

That being so, we can note the following guidelines:

- Testing code is written, as part of TDD, by the developer of the base code, who can identify the relevant database inputs and outputs
- Some, but not necessarily all, test data may be created in a setup step; static reference data that are required for the code to work usually should not be created in setup
- Testing code should be instrumented and logged liberally
- The base code should be timed and a time limit included in the testing; this will help to quickly identify issues such as necessary indexes being dropped
- Consideration should be given to running the unit test suites in performance and other instances

**2025 Note:** The author no longer supports Test Driven Development as a methodology, since, as commonly described, it is more about micro-testing than API testing. His current approach is described in the June 2023 article, [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html).

## ERD of Input and Output Data Structures in Relation to Scenarios

<img src="/migrated_images/2016/05/Unit-Testing-ERD.png" alt="ERD" title="ERD" />

- In the diagram, a scenario corresponds, in the first example, to a web service call with a set of input records
- The result of the call can be described as a set of output groups, each having a set of records
- In the first example the output array and the base table form two output groups, with a global group for average call timing
- The logical diagram in terms of sets of records can be translated into an array structure diagram

<img src="/migrated_images/2016/05/Unit-Testing-ASD.png" alt="ASD" title="ASD" />

If we follow a similarly generic approach at the coding level, it becomes very easy to extend a simple example by adding groups, fields and records as necessary.

## General Unit Test Design Process

The design process involves two high level steps

- Identify a good set of scenarios with corresponding input records
- Identify the expected outputs for each output group identified, for each scenario (there will also be a global group, for timing)

### Scenarios and Subscenarios

#### Scenario definition

We may define a _scenario_ as being the set of all relevant records, both on the database and passed as parameters, to a single program call. API or view testing involves creating one or more scenarios, calling the program (or executing the process) for each scenario, and verifying that the output records are as expected.

Good testing is achieved when the scenarios are chosen to validate as wide a range of behaviours as possible. It is not always, or usually, necessary to create a new scenario for each aspect of behaviour to be tested.

#### Subscenario definition

Often, several features can be tested in the same program call by setting up different records in the scenario that will independently test the different features. For example, in our HR use cases we can create employees with and without a department, and with and without a manager in the same scenario to test the different types of join.

It may be helpful to think of these separate records, or fields within a record, as corresponding to _subscenarios_, and try to construct scenarios as efficiently as possible without making more calls than necessary.

**2025 Note:** In October 2021 I published [Unit Testing, Scenarios and Categories: The SCAN Method](https://brenpatf.github.io/2021/10/17/unit-testing-scenarios-and-categories-the-scan-method.html), which proposes a new approach to scenario selection that tends to result in a larger number of scenarios, with clearer descriptions. Also, since the original series of articles, I have further developed the unit testing design process, and generalised it into a design pattern for any programming language, as described in June 2023, [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html).

## API Testing Antipatterns

Automated unit testing of database APIs is often considered difficult and time-consuming. Unfortunately I believe it is made much worse by widespread following of antipattern approaches, some of which appear on popular database testing-related sites as examples to follow.

Here are a few antipatterns that I have identified, and which my code avoids.

### Test this then test this then...

#### Antipattern description

This occurs when the unit test code is written as a long sequence of method calls, each followed by assertions, and each usually having hard-coded data values in both method calls and assertions. It is a classic antipattern because it is widespread and developers often follow it thinking they are doing the right thing.

#### Antipattern consequence

The testing code becomes very long-winded and hard to follow, and tends to result in less rigorous testing.

#### Pattern alternative

Store all input data in arrays at the start, loop over the input scenarios array, accumulating the outputs in arrays, and loop over the output arrays for the assertions.

### Method-based testing

#### Antipattern description

This occurs when the test suite is based on testing all the methods in a package, rather than units of behaviour, such as a procedure serving a web service.

#### Antipattern consequence

Ian Cooper explains very well in the video link below the adverse consequences of this antipattern from a Java perspective, but it applies equally to database testing.

- It results in a great deal more testing code, with a lot of redundancy, which deters people from the whole concept of test-driven development
- Re-factoring is more difficult because the unit test code tests the implementation
- Shifting the focus of testing from the behavioural side is also unlikely to improve testing quality

[TDD: Where Did It All Go Wrong?](http://www.infoq.com/presentations/tdd-original)

#### Pattern alternative

Include in your test suite only tests of well-defined units of behaviour such as a web service entry point procedure, ignoring helper methods.

### Field-level assertion

#### Antipattern description

This occurs when the individual fields written to the database or in output arrays have their own assertions.

#### Antipattern consequence

Real database applications often have tables with large numbers of fields, and the numbers of assertions can consequently become very large.

#### Pattern alternative

Assert at the record level by concatenating the fields in a record into one string.

### Coupled tests

#### Antipattern description

This occurs when testing one scenario affects another; for example, when all the test data are created at once and not rolled back after each scenario and re-created as needed for the next.

This antipattern is strongly promoted by some popular testing frameworks, where package level setup and teardown are mandatory, at least by default. My own framework deliberately does not support these, preferring the test program to call its own private procedures as necessary at the appropriate level.

#### Antipattern consequence

The coding becomes more complex as it is necessary to distentangle what results from earlier test scenarios from that of the current scenario.

#### Pattern alternative

Set up test data at the scenario level, not at the procedure level, and roll it back at the end of the scenario testing.

### Opaque output

#### Antipattern description

This occurs when the output does not show what was tested.

#### Antipattern consequence

It is harder to review the testing, especially when combined, as it usually is, with the _Test this then test this then..._ antipattern. This results in lower quality.

#### Pattern alternative

Make the output self-documenting with clear scenario descriptions and listings of entities and fields that are being tested.

**2025 Note:** See also this article from June 2023, [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html).
