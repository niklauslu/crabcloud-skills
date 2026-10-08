---
name: crabcloud-atelier
description: Fashion R&D workshop management via the `crab atelier` CLI — apply to create an atelier, manage members & permission sets, invite links, roles, collections, styles (with stage markers), sample rounds (proto/fit/pp with structured fitting notes and verdicts), the org-wide record stream and the desk digest. Use when the user asks anything about their fashion atelier on Crab Cloud: 工坊、款式、样衣、打样、审版、样衣轮次、系列、款号、配色、设计师、服装研发、核价准备、建坊、成员、权限、邀请链接、岗位、记录、工作台 — even if they never say "crab".
---

# Crab Cloud 服装研发工坊（crab atelier）

管理服装研发工坊：成员与权限集、邀请链接、岗位（权限组合模板）、系列（企划容器）、
款式库（款号 + 阶段定位标记 + 配色 + 样衣轮次）、样衣时间线（头样/改样/产前样 +
结构化修改意见 + 审版结论）、全量记录流与工作台聚合。独立产品「Atelier · Crab
Cloud」，同一套 REST API 服务 Web 与 CLI。款（style）是唯一锚点——建档、轮次、
结论都挂在款上并自动落时间线。

跨域时参考：邮件 `crabcloud-mail`、云盘 `crabcloud-storage`、协作
`crabcloud-collab`、店铺 `crabcloud-shop`、人事行政 `crabcloud-hr`、平台账号
`crabcloud`。

## 第一步：确认可用

- 运行 `crab --version` 确认已安装。`crab` 命令缺失时先 `npx @crabcloud/cli init`
  （只登录 + 装 skill、不含命令本体）；已装但缺 `atelier` 命令用
  `npx @crabcloud/cli@latest init` 升级。不想装全局则整段跳过，下文所有
  `crab <命令>` 直接 `npx @crabcloud/cli <命令>` 等价执行。
- `crab whoami` 报「未登录」或退出码 3 → 引导用户 `crab login`。授权页摆出全
  目录供用户勾选，**实际授予集由人定**——登录后先 `crab whoami` 核对 scopes。
- 永远不要让用户把 token 粘贴进对话；凭证文件与你无关，也不要读取它。

atelier 命令需要 **atelier scope + 工坊绑定**的令牌：`crab login --scopes atelier
--org org:<工坊名>`。未绑工坊或绑错工坊会得到 `atelier_org_not_bound` /
`atelier_org_mismatch`（退出码 4），按提示重新授权即可。

退出码：0 成功 · 2 用法错误 · 3 认证失效 · 4 scope/绑定不足 · 5 不存在 · 7 配额。

## 工坊生命周期（申请-审核制）

建坊走**申请-审核**（与建公司/开店同款）：提交申请 → 平台审核 → 通过时扣
**1 积分**并自动建坊（申请人成 owner）。每账号同时只有一条待审申请。

| 意图 | 命令 |
|---|---|
| 看我在哪些工坊 | `crab atelier orgs` |
| 申请创建工坊 | `crab atelier org apply <名称> [--note 说明]`（审核通过自动建坊） |
| 查申请进度 | `crab atelier org application`（pending / approved〔含驳回意见〕/ 从未申请） |
| 工坊改名 | `crab atelier org rename <新名称>`（settings 权限；改名后 `org:<名称>` 标识随新名） |

## 成员、权限与邀请

权限模型：角色 owner（恒全权不可移除）/ member，member 按**权限集**（七个写权限
flags 的数组）判定：`styles`（款式库）、`records`（记录）、`materials`（面辅料）、
`costing`（核价）、`production`（生产）、`team`（成员管理）、`settings`（组织设置）。
只读不需要 flag——active 成员默认可读全部。岗位 = 命名的权限组合模板（成员不存
岗位、套用即拷贝权限集），`roles` 只读可查。

| 意图 | 命令 |
|---|---|
| 看成员与权限 | `crab atelier members [--page N]` |
| 拉人进来 | `crab atelier member add <用户名> [--title 头衔]`（加入后一律 member 只读） |
| 给人配权限 | `crab atelier member set <memberId> --permissions styles,records`（**整体替换**；`--permissions none` = 只读） |
| 设工坊内显示名 | `crab atelier member set <memberId> --permissions styles --display-name "阿May"`（permissions 必填一并提交） |
| 移除成员 | `crab atelier member rm <memberId>` |
| 发邀请链接 | `crab atelier invite`（站内 `/join/:token` 加入页：注册用户一键入坊，恒 member 只读） |
| 作废旧链接 | `crab atelier invite --reset`（旧链立即失效） |

多工坊账号用 `--org org:<名称>`（或工坊 id）指定；仅一个工坊时自动采用。

## 系列与款式库

阶段是**定位标记不是状态机**（企划 → 设计 → 打样 → 审版 → 核算 → 生产 → 完成；
搁置为旁路）——任意阶段可编辑、可补录，阶段变更自动落时间线。金额一律**整数分**
（`--retail-cents 19900` = ¥199.00）。

| 意图 | 命令 |
|---|---|
| 看系列 | `crab atelier collections`（名称/季度/在库款数） |
| 建系列 | `crab atelier collection create "2026 秋冬" [--season AW26] [--note 备注]` |
| 看款式库 | `crab atelier styles [--q 关键字] [--stage 打样] [--collection <id>] [--page N]`（带阶段分布计数） |
| 看一款 | `crab atelier style <id>`（卡片 + 轮次时间线 + 最近事件，一次取全） |
| 款式建档 | `crab atelier style create --code KH001 --name "羊毛大衣" [--category 大衣] [--collection <id>] [--designer ...] [--retail-cents 199000] [--colorways 驼色,黑] [--notes ...]` |
| 编辑款式 | `crab atelier style edit <id> --stage 打样`（给的 flag 才改；`--collection clear` 解绑系列） |
| 登记配色 | `crab atelier style edit <id> --colorways 驼色,黑,雾蓝`（整体替换） |

款号 = 工坊内唯一业务主键（`--code`，自由录入）；建档自动落时间线。

## 样衣轮次与审版

轮次类型：`proto` 头样 / `fit` 改样（自动递增改样序号）/ `pp` 产前样。流程：
发起轮次 → 打样中 → （登记修改意见）→ 登记回样转审版中 → 下结论。
结论流转：`approved`（产前样通过自动转核算）/ `rejected`（打回重做，回打样）/
`dropped`（放弃，款式搁置）。

| 意图 | 命令 |
|---|---|
| 看某款的轮次 | `crab atelier samples <styleId>`（新→旧，含意见清单） |
| 发起轮次 | `crab atelier sample add <styleId> --kind proto [--maker 版房名] [--due 2026-11-01] [--note ...]` |
| 登记修改意见 | `crab atelier sample note <styleId> <sampleId\|轮次号> --part 领口 --issue "领头不平服" [--fix "改领衬"]`（追加一条） |
| 下审版结论 | `crab atelier sample verdict <sampleId> --verdict approved [--note "准产"]` |

## 记录与工作台

| 意图 | 命令 |
|---|---|
| 全量记录流 | `crab atelier records [--type sample\|costing\|production\|note] [--page N]`（各款时间线汇聚） |
| **总览（推荐首入口）** | `crab atelier desk`——款式统计带 + 阶段分布 + 待审样衣（reviewTodos）+ 最近记录，一次取全 |

## 机器可读输出（--json）

- `crab atelier orgs --json` → `[{ id, name, status, myRole, myPermissions, memberCount }]`
- `crab atelier styles --json` → `{ styles: [...], total, page, totalPages, stageCounts }`；
  款式字段 camelCase，`latestRound` = 最新一轮摘要（kind/fitSeq/roundNo/status）
- `crab atelier style <id> --json` → `{ style, samples, events }`（轮次新→旧、事件新→旧最近 50）
- `crab atelier samples <styleId> --json` → `[{ id, kind, fitSeq, roundNo, status, costCents, notes: [{part, issue, fix}], ... }]`
- `crab atelier records --json` → `{ records: [...], total, page, totalPages }`
- `crab atelier desk --json` → `{ org, stats: { totalStyles, activeStyles, samplingStyles, reviewPending, stageCounts }, reviewTodos, recentRecords }`
- `crab atelier invite --json` → `{ token, createdAt, link }`
- 时间字段一律纪元毫秒；金额字段 `*Cents` 一律整数分

## 纪律

- **写操作先复述再执行**：建坊申请 / 改名 / 拉人 / 改权限 / 移除成员 / 重置邀请链接 /
  建档 / 改阶段 / 发轮次 / 下审版结论前，先向用户复述将要发生的动作。
- **权限是整体替换**：`member set --permissions` 是「该成员写权限全集」，不是增量——
  改一个人的权限先 `members` 看现值再给全集，避免误删既有权限。
- 审版结论改变款式走向（approved 转核算 / rejected 回打样 / dropped 搁置），下结论前
  与用户确认；结论与意见是研发档案证据链，写了就进时间线。
- 金额一律整数分（`--retail-cents` 等 `*-cents` flag）；禁止元/浮点输入。
- 阶段是定位标记不是审批流：不要替用户「推进阶段」，除非明确要求。
- 多工坊账号务必确认 `--org` 后再写，写错工坊的数据不属于本工坊。
