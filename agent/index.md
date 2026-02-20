---
layout: default
title: Agent
permalink: /agent/
---

<section class="page">
  <h1>Agent</h1>
  <p class="cat-desc">AI 编程助手专题：涵盖 Claude Code、Codex、OpenCode、OpenClaw 的使用、配置与深度记录。</p>

  <div class="topic-cards" style="margin-top: 24px;">
    {%- for section in site.data.sections -%}
      {%- if section.group == "Agent" -%}
        {%- assign sec_posts = site.posts | where_exp: "post", "post.categories contains section.slug" -%}
        <a class="topic-card" href="{{ section.url | relative_url }}">
          <div class="topic-title">{{ section.title }}</div>
          <p class="topic-desc">{{ section.desc }}</p>
          <div class="topic-count">
            {%- if sec_posts.size > 0 -%}
              {{ sec_posts.size }} 篇
            {%- else -%}
              即将上线
            {%- endif -%}
          </div>
        </a>
      {%- endif -%}
    {%- endfor -%}
  </div>
</section>
