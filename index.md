---
layout: default
title: Inquiries in Software Development
---

# Nikita Baksalyar

## Handbook of Async Rust

<p>A <a href="handbook-of-async-rust.html">small, free e-book</a> exploring the topics of asynchronous computation in Rust, starting from the basics.</p>
<p>Coming in Q2 2019.</p>

## Rust in Detail Series (2015)

<ul class="posts">
  {% for post in site.posts reversed %}
    {% if post.tags contains "rustindetail" %}
    <li><a href="{{ BASE_PATH }}{{ post.url }}">{{ post.index_title }}</a> ({{ post.date | date_to_string }})</li>
    {% endif %}
  {% endfor %}
</ul>
