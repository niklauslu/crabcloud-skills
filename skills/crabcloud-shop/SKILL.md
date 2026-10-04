---
name: crabcloud-shop
description: Manage the user's Crab Cloud shop stores through the `crab` CLI — store status, opening applications, products, members and role presets. Use this skill when the user asks about their shops, wants to open or apply for a store, check orders or products, manage store members/roles ("我的店铺", "开店", "商店订单", "店铺成员", "crab shop"), or asks what their stores look like — even if they never say "crab" or "Crab Cloud" explicitly.
---

# Crab Cloud 商店（crab shop）

用户的 Crab Cloud 账号可以开店卖数字/实物商品（无账号买家直购、平台代收款）。
本 skill 覆盖商店域的 agent 通道；账号/令牌管理见 `crabcloud` skill。

## 第一步：确认可用

- 运行 `crab --version` 确认已安装。本机没有 `crab` 命令时引导
  `npm i -g @crabcloud/cli@latest` 安装；不想装全局则下文所有 `crab <命令>`
  直接 `npx @crabcloud/cli <命令>` 等价执行。
- 未登录（退出码 3）引导用户 `crab login`（商店命令跟随登录身份的店铺成员
  关系鉴权；成员权限不足会返回 `missing permission: <项>`，如实转述缺哪项）。
- **多店账号**：先 `crab shop use <slug>` 设置当前店（持久保存，店级命令
  缺省作用于它）；单条命令也可 `--store <slug>` 临时覆盖。只有一家店时全部
  自动选中。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 我有哪些店 / 开店申请进度 | `crab shop status [--json]` |
| 切换当前管理的店（多店账号） | `crab shop use <slug>`（持久；status 里 ★ = 当前店） |
| 开店 / 再开一家 | `crab shop apply <店铺名> [--tagline 一句话介绍]` |
| 店铺角色预设（权限模板） | `crab shop roles --store <slug>` |
| 店铺成员（权限组合/业务身份） | `crab shop members --store <slug>` |
| 改成员权限 / 业务身份 | `crab shop member edit <用户名> --perms products,orders --biz sales --store <slug>` |
| 查看 / 重置常驻邀请链接 | `crab shop invite [--reset --perms orders,customers --biz sales] --store <slug>` |
| 看分类（多级树） | `crab shop categories --store <slug>`（缩进树 + 商品数 + id） |
| 加分类 / 加子分类 | `crab shop category add <名称> [--parent <id或名称>] --store <slug>`（新建落同级末尾） |
| 分类改名 / 排序 / 删除 | `crab shop category rename <id或名称> <新名称>` · `crab shop category move <id或名称> <up\|down>`（同级换位，边界不动）· `crab shop category delete <id或名称>`（有子级拒绝；叶子删除后商品迁入「未分类」） |
| 看规格库模板 | `crab shop specs --store <slug>`（名称/选项值/引用商品数） |
| 建 / 改规格模板 | `crab shop spec add <名称> --values <值1,值2,…>` · `crab shop spec update <id或名称> [--name <新名>] [--values <a,b,c>]` |
| 删规格模板 | `crab shop spec delete <id或名称>`（引用 = 快照拷贝：已建商品不受影响） |

分类/模板的 `<id或名称>` 参数：id 优先，名称须唯一（重名会提示改用 id）。

## 权限项（member edit --perms 可用值）

`products`（商品与库存）· `orders`（订单与发货）· `aftercare`（售后与退款）·
`customers`（客户与归属）· `marketing`（营销）· `finance`（账单与争议）·
`team`（团队管理）· `settings`（店铺设置）

分类与规格库的写操作（add/rename/move/delete）需要 `products` 权限；列表
只需成员身份。

业务身份（--biz）：`sales` / `support`，逗号分隔可多选——决定客户归属与业绩
归因资格，与权限正交。

## 纪律

- 店主不可改权限（由店铺归属决定）；改成员权限前先 `shop members` 确认目标。
- `shop invite --reset` 会使旧链接立即作废——先和用户确认再重置。
- 金额一律整数分（API 返回 `*_cents`，展示 ÷100 加货币符号）。
- 订单/商品/库存等经营命令按批次扩充；本 skill 随 CLI 更新同步维护。
