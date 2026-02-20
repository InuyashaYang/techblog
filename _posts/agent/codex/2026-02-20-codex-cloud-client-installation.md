---
title: "云端从零安装 Codex CLI "
subtitle: "含自定义网关和权限、持久化管理"
date: 2026-02-20 12:00:00 +0800
tags: [Codex, SSH, VSCode, Windows, ACL]
categories: [codex]
description: "云端机器默认无法直连外网；需要安装 Codex CLI"
---

## 目标
云端机器默认无法直连外网；需要安装 Codex CLI、接入网关 `http://152.53.52.170:3003/v1`，并保证 SSH 重连后仍可用，且不污染系统与其他用户环境。

## 遇到的问题
1) 外网访问需走代理；2) 系统 Node 版本过旧；3) Codex CLI 在该版本下从 `config.toml` 读取 `base_url` 不稳定，但读取 `OPENAI_BASE_URL` 稳定；4) Key 需要用 `codex login` 持久化，不建议写入环境变量。

---

## 0. 前置：确认在用户级环境操作（不需要 sudo）
```bash
whoami; id
which python; python -V
```
**检查点：**
- 当前用户为个人账号
- Python 来自个人环境（conda/venv 任一均可）

---

## 1. 配置集群代理（临时，仅当前会话）
> 若可直连 GitHub，可跳过。

```bash
export http_proxy=http://192.168.102.101:7890
export https_proxy=http://192.168.102.101:7890
export all_proxy=socks://192.168.102.101:7890
curl -I https://raw.githubusercontent.com | head
```

**检查点：**
- 出现 `HTTP/2 301` 或 `200`

---

## 2. 安装 nvm（用户级）
```bash
curl -fL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh -o /tmp/install_nvm.sh
bash /tmp/install_nvm.sh
source ~/.nvm/nvm.sh && nvm --version
```

**检查点：**
- `nvm --version` 输出 `0.40.3`

### 2.1 若遇到 git SSL backend 报错（常见）
报错关键行：
```text
fatal: Unsupported SSL backend 'openssl'. Supported SSL backends: gnutls
```

修复：
```bash
git config --global --unset-all http.sslbackend || true
git config --global --get-all http.sslbackend; echo EXIT:$?
```

**检查点：**
- 最后一行 `EXIT:1`（表示已清掉）

---

## 3. 安装 Node 20（用户级，不动系统）
```bash
source ~/.nvm/nvm.sh && nvm install 20 && nvm use 20
node -v && npm -v
```

**检查点：**
- `node -v` 为 `v20.x`
- `npm -v` 有版本号

---

## 4. 安装 Codex CLI
```bash
source ~/.nvm/nvm.sh && npm install -g @openai/codex
codex --version
```

**检查点：**
- 输出 `codex-cli 0.xxx.x`
- 无 `EACCES` 权限错误

---

## 5. 用官方方式持久化 API Key（不写环境变量）
> 凭据写入 `~/.codex/auth.json`，仅对当前用户生效。

```bash
read -rsp "OpenAI API Key: " OPENAI_API_KEY; echo
export OPENAI_API_KEY
printenv OPENAI_API_KEY | codex login --with-api-key
codex login status
```

**检查点：**
- `Successfully logged in`
- `codex login status` 显示已登录（不要泄露 key）

---

## 6. 权限收紧（防止同机用户窥探目录结构）
```bash
chmod 700 ~/.codex ~/.codex/sessions ~/.codex/shell_snapshots ~/.codex/skills ~/.codex/tmp 2>/dev/null || true
chmod 600 ~/.codex/auth.json 2>/dev/null || true
ls -ld ~/.codex; ls -l ~/.codex/auth.json
```

**检查点：**
- `~/.codex` 为 `drwx------`
- `auth.json` 为 `-rw-------`

---

## 7. 写入极简 config.toml（只管默认模型；不在此处管 URL）
> 该版本对 toml 的 `base_url` 读取可能不稳定，URL 改为运行时注入更稳。

```bash
umask 077 && cat > ~/.codex/config.toml <<'EOF'
model = "gpt-5.2-codex"
model_provider = "openai"

[profiles."gpt-5.3-codex"]
model = "gpt-5.3-codex"
model_provider = "openai"
EOF
ls -l ~/.codex/config.toml; sed -n '1,80p' ~/.codex/config.toml
```

**检查点：**
- `config.toml` 为 `-rw-------`
- 默认 `gpt-5.2-codex`，并记录 `gpt-5.3-codex` profile

---

## 8. 关键持久化：仅在调用 Codex 时注入网关 URL（推荐）
目标：环境变量不常驻（避免污染其它命令），但每次运行 codex 都自动带上网关。

### 8.1 清理之前可能写入的全局 URL（可选）
```bash
sed -i '/^# Codex gateway (user scope only)$/d;/^export OPENAI_BASE_URL="http:\/\/152\.53\.52\.170:3003\/v1"$/d' ~/.bashrc
```

### 8.2 写入“codex 包装函数”
```bash
cat >> ~/.bashrc <<'EOF'

# Codex wrapper: inject gateway URL only for codex invocation
codex() {
  OPENAI_BASE_URL="http://152.53.52.170:3003/v1" command codex "$@"
}
EOF
source ~/.bashrc
```

**检查点：**
```bash
type codex | head
```
- 显示 `codex is a function`

---

## 9. 验收：unset 全部环境变量后仍可用
```bash
unset OPENAI_API_KEY OPENAI_BASE_URL OPENAI_MODEL http_proxy https_proxy all_proxy
echo "OPENAI_API_KEY=${OPENAI_API_KEY:-<empty>} OPENAI_BASE_URL=${OPENAI_BASE_URL:-<empty>} http_proxy=${http_proxy:-<empty>}"
codex login status
codex exec --skip-git-repo-check "请仅回复：still-ok"
```

**通过标准：**
- 变量都显示 `<empty>`
- `codex login status` 仍已登录（走 `auth.json`）
- `codex exec ...` 返回 `still-ok`（函数注入 URL 生效）

---

## 10. 常见坑与处理

### 10.1 `-p` profile 不生效
该版本中 `-p` 可能不稳定，切模型建议直接用：
```bash
codex exec -m gpt-5.3-codex --skip-git-repo-check "请仅回复：ok"
```

### 10.2 `Not inside a trusted directory`
非 git 仓库目录运行会被拦：
```bash
codex exec --skip-git-repo-check "..."
```
或进入 git 仓库再运行。

### 10.3 目录清理警告
```text
WARNING: failed to clean up stale arg0 temp dirs: Directory not empty (os error 39)
```
通常可忽略；若需处理再清理 `~/.codex/tmp`（谨慎）。

---

## 防复发（最短巡检）
每次 SSH 重连后：
```bash
node -v; codex --version; codex login status; sed -n '1,40p' ~/.codex/config.toml; type codex | head -n 2
```

**通过标准：**
- Node 为 20
- 登录态有效
- `codex` 为 function（网关注入仍在）

---

## 结论
采用用户级 nvm/Node20 安装 Codex CLI；用 `codex login` 持久化 key；用 bash 函数在调用时注入 `OPENAI_BASE_URL`；配合收紧 `~/.codex` 权限，即可实现“可重连、可复现、权限安全、且不影响其他用户”。