---
title: 做一个短链接服务
date: 2025-12-03 13:23:21
tags:
  - Workers
---

使用 Cloudflare Workers + KV 搭建安全短链接服务，虽然还不知道短链接对我的工作和生活有什么帮助，本着有成熟的实践，自己也做一套罢。

非说新颖的地方就是我还制作了针对 chrome 和 edge 浏览器的扩展程序，在浏览器里右键，可以直接选择生成当前 web url 的短链接。

全程浏览器操作，无需本地开发环境，API 集成 Bearer Token 鉴权。

## 第一步：创建 KV 命名空间（用于存储短链接映射）

目标：创建一个名为 URLS 的 KV 存储空间，用于保存“短码 → 长链接”的映射关系。
操作路径：
Dashboard 首页 → 左侧边栏 「账户和主页」 → 「存储和数据库」 → 「Workers KV」
操作步骤：
1. 点击右上角 「Create instance」
2. 填写：
Name: URLS
3. 点击 「Create」
提示：无需记录 Namespace ID，后续通过变量名绑定即可。

## 第二步：创建 Worker 并粘贴代码

目标：部署处理短链接生成与跳转逻辑的 Worker。
操作路径：
Dashboard 首页 → 左侧边栏 「账户和主页」 → 「计算和 AI」 → 「Workers 和 Pages」
操作步骤：
1. 点击 「创建应用」 → 选择 「从 Hello World! 开始」
2. 应用名称输入：go-worker（可自定义）
3. 进入代码编辑器后，全选并删除默认代码
4. 在粘贴前，请先修改以下三处为你自己的信息！
🔔 必须修改项：
```html
<!-- 1. 页面标题（HTML 第 4 行） -->
<title>🔗 go.yourdomain.com</title>
```
```html
<!-- 2. 网站图标（可选，第 5 行） -->
<link rel="icon" type="image/png" href="https://yourdomain.com/favicon.png" />
```
```js
// 3. 短链接拼接域名（Worker 代码中两处）
shortUrl: "https://go.yourdomain.com/" + shortCode
```
5. 将下方完整代码 逐字粘贴 到编辑区：

<details>
<summary>▶ 点击展开完整 Worker 代码（含公开接口 + Bearer Token 鉴权）</summary>

```javascript
const HTML =
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>🔗 go.yourdomain.com</title>
<link rel="icon" type="image/png" href="https://yourdomain.com/favicon.png" />
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'PingFang SC', 'Microsoft YaHei', sans-serif; max-width: 600px; margin: 50px auto; padding: 20px; line-height: 1.6; }
h1 { text-align: center; margin-bottom: 30px; }
input, button { padding: 12px; width: 100%; margin: 10px 0; box-sizing: border-box; border: 1px solid #ccc; border-radius: 4px; }
button { background: #007bff; color: white; cursor: pointer; font-size: 16px; }
button:hover { background: #0069d9; }
#result { margin-top: 20px; padding: 12px; background: #f8f9fa; border: 1px solid #e9ecef; border-radius: 4px; word-break: break-all; }
a { color: #007bff; text-decoration: none; }
a:hover { text-decoration: underline; }
</style>
</head>
<body>
<h1>短链接生成</h1>
<input id="longUrl" placeholder="请输入长链接（例如：https://...）" />
<button onclick="createShortLink()">生成短链接</button>
<div id="result"></div>

<script>
async function createShortLink() {
const longUrl = document.getElementById('longUrl').value.trim();
if (!longUrl) {
alert("请输入一个有效的网址");
return;
}

const shortCode = Math.random().toString(36).substring(2, 8);

try {
const res = await fetch('/api/create-public', {
method: 'POST',
headers: { 'Content-Type': 'application/json' },
body: JSON.stringify({ longUrl, shortCode })
});

const data = await res.json();
const resultDiv = document.getElementById('result');
if (data.ok) {
resultDiv.innerHTML = '<strong>您的短链接：</strong><br>' +
'<a href="' + data.shortUrl + '" target="_blank">' + data.shortUrl + '</a>';
} else {
resultDiv.innerText = "错误：" + (data.error "未知错误");
}
} catch (err) {
resultDiv.innerText = "网络错误：" + err.message;
}
}
</script>
</body>
</html>;

export default {
async fetch(request, env) {
const url = new URL(request.url);
const { pathname } = url;

// 首页
if (pathname === "/") {
return new Response(HTML, {
headers: { "Content-Type": "text/html; charset=utf-8" }
});
}

// CORS 预检
if (request.method === "OPTIONS") {
return new Response(null, {
headers: {
"Access-Control-Allow-Origin": "",
"Access-Control-Allow-Methods": "POST, OPTIONS",
"Access-Control-Allow-Headers": "Content-Type, Authorization"
}
});
}

// ───────────────────────────────────────
// 公开创建接口：/api/create-public（无需 Token）
// ───────────────────────────────────────
if (pathname === "/api/create-public" && request.method === "POST") {
try {
const { longUrl, shortCode } = await request.json();
if (!longUrl !shortCode) {
return new Response(JSON.stringify({ error: "缺少 longUrl 或 shortCode" }), {
status: 400,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}
await env.URLS.put(shortCode, longUrl);
return new Response(JSON.stringify({
ok: true,
shortUrl: "https://go.yourdomain.com/" + shortCode
}), {
headers: {
"Content-Type": "application/json",
"Access-Control-Allow-Origin": ""
}
});
} catch (e) {
return new Response(JSON.stringify({ error: "服务器内部错误" }), {
status: 500,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}
}

// ───────────────────────────────────────
// 受控创建接口：/api/create（需要 API_TOKEN）
// ───────────────────────────────────────
if (pathname === "/api/create" && request.method === "POST") {
const expectedToken = env.API_TOKEN;
if (!expectedToken) {
return new Response(JSON.stringify({ error: "服务器未配置 API_TOKEN Secret" }), {
status: 500,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}

const authHeader = request.headers.get("Authorization");
if (!authHeader !authHeader.startsWith("Bearer ")) {
return new Response(JSON.stringify({ error: "缺少或无效的 Authorization 头。格式应为：Bearer <API_TOKEN>" }), {
status: 401,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}

const token = authHeader.substring(7);
if (token !== expectedToken) {
return new Response(JSON.stringify({ error: "API Token 无效" }), {
status: 403,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}

try {
const { longUrl, shortCode } = await request.json();
if (!longUrl !shortCode) {
return new Response(JSON.stringify({ error: "缺少 longUrl 或 shortCode" }), {
status: 400,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}
await env.URLS.put(shortCode, longUrl);
return new Response(JSON.stringify({
ok: true,
shortUrl: "https://go.yourdomain.com/" + shortCode
}), {
headers: {
"Content-Type": "application/json",
"Access-Control-Allow-Origin": ""
}
});
} catch (e) {
return new Response(JSON.stringify({ error: "服务器内部错误" }), {
status: 500,
headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "" }
});
}
}

// 短链接跳转
const code = pathname.slice(1);
if (code) {
const target = await env.URLS.get(code);
if (target) {
return Response.redirect(target, 302);
}
}

// 未找到
return new Response("短链接不存在", { status: 404 });
}
};
```
</details>

6. 点击右上角 「Save and Deploy」

## 第三步：绑定 KV 命名空间 + 添加 API Token Secret
操作路径：
在 Worker 编辑页面 → 顶部标签栏选择 「绑定」
操作步骤：
1. 绑定 KV 命名空间
点击 「添加绑定」 → 选择 「KV 命名空间」
弹窗中填写：
变量名称（Variable name）: URLS
KV 命名空间（KV namespace）: 选择第一步创建的 URLS
点击 「添加」
2. 添加 API Token Secret
切换到顶部标签 「设置」 → 向下滚动至 「变量和机密」
点击 「添加变量」 → 选择 「密钥（Secret）」
填写：
变量名称: API_TOKEN
值: 你的高熵 Token（如 sk-xxxxxxxxxxxxx）
点击 「添加」
变量名必须严格匹配代码中的 env.URLS 和 env.API_TOKEN

## 第四步：绑定自定义域名路由

前提：你的子域名（如 go.yourdomain.com）已在 Cloudflare DNS 中托管，且代理状态为 Proxied（橙色云图标）。
操作路径：
在 Worker 详情页 → 顶部标签 「设置」 → 滚动到 「Routes」 区域
操作步骤：
1. 点击 「Add Route」
2. 输入：
Route: go.yourdomain.com/
3. 点击 「保存」
注意：必须带 /，否则根路径 / 无法匹配

## 第五步：验证功能

测试项 操作 预期结果
------- ------ --------
首页访问 浏览器打开 https://go.yourdomain.com 显示“短链接生成”页面
公开创建 在页面输入 URL 并点击生成 返回短链接（如 go.yourdomain.com/abc123）
受控 API 创建 使用 curl 调用 /api/create 成功返回短链接（需正确 Token）
跳转测试 访问 https://go.yourdomain.com/abc123 自动重定向到原始长链接
API 测试命令（替换 YOUR_TOKEN）：
```bash
curl -X POST https://go.yourdomain.com/api/create \
-H "Authorization: Bearer YOUR_TOKEN" \
-H "Content-Type: application/json" \
-d '{"longUrl":"https://example.com","shortCode":"test1"}'
```
成功响应：
```json
{"ok":true,"shortUrl":"https://go.yourdomain.com/test1"}
```
## 注意事项 & 最佳实践
1. 域名替换
所有出现 go.air1.cn 的地方都需替换为你的域名
包括 HTML 标题、favicon、JS 拼接逻辑（共 3 处）
2. 短码冲突风险
当前使用 Math.random().toString(36).substring(2, 8) 生成 6 位随机码
冲突概率较低，但未做去重检测，适用于个人或低频场景
高频场景建议改用 Durable Objects 或加锁机制
3. 安全增强建议
使用高熵 Token（如 crypto.randomUUID() 生成）
限制 /api/create 的来源 IP（通过 Firewall Rules）
添加速率限制（需结合 Cloudflare Rate Limiting 或 Durable Objects）
4. HTTPS 与性能
Cloudflare 自动为所有 Worker 提供免费 HTTPS
无需额外配置证书，全球 CDN 加速自动生效

至此，一个安全、简洁、可公开使用也可 API 控制的短链接服务已成功上线！适合用于分享、营销、临时跳转等场景。
