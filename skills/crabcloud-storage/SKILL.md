---
name: crabcloud-storage
description: Manage the user's Crab Cloud Drive (云盘 — unified storage for drive uploads, media assets and mail attachments on one quota, with end-to-end encryption) through the `crab` CLI — list and filter objects, upload and download files (including decrypting share links), rename, tag, categorize, publish/unpublish public links, create expiring share links with passcodes, manage the locally remembered encryption key, trash/restore/purge, and check the storage quota. Use this skill when the user asks to upload, find, organize, share or clean up files on Crab Cloud ("上传这个文件", "云盘里有什么", "把这个文件公开分享", "看看存储用了多少", "crab storage"), or wants files moved in or out of their drive — even if they never say "crab" or "Crab Cloud" explicitly.
---

# Crab Cloud 云盘（crab storage）

云盘是用户的统一存储：云盘上传 / 素材库 / 邮件附件三来源共用一本 2 GB 容量账；
属性优先、位置无关——分类（至多一个）+ 标签（AND 检索）代替文件夹。files 域
上传默认端到端加密（服务器只有包裹态密钥，明文密钥只在本机）。本 skill 覆盖
云盘域的 agent 通道；账号/令牌管理见 `crabcloud` skill，邮箱见 `crabcloud-mail`
skill。

## 第一步：确认可用

- 运行 `crab --version` 确认已安装。命令不存在时让用户运行
  `npx @crabcloud/cli init`（登录 + 安装本 skill 一步完成）；已装过但命令缺失
  时运行 `npx @crabcloud/cli@latest init` 升级。
- CLI 基于 npx 分发，**本机没有 `crab` 命令时，下文所有 `crab <命令>` 都可以
  `npx @crabcloud/cli <命令>` 等价执行**（例如 `npx @crabcloud/cli storage ls`）。
- 未登录（退出码 3）引导用户 `crab login`。
- 动作需要对应 scope：列取/下载 `storage.read`，上传/改名/标签/分类/公开
  `storage.write`，移回收站/恢复/彻底删除 `storage.delete`。退出码 4 = scope
  不足：如实转述缺哪个，引导 `crab login --scopes storage`（组名 = 三项整组）
  重新授权（你只发起，不代批）。
- 退出码 7 = `storage.quotaExceeded`（统一容量账满，2 GB）：告知用户，可建议
  清理回收站或大文件（`storage quota` 看分项），不要自动重试。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 云盘里有什么 | `crab storage ls`（按时间倒序，默认 30 条；`--limit` / `--offset` 翻页） |
| 找文件 | `crab storage ls --q 关键字`（文件名匹配；可叠 `--source files\|media\|mail`、`--type doc\|image\|video\|archive\|code`、`--category 分类id\|none`、`--tag a,b`、`--shared`、`--recent`、`--message-id <id>`） |
| 看回收站 | `crab storage ls --trash` |
| 上传文件 | `crab storage upload <路径...>`（多文件串行；直传优先，自动回退中转；默认端到端加密——口令三来源见下节，首次使用自动初始化并打印一次性恢复码） |
| 下载文件 | `crab storage download <id\|文件名\|分享链接> [-o 输出路径] [--passcode 提取码]`（默认按原文件名存当前目录；**凭完整分享链接可免登录下载并本地解密**——`#` 后是密钥片段，这是 agent→agent 的主通道；带提取码的分享加 `--passcode`） |
| 改名 | `crab storage rename <id\|文件名> <新名称>` |
| 加/删标签 | `crab storage tag <id\|文件名> --add a,b [--remove c,d]` |
| 归类 | `crab storage category <id\|文件名> <分类id\|分类名\|none>`（分类名唯一命中即解析） |
| 建分类 | `crab storage mkcat <名称> [--color 色]` |
| 改分类名/颜色 | `crab storage rencat <id\|分类名> <新名称> [--color 色]` |
| 删分类 | `crab storage rmcat <id\|分类名>`（分类下文件回落未分类，不删文件） |
| 公开分享 | `crab storage publish <id\|文件名>`（打印公开直链；收回用 `unpublish`） |
| 可控分享（有效期/提取码/撤销） | `crab storage share <id\|文件名> [--expires 7\|30\|90\|forever] [--passcode 提取码]`（打印完整分享链接；加密对象链接自带 `#` 密钥片段）；列表 `crab storage shares <id\|文件名>`、撤销 `crab storage unshare <shareId>` |
| 云盘钥匙管理 | `crab storage key`（查状态）/ `--remember`（解锁一次并记住，之后免口令）/ `--forget`（清除）——见下节 |
| 删除 | `crab storage rm <id\|文件名>`（进回收站，7 天后自动清理）→ `restore` 恢复 / `purge` 彻底删除 |
| 看水位 | `crab storage quota`（三来源分项 + 回收站占用 + 对象计数） |

`<id|文件名>`：文件名全库精确唯一命中时自动解析为 id（回收站内同样可解析，
restore/purge 直接用文件名即可）；命中多个或查不到时按字面当 id 用——拿不准
先 `crab storage ls --q <关键字>` 核对再操作。分类指认同款：id 精确命中优先，
名称唯一命中其次（`rencat` / `rmcat` / `--category` 都吃 `<id|分类名>`）。

## 机器可读输出

程序化消费**始终加 `--json`**：

- `storage ls --json` → `{ objects: [...], nextOffset, total }`，条目含
  `id / source / filename / mimeType / sizeBytes / categoryId / tags / isPublic / createdAt`；
- `storage quota --json` → `{ usedBytes, quotaBytes, bySource: { files, media, mail }, trashBytes, counts }`；
- `upload` / `categories` / `tags` / `shares` 的 `--json` 同构（单对象 / 分类数组 /
  标签数组 / 分享数组）。

## 云盘口令与本机钥匙

files 域是端到端加密：服务器只有口令包裹态的主密钥，**明文密钥只存在于用户
本机**——CLI 不可能也不允许向服务器索取。涉及加密对象的命令（upload /
download / share 加密文件）需要云盘口令，来源优先级：

1. `--passphrase <口令>`（flag）
2. 环境变量 `CRAB_PASSPHRASE`（**Agent/脚本推荐**——非交互终端不会弹提问，
   缺失时明确报错）
3. 交互终端（TTY）隐藏输入提问（人不经管道跑 CLI 时的默认体验）

本机钥匙（`crab storage key`）：口令解锁成功后可把**主密钥**（非口令）记住到
本机 `~/.config/crabcloud/drive-key-<账号id>.json`（0600，按账号+环境隔离），
之后所有命令免口令——等同 Web 端「记住此设备」。`--remember` 主动记住、
`--forget` 清除、默认查状态；Web 端轮换口令/恢复码后本机钥匙自动验签失效并
回落口令，无需手工干预。Agent 视角：**用户没让你动钥匙设置时不要主动调
`storage key`**；用户口令遗忘是恢复码流程（Web 端），CLI 侧无法代找回。

## 纪律

- **publish 是对外公开**：公开后任何持链接者可访问，先向用户复述对象与后果，
  确认后再执行；收回用 `unpublish`（已缓存浏览器至多 5 分钟内失效）。
- **share 是可控分享**：默认 7 天有效、可选提取码、随时 `unshare` 撤销；加密
  对象的分享链接自带解密钥匙（`#` 后缀）——泄露链接即泄露文件，外发前向
  用户复述有效期与提取码设置，确认后再执行。收件人在浏览器打开加密分享链接
  会看到内联解密落地页（图片/视频/音频/PDF/文本直接预览，其余类型仅下载）。
- **提取码错误重试有服务端限流**：同一链接+来源 IP 错误次数超限返回 429
  （退出码 1，消息「尝试过于频繁」）——**不要程序化暴力重试**，如实转述用户
  等待即可；凭链接下载时提取码用 `--passcode` 正确携带。
- **purge 是彻底删除**，不可恢复；用户没明说「彻底删除」时一律用 `rm`
  （回收站保留 7 天可找回）。
- 邮件附件与用户上传同账：水位告急时用 `quota` 分项定位来源再建议清理。
- 公开直链的呈现方式由服务端策略决定（图片/视频/音频/PDF/纯文本内联，SVG 与
  其他类型强制下载），CLI 侧没有、也不需要有开关。
