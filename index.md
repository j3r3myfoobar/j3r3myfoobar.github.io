---
layout: page
navigation: true
---

## Posts

{% for post in site.posts %}

- [{{ post.title }}]({{ post.url }})
  <small>{{ post.date | date: "%Y-%m-%d" }}</small>
  {% endfor  %}
