---
layout: default
title: Migrated Articles
permalink: /migrated/
---

# Migrated Articles

{% assign sorted = site.migrated | sort: 'date' | reverse %}

<ul>
  {% for post in sorted %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})</li>
  {% endfor %}
</ul>
