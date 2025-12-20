---
title: Fuwari 主题自定义修改选项
date: 2025-12-20 19:31:48
tags:
  - Fuwari
---

# Fuwari 主题按照个人喜好修改记录

## 1. 导航栏 (Navbar) 毛玻璃效果

**目标**：让顶部导航栏呈现半透明并带有模糊效果（毛玻璃），适配浅色和深色模式。

**修改文件**：`src/layouts/MainGridLayout.astro`

**修改内容**：

1.  找到 `<Navbar ... />` 组件的调用。
2.  将其 `class` 属性替换为以下内容，以应用毛玻璃样式：
    ```astro
    <Navbar class="!bg-white/70 !backdrop-blur-md border-b border-gray-200/70 dark:!bg-black/80 dark:!backdrop-blur-md dark:border-gray-800/70" />
    ```
    *   **说明**:
        *   `!bg-white/70`: 浅色模式下，设置 70% 不透明度的白色背景（可根据需要调整数值，如 60/80）。
        *   `!backdrop-blur-md`: 添加中等强度的背景模糊效果。
        *   `border-b ...`: 添加底部边框线以明确导航栏边界。
        *   `dark:...`: 对应的深色模式样式。
3.  **关键**：确保包裹 `<Navbar>` 的 `#navbar-wrapper` div 保留其原始功能性类 `class="pointer-events-auto sticky top-0 transition-all"`。
4.  保存文件，重启开发服务器 (`npm run dev`) 查看效果。

**备注**：
*   透明度数值越小（如 `/60`），看起来越透明；数值越大（如 `/80`），看起来越不透明。
*   可根据视觉效果微调 `!bg-white/**70**` 中的数字，以在浅色和深色模式下达到理想的“透”度平衡。

---

## 2. 文章卡片与 Banner 重叠程度调整

**目标**：调整首页文章卡片列表区域与顶部 Banner 图像的重叠量。

**修改文件**：`src/constants/constants.ts`

**修改内容**：

1.  打开文件 `src/constants/constants.ts`。
2.  找到定义重叠高度的常量：
    ```typescript
    // The height the main panel overlaps the banner, unit: rem
    export const MAIN_PANEL_OVERLAPS_BANNER_HEIGHT = 3.5;
    ```
3.  修改该常量的数值：
    *   **增大数值** (例如改为 `4.5`) → **增加重叠** (文章卡片向上移动，更深入 Banner 区域)。
    *   **减小数值** (例如改为 `2.5`) → **减少重叠** (文章卡片向下移动，远离 Banner 区域)。
4.  保存文件。
5.  重启开发服务器 (`npm run dev`) 查看效果。

**备注**：
*   当前 Banner 高度由 `BANNER_HEIGHT` (35vh) 定义。
*   最终文章卡片顶部位置由 `calc(BANNER_HEIGHT vh - MAIN_PANEL_OVERLAPS_BANNER_HEIGHT rem)` 计算得出。

---