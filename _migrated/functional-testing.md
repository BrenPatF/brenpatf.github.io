---
layout: post
title: Functional Testing Posts
permalink: /migrated/func-testing/
is_index: true
---

Here are some Oracle functional testing articles originally published on my WordPress blog:

<ul>
  {% assign posts = site.migrated | where: "group", "func-testing" | sort: "date" | reverse %}
  {% for post in posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})
    </li>
  {% endfor %}
</ul>
