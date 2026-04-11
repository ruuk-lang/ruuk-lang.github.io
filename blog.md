---
layout: default
title: Blog — Ruuk
permalink: /blog
---

<div class="post-list">
  <div class="post-list-heading">posts</div>
  {% for post in site.posts %}
  <a href="{{ post.url }}" style="text-decoration:none">
    <div class="post-card">
      <h3>{{ post.title }}</h3>
      {% if post.excerpt %}<p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>{% endif %}
      <div class="post-meta">{{ post.date | date: "%B %d, %Y" }}</div>
    </div>
  </a>
  {% endfor %}
</div>
