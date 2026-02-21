---
title: "OpenClaw 接入 OpenRouter 免费模型"
subtitle: "FreeRide 一键配置，支持双模式切换与指定模型锁定"
date: 2026-02-21 17:00:00 +0800
tags: [OpenClaw, OpenRouter, FreeRide, 免费模型, 模型切换]
categories: [openclaw]
description: "通过 FreeRide 工具为 OpenClaw 配置 OpenRouter 免费模型，支持主力模型与免费模型之间的一键切换。"
---

## 现象

默认 OpenClaw 使用付费模型（如 `openai-next/claude-sonnet-4-6`）。通过 FreeRide 工具，可一键接入 OpenRouter 30+ 免费模型，并设置多级 fallback 避免因限速中断。

## 根因

OpenClaw 的模型配置集中于 `~/.openclaw/openclaw.json` 的 `agents.defaults.model` 字段。FreeRide 自动拉取 OpenRouter 免费模型列表、排名评分，并写入该配置文件。

## 修复步骤

### 前提条件

- OpenClaw 已安装（Node ≥ 22）
- Python 3.8+（WSL2 Ubuntu 默认 3.10 可用）
- OpenRouter 免费 Key：[openrouter.ai/keys](https://openrouter.ai/keys)（无需绑卡）

### 1. 安装 pip 与 FreeRide

```bash
# 若无 pip
curl -sS https://bootstrap.pypa.io/get-pip.py | python3 - --user

# 安装 FreeRide（skill 已放入工作区）
cd ~/.openclaw/workspace/skills/free-ride
python3 -m pip install -e .
```

成功标准：

```bash
freeride --help  # 输出命令列表，包含 mode、pin 等子命令
```

### 2. 写入 OpenRouter API Key

```bash
openclaw config set env.OPENROUTER_API_KEY "sk-or-v1-xxx"
```

验证：

```bash
freeride status  # 应显示 OpenRouter API Key: sk-or-v1-...
```

### 3. 存档主力模型（一次性）

```bash
freeride save-main
```

关键输出：

```text
Main profile saved!
  Primary : openai-next/claude-sonnet-4-6
Restore later with: freeride mode main
```

### 4. 锁定指定免费模型（可选）

若只信任特定模型，跳过自动排名，手动 pin：

```bash
freeride pin stepfun/step-3.5-flash qwen/qwen3-vl-235b-a22b-thinking
```

验证：

```bash
freeride pin  # 列出 Primary 与 Fallback
```

### 5. 切换模式

```bash
# 切至免费模型
freeride mode free
openclaw gateway restart

# 切回主力
freeride mode main
openclaw gateway restart
```

## 关键输出

```text
✓ Mode: FREE (pinned models)
  Primary : openrouter/stepfun/step-3.5-flash:free
  Fallbacks: openrouter/free, qwen/qwen3-vl-235b-a22b-thinking:free
Restart OpenClaw: openclaw gateway restart
```

## 验证方法

```bash
freeride status
```

通过标准：`Primary Model` 以 `openrouter/` 开头。

## 防复发

| 风险 | 措施 |
|------|------|
| 免费模型下线或 ID 变更 | 定期运行 `freeride refresh` 刷新缓存 |
| 误覆盖主力配置 | 首次运行 `freeride save-main`，切回用 `freeride mode main` |
| 目标模型不在 OpenRouter 免费列表 | 先用 `freeride list -n 30` 确认 ID 再 pin |

最短巡检命令：

```bash
freeride status
```

## 结论

通过 FreeRide 的 `pin` + `mode` 命令，可在 OpenDev 主力模型与 OpenRouter 免费模型之间一键切换，配置持久化，无需手动编辑 `openclaw.json`。
