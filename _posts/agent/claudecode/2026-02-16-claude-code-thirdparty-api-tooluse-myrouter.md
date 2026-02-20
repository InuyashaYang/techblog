---
title: "工具调用出错与 MyRouter 修复"
subtitle: "SSE 空参数与 JSON Parse error 根因定位"
date: 2026-02-16 14:00:00 +0800
tags: [Claude Code, VSCode, Anthropic, OpenAI, Router, SSE, ToolUse, Troubleshooting]
categories: [claudecode]
description: "工具调用空参数与 JSON Parse error 的根因定位，以及用 MyRouter 通过非流式兼容策略跑稳。"
---

## 1. 难点在哪

我们要做的事很明确：让 VS Code 里的 Claude Code 不走官方登录，改走我们自己的第三方 OpenAI 兼容上游。

真正的难点不在于能不能对话，而在于 Claude Code 的核心能力依赖两件事。

第一，Claude Code 说的是 Anthropic Messages API，而我们的上游说的是 OpenAI ChatCompletions，需要协议转换。

第二，Claude Code 要稳定使用工具，也就是读写文件、搜索、运行命令。工具调用依赖严格的结构化输出。只要结构化输出在转发或流式过程中被打碎，Claude Code 就会出现空参数，直接报错。

我们遇到的典型症状是：

- Bash 工具缺 command
- Glob 工具缺 pattern
- Write 工具缺 file_path 或 content
- 同时伴随 JSON Parse error

结论很直接：工具调用结构被解析坏了，字段变成 undefined，Claude Code 按规则拒绝执行。

---

## 2. 自研 MyRouter

我们决定自己做一个本地网关 MyRouter，对外伪装成 Anthropic Messages API，对内转发到第三方 OpenAI ChatCompletions。

对外暴露的接口很小，但够用：

- POST 127.0.0.1:8787 v1 messages
- GET 127.0.0.1:8787 v1 models
- 可选 healthz

鉴权用本地伪 key，长得像 Anthropic：

- sk-ant-local-xxxxxxxx

VS Code 侧只需要把环境变量指向 MyRouter：

```json
{
  "claudeCode.disableLoginPrompt": true,
  "claudeCode.selectedModel": "default",
  "claudeCode.environmentVariables": [
    { "name": "ANTHROPIC_BASE_URL", "value": "http://127.0.0.1:8787" },
    { "name": "ANTHROPIC_AUTH_TOKEN", "value": "sk-ant-local-xxxxxxxx" }
  ]
}
```

到这一步，聊天能通，属于阶段性胜利。

---

## 3. 新问题一：思考过程外泄

现象
回复里出现 think 内容。

原因
上游把 reasoning 原样返回，网关没有过滤。

处理
网关侧过滤 think 或 reasoning，只回传最终文本块。

---

## 4. 新问题二：网关崩溃 ReadableStream locked

报错
Invalid state ReadableStream is locked。

原因
Node 22 的 Web Streams 更严格，流式转发里对同一个流做了不允许的 cancel 或重复操作。

处理方向
先保证不崩，再谈优雅。

我们采取的策略是先不执着流式，把流式降级为非流式，先把功能跑稳。

---

## 5. 新问题三：工具调用还是会空参数

日志关键特征非常稳定：

- Stream started received first chunk
- JSON Parse error Unable to parse JSON string
- 然后 Bash command 缺失，Glob pattern 缺失

精确结论
流式返回里混入了 Claude Code 解析不了的 JSON 片段，导致工具调用结构被解析坏，字段变成 undefined。

---

## 6. 止血动作：关闭流式后又出现 400

我们一开始把流式关掉，结果 Claude Code 仍然带 stream true，网关直接 400：

- Streaming is disabled on this gateway Remove stream true

精确结论
Claude Code 默认会带 stream true。网关不能直接拒绝，必须做兼容降级。

正确做法
对外接受 stream true，但内部强制当作非流式处理，并返回非流式 Anthropic message JSON。

---

## 7. 最终可用形态

最终稳定形态总结：

1. 网关对外仍接受 stream true
2. 网关内部强制降级为非流式，返回标准 Anthropic message JSON
3. 工具调用参数必须保证是完整合法 JSON，不能发半截
4. v1 models 返回白名单，减少客户端模型校验问题
5. VS Code 侧 selectedModel 设 default，避免本地模型名校验拦路

---

## 8. 常用排查命令

找最新 Claude Code 插件日志并过滤关键错误：

```powershell
$log = Get-ChildItem "$env:APPDATA\Code\logs" -Recurse -File |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

$log.FullName
Select-String -Path $log.FullName -Pattern "JSON Parse error|tool input error|required parameter|Bash failed|Glob failed" -Context 2,6 |
  Select-Object -Last 80
```

判断网关端口是否在监听：

```powershell
netstat -ano | findstr ":8787"
```

直接打网关的 models 或 healthz：

```powershell
curl.exe -i "http://127.0.0.1:8787/v1/models"
curl.exe -i "http://127.0.0.1:8787/healthz"
```
