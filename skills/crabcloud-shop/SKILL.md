---
name: crabcloud-shop
description: Manage the user's Crab Cloud shop stores through the `crab` CLI — store status, opening applications, store settings (SKU prefix, currency, contact info), products, stock and inventory, coupons, announcements and campaigns, storefront homepage decoration blocks, membership tiers, customers and customer wallets (balance, points, recharge packs), members and role presets. Use this skill when the user asks about their shops, wants to open or apply for a store, create or update products, adjust stock, manage membership tiers, check orders or inventory movements, manage store members/roles, check customers or wallet balances ("我的店铺", "开店", "上架商品", "改库存", "出入库", "优惠券", "发券", "公告", "活动", "首页装修", "装修", "会员等级", "商店订单", "店铺成员", "店铺设置", "SKU 前缀", "客服电话", "微信号", "货币", "客户", "余额", "积分", "充值", "钱包", "crab shop"), or asks what their stores look like — even if they never say "crab" or "Crab Cloud" explicitly.
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
| 经营概览 | `crab shop overview`（今日/本月营收与单量、待支付/待发货计数、最近五单） |
| 店铺成员（权限组合） | `crab shop members --store <slug>` |
| 改成员权限 | `crab shop member edit <用户名> --perms products,orders --store <slug>` |
| 移出成员 | `crab shop member rm <用户名> --store <slug>`（客服名片保留、客户归属不受影响） |
| 查看 / 重置常驻邀请链接 | `crab shop invite [--reset --perms orders,customers] --store <slug>` |
| 客服列表（名片/关联成员/名下客户/绑定链接） | `crab shop cs --store <slug>` |
| 添加客服（可选关联店铺成员） | `crab shop cs add <名称> [--member <用户名>] --store <slug>` |
| 编辑客服 / 清除关联 | `crab shop cs edit <id\|前缀> [--name 新名] [--member <用户名>\|none] --store <slug>` |
| 删除客服 / 重置绑定链接 | `crab shop cs rm <id\|前缀>` · `crab shop cs reset <id\|前缀> --store <slug>`（重置后旧绑定链接作废；删除后名下客户转未分配） |
| 看店铺资料（SKU 前缀/货币/联系方式/链接） | `crab shop settings [--store <slug>] [--json]` |
| 改店铺资料 / SKU 前缀 / 货币 / 联系方式 | `crab shop settings edit [--name … --tagline … --description … --sku-prefix … --currency CNY\|USD\|EUR\|GBP\|HKD --contact-email … --contact-phone … --contact-wechat …] --store <slug>`（给 flag 才改；前缀自动大写去非法字符截 8 位、空串 = 纯流水；货币切换全店商品行同步、价格数值不变；联系方式传空串 = 清除；写需 `settings` 权限） |
| 看分类（多级树） | `crab shop categories --store <slug>`（缩进树 + 商品数 + id） |
| 加分类 / 加子分类 | `crab shop category add <名称> [--parent <id或名称>] --store <slug>`（新建落同级末尾） |
| 分类改名 / 排序 / 删除 | `crab shop category rename <id或名称> <新名称>` · `crab shop category move <id或名称> <up\|down>`（同级换位，边界不动）· `crab shop category delete <id或名称>`（有子级拒绝；叶子删除后商品迁入「未分类」） |
| 看商品列表 | `crab shop products [--status draft\|active\|archived] [--category <id或名称>] [--q 文本] [--limit N --offset N]`（多规格行库存 = 变体合计） |
| 看商品详情 | `crab shop product <id\|SKU码>`（先 id 直查、404 再按店内码反查；含规格组合与两层库存） |
| 新建商品 | `crab shop product create --kind physical\|digital --name 名称 --variant-mode single\|multi [--spec-axis "颜色=红,蓝"]… --price-cents 1990 [--status draft\|active]`（类型与规格模式创建即锁定；多规格可先只建轴，组合价后续 spec 配置；**实物创建期不录销售库存**——入库后经 `shop listed` 设额度，`--stock` 仅数字限量） |
| 改商品资料 / 上下架 | `crab shop product update <id\|SKU码> [--name … --price-cents N --category <id\|名称\|none> --status active …]`（类型与规格模式不可变；clear 撤销划线价/限购） |
| 配多规格组合价 / 启停组合 | `crab shop product spec <id> --set "红,M=1990,2500,on" …`（值按轴顺序；未提及组合保持原值；不改库存） |
| 下架归档商品 | `crab shop product archive <id\|SKU码>`（软删；恢复 = product update --status draft） |
| 入库 / 出库 / 盘点 | `crab shop stock in\|out\|set <id\|SKU码> --amount N [--variant "红/M"] --note "说明"`（实物动仓库账；数字限量同命令走上架账；set 或未给 --reason 必带 --note） |
| 设销售库存（上架额度） | `crab shop listed <id\|SKU码> --stock N [--variant …]`（仅实物；0 ≤ N ≤ 实际库存，不够先入库） |
| 看出入库台账 | `crab shop movements [--product <id\|SKU码>] [--limit N --offset N]`（新→旧；发货/退款自动落账也在内） |
| 看优惠券 | `crab shop coupons [--status active\|disabled] --store <slug>`（发放/核销 + 码数；券无主码） |
| 新建优惠券 | `crab shop coupon create --kind amount_off\|percent_off --name 名称 [--amount-cents N \| --percent N] [--min-spend-cents N] [--max-uses N] [--per-user N] [--valid-days N] [--expires YYYY-MM-DD] [--auto-trigger register\|order_paid] --store <slug>`（所有券先领取/兑换再使用；--valid-days = 领取后 N 天有效、--expires = 可领取截止，可并存；--auto-trigger = 新用户自动发 / 该店订单支付后自动发） |
| 给兑换码 / 看兑换码 | `crab shop coupon code add <id> [--count N] [--code 码] [--max-uses N] [--note 渠道]`（批量 ≤100 或单个自定义码，每码独立限次）· `crab shop coupon codes <id>` |
| 定向发券 | `crab shop coupon grant <id> --accounts "用户名1,用户名2" --store <slug>`（平台账号直发，券绑账号进其卡包；找不到用户名会报错）· 邮箱兜底 `--emails "a@x.com,b@x.com"`；`--notify` 发通知邮件 |
| 看订单 | `crab shop orders [--status pending\|paid\|canceled\|refunded] --store <slug>`（新→旧；复制完整订单 id 供 order 子命令） |
| 看订单详情 | `crab shop order <订单id>`（金额口径 / 券减 / 余额抵扣 / 支付流水 / 授权 / 出库记录） |
| 待付单改价 | `crab shop order reprice <订单id> --amount 88.00`（mock 待支付单限定；Stripe 单价格冻结不可改；新应付 ≥ 已抵扣余额） |
| 确认收款 | `crab shop order confirm <订单id>`（线下收款后手动转已支付；mock 单限定；Stripe 单走线上收银台） |
| 取消 / 退款 | `crab shop order cancel <订单id>`（待付单，回补库存 / 释放券与余额）· `crab shop order refund <订单id>`（已支付单全额退款，实物回仓、券与钱包回滚；aftercare 权限） |
| 实物发货 | `crab shop order ship <订单id> --tracking 单号 [--qty N]`（默认发剩余全部；自动扣仓库账；orders 权限） |
| 看领取记录 / 启停 | `crab shop coupon claims <id> [--status unused\|used]` · `crab shop coupon on\|off <id>` |
| 删 / 启停兑换码 | `crab shop coupon code rm <码>`（已使用的码只能停用不能删）· `crab shop coupon code on\|off <码>` |
| 看会员等级 | `crab shop tiers --store <slug>`（按累计实付门槛升序；每店上限 10 级） |
| 加会员等级 | `crab shop tier add <名称> --threshold-cents 10000 [--benefits "权益说明"] [--enabled off] --store <slug>`（默认启用；名称店内唯一；门槛整数分 10000 = $100.00） |
| 发店铺公告 | `crab shop announcement add <内容> [--link <URL>] [--start 2026-10-08T10:00 --end 2026-10-09] [--sort N] --store <slug>`（多条按排序轮播在店面首页；时间窗可空 = 长期） |
| 改 / 删公告 | `crab shop announcement edit <id\|前缀> [--content … --link URL\|none --start …\|none --enabled on\|off]`（只改显式字段，none 清除）· `crab shop announcement rm <id\|前缀>` |
| 看活动 | `crab shop campaigns --store <slug>`（标题/时间窗/四态/关联项；进行中的轮播在店面首页） |
| 发活动（带关联项） | `crab shop campaign add <标题> [--subtitle … --description …] --link "看直播\|https://…" --coupon <券id> --products <商品id,id> --store <slug>`（关联项可重复，顺序即活动页展示顺序） |
| 改 / 删活动 | `crab shop campaign edit <id\|前缀> [--title … --enabled on\|off --sort N]`（关联项 flags 任一给出 = 整单替换全部，`--items-clear` 清空，不给则不动）· `crab shop campaign rm <id\|前缀>` |
| 看首页装修块 | `crab shop decor --store <slug>`（按页面顺序：类型/标题/摘要/启停/id；买家端发现页照此渲染，未装修 = 平台默认页） |
| 加装修块 | `crab shop decor add <banner\|products\|categories\|coupons\|campaigns\|richtext\|links> … --store <slug>`——banner `--slide "图片id\|标题\|副标题\|跳转"` 可重复(1..5)；products `--products <id,id> --style grid\|scroll`；categories `--categories <id,id>`；coupons `--coupons <id,id>`；campaigns `--campaigns <id,id>`；richtext `--text "正文" [--image 图片id]`；links `--link-item "标题\|跳转"` 可重复(1..8)。跳转目标统一 `none\|product:<id>\|category:<id>\|campaign:<id>\|url:<https://…>`；引用须本店存在（bad_blocks） |
| 改装修块 | `crab shop decor edit <id\|前缀> [--title … --slide … --products … --text … --image none --height lg --style scroll --enabled on\|off …]`（读改写只改显式字段；列表字段给了 = 整体替换；类型不可改，改类型 = 删了重建） |
| 装修块启停 / 排序 / 删除 | `crab shop decor on\|off <id\|前缀>`（停用即刻从买家端隐藏）· `crab shop decor move <id\|前缀> <up\|down>`（页面顺序）· `crab shop decor rm <id\|前缀>` |
| 改 / 删会员等级 | `crab shop tier edit <id\|名称> [--name 新名称 --threshold-cents 分 --benefits 文本 --enabled on\|off]`（只改显式给出的字段）· `crab shop tier delete <id\|名称>` |
| 看客户（含余额/积分） | `crab shop customers [--assigned <id\|unassigned>] [--limit N --offset N] --store <slug>`（等级/订单数/消费/余额/积分随列） |
| 看客户档案 / 钱包流水 | `crab shop customer <id\|邮箱> --store <slug>`（资料 + 最近钱包流水：充值/抵扣/释放/返积分明细） |
| 看充值档位 | `crab shop packs --store <slug>`（实付 + 赠送 = 到账余额；买家充值进余额不可提现） |
| 建 / 改 / 删充值档位 | `crab shop pack add --price-cents 10000 [--bonus-cents 1000 --sort N --enabled off]` · `crab shop pack edit <id> [--price-cents … --bonus-cents … --sort … --enabled on\|off]` · `crab shop pack rm <id>`（marketing 权限；每店上限 20；已创建的充值订单不受删档影响） |
| 设消费返积分 | `crab shop settings edit --points-rate N --store <slug>`（每消费 1 元返 N 积分；0 = 不返；写需 settings 权限） |

商品引用统一 `<id|SKU码>`：id 优先，SKU 码店内反查（组合码优先于商品级码）。

## 商品与库存纪律（服务端 fail-closed，客户端前置校验）

- **金额一律整数分**：所有 `--*-cents` 传整数分（1990 = $19.90），禁止元/浮点；
  API 返回 `*_cents`，展示 ÷100 加货币符号。
- **类型与规格模式创建即锁**：physical/digital 与 single/multi 不可改——库存
  挂账层级不同；单规格商品本身即唯一 SKU，多规格的码在每个组合上。
- **规格轴数量首次保存即锁**（轴不能增删）；规格名与选项值可调，但被出入库
  台账引用的组合删不掉（`--set "…,off"` 关闭在售代替）。
- **两层库存**（实物）：`stock` 动的是仓库账（实际库存）；`listed` 设的是
  上架额度（可售），0 ≤ N ≤ 实际库存，不够先 `stock in` 入库再上调。
- **动账留痕**：`--reason purchase` 仅入库有效；set 盘点或未注明原因（落
  manual）必须带 `--note`；退货入库与发货出库由售后/发货端点自动落账，
  不要手工登记。
- 多规格商品动账/设额度必须 `--variant`（组合码 / 变体 id / "值1/值2"）。

## 权限项（member edit --perms 可用值）

`products`（商品与库存）· `orders`（订单与发货）· `aftercare`（售后与退款）·
`customers`（客户与归属）· `marketing`（营销）· `finance`（账单与争议）·
`team`（团队管理）· `settings`（店铺设置）

商品/库存命令的写操作（create/update/spec/archive/stock/listed）需要
`products` 权限；优惠券写操作（coupon create/grant/on/off、coupon code
add/on/off/rm）、公告/活动写操作（announcement/campaign 的 add/edit/rm）与
首页装修写操作（decor add/edit/on/off/move/rm）需要 `marketing` 权限、会员
等级写操作（tier add/edit/delete）需要 `customers` 权限、店铺资料写操作
（settings edit）需要 `settings` 权限；列表、详情与台账
只需成员身份。公告/活动的时间窗可空 = 长期/不限；装修块无时间窗，enabled
即上下架；装修块引用的图片/商品/分类/券/活动必须本店存在（跨店引用报
bad_blocks）。

客服（shop cs）：独立名片实体（头像 web 端传，CLI 管文字字段与绑定链接），
客户归属按客服记——买家扫绑定链接或二维码登录后即绑为该店专属客服（重复
绑定 = 覆盖）；删除客服后名下客户转未分配、绑定链接立即失效。

## 纪律

- 店主不可改权限（由店铺归属决定）；改成员权限前先 `shop members` 确认目标。
- `shop invite --reset` 会使旧链接立即作废——先和用户确认再重置。
- 优惠券纪律：所有券先领取/兑换再使用（买家在店面领券卡或领券中心输兑换码），
  结账一单一券不叠加；`grant --notify` 按封扣积分，积分不足自动跳过邮件；
  发放优先 `--accounts`（绑账号、买家登录即见）；删除仅限未使用的兑换码。
- 订单操作门槛：改价/确认收款仅 mock 待支付单（Stripe 单价格冻结、走线上）；
  取消仅待付单、退款仅已支付单；发货仅实物已支付单。
