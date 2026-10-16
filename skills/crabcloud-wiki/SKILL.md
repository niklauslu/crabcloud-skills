---
name: crabcloud-wiki
description: Organization knowledge base (LLM wiki) via the `crab wiki` CLI — navigate by index (root index → section index → document, no RAG), ingest material as structured markdown documents with one-line summaries, cross-link related pages, grant per-node read/write access to members, run structure lint, export the whole library as a skill-references style folder. Use when the user asks anything about their knowledge base or internal docs: 知识库、wiki、内部资料、文档库、资料库、归档、整理资料、索引、目录、摘要、手册、交接文档、产品资料、制度、know-how、导入文档、整理 PDF、授权、可读可写、体检、时间线 — even if they never say "crab".
---

# Wiki 知识库（crab wiki）

组织内部资料的典藏馆：**索引进正文，分册按需读**。目录与文档都是节点，每个
节点必有一行摘要——摘要就是父目录索引里的一行。全库默认私有，文件按人授权
（可读 / 可写两档；目录授权对整棵子树生效）。这是 Karpathy llm-wiki 模式的
组织级实现：**你是维护者**——摄取、回链、体检都是你的活，维护负担由你承担。

跨域时参考：邮件 `crabcloud-mail`、云盘 `crabcloud-storage`、协作
`crabcloud-collab`、店铺 `crabcloud-shop`、人事行政 `crabcloud-hr`、平台账号
`crabcloud`。

## 三操作：ingest / query / lint

**query（查）**：回答问题先读索引，禁止整库扫读——
`crab wiki read root`（根索引）→ 看中条目 → `crab wiki read <路径>`（目录出
索引、文档出正文）→ 顺「相关页面」扩展。综合类答案值得归档：写成新文档
（如「综合/」目录下），让探索也复利。

**ingest（摄）**：把一份材料整理入库——读原文 → 和用户确认要点 → 找挂载
目录（没有合适的先建目录）→ `crab wiki add` 写入 → `crab wiki link` 回写
相关页 → `crab wiki lint` 收尾。一次摄入一个源，保持参与。实战细节（多文件、
去重、frontmatter）按 **`references/ingest-playbook.md`** 指路（同目录分册）。

**lint（体检）**：改动后 `crab wiki lint`——空摘要、空目录、悬挂互链、跨层
重名、stale 候选；矛盾与过时断言这类语义体检按用户要求做。

## 第一步：确认可用

- 运行 `crab --version` 确认已安装。`crab` 命令缺失时先 `npx @crabcloud/cli init`
  （只登录 + 装 skill、不含命令本体）；已装但缺 `wiki` 命令用
  `npx @crabcloud/cli@latest init` 升级。不想装全局则整段跳过，下文所有
  `crab <命令>` 直接 `npx @crabcloud/cli <命令>` 等价执行。
- `crab whoami` 报「未登录」或退出码 3 → 引导用户 `crab login`。wiki 命令
  还要求令牌绑定知识库：`crab login --scopes wiki --org org:<名称>`（授权页
  批准后生效；`crab wiki orgs` 看绑定与列表）。
- 退出码：0 成功 · 2 用法错误 · 3 认证失效 · 4 scope/绑定不足 · 5 资源不存在。

## 命令表

| 意图 | 命令 |
| --- | --- |
| 我的库 | `crab wiki orgs` |
| 看结构 | `crab wiki tree`（穿越容器标「部分可见」；✎可写 👁可读） |
| 看一层 | `crab wiki ls <id\|路径>` |
| 读 | `crab wiki read <id\|路径>`（目录出索引 markdown，文档出正文） |
| 建目录 | `crab wiki add-dir --parent <id\|路径> --title <标题> --summary <一行摘要> [--description 前言]` |
| 建文档 | `crab wiki add --parent <id\|路径> --title <标题> --summary <一行摘要> [--content-file <md\|-]` |
| 改 | `crab wiki edit <id\|路径> [--title] [--summary] [--content-file <md\|-]` |
| 移动 | `crab wiki mv <id\|路径> --parent <id\|路径\|root> [--before <同级id>]` |
| 互链 | `crab wiki link <id\|路径> --to <id\|路径> [--unlink]` |
| 授权 | `crab wiki grant <id\|路径> --to <用户名> [--write]` |
| 收权 | `crab wiki revoke <id|路径> --from <用户名>` |
| 权限 | `crab wiki acl <id\|路径>`（我的档位与来源 + 直授名单） |
| 体检 | `crab wiki lint` |
| 时间线 | `crab wiki log [--limit N]`（owner 专属） |
| 传图 | `crab wiki upload <文件>`（返回正文引用片段） |
| 导出 | `crab wiki export [--out 目录]`（每层 INDEX.md + AGENTS.md，可直接挂给 agent） |
| 导入 | `crab wiki import <本地md目录> [--parent <id|路径>] [--dry-run]` |

寻址统一 `<id|路径>`：索引行尾的 `(wn_…)` 是 id；路径用标题分段（`产品资料/
安装部署`）；裸标题全库唯一命中也可用，多命中会列候选要求消歧。

## 机器可读输出（--json）

所有命令支持 `--json`（结构化 data 直出）。程序化消费一律加 `--json` 解析
stdout，不要解析人类可读格式；`tree`/`log` 的文本款仅供人看。

## 纪律

1. **先读索引再下钻**——索引进正文是这个库的消费协议；不要 `--json` 整库
   拉取后自行排序检索。
2. **摘要必填且写好**——它出现在父目录索引里，写给「决定要不要读的人」；
   禁「各种说明」「相关资料」这类空词。
3. **正文不重复 H1 标题**——页面与索引已经展示标题，正文从引言写起。
4. **超长拆分**——单文档上限 100k 字符，到不了就该拆：一个主题一个节点，
   目录即分类。
5. **ingest 后回链**——新文档与既有相关页面互 `link`，保持知识网连通。
6. **权限不绕过**——你以绑定成员的身份行事，成员看不到的你也不该看；
   需要访问就引导用户找 Owner 开授权（`crab wiki grant`）。
7. **写前复述**——创建/编辑前向用户复述：标题、摘要、挂载目录；批量导入
   先 `--dry-run` 给用户过目。
