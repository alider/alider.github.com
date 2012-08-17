---
layout: default
title: Just some implementation details
tagline:
---

<div class="posts">
  {% for post in site.posts %}
    <div>
      <h2><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></h2>
      <div>
        {{ post.content }}
      </div>
    </div>
    <hr/>
  {% endfor %}
</div>