---
layout: default
---

Hi this is Luke. Bla bla bla.

## Recent posts
<hr />

{% for post in site.posts limit:10 %}
 <p><a href="{{ post.url }}">{{ post.title }} - {{post.date | date: '%B %d %Y'}}</a></p>
{% endfor %}
