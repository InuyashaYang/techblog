---
title: "Claude Code（VS Code）接入第三方 OpenAI 兼容 API：完整操作与踩坑记录（含报错与标准处理）"
date: 2026-02-16 13:00:00 +0800
tags: [Claude Code, VSCode, OpenAI, CCR, Troubleshooting]
description: "在 VS Code 使用 Claude Code，通过 CCR 转发到第三方 OpenAI 兼容 API，并记录关键报错与处理。"
---

目标很简单。在 VS Code 里用 Claude Code，并把请求接到我们自己的第三方 OpenAI 兼容 API 上。
现实也很简单。我登录不了 Claude，所以只能走非官方路径。
最终效果是能聊天，能走通请求，但工具能力是否可用取决于上游对工具调用协议的兼容程度。

这篇文章给出一份可复现的流程，并把我们遇到的关键报错和对应处理写清楚，避免以后再从头受一遍罪。

---

## 一 环境与前提

系统为 Windows。
需要准备以下工具：

- VS Code
- Node.js 和 npm
- Claude Code VS Code 插件
- Claude Code Router，也就是 ccr

另外，如果你也无法登录 Claude，不要在 CLI 上浪费时间。直接从 VS Code 插件路线开始。

---

## 二 安装 Claude Code

PowerShell 执行安装脚本：

```powershell
irm https://claude.ai/install.ps1 | iex
```

检查安装情况：

```powershell
claude doctor
```

这一步通常没问题。真正的问题会在运行 claude 时出现，因为 CLI 会要求登录。

---

## 三 VS Code 插件安装与关闭登录提示

在 VS Code 扩展商店安装 Claude Code 插件，发布者是 Anthropic。

插件默认会弹登录页。我们这次要走第三方转发路线，所以先关闭登录提示。

用 PowerShell 写入 VS Code 用户设置：

```powershell
$settings = Join-Path $env:APPDATA "Code\User\settings.json"
New-Item -ItemType File -Force -Path $settings | Out-Null

$jsonText = Get-Content $settings -Raw
$obj = if ([string]::IsNullOrWhiteSpace($jsonText)) { @{} } else { $jsonText | ConvertFrom-Json }

$obj | Add-Member -Force NoteProperty "claudeCode.disableLoginPrompt" $true
$obj | ConvertTo-Json -Depth 30 | Set-Content -Encoding UTF8 $settings
```

然后在 VS Code 里执行 Reload Window。

---

## 四 安装并启动 CCR

Claude Code 使用的是 Anthropic 的 messages 协议，而我们的第三方服务提供的是 OpenAI 的 chat completions 协议。
需要一个中间层把协议转换一下。这里用 Claude Code Router。

安装：

```powershell
npm install -g @musistudio/claude-code-router
```

启动服务并打开管理界面：

```powershell
ccr start
ccr ui
```

管理界面一般是本机 3456 端口。

---

## 五 让 Claude Code 指向 CCR

Claude Code 插件支持给它启动的进程注入环境变量。我们把 Anthropic Base URL 指向 CCR。

用 PowerShell 写入 VS Code 用户设置：

```powershell
$settings = Join-Path $env:APPDATA "Code\User\settings.json"
$jsonText = Get-Content $settings -Raw
$obj = if ([string]::IsNullOrWhiteSpace($jsonText)) { @{} } else { $jsonText | ConvertFrom-Json }

$obj | Add-Member -Force NoteProperty "claudeCode.environmentVariables" @(
  @{ name = "ANTHROPIC_BASE_URL"; value = "http://127.0.0.1:3456" },
  @{ name = "ANTHROPIC_AUTH_TOKEN"; value = "sk-ant-local-ccr-xxxx" }
)

$obj | ConvertTo-Json -Depth 30 | Set-Content -Encoding UTF8 $settings
```

说明：
- `ANTHROPIC_AUTH_TOKEN` 是 CCR 的本地密钥，用来保护本地路由服务
- 它不是上游第三方的真实 API key

然后在 VS Code 里 Reload Window。

---

## 六 配置 CCR 的上游服务

我们这次遇到的最大坑是上游 base url 的拼接规则。
第三方服务往往同时提供管理后台页面和 API：根路径返回 HTML 页面，API 则走 `v1` 路径下的 chat completions。

先用 curl 验证一下服务形态：

```powershell
curl.exe -i "http://<YOUR_API_HOST>:3003/"
```

你会看到 `Content-Type` 是 `text/html`，说明这是后台页面。

再验证 API endpoint。注意必须用 POST：

```powershell
$body = @{
  model = "claude-sonnet-4-5-20250929-thinking"
  messages = @(@{ role = "user"; content = "say ok" })
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Method Post `
  -Uri "http://<YOUR_API_HOST>:3003/v1/chat/completions" `
  -Headers @{ Authorization = "Bearer sk-..." } `
  -ContentType "application/json" `
  -Body $body
```

这条能成功说明上游接口本身没问题。

关键点在 CCR 的配置。为了避免它拼错路径，我们直接把 `api_base_url` 配成完整 endpoint：

```text
api_base_url: http://<YOUR_API_HOST>:3003/v1/chat/completions
```

CCR 配好后重启：

```powershell
ccr stop
ccr start
```

---

## 七 推荐把 Claude Code 的 selectedModel 设为 default

Claude Code 对模型名会做本地校验。自定义模型名或带 thinking 后缀的模型名容易被它挡住。
最省事的办法是把 `selectedModel` 设成 `default`，让 CCR 决定最终走哪个模型。

PowerShell 写入 VS Code 用户设置：

```powershell
$settings = Join-Path $env:APPDATA "Code\User\settings.json"
$jsonText = Get-Content $settings -Raw
$obj = if ([string]::IsNullOrWhiteSpace($jsonText)) { @{} } else { $jsonText | ConvertFrom-Json }

$obj | Add-Member -Force NoteProperty "claudeCode.selectedModel" "default"
$obj | ConvertTo-Json -Depth 30 | Set-Content -Encoding UTF8 $settings
```

然后 Reload Window。

---

## 八 常见报错与标准处理

### 报错一 Not logged in 请运行 login

发生在 CLI。说明没登录，CLI 基本不可用。转到 VS Code 插件并关闭登录提示。

### 报错二 ConnectionRefused

说明本机 CCR 服务没开或端口不对。检查：

```powershell
ccr status
ccr start
```

必要时 VS Code Reload Window。

### 报错三 Unexpected token 小于号

说明客户端以为拿到了 JSON，实际拿到了 HTML。
通常是 CCR 拼错了上游路径，打到了后台页面。
解决办法是把 CCR 的 `api_base_url` 配成完整的 chat completions endpoint。

### 报错四 Invalid URL POST v1

这表示请求被发到了 `v1` 根路径，而不是 `v1/chat/completions`。
同样通过把 `api_base_url` 改成完整 endpoint 解决。

---

## 九 重要结论 工具调用协议可能不兼容

即使聊天已经能通，Claude Code 的真正价值在于读写文件、搜索、运行命令。
这部分依赖模型输出结构化的工具调用信息。

如果上游或转发链路对工具调用协议不兼容，会出现类似错误：

- 写文件失败，缺少 `file_path` 或 `content`
- 搜索失败，缺少 `pattern`
- 执行命令失败，缺少 `command`

日志里会看到类似信息，表示工具输入校验失败，字段变成 `undefined`。

这不是文件权限问题，也不是 VS Code 问题，根本原因是工具调用结构没有被正确生成或正确转换。

在这种情况下有两条路：

- 继续完善网关，使其支持并正确转换工具调用协议
- 先暂时禁用工具能力，只让它输出文本内容，文件由人手动创建和修改

---

## 十 快速排查 CCR 日志

当出现问题时，先确认请求是否到达 CCR，再看它最终发给上游的 URL 是什么。

查看 CCR 状态：

```powershell
ccr status
```

从日志里筛关键行：

```powershell
$log = (Get-ChildItem "$env:USERPROFILE\.claude-code-router\logs" | Sort-Object LastWriteTime -Descending | Select-Object -First 1).FullName
Select-String -Path $log -Pattern "messages","final request","provider_response_error","statusCode" | Select-Object -Last 80
```

---

## 十一 收尾

这条路线能跑，但不算省心。
建议把最终可用配置固化成两份文件备份起来：

- VS Code 用户 settings 里与 `claudeCode` 相关的字段
- CCR 的 config 文件

等后面要复现时，直接复制粘贴，别再重新踩一遍。
