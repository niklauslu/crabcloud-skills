---
name: crabcloud-short
description: Short-drama studio via the `crab short` CLI — create series, episodes (with editable scripts), a subject library (characters/scenes/props with appearance prompts and generated-image references), and two-level storyboards (scenes → shots with shot size, camera move, dialogue, duration, image/video prompts, subject links). Also manages the studio itself: apply/create/switch studios, members & permission sets, invite links, roles. Also sells episodes as products: list targets, create products with spec variants and quantity limits, place and pay orders, issue license codes. Built as an AI production workbench: agents read the series bible and write scripts/storyboards back. Use when the user asks anything about their short-drama studio: 短剧、剧目、剧集、分集、剧本、分镜、场、镜头、景别、运镜、主体库、角色、场景、道具、生图、prompt、美术风格、发布、制作状态、工作台、售卖、商品、规格、订单、兑换码、工作室、建坊、成员、权限、邀请链接、岗位 — even if they never say "crab".
---

# Short 短剧工作室（crab short）

短剧团队的制作工作台：**剧（series）→ 集（episode）** 是唯一主轴。剧挂对外
档案 + 内部制作信息（背景设定/总体大纲/美术风格指导）；集挂剧本（可反复改）
与分镜；**主体库（角色/场景/道具）挂剧、跨集复用**；分镜两级——场（sequence）
→ 镜头（shot：画面/动作/台词 + 景别/运镜 + 时长 + 图视频 prompt + 主体关联）。

跨域时参考：云盘 `crabcloud-storage`（成片/素材同库）、邮件 `crabcloud-mail`、
协作 `crabcloud-collab`、店铺 `crabcloud-shop`、平台账号 `crabcloud`。

## 第一步：确认可用

- 运行 `crab --version` 确认已安装。`crab` 命令缺失时先 `npx @crabcloud/cli init`
  （只登录 + 装 skill、不含命令本体）；已装但缺 `short` 命令用
  `npx @crabcloud/cli@latest init` 升级。不想装全局则整段跳过，下文所有
  `crab <命令>` 直接 `npx @crabcloud/cli <命令>` 等价执行。
- `crab whoami` 报「未登录」或退出码 3 → 引导用户 `crab login`。**实际授予集
  由人定**——登录后先 `crab whoami` 核对 scopes。
- 永远不要让用户把 token 粘贴进对话；凭证文件与你无关，也不要读取它。

short 命令分两侧：

- **管理侧**（看概览、建剧/改剧、传素材、成员/岗位/邀请、售卖管理）需要
  **short scope 且绑定工作室**的令牌：登录时直接带上
  `crab login --scopes short --org org:<工作室名>`。
- **买家侧**（浏览商品、下单、支付、我的订单、兑换码）任何 short scope 令牌
  可用，不需要绑定。

**报「当前令牌没有绑定工作室」时**（或换一个工作室干活）：

```
crab short orgs                          # 看你在哪些工作室
crab short org switch org:<工作室名>      # 浏览器授权一次，即绑定/换绑
crab short org                           # 核对当前绑定的工作室
```

退出码：0 成功 · 2 用法错误 · 3 认证失效 · 4 scope 不足 · 5 不存在 · 7 配额。

## AI 制作约定（生图/生视频输入怎么拼）

生成图像时按此顺序拼接 prompt，保证全剧视觉一致：

1. **剧级美术风格指导**（`short series show <id>` 的 `styleGuide`）；
2. **主体外观提示词**（`short subjects <seriesId>` 里关联主体的 `appearancePrompt`）；
3. **镜头生图 prompt**（shot 的 `imagePrompt`）。

生成结果用 `--image <素材对象id>` 挂回主体或镜头留痕，生视频同理
（`videoPrompt` / 镜头 `--video`）。**素材必须用 `crab short upload <文件>` 上传**
（见下节「素材」）——用云盘 `crab storage upload` 传的文件挂不到 short 上
（即便挂上，页面也读不到）。

## 素材（成片 / 封面 / 生成图：云盘一本账）

| 意图 | 命令 |
|---|---|
| 上传素材 | `crab short upload <文件> [--org …]` → 输出**对象 id**（`--json` 取 `fileId`） |
| 挂成片 | `crab short episode edit <epId> --video <对象id>`（`clear` 解除） |
| 挂剧封面 | `crab short series edit <seriesId> --cover <对象id>` |
| 挂主体生成图 | `crab short subject edit <id> --image <对象id>` |
| 挂镜头生成结果 | `crab short shot edit <id> --image <对象id>` / `--video <对象id>`（create 也能带） |
| 看占用 | `crab short stats`（本工作室占用 + 所有者云盘池用量） |

规则：
- **素材归档在工作室所有者的云盘「短剧制作」分类**，配额是所有者的云盘总池，
  **同一所有者的多个工作室共享一池**；池满会报存储配额不足——清云盘或升档位
  （额度是云盘的，不是 short 的）。
- **上限**：图片 10MB / 视频 200MB。CLI **不压缩**；web 上传弹窗会自动压
  （视频 720p / 图片 1080p），大视频优先走 web 传，CLI 适合 agent 生成图、小样。
- **只能挂本工作室上传的对象**：别的工作室的、云盘直接上传的、已删除的都挂
  不上；解除挂接传 `clear`。
- 替换素材 = 挂新 id；旧对象留在所有者云盘（不自动删，所有者在云盘回收站清；
  还被引用的对象云盘侧删不掉）。

## 工作室生命周期（申请-审核制）

建工作室走**申请-审核**（与建坊/建库同款）：提交申请 → 平台审核 → 通过时扣
**1 积分**并自动建工作室（申请人成 owner）。每账号同时只有一条待审申请。

| 意图 | 命令 |
|---|---|
| 看我在哪些工作室 | `crab short orgs`（当前绑定的带 `← 当前` 标记） |
| 看当前工作室 | `crab short org`（当前绑定的工作室与 scope） |
| 切换工作室 | `crab short org switch <org:名称\|id>`（浏览器授权一次即换绑，原令牌不撤销；多工作室轮换用） |
| 申请创建工作室 | `crab short org apply <名称> [--note 说明]`（审核通过自动建） |
| 查申请进度 | `crab short org application`（pending / approved〔含驳回意见〕/ 从未申请） |
| 工作室改名 | `crab short org rename <新名称>`（settings 权限；改名后用新名字引用即可，已有绑定不受影响） |

## 成员、权限与邀请

权限模型：角色 owner（恒全权不可移除）/ member，member 按**权限集**判定，共
五项：`content`（剧/集/剧本/分镜）、`media`（素材上传与挂接）、`distribution`
（兑换码、售卖、受众运营）、`team`（成员管理）、`settings`（组织改名、品牌
标识、收款账户）。只读不需要任何权限——active 成员默认可读全部。岗位是
**纯职务标识**（编剧/导演/剪辑等，不影响权限），`roles` 查名录、
`role add/rename/rm` 管理。新成员初始权限取工作室「新成员默认权限」
（默认只读，Web 设置页可配）。

| 意图 | 命令 |
|---|---|
| 看成员与权限 | `crab short members [--page N]` |
| 拉人进来 | `crab short member add <用户名> [--title 头衔]`（平台账号用户名；初始权限 = 新成员默认权限） |
| 给人配权限 | `crab short member set <memberId> --permissions content,media`（**整体替换**；`--permissions none` = 只读） |
| 给人挂岗位 | `crab short member set <memberId> --permissions <现有权限不动> --role 剪辑`（`--role none` = 清除；permissions 必填一并提交） |
| 设工作室内显示名 | `crab short member set <memberId> --permissions content --display-name "小剪"`（permissions 必填一并提交） |
| 看岗位名录 | `crab short roles`（名字 + 实挂成员数） |
| 建岗位 | `crab short role add 剪辑`（纯职务标识，不能重名；挂人用上面 member set --role） |
| 岗位改名 | `crab short role rename <roleId> <新名称>`（实时生效，不影响已挂成员） |
| 删岗位 | `crab short role rm <roleId>`（**= 摘标记**：挂该岗位的成员被清岗位标记，权限不动；删前与用户复述） |
| 移除成员 | `crab short member rm <memberId>`（可随时重新拉入） |
| 发邀请链接 | `crab short invite`（站内 `/join/:token` 加入页：注册用户一键入工作室） |
| 作废旧链接 | `crab short invite --reset`（旧链立即失效） |

改自己的权限/岗位会被拒——让 owner 或其他有 team 权限的成员来操作。

## 剧与分集

| 意图 | 命令 |
|---|---|
| 看工作室概览 | `crab short stats`（剧目/分集计数 + 存储占用） |
| 建剧 | `crab short series create "剧名" --genre 都市 --logline "一句话" --audience 女频 --episodes 60` |
| 剧目列表 | `crab short series [--status draft\|published\|archived\|all]` |
| 剧详情 | `crab short series show <id>`（基础档案 + 制作信息 + 集数进度） |
| 编辑剧 | `crab short series edit <id> [--logline … --outline … --style … --status …]`（给的 flag 才改） |
| 填制作信息 | `--background`（世界观背景）`--outline`（总体大纲）`--style`（美术风格指导）——内部文档，不进对外档案 |
| 分集列表 | `crab short episodes <seriesId>` |
| 新建分集 | `crab short episode create <seriesId> --title "第1集"`（seq 自动顺延） |
| 写剧本 | `crab short episode edit <id> --script "…"`（整体替换；修改是常事，直接覆盖） |
| 推制作状态 | `--prod-status outline\|scripting\|prep\|shooting\|post\|ready` |
| 发布 | `--published on`（制作态要先推到 ready，否则发不了） |
| 挂成片 | `--video <对象id>`（对象 id 来自 `crab short upload`；`clear` 解除） |
| 下架 | `--published off`（已发布的集要先下架才能降制作态，保持发布的同时降级会被拒） |
| 删集 | `crab short episode rm <id>`（分镜一并删除，不可恢复） |

剧状态：`draft → published → archived`。归档不是删除——archived 剧的内容
只读（原样保存不算改动），唯一出口是 `series edit <id> --status draft|published`
改回来。

## 主体库（挂剧，跨集复用）

| 意图 | 命令 |
|---|---|
| 列主体 | `crab short subjects <seriesId> [--kind character\|scene\|prop]` |
| 建角色/场景/道具 | `crab short subject create <seriesId> "名字" --kind character --prompt "外貌/服装/气质"` |
| 补声音设定 | `--voice "低沉沙哑"`（角色用） |
| 编辑 | `crab short subject edit <id> [--prompt … --voice …\|clear --image <素材id>\|clear]` |
| 挂生成图 | 先 `crab short upload 图片` 拿对象 id → `subject edit <id> --image <对象id>`（**不是** storage upload） |
| 删主体 | `crab short subject rm <id>` |

同剧同类型不能重名。`--prompt` 是生图输入的核心——写具体
（画风/外貌/服装/气质），生成时与剧的 styleGuide 拼接。

## 分镜（场 → 镜头，挂集）

| 意图 | 命令 |
|---|---|
| 看整树 | `crab short board <episodeId>`（场→镜头一次取全，写分镜前的必读物） |
| 添加场 | `crab short sequence create <episodeId> --title "天台·夜"`（场号自动顺延） |
| 改场 | `crab short sequence edit <id> --title "…"` |
| 添加镜头 | `crab short shot create <sequenceId> --desc "画面" --size 特写 --move 固定 --duration 8 --dialogue "…" --subjects su_x,su_y` |
| 镜头 prompt | `--image-prompt`（生图）/ `--video-prompt`（生视频）；镜号自动顺延 |
| 挂生成结果 | `--image <对象id>` / `--video <对象id>`（`crab short upload` 产物；`clear` 解除） |
| 编辑镜头 | `crab short shot edit <id> [同 create 的 flags]`（给的 flag 才改；--subjects 整组替换，clear 全摘） |
| 删场/删镜 | `crab short sequence rm <id>`（含场内镜头）/ `crab short shot rm <id>` |

镜头字段：`--desc` 画面描述、`--action` 动作/表演、`--dialogue` 台词、
`--size` 景别（远景/全景/中景/近景/特写…自由文本）、`--move` 运镜（固定/推/
拉/摇/移/跟…）、`--duration` 预估秒数、`--subjects` 关联主体 id（只能关联
本剧的主体）。

## 售卖（商品 / 规格 / 订单 / 兑换码）

**集是唯一可售目标**（能上架的集 = 已发布 + 非免费 + 剧在架）。卖出的交付物
是**兑换码**：买家买 N 件发 N 个码，买家在分集页输码解锁；价格、订单、退款
都在这套命令里管。

### 卖家侧

| 意图 | 命令 |
|---|---|
| 看可售目标（哪些集能上架） | `crab short targets`（已发布 + 非免费 + 剧在架的集） |
| 上架商品 | `crab short product create --target <集id> --offer-key <键> --title "标题" --price 1990` |
| 商品列表 | `crab short products`（含规格分组与变体） |
| 改商品 | `crab short product edit <id> [--title … --price <分> --status … --max-per-order N --max-per-buyer N --shipping <分>]` |
| 下架商品 | `crab short product rm <id>`（历史订单不受影响） |
| 规格变体 | `crab short variants <productId>` / `crab short variant upsert <productId> --variant "title=高清;price=1990;画质=高清"` |
| 下架变体 | `crab short variant rm <productId> <variantId>`（下架不影响历史订单） |
| 卖家订单 | `crab short orders`（本工作室的订单） |
| 订单详情 | `crab short order <id>`（金额三段式 + N 个兑换码） |
| 退款 | `crab short order refund <id> --reason "原因"`（全额；重复提交不会重复退） |
| 发出的兑换码 | `crab short licenses --org <arg>`（不带 `--org` = 我买到的） |

**金额一律整数分**（`--price 1990` = 19.90 元），禁元/浮点。

### 规格（多规格商品）

**规格 = 一个商品 + 规格配置**（不是多个商品）：

- `--spec "画质=高清,超清;套餐=单集"` 声明**分组与可选取值**（组间 `;`，组内
  取值 `,`）；
- `--variant "sku=EP1-HD;title=高清单集;price=1990;画质=高清;套餐=单集"` 建
  **一条可售配置**（`sku/title/price/status/限购/库存/排序` 是功能键，**其余
  键都当规格取值**）；
- 买家只能选到你**建出变体的组合**，没建的组合在前台直接置灰；
- 变体可各自定价 / 各自下架 / 各自限购。

### 买家侧

| 意图 | 命令 |
|---|---|
| 某目标在售商品 | `crab short catalog <集id>`（买家视角） |
| 下单 | `crab short buy --product <productId> [--variant-id <变体id> --spec-values "画质=高清"] [--qty N] [--address "<json>"]` |
| 我买过的单 | `crab short my-orders` |
| 可用支付方式 | `crab short payment-methods <orderId>`（只列同计价单位的） |
| 去支付 | `crab short pay <orderId>`（法币返回收银台地址，实付由网关回调驱动） |
| 取消订单 | `crab short order close <id>`（只关未付的单；关单释放优惠占用） |

买家身份就是当前登录的 crab 账号，不用（也不能）替别人下单。`--qty` 买 N 件
= 发 N 个码；实物商品必须带 `--address "<json>"`（收货地址）。

## 典型工作流（从零到一部剧）

```
crab short series create "剧名" --logline "…" --style "赛博朋克霓虹夜景，高对比"
crab short subject create <id> "林晚" --kind character --prompt "25岁女性，黑色皮衣，冷白皮"
crab short episode create <id> --title "第1集"
crab short episode edit <epId> --script "第一场…" --prod-status scripting
crab short sequence create <epId> --title "天台·夜"
crab short shot create <seqId> --desc "林晚回望城市" --size 中景 --move 推 --duration 6 \
  --subjects su_x --image-prompt "林晚立于天台边缘回望，霓虹倒影"
# 生成图后：crab short upload 生成图.png → 拿对象 id → crab short shot edit <shotId> --image <对象id>
# 成片：crab short upload 成片.mp4 → crab short episode edit <epId> --video <对象id> --prod-status ready
```
