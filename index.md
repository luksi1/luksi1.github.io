---
layout: default
---

Hi this is Luke. Bla bla bla.
<hr />

## Recent posts
<hr />

{% for post in site.posts limit:10 %}
####<a href="{{ post.url }}">{{ post.title }}</a>
{% endfor %}
