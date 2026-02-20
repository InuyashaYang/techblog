---
layout: default
title: 搜索
permalink: /search/
---

<section class="page">
  <h1>搜索</h1>
  <div class="search-bar">
    <input id="search-input" type="text" placeholder="输入标题、标签或摘要…" autocomplete="off" spellcheck="false">
  </div>
  <div id="search-status" class="search-hint">输入关键词以搜索文章</div>
  <div id="search-results" class="post-grid" style="margin-top:16px;"></div>
</section>

<script>
(function () {
  var SEARCH_URL = {{ '/search.json' | relative_url | jsonify }};
  var input = document.getElementById('search-input');
  var status = document.getElementById('search-status');
  var results = document.getElementById('search-results');
  var posts = null;

  function load() {
    if (posts !== null) return Promise.resolve(posts);
    return fetch(SEARCH_URL).then(function (r) { return r.json(); }).then(function (data) {
      posts = data;
      return posts;
    });
  }

  function esc(s) {
    return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');
  }

  function render(matches) {
    if (!matches.length) {
      status.textContent = '没有匹配的文章';
      results.innerHTML = '';
      return;
    }
    status.textContent = '找到 ' + matches.length + ' 篇';
    results.innerHTML = matches.map(function (p) {
      var tags = (p.tags || []).slice(0, 3).map(function (t) {
        return '<span class="tag">' + esc(t) + '</span>';
      }).join('');
      return '<article class="post-card">'
        + '<div class="post-date">' + esc(p.date) + '</div>'
        + '<h3 class="post-title"><a href="' + esc(p.url) + '">' + esc(p.title) + '</a></h3>'
        + '<p class="post-excerpt">' + esc(p.desc) + '</p>'
        + '<div class="post-meta"><div class="tags">' + tags + '</div>'
        + '<a class="read" href="' + esc(p.url) + '">阅读</a></div>'
        + '</article>';
    }).join('');
  }

  input.addEventListener('input', function () {
    var q = input.value.trim().toLowerCase();
    if (!q) {
      status.textContent = '输入关键词以搜索文章';
      results.innerHTML = '';
      return;
    }
    load().then(function (all) {
      var matches = all.filter(function (p) {
        return p.title.toLowerCase().indexOf(q) !== -1
          || (p.desc || '').toLowerCase().indexOf(q) !== -1
          || (p.tags || []).some(function (t) { return t.toLowerCase().indexOf(q) !== -1; });
      });
      render(matches);
    });
  });

  input.focus();
})();
</script>
