# crabcloud-skills

[Crab Cloud](https://crabcloud.cc) 的 agent skills（SKILL.md）——让 Claude Code /
Codex / ZCode 等 coding agent 学会通过 `crab` CLI 操作用户的个人云。本仓库是
[skills.sh](https://www.skills.sh) 生态的收录源，内容与 npm 包
[@crabcloud/skills](https://www.npmjs.com/package/@crabcloud/skills) 同源。

## 安装

经开放生态目录（推荐）：

```bash
npx skills add niklauslu/crabcloud-skills --skill crabcloud
```

或经 npm 专门包（自动探测本机 agent，含卸载器）：

```bash
npx @crabcloud/skills install
```

要完整命令面（设备授权登录、账号信息、令牌管理等）请装 CLI：

```bash
npx @crabcloud/cli init
```

## skills 一览

| skill | 覆盖域 | 状态 |
| --- | --- | --- |
| [`crabcloud`](skills/crabcloud/SKILL.md) | 平台 / 账号（身份、订阅积分、Agent 令牌管理、设备授权） | 已发布 |
| `crabcloud-mail` 等 | 邮箱 / 云文件 / 协作项目 | 随各应用上线 |

## 同步说明

技能内容的唯一事实源在 [CrabCloud](https://github.com/niklauslu/CrabCloud)
monorepo 的 `packages/skills/skills/`（私有开发仓库），经同步脚本单向推送至本仓库；
请勿直接向本仓库提交内容改动。

## 安全模型

凭证只存在用户本机（`~/.config/crabcloud/credentials.json`，0600），服务端只存
令牌哈希、动作全程审计；scope 按需最小化授权。

## License

[MIT](LICENSE)
