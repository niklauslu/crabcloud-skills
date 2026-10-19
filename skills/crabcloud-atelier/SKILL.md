---
name: crabcloud-atelier
description: Fashion R&D workshop management via the `crab atelier` CLI — apply to create an atelier, manage members & permission sets, invite links, roles, collections, styles (with stage markers and archive/restore), the category dictionary for style creation, design-stage documents (design/pattern/sample drafts with round-based reviews), the supplier directory (fabric vendors & factories), production orders (size breakdown, delivery, factory, frozen costing snapshot), the org-wide record stream and the desk digest. Use when the user asks anything about their fashion atelier: 工坊、款式、样衣、打样、审版、样衣轮次、系列、款号、配色、设计师、服装研发、核价准备、建坊、成员、权限、邀请链接、岗位、记录、工作台、供应商、面辅料、加工厂、品类、归档、制单、生产单、排产、交期、投产、工艺单 — even if they never say "crab".
---

# Atelier 服装研发工坊（crab atelier）

管理服装研发工坊：成员与权限集、邀请链接、岗位（纯职务标识）、系列（批次容器）、
款式库（款号 + 阶段定位标记 + 配色 + 打样稿件）、设计阶段时间线（稿件上传 /
审版定版留痕）、供应商目录（面辅料商家 / 加工厂）、全量记录流与工作台聚合。独立产品 Atelier，同一套 REST API 服务 Web 与 CLI。
款（style）是唯一锚点——建档、轮次、结论都挂在款上并自动落时间线。

跨域时参考：邮件 `crabcloud-mail`、云盘 `crabcloud-storage`、协作
`crabcloud-collab`、店铺 `crabcloud-shop`、人事行政 `crabcloud-hr`、平台账号
`crabcloud`。

## 网页操作在哪点（references/web-ui.md）

用户问「网页上在哪里操作 / 点哪里」——建款、传稿、审版、核价、制单、报数、
入库、加人调权限、打印单据——按 **`references/web-ui.md`** 指路（同目录分册）：
产品库四板块 + 款式详情六流程 tab（信息/打样/核算/制单/报数/入库）+ 记录页
三 tab + 设置页四 tab，含常见意图 → 操作位置对照表与权限速查。

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
| 看我在哪些工坊 | `crab atelier orgs`（当前绑定的带 `← 当前` 标记） |
| 看当前工坊 | `crab atelier org`（令牌绑定的工坊与 scope；绑定是服务端状态） |
| 切换工坊 | `crab atelier org switch <org:名称\|id>`（浏览器授权一次即换绑，原令牌不撤销；多工坊轮换用） |
| 申请创建工坊 | `crab atelier org apply <名称> [--note 说明]`（审核通过自动建坊） |
| 查申请进度 | `crab atelier org application`（pending / approved〔含驳回意见〕/ 从未申请） |
| 工坊改名 | `crab atelier org rename <新名称>`（settings 权限；改名后 `org:<名称>` 标识随新名） |

## 成员、权限与邀请

权限模型：角色 owner（恒全权不可移除）/ member，member 按**权限集**（七个写权限
flags 的数组）判定：`styles`（款式库）、`records`（记录）、`materials`（面辅料）、
`costing`（核价）、`production`（生产）、`team`（成员管理）、`settings`（组织设置）。
只读不需要 flag——active 成员默认可读全部。岗位是**纯职务标识**（设计师/版师等，
不影响权限；0124 起与权限解耦），`roles` 查名录、`role add/rename/rm` 管理；
「管理员 = 全功能、成员 =
只读」只是 Web 成员抽屉的快捷预设。新成员初始权限取工坊「新成员默认权限」（默认
只读，Web 设置页可配）。

| 意图 | 命令 |
|---|---|
| 看成员与权限 | `crab atelier members [--page N]` |
| 拉人进来 | `crab atelier member add <用户名> [--title 头衔]`（初始权限 = 工坊新成员默认权限，默认只读） |
| 给人配权限 | `crab atelier member set <memberId> --permissions styles,records`（**整体替换**；`--permissions none` = 只读） |
| 给人挂岗位 | `crab atelier member set <memberId> --permissions <现有权限不动> --role 版师`（`--role none` = 清除；permissions 必填一并提交） |
| 设工坊内显示名 | `crab atelier member set <memberId> --permissions styles --display-name "阿May"`（permissions 必填一并提交） |
| 看岗位名录 | `crab atelier roles`（名字 + 实挂成员数） |
| 建岗位 | `crab atelier role add 版师`（纯职务标识，重名 409；挂人用上面 member set --role） |
| 岗位改名 | `crab atelier role rename <roleId> <新名称>`（实时生效，不影响已挂成员） |
| 删岗位 | `crab atelier role rm <roleId>`（**= 摘标记**：挂该岗位的成员被清岗位标记，权限不动；删前与用户复述） |
| 移除成员 | `crab atelier member rm <memberId>` |
| 发邀请链接 | `crab atelier invite`（站内 `/join/:token` 加入页：注册用户一键入坊，初始权限 = 工坊新成员默认权限） |
| 作废旧链接 | `crab atelier invite --reset`（旧链立即失效） |

多工坊账号用 `--org org:<名称>`（或工坊 id）指定；仅一个工坊时自动采用。

## 系列批次与款式库

**系列批次**（批次容器）：标题 + 年月（`YYYY-MM` 必填）+ 选填说明 + 归档状态——
归档不是删除，可随时恢复；归档后新建款式不可再挂该系列，存量款式引用不受影响。
**款式归档**同理（2026-10-14）：归档款从默认列表退场（`styles --status archived|all`
带出），记录与引用全部保留、可恢复、无删除。**品类是辅助输入不是关联**（2026-10-14
拍板）：品类字典只供建档时选词与记录页管理，款式上的品类仍是自由文本、不挂 id——
`--category` 直接写文本即可，想进字典供团队复选再 `category create`。
款式阶段是**定位标记不是状态机**（设计 → 核算 → 生产 → 完成；打样与审版收在
设计段内）——任意阶段可编辑、可补录，阶段变更自动落时间线。金额一律**整数分**
（`--retail-cents 19900` = ¥199.00）。款号 = 工坊内唯一业务主键（重复建档 409）。

| 意图 | 命令 |
|---|---|
| 看系列 | `crab atelier collections [--status active\|archived\|all] [--q 关键字]`（默认只看进行中；带状态计数） |
| 建系列 | `crab atelier collection create "2026 秋冬首批" --ym 2026-09 [--note 备注]`（年月必填） |
| 改系列 | `crab atelier collection edit <id> [--name ... --ym 2026-10 --note ...]`（给的 flag 才改） |
| 归档 / 恢复 | `crab atelier collection archive <id>` / `crab atelier collection restore <id>`（幂等无删除） |
| 看品类 | `crab atelier categories [--status active\|archived\|all] [--q 关键字]`（默认只看在册；带状态计数） |
| 建品类 | `crab atelier category create "连衣裙"`（同 org 重名 409；归档同名同样占用，恢复即可） |
| 改品类 | `crab atelier category edit <id> --name 新名称`（撞名 409） |
| 归档 / 恢复品类 | `crab atelier category archive <id>` / `crab atelier category restore <id>`（归档后建档选择器不再出，已录款式文本不受影响） |
| 看款式库 | `crab atelier styles [--q 关键字] [--stage 核算] [--collection <id>] [--status active\|archived\|all] [--page N]`（默认只看在库；--q 搜款号/品名/品类/色系；带阶段与状态计数） |
| 看一款 | `crab atelier style <id>`（信息按组展示：01 基础信息 / 02 商品规格 / 03 生产信息 / 04 尺寸表〔部位 × 尺码矩阵 + 档差，尺码列随商品规格联动〕，+ 轮次时间线 + 最近事件，一次取全） |
| 款式建档 | `crab atelier style create --code KH001 --name "羊毛大衣" [--category 大衣] [--collection <id>] [--designer ... --pattern-maker 纸样师 --sample-maker 车版师] [--retail-cents 199000] [--notes ...]`（款号唯一；归档系列不可挂）。**开款 ≠ 补录**：建档只收基本字段，色板/尺码/工艺开款后补录（Web 详情页分组编辑或下方 CLI flag 均可；建档时也可一并带上，见 `--colorways/--sizes/--craft-notes`） |
| 归档款式 | `crab atelier style archive <styleId>` / `crab atelier style restore <styleId>`（幂等无删除；归档款详情仍可看，时间线落归档句） |
| 编辑款式 | `crab atelier style edit <id> --stage 生产 [--pattern-maker 纸样师 --sample-maker 车版师]`（给的 flag 才改；`--collection clear` 解绑系列；人员字段均选填自由文本） |
| 关联加工厂 | `crab atelier style create/edit <...> --factory <加工厂id\|名称>`（值引用供应商目录 type='factory'；`--factory clear` 解绑；面辅料型供应商会被拒；详情 03 生产信息与制单建单默认带出） |
| 设款式封面 | `crab atelier style cover <styleId> <本地图片路径>`（png/jpg/webp/gif ≤20MB；走工坊素材中转上传，素材自动进 owner 云盘「服装研发」分类，Web 产品库/详情画册位即显；`--clear` 移除引用素材不删）。封面引用只认工坊素材——云盘直传的通用文件挂不上 |
| 登记规格 | `crab atelier style edit <id> --colorways 驼色,黑,雾蓝 --sizes S,M,L --craft-notes "领口包边，成衣水洗"`（各项各自整体替换——空数组/空串 = 清空；色板与尺码是商品需要的规格，特殊工艺是生产需要的，Web 详情页即按此分组） |
| 登记尺寸表 | `crab atelier style edit <id> --size-chart "衣长:沿肩缝直量:2:S=66,M=68,L=70|胸围::S=100"`（**整体替换**，`clear` = 清空；行间 `\|`，行内 `部位:测量方式:档差:码=值,码=值`——测量方式段可整体省略，档差可省略或显式留空〔`部位::` = 无档差〕，值 0–1000 留空 = 该格不记；尺码列与商品规格联动，先配 `--sizes` 再录尺寸表；Web 信息 tab 04 组同源。部位枚举（KHARITES 纸单）：后中长/肩宽/胸围/腰围/臀围/脚围/袖长/袖肥/袖口围/夹圈/裤长·裙长/脾围/膝围/前浪/后浪/肩带长——Web 下拉选择 + 其他自填，CLI 自由文本均可） |

## 供应商目录

面辅料商家 `fabric` / 加工厂 `factory` 两类，字段 = 类型 + 名称 + 联系人 +
联系方式（电话/微信自由文本单字段）+ 地址 + 备注。归档不是删除——归档后供应商
选择器不再出（面辅料库、制单生产引用时），存量引用不受影响，可随时恢复。
写需 `materials` 或 `production` 任一权限。

| 意图 | 命令 |
|---|---|
| 看供应商 | `crab atelier suppliers [--type fabric\|factory] [--status active\|archived\|all] [--q 关键字]`（默认只看在册；带状态与类型双维计数） |
| 建档 | `crab atelier supplier create "绍兴晨曦纺织" --type fabric [--contact-name 王经理 --contact 139xxx --address 地址 --note 备注]` |
| 编辑 | `crab atelier supplier edit <id> [--type ... --name ... --contact-name ... --contact ... --address ... --note ...]`（给的 flag 才改） |
| 归档 / 恢复 | `crab atelier supplier archive <id>` / `crab atelier supplier restore <id>`（幂等无删除） |

## 打样（设计阶段稿件与审版）

轮模型（2026-10-08 定稿）：**上传即当前稿，未审版都属同一轮**——设计稿 / 纸样稿 /
样衣稿三类稿件，每款每类至多一个未定版当前稿（再传 = 修改当前稿：素材/备注/
关联人员整体替换）；**审版是款级动作**（不挂单个稿件）——通过时当前轮全部未定版
稿件各自定版（每类 style+kind 内递增 vN，如「设计稿 v1、纸样稿 v1」），不通过留
当前稿继续改。素材与审版图走 Web 端上传（atelier 存储域，owner 云盘「服装研发」
分类）。**打样不进 CLI**——旧 `crab atelier samples/sample ...` 命令面已随轮模型
退役（已从命令面移除），打样操作请走 Web 款式详情「打样」tab；CLI 侧款式
`style <id>` 详情可看款式卡片与时间线。

## 记录与工作台

| 意图 | 命令 |
|---|---|
| 看核算单 | `crab atelier costing <styleId>`（面辅料/工艺/其他成本行 + 单件合计 + 工艺说明；面辅料行带成份/门幅/克重/颜色/部位/缩率/烫朴/制版单耗/损耗/提供方式/备注等属性行；rowId 都在这里取。A4 打印单在 Web 款式详情「核算」tab） |
| 加面辅料行 | `crab atelier material add <styleId> --name "澳毛纱" --category fabric --supplier <id\|前缀\|名称> --ref-price-cents 4800 --qty 0.4 --unit 公斤 [--article-no 货号 --spec 规格 --purpose 主面料 --composition "65%锦35%棉" --width 135 --gsm 70 --loss-pct 15 --pattern-usage 0.27 --colors 黑,橘 --supply-mode 现货\|订纺 --parts 所用部位 --shrinkage "经3%纬3%" --interlining-thickness 30g --note 备注 --image 本地图片路径]…`（--qty/--pattern-usage 小数；**--width 单位 cm、--loss-pct 单位 %**；金额一律 --*-cents 整数分；供应商引用 id/前缀/名称，clear = 解绑；--image 小样图可重复 ≤3 张，material edit 另支持 `--image clear` 清空） |
| 加工艺行 | `crab atelier process add <styleId> --name "打鸡眼" --qty 100 --price-cents 100 [--requirement 要求 --supplier ... --unit 次]`（金额自动计入成本核算） |
| 加其他成本 | `crab atelier extra add <styleId> --name "包装" --price-cents 3000 [--qty 1 --unit ...]`（包装/运费/损耗） |
| 改/删核算行 | `crab atelier material\|process\|extra edit <rowId> [同 add 的 flag，给的才改]` / `… rm <rowId>`（合计随单自动重算；行编辑与 A4 打印单也可在 Web 核算 tab） |
| 全量记录流 | `crab atelier records [--type sample\|costing\|production\|inbound\|purchase\|note] [--page N]`（各款时间线汇聚） |
| **工坊审计（owner 专属）** | `crab atelier audits [--action 前缀] [--page N]`——工坊维度操作留痕（谁做了什么、含 CLI/agent 动作）；非 owner 令牌 403。分类筛选用前缀组：`--action "atelier.member,atelier.role,atelier.invite"` |
| **总览（推荐首入口）** | `crab atelier desk`——款式统计带（总数/进行中/打样中/完成）+ 阶段分布 + 资源速览（在册系列/面辅料/加工厂）+ 最近记录，一次取全 |

## 制单与生产（写闸 production）

**制单 = 三态生命周期的投产通知单**（2026-10-09 定稿）：**未确认（草稿，可
编辑）→ 确认（定稿：快照冻结、可打印工艺单）→ 作废（单向终态：不可打印、
留档）**。只有已确认单据出 A4 工艺单（读确认时点快照、**不含任何金额**，
留档 + 给工厂）。标题默认「款式名 + 日期」可改。规格明细 = 「色板 × 尺码」
矩阵：尺寸必有（款式未录尺码 400，先补商品规格）；色板可有可无（单色款不
填色）。加工厂只收 factory 类型供应商（留空 = 自加工）；交期 = 预计交期。
**工序列表 = 单据级、默认四道（裁剪/车缝/后整/质检）**：草稿期可增删改名
（毛织无裁剪、加「绣花（外发）」等例外），确认后冻结，报数按工序名挂。
建单不自动推进款式阶段（阶段是定位标记，需手动 `style edit --stage 生产`）。
**报数与回货（2026-10-09 拍板；2026-10-11 矩阵化）**：仅已确认单据可报数/
登记回货；两者都按单据规格「色 × 码」逐格报数（`--breakdown` 矩阵语法同制
单，总数 = 服务端按格求和派生、客户端不传总数；单据没订的色码格不可报）。
报数流水只增不改（QC 简化 = 质检工序的次品数即结果，次品全单口径 ≤ 总数）；
回货分批多次，单据累计对账「已入库 X/件数 Y」，累计 ≥ 件数即可在 Web 入库
tab 一键标记款式完成。**超量不拦（流水账语义）**：累计可超单据件数（超产/
多交真实存在）——Web 侧超量弹窗二次确认，CLI 提交成功后输出 ⚠ 超量警告行；
代报时见到 ⚠ 先向用户复述确认再继续。Web 操作位置见 `references/web-ui.md`。
**采购记录（2026-10-12 切片）**：细化 planning「一期不做独立采购单」——逐条
台账挂已确认制单，**物料必须关联核算面辅料行**（`--material` = 核算行 id，须
在该单确认时点快照清单内；名称/货号/规格/单位由服务端从快照带出，不自由填）。
`--amount-cents` = 实付金额整数分（非单价×数量派生）；到货两态单向（待到货 →
已到货）。对账口径：用量 = 快照单件耗料 × 件数，差数即余缺（用量/已买/实付为 0 时输出省略该段，与 Web 0 值空白同口径）。删除是金额录错
兜底（时间线留痕不撤）；Web 的「生产」tab = 采购/报数/入库合流（选单 + 工单头
  速览 + 三板块上下平铺：01 采购 / 02 报数 / 03 入库）。

| 意图 | 命令 |
|---|---|
| 看一款的制单 | `crab atelier orders <styleId> [--status draft\|confirmed\|voided]`（三态计数随行） |
| 看一单 | `crab atelier order <orderId>`（标题 + 规格明细 + 工序报数累计 + 入库对账 + **采购对账与流水** + **报数/回货流水〔含色码矩阵〕** + 快照合计；orderId 见 orders） |
| 建草稿 | `crab atelier order add <styleId> --breakdown "黑色:S=30,M=45\|米白:S=20" [--title 标题 --supplier <id\|前缀\|名称> --delivery 2026-10-20 --notes 备注]`（--breakdown 色码=件数：多色 `\|` 分段、`色:` 前缀；单色款直接 `"S=10,M=20"`） |
| 改草稿 | `crab atelier order edit <orderId> [--title ... --breakdown "..." --supplier ... --delivery ... --notes ...]`（仅未确认可编辑） |
| 确认定稿 | `crab atelier order confirm <orderId>`（快照冻结、可打印；定稿后不可再改） |
| 作废 | `crab atelier order void <orderId>`（单向终态：不可打印、留档） |
| 报数 | `crab atelier order log <orderId> --step 裁剪 --breakdown "S=10,M=20" [--defect 2 --note 备注]`（--step 须在单据工序列表内〔order 详情可看〕；--breakdown 按单据规格逐格报、总数服务端按格求和；次品 ≤ 总数） |
| 登记回货 | `crab atelier order inbound <orderId> --breakdown "S=10,M=20" [--note 备注]`（本批按单据规格逐格填；分批多次；差数 = 未交尾数） |
| 看采购台账 | `crab atelier order purchase <orderId>`（物料对账段：用量/已买/余缺/实付合计 + 采购流水） |
| 记采购 | `crab atelier order purchase add <orderId> --material <costMaterialId> --qty 2.5 --amount-cents N [--supplier <id\|前缀\|名称> --note 备注]`（仅已确认单；--material 须在该单快照内〔order 详情可看快照物料〕；实付金额整数分） |
| 标记到货 | `crab atelier order purchase recv <purchaseId>`（待到货 → 已到货，单向不可逆） |
| 删采购 | `crab atelier order purchase rm <purchaseId>`（金额录错兜底；时间线留痕不撤） |

## 机器可读输出（--json）

- `crab atelier orgs --json` → `[{ id, name, status, myRole, myPermissions, memberCount }]`
- `crab atelier styles --json` → `{ styles: [...], total, page, totalPages, stageCounts }`；
  款式字段 camelCase，`latestRound` = 旧样衣轮次模型遗留摘要（新档为 null）
- `crab atelier style <id> --json` → `{ style, samples, events }`（samples 为旧模型遗留空表、恒 `[]`；事件新→旧最近 50）
- `crab atelier suppliers --json` → `{ suppliers: [...], total, page, totalPages, statusCounts, typeCounts }`
- `crab atelier costing <styleId> --json` → `{ styleId, craftNotes, materials, processes, extras, totals: { materialsCents, processCents, extrasCents, totalCents } }`（行金额服务端算好；数量为毫单位 ×1000）
- `crab atelier orders <styleId> --json` → `{ orders: [...], total, page, totalPages, statusCounts: { draft, confirmed, voided } }`；订单含 `title`、`orderNo`（款号-seq）、`breakdown`（色板→尺码→件数两级矩阵）、`steps`（工序列表）、`progress`（各工序报数累计）、`inboundQty`（累计入库）、`costingSnapshot`（确认时点冻结）
- `crab atelier order <orderId> --json` → 制单全量视图（含快照三表 + totals + sizes/sizeChart + progress + inboundQty + `logs` 报数流水〔`qtyDone` 服务端按矩阵派生 + `breakdown`〕 + `inbounds` 回货流水〔同上〕 + `purchases` 采购台账 + `purchaseSummary` 物料对账〔`demandMilli` 用量/`purchasedMilli` 已买/`amountCents` 实付合计/`pendingCount` 待到笔数〕）
- `crab atelier order purchase <orderId> --json` → `{ purchases, summary }`（台账 + 对账一次取）
- `crab atelier records --json` → `{ records: [...], total, page, totalPages }`（type 含 inbound = 回货入库事件）
- `crab atelier audits --json` → `{ events: [...], total, page, totalPages }`（事件含 `action`/`actorName`/`actorType`〔agent = 令牌动作〕/`meta`；仅 owner 令牌可查）
- `crab atelier desk --json` → `{ org, stats: { totalStyles, activeStyles, doneStyles, samplingStyles, reviewPending, stageCounts, activeCollections, suppliersFabric, suppliersFactory }, reviewTodos, recentRecords }`
- `crab atelier invite --json` → `{ token, createdAt, link }`
- 时间字段一律纪元毫秒；金额字段 `*Cents` 一律整数分

## 纪律

- **写操作先复述再执行**：建坊申请 / 改名 / 拉人 / 改权限 / 移除成员 / 重置邀请链接 /
  建档 / 改阶段 / 发轮次 / 下审版结论前，先向用户复述将要发生的动作。
- **权限是整体替换**：`member set --permissions` 是「该成员写权限全集」，不是增量——
  改一个人的权限先 `members` 看现值再给全集，避免误删既有权限。
- 审版是款级动作（通过 = 当前轮稿件各自定版），下结论前与用户确认；结论与意见是研发档案证据链，写了就进时间线。
- 金额一律整数分（`--retail-cents` 等 `*-cents` flag）；禁止元/浮点输入。
- 阶段是定位标记不是审批流：不要替用户「推进阶段」，除非明确要求。
- 系列**归档不是删除**（可恢复）；归档后新建款式不可再挂，不要建议「删掉重建」。
- 款号是工坊内唯一业务主键：建档前如不确定是否已有同款号，先 `styles --q <款号>` 查。
- 供应商**归档不是删除**（可恢复）；归档只影响选择器，存量引用不受影响——不要
  建议「删掉重录」。联系人/联系方式是自由文本，不要拆成结构化字段。
- 制单生命周期：草稿可编辑 → **确认定稿**（不可再改、可打印）→ 作废终态
  （不可打印、留档）。确认/作废前与用户复述单据标题与数量矩阵。
- 报数/回货**只对已确认单据**，且流水只增不改（写错不能撤，报数前与用户
  复述工序与色码数量）；两者都按单据规格「色 × 码」矩阵逐格报（`--breakdown`），
  总数服务端派生——不要凭总数拼矩阵（单据没订的格子会被 400 拒）；报数不影响
  时间线，回货每笔落「入库」记录。
- 多工坊账号务必确认 `--org` 后再写，写错工坊的数据不属于本工坊。
