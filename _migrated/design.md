---
layout: post
title: Design Posts
permalink: /migrated/design/
is_index: true
---

Here are some Oracle eBusiness articles originally published on my WordPress blog:

<ul>
  {% assign posts = site.migrated | where: "group", "design" | sort: "date" | reverse %}
  {% for post in posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})
    </li>
  {% endfor %}
</ul>
