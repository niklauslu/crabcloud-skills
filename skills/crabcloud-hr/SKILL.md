---
name: crabcloud-hr
description: HR & admin assistant via the `crab hr` CLI — manage companies/orgs, members & permission flags, the roster (people + event timelines), records/audit trail, tasks and the desk digest. Use when the user asks anything about their company on Crab Cloud: 花名册、员工、入职、离职、合同、试用期、考勤登记、任务流、工作台、组织、成员、权限、邀请链接 — even if they never say "crab".
---

# Crab Cloud 人事行政（crab hr）

管理公司 / 组织：成员与权限、花名册（人员 + 事件时间线 = 记录与证据链）、任务流与工作台聚合。独立产品「人事行政 · Crab Cloud」，同一套 REST API 服务 Web 与 CLI。

跨域时参考：邮件 `crabcloud-mail`、云盘 `crabcloud-storage`、协作 `crabcloud-collab`、平台账号 `crabcloud`。

## 第一步：确认可用

```bash
crab --version
```

`crab` 命令缺失 → `crab init` 只登录 + 装 skill、不含命令本体，需先 `npm i -g @crabcloud/cli@latest`。下文所有 `crab <命令>` 都可以用 `npx @crabcloud/cli <命令>` 等价执行。

hr 命令需要 **hr scope + 组织绑定**的令牌：`crab login --scopes hr --org org:<公司名>`。未绑组织或绑错组织会得到 `hr_org_not_bound` / `hr_org_mismatch`（退出码 4），按提示重新授权即可。

退出码：0 成功 · 2 用法错误 · 3 认证失效 · 4 scope/绑定不足 · 5 不存在 · 7 配额。

## 意图 → 命令映射

| 意图 | 命令 |
|---|---|
| 看我在哪些公司 | `crab hr orgs` |
| 新建公司 | `crab hr org create <名称>`（创建者是 owner；新令牌要重新 `crab login --scopes hr --org org:<新名>` 绑定） |
| 看成员与权限 | `crab hr members` |
| 拉人进来 | `crab hr member add <用户名> [--title 头衔]`（进来是 member 只读） |
| 给人配权限 | `crab hr member set <memberId> --role admin` 或 `--permissions people.write=1,tasks.write=1`（flags：org.manage / people.write / tasks.write / sensitive.read；owner 恒全权） |
| 移除成员 | `crab hr member rm <memberId>` |
| 未注册的老板要进来 | `crab hr invite`（常驻链接，注册即入组织；`--reset` 作废旧链） |
| 看花名册 | `crab hr people [--q 关键字] [--status active\|probation\|resigned]` |
| 看一个人 | `crab hr person <id>` |
| 入职建档 | `crab hr person create --name "姓名" --hired-at YYYY-MM-DD [--dept 部门] [--role 岗位] [--probation-end YYYY-MM-DD] [--contract-end YYYY-MM-DD]` |
| 转正 | `crab hr person edit <id> --status active`（自动落痕） |
| 办离职 | `crab hr person offboard <id> [--date YYYY-MM-DD]`（档案封存不删，记录依法保留） |
| 记一件事 | `crab hr event add <personId> --type <onboard\|contract\|change\|attendance\|payroll\|award\|asset\|offboard\|note> --note "..." [--at YYYY-MM-DD]` |
| 看时间线 | `crab hr events [--person <id>] [--type <类型>]` |
| 从表格导入花名册 | `crab hr import <文件.csv|.tsv|.json>`（≤200 行，姓名/入职列必填，表头自动识别） |
| 看任务 | `crab hr tasks [--status alert\|confirm\|queued\|done]` |
| 建任务 / 流转 | `crab hr task create --kind attendance --title "9 月考勤月结"`；`crab hr task update <id> --status done` |
| **总览（推荐首入口）** | `crab hr desk`——等你拍板 / 助理在办 / 近期完成 + 大事提醒（合同到期 60 天、试用期届满 30 天、过期存量风险）+ 最近动态 |

所有命令支持 `--json` 输出机器可读结构；多组织时用 `--org org:<名称>`（或组织 id）指定，仅一个组织时自动采用。

## 机器可读输出（--json）

- `crab hr people --json` → `{ people: [...], total, org: {...} }`；人员字段 camelCase（`hiredAt`/`contractEnd` 为纪元毫秒；`idNumber`/`salaryCents` 对令牌恒为 `null`——见下方纪律）
- `crab hr desk --json` → `{ pending: [...], running: [...], done: [...], reminders: [{ text, days }], recentEvents: [...] }`
- `crab hr import --json` → `{ created, failed: [{ row, reason }] }`

## 纪律

- **敏感信息不主动读、读到不转述**：身份证号与薪资对 CLI/Agent 令牌在服务端就不可见（恒为 null，与「无权限」同形）。用户当面问到某人薪资时，引导其在 Web 端查看，不要猜测。
- **写操作先复述再执行**：建档 / 离职 / 移除成员 / 重置邀请链接前，先向用户复述将要发生的动作；离职是封存不是删除，但仍是敏感流转。
- 事件时间线是**劳动纠纷证据链**：只追加、不改写；`event add` 的 note 要写清事实（什么节点做了什么），不要写推测。
- 多组织账号务必确认 `--org` 后再写，写错组织的数据不属于本组织。
