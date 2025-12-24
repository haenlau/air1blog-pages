---
title: Hexo在Cloudflare文件缓存导致部署失败
date: 2025-07-21 12:56:33
tags:
  - Hexo
---

Hexo 在 Cloudflare 构建时报错：layout/index.styl 不存在却报错
问题现象

在向 Hexo 博客（使用自定义主题 oranges）中添加 Umami 统计代码后，本地预览正常，但 Cloudflare Pages 部署失败，报错如下：

```
Error: /opt/buildhome/repo/themes/oranges/layout/index.styl:1:9
1 @import 'footer'
failed to locate @import file footer.styl
```

然而：
本地 themes/oranges/layout/ 目录下仅有 .ejs 文件，无任何 .styl 文件；
GitHub 仓库中也不存在 layout/index.styl；
package.json 中未通过 npm 安装 hexo-theme-oranges，仅使用本地主题目录。
排查过程
1. 检查本地文件结构

```bash
ls -la themes/oranges/layout/
```

输出结果为：

```
archive.ejs
category.ejs
index.ejs
layout.ejs
post.ejs
tag.ejs
```

确认无 .styl 文件。

2. 检查 node_modules 是否包含该主题

```bash
ls -la node_modules/hexo-theme-oranges/
```

返回 “No such file or directory”，说明未通过 npm 安装该主题，排除依赖冲突。

3. 检查 Git 历史是否曾存在该文件

```bash
git log --all --full-history -- themes/oranges/layout/index.styl
```

无任何输出，表明该文件从未出现在 Git 历史中。

4. 观察重新部署后的构建日志

执行全新部署后，Cloudflare 构建日志显示：

```
INFO 80 files generated in 183 ms
...
Success: Your site was deployed!
```

整个过程无 Stylus 相关错误，构建成功。
根本原因分析

问题并非代码或配置错误，而是 Cloudflare Pages 构建环境的缓存异常 所致：
可能在某次构建中，缓存目录残留了旧状态或临时文件；
尽管源码中不存在 layout/index.styl，但构建工具（如 hexo-renderer-stylus）在缓存环境中“误读”了某个路径；
添加 Umami 后触发更完整的模板渲染流程，使该异常暴露；
全新部署（清空缓存）后问题消失，证实为偶发性环境问题。
解决方案

1. 强制清除 Cloudflare Pages 缓存
在 Cloudflare Dashboard 的 Pages 项目中，找到最近的部署，点击 “Retry with cleared cache”；
或通过空提交触发全新构建：
```bash
git commit --allow-empty -m "fix: force fresh build on Cloudflare"
git push
```

2. 避免在 EJS 模板中混入 Stylus 语法
确保 layout/*.ejs 文件内容为合法 EJS/HTML，不含 CSS 或 Stylus 代码；
Stylus 样式应放在 source/css/ 目录下，由 Hexo 自动编译。
总结

该问题属于 CI/CD 构建缓存导致的偶发故障，非代码逻辑错误。通过强制刷新构建环境即可解决。建议在遇到类似“文件不存在却报错”的问题时，优先考虑清除远程构建缓存。