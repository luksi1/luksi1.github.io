---
layout: default
---

Hi this is Luke. Bla bla bla.

## Puppet

{% for post in site.categories.Puppet %}
 <li><span>{{ post.date | date_to_string }}</span> &nbsp; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}

## Recent posts
<hr />

{% for post in site.posts limit:10 %}
 <p><a href="{{ post.url }}">{{ post.title }} - {{post.date | date: '%B %d %Y'}}</a></p>
{% endfor %}
