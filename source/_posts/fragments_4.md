---
title: 如何在Cloudflare搭建临时文件存储服务
date: 2025-12-03 20:39:10
tags:
  - Workers
---

全程在浏览器操作，无需本地开发环境，支持公开上传 + 自动过期 + 原始文件名下载

## 第一步：创建 KV 命名空间（用于存储临时文件）

目标：创建一个名为 TEMP_STORE 的 KV 存储空间。
操作路径：
Dashboard 首页 → 左侧边栏 「账户和主页」 → 「存储和数据库」 → 「Workers KV」
操作步骤：
1. 点击右上角 「Create instance」 按钮
2. 填写：
Name: TEMP_STORE
（其他选项保持默认）
3. 点击 「Create」
提示：无需记录 Namespace ID，后续通过变量名绑定即可。

## 第二步：创建 Worker 并粘贴代码

目标：部署处理上传/下载逻辑的 Worker。
操作路径：
Dashboard 首页 → 左侧边栏 「账户和主页」 → 「计算和 AI」 → 「Workers 和 Pages」
操作步骤：
1. 点击 「创建应用」 → 选择 「从 Hello World! 开始」
2. 应用名称输入：tmp-worker（可自定义）
3. 进入代码编辑器后，全选并删除默认代码
4. 将下方完整 JS 代码 逐字粘贴 到编辑区
重要：请先修改以下两处为你自己的信息！
```js
// 1. HTML 标题（第 4 行）
<title>📁 tmp.yourdomain.com</title>

// 2. 下载链接域名（约第 120 行）
const downloadUrl = https://tmp.yourdomain.com/${fileId};
```
<details>
<summary>▶ 点击展开完整 Worker 代码（含 4 位短 ID + 自动去重 + 12 小时过期）</summary>

```javascript
// ====== HTML 页面 ======
const HTML = `
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Air1临时文件</title>
  <link rel="icon" type="image/png" href="https://air1.cn/favicon.png" />
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; max-width: 500px; margin: 40px auto; padding: 20px; }
    h1 { text-align: center; }
    input, button { width: 100%; padding: 12px; margin: 10px 0; box-sizing: border-box; border: 1px solid #ccc; border-radius: 6px; }
    button { background: #007bff; color: white; border: none; cursor: pointer; }
    button:hover { background: #0069d9; }
    #result { margin-top: 15px; padding: 12px; background: #e8f4ff; border-radius: 6px; word-break: break-all; }
    a { color: #007bff; text-decoration: none; }
    a:hover { text-decoration: underline; }
  </style>
</head>
<body>
  <h1>📁 临时文件保存</h1>
  <input type="file" id="fileInput" />
  <button onclick="upload()">上传（≤25MB）</button>
  <div id="result"></div>
  <p style="text-align:center;color:#666;font-size:14px;">文件 12 小时后自动删除</p>

  <script>
    async function upload() {
      const file = document.getElementById('fileInput').files[0];
      if (!file) return alert("请选择文件");
      if (file.size > 26112000) return alert("文件不能超过 25MB");

      const formData = new FormData();
      formData.append("file", file);
      const btn = document.querySelector('button');
      btn.disabled = true;
      btn.textContent = "上传中…";

      try {
        const res = await fetch("/api/upload-public", { method: "POST", body: formData });
        const data = await res.json();
        const el = document.getElementById('result');
        if (res.ok && data.downloadUrl) {
          el.innerHTML = '<strong>✅ 分享链接：</strong><br><a href="' + data.downloadUrl + '" target="_blank">' + data.downloadUrl + '</a>';
        } else {
          el.innerText = "❌ " + (data.error || "上传失败");
        }
      } catch (e) {
        document.getElementById('result').innerText = "网络错误：" + e.message;
      } finally {
        btn.disabled = false;
        btn.textContent = "上传（≤25MB）";
      }
    }
  </script>
</body>
</html>
`;

// ====== 工具函数 ======
function generateFileId() {
  return Math.random().toString(36).substring(2, 6); // 2 到 8 → 6字符
}

// ====== 处理文件上传 ======
async function handleFileUpload(file, env) {
  const MAX_SIZE = 26112000; // 25 MB
  if (file.size > MAX_SIZE) {
    return new Response(JSON.stringify({ error: "文件不能超过 25MB" }), {
      status: 400,
      headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
    });
  }

  const fileId = generateFileId();
  const arrayBuffer = await file.arrayBuffer();

  await env.TEMP_STORE.put(fileId, arrayBuffer, {
    metadata: {
      filename: file.name || "file",
      contentType: file.type || "application/octet-stream"
    },
    expirationTtl: 43200 // 12小时 = 43200秒
  });

  const downloadUrl = `https://tmp.air1.cn/${fileId}`;

  return new Response(JSON.stringify({ downloadUrl }), {
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*"
    }
  });
}

// ====== 主入口 ======
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    const { pathname } = url;

    // 1. 首页
    if (pathname === "/") {
      return new Response(HTML, {
        headers: { "Content-Type": "text/html; charset=utf-8" }
      });
    }

    // 2. CORS 预检
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": "POST, OPTIONS",
          "Access-Control-Allow-Headers": "Content-Type"
        }
      });
    }

    // 3. 上传接口
    if (pathname === "/api/upload-public" && request.method === "POST") {
      try {
        const formData = await request.formData();
        const file = formData.get("file");
        if (!file || !(file instanceof File)) {
          return new Response(JSON.stringify({ error: "未提供有效文件" }), {
            status: 400,
            headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
          });
        }
        return await handleFileUpload(file, env);
      } catch (e) {
        console.error("上传处理出错:", e);
        return new Response(JSON.stringify({ error: "服务器内部错误" }), {
          status: 500,
          headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
        });
      }
    }

    // 4. 文件下载：直接通过 /{id} 访问（例如 /ajjdfs）
    const segments = pathname.split('/').filter(Boolean);
    if (segments.length === 1 && segments[0].length >= 4) {
      const id = segments[0];
      // 保留路径白名单（防止与未来功能冲突）
      const reservedPaths = new Set([
        'api', 'upload', 'f', 'favicon.ico', 'robots.txt', 'about', 's'
      ]);
      if (!reservedPaths.has(id)) {
        const entry = await env.TEMP_STORE.getWithMetadata(id, "arrayBuffer");
        if (entry.value) {
          return new Response(entry.value, {
            headers: {
              "Content-Type": entry.metadata?.contentType || "application/octet-stream",
              "Content-Disposition": "attachment; filename=\"" +
                encodeURIComponent(entry.metadata?.filename || 'file') + "\"",
              "Cache-Control": "no-store"
            }
          });
        }
      }
    }

    // 5. 未匹配任何路由 → 404
    return new Response("Not Found", { status: 404 });
  }
};
```

</details>

5. 点击右上角 「Save and Deploy」

## 第三步：绑定 KV 命名空间到 Worker

目标：让 Worker 能读写你刚创建的 TEMP_STORE。
操作路径：
在 Worker 编辑页面 → 顶部标签栏选择 「绑定」
操作步骤：
1. 点击 「添加绑定」 → 选择 「KV 命名空间」
2. 弹窗中填写：
变量名称（Variable name）: TEMP_STORE ← 必须与代码中 env.TEMP_STORE 一致
KV 命名空间（KV namespace）: 选择你刚创建的 TEMP_STORE
3. 点击 「添加」
此时无需 Secret，因为服务是公开上传。

## 第四步：绑定自定义域名路由

前提：你的域名（如 tmp.yourdomain.com）已在 Cloudflare DNS 托管，且状态为 Proxied（橙色云图标）。
操作路径：
在 Worker 详情页 → 顶部标签栏选择 「设置」 → 滚动到 「Routes」 区域
操作步骤：
1. 点击 「Add Route」
2. 输入：
Route: tmp.yourdomain.com/
3. 点击 「保存」
📌 注意：
必须带 /，否则根路径 / 无法匹配
域名必须已在 Cloudflare DNS 中，且代理开启（橙色云）

## 第五步：验证功能

测试项 操作 预期结果
------- ------ --------
首页访问 浏览器打开 https://tmp.yourdomain.com 显示文件上传页面
上传文件 选择 ≤25MB 文件点击上传 返回短链接，如 https://tmp.yourdomain.com/abcd
下载文件 访问该短链接 浏览器自动下载，保留原始文件名
过期测试 12 小时后再次访问 返回 404 Not Found
API 测试（可选）：
```bash
curl -X POST https://tmp.yourdomain.com/api/upload-public \
-F "file=@test.txt"
```
成功响应：
```json
{"downloadUrl":"https://tmp.yourdomain.com/abcd"}
```
## 注意事项 & 最佳实践
1. ID 长度与容量
当前使用 4 位 ID（如 abcd），安全上限：≈1,600 文件 / 12 小时
若需更高容量，改为 5 位：
js
return Math.random().toString(36).substring(2, 7); // 5字符

并将路由判断改为 segments[0].length >= 5
2. 文件限制
单文件 ≤ 25 MB（Cloudflare Workers 限制）
自动 12 小时过期（通过 expirationTtl: 43200 实现）
3. 路径冲突防护
已预留以下路径，不会被当作文件 ID：
js
const reservedPaths = new Set([
'api', 'upload', 'f', 'favicon.ico', 'robots.txt', 'about'
]);
4. HTTPS 与安全性
Cloudflare 自动提供 HTTPS，无需配置证书
上传接口为公开，如需鉴权可参考短链接服务增加 API_TOKEN

至此，你的临时文件存储服务已上线！链接简洁、自动清理、保留文件名，适合分享日志、截图、临时文档等场景。