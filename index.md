---
layout: home
---

<div style="font-family: monospace, monospace;">
<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date: "%Y-%m-%d" }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


<footer style="text-align: center;border-top: solid 1px #eaecef;">
    <p>{{ site.time | date: '%Y' }}<a href="/about.html">{{ site.github_username }}&copy;</a></p>
</footer>
</div>
