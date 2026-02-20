---
title: "Codex 沙盒设置对 VS Code Remote-SSH 连接的影响"
date: 2026-02-20 12:00:00 +0800
tags: [Codex, SSH, VSCode, Windows, ACL]
description: "Codex sandbox 用户被写入 .ssh/config ACL，导致 OpenSSH 拒绝读取配置，Remote-SSH 连接失败。记录根因分析与 ACL 修复过程。"
---

## 现象

VS Code Remote-SSH 连接 `hpc` 失败。
关键日志显示本机 SSH 配置文件权限异常。

## 关键日志

```text
Bad permissions. Try removing permissions for user: ...\CodexSandboxUsers ... on file C:/Users/Inuyasha/.ssh/config.
Bad owner or permissions on C:\\Users\\Inuyasha/.ssh/config
Failed to parse remote port from server output
```

## 根因

根因是本机 `C:\Users\Inuyasha\.ssh\config` 的 ACL 不符合 OpenSSH 要求。
`config` 上出现了不应存在的主体或权限（如 `CodexSandboxUsers`）。
OpenSSH 拒绝读取该文件，Remote-SSH 随后失败。

## 与 sandbox 用户的关系

- 仅"创建用户"不一定触发问题
- 触发条件是该用户或组被写入 `.ssh/config` 的 ACL
- 本次是 sandbox 相关主体进入 ACL 后导致失败

## 修复命令

```powershell
$sshDir = "$env:USERPROFILE\.ssh"
$config = "$sshDir\config"

takeown /f "$sshDir" /r /d y
icacls "$sshDir" /setowner "$env:USERNAME" /t /c

icacls "$sshDir" /inheritance:r
icacls "$sshDir" /grant:r "$($env:USERNAME):(OI)(CI)F"
icacls "$sshDir" /remove:g "Users" "Authenticated Users" "Everyone" "CodexSandboxUsers" /t /c

if (Test-Path "$config") {
  icacls "$config" /inheritance:r
  icacls "$config" /grant:r "$($env:USERNAME):(R,W)"
  icacls "$config" /remove:g "Users" "Authenticated Users" "Everyone" "CodexSandboxUsers"
}
```

## 关键执行结果

```text
C:\Users\Inuyasha\.ssh\config DESKTOP-5LFERS5\Inuyasha:(R,W)
```

这表示 `config` 仅保留当前用户读写权限，满足要求。

## 验证命令

```powershell
icacls "$env:USERPROFILE\.ssh\config"
ssh -v hpc
```

验证标准：

- `icacls` 不再包含 `CodexSandboxUsers`、`Users`、`Everyone`
- `ssh -v hpc` 不再出现 `Bad owner or permissions`

## 结论

本次故障由本机 `.ssh/config` 权限问题引起，不是远端 `hpc` 故障。
修复 ACL 后，SSH 与 VS Code Remote-SSH 均恢复。

## 防复发建议

1. 将 Codex 与主账号 `.ssh` 隔离，使用独立 SSH 配置
2. 定期执行 `icacls "$env:USERPROFILE\.ssh\config"` 巡检 ACL
