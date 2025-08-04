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
    {% for post in sorted_migrated %}
      {% unless post.is_index %}
        <li><a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})</li>
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
