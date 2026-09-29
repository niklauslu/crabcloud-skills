---
name: crabcloud-collab
description: Work in the user's Crab Cloud collaboration spaces (协作空间) through the `crab` CLI — list spaces and members, create and complete tasks, push immutable execution records (Markdown + attachments/links), maintain the mutable task summary, and invite linked contacts. Use this skill when the user asks to create/update/check collaboration tasks or spaces on Crab Cloud ("建个任务", "协作空间里有什么", "把这个进度推送到任务", "邀请他进空间", "crab collab"), or wants an agent to record work progress into a task — even if they never say "crab" or "Crab Cloud" explicitly.
---

# Crab Cloud 协作（crab collab）

协作是用户的任务协作应用：容器是协作空间（默认个人空间注册即有 + 独立付费
空间），原子单位是任务。任务 = 不可改的执行记录流（GitHub issue 式：什么节点
做了什么，人机同写），另有一个可随时覆盖的「摘要」字段表达当前态概览。本
skill 覆盖协作域的 agent 通道；账号/令牌管理见 `crabcloud` skill，邮箱见
`crabcloud-mail`，云盘见 `crabcloud-storage`。

## 第一步：确认可用

- 运行 `crab --version` 确认已安装。命令不存在时让用户运行
  `npx @crabcloud/cli init`（登录 + 安装本 skill 一步完成）；已装过但命令缺失
  时运行 `npx @crabcloud/cli@latest init` 升级。
- CLI 基于 npx 分发，**本机没有 `crab` 命令时，下文所有 `crab <命令>` 都可以
  `npx @crabcloud/cli <命令>` 等价执行**（例如 `npx @crabcloud/cli collab tasks`）。
- 未登录（退出码 3）引导用户 `crab login`。
- **协作命令需要空间绑定**：授权时必须带 `--space`（`default` = 默认空间，
  `space:<名称>` = 按名唯一命中）。含 collab scope 但没带 `--space` 会直接报
  用法错误（退出码 2）。正确示例：
  `crab login --scopes collab --space default`
- 动作需要对应 scope：读（spaces/tasks/records/members）`collab.read`，写
  （建任务/推送/完成/摘要/邀请）`collab.write`——一般整组授权 `--scopes collab`。
- 退出码 4 有两种协作专属情形，处置都是重新授权：
  - `collab_space_not_bound`：令牌没绑空间——按上面带 `--space` 重新 `crab login`；
  - `collab_space_mismatch`：令牌绑的是别的空间——换绑到目标空间（`--space`
    指对）重新授权。**不要**反复重试同一令牌。
- 推送记录免费（不消耗积分）；建独立空间（5 积分）与扩容才计费，见「纪律」。

## 空间怎么指（--space）

所有落到一个空间的命令都接受 `--space <arg>`，`<arg>` 三种形态：

1. `default` —— 用户的默认个人协作空间（注册即有）；
2. `space:<名称>` —— 按名称唯一命中（active 空间）；
3. 空间 id —— `crab collab spaces` 查到的 id。

**缺省 `--space` 时**：只在一个空间则自动采用；多个则 CLI 列出候选并要求显式
指定——不要瞎猜，先 `crab collab spaces` 看一眼再带上正确的 `--space`。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 有哪些协作空间 | `crab collab spaces`（任务/成员计数、限额、独立空间名额） |
| 建独立协作空间 | `crab collab space create <名称> [--allocation-mb 500]`（**消耗 5 积分**，见纪律） |
| 空间里有什么任务 | `crab collab tasks [--space <arg>]`（倒序，含摘要/记录数；`--status open\|done`、`--q 关键字` 搜标题与记录正文） |
| 任务详情 | `crab collab task <id>`（状态/摘要/记录数） |
| 建任务 | `crab collab task create --space <arg> --title "..." --body "首条记录(Markdown)"`（也可 `--body-file <路径>`；附件 `--attach <fileId,...>`、链接 `--link <url,...>`） |
| 读任务的来龙去脉 | `crab collab task records <id>`（倒序记录流，Markdown 源文；`--cursor` 翻更早） |
| 推送工作记录 | `crab post <taskId> --body "做了什么(Markdown)"`（糖命令，等价 records 写入；免费） |
| 任务做完了 | `crab collab task done <id>`（入流一条状态变更）；反悔 `crab collab task reopen <id>` |
| 维护任务摘要 | `crab collab task summary <id> --text "当前态概览"`（可随时覆盖；`--clear` 清空；不入记录流） |
| 谁在空间里 | `crab collab members [--space <arg>]` |
| 邀人进空间 | `crab collab invite <username> [--space <arg>]`（**只能邀联系人中已关联的平台用户**，见纪律） |
| 看协作活动 | `crab collab activity`（Agent 推送/读取、任务创建、摘要更新事件） |

任务 id 从 `crab collab tasks` 拿；`crab post` 的 `<taskId>` 同源。

## 机器可读输出

程序化消费**始终加 `--json`**：

- `collab spaces --json` → `{ spaces: [...], independentLimit, independentUsed }`，
  条目含 `id / kind / name / status / taskCount / memberCount / limits`；
- `collab tasks --json` → `{ tasks: [...], nextCursor }`，条目含
  `id / title / status / summary / recordCount / latestRecordAt`（游标分页）；
- `collab task records --json` → `{ records: [...], nextCursor }`，条目含
  `id / kind / body / authorName / via / refs / createdAt`；
- `post --json` → `{ recordId, via?: { token, eventId } }`（via.eventId 即该条
  记录 id，可确定性回查）；
- `collab task <id> --json` / `collab members --json` / `collab activity --json`
  同构。

## 纪律

- **记录流不可改**：推上去就是历史，没有编辑/删除——推送前把内容写对；写错了
  补一条更正记录，不要试图「撤回」。
- **摘要是可变当前态**：表达「现在进展到哪」，不是流水账；每次推送记录后若
  摘要已过时，顺手 `crab collab task summary` 刷新，让不看全程的人一眼看懂。
- **建独立空间是付费动作（5 积分）**：先向用户复述名称与自分配存储（默认
  500MB，占订阅总池），确认后再执行；`collab space create` 不要当作顺手操作。
- **邀请有前置**：目标必须是联系人簿里已关联的平台用户（`crab contacts add`
  站内地址自动关联）；报「只能邀请联系人中已关联的平台用户」时，先引导建联
  系人，不要换名字硬试。
- **附件引用不复制**：`--attach` 的 fileId 是云盘/空间已有文件的引用（≤10 个），
  只能引用空间成员的云盘文件或本空间直传文件；跨空间文件会被 403 拒绝。
- 单任务记录流上限 500 条（状态变更也占位）：任务粒度别太粗，做完就 `done`，
  新工作开新任务。
- 读记录流对 Agent 是一等审计事件（thread.read 入活动流）——正常读即可，但
  不要在循环里无意义地反复拉全量记录。
