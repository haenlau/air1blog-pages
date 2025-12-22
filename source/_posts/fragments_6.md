---
title: Fuwari主题自定义修改选项二
date: 2025-12-20 13:51:17
tags:
  - Fuwari
---


为 Fuwari 博客添加公告栏

目标: 在博客侧边栏 (Sidebar) 中添加一个“公告栏”组件，使其样式与其他组件（如“分类”、“标签”）保持一致，并支持国际化。
1. 创建公告栏组件

目的: 创建一个独立的组件来承载公告内容。
操作:
1. 在 src/components/ 目录下，如果不存在，则创建 widget 文件夹。
2. 在 src/components/widget/ 目录下，新建文件 NoticeBoard.astro。
3. 编辑 NoticeBoard.astro，编写组件代码。

文件: src/components/widget/NoticeBoard.astro

```astro

// src/components/widget/NoticeBoard.astro
import { i18n } from "../../i18n/translation";
import I18nKey from "../../i18n/i18nKey"; // Default import for enum
import WidgetLayout from "./WidgetLayout.astro";

// --- 定义公告内容 ---
const noticeContent = "欢迎来到我的博客！这里是公告栏，我会不定期更新一些重要信息。";
// --- End 定义内容 ---

interface Props {
class?: string;
style?: string;
}
const className = Astro.props.class;
const style = Astro.props.style;

<!-- 使用 WidgetLayout 包裹，使其样式与其他侧边栏组件一致 -->
<WidgetLayout name={i18n(I18nKey.notice_board)} id="notice-board" isCollapsed={false} collapsedHeight="7.5rem" class={className} style={style}>
<div class="text-sm text-neutral-600 dark:text-neutral-300">
{noticeContent}
</div>
</WidgetLayout>
```

2. 添加国际化支持

目的: 使公告栏标题能够根据网站语言显示对应的文字。
操作:
1. 添加键 (Key): 编辑 src/i18n/i18nKey.ts，在 enum I18nKey 中添加新成员。
2. 添加翻译 (Translation): 编辑 src/i18n/zh-CN.ts，在翻译对象中添加对应的中文翻译。

文件: src/i18n/i18nKey.ts

```typescript
// ... existing keys ...
enum I18nKey {
// ... other members ...
notice_board = "notice_board", // <-- Add this line
// ... other members ...
}
// ... rest of the file ...
```

文件: src/i18n/zh-CN.ts

```typescript
// ... imports ...
export const zh_CN: Translation = {
// ... existing translations ...
[Key.notice_board]: "公告栏", // <-- Add this line
// ... existing translations ...
};
// ... rest of the file ...
```

3. 在侧边栏中引入并使用公告栏

目的: 将新创建的 NoticeBoard 组件添加到侧边栏中。
操作:
1. 导入组件: 编辑 src/components/Sidebar/Sidebar.astro，在顶部导入 NoticeBoard。
2. 放置组件: 在侧边栏的 HTML 结构中，选择合适的位置插入 <NoticeBoard /> 标签。

文件: src/components/Sidebar/Sidebar.astro (导入部分)

```astro

import type { MarkdownHeading } from "astro";
import Categories from "./Categories.astro";
import Profile from "./Profile.astro";
import Tag from "./Tags.astro";
// --- 新增导入 ---
import NoticeBoard from "../widget/NoticeBoard.astro";
// --- End 新增导入 ---
// ... rest of the imports and logic ...
```

文件: src/components/Sidebar/Sidebar.astro (使用部分)

```astro
<!-- ... previous sidebar content ... -->
<div id="sidebar-sticky" class="transition-all duration-700 flex flex-col w-full gap-4 top-4 sticky top-4">
<Categories class="onload-animation" style="animation-delay: 150ms"></Categories>
<Tag class="onload-animation" style="animation-delay: 200ms"></Tag>
<!-- === 新增：公告栏 === -->
<NoticeBoard class="onload-animation" style="animation-delay: 250ms" />
<!-- === End 新增：公告栏 === -->
</div>
<!-- ... rest of the sidebar content ... -->
```

4. 重启开发服务器

目的: 使所有更改生效。
操作:
1. 保存所有已编辑的文件。
2. 在终端中，停止当前运行的开发服务器（通常是按 Ctrl+C）。
3. 重新启动开发服务器，例如运行 pnpm dev。
4. 打开浏览器，访问你的博客地址，刷新页面查看新增的公告栏。
