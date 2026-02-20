---
layout: default
title: 文章归档
permalink: /archive/
---

<section class="page">
  <h1>文章归档</h1>

  <div class="archive-toolbar">
    <div class="filter-bar">
      <span class="filter-label">Tag</span>
      {%- for tag in site.tags -%}
        <button class="filter-tag" data-tag="{{ tag[0] }}">{{ tag[0] }}</button>
      {%- endfor -%}
    </div>
    <button id="sort-btn" class="sort-btn" title="切换日期排序">日期 ↓</button>
  </div>

  {%- for section in site.data.sections -%}
    {%- assign sec_posts = site.categories[section.slug] -%}
    {%- if sec_posts.size > 0 -%}
      <div class="archive-section">
        <div class="archive-section-head">
          <span class="archive-section-group">{{ section.group }}</span>
          <a class="archive-section-title" href="{{ section.url | relative_url }}">{{ section.title }}</a>
          <span class="archive-section-count">{{ sec_posts.size }} 篇</span>
        </div>
        <div class="archive">
          {%- for post in sec_posts -%}
            <div class="archive-item" data-tags="{{ post.tags | join: ',' }}">
              <div class="archive-date">{{ post.date | date: "%Y-%m-%d" }}</div>
              <div class="archive-title"><a href="{{ post.url | relative_url }}">{{ post.title }}</a></div>
            </div>
          {%- endfor -%}
        </div>
      </div>
    {%- endif -%}
  {%- endfor -%}
</section>

<script>
(function () {
  /* ── Tag filter ── */
  var activeTag = null;
  document.querySelectorAll('.filter-tag').forEach(function (btn) {
    btn.addEventListener('click', function () {
      var tag = btn.dataset.tag;
      if (activeTag === tag) {
        activeTag = null;
        document.querySelectorAll('.filter-tag').forEach(function (b) { b.classList.remove('active'); });
        document.querySelectorAll('.archive-item').forEach(function (el) { el.hidden = false; });
        document.querySelectorAll('.archive-section').forEach(function (el) { el.hidden = false; });
      } else {
        activeTag = tag;
        document.querySelectorAll('.filter-tag').forEach(function (b) { b.classList.remove('active'); });
        btn.classList.add('active');
        document.querySelectorAll('.archive-item').forEach(function (el) {
          var tags = el.dataset.tags ? el.dataset.tags.split(',') : [];
          el.hidden = tags.indexOf(tag) === -1;
        });
        document.querySelectorAll('.archive-section').forEach(function (sec) {
          var anyVisible = Array.from(sec.querySelectorAll('.archive-item')).some(function (el) { return !el.hidden; });
          sec.hidden = !anyVisible;
        });
      }
    });
  });

  /* ── Date sort ── */
  var sortAsc = false;
  document.getElementById('sort-btn').addEventListener('click', function () {
    sortAsc = !sortAsc;
    this.textContent = sortAsc ? '日期 ↑' : '日期 ↓';
    document.querySelectorAll('.archive').forEach(function (archive) {
      var items = Array.from(archive.children);
      items.reverse().forEach(function (item) { archive.appendChild(item); });
    });
  });
})();
</script>
