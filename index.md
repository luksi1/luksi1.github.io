---
layout: default
---

Hi this is Luke. Bla bla bla.
<hr />

## Recent posts

{% for post in site.posts %}
  <p>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </p>
{% endfor %}
