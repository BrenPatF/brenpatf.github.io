---
layout: post
title: Optimization Problems with Items and Categories in Oracle
date: 2024-06-30
---

<blockquote>The knapsack problem is a problem in combinatorial optimization: Given a set of items, each with a weight and a value, determine the number of each item to include in a collection so that the total weight is less than or equal to a given limit and the total value is as large as possible.</blockquote>

- [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem)

The knapsack problem and many other problems in combinatorial optimization require the selection of a subset of items to maximize an objective function subject to constraints. A common approach to solving these problems algorithmically involves recursively generating sequences of items of increasing length in a search for the best subset that meets the constraints.

I applied this kind of approach using SQL for a number of problems, starting in January 2013 with [A Simple SQL Solution for the Knapsack Problem (SKP-1)](https://aprogrammerwrites.eu/?p=560), and I wrote a summary article, [Knapsacks and Networks in SQL](https://aprogrammerwrites.eu/?p=2232), in December 2017 when I put the code onto GitHub, [sql_demos - Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos).

This 8-part series of articles aims to provide a more formal treatment of algorithms for item sequence generation and optimization, together with practical implementations, examples and verification techniques in SQL and PL/SQL.

<ul>
  <li><a href="/opico/opico-1/">OPICO 1: Algorithms for Item Sequence Generation</a></li>
  <li><a href="/opico/opico-2/">OPICO 2: SQL for Item Sequence Generation</a></li>
  <li><a href="/opico/opico-3/">OPICO 3: Algorithms for Item/Category Optimization</a></li>
  <li><a href="/opico/opico-4/">OPICO 4: Recursive SQL for Item/Category Optimization</a></li>
  <li><a href="/opico/opico-5/">OPICO 5: Tuning Recursive SQL for Item/Category Optimization</a></li>
  <li><a href="/opico/opico-6/">OPICO 6: Mixed SQL and PL/SQL Methods for Item/Category Optimization</a></li>
  <li><a href="/opico/opico-7/">OPICO 7: Verification</a></li>
  <li><a href="/opico/opico-8/">OPICO 8: Automation</a></li>
</ul>

#### GitHub <img src="/images/common/github-mark.png" style="width: 10%; max-width: 5%;"/><br />

- [Optimization Problems with Items and Categories in Oracle](https://github.com/BrenPatF/item_category_optimization_oracle)<br />
[See README for references]

#### Twitter <img src="/images/common/twitter.png" style="width: 10%; max-width: 5%;"/><br />
- [Thread with Short Recordings](https://x.com/BrenPatF/status/1807642673748033675)

<img src="/images/2024/06/30/student-8206673_1280.jpg" style="width: 100%; max-width: 100%;" /><br />
[Image by <a href="https://pixabay.com/users/aviavlad-9412685/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=8206673">Владимир</a> from <a href="https://pixabay.com//?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=8206673">Pixabay</a>]
