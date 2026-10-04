---
name: crabcloud-hr
description: HR & admin assistant via the `crab hr` CLI — apply to create a company, manage members & permission flags, invite links with default permissions, the roster (people + event timelines + stats), dicts, tasks (with candidate form links), announcements, inbox and the desk digest. Use when the user asks anything about their company on Crab Cloud: 花名册、员工、入职、离职、合同、试用期、考勤、任务流、公告、收件箱、工作台、组织、成员、权限、邀请链接、字典、招聘收集 — even if they never say "crab".
---

# Crab Cloud 人事行政（crab hr）

管理公司 / 组织：成员与权限、邀请链接、花名册（人员 + 事件时间线 = 记录与证据链）、
字典、任务流（含候选人填单链接）、公告与收件箱、工作台聚合。独立产品
「人事行政 · Crab Cloud」，同一套 REST API 服务 Web 与 CLI。

跨域时参考：邮件 `crabcloud-mail`、云盘 `crabcloud-storage`、协作
`crabcloud-collab`、店铺 `crabcloud-shop`、平台账号 `crabcloud`。

## 第一步：确认可用

```bash
crab --version
```

`crab` 命令缺失 → `crab init` 只登录 + 装 skill、不含命令本体，需先 `npm i -g @crabcloud/cli@latest`。下文所有 `crab <命令>` 都可以用 `npx @crabcloud/cli <命令>` 等价执行。

hr 命令需要 **hr scope + 组织绑定**的令牌：`crab login --scopes hr --org org:<公司名>`。未绑组织或绑错组织会得到 `hr_org_not_bound` / `hr_org_mismatch`（退出码 4），按提示重新授权即可。

退出码：0 成功 · 2 用法错误 · 3 认证失效 · 4 scope/绑定不足 · 5 不存在 · 7 配额。

## 组织生命周期（申请-审核制）

建公司走**申请-审核**（与开店同款）：提交申请 → 平台审核 → 通过时扣
**1 积分**并自动建司（申请人成 owner）。每账号同时只有一条待审申请。

| 意图 | 命令 |
|---|---|
| 看我在哪些公司 | `crab hr orgs` |
| 申请创建公司 | `crab hr org apply <名称> [--note 说明]`（审核通过自动建司，无需轮询操作） |
| 查申请进度 | `crab hr org application`（pending / approved〔含驳回意见〕/ 从未申请） |
| 组织改名 | `crab hr org rename <新名称>`（org.manage；改名后 `org:<名称>` 标识随新名） |

## 成员、权限与邀请

权限模型：角色 owner（恒全权）/ admin / member + 四项权限 flags 逐项覆盖：
`org.manage`（成员邀请与组织设置）、`people.write`（花名册写）、
`tasks.write`（任务写）、`sensitive.read`（身份证/薪资可见）。

| 意图 | 命令 |
|---|---|
| 看成员与权限 | `crab hr members` |
| 拉人进来 | `crab hr member add <用户名> [--title 头衔] [--permissions org.manage=1,...]`（缺省只读） |
| 给人配权限 | `crab hr member set <memberId> --role admin` 或 `--permissions people.write=1,tasks.write=1` |
| 移除成员 | `crab hr member rm <memberId>` |
| 发邀请链接 | `crab hr invite`（站内 `/join/:token` 加入页：未注册者注册即入组，已登录一键接受） |
| 改加入后默认权限 | `crab hr invite --permissions people.write=1`（只改配置不换链接） |
| 作废旧链接 | `crab hr invite --reset`（旧链立即失效，权限配置保留） |

多组织账号用 `--org org:<名称>`（或组织 id）指定；仅一个组织时自动采用。

## 花名册（员工圈，无账号）

| 意图 | 命令 |
|---|---|
| 看花名册 | `crab hr people [--q 关键字] [--status active\|probation\|resigned] [--dept]` |
| 统计带 | `crab hr stats`（在职/试用/离职 + 合同 60 天、试用期 30 天风险计数） |
| 看一个人 | `crab hr person <id>` |
| 入职建档 | `crab hr person create --name "姓名" --hired-at YYYY-MM-DD [--dept] [--role] [--probation-end] [--contract-type] [--contract-end]` |
| 转正 | `crab hr person edit <id> --status active`（自动落痕） |
| 办离职 | `crab hr person offboard <id> [--date YYYY-MM-DD]`（档案封存不删，记录依法保留） |
| 删错档案 | `crab hr person rm <id>`（**仅零记录档案可删**；有时间线服务端 409） |
| 记一件事 | `crab hr event add <personId> --type <onboard\|contract\|change\|attendance\|payroll\|award\|asset\|offboard\|note> --note "..." [--at YYYY-MM-DD]` |
| 看时间线 | `crab hr events [--person <id>] [--type <类型>]` |
| 从表格导入 | `crab hr import <文件.csv|.tsv|.json>`（≤200 行，姓名/入职列必填，表头自动识别） |

## 字典（部门 / 岗位基础维护）

| 意图 | 命令 |
|---|---|
| 看字典 | `crab hr dicts` |
| 新增 | `crab hr dict add <dept\|position> <名称>`（同名幂等） |
| 删除 | `crab hr dict rm <entryId>`（被花名册在用服务端 409 拒删） |

## 任务流与申请单链接

| 意图 | 命令 |
|---|---|
| 看任务 | `crab hr tasks [--status alert\|confirm\|queued\|done] [--kind]` |
| 建任务 | `crab hr task create --kind attendance --title "9 月考勤月结" [--node] [--detail] [--due YYYY-MM-DD]` |
| 流转 | `crab hr task update <id> --status done` |
| 发候选人填单链接 | `crab hr task link <taskId>`（入职/转正/离职申请单专属；`--reset` 作废旧链） |
| 发入职收集链接 | `crab hr apply-link`（常驻码：候选人提交即自动生成入职申请单；`--reset` 全部作废） |

## 公告、收件箱与工作台

| 意图 | 命令 |
|---|---|
| 发公告 | `crab hr announce post <标题> --body "正文"`（org.manage；对象 = 组织全体成员） |
| 看公告 | `crab hr announce list [--all]`（默认只看生效；--all 含已撤回） |
| 撤回公告 | `crab hr announce withdraw <公告id>`（发布者本人或 org.manage；留档不删） |
| 查收件箱 | `crab hr inbox`（未读公告 + 待拍板任务 + 确定性提醒） |
| 标已读 | `crab hr inbox --seen` |
| **总览（推荐首入口）** | `crab hr desk`——花名册统计 + 替你盯着（合同 60 天 / 试用期 30 天 / 入职申请超时）+ 任务三段（等你拍板 / 助理在办 / 近期完成）+ 最近公告 + 最近动态，一次取全 |

## 机器可读输出（--json）

- `crab hr people --json` → `{ people: [...], total, org }`；人员字段 camelCase（`hiredAt`/`contractEnd` 为纪元毫秒；`idNumber`/`salaryCents` 对令牌恒为 `null`——见下方纪律）
- `crab hr stats --json` → `{ stats: { total, active, probation, resigned, probationDueSoon, probationOverdue, contractDueSoon, contractOverdue } }`
- `crab hr desk --json` → `{ stats: { total, active, … }, pending, running, done, reminders: [{ kind, text, days }], announcements: [{ title, createdByName, createdAt }], recentEvents }`
- `crab hr inbox --json` → `{ inbox: { lastSeenAt, announcements, unreadAnnouncementCount, pendingTasks, pendingTaskCount, reminders } }`
- `crab hr announce list --json` → `{ announcements: [...], total }`（status: active / withdrawn）
- `crab hr dicts --json` → `{ entries: [{ id, type, name }] }`
- `crab hr import --json` → `{ created, failed: [{ row, reason }] }`
- `crab hr invite --json` → `{ token, createdAt, permissions, link }`

## 纪律

- **敏感信息不主动读、读到不转述**：身份证号与薪资对 CLI/Agent 令牌在服务端就不可见（恒为 null，与「无权限」同形）。用户当面问到某人薪资时，引导其在 Web 端查看，不要猜测。
- **写操作先复述再执行**：建档 / 离职 / 删人 / 移除成员 / 重置邀请或收集链接 / 发公告前，先向用户复述将要发生的动作；离职是封存不是删除，但仍是敏感流转。
- 事件时间线是**劳动纠纷证据链**：只追加、不改写；`event add` 的 note 要写清事实（什么节点做了什么），不要写推测。有时间线的档案不可删（服务端强制）。
- 撤回的公告与驳回的申请都**留档不删**——不要向用户暗示记录会消失。
- 多组织账号务必确认 `--org` 后再写，写错组织的数据不属于本组织。
