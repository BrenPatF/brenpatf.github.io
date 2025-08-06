---
layout: default
title: Migrated Articles
permalink: /migrated/
---

<h1>Migrated Articles</h1>

<div>
  <button onclick="showView('index')">🗂 Themes</button>
  <button onclick="showView('all')">📄 All Posts</button>
</div>

<div id="index-view">
  <p>This section contains themed collections of posts from my previous blog:</p>
  <ul>
    <li><a href="/migrated/recursive-sql/">Recursive SQL</a></li>
    <li><a href="/migrated/general-sql/">General SQL</a></li>
    <li><a href="/migrated/ebusiness/">Oracle eBusiness</a></li>
    <li><a href="/migrated/design/">Design</a></li>
    <li><a href="/migrated/perf-testing/">Performance Testing</a></li>
    <li><a href="/migrated/func-testing/">Functional Testing</a></li>
  </ul>
</div>

<div id="all-posts-view" style="display: none;">
  <p>Full list of all migrated articles:</p>
  <ul>
    {% assign sorted_migrated = site.migrated | sort: "date" | reverse %}
    {% assign current_year = "" %}

    {% for post in sorted_migrated %}
      {% unless post.is_index %}
        {% assign post_year = post.date | date: "%Y" %}
        {% if post_year != current_year %}
          {% assign current_year = post_year %}
          </ul><h3>{{ current_year }}</h3><ul>
        {% endif %}

        <li><a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})</li>

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
      {% endunless %}
    {% endfor %}
  </ul>
</div>

<script>
function showView(view) {
  document.getElementById('index-view').style.display = (view === 'index') ? 'block' : 'none';
  document.getElementById('all-posts-view').style.display = (view === 'all') ? 'block' : 'none';
}
</script>
