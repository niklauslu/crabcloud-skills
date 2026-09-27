---
name: crabcloud
description: Manage the user's Crab Cloud personal cloud account and agent tokens through the `crab` CLI — check identity/subscription/credits, list and revoke authorized agent tokens, and re-authorize with specific scopes. Use this skill whenever the user asks about their Crab Cloud account ("我的账号信息", "who am I on crabcloud"), wants to see or clean up authorized agents/tokens ("看看我授权了哪些 agent", "revoke that old token", "撤销那个旧令牌"), or asks how to connect their coding agent to Crab Cloud — even if they never say "crab" or "Crab Cloud" explicitly. Mail, files and projects capabilities ship with their own skills (crabcloud-mail etc.) in later phases.
---

# Crab Cloud（平台 / 账号域）

Crab Cloud 是用户的个人云底座：账号、订阅与积分是平台层，邮箱 / 云文件 / 协作项目
是独立应用。`crab` CLI 是 Agent 的稳定接口：凭证在用户本机
（`~/.config/crabcloud/credentials.json`，0600），服务端只存令牌哈希、全程审计。
本 skill 目前覆盖**平台与账号域**；邮箱（`crabcloud-mail`）、云文件、协作项目
能力随各阶段上线——用户问到这些时如实说明尚未开放，**不要猜命令**。

## 第一步：确认 CLI 可用

- 运行 `crab --version` 确认已安装。命令不存在时让用户运行
  `npx @crabcloud/cli init`（登录 + 安装本 skill 一步完成）；已装过但命令缺失
  时运行 `npx @crabcloud/cli@latest init` 升级。
- `crab whoami` 报「未登录」或退出码 3 → 引导用户 `crab login`。登录走浏览器
  设备授权（用户逐条确认 scope）——**你只发起，不代批**。
- 永远不要让用户把 token 粘贴进对话；凭证文件与你无关，也不要读取它。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 看账号身份 / 订阅 / 积分 | `crab whoami` |
| 程序化读取身份 | `crab whoami --json` |
| 列出已授权的 Agent 令牌 | `crab tokens list`（脚本消费加 `--json`） |
| 撤销某个令牌（高危，先确认） | `crab tokens revoke <id>` |
| 重新授权 / 调整 scope | `crab login --scopes mail.read`（浏览器确认，你只发起） |
| 登出并吊销当前令牌 | `crab logout` |
| 管理登录设备 / 会话、修改密码 | 网页个人中心（crabcloud.cc/account）——CLI 有意不开放 |

拿不准令牌 id 时先 `crab tokens list` 核对；`revoke` 只接受列表里的 id。

## 机器可读输出

程序化消费时**始终加 `--json`**（`whoami --json` / `tokens list --json`）：

- `whoami --json` → `{ actor, scopes, account: { username, displayName, … }, personalSpace, subscription, credits, apps }`；
- `tokens list --json` → `[{ id, name, scopes, createdAt, lastUsedAt, expiresAt, revoked }]`：
  后续 `revoke` 用 `id`，向用户转述时用 `name` + `scopes`。

## 退出码与处置

| 码 | 含义 | 你该做什么 |
|---|---|---|
| 0 | 成功 | 继续 |
| 2 | 用法错误 | 检查命令参数，不要原样重试 |
| 3 | 认证失效 | 引导用户运行 `crab login` 重新授权，不要自行重试 |
| 4 | scope 不足 | 如实转述缺哪个 scope，引导按需最小化重新授权 |
| 5 | 资源不存在 | 先 `crab tokens list` 核对 id |
| 7 | 配额/积分不足 | 告知用户，不要自动重试 |

## 纪律

- **撤销令牌是高危操作**：先向用户复述目标令牌的名称与 scope，得到确认后再
  执行 `crab tokens revoke`；撤销立即生效，不可恢复（重新授权即可再建）。
- **scope 最小化**：建议按需授权（只读场景 `crab login --scopes account.read`），
  不要怂恿用户一次性给全量 scope。
- 如实转述能力边界：邮箱 / 云文件 / 协作项目尚未上线，相关请求引导用户关注
  后续版本，不要编造命令或输出。
- 账号安全操作（修改密码、登录设备/会话的查看与吊销）**有意不开放给 agent
  通道**：改密对令牌持有者是账号接管面，会话属于"人的浏览器会话"。用户让
  agent 做这些时，如实说明并引导到网页个人中心（crabcloud.cc/account）操作，
  不要猜命令、也不要试图直接调 API。
