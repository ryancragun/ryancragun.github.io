---
layout: page
title: Article Index
description: Previously published articles
category: base
---

<section id="archive">
  <ul class="post-list">
    {%for post in site.posts %}
      <li><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
</section>
