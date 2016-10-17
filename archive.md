---
title: Archive
---

<ul>
  {%for post in site.posts %}
  <li><time>{{ post.date | date:"%Y.%m.%d" }}</time> - <a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
