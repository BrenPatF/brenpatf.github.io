---
layout: post
title: "Knapsacks and Networks in SQL"
date: 2017-12-03
migrated: true
group: recursive-sql
categories: 
  - "analytics"
  - "oracle"
  - "performance"
  - "pipelined"
  - "plsql"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
tags: 
  - "analytics"
  - "bin-fitting"
  - "combinatorial"
  - "hierarchical"
  - "network"
  - "oracle"
  - "performance-2"
  - "qsd"
  - "recursive"
  - "sql"
  - "subquery-factor"
  - "tuning"
---

I opened a GitHub account, [Brendan's GitHub Page](https://github.com/BrenPatF) last year and have added a number of projects since then, in PL/SQL and other 3GL languages. Partly in response to a request for the code for one of my blog articles on an interesting SQL problem, I decided recently to create a new repo for the SQL behind a group of articles on solving difficult combinatorial optimisation problems via 'advanced' SQL techniques such as recursive subquery factoring and model clause, [sql\_demos - Brendan's repo for interesting SQL](https://github.com/BrenPatF/sql_demos). It includes installation scripts with object creation and data setup, and scripts to run the SQL on the included datasets. The idea is that anyone with the pre-requisites should be able to reproduce my results within a few minutes of downloading the repo.

<img src="/migrated_images/2017/12/Knapsack_Network.png" />
\[Left image from [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem); right image copied from [Chapter 11 Dynamic Programming](http://www.ime.unicamp.br/~andreani/MS515/capitulo7.pdf)\]

In this article I embed each of the earlier articles relevant to the GitHub repo with a brief preamble.

#### [A Simple SQL Solution for the Knapsack Problem (SKP-1)](https://brenpatf.github.io/migrated/a-simple-sql-solution-for-the-knapsack-problem/)

The first two articles are from January 2013 and use recursive subquery factoring to find exact solutions for the single and multiple knapsack problem, and also include PL/SQL solutions for comparison. They avoid the 'brute force' approach by truncating search paths as soon as limit constraints are exceeded. The cumulative paths are stored in string variables passed through the iterations (which would not be possible with the older Connect By hierarchical syntax).

In these articles I illustrate the nature of the problems using Visio diagrams, and include dimensional performance benchmarking results, using a technique that I presented on at last year's Ireland OUG conference: [Dimensional Performance Benchmarking of SQL - IOUG Presentation](http://www.slideshare.net/brendanfurey7/dimensional-performance-benchmarking-of-sql). I also illustrate the queries using my own method for diagramming SQL queries.
<iframe src="https://brenpatf.github.io/migrated/a-simple-sql-solution-for-the-knapsack-problem/" width="100%" height="570" scrolling="yes"></iframe>

<br />
#### [An SQL Solution for the Multiple Knapsack Problem (SKP-m)](https://brenpatf.github.io/migrated/an-sql-solution-for-the-multiple-knapsack-problem-skp-m/)

<iframe src="https://brenpatf.github.io/migrated/an-sql-solution-for-the-multiple-knapsack-problem-skp-m/" width="100%" height="705" scrolling="yes"></iframe>

<br />
#### [SQL for the Balanced Number Partitioning Problem](https://brenpatf.github.io/migrated/sql-for-the-balanced-number-partitioning-problem/)
The next article uses Model clause to find a more general solution to a problem posed on AskTom, as a 'bin fitting' problem. I also solved the problem by other methods including recursive subquery factoring. I illustrate the problem itself, as well as the Model iteration scheme using Visio diagrams, and again include dimensional performance benchmarking. The results show how quadratic performance variation can be turned into much faster linear variation by means of a temporary table in this kind of problem.

<iframe src="https://brenpatf.github.io/migrated/sql-for-the-balanced-number-partitioning-problem/" width="100%" height="750" frameborder="0" scrolling="yes"></iframe>

<br />
#### [Optimization Problems with Items and Categories in Oracle](http://brenpatf.github.io/2024/06/30/opico-series-index.html)
The original article arose from a question on OTN, and concerns a type of knapsack or bin-fitting problem that is quite tricky to solve in SQL, where the items fall into categories on which there are separate constraints. I introduced a new idea here, to filter out unpromising paths within recursive subquery factoring by means of analytic functions, in order to allow the technique to be used to generate solutions for larger problems without guaranteed optimality, but in shorter time. Two realistic datasets were used, one from the original poster, and another I got from a scraping website.

**Note, 2025:** The original article, from June 2013, was entitled "SQL for the Fantasy Football Knapsack Problem", and was much narrower in scope than the 2024 series of articles.
<iframe src="http://brenpatf.github.io/2024/06/30/opico-series-index.html" width="100%" height="930" frameborder="0" scrolling="yes"></iframe>
<br />
#### [SQL for the Travelling Salesman Problem](https://brenpatf.github.io/migrated/sql-for-the-travelling-salesman-problem/)
This article is on a classic 'hard' optimisation problem, and uses recursive subquery factoring with the same filtering technique as the previous article, and shows that it's possible to solve a problem involving 312 American cities quite quickly in pure SQL using the approximation technique. It also uses a simple made-up example dataset to illustrate its working.

<iframe src="https://brenpatf.github.io/migrated/sql-for-the-travelling-salesman-problem/" width="100%" height="580" frameborder="0" scrolling="yes"></iframe>
<br />
#### [SQL for Shortest Path Problems](https://brenpatf.github.io/migrated/sql-for-shortest-path-problems/)
The following two articles concern finding shortest paths between given nodes in a network, and arose from a question on OTN. The first one again uses recursive subquery factoring with a filtering mechanism to exclude paths as early as possible, in a similar way to the approximative solutios methods in the earlier articles. In this case, however, reasoning about the nature of the problem shows that we are not in fact sacrificing optimality. The article has quite a lot of explanatory material on how the SQL works, and uses small dataset examples.

<iframe src="https://brenpatf.github.io/migrated/sql-for-shortest-path-problems/" width="100%" height="660" frameborder="0" scrolling="yes"></iframe>
<br />
#### [SQL for Shortest Path Problems 2: A Branch and Bound Approach](https://brenpatf.github.io/migrated/sql-for-shortest-path-problems-2-a-branch-and-bound-approach/)
The second article considers how to improve performance further by obtaining a preliminary approximate solution that can be used as a bounding mechanism in a second step to find the exact solutions. This article uses two realistic networks as examples, including one having 428,156 links.

<iframe src="https://brenpatf.github.io/migrated/sql-for-shortest-path-problems-2-a-branch-and-bound-approach/" width="100%" height="875" frameborder="0" scrolling="yes"></iframe>

<br />
#### [Recursive SQL for Network Analysis, and Duality](https://brenpatf.github.io/migrated/recursive-sql-for-network-analysis-and-duality/)
In the article above I cited results from a general network analysis package I had developed that obtains all the distinct connected subnetworks with their structures in an efficient manner using PL/SQL recursion. It is worth noting that for that kind of problem recursive SQL alone is very inefficient, and I wrote the following article to try to explain why that is so, and why the Connect By syntax is generally much worse than recursive subquery factoring.

<iframe src="https://brenpatf.github.io/migrated/recursive-sql-for-network-analysis-and-duality/" width="100%" height="930" frameborder="0" scrolling="yes"></iframe>

<br />
#### [PL/SQL Pipelined Function for Network Analysis](https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/)

<iframe src="https://brenpatf.github.io/migrated/plsql-pipelined-function-for-network-analysis/" width="100%" height="710" frameborder="0" scrolling="yes"></iframe>

