---
layout: index
title: Home
---
<ul class="posts-list">
{% for post in site.posts limit:10 %}
<li>
  <h3>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <small>{{ post.date | date_to_string }}</small>
  </h3>
</li>
{% endfor %}
</ul>

