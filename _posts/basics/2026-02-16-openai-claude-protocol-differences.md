---
title: "OpenAI 与 Claude 协议差异速览（Messages / Chat Completions / Responses）"
date: 2026-02-16 15:00:00 +0800
tags: [OpenAI, Claude, API, Protocol, Messages, Responses, Tools]
description: "用请求体对比 OpenAI 与 Claude 的协议结构、端点与工具字段映射。"
---

OpenAI 与 Claude 的能力相近，但协议结构不同。结合请求体看差异最直观。

OpenAI Chat Completions（`/v1/chat/completions`）以 `messages` 为中心：

```json
{
  "model": "gpt-5.3-codex",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "写一个 hello world" }
  ],
  "temperature": 0.2,
  "stream": false
}
```

Claude Messages（`/v1/messages`）把 `system` 放在顶层，并要求 `messages`：

```json
{
  "model": "claude-sonnet-4-5-20250929",
  "system": "You are a helpful assistant.",
  "messages": [
    { "role": "user", "content": "写一个 hello world" }
  ],
  "max_tokens": 512,
  "temperature": 0.2,
  "stream": false
}
```

主要差异：

- OpenAI 常把系统指令放在 `messages`（或 Responses 的 `instructions`）
- Claude 明确使用顶层 `system`
- Claude 侧 `max_tokens` 通常更显式

OpenAI Responses（`/v1/responses`）是另一种风格，以 `input` 为中心：

```json
{
  "model": "gpt-5.3-codex",
  "input": "写一个 hello world",
  "temperature": 0.2,
  "stream": false
}
```

这解释了为何 Codex 会请求 `/v1/responses`：它不一定走 `/v1/chat/completions`。
如果网关只实现了 `/v1/messages`，就会出现 404。

工具定义也很像，但字段名不同。

OpenAI tools：

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "获取天气",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string" }
          },
          "required": ["city"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

Claude tools：

```json
{
  "tools": [
    {
      "name": "get_weather",
      "description": "获取天气",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string" }
        },
        "required": ["city"]
      }
    }
  ]
}
```

这里的核心映射是：

- OpenAI `function.parameters` ↔ Claude `input_schema`

结论：

- 能力层基本一致（对话、工具、流式）
- 协议层有差异（端点、字段、事件、工具回传）
- 网关要同时兼容时，至少要补齐 `/v1/responses`（给 Codex）与 `/v1/messages`（给 Claude）两条入口。
