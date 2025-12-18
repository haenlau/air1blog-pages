---
title: 利用闲置的云盘搭建临时文件存储服务
date: 2025-12-07 21:22:31
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
const HTML = 
'<!DOCTYPE html>\n' +
'<html lang="zh-CN">\n' +
'<head>\n' +
'  <meta charset="utf-8" />\n' +
'  <meta name="viewport" content="width=device-width, initial-scale=1" />\n' +
'  <title>✨ Air1 TempFile</title>\n' +
'  <link rel="icon" type="image/png" href="https://air1.cn/favicon.png" />\n' +
'  <style>\n' +
'    :root {\n' +
'      --bg: #0f0f12;\n' +
'      --card-bg: rgba(255, 255, 255, 0.06);\n' +
'      --card-border: rgba(255, 255, 255, 0.1);\n' +
'      --text: #f0f0f5;\n' +
'      --muted: #a0a0b0;\n' +
'      --primary: #4d9fff;\n' +
'      --success: #4ade80;\n' +
'      --danger: #f87171;\n' +
'      --shadow: 0 8px 32px rgba(0, 0, 0, 0.3);\n' +
'      --radius: 18px;\n' +
'    }\n' +
'\n' +
'    @media (prefers-color-scheme: light) {\n' +
'      :root {\n' +
'        --bg: #f8f9ff;\n' +
'        --card-bg: rgba(255, 255, 255, 0.7);\n' +
'        --card-border: rgba(0, 0, 0, 0.08);\n' +
'        --text: #1a1a25;\n' +
'        --muted: #666;\n' +
'        --primary: #2563eb;\n' +
'        --success: #16a34a;\n' +
'        --danger: #dc2626;\n' +
'        --shadow: 0 8px 24px rgba(0, 0, 0, 0.08);\n' +
'      }\n' +
'    }\n' +
'\n' +
'    * {\n' +
'      margin: 0;\n' +
'      padding: 0;\n' +
'      box-sizing: border-box;\n' +
'    }\n' +
'\n' +
'    body {\n' +
'      background: var(--bg);\n' +
'      color: var(--text);\n' +
'      font-family: \'SF Pro Display\', -apple-system, BlinkMacSystemFont, \'Segoe UI\', sans-serif;\n' +
'      min-height: 100vh;\n' +
'      display: flex;\n' +
'      flex-direction: column;\n' +
'      align-items: center;\n' +
'      justify-content: center;\n' +
'      padding: 2rem 1.5rem;\n' +
'      position: relative;\n' +
'      overflow-x: hidden;\n' +
'    }\n' +
'\n' +
'    body::before {\n' +
'      content: "";\n' +
'      position: absolute;\n' +
'      top: 0;\n' +
'      left: 0;\n' +
'      width: 100%;\n' +
'      height: 100%;\n' +
'      background: \n' +
'        radial-gradient(circle at 20% 30%, rgba(77, 159, 255, 0.06) 0%, transparent 40%),\n' +
'        radial-gradient(circle at 80% 70%, rgba(77, 159, 255, 0.04) 0%, transparent 50%);\n' +
'      pointer-events: none;\n' +
'      z-index: -1;\n' +
'    }\n' +
'\n' +
'    .container {\n' +
'      width: 100%;\n' +
'      max-width: 520px;\n' +
'      text-align: center;\n' +
'    }\n' +
'\n' +
'    h1 {\n' +
'      font-weight: 700;\n' +
'      font-size: 2.2rem;\n' +
'      margin-bottom: 0.4rem;\n' +
'      background: linear-gradient(135deg, #ffffff, #a0a0ff);\n' +
'      -webkit-background-clip: text;\n' +
'      background-clip: text;\n' +
'      color: transparent;\n' +
'      background-size: 200% 200%;\n' +
'      animation: gradientShift 8s ease infinite;\n' +
'    }\n' +
'\n' +
'    @keyframes gradientShift {\n' +
'      0% { background-position: 0% 50%; }\n' +
'      50% { background-position: 100% 50%; }\n' +
'      100% { background-position: 0% 50%; }\n' +
'    }\n' +
'\n' +
'    .subtitle {\n' +
'      color: var(--muted);\n' +
'      font-size: 0.95rem;\n' +
'      margin-bottom: 2.2rem;\n' +
'    }\n' +
'\n' +
'    .upload-card {\n' +
'      background: var(--card-bg);\n' +
'      border: 1px solid var(--card-border);\n' +
'      backdrop-filter: blur(12px);\n' +
'      -webkit-backdrop-filter: blur(12px);\n' +
'      border-radius: var(--radius);\n' +
'      padding: 2.2rem 1.5rem;\n' +
'      margin-bottom: 1.8rem;\n' +
'      cursor: pointer;\n' +
'      transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);\n' +
'      position: relative;\n' +
'      overflow: hidden;\n' +
'    }\n' +
'\n' +
'    .upload-card:hover {\n' +
'      transform: translateY(-4px);\n' +
'      box-shadow: var(--shadow);\n' +
'      border-color: rgba(77, 159, 255, 0.3);\n' +
'    }\n' +
'\n' +
'    .upload-card.dragover {\n' +
'      border-color: var(--primary);\n' +
'      background: rgba(77, 159, 255, 0.08);\n' +
'    }\n' +
'\n' +
'    .upload-icon {\n' +
'      font-size: 3.2rem;\n' +
'      margin-bottom: 1.2rem;\n' +
'      display: block;\n' +
'      transition: transform 0.3s;\n' +
'    }\n' +
'\n' +
'    .upload-card:hover .upload-icon {\n' +
'      transform: scale(1.1) rotate(3deg);\n' +
'    }\n' +
'\n' +
'    .upload-text {\n' +
'      font-size: 1.1rem;\n' +
'      font-weight: 500;\n' +
'      margin-bottom: 0.4rem;\n' +
'    }\n' +
'\n' +
'    .upload-hint {\n' +
'      font-size: 0.85rem;\n' +
'      color: var(--muted);\n' +
'    }\n' +
'\n' +
'    .selected-file {\n' +
'      margin-top: 0.6rem;\n' +
'      font-size: 0.85rem;\n' +
'      color: var(--primary);\n' +
'      display: none;\n' +
'    }\n' +
'\n' +
'    #fileInput {\n' +
'      display: none;\n' +
'    }\n' +
'\n' +
'    .btn {\n' +
'      width: 100%;\n' +
'      padding: 0.95rem;\n' +
'      background: var(--primary);\n' +
'      color: white;\n' +
'      border: none;\n' +
'      border-radius: 14px;\n' +
'      font-size: 1.05rem;\n' +
'      font-weight: 600;\n' +
'      cursor: pointer;\n' +
'      transition: all 0.25s;\n' +
'      letter-spacing: 0.3px;\n' +
'    }\n' +
'\n' +
'    .btn:hover:not(:disabled) {\n' +
'      background: #3a8bff;\n' +
'      transform: translateY(-2px);\n' +
'      box-shadow: 0 6px 16px rgba(77, 159, 255, 0.3);\n' +
'    }\n' +
'\n' +
'    .btn:disabled {\n' +
'      opacity: 0.7;\n' +
'      cursor: not-allowed;\n' +
'      transform: none;\n' +
'      box-shadow: none;\n' +
'    }\n' +
'\n' +
'    .result-card {\n' +
'      background: var(--card-bg);\n' +
'      border: 1px solid var(--card-border);\n' +
'      backdrop-filter: blur(12px);\n' +
'      -webkit-backdrop-filter: blur(12px);\n' +
'      border-radius: var(--radius);\n' +
'      padding: 1.6rem;\n' +
'      margin-top: 1.5rem;\n' +
'      display: none;\n' +
'    }\n' +
'\n' +
'    .result-card.show {\n' +
'      display: block;\n' +
'      animation: fadeIn 0.4s ease;\n' +
'    }\n' +
'\n' +
'    @keyframes fadeIn {\n' +
'      from { opacity: 0; transform: translateY(10px); }\n' +
'      to { opacity: 1; transform: translateY(0); }\n' +
'    }\n' +
'\n' +
'    .result-title {\n' +
'      font-size: 1.1rem;\n' +
'      margin-bottom: 1rem;\n' +
'      color: var(--success);\n' +
'      font-weight: 600;\n' +
'    }\n' +
'\n' +
'    .result-item {\n' +
'      margin-bottom: 1rem;\n' +
'      text-align: center;\n' +
'      padding: 0.8rem;\n' +
'      background: rgba(77, 159, 255, 0.05);\n' +
'      border-radius: 10px;\n' +
'      display: flex;\n' +
'      flex-direction: column;\n' +
'      align-items: center;\n' +
'    }\n' +
'    .result-filename {\n' +
'      font-weight: bold;\n' +
'      margin-bottom: 0.3rem;\n' +
'      color: var(--text);\n' +
'    }\n' +
'\n' +
'    .result-link {\n' +
'      display: block;\n' +
'      word-break: break-all;\n' +
'      color: var(--primary);\n' +
'      text-decoration: none;\n' +
'      font-size: 0.9rem;\n' +
'      font-family: monospace;\n' +
'      margin: 0.2rem 0;\n' +
'    }\n' +
'\n' +
'    .copy-btn {\n' +
'      background: rgba(255, 255, 255, 0.12);\n' +
'      color: var(--text);\n' +
'      border: none;\n' +
'      padding: 0.4rem 0.8rem;\n' +
'      border-radius: 8px;\n' +
'      font-weight: 600;\n' +
'      cursor: pointer;\n' +
'      transition: all 0.2s;\n' +
'      font-size: 0.85rem;\n' +
'    }\n' +
'\n' +
'    .copy-btn:hover {\n' +
'      background: rgba(255, 255, 255, 0.2);\n' +
'    }\n' +
'\n' +
'    .copy-btn.copied {\n' +
'      background: var(--success);\n' +
'      color: white;\n' +
'    }\n' +
'\n' +
'    .error-msg {\n' +
'      color: var(--danger);\n' +
'      margin-top: 0.5rem;\n' +
'      font-size: 0.85rem;\n' +
'    }\n' +
'\n' +
'    .progress-track {\n' +
'      height: 6px;\n' +
'      background: rgba(255, 255, 255, 0.1);\n' +
'      border-radius: 3px;\n' +
'      margin-top: 1.2rem;\n' +
'      overflow: hidden;\n' +
'    }\n' +
'\n' +
'    .progress-fill {\n' +
'      height: 100%;\n' +
'      background: var(--primary);\n' +
'      width: 0%;\n' +
'      border-radius: 3px;\n' +
'      transition: width 0.2s ease;\n' +
'    }\n' +
'\n' +
'    .footer {\n' +
'      margin-top: 2.5rem;\n' +
'      color: var(--muted);\n' +
'      font-size: 0.8rem;\n' +
'      opacity: 0.8;\n' +
'    }\n' +
'\n' +
'    @media (max-width: 480px) {\n' +
'      h1 { font-size: 1.8rem; }\n' +
'      .upload-card { padding: 1.8rem 1rem; }\n' +
'    }\n' +
'  </style>\n' +
'</head>\n' +
'<body>\n' +
'  <div class="container">\n' +
'    <h1>Air1 TempFile</h1>\n' +
'    <p class="subtitle">安全上传 · 多文件自动打包 · 7天自动销毁</p>\n' +
'\n' +
'    <div class="upload-card" id="dropArea">\n' +
'      <span class="upload-icon">📤</span>\n' +
'      <p class="upload-text">拖拽文件或点击上传</p>\n' +
'      <p class="upload-hint">支持任意格式</p>\n' +
'      <p class="selected-file" id="selectedFile"></p>\n' +
'      <input type="file" id="fileInput" multiple />\n' +
'    </div>\n' +
'\n' +
'    <button class="btn" id="uploadBtn" onclick="uploadFiles()">确认上传</button>\n' +
'\n' +
'    <div class="progress-track" id="progressTrack" style="display:none;">\n' +
'      <div class="progress-fill" id="progressFill"></div>\n' +
'    </div>\n' +
'\n' +
'    <div class="result-card" id="resultCard">\n' +
'      <div class="result-title">✅ 上传完成</div>\n' +
'      <div id="resultsList"></div>\n' +
'    </div>\n' +
'\n' +
'    <p class="footer">Part of <strong>Air1 Quick Tools</strong> · Powered by Cloudflare</p>\n' +
'  </div>\n' +
'\n' +
'  <script>\n' +
'    const dropArea = document.getElementById(\'dropArea\');\n' +
'    const fileInput = document.getElementById(\'fileInput\');\n' +
'    const uploadBtn = document.getElementById(\'uploadBtn\');\n' +
'    const resultCard = document.getElementById(\'resultCard\');\n' +
'    const resultsList = document.getElementById(\'resultsList\');\n' +
'    const progressTrack = document.getElementById(\'progressTrack\');\n' +
'    const progressFill = document.getElementById(\'progressFill\');\n' +
'    const selectedFileEl = document.getElementById(\'selectedFile\');\n' +
'\n' +
'    [\'dragenter\', \'dragover\', \'dragleave\', \'drop\'].forEach(e => {\n' +
'      dropArea.addEventListener(e, preventDefaults, false);\n' +
'    });\n' +
'\n' +
'    function preventDefaults(e) {\n' +
'      e.preventDefault();\n' +
'      e.stopPropagation();\n' +
'    }\n' +
'\n' +
'    [\'dragenter\', \'dragover\'].forEach(e => {\n' +
'      dropArea.addEventListener(e, () => dropArea.classList.add(\'dragover\'), false);\n' +
'    });\n' +
'\n' +
'    [\'dragleave\', \'drop\'].forEach(e => {\n' +
'      dropArea.addEventListener(e, () => dropArea.classList.remove(\'dragover\'), false);\n' +
'    });\n' +
'\n' +
'    dropArea.addEventListener(\'drop\', e => {\n' +
'      fileInput.files = e.dataTransfer.files;\n' +
'      fileInput.dispatchEvent(new Event(\'change\'));\n' +
'    });\n' +
'\n' +
'    dropArea.addEventListener(\'click\', () => fileInput.click());\n' +
'\n' +
'    fileInput.addEventListener(\'change\', () => {\n' +
'      const files = fileInput.files;\n' +
'      if (files.length > 0) {\n' +
'        let names = Array.from(files).map(f => f.name).join(\', \');\n' +
'        if (names.length > 60) names = names.substring(0, 60) + \'...\';\n' +
'        selectedFileEl.textContent = `已选择 ${files.length} 个文件：${names}`;\n' +
'        selectedFileEl.style.display = \'block\';\n' +
'      } else {\n' +
'        selectedFileEl.style.display = \'none\';\n' +
'      }\n' +
'    });\n' +
'\n' +
'    async function uploadFiles() {\n' +
'      const files = Array.from(fileInput.files);\n' +
'      if (!files.length) return alert(\'请选择至少一个文件\');\n' +
'\n' +
'      for (const file of files) {\n' +
'        if (file.size > 99 * 1024 * 1024) {\n' +
'          return alert(`「${file.name}」不能超过 99MB`);\n' +
'        }\n' +
'      }\n' +
'\n' +
'      resultCard.classList.remove(\'show\');\n' +
'      resultsList.innerHTML = \'\';\n' +
'      uploadBtn.disabled = true;\n' +
'      progressTrack.style.display = \'block\';\n' +
'      progressFill.style.width = \'0%\';\n' +
'\n' +
'      const formData = new FormData();\n' +
'      files.forEach(f => formData.append(\'file\', f));\n' +
'\n' +
'      try {\n' +
'        const response = await fetch(\'/api/upload-public\', {\n' +
'          method: \'POST\',\n' +
'          body: formData,\n' +
'        });\n' +
'\n' +
'        const res = await response.json();\n' +
'\n' +
'        if (response.ok && res.downloadUrl) {\n' +
'          const div = document.createElement(\'div\');\n' +
'          div.className = \'result-item\';\n' +
'          let filename = files.length === 1 ? files[0].name : `upload_${res.fileId}.zip`;\n' +
'          div.innerHTML = `\n' +
'            <div class="result-filename">📁 ${filename}</div>\n' +
'            <a class="result-link" href="${res.downloadUrl}" target="_blank">${res.downloadUrl}</a>\n' +
'            <button class="copy-btn" onclick="copyText(event, \'${res.downloadUrl}\')">📋 复制</button>\n' +
'            ${res.notifyError ? `<div class="error-msg">⚠️ 通知失败：${res.notifyError}</div>` : \'\'}\n' +
'          `;\n' +
'          resultsList.appendChild(div);\n' +
'          resultCard.classList.add(\'show\');\n' +
'        } else {\n' +
'          alert(res.error || \'上传失败\');\n' +
'        }\n' +
'      } catch (e) {\n' +
'        alert(\'网络错误，请重试\');\n' +
'      } finally {\n' +
'        uploadBtn.disabled = false;\n' +
'      }\n' +
'    }\n' +
'\n' +
'    // ====== 兼容性复制函数 ======\n' +
'    function copyText(event, text) {\n' +
'      if (navigator.clipboard && window.isSecureContext) {\n' +
'        navigator.clipboard.writeText(text).then(() => {\n' +
'          showCopySuccess(event.target);\n' +
'        }).catch(() => {\n' +
'          fallbackCopyText(event.target, text);\n' +
'        });\n' +
'      } else {\n' +
'        fallbackCopyText(event.target, text);\n' +
'      }\n' +
'    }\n' +
'\n' +
'    function fallbackCopyText(btn, text) {\n' +
'      const textarea = document.createElement(\'textarea\');\n' +
'      textarea.value = text;\n' +
'      textarea.style.position = \'fixed\';\n' +
'      textarea.style.opacity = \'0\';\n' +
'      document.body.appendChild(textarea);\n' +
'      textarea.select();\n' +
'      try {\n' +
'        const ok = document.execCommand(\'copy\');\n' +
'        if (ok) {\n' +
'          showCopySuccess(btn);\n' +
'        } else {\n' +
'          alert(\'复制失败，请手动长按链接复制\');\n' +
'        }\n' +
'      } catch (e) {\n' +
'        alert(\'复制失败，请手动复制\');\n' +
'      } finally {\n' +
'        document.body.removeChild(textarea);\n' +
'      }\n' +
'    }\n' +
'\n' +
'    function showCopySuccess(btn) {\n' +
'      btn.classList.add(\'copied\');\n' +
'      btn.textContent = \'✅ 已复制\';\n' +
'      setTimeout(() => {\n' +
'        btn.classList.remove(\'copied\');\n' +
'        btn.textContent = \'📋 复制\';\n' +
'      }, 2000);\n' +
'    }\n' +
'  </script>\n' +
'</body>\n' +
'</html>';

// ====== 常量 ======
const MAX_TOTAL_SIZE = 50 * 1024 * 1024; // 总大小限制 50MB（安全值）
const EXPIRATION_TTL = 7 * 24 * 3600;     // 7天

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
    const header = new Uint8Array([
      0x50, 0x4b, 0x03, 0x04, // local file header signature
      0x14, 0x00,             // version needed to extract
      0x00, 0x00,             // general purpose bit flag
      0x00, 0x00,             // compression method (0 = store)
      0x00, 0x00,             // file last mod time/date (0 = ignored)
      0x00, 0x00, 0x00, 0x00, // crc-32 (skipped for simplicity)
      file.data.byteLength & 0xff, (file.data.byteLength >> 8) & 0xff,
      (file.data.byteLength >> 16) & 0xff, (file.data.byteLength >> 24) & 0xff,
      filenameBytes.length & 0xff, (filenameBytes.length >> 8) & 0xff,
      0x00, 0x00              // extra field length
    ]);

    const localFileHeader = new Uint8Array(header.length + filenameBytes.length);
    localFileHeader.set(header);
    localFileHeader.set(filenameBytes, header.length);

    zip.push(localFileHeader, file.data);

    // Central directory record
    const cdHeader = new Uint8Array([
      0x50, 0x4b, 0x01, 0x02, // central file header signature
      0x14, 0x03,             // version made by
      0x14, 0x00,             // version needed to extract
      0x00, 0x00,             // general purpose bit flag
      0x00, 0x00,             // compression method
      0x00, 0x00,             // file last mod time/date
      0x00, 0x00, 0x00, 0x00, // crc-32 (skipped)
      file.data.byteLength & 0xff, (file.data.byteLength >> 8) & 0xff,
      (file.data.byteLength >> 16) & 0xff, (file.data.byteLength >> 24) & 0xff,
      filenameBytes.length & 0xff, (filenameBytes.length >> 8) & 0xff,
      0x00, 0x00,             // extra field length
      0x00, 0x00,             // file comment length
      0x00, 0x00,             // disk number start
      0x00, 0x00,             // internal file attributes
      0x00, 0x00, 0x00, 0x00, // external file attributes
      offset & 0xff, (offset >> 8) & 0xff,
      (offset >> 16) & 0xff, (offset >> 24) & 0xff
    ]);
    const centralRecord = new Uint8Array(cdHeader.length + filenameBytes.length);
    centralRecord.set(cdHeader);
    centralRecord.set(filenameBytes, cdHeader.length);
    centralDir.push(centralRecord);

    offset += localFileHeader.length + file.data.byteLength;
  }

  const totalEntries = files.length;
  const centralSize = centralDir.reduce((sum, d) => sum + d.length, 0);
  const eocd = new Uint8Array([
    0x50, 0x4b, 0x05, 0x06, // end of central dir signature
    0x00, 0x00,             // number of this disk
    0x00, 0x00,             // number of the disk with the start of the central directory
    totalEntries & 0xff, (totalEntries >> 8) & 0xff,
    totalEntries & 0xff, (totalEntries >> 8) & 0xff,
    centralSize & 0xff, (centralSize >> 8) & 0xff,
    (centralSize >> 16) & 0xff, (centralSize >> 24) & 0xff,
    offset & 0xff, (offset >> 8) & 0xff,
    (offset >> 16) & 0xff, (offset >> 24) & 0xff,
    0x00, 0x00              // comment length
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
  const content = `📁 新文件上传\n\n文件名：${filename}\n大小：${sizeText}\n时间：${now}\n\n🔗 下载地址：\n${downloadUrl}`;

  const payload = { msgtype: "text", text: { content } };
  const resp = await fetch(WEBHOOK_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(payload)
  });

  const result = await resp.json();
  if (result.errcode !== 0) {
    throw new Error(`Webhook 发送失败: ${result.errmsg}`);
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
            return new Response(JSON.stringify({ error: `文件「${file.name}」超过 99MB` }), {
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
            const webdavFilename = `file_${fileId}_${filename}`;
            const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);

            const resp = await fetch(webdavUrl, {
              method: 'PUT',
              headers: {
                'Authorization': `Basic ${credentials}`,
                'Content-Type': contentType
              },
              body: buffer
            });

            if (!resp.ok) throw new Error(`WebDAV upload failed: ${resp.status}`);

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
              downloadUrl: `https://tmp.air1.cn/${fileId}`
            });
          } catch (e) {
            notifyError = e.message;
          }

          return new Response(JSON.stringify({
            downloadUrl: `https://tmp.air1.cn/${fileId}`,
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
            return new Response(JSON.stringify({ error: `总大小不能超过 ${(MAX_TOTAL_SIZE / (1024*1024)).toFixed(1)}MB` }), {
              status: 400,
              headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
            });
          }

          const zipBuffer = zipFiles(fileBuffers);
          const zipSize = zipBuffer.byteLength;
          const zipName = `upload_${fileId}.zip`;

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
            const webdavFilename = `zip_${fileId}.zip`;
            const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);

            const resp = await fetch(webdavUrl, {
              method: 'PUT',
              headers: {
                'Authorization': `Basic ${credentials}`,
                'Content-Type': 'application/zip'
              },
              body: zipBuffer
            });

            if (!resp.ok) throw new Error(`WebDAV upload failed: ${resp.status}`);

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
              downloadUrl: `https://tmp.air1.cn/${fileId}`
            });
          } catch (e) {
            notifyError = e.message;
          }

          return new Response(JSON.stringify({
            downloadUrl: `https://tmp.air1.cn/${fileId}`,
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
            "Content-Disposition": `attachment; filename="${encodeURIComponent(filename)}"`,
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
            headers: { 'Authorization': `Basic ${credentials}` }
          });
        } catch (e) {
          return new Response("Storage unavailable", { status: 502 });
        }

        if (!resp.ok) {
          return new Response("File not found", { status: 404 });
        }

        const headers = new Headers({
          "Content-Type": contentType,
          "Content-Disposition": `attachment; filename="${encodeURIComponent(filename)}"`,
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