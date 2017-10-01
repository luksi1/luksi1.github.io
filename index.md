---
layout: index
title: Home
---
<ul class="posts-list">
{% for post in site.posts limit:10 %}
<li>
  <h3>
    <a href="{{ post.url }}">{{ post.title }}
    <small>{{ post.date | date_to_string }}</small><a>
  </h3>
</li>
{% endfor %}
</ul>

## Puppet
<hr />
<table>
{% for post in site.categories.Puppet %}
<tr>
<td><span>{{ post.date | date: "%Y-%m-%d" }}</span></td>
<td><a href="{{ post.url }}">{{ post.title }}</a></td>
</tr>
{% endfor %}
</table>
