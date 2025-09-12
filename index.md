---
layout: default
---

# I'm J3r3my.

I am exploring cognitive offloading, one article at a time. I use this blog to empty my mind.

## Posts

{% for post in site.posts %}

- [{{ post.title }}]({{ post.url }})
  <small>{{ post.date | date: "%Y-%m-%d" }}</small>
  {% endfor %}
