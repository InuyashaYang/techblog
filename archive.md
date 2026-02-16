---
layout: default
title: 文章归档
permalink: /archive/
---

<section class="page">
  <h1>文章归档</h1>
  <div class="archive">
    {%- for post in site.posts -%}
      <div class="archive-item">
        <div class="archive-date">{{ post.date | date: "%Y-%m-%d" }}</div>
        <div class="archive-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></div>
      </div>
    {%- endfor -%}
  </div>
</section>
