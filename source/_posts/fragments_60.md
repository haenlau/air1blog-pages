---
title: 增加oranges主题功能
date: 2025-11-23 19:12:23
tags:
  - Hexo
---

Oranges 主题添加导航链接

只需修改主题配置文件，不用新建 EJS 文件。
步骤

1. 打开 _config.oranges.yml
2. 在 navbar 列表里加一项：

yaml
name: 开往
enable: true
path: https://
key: kaiwang

3. 保存后运行：

```bash
hexo clean && hexo g && hexo s
```

完成！导航栏会自动显示「开往」并跳转。

仅普通链接才不用 .ejs；需要图标或复杂结构时才要创建。