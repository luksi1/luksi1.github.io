---
layout: default
---

Hi this is Luke. Bla bla bla.

## Puppet

<table>
{% for post in site.categories.Puppet %}
 <tr>
 <td><span>{{ post.date | date_to_string }}</span></td>
 <td><a href="{{ post.url }}">{{ post.title }}</a></td>
 </tr>
{% endfor %}
</table>

## Recent posts
<hr />

{% for post in site.posts limit:10 %}
 <p><a href="{{ post.url }}">{{ post.title }} - {{post.date | date: '%B %d %Y'}}</a></p>
{% endfor %}
