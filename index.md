---
layout: index
title: Home
---

Hi. My name is Luke Simmons. I'm an American expat raising three kids with my wife in Gothenburg, Sweden. I have a passion for the forest, chantarells, <a http="https://en.wikipedia.org/wiki/Fika_(Sweden)">fika</a>, and programming in the operations sphere. Puppet, Docker, and Ubuntu is the sauce. Perl, Python, Java, C#, PowerShell, Ruby, and Bash is the wisp. 

I've worked in a variety of rolls at Sweden's largest healthcare provider, Västra Götalandsregionen, over the past 10 years. Unix architect. Scrummaster. Database Administrator. Currently I work as an Ops programmer, focusing on ESB solutions.

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
