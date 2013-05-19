---
layout: page
title: Tech Rest
tagline: Tie Wei's blog
---

{% include JB/setup %}

Posts list 

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


