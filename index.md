---
layout: default
title: Inquiries in Software Development
---

# Nikita Baksalyar

## Inquiries in Software Development

  <ul class="posts">
  {% for post in site.posts %}
    <li><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a> ({{ post.date | date_to_string }})</li>
  {% endfor %}
  </ul>
