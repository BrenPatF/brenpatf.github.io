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
    {% if post.index %}
      {% assign collection_name = post.index %}
      {% assign collection = site[collection_name] %}
    
      {% if collection %}
        <ul style="margin-left: 1.5em; margin-top: 0.5em;">
          {% for child in collection %}
            <li>
              <a href="{{ child.url }}">{{ child.title }}</a>
              ({{ child.date | date: "%Y-%m-%d" }})
            </li>
          {% endfor %}
        </ul>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>
