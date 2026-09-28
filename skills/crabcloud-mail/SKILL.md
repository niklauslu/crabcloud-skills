---
name: crabcloud-mail
description: Read, search and send the user's Crab Cloud platform email (username@crabcloud.cc) through the `crab` CLI — list folders, read messages, full-text search, send mail, archive/trash/delete. Use this skill when the user asks to check, read, find, draft or send email on Crab Cloud ("看看我的邮箱", "读一下这封邮件", "帮我回复", "发邮件给…", "crab mail"), or asks what their mailbox looks like — even if they never say "crab" or "Crab Cloud" explicitly.
---

# Crab Cloud 邮箱（crab mail）

用户的 Crab Cloud 账号自带平台邮箱 `username@crabcloud.cc`（用户名即地址本地部分）。
本 skill 覆盖邮箱域的 agent 通道；账号/令牌管理见 `crabcloud` skill。云盘见
`crabcloud-storage` skill（`crab storage …`），协作尚未上线——用户问到时如实说明，
**不要猜命令**。

## 第一步：确认可用

- `crab --version` 确认已安装；未登录（退出码 3）引导用户 `crab login`。
- 邮箱动作需要对应 scope：读 `mail.read`、搜索 `mail.search`、发信 `mail.send`、
  删除 `mail.delete`。退出码 4 = scope 不足：如实转述缺哪个，引导用户
  `crab login --scopes mail`（组名 = 四项整组）或单列所需 scope 按需最小化
  重新授权（你只发起，不代批）。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 看收件箱 / 各文件夹 | `crab mail list`（默认 inbox；`--folder sent\|drafts\|archive\|trash`） |
| 只看未读 | `crab mail list --unread` |
| 翻页 | `crab mail list --cursor <nextCursor>`（列表末尾会提示） |
| 读某封邮件 | `crab mail read <id>`（自动置已读；正文 + 操作记录） |
| 找邮件 | `crab mail search <关键词>`（主题/发件人/收件人/正文，不含废纸篓） |
| 发邮件 | `crab mail send --to a@x.cc --subject "..." --body "..."`（长正文用 `--body-file`） |
| 发邮件带附件 | `crab mail send … --attach <云盘id\|文件名>[,…]`（引用云盘已有对象，不复制，≤10 个；先 `crab storage ls` 核对对象，文件名需精确唯一命中） |
| 按名字/备注发邮件 | `--to`/`--cc` 可直接写联系人名（不含 `@`）：CLI 查联系人簿解析成地址，唯一命中即用；多候选会报错列出，改用完整地址重试。也可先 `crab contacts list <名字>`（`crabcloud` skill）查地址再发 |
| 回信（带线程锚点） | 先 `crab mail read <id>` 取 Message-ID，再 `crab mail send --to … --in-reply-to <message-id> …` |
| 整理邮箱 | `crab mail archive <id>` / `crab mail trash <id>` / `crab mail delete <id>` |

拿不准邮件 id 时先 `crab mail list` / `crab mail search` 核对再操作。

## 计费口径（必须如实转述）

- **平台内互发免费**：收件人也是 `@crabcloud.cc` 地址时不消耗积分。
- **外部出站 2 积分/封**：发送前向用户复述收件人与计费；退出码 7 = 积分不足，
  告知用户不要自动重试。
- 发送结果会报告 `消耗积分` 与失败数；部分失败时失败部分的积分自动退回
  （`发送:partial` 标记）。

## 机器可读输出

程序化消费**始终加 `--json`**：

- `mail list --json` → `{ items: [...], nextCursor }`，条目含
  `id/folder/fromAddr/to/subject/snippet/isUnread/hasAttachments/createdAt`；
- `mail read --json` → 完整正文 + `attachments`（id/filename/mimeType/sizeBytes）
  + `actors`（谁读过/谁代发，审计源）；
- `mail send --json` → `{ message, chargedCredits, failedExternal }`。

## 纪律

- **发邮件是外发动作**：发送前向用户复述收件人、主题与计费口径，确认后再执行；
  不确定收件人身份时优先查联系人簿（`crab contacts list <名字>`，见 `crabcloud`
  skill），其次搜索历史邮件核对，不要猜测地址。
- **`mail delete` 是彻底删除**，不可恢复；用户没明说「彻底删除」时用
  `mail trash`（可找回，废纸篓 30 天后自动清除）。
- 草稿流程：先 `mail list --folder drafts` 查看草稿；本阶段 CLI 不直接改草稿，
  确认发送请把草稿内容转述给用户后用 `mail send` 发出。
- 附件：`mail read` 输出附件清单（id/文件名/类型/体积）；附件内容归云盘管
  ——下载用 `crab storage download <id>`（见 `crabcloud-storage` skill，跨来
  源通用），发送引用云盘对象用 `mail send --attach`（对象须未被其他邮件绑
  定；若报错说明已被引用，改走正文链接）。邮件详情附件显示「已在云盘删除」
  表示对象已被软删，如实转述。
