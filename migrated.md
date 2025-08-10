---
layout: default
title: Migrated Articles
permalink: /migrated/
---

<div>
  <button onclick="showView('index')">📚 Themes</button>
  <button onclick="showView('all')">🗓 Years</button>
</div>

{% assign total_posts = site.migrated | size %}
<div id="index-view">
  <h2>{{ total_posts }} articles from my previous blog grouped by theme</h2>

  {% assign themes = "func-testing,perf-testing,design,recursive-sql,general-sql,ebusiness" | split: "," %}

  {% assign theme_names = 
    "recursive-sql:Recursive SQL,
     general-sql:General SQL,
     ebusiness:Oracle eBusiness,
     design:Design,
     perf-testing:Performance Testing,
     func-testing:Functional Testing" | split: "," 
  %}
  
  {% for theme in themes %}
    {% assign theme_posts = site.migrated | where: "group", theme | sort: "date" | reverse %}
  
    {% if theme_posts.size > 0 %}
      {% assign theme_name = theme %}
      {% for mapping in theme_names %}
        {% assign parts = mapping | strip | split: ":" %}
        {% if parts[0] == theme %}
          {% assign theme_name = parts[1] %}
        {% endif %}
      {% endfor %}
  
      <h3>{{ theme_name }} ({{ theme_posts.size }})</h3>
      <ul>
        {% for post in theme_posts %}
          {% unless post.is_index %}
            <li>
              <a href="{{ post.url }}">{{ post.title }}</a>
              ({{ post.date | date: "%Y-%m-%d" }})
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
          {% endunless %}
        {% endfor %}
      </ul>
    {% endif %}
  {% endfor %}
</div>

<div id="all-posts-view" style="display: none;">
  <h2>{{ total_posts }} articles from my previous blog grouped by year</h2>
    {% assign posts_by_year = site.migrated | group_by_exp: "post", "post.date | date: '%Y'" %}
    {% assign sorted_years = posts_by_year | sort: "name" | reverse %}
    
    {% comment %}
      Separate 2017–2023 and older/newer years
    {% endcomment %}
    {% assign recent_years = "" | split: "" %}
    {% assign older_years = "" | split: "" %}
    
    {% for year_group in sorted_years %}
      {% assign year_num = year_group.name | plus: 0 %}
      {% if year_num >= 2017 and year_num <= 2023 %}
        {% assign recent_years = recent_years | push: year_group %}
      {% else %}
        {% assign older_years = older_years | push: year_group %}
      {% endif %}
    {% endfor %}
    
    {%- comment -%}
      Show combined heading for 2017–2023
    {%- endcomment -%}
    {% if recent_years.size > 0 %}
      {% assign recent_total = 0 %}
      {% for yr in recent_years %}
        {% assign recent_total = recent_total | plus: yr.items.size %}
      {% endfor %}
    
      <h3>2017–2023 ({{ recent_total }})</h3>
        <ul>
      {% for yr in recent_years %}
        {% assign yr_num = yr.name %}
          {% assign sorted_posts = yr.items | sort: "date" | reverse %}
          {% for post in sorted_posts %}
            <li><a href="{{ post.url }}">{{ post.title }}</a> ({{ post.date | date: "%Y-%m-%d" }})</li>
          {% endfor %}
      {% endfor %}
        </ul>
    {% endif %}
    
    {%- comment -%}
      Show each other year individually
    {%- endcomment -%}
    {% for yr in older_years %}
      <h3>{{ yr.name }} ({{ yr.items.size }})</h3>
      <ul>
        {% assign sorted_posts = yr.items | sort: "date" | reverse %}
        {% for post in sorted_posts %}
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
        {% endfor %}
      </ul>
    {% endfor %}
</div>

<script>
function showView(view) {
  document.getElementById('index-view').style.display = (view === 'index') ? 'block' : 'none';
  document.getElementById('all-posts-view').style.display = (view === 'all') ? 'block' : 'none';
}
</script>
