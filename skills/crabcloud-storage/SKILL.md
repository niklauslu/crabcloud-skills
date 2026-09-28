---
name: crabcloud-storage
description: Manage the user's Crab Cloud Drive (云盘 — unified storage for drive uploads, media assets and mail attachments on one quota) through the `crab` CLI — list and filter objects, upload and download files, rename, tag, categorize, publish/unpublish public links, trash/restore/purge, and check the storage quota. Use this skill when the user asks to upload, find, organize, share or clean up files on Crab Cloud ("上传这个文件", "云盘里有什么", "把这个文件公开分享", "看看存储用了多少", "crab storage"), or wants files moved in or out of their drive — even if they never say "crab" or "Crab Cloud" explicitly.
---

# Crab Cloud 云盘（crab storage）

云盘是用户的统一存储：云盘上传 / 素材库 / 邮件附件三来源共用一本 2 GB 容量账；
属性优先、位置无关——分类（至多一个）+ 标签（AND 检索）代替文件夹。本 skill
覆盖云盘域的 agent 通道；账号/令牌管理见 `crabcloud` skill，邮箱见
`crabcloud-mail` skill。

## 第一步：确认可用

- `crab --version` 确认已安装；未登录（退出码 3）引导用户 `crab login`。
- 动作需要对应 scope：列取/下载 `storage.read`，上传/改名/标签/分类/公开
  `storage.write`，移回收站/恢复/彻底删除 `storage.delete`。退出码 4 = scope
  不足：如实转述缺哪个，引导 `crab login --scopes storage`（组名 = 三项整组）
  重新授权（你只发起，不代批）。
- 退出码 7 = `storage.quotaExceeded`（统一容量账满，2 GB）：告知用户，可建议
  清理回收站或大文件（`storage quota` 看分项），不要自动重试。

## 意图 → 命令映射

| 用户意图 | 命令 |
|---|---|
| 云盘里有什么 | `crab storage ls`（按时间倒序，默认 30 条） |
| 找文件 | `crab storage ls --q 关键字`（文件名匹配；可叠 `--source files\|media\|mail`、`--type doc\|image\|video\|archive\|code`、`--tag a,b`、`--shared`、`--recent`） |
| 看回收站 | `crab storage ls --trash` |
| 上传文件 | `crab storage upload <路径...>`（多文件串行；直传优先，自动回退中转） |
| 下载文件 | `crab storage download <id\|文件名> [-o 输出路径]`（默认按原文件名存当前目录） |
| 改名 | `crab storage rename <id\|文件名> <新名称>` |
| 加/删标签 | `crab storage tag <id\|文件名> --add a,b [--remove c,d]` |
| 归类 / 建分类 | `crab storage category <id\|文件名> <categoryId\|none>`；`crab storage mkcat <名称>`；分类 id 见 `crab storage categories` |
| 公开分享 | `crab storage publish <id\|文件名>`（打印公开直链；收回用 `unpublish`） |
| 删除 | `crab storage rm <id\|文件名>`（进回收站，7 天后自动清理）→ `restore` 恢复 / `purge` 彻底删除 |
| 看水位 | `crab storage quota`（三来源分项 + 回收站占用 + 对象计数） |

`<id|文件名>`：文件名全库精确唯一命中时自动解析为 id（回收站内同样可解析，
restore/purge 直接用文件名即可）；命中多个或查不到时按字面当 id 用——拿不准
先 `crab storage ls --q <关键字>` 核对再操作。

## 机器可读输出

程序化消费**始终加 `--json`**：

- `storage ls --json` → `{ objects: [...], nextOffset }`，条目含
  `id / source / filename / mimeType / sizeBytes / categoryId / tags / isPublic / createdAt`；
- `storage quota --json` → `{ usedBytes, quotaBytes, bySource: { files, media, mail }, trashBytes, counts }`；
- `upload` / `categories` / `tags` 的 `--json` 同构（单对象 / 分类数组 / 标签数组）。

## 纪律

- **publish 是对外公开**：公开后任何持链接者可访问，先向用户复述对象与后果，
  确认后再执行；收回用 `unpublish`（已缓存浏览器至多 5 分钟内失效）。
- **purge 是彻底删除**，不可恢复；用户没明说「彻底删除」时一律用 `rm`
  （回收站保留 7 天可找回）。
- 邮件附件与用户上传同账：水位告急时用 `quota` 分项定位来源再建议清理。
- 公开直链的呈现方式由服务端策略决定（图片/视频/音频/PDF/纯文本内联，SVG 与
  其他类型强制下载），CLI 侧没有、也不需要有开关。
