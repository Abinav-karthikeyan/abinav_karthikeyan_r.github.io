---
layout: page
title: Blog
permalink: /blog/
---

Notes on the engineering side of AI — mostly agentic systems, retrieval, knowledge graphs, and the unglamorous work of getting any of it into production. Written to think out loud, not to conclude.

<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    {% if post.excerpt %}
    <p>{{ post.excerpt | strip_html | truncatewords: 40 }}</p>
    {% endif %}
  </li>
  {% endfor %}
</ul>
