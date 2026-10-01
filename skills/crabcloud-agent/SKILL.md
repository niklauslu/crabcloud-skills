---
name: crabcloud-agent
description: Check the status of the user's personal cloud Agent in Crab Cloud through the `crab agent` CLI — running sessions and tasks, and the outbound-action approval queue (pending / expired). Use this skill whenever the user asks about their in-cloud agent ("我的 agent 在干嘛", "agent 跑完了吗", "有没有待审批的操作", "看看 agent 审批队列"), wants a quick health check of their Agent workbench, or asks what the agent has been doing — even if they never say "crab" or "Crab Cloud" explicitly. This skill is read-only; approving or rejecting happens in the Agent workbench web UI (agent.crabcloud.cc), not from the CLI.
---

# 个人云 Agent（查看域）

Crab Cloud 的个人云 Agent 是一个用户一个固定 agent：长记忆、权限三档、
能读平台数据（邮箱/云盘/协作）、写动作过审批全留痕。本 skill 覆盖**只读
查看**（`crab agent …`，需 `agent.read` scope）；审批的批准/拒绝在网页
工作台完成（流内卡片），CLI 不做决策。

## 第一步：确认可用

先确认 `crab` 命令存在：`crab --help`。若命令缺失，引导安装：

```
npx @crabcloud/cli init
```

已安装但命令缺失时升级：`npx @crabcloud/cli@latest init`。所有
`crab <cmd>` 都可用 `npx @crabcloud/cli <cmd>` 等价执行。

## 命令

### 状态总览

```
crab agent status
```

输出：会话总数、运行中数量、待审批数量。`--json` 输出机器可读 JSON。

### 会话列表

```
crab agent sessions
```

列出对话与任务（标题 / 状态 / 定时 cron / 最近更新时间）。`--json` 可用。

### 审批队列

```
crab agent approvals
```

列出待处理与已超时的出站审批（工具名 / 参数摘要 / 来源会话）。
**这里只看，不批**——告诉用户：打开 Agent 工作台（agent.crabcloud.cc）→
对应会话的流内审批卡上点「批准执行 / 拒绝」；24 小时未处理的会自动失效，
且失效后即使补批也不会执行（agent 需重新发起）。

## 边界

- 只读：本 skill 的命令都不会改变 agent 状态；决策一律引导到工作台网页。
- BYOK 配置（API 端点 / Key）不在 CLI 视野——在「模型」管理页维护。
- 需要写操作走平台时，外部 agent 用既有域命令：`crab mail …` /
  `crab storage …` / `crab collab …`（各自 skill），与本 skill 互补。
