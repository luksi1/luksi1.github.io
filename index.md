---
layout: default
---

Hi this is Luke. Bla bla bla. Bla. Bla.

## Puppet
<hr />
<table>
{% for post in site.categories.Puppet %}
<tr>
<td><span>{{ post.date | date_to_string }}</span></td>
<td><a href="{{ post.url }}">{{ post.title }}</a></td>
</tr>
{% endfor %}
</table>
<br />
## Recent posts
<hr />
<table>
{% for post in site.posts limit:10 %}
<tr>
<td><span>{{ post.date | date_to_string }}</span></td>
<td><a href="{{ post.url }}">{{ post.title }}</a></td>
</tr>
{% endfor %}
</table>
