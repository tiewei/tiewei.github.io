---
layout: home
---

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

---
<footer>
    <p>&copy; {{ site.time | date: '%Y' }}
      <a href="/about.html">{{ site.github_username }}</a>
      All rights reserved.
    </p>
</footer>
