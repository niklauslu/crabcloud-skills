# ingest 实战手册（crabcloud-wiki 分册）

何时读本文：用户给了一份或多份原始材料（PDF 导出文本、网页剪存、会议记录、
现成 markdown 目录）让你「放进知识库 / 整理归档」。CLI 主管道见 SKILL.md，
本文只管摄入实战。原文件（PDF/图片）本体**不进 wiki 正文**——wiki 只收
整理好的 markdown；原文件上传到 wiki（`crab wiki upload`）即自动入**收件箱**
（落云盘留痕，`pending` 待整理），整理写回时 `--attach` 挂回原文档留来源。

## 从收件箱摄入（原始文件流，默认路径）

用户说「把收件箱里的资料整理一下」或上传了新文件后：

1. **看队列**：`crab wiki inbox`（缺省只列未处理；`--status all` 看全量）。
2. **取料**：`crab wiki inbox get <fileId>` 下载原文到本地（`-o` 定输出名）。
3. **读原文清洗**：提炼结构——这份材料讲什么？分几个主题？哪些值得留？
   PDF/图片按内容读取；和用户确认拆分与挂载口径。
4. **写入并留痕**：`crab wiki add --parent <id|路径> --title <标题>
   --summary <一行摘要> --content-file <整理好的.md> --attach <fileId>`——
   引用即收件箱自动完结（status → processed，产出节点自动回填，来源可反查）。
5. **收尾**：`crab wiki link` 回写相关页 → `crab wiki lint`。
   确认不入库的垃圾文件 `crab wiki inbox skip <fileId> --note <原因>`。

## 单份材料（用户直接给文本/markdown）

1. **读原文**，提炼结构：这份材料讲什么？分几个主题？哪些值得留？
2. **与用户确认**：挂到哪个目录、文档怎么拆（一个主题一个节点）、有没有
   不该入库的内容。
3. **找挂载点**：`crab wiki read root` → 逐层读目录索引。没有合适目录就
   先 `crab wiki add-dir`（目录也是节点，同样要有摘要）。
4. **写入**：`crab wiki add --parent <id|路径> --title <标题> --summary <一行摘要>
   --content-file <整理好的.md>`。正文用 markdown（标题/列表/表格/代码）；
   首行不要重复 `# 大标题`（标题已在页眉）。
5. **回链**：`crab wiki link <新文档> --to <相关文档>`（双向信息用两条 link）。
   相关页也可以补一条反向 link。
6. **收尾**：`crab wiki lint`；有 stale/空目录顺手处理。

## 多份材料 / 现成目录

材料已是一组 markdown 文件（比如另一次导出）：

1. 先摆目录镜像给用户确认：`crab wiki import <目录> --dry-run`
   （子目录 → 目录，`.md` → 文档；标题取 frontmatter `title:` 或文件名；
   摘要取 frontmatter `summary:` 或首段）。
2. 目录默认描述：各子目录放一个 `README.md`，首段会作为该目录的「前言」
   显示在它的索引顶部。
3. 确认后去掉 `--dry-run` 真跑。**同名冲突会整条失败**（wiki.titleTaken）——
   二次导入同一目录前先改标题或清理旧节点。

## 去重与归并

- 入库前先查重：`crab wiki read <疑似已有主题>`——已有文档就「增补」而非
  新建（`crab wiki edit --content-file`，注意乐观锁：冲突时重读再改）。
- 同主题多来源：并成一个文档，来源差异写进正文「来源与差异」小节；
  不要一来源一节点。

## 与用户确认的口径

- 拆分粒度：默认「一个主题一个节点」；用户没意见就不要一次问拆分细节。
- 摘要写法：一句话说清「这是什么 + 什么时候读它」，≤200 字，无空词。
- 敏感内容：词条标题/摘要成员全库可见（内容才按授权锁），所以仍要确认
  「这份要进库吗、给谁看」——标题里也别写敏感细节；入库后用
  `crab wiki grant` 圈人（目录授权 = 整棵子树）。
