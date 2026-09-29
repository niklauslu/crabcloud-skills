---
name: crabcloud
description: Manage the user's Crab Cloud personal cloud account and agent tokens through the `crab` CLI — check identity/subscription/credits, list and revoke authorized agent tokens, and re-authorize with specific scopes or scope groups. Use this skill whenever the user asks about their Crab Cloud account ("我的账号信息", "who am I on crabcloud"), wants to see or clean up authorized agents/tokens ("看看我授权了哪些 agent", "revoke that old token", "撤销那个旧令牌"), manages their contacts ("我的联系人", "把 zhangsan 加到联系人", "我的邀请码"), or asks how to connect their coding agent to Crab Cloud — even if they never say "crab" or "Crab Cloud" explicitly. Contacts are managed with `crab contacts …` in this skill; email lives in the crabcloud-mail skill (crab mail …) and the Drive in crabcloud-storage (crab storage …); collab ships in a later phase.
---

# Crab Cloud（平台 / 账号域）

Crab Cloud 是用户的个人云底座：账号、订阅与积分是平台层，邮箱 / 云盘 / 协作 /
联系人是独立应用。`crab` CLI 是 Agent 的稳定接口：凭证在用户本机
（`~/.config/crabcloud/credentials.json`，0600），服务端只存令牌哈希、全程审计。
本 skill 覆盖**平台、账号域与联系人**（`crab contacts …`）；邮箱见
`crabcloud-mail` skill（`crab mail …`），
云盘见 `crabcloud-storage` skill（`crab storage …`）；协作随后续阶段
提供——用户问到时如实说明，**不要猜命令**。

## 第一步：确认 CLI 可用

- 运行 `crab --version` 确认已安装。命令不存在时让用户运行
  `npx @crabcloud/cli init`（登录 + 安装本 skill 一步完成）；已装过但命令缺失
  时运行 `npx @crabcloud/cli@latest init` 升级。
- `crab whoami` 报「未登录」或退出码 3 → 引导用户 `crab login`。登录走浏览器
  设备授权——授权页摆出全目录供用户勾选，**实际授予集由人定**，可能多于或少于
  `--scopes` 请求——**你只发起，不代批**；登录后先 `crab whoami` 核对实际
  scopes 再干活，缺什么就引导用户带组名/scope 重新授权。
- 永远不要让用户把 token 粘贴进对话；凭证文件与你无关，也不要读取它。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 看账号身份 / 订阅 / 积分 | `crab whoami` |
| 程序化读取身份 | `crab whoami --json` |
| 积分余额（月送/充值/合计） | `crab credits`（需 `credits.read`，脚本消费加 `--json`） |
| 积分流水（花了多少、花在哪） | `crab credits history [--limit N] [--offset N]`（变动 ±：消耗为负；`--json` 含 total） |
| 查动作计价（如发一封外域邮件扣多少） | `crab credits pricing`（公开端点，无需登录） |
| 列出已授权的 Agent 令牌 | `crab tokens list`（脚本消费加 `--json`） |
| 撤销某个令牌（高危，先确认） | `crab tokens revoke <id>` |
| 重新授权 / 调整 scope | `crab login --scopes mail`（组名整组授权；也可单 scope 如 `mail.read`。浏览器确认，你只发起） |
| 登出并吊销当前令牌 | `crab logout` |
| 联系人：列出 / 搜索 | `crab contacts list [query]`（`--offset`/`--limit` 分页，输出含「共 N 条」） |
| 联系人：添加 / 删除 | `crab contacts add <address> [--name 名] [--note 备注]` / `crab contacts rm <id|address>` |
| 我的邀请码、邀请链接与受邀名单 | `crab contacts invite [--reset]` / `crab contacts invitees` |
| 管理登录设备 / 会话、修改密码 | 网页个人中心（crabcloud.cc/account）——CLI 有意不开放 |

积分命令需 `credits.read` scope（`account` 组含它）：退出码 4 报缺该 scope 时，
引导 `crab login --scopes account` 重新授权。积分额度一律以**积分数**表述
（100 积分 = $1.00，仅价格类口径用美元）；充值/下单是 Web 会话能力，CLI/Agent
令牌不动账——用户要充值时引导网页账单页（crabcloud.cc/account/billing）。
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
  不要怂恿用户一次性给全量 scope。`--scopes` 支持组名（`mail` = 读写搜删四项
  整组）与单个 scope 混用；组目录见 `crab help`，可用组：account / mail /
  storage（云盘，含素材与附件）/ collab / contacts。
- 如实转述能力边界：协作尚未上线——相关请求如实说明，引导用户
  用网页（crabcloud.cc），不要编造命令或输出；联系人已上线，用 `crab contacts`
  （list / add / rm / invite / invitees；scope contacts.read/write，程序消费加
  `--json`）处理；邮箱请求转交 `crabcloud-mail`
  skill，云盘请求转交 `crabcloud-storage` skill。
- 账号安全操作（修改密码、登录设备/会话的查看与吊销）**有意不开放给 agent
  通道**：改密对令牌持有者是账号接管面，会话属于"人的浏览器会话"。用户让
  agent 做这些时，如实说明并引导到网页个人中心（crabcloud.cc/account）操作，
  不要猜命令、也不要试图直接调 API。
