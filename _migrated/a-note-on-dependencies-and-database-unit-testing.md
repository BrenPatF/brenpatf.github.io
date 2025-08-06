---
layout: post
title: "A Note on Dependencies and Database Unit Testing"
date: 2016-08-06
migrated: true
group: func-testing
categories: 
  - "object"
  - "oracle"
  - "plsql"
  - "testing"
  - "utplsql"
tags: 
  - "fowler"
  - "oop"
  - "oracle"
  - "plsql"
  - "unit-test"
  - "utplsql"
---
**Note, 2025:** See [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html), from June 2023, for the author's current views on unit testing.

Ideas on unit testing for the database are often heavily influenced by the world of object oriented programming (OOP), usually Java in practice. This is no doubt because much of modern thinking on development methodologies, including test driven development (TDD), originated in this world. Some of these ideas appear to translate very well into the database world, including that of TDD itself, with automated unit tests. However, some ideas may not translate so well, or even make sense, in database unit testing. For example, Roy Osherove (2011), [Unit Test - Definition](http://artofunittesting.com/definition-of-a-unit-test/) says:

> A good unit test ... runs in memory (no DB or File access, for example)

One concept that appears very important in the OOP world is that of dependencies, and of isolation of the code under test from its dependencies. This gives rise to complex mechanisms of 'mocking' and 'dependency injection' to bring about said isolation. Osherove mentions isolation in the same article as a requirement of good unit testing, and his view appears to be widespread. It's worth mentioning though that not everyone in the OOP world shares his insistence. The influential Martin Fowler (2014) uses a nice terminology of 'sociable' tests (as opposed to 'isolated' tests) for tests that rely on other units to fulfill the behaviour under test, and he uses this approach himself when practicable, [UnitTest](http://martinfowler.com/bliki/UnitTest.html).

In the case of database unit testing, it seems to me to make very little sense to think in terms of isolating code under test from its dependencies. The following two diagrams represent how I see the relationships between base code, dependencies and unit test code across two distinct phases.

## Development Phase
<img src="/migrated_images/2016/08/UT-Phases-dev.png" alt="UT Phases-dev" title="UT Phases-dev" />

## Regression Phase
<img src="/migrated_images/2016/08/UT-Phases-reg.png" alt="UT Phases-reg" title="UT Phases-reg" />
