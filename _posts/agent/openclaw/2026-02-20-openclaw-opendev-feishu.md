---
title: "OpenClaw 配置与飞书接入"
subtitle: "最少步骤跑通，整理常见死锁与权限问题修复方法"
date: 2026-02-20 16:00:00 +0800
tags: [OpenClaw, WSL, 飞书, Gateway, Node.js, Troubleshooting]
categories: [openclaw]
description: "WSL2 安装 OpenClaw 并接入飞书，含 Gateway pairing 死锁、apiKey 环境变量、baseUrl 路径等常见问题的修复方法。"
---

推荐查看此教程，覆盖 OpenClaw 安装的全部流程，本博客作为补充：[飞书开发者文档](https://open-dev.feishu.cn/docx/UwA1dI4e4o5Un6xfsAOcW8iNnwg?from=from_copylink)

使用的模型网关服务：[api.openai-next.com](https://api.openai-next.com/)

---

## 现象

在 Windows（WSL2 Ubuntu22.04）安装 OpenClaw 后：

- Gateway 偶发 1006/1008、`pairing required`
- `openclaw agent` 走 gateway 失败但 `--local` 能跑
- 飞书插件在 `configure` 中死循环安装
- 飞书能建文档但写入内容返回 400

## 关键日志（定位根因）

```text
gateway closed (1008): pairing required
cause=pairing-required

HTTP 404 invalid_request_error: Invalid URL (POST /v1/v1/messages)

HTTP 401 shell_api_error: [$OPENAI_NEXT_KEY]无效的令牌

[plugins] feishu failed to load ... Cannot find module '@sinclair/typebox'

feishu_doc: No Feishu accounts configured, skipping doc tools

... missing docx:document.block:convert ...
```

## 根因

- Gateway 鉴权状态（devices/pairing/auth）进入冲突/死锁 → CLI 探测或 status 触发 scope 升降 → 1008 pairing required。
- OpenClaw 不展开 `"$ENV_VAR"` → `apiKey` 变成字面量 `$OPENAI_NEXT_KEY` → 401。
- baseUrl 误带 `/v1` 且适配器再拼 `/v1/messages` → `/v1/v1/messages` → 404。
- `configure` 对 Feishu 依赖 npm 插件安装路径且存在循环/失败 → 无法交互配置。
- 飞书 docx 写入需要额外 scope（`docx:document.block:convert`）未授权 → 400。

---

## 修复步骤

### 1) WSL 内独立 Node.js（避免调用 Windows npm）

```bash
apt update
apt install -y ca-certificates curl gnupg
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
which node && which npm
node -v && npm -v
```

**成功标准**

- `which npm` 为 `/usr/bin/npm`（不是 `/mnt/c/...`）

---

### 2) 正确配置模型（关键三点）

> 模型网关可用：[api.openai-next.com](https://api.openai-next.com/)
> 若你用自建反代，则以你的 baseUrl 为准。

**a. baseUrl 不要重复 /v1（Anthropic messages 适配器会拼 /v1/messages）**

```bash
jq '.models.providers["openai-next"].baseUrl="http://YOUR_HOST:PORT"' \
  ~/.openclaw/openclaw.json > ~/.openclaw/openclaw.json.tmp \
&& mv ~/.openclaw/openclaw.json.tmp ~/.openclaw/openclaw.json
```

**b. 不用 `"$OPENAI_NEXT_KEY"` 这类占位符（该版本不展开环境变量）**

```bash
set -a; source ~/.openclaw/.env; set +a

jq --arg k "$OPENAI_NEXT_KEY" '.models.providers["openai-next"].apiKey=$k' \
  ~/.openclaw/openclaw.json > ~/.openclaw/openclaw.json.tmp \
&& mv ~/.openclaw/openclaw.json.tmp ~/.openclaw/openclaw.json
```

**c. 最小验证（不走 gateway）**

```bash
openclaw agent --local --session-id sanity --message "回复OK并给出模型名" --json
```

**成功标准**

- JSON 输出包含模型名 `claude-sonnet-4-6`

---

### 3) Gateway pairing 死锁：重置鉴权状态文件（仅在卡死时用）

当出现：

```text
gateway closed (1008): pairing required
```

执行：

```bash
openclaw gateway stop || true
pkill -f openclaw-gateway || true

rm -f ~/.openclaw/devices/pending.json
rm -f ~/.openclaw/devices/paired.json
rm -f ~/.openclaw/identity/device.json
rm -f ~/.openclaw/identity/device-auth.json
rm -f ~/.openclaw/agents/main/agent/auth.json
```

重启并验证：

```bash
set -a; source ~/.openclaw/.env; set +a
openclaw gateway start
sleep 1
openclaw gateway call health
```

**成功标准**

- `openclaw gateway call health` 返回 `"ok": true`

---

### 4) WSL 启用 systemd（稳定管理 gateway）

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true

[interop]
appendWindowsPath=false
EOF
```

PowerShell：

```powershell
wsl --shutdown
wsl -d Ubuntu22.04 -u inuyasha
```

验证：

```bash
ps -p 1 -o comm=
systemctl --user status
```

安装并启动服务：

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway status
```

**成功标准**

- 端口监听：

```bash
ss -lntp | grep 18789
```

---

### 5) 飞书：绕过 configure 的插件安装循环，直接写 channel 配置

飞书侧参考：[飞书开发者文档](https://open-dev.feishu.cn/docx/UwA1dI4e4o5Un6xfsAOcW8iNnwg?from=from_copylink)

确认插件干净：

```bash
openclaw plugins list | rg -i feishu
```

写入账号凭据（不回显 secret）：

```bash
cp -a ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%F_%H%M%S)

jq '
  .channels.feishu.accounts.default.appId = "cli_xxx"
  | .channels.feishu.accounts.default.appSecret = "YOUR_SECRET"
  | .channels.feishu.enabled = true
' ~/.openclaw/openclaw.json > ~/.openclaw/openclaw.json.tmp \
&& mv ~/.openclaw/openclaw.json.tmp ~/.openclaw/openclaw.json

jq '{enabled:.channels.feishu.enabled, accountKeys:(.channels.feishu.accounts.default|keys)}' ~/.openclaw/openclaw.json
```

启动并验证：

```bash
openclaw gateway start
openclaw channels list
openclaw channels logs --channel feishu
```

**成功标准**

- `channels list` 显示 `Feishu default: configured, enabled`
- 日志出现：

```text
starting feishu[default] (mode: websocket)
WebSocket client started
```

---

### 6) 飞书首次对话会要求配对（正常安全策略）

飞书里收到：

```text
Pairing code: XXXXXXXX
openclaw pairing approve feishu XXXXXXXX
```

执行：

```bash
openclaw pairing approve feishu XXXXXXXX
```

**成功标准**

- 再发消息机器人能直接回复，不再给 pairing code

---

### 7) 飞书文档写入 400：补 scope

现象：

```text
missing docx:document.block:convert
```

修复：

- 飞书开放平台给应用补 `docx:document.block:convert`
- 发布版本后重试写入

**成功标准**

- 文档创建 + 正文写入都成功

---

## 验证方法（最小闭环）

```bash
openclaw gateway call health
openclaw agent --session-id gwtest --message "回复OK并给模型名" --json
openclaw channels list
openclaw channels logs --channel feishu
```

通过标准：

- health ok
- agent 返回模型名
- feishu websocket started
- 飞书消息可回、文档可写

---

## 防复发（最有效 2 条）

1) 只用一个 Linux 用户运行 OpenClaw（例如 `inuyasha`），不要 root 混跑

```bash
ps -ef | rg openclaw-gateway
```

2) 遇到 `pairing required` 不要反复 probe/status，直接重置鉴权状态文件一次（见步骤 3）

---

## 结论

WSL2 上跑 OpenClaw 并接入 Claude 4.6 + 飞书可稳定实现；核心避坑是：Node 环境隔离、apiKey 不用 `$VAR` 占位、baseUrl 不重复 `/v1`、pairing 死锁用鉴权状态文件重置、飞书写文档补齐 `docx:document.block:convert`。
