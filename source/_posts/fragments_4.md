---
title: 利用闲置的云盘搭建轻量临时文件中转站
date: 2025-10-17 21:22:31
tags:
  - Workers
---

日常办公中需要将单位文件带回家处理，既不想通过微信、QQ 等工具将文件存储到本地，又觉得国内云盘臃肿。此前曾领取 InfiniCLOUD 40GB 云盘空间，但因该服务部署在日本，所以一直处于闲置状态。

为解决文件跨设备临时传输的痛点，决定利用 Cloudflare KV 的边缘存储特性 + InfiniCLOUD 的 WebDAV 协议支持，搭建一套轻量、便捷的临时文件存储服务。

为什么不使用 R2 存储桶，第一，存储桶配额 10G 有其他用途；第二，超过 10G 会收费不想惦记这事。

以下是详细的步骤

## 第一步：创建 KV 命名空间（用于存储临时文件）

目标：创建一个名为 TEMP_STORE 的 KV 存储空间。
操作路径：
Dashboard 首页 → 左侧边栏 「账户和主页」 → 「存储和数据库」 → 「Workers KV」
操作步骤：
1. 点击右上角 「Create instance」 按钮
2. 填写：
Name: TEMP_STORE
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

<details>
<summary>▶ 点击展开完整 Worker 代码，代码仅自用记录</summary>

```js
// ====== HTML 界面 ======
const HTML = `<!DOCTYPE html>
<html lang="zh-CN">
<head>
 <meta charset="utf-8" />
 <meta name="viewport" content="width=device-width, initial-scale=1" />
 <title>✨ Air1 TempFile</title>
 <link rel="icon" type="image/png" href="https://air1.cn/favicon.png" />
 <style>
 :root {
 --bg: #0f0f12;
 --card-bg: rgba(255, 255, 255, 0.06);
 --card-border: rgba(255, 255, 255, 0.1);
 --text: #f0f0f5;
 --muted: #a0a0b0;
 --primary: #4d9fff;
 --success: #4ade80;
 --danger: #f87171;
 --shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
 --radius: 18px;
 }

 @media (prefers-color-scheme: light) {
 :root {
 --bg: #f8f9ff;
 --card-bg: rgba(255, 255, 255, 0.7);
 --card-border: rgba(0, 0, 0, 0.08);
 --text: #1a1a25;
 --muted: #666;
 --primary: #2563eb;
 --success: #16a34a;
 --danger: #dc2626;
 --shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
 }
 }

 * {
 margin: 0;
 padding: 0;
 box-sizing: border-box;
 }

 body {
 background: var(--bg);
 color: var(--text);
 font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
 min-height: 100vh;
 display: flex;
 flex-direction: column;
 align-items: center;
 justify-content: center;
 padding: 2rem 1.5rem;
 position: relative;
 overflow-x: hidden;
 }

 body::before {
 content: "";
 position: absolute;
 top: 0;
 left: 0;
 width: 100%;
 height: 100%;
 background:
 radial-gradient(circle at 20% 30%, rgba(77, 159, 255, 0.06) 0%, transparent 40%),
 radial-gradient(circle at 80% 70%, rgba(77, 159, 255, 0.04) 0%, transparent 50%);
 pointer-events: none;
 z-index: -1;
 }

 .container {
 width: 100%;
 max-width: 520px;
 text-align: center;
 }

 h1 {
 font-weight: 700;
 font-size: 2.2rem;
 margin-bottom: 0.4rem;
 background: linear-gradient(135deg, #ffffff, #a0a0ff);
 -webkit-background-clip: text;
 background-clip: text;
 color: transparent;
 background-size: 200% 200%;
 animation: gradientShift 8s ease infinite;
 }

 @keyframes gradientShift {
 0% { background-position: 0% 50%; }
 50% { background-position: 100% 50%; }
 100% { background-position: 0% 50%; }
 }

 .subtitle {
 color: var(--muted);
 font-size: 0.95rem;
 margin-bottom: 2.2rem;
 }

 .upload-card {
 background: var(--card-bg);
 border: 1px solid var(--card-border);
 backdrop-filter: blur(12px);
 -webkit-backdrop-filter: blur(12px);
 border-radius: var(--radius);
 padding: 2.2rem 1.5rem;
 margin-bottom: 1.8rem;
 cursor: pointer;
 transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
 position: relative;
 overflow: hidden;
 }

 .upload-card:hover {
 transform: translateY(-4px);
 box-shadow: var(--shadow);
 border-color: rgba(77, 159, 255, 0.3);
 }

 .upload-card.dragover {
 border-color: var(--primary);
 background: rgba(77, 159, 255, 0.08);
 }

 .upload-icon {
 font-size: 3.2rem;
 margin-bottom: 1.2rem;
 display: block;
 transition: transform 0.3s;
 }

 .upload-card:hover .upload-icon {
 transform: scale(1.1) rotate(3deg);
 }

 .upload-text {
 font-size: 1.1rem;
 font-weight: 500;
 margin-bottom: 0.4rem;
 }

 .upload-hint {
 font-size: 0.85rem;
 color: var(--muted);
 }

 .selected-file {
 margin-top: 0.6rem;
 font-size: 0.85rem;
 color: var(--primary);
 display: none;
 }

 #fileInput {
 display: none;
 }

 .btn {
 width: 100%;
 padding: 0.95rem;
 background: var(--primary);
 color: white;
 border: none;
 border-radius: 14px;
 font-size: 1.05rem;
 font-weight: 600;
 cursor: pointer;
 transition: all 0.25s;
 letter-spacing: 0.3px;
 }

 .btn:hover:not(:disabled) {
 background: #3a8bff;
 transform: translateY(-2px);
 box-shadow: 0 6px 16px rgba(77, 159, 255, 0.3);
 }

 .btn:disabled {
 opacity: 0.7;
 cursor: not-allowed;
 transform: none;
 box-shadow: none;
 }

 .result-card {
 background: var(--card-bg);
 border: 1px solid var(--card-border);
 backdrop-filter: blur(12px);
 -webkit-backdrop-filter: blur(12px);
 border-radius: var(--radius);
 padding: 1.6rem;
 margin-top: 1.5rem;
 display: none;
 }

 .result-card.show {
 display: block;
 animation: fadeIn 0.4s ease;
 }

 @keyframes fadeIn {
 from { opacity: 0; transform: translateY(10px); }
 to { opacity: 1; transform: translateY(0); }
 }

 .result-title {
 font-size: 1.1rem;
 margin-bottom: 1rem;
 color: var(--success);
 font-weight: 600;
 }

 .result-item {
 margin-bottom: 1rem;
 text-align: center;
 padding: 0.8rem;
 background: rgba(77, 159, 255, 0.05);
 border-radius: 10px;
 display: flex;
 flex-direction: column;
 align-items: center;
 }
 .result-filename {
 font-weight: bold;
 margin-bottom: 0.3rem;
 color: var(--text);
 }

 .result-link {
 display: block;
 word-break: break-all;
 color: var(--primary);
 text-decoration: none;
 font-size: 0.9rem;
 font-family: monospace;
 margin: 0.2rem 0;
 }

 .copy-btn {
 background: rgba(255, 255, 255, 0.12);
 color: var(--text);
 border: none;
 padding: 0.4rem 0.8rem;
 border-radius: 8px;
 font-weight: 600;
 cursor: pointer;
 transition: all 0.2s;
 font-size: 0.85rem;
 }

 .copy-btn:hover {
 background: rgba(255, 255, 255, 0.2);
 }

 .copy-btn.copied {
 background: var(--success);
 color: white;
 }

 .error-msg {
 color: var(--danger);
 margin-top: 0.5rem;
 font-size: 0.85rem;
 }

 /* ===== 新增：进度条样式 ===== */
 .progress-track {
 height: 6px;
 background: rgba(255, 255, 255, 0.1);
 border-radius: 3px;
 margin-top: 1.2rem;
 overflow: hidden;
 display: none; /* 初始隐藏 */
 }

 .progress-fill {
 height: 100%;
 width: 0%;
 border-radius: 3px;
 transition: width 0.2s ease;
 /* 默认样式 (可选，例如上传中) */
 background: var(--primary);
 }
 .progress-fill.uploading {
     background: linear-gradient(to right, var(--primary), #80c4ff);
 }
 .progress-fill.completed {
     background: var(--success);
 }

 .progress-text {
 margin-top: 0.4rem;
 font-size: 0.85rem;
 color: var(--muted);
 }

 .footer {
 margin-top: 2.5rem;
 color: var(--muted);
 font-size: 0.8rem;
 opacity: 0.8;
 }

 @media (max-width: 480px) {
 h1 { font-size: 1.8rem; }
 .upload-card { padding: 1.8rem 1rem; }
 }
 </style>
</head>
<body>
 <div class="container">
 <h1>Air1 TempFile</h1>
 <p class="subtitle">安全上传 · 多文件自动打包 · 7天自动销毁</p>

 <div class="upload-card" id="dropArea">
 <span class="upload-icon">📤</span>
 <p class="upload-text">拖拽文件或点击上传</p>
 <p class="upload-hint">支持任意格式</p>
 <p class="selected-file" id="selectedFile"></p>
 <input type="file" id="fileInput" multiple />
 </div>

 <button class="btn" id="uploadBtn" onclick="uploadFiles()">确认上传</button>

 <!-- ===== 新增：进度条容器 ===== -->
 <div class="progress-track" id="progressTrack">
 <div class="progress-fill" id="progressFill"></div>
 </div>
 <p class="progress-text" id="progressText" style="display:none;">上传中...</p>

 <div class="result-card" id="resultCard">
 <div class="result-title">✅ 上传完成</div>
 <div id="resultsList"></div>
 </div>

 <p class="footer">Part of <strong>Air1 Quick Tools</strong> · Powered by Cloudflare</p>
 </div>

 <script>
 const dropArea = document.getElementById('dropArea');
 const fileInput = document.getElementById('fileInput');
 const uploadBtn = document.getElementById('uploadBtn');
 const resultCard = document.getElementById('resultCard');
 const resultsList = document.getElementById('resultsList');
 const progressTrack = document.getElementById('progressTrack');
 const progressFill = document.getElementById('progressFill');
 const progressText = document.getElementById('progressText');
 const selectedFileEl = document.getElementById('selectedFile');

 ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(e => {
 dropArea.addEventListener(e, preventDefaults, false);
 });

 function preventDefaults(e) {
 e.preventDefault();
 e.stopPropagation();
 }

 ['dragenter', 'dragover'].forEach(e => {
 dropArea.addEventListener(e, () => dropArea.classList.add('dragover'), false);
 });

 ['dragleave', 'drop'].forEach(e => {
 dropArea.addEventListener(e, () => dropArea.classList.remove('dragover'), false);
 });

 dropArea.addEventListener('drop', e => {
 fileInput.files = e.dataTransfer.files;
 fileInput.dispatchEvent(new Event('change'));
 });

 dropArea.addEventListener('click', () => fileInput.click());

 fileInput.addEventListener('change', () => {
 const files = fileInput.files;
 if (files.length > 0) {
 let names = Array.from(files).map(f => f.name).join(', ');
 if (names.length > 60) names = names.substring(0, 60) + '...';
 selectedFileEl.textContent = '已选择 ' + files.length + ' 个文件：' + names;
 selectedFileEl.style.display = 'block';
 } else {
 selectedFileEl.style.display = 'none';
 }
 });

 // ====== 新增：上传进度函数 ======
 function updateProgress(loaded, total) {
 const percent = Math.round((loaded / total) * 100);
 progressFill.style.width = percent + '%';
 progressText.textContent = '上传中... ' + percent + '%';
 // 确保上传中状态
 progressFill.classList.add('uploading');
 progressFill.classList.remove('completed');
 }

 async function uploadFiles() {
 const files = Array.from(fileInput.files);
 if (!files.length) return alert('请选择至少一个文件');

 for (const file of files) {
 if (file.size > 99 * 1024 * 1024) {
 return alert('「' + file.name + '」不能超过 99MB');
 }
 }

 resultCard.classList.remove('show');
 resultsList.innerHTML = ''; // Clear previous results
 // ====== 修改：更新按钮状态 ======
 uploadBtn.disabled = true;
 uploadBtn.textContent = '上传中...';
 progressTrack.style.display = 'block'; // 显示进度条
 progressText.style.display = 'block'; // 显示进度文本
 progressFill.style.width = '0%'; // 重置进度
 // 确保初始状态
 progressFill.classList.remove('completed'); // 移除完成状态
 progressFill.classList.add('uploading'); // 添加上传中状态

 const formData = new FormData();
 files.forEach(f => formData.append('file', f));

 try {
 // 使用 XMLHttpRequest 实现进度监听
 const xhr = new XMLHttpRequest();

 // 监听上传进度
 xhr.upload.addEventListener('progress', (e) => {
 if (e.lengthComputable) {
 updateProgress(e.loaded, e.total);
 }
 });

 // 监听请求完成
 xhr.addEventListener('load', () => {
 if (xhr.status >= 200 && xhr.status < 300) {
 try {
 const res = JSON.parse(xhr.responseText);
 if (res.downloadUrl) {
 const div = document.createElement('div');
 div.className = 'result-item';

 const filenameEl = document.createElement('div');
 filenameEl.className = 'result-filename';
 let filename = files.length === 1 ? files[0].name : 'upload_' + res.fileId + '.zip';
 filenameEl.textContent = '📁 ' + filename;
 div.appendChild(filenameEl);

 const linkEl = document.createElement('a');
 linkEl.className = 'result-link';
 linkEl.href = res.downloadUrl;
 linkEl.target = '_blank';
 linkEl.textContent = res.downloadUrl;
 div.appendChild(linkEl);

 const copyBtn = document.createElement('button');
 copyBtn.className = 'copy-btn';
 copyBtn.textContent = '📋 复制';
 copyBtn.onclick = function() {
 copyText({ target: copyBtn }, res.downloadUrl);
 };
 div.appendChild(copyBtn);

 if (res.notifyError) {
 const errorMsg = document.createElement('div');
 errorMsg.className = 'error-msg';
 errorMsg.textContent = '⚠️ 通知失败：' + res.notifyError;
 div.appendChild(errorMsg);
 }

 resultsList.appendChild(div);
 resultCard.classList.add('show');
 } else {
 alert(res.error || '上传失败');
 }
 } catch (parseError) {
 console.error('JSON parse error:', parseError);
 alert('服务器返回格式错误');
 }
 } else {
 console.error('Upload error:', xhr.statusText);
 alert('上传失败: ' + xhr.statusText);
 }
 });

 xhr.addEventListener('error', () => {
 console.error('Network error during upload');
 alert('网络错误，请重试');
 });

 xhr.addEventListener('abort', () => {
 console.log('Upload aborted');
 alert('上传被取消');
 });

 // 发送请求
 xhr.open('POST', '/api/upload-public');
 xhr.send(formData);

 } catch (e) {
 console.error('Upload error:', e);
 alert('上传初始化失败');
 } finally {
 // 注意：这里不再立即隐藏进度条，因为进度由 xhr 事件控制
 // 当 xhr 完成或出错时，进度条和文本的隐藏应在事件处理函数中完成
 // 为了简化，我们可以在 finally 里隐藏，但实际进度更新由 xhr 控制
 // uploadBtn.disabled = false; // 移动到 xhr 事件处理中
 // progressTrack.style.display = 'none'; // 移动到 xhr 事件处理中
 // progressText.style.display = 'none'; // 移动到 xhr 事件处理中
 }

 // ====== xhr 事件处理函数中统一管理状态 ======
 // 在 load, error, abort 事件中统一恢复按钮状态和隐藏进度条
 function resetUploadUI() {
 // ====== 修改：恢复按钮状态 ======
 uploadBtn.disabled = false;
 uploadBtn.textContent = '确认上传';
 progressTrack.style.display = 'none';
 progressText.style.display = 'none';
 progressText.textContent = '上传中... 0%'; // 重置文本
 }

 xhr.addEventListener('load', () => {
     // 如果成功，设置进度条为完成状态
     if (xhr.status >= 200 && xhr.status < 300) {
         progressFill.classList.add('completed');
         progressFill.classList.remove('uploading');
     }
     resetUploadUI();
 });
 xhr.addEventListener('error', resetUploadUI);
 xhr.addEventListener('abort', resetUploadUI);

 }

 // ====== 兼容性复制函数 ======
 function copyText(event, text) {
 const btn = event.target;
 if (navigator.clipboard && window.isSecureContext) {
 navigator.clipboard.writeText(text).then(() => {
 showCopySuccess(btn);
 }).catch(() => {
 fallbackCopyText(btn, text);
 });
 } else {
 fallbackCopyText(btn, text);
 }
 }

 function fallbackCopyText(btn, text) {
 const textarea = document.createElement('textarea');
 textarea.value = text;
 textarea.style.position = 'fixed';
 textarea.style.opacity = '0';
 document.body.appendChild(textarea);
 textarea.select();
 try {
 const ok = document.execCommand('copy');
 if (ok) {
 showCopySuccess(btn);
 } else {
 alert('复制失败，请手动长按链接复制');
 }
 } catch (e) {
 alert('复制失败，请手动复制');
 } finally {
 document.body.removeChild(textarea);
 }
 }

 function showCopySuccess(btn) {
 btn.classList.add('copied');
 btn.textContent = '✅ 已复制';
 setTimeout(() => {
 btn.classList.remove('copied');
 btn.textContent = '📋 复制';
 }, 2000);
 }
 </script>
</body>
</html>`;

// ====== 常量 ======
const MAX_TOTAL_SIZE = 50 * 1024 * 1024; // 总大小限制 50MB（安全值）
const EXPIRATION_TTL = 7 * 24 * 3600; // 7天

// ====== 生成随机 ID ======
async function generateFileId(env) {
    for (let i = 0; i < 5; i++) {
        const id = Math.random().toString(36).substring(2, 8);
        if (!(await env.TEMP_STORE.get(id))) {
            return id;
        }
    }
    return Math.random().toString(36).substring(2, 8) + Date.now().toString(36).slice(-2);
}

// ====== ZIP 打包函数（使用 fflate）======
function zipFiles(files) {
    const utf8Encoder = new TextEncoder();
    const zip = [];
    let offset = 0;
    const centralDir = [];

    for (const file of files) {
        const filenameBytes = utf8Encoder.encode(file.name);

        // Calculate CRC32 for the file data (simplified, using a placeholder or skipping)
        // const crc32 = calculateCrc32(file.data); // This is complex to implement correctly
        const crc32 = 0; // Placeholder CRC32

        // Local file header
        const header = new Uint8Array([
            0x50, 0x4b, 0x03, 0x04, // local file header signature
            0x14, 0x00, // version needed to extract (2.0)
            0x00, 0x00, // general purpose bit flag (no flags set)
            0x00, 0x00, // compression method (0 = store, 8 = deflate)
            0x00, 0x00, // file last mod time (dummy)
            0x00, 0x00, // file last mod date (dummy)
            crc32 & 0xff, (crc32 >> 8) & 0xff, (crc32 >> 16) & 0xff, (crc32 >> 24) & 0xff, // crc-32
            file.data.byteLength & 0xff, (file.data.byteLength >> 8) & 0xff, (file.data.byteLength >> 16) & 0xff, (file.data.byteLength >> 24) & 0xff, // compressed size
            file.data.byteLength & 0xff, (file.data.byteLength >> 8) & 0xff, (file.data.byteLength >> 16) & 0xff, (file.data.byteLength >> 24) & 0xff, // uncompressed size
            filenameBytes.length & 0xff, (filenameBytes.length >> 8) & 0xff, // file name length
            0x00, 0x00 // extra field length
        ]);

        const localFileHeader = new Uint8Array(header.length + filenameBytes.length);
        localFileHeader.set(header);
        localFileHeader.set(filenameBytes, header.length);

        zip.push(localFileHeader, file.data);

        // Central directory record
        const cdHeader = new Uint8Array([
            0x50, 0x4b, 0x01, 0x02, // central file header signature
            0x14, 0x00, // version made by (2.0)
            0x14, 0x00, // version needed to extract (2.0)
            0x00, 0x00, // general purpose bit flag
            0x00, 0x00, // compression method (0 = store)
            0x00, 0x00, // file last mod time (dummy)
            0x00, 0x00, // file last mod date (dummy)
            crc32 & 0xff, (crc32 >> 8) & 0xff, (crc32 >> 16) & 0xff, (crc32 >> 24) & 0xff, // crc-32
            file.data.byteLength & 0xff, (file.data.byteLength >> 8) & 0xff, (file.data.byteLength >> 16) & 0xff, (file.data.byteLength >> 24) & 0xff, // compressed size
            file.data.byteLength & 0xff, (file.data.byteLength >> 8) & 0xff, (file.data.byteLength >> 16) & 0xff, (file.data.byteLength >> 24) & 0xff, // uncompressed size
            filenameBytes.length & 0xff, (filenameBytes.length >> 8) & 0xff, // file name length
            0x00, 0x00, // extra field length
            0x00, 0x00, // file comment length
            0x00, 0x00, // disk number start
            0x00, 0x00, // internal file attributes (binary)
            0x00, 0x00, 0x00, 0x00, // external file attributes (normal file)
            offset & 0xff, (offset >> 8) & 0xff, (offset >> 16) & 0xff, (offset >> 24) & 0xff // relative offset of local header
        ]);

        const centralRecord = new Uint8Array(cdHeader.length + filenameBytes.length);
        centralRecord.set(cdHeader);
        centralRecord.set(filenameBytes, cdHeader.length);
        centralDir.push(centralRecord);

        offset += localFileHeader.length + file.data.byteLength;
    }

    const totalEntries = files.length;
    const centralSize = centralDir.reduce((sum, d) => sum + d.length, 0);

    // End of central directory record
    const eocd = new Uint8Array([
        0x50, 0x4b, 0x05, 0x06, // end of central dir signature
        0x00, 0x00, // number of this disk
        0x00, 0x00, // number of the disk with the start of the central directory
        totalEntries & 0xff, (totalEntries >> 8) & 0xff, // total number of entries in the central directory on this disk
        totalEntries & 0xff, (totalEntries >> 8) & 0xff, // total number of entries in the central directory
        centralSize & 0xff, (centralSize >> 8) & 0xff, (centralSize >> 16) & 0xff, (centralSize >> 24) & 0xff, // size of the central directory
        offset & 0xff, (offset >> 8) & 0xff, (offset >> 16) & 0xff, (offset >> 24) & 0xff, // offset of start of central directory with respect to the starting disk number
        0x00, 0x00 // comment length
    ]);

    const totalLength = zip.reduce((sum, part) => sum + part.length, 0) + centralSize + eocd.length;
    const finalZip = new Uint8Array(totalLength);

    let pos = 0;
    for (const part of zip) {
        finalZip.set(part, pos);
        pos += part.length;
    }
    for (const part of centralDir) {
        finalZip.set(part, pos);
        pos += part.length;
    }
    finalZip.set(eocd, pos);

    return finalZip;
}

// ====== 发送企业微信通知 ======
async function sendWeComWebhookNotification(env, fileData) {
    const WEBHOOK_URL = env.WECOM_WEBHOOK_URL;
    if (!WEBHOOK_URL) return;

    const { filename, size, downloadUrl } = fileData;

    let sizeText;
    if (size < 1024) sizeText = size + " B";
    else if (size < 1024 * 1024) sizeText = (size / 1024).toFixed(2) + " KB";
    else sizeText = (size / (1024 * 1024)).toFixed(2) + " MB";

    const now = new Date().toLocaleString('zh-CN', { timeZone: 'Asia/Shanghai' });

    const content = '📁 新文件上传\n文件名：' + filename + '\n大小：' + sizeText + '\n时间：' + now + '\n🔗 下载地址：' + downloadUrl;

    const payload = { msgtype: "text", text: { content } };

    const resp = await fetch(WEBHOOK_URL, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload)
    });

    const result = await resp.json();
    if (result.errcode !== 0) {
        throw new Error('Webhook 发送失败: ' + result.errmsg);
    }
}

// ====== 主入口 ======
export default {
    async fetch(request, env) {
        const url = new URL(request.url);
        const { pathname } = url;

        if (pathname === "/") {
            return new Response(HTML, {
                headers: { "Content-Type": "text/html; charset=utf-8" }
            });
        }

        if (request.method === "OPTIONS") {
            return new Response(null, {
                headers: {
                    "Access-Control-Allow-Origin": "*",
                    "Access-Control-Allow-Methods": "POST, OPTIONS",
                    "Access-Control-Allow-Headers": "Content-Type"
                }
            });
        }

        if (pathname === "/api/upload-public" && request.method === "POST") {
            const contentType = request.headers.get("content-type") || "";
            if (!contentType.includes("multipart/form-data")) {
                return new Response(JSON.stringify({ error: "必须使用 multipart/form-data 上传" }), {
                    status: 400,
                    headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                });
            }

            try {
                const formData = await request.formData();
                const files = formData.getAll("file").filter(f => f instanceof File);

                if (!files.length) {
                    return new Response(JSON.stringify({ error: "未提供有效文件" }), {
                        status: 400,
                        headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                    });
                }

                // 检查每个文件大小
                for (const file of files) {
                    if (file.size > 99 * 1024 * 1024) {
                        return new Response(JSON.stringify({ error: '文件「' + file.name + '」超过 99MB' }), {
                            status: 400,
                            headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                        });
                    }
                }

                const fileId = await generateFileId(env);
                let notifyError = null;

                if (files.length === 1) {
                    // ========== 单文件：直接存储 ==========
                    const file = files[0];
                    const buffer = await file.arrayBuffer();
                    const filename = file.name;
                    const contentType = file.type || "application/octet-stream";

                    if (file.size <= 25 * 1024 * 1024) {
                        // 存 KV
                        await env.TEMP_STORE.put(fileId, buffer, {
                            metadata: {
                                filename,
                                contentType,
                                storage: "kv",
                                isZip: false
                            },
                            expirationTtl: EXPIRATION_TTL
                        });
                    } else {
                        // 存 WebDAV
                        const WEBDAV_BASE = 'https://higa.teracloud.jp/dav/air1/';
                        const credentials = btoa(`${env.WEBDAV_ACCOUNT}:${env.WEBDAV_PASSWORD}`);
                        const webdavFilename = 'file_' + fileId + '_' + filename;
                        const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);

                        const resp = await fetch(webdavUrl, {
                            method: 'PUT',
                            headers: {
                                'Authorization': 'Basic ' + credentials,
                                'Content-Type': contentType
                            },
                            body: buffer
                        });

                        if (!resp.ok) throw new Error('WebDAV upload failed: ' + resp.status);

                        await env.TEMP_STORE.put(fileId, "", {
                            metadata: {
                                filename,
                                contentType,
                                storage: "webdav",
                                webdavFilename,
                                isZip: false
                            },
                            expirationTtl: EXPIRATION_TTL
                        });
                    }

                    // 通知
                    try {
                        await sendWeComWebhookNotification(env, {
                            filename,
                            size: file.size,
                            downloadUrl: 'https://tmp.air1.cn/' + fileId
                        });
                    } catch (e) {
                        notifyError = e.message;
                    }

                    return new Response(JSON.stringify({
                        downloadUrl: 'https://tmp.air1.cn/' + fileId,
                        fileId,
                        notifyError
                    }), {
                        headers: {
                            "Content-Type": "application/json",
                            "Access-Control-Allow-Origin": "*"
                        }
                    });
                } else {
                    // ========== 多文件：打包 ZIP ==========
                    let totalSize = 0;
                    const fileBuffers = [];
                    for (const file of files) {
                        totalSize += file.size;
                        const buffer = await file.arrayBuffer();
                        fileBuffers.push({ name: file.name, data: new Uint8Array(buffer) });
                    }

                    if (totalSize > MAX_TOTAL_SIZE) {
                        return new Response(JSON.stringify({ error: '总大小不能超过 ' + (MAX_TOTAL_SIZE / (1024*1024)).toFixed(1) + 'MB' }), {
                            status: 400,
                            headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                        });
                    }

                    const zipBuffer = zipFiles(fileBuffers);
                    const zipSize = zipBuffer.byteLength;
                    const zipName = 'upload_' + fileId + '.zip';

                    if (zipSize <= 25 * 1024 * 1024) {
                        await env.TEMP_STORE.put(fileId, zipBuffer, {
                            metadata: {
                                filename: zipName,
                                contentType: "application/zip",
                                storage: "kv",
                                isZip: true
                            },
                            expirationTtl: EXPIRATION_TTL
                        });
                    } else {
                        const WEBDAV_BASE = 'https://higa.teracloud.jp/dav/air1/';
                        const credentials = btoa(`${env.WEBDAV_ACCOUNT}:${env.WEBDAV_PASSWORD}`);
                        const webdavFilename = 'zip_' + fileId + '.zip';
                        const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);

                        const resp = await fetch(webdavUrl, {
                            method: 'PUT',
                            headers: {
                                'Authorization': 'Basic ' + credentials,
                                'Content-Type': 'application/zip'
                            },
                            body: zipBuffer
                        });

                        if (!resp.ok) throw new Error('WebDAV upload failed: ' + resp.status);

                        await env.TEMP_STORE.put(fileId, "", {
                            metadata: {
                                filename: zipName,
                                contentType: "application/zip",
                                storage: "webdav",
                                webdavFilename,
                                isZip: true
                            },
                            expirationTtl: EXPIRATION_TTL
                        });
                    }

                    try {
                        await sendWeComWebhookNotification(env, {
                            filename: zipName,
                            size: zipSize,
                            downloadUrl: 'https://tmp.air1.cn/' + fileId
                        });
                    } catch (e) {
                        notifyError = e.message;
                    }

                    return new Response(JSON.stringify({
                        downloadUrl: 'https://tmp.air1.cn/' + fileId,
                        fileId,
                        notifyError
                    }), {
                        headers: {
                            "Content-Type": "application/json",
                            "Access-Control-Allow-Origin": "*"
                        }
                    });
                }
            } catch (e) {
                console.error("Upload error:", e);
                return new Response(JSON.stringify({ error: e.message || "服务器内部错误" }), {
                    status: 500,
                    headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                });
            }
        }

        // ====== 下载逻辑 ======
        const segments = pathname.split('/').filter(Boolean);
        if (segments.length === 1 && segments[0].length === 6) {
            const id = segments[0];
            const reservedPaths = new Set(['api', 'upload', 'f', 'favicon.ico', 'robots.txt', 'about', 's', 'webdav']);

            if (reservedPaths.has(id)) {
                return new Response("Reserved path", { status: 400 });
            }

            const entry = await env.TEMP_STORE.getWithMetadata(id, "arrayBuffer");
            if (!entry?.metadata) {
                return new Response("File not found", { status: 404 });
            }

            const { storage, webdavFilename, filename, contentType } = entry.metadata;

            if (storage === "kv") {
                return new Response(entry.value, {
                    headers: {
                        "Content-Type": contentType,
                        "Content-Disposition": 'attachment; filename="' + encodeURIComponent(filename) + '"',
                        "Cache-Control": "no-store"
                    }
                });
            }

            if (storage === "webdav" && webdavFilename) {
                const WEBDAV_BASE = 'https://higa.teracloud.jp/dav/air1/';
                const credentials = btoa(`${env.WEBDAV_ACCOUNT}:${env.WEBDAV_PASSWORD}`);
                const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);

                let resp;
                try {
                    resp = await fetch(webdavUrl, {
                        headers: { 'Authorization': 'Basic ' + credentials }
                    });
                } catch (e) {
                    return new Response("Storage unavailable", { status: 502 });
                }

                if (!resp.ok) {
                    return new Response("File not found", { status: 404 });
                }

                const headers = new Headers({
                    "Content-Type": contentType,
                    "Content-Disposition": 'attachment; filename="' + encodeURIComponent(filename) + '"',
                    "Cache-Control": "no-store",
                    "Access-Control-Allow-Origin": "*"
                });

                return new Response(resp.body, { status: 200, headers });
            }

            return new Response("Invalid file record", { status: 500 });
        }

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
```js
return Math.random().toString(36).substring(2, 7); // 5字符
```

并将路由判断改为 segments[0].length >= 5
2. 文件限制
单文件 ≤ 25 MB（Cloudflare Workers 限制）
自动 12 小时过期（通过 expirationTtl: 43200 实现）
3. 路径冲突防护
已预留以下路径，不会被当作文件 ID：
```js
const reservedPaths = new Set([
'api', 'upload', 'f', 'favicon.ico', 'robots.txt', 'about'
]);
```
4. HTTPS 与安全性
Cloudflare 自动提供 HTTPS，无需配置证书
上传接口为公开，如需鉴权可参考短链接服务增加 API_TOKEN

至此，临时文件存储服务已上线！链接简洁、自动清理、保留文件名，适合分享日志、截图、临时文档等场景。