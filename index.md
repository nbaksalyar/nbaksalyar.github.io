---
layout: default
title: Inquiries in Software Development
---

# Nikita Baksalyar

## Rust in Detail Series

<ul class="posts">
  {% for post in site.posts reversed %}
    {% if post.tags contains "rustindetail" %}
    <li><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.index_title }}</a> ({{ post.date | date_to_string }})</li>
    {% endif %}
  {% endfor %}
  <li>Part 3: Multithreading and Optimization</li>
  <li>Part 4: Application and Deployment</li>
  <li>Part 5: Secure WebSocket over TLS</li>
</ul>
