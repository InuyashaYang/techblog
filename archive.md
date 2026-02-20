---
layout: default
title: 文章归档
permalink: /archive/
---

<section class="page">
  <h1>文章归档</h1>

  {%- for section in site.data.sections -%}
    {%- assign sec_posts = site.posts | where_exp: "post", "post.categories contains section.slug" -%}
    {%- if sec_posts.size > 0 -%}
      <div class="archive-section">
        <div class="archive-section-head">
          <span class="archive-section-group">{{ section.group }}</span>
          <a class="archive-section-title" href="{{ section.url | relative_url }}">{{ section.title }}</a>
          <span class="archive-section-count">{{ sec_posts.size }} 篇</span>
        </div>
        <div class="archive">
          {%- for post in sec_posts -%}
            <div class="archive-item">
              <div class="archive-date">{{ post.date | date: "%Y-%m-%d" }}</div>
              <div class="archive-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></div>
            </div>
          {%- endfor -%}
        </div>
      </div>
    {%- endif -%}
  {%- endfor -%}
</section>
