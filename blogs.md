---
layout: page
title: Blogs
permalink: /blogs/
---

<!-- Loop through all blog posts -->
{% for post in site.blogs %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <p class="author">
    <span class="date">{{ post.date | date_to_string }}</span>
  </p>
{% endfor %}
