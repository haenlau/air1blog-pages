---
title: Hexo修改配置文件允许文章内嵌入html
date: 2025-08-16 10:03:56
tags:
  - Hexo
---


想着丰富下文章，实在不知道写什么，又偶然看见腾讯微博的备份记录，打开看看不禁让人感叹时间飞快恍如昨日。

遂计划将备份记录作为一篇文章上传至 air1 ，不过 Hexo 框架兴许为了安全考虑并没有使能 md 文章内嵌 HTML ，需要修改`_config.yml`文件才能实现。

修改配置文件后，直接复制html代码进入md文件后，又存在内容被转义或导致全站背景/布局异常等问题。以下是解决过程。


1. 启用 Markdown 中的 HTML 渲染

在站点根目录的`_config.yml`中，找到或添加 marked 配置项，并确保开启 HTML 支持：

```yaml
# 启用 Markdown 中的 HTML 渲染
marked:
  gfm: true
  breaks: true
  html: true          # 允许 HTML 标签
  sanitize: false     # 禁用安全过滤（关键）
  smartLists: true
  smartypants: true
```
注意：sanitize: false 是关键，否则即使写了 HTML 也会被过滤掉。


2. 更安全的方案：使用 `<iframe srcdoc>`

为避免污染全局样式、破坏主题布局，强烈建议优先采用 `<iframe>` 内嵌方式：

```html
<iframe
srcdoc="
<!DOCTYPE html>
<html><head><meta charset='utf-8'><style>/ 局部 CSS /</style></head>
<body>
<!-- 微博内容 -->
</body></html>"
sandbox="allow-same-origin"
style="width:100%; height:600px; border:none;"
</iframe>
```
此方法将内容完全隔离在独立文档上下文中，既安全又干净，不会影响原有样式。
