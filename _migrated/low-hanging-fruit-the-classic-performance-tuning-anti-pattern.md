---
layout: post
title: "Low-Hanging Fruit: The Classic Performance Tuning Anti-pattern"
date: 2014-07-28
migrated: true
categories: 
  - "oracle"
  - "performance"
  - "plsql"
  - "sql"
tags: 
  - "anti-pattern"
  - "oracle"
  - "performance-2"
  - "plsql"
  - "profiling"
  - "sql"
  - "tuning"
---

The July/August 2014 edition of Oracle Magazine contains an interesting PL/SQL performance tuning article by Steve Feuerstein, [The Joy of Low-Hanging Fruit](https://asktom.oracle.com/Misc/oramag/the-joy-of-low-hanging-fruit.html).

<figure>
  <img src="/migrated_images/2014/07/cherries.jpg" alt="Low-Hanging Cherries" title="Low-Hanging Cherries" />
  <figcaption>Low-Hanging Cherries</figcaption>
</figure>

The article is a case study in which SF finds a 
> cursor FOR loop that contained two nonquery DML statements

and which he calls a _classic anti-pattern_. As he says:
> the inserts and updates are changing the tables on a row-by-row basis, which maximizes the number of context switches.

He also identifies an inefficiency in the code that is more problem-specific: An update process occurs within the loop that only needs to be performed at the end of the loop. (Although this is a generic kind of performance 'bug' I would resist the temptation to call it an _anti-pattern_, preferring to reserve the term for more deliberate strategies). The article goes on to replace the row-by-row DML using bulk processing, and to move the in-loop update outside the loop. SF claims a two to three-fold performance improvement in testing.

The performance improvement sounds good, but at the same time it occurs to me that the article itself could be seen as an illustration of the **classic performance-tuning anti-pattern**. This is when the tuner looks for things to optimise before he or she knows what is actually taking up the bulk of the time (and the article's title summarises it nicely - so I will use it to name the _anti-pattern_ :) ). The main thrust of the article, as indicated in the subtitle, is the performance benefits of BULK COLLECT, but there is actually no indication that this is where the performance gains were made - it's possible, likely even, that all significant gains came from the other, more problem-specific, improvement.

The first paragraph of the final section of the article reads:

> It might be lots of fun to completely reorganize one's program in hopes of improving performance, but we can't just assume that the resulting code actually does run quickly.

This is exactly right, and is why the first step in any performance tuning process should be to identify the hot-spots. Tuning should then focus on those, ignoring areas where theoretical improvements will make no practical difference. If one might be permitted a second sylvan metaphor, it's important to perceive wood as well as trees.

<figure>
  <img src="/migrated_images/2014/07/The_Wood_of_the_Self-Murderers.jpg" alt="William Blake illustration of Dante's Divine Comedy" title="William Blake illustration of Dante's Divine Comedy" />
  <figcaption>The Wood of the Self-Murderers, by William Blake</figcaption>
</figure>

There are many ways to identify the hot-spots, and I would recommend Oracle's PL/SQL profiling features, which I wrote about here, [PL/SQL Profiling 1 – Overview](https://brenpatf.github.io/migrated/notes-on-oracles-hierarchical-plsql-profiler/).

## Notes on the Metaphors

### Low-Hanging Fruit

For such a ubiquitous expression, this seems to be of surprisingly recent origin, according to: [What Is the World’s Actual Lowest Hanging Fruit?](http://www.psmag.com/navigation/business-economics/actual-lowest-hanging-fruit-85491/).

> The metaphorical usage doesn’t appear to have a very long history, though. According to Liberman, the Oxford English Dictionary’s first partial reference is from a 1968 Guardian article: “His rare images are picked aptly, easily, like low-hanging fruit.”

### Not Seeing the Wood for the Trees

This expression is apparently much older, being traced back to (at least) 1546 here, [Can't see the Economist for the Trees](http://www.maths.tcd.ie/~dwmalone/p/economist02.html), where it is attributed to "The Proverbs of John Heywood" and the lines:

> Plentie is no deyntie. ye see not your owne ease. I see, ye can not see the wood for trees.

The article is ostensibly about an incorrect usage of the expression by The Economist, but is really more interesting than that, and includes equivalents in other languages. For example, Americans say:

> Can't see the forest for the trees

while the French say:

> L'arbre qui cache la forêt

### The Wood of the Self-Murderers

This intriguing painting hangs at the Tate Gallery in London, and you can read more about it here, [The Wood of the Self-Murderers: The Harpies and the Suicides](http://en.wikipedia.org/wiki/The_Wood_of_the_Self-Murderers:_The_Harpies_and_the_Suicides)

> The work was completed between 1824 and 1827 and illustrates a passage from the Inferno canticle of the Divine Comedy by Dante Alighieri (1265–1321)

The passage concerned is:

> Here the repellent harpies make their nests, Who drove the Trojans from the Strophades With dire announcements of the coming woe. They have broad wings, a human neck and face, Clawed feet and swollen, feathered bellies; they caw Their lamentations in the eerie trees
