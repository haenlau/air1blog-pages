---
title: Hexo在Cloudflare日期偏移问题
date: 2025-08-11 09:10:41
tags:
  - Hexo
---

在使用 Hexo 搭建静态站点后使用 Oranges 主题并部署到 Cloudflare Pages 时，可能会遇到一个隐蔽问题：本地预览和部分平台（如腾讯 EdgeOne Pages）显示的日期完全正确，但 Cloudflare Pages 上所有文章的日期却统一提前了一天。

例如：
- 本地显示：2025-11-18、2025-11-19
- Cloudflare Pages 显示：2025-11-17、2025-11-18

问题根源在于时区处理差异。Hexo 在解析 front-matter 中的 `date: 2025-11-18`（无时间、无偏移）时，会将其视为 UTC 时间。而 Cloudflare Pages 的构建环境默认使用 UTC 时区，且其底层镜像可能缺少完整的时区数据库，导致 `_config.yml` 中配置的 `timezone: Asia/Shanghai` 未被正确应用。因此，日期在生成 URL 和渲染页面时被错误地按 UTC 解析，造成“早一天”的现象。

解决方案如下：

1. 登录 Cloudflare Dashboard，进入对应 Pages 项目。
2. 点击 Settings，滚动至 Environment variables 区域。
3. 在 Build-time environment variables 中添加变量：
   - Variable name: `TZ`
   - Value: `Asia/Shanghai`
4. 保存设置，并手动触发一次 Redeploy。

该环境变量会令 Hexo 构建过程运行于东八区上下文，从而正确解析日期。部署完成后，文章路径与页面显示的日期将与本地一致。

若未来在其他平台遇到类似问题，也可考虑在 front-matter 中直接使用带时区偏移的格式（如 `date: 2025-11-18T00:00:00+08:00`），以实现跨平台一致性。但针对 Cloudflare Pages，设置 `TZ` 环境变量是最简洁有效的解法。