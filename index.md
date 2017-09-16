---
layout: default
---

Hi this is Luke. Bla bla bla.
<hr />

## Recent posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
