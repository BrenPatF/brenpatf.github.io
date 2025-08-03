---
layout: post
title: Recursive SQL Posts
permalink: /migrated/recursive-sql/
is_index: true
---

Here are some recursive SQL articles originally published on my WordPress blog:

<ul>
  {% assign posts = site.migrated | where: "group", "recursive" | sort: "date" | reverse %}
  {% for post in posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})
    </li>
  {% endfor %}
</ul>
