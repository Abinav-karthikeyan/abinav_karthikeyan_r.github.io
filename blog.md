---
layout: page
title: Blog
permalink: /blog/
---

{% if site.posts.size == 0 %}
No posts yet — check back soon.
{% else %}
<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    {% if post.excerpt %}
    <p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
    {% endif %}
  </li>
  {% endfor %}
</ul>
{% endif %}
