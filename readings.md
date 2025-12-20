---
layout: page
title: Readings
permalink: /readings/
---

<!-- Loop through all readings -->
{% assign sorted_readings = site.readings | sort: 'date' | reverse %}
{% for post in sorted_readings %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <p class="author">
    <span class="date">{{ post.date | date_to_string }}</span>
  </p>
{% endfor %}
