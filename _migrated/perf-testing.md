---
layout: post
title: Performance Testing Posts
permalink: /migrated/perf-testing/
is_index: true
---

Here are some Oracle performance testing articles originally published on my WordPress blog:

<ul>
  {% assign posts = site.migrated | where: "group", "perf-testing" | sort: "date" | reverse %}
  {% for post in posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})
    </li>
  {% endfor %}
</ul>
