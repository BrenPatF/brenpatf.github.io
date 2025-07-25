---
layout: post
title: "Messages from Outer Space"
date: 2013-10-15
migrated: true
---

I began my career in IT in 1984, before the internet and even before widespread email availability. In those days one learnt about technology and other subjects largely through books and papers, training courses, and one's immediate colleagues. Since then the rise of the internet has transformed the learning process, vastly expanding the material available.

In its earlier years users could be roughly divided into a large group of consumers and a much smaller group of producers; later, and increasingly in recent years, the emergence of what is sometimes called _'Web 2'_ has enabled many consumers to become producers as well. As Wikipedia (itself a prime example of the phenomenon), [Web 2.0](http://en.wikipedia.org/wiki/Web_2.0), puts it:

_"A Web 2.0 site may allow users to interact and collaborate with each other in a social media dialogue as creators of user-generated content in a virtual community..."_

For IT developers and other technical people this is hugely important because it expands the pool of available expertise in one's field from a few colleagues and a relatively small group of publishers to potentially everyone working in the field. This means that good ideas and best practices quickly become available to all. Where in the past bad practices or _antipatterns_ could become entrenched in a company through the influence of a small number of people sharing a bad idea, today we can check what the rest of the world thinks. It's a bit like the old scifi motif in which an advanced civilisation comes to share its ideas with Planet Earth :).

<img src="/migrated_images/2013/10/et.jpg" alt="et" title="et" style="max-width: 800px; width: 100%; height: auto;" />

However, great though this is, it's necessary to make the effort to keep up with best practice in one's own field, and not all developers do that. In my main field, Oracle database development, the resources I find most useful are:

- [Ask Tom](http://asktom.oracle.com/pls/apex/f?p=100:1:0) - Oracle questions answered by Tom Kyte with comments from the general Oracle community
- [OTN: SQL and PL/SQL](https://forums.oracle.com/ords/apexds/domain/dev-community/category/sql_and_pl_sql) - Oracle SQL and PL/SQL questions answered by the general Oracle community
- [OTN: Oracle General Questions](https://forums.oracle.com/ords/apexds/domain/dev-community/category/3063-general_questions) - Oracle general questions answered by the general Oracle community
- Oracle Blogs, such as:
    
    - [Rob van Wijk's Blog](http://rwijk.blogspot.com/)
    
    - [Adrian Billington's Blog](http://www.oracle-developer.net/)
- [Twitter](https://twitter.com/) - an excellent medium for learning about new articles and blog posts
- [Twitter](https://twitter.com/) - an excellent medium for learning about new articles and blog posts

**This article was written in November 2010 on Wordpress, and on migrating it to GitHub Pages in July 2025, we can add this line (itself generated by ChatGPT):**

- AI tools, including ChatGPT, can be valuable learning aids, providing immediate explanations, tailored examples, and interactive support for self-directed study

As well as keeping up with the general Oracle community, developers should ideally run their own blog to maximise learning.

The remainder of this article consists of useful links concerning best (or worst) practices that I deem significant in database development, largely in areas where people commonly get it wrong. This article will be an ongoing work in progress.

## Database Design

[Database Design and the Leaning Tower of Pisa](http://rodgersnotes.wordpress.com/2010/09/14/database-design-mistakes-to-avoid/)

This article by Rodger Lepinsky notes that database table designs are usually done very quickly and tend not to be re-worked as the application is developed to avoid impacting developers. The problem is that this apparently justifiable reluctance to re-work often results in much additional code to work around inevitable deficiencies in the initial data model, and ultimately a much more complex system: It's short-sighted in other words.

Here's a nice cartoon illustrating a common database design antipattern taken to its logical conclusion :)

[The EAV Data Model](http://static.squarespace.com/static/518f5d62e4b075248d6a3f90/t/51edaab6e4b008f85ce59859/1374530255950/gdm.jpg?format=1000w)

## Design Consistency

Rodger refers to database designs becoming frozen too early in the article mentioned in the previous section. I believe this is a design antipattern that occurs very widely in general. Often a new technology is applied for a new project, one that the developers may not know very well at first; similarly, custom frameworks are often developed, in Unix or Perl, or whatever, and the initial production versions are rarely perfect. However, once something works, the tendency is to freeze it, with all its limitations, for fear of unforeseen impacts, or even purely from the notion that consistency is more important than quality.

I think that this excessive emphasis on 'consistency' is one of the main reasons that so many over-complex, poor quality systems persist indefinitely. It might be better to focus on 'consistently' applying best practice as currently understood.

## Data Access Layers

[Considering SQL as a Service](http://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:672724700346558185)

The idea of 'modularising' SQL through data access layers is one that comes up frequently on Oracle forums, including the Ask Tom thread above, and is well known to be an antipattern. I deal with it in a wider context here:

[SQL and Modularity: Patterns, Anti-Patterns and the Kitchen Sink](https://brenpatf.github.io/migrated/modularity-in-sql-patterns-anti-patterns-and-the-kitchen-sink/)

Here is another relevant AskTom thread:

[Multi-Level Views](http://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:7082182300346634611)

_"I prefer a SINGLE LEVEL of views. Yes, there will be some repetition in their definition but I don't care about that. You can document that. You can maintain that"_

## ETL vs ELT

[ETL - Using the wrong tool for the job](http://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:60721784666295)

Companies often use a mix of ETL and ELT with Informatica and Oracle, among other tools. Here is Tom Kyte's pithy summation:

_"elt = extract, load and then transform (forget any tools, if you have hundreds of gb's or tb's of data - you'll be doing this down to the wire, not with pretty pictures and push buttons)_

_etl = extract, transform and then load - without using the database to transform_

_elt = going light speed_

_etl = going by boat"_

## Object Relational Madness (ORM)

[Object-Relational Mapping is the Vietnam of Computer Science](https://blog.codinghorror.com/object-relational-mapping-is-the-vietnam-of-computer-science/)
