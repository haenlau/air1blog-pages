---
title: Fuwari主题修改社交图标
date: 2025-12-04 12:36:23
tags:
  - Fuwari
---

Fuwari 主题：使用 line-md 图标修改社交链接

本指南适用于在 [Fuwari](https://github.com/saicaca/fuwari) 主题中，将默认的社交图标替换为 line-md 风格图标。

1. 查找并选择图标

推荐方式：使用 Icones 官网
访问图标集合页面：
[https://icones.js.org/collection/line-md](https://icones.js.org/collection/line-md)

支持关键词搜索（如 x、github、mail、discord）

点击任意图标即可复制其完整标识，格式为：

```text
line-md:图标名
```

`示例：line-md:twitter-x`

注：浏览 GitHub 或 npm 源码对日常配置无必要。

2. 确认图标集是否可用

line-md 是一个独立的图标集合。只要项目已集成该图标集，即可自由使用其中任意图标。

例如：
`line-md:twitter-x`
`line-md:github`

优势：一次集成，全量可用 —— 无需为每个图标单独操作。
安装 line-md 图标集

在项目根目录执行以下命令：

```bash
pnpm add @iconify-json/line-md
```

3. 修改主题配置文件

编辑主题目录下的配置文件：

theme/config.ts

找到 profileConfig.links 数组，修改对应项。例如：

```ts
{
name: "X",
icon: "line-md:twitter-x",
url: "https://x.com/yourname"
}
```
字段说明
name：链接显示的文字（建议改为 “X”）
icon：从 Icones 复制的完整图标标识（如 line-md:twitter-x）
url：完整的跳转地址
X 平台请使用 x.com 域名

4. 验证效果

保存文件后刷新页面（开发模式通常支持热更新）。

若图标未显示，请检查：
图标名称是否拼写正确（区分大小写）
项目是否已安装 @iconify-json/line-md
浏览器控制台是否有加载错误

5. 常用社交平台配置示例

平台（原名） 推荐图标（line-md 风格） URL 示例
-------------------- -------------------------- ------------------------------
X（原 Twitter） line-md:twitter-x https://x.com/yourname

GitHub line-md:github https://github.com/yourname

电子邮件 line-md:email mailto:you@example.com

如果 line-md 中没有某平台图标，可回退到其他图标集（如 fa6-brands:mastodon），但需确保该图标集已安装并可用。

6. 温馨提示
所有图标由 Iconify 自动按需加载，无需手动引入 SVG 文件。
图标颜色和尺寸可通过 CSS 控制。例如，在主题样式中添加：

修改 theme/config.ts 后通常无需重启开发服务器，热更新会自动生效。