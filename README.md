# 犬夜叉的技术博客 (Jekyll + GitHub Pages)

这个仓库是一个可以被 GitHub Pages 直接渲染的 Jekyll 站点。

## 写文章

在 `_posts/` 新建文件：`YYYY-MM-DD-title.md`

示例：

```md
---
layout: post
title: "标题"
date: 2026-02-16 12:00:00 +0800
tags: [jekyll, notes]
---

正文...
```

## 本地预览 (可选)

你的电脑需要安装 Ruby + Bundler。

```bash
bundle install
bundle exec jekyll serve
```

然后访问：`http://localhost:4000`

## 部署到 GitHub Pages

- User Pages: 仓库名为 `<你的GitHub用户名>.github.io`
- Project Pages: 任意仓库名，在仓库 Settings -> Pages 里启用
