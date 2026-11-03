# 006 部署

使用本指南安装、配置并运行 dsh-superagent。涵盖账户、启动检查、升级与故障排查，在 [001 领域模型](001-domain.md) 至 [005 面板](005-panel.md) 之上展开。

## 运行内容

- **专用 DeepSeek Harness profile**，本指南中称为 `superagent`。从 Web profile 模板创建。它加载 bundle 并在 `127.0.0.1` 上提供面板。与工作用的 profile 分开：插件会改变进程级 `PATH` 与 git 设置，其进程需要独立重启与升级。
- **Agent 的 GitHub 账户。** Agent 以该账户身份评论、推送分支并打开 pull request，通过仅以该账户登录的 GitHub CLI。
- **可选：** 用于聊天请求的 Feishu bot。

插件配置保存 state directory 与 machine limits。其余一切通过面板：**Sources** 配置仓库与账户，**Policy** 配置 allowed lists，**Execution** 配置 worker 目录与命令。

### 权限与隔离

bundle 为 Executor 定义 `dsh-superagent-worker`，为主 Session 定义 `dsh-superagent-router`。它们出现在 DeepSeek Harness 预设列表中，但仅供 dsh-superagent 使用。不要为个人 Session 选择它们，也不要禁用其配置行。应用需要它们来创建与恢复 Session。

在 Web UI 中打开空白托管 Session 可能选中 profile 的默认 preset。dsh-superagent 在首轮之前恢复其角色专用 preset。

两个 preset 均以操作者 OS 用户身份在 `danger-full-access` 下运行，并从已订阅 Thread 读取消息。账户 wrapper 默认选择 Agent 凭据；它们不隔离进程。worker 可按路径执行程序并读取操作者 home 目录。若要隔离，请使用仅含 Agent 凭据的专用 OS 账户。

## 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 与 tmux
- GitHub CLI（`gh`）2.48 或更高。Source CLI 字段必须指向 `gh` 或 `lark-cli` wrapper；主机依赖其子命令与输出格式。
- github.com 上的仓库；不支持 GitHub Enterprise Server。若要为每个 Ticket 创建 checkout，需保留本地 clone，以便 prepare command 可添加 worktree。若仓库使用 Git LFS，请安装 `git-lfs`。
- 为 DeepSeek Harness 配置的 model provider。dsh-superagent 使用 profile 默认 model，除非 Eval 或 Execution 设置指定其他 model。
- Feishu：`lark-cli`（测试版本 1.0.90）及带 bot 的 Feishu 或 Lark 应用

## 账户

### Agent GitHub 账户

1. 为 Agent 创建 GitHub 用户账户并授予仓库写权限。GitHub App 不能作为 Agent，因为 dsh-superagent 读取账户的通知。
2. 为其配置仅使用自身凭据的 GitHub CLI：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   以 Agent 账户登录；最后一条命令打印其 login。dsh-superagent 以同样方式读取 login。

### Feishu bot（可选）

1. 创建带 bot 能力的自建应用。授予接收消息事件、读写聊天消息、读取聊天成员的权限，然后发布应用版本。
2. 为 bot 配置独立的 `lark-cli`，限制为 bot 身份，并添加 wrapper：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读取 app secret。使用独立配置目录：`lark-cli` 每个 app id 仅允许一个 profile，你的个人配置可能已包含该 app 的用户登录。

   主机拒绝可充当用户的 CLI。它从 bot 配置读取 app id、Feishu/Lark 品牌与 bot `open_id`。

   两种配置共享 app secret 的 keychain 条目。若仅删除 bot 配置，删除其目录即可；不要运行 `lark-cli config remove`，否则会影响共享 secret。

## 安装

在尚无 `superagent` profile 的机器上运行一次以下块：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上述固定版本为公开版；其他版本须满足 `package.json` 中的 `engines.dsh`。

第四条命令从 DeepSeek Harness Web 模板创建 `~/.dsh/profiles/superagent`。第五条安装本 bundle。设置 `DSH_HOME` 可迁移默认 `~/.dsh` 目录。

profile 创建命令仅用于新安装。若 profile 已存在，该命令会替换它并删除其 `cordis.patch.yml`（含 machine configuration）。不要用于升级。

## 机器配置

将 [examples/profile.cordis.patch.yml](../examples/profile.cordis.patch.yml) 复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`（替换该文件），并替换每个 `<placeholder>`。若使用 Feishu，还需启用其中的 `dsh-superagent-feishu` 条目。`webserver` 条目设置面板端口。

| Key | Required | Meaning |
| --- | --- | --- |
| `stateDir` | yes | 绝对路径：数据库、锁、`worker-bin` 中的 command shims、Agent 的 git 配置，以及默认情况下 `work` 中的 Ticket 目录。升级前请备份 |
| `promptMaxChars`, `admissionContextMaxPages`, `admissionContextTimeoutMs` | no | 最大已保存 prompt 配置，以及单次 assessment 在限定时间内读取的 Thread 页数 |

首次启动后，在 **Settings → Deployment** 或 DeepSeek Harness Plugin Manager 中编辑 machine limits。变更无需重启即生效。`stateDir` 不能在那里修改。无效值或未知配置键会阻止启动；日志会指出键名。

## 启动与停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 在 tmux session `dsh-superagent` 中运行 `dsh --profile superagent`（其他 profile 为 `dsh-<profile>`）。最多等待一分钟直至 runtime 就绪或报告 setup 要求。首次运行会打印缺失的配置信息。拒绝重复实例，包括本脚本外启动的实例。

`stop` 与 `restart` 请求优雅关闭，最多等待 Executor 五分钟。从不强制 kill 进程，也不触碰其他 tmux session。

`status` 打印 process id、health 与 setup 问题。其他 profile 或端口使用 `--profile` 与 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 与 `DSH_SUPERAGENT_URL`。

`status`、`start` 与 `restart` 以四种退出码之一结束：

| Code | Meaning |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 未运行、重复运行、无响应或 runtime 报错；重启可能修复 |
| 2 | 在运行，但 runtime 以外的部分（如 Source 或 Policy sync（`sync:<source id>`））报错；重启无法修复 |
| 3 | 在运行并等待 setup；重启无法修复 |

仍在启动的 Source 计为正常运行。setup 无法检查的 Feishu bot 亦然。

`status` 在 `errors` 下返回失败部分及其详情，在 `setup` 下返回 setup 要求。`start` 与 `restart` 将这些 problem 打印为行；若服务始终未起来则退出 1。仅 intentional `stop` 会暂停自动恢复。

每个 profile 在 `.artifacts/` 下存储日志与控制文件：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 属于脚本。保留最新 20 个。
- 控制文件：`control/profiles/<profile>/`，含 stop marker、lock 与 process id。

打开日志中打印的 URL，然后在侧边栏选择 **Superagent**。

### 每小时检查

在你平时使用的 DeepSeek Harness profile 中创建 hourly check，而非 `superagent`。Agent 服务宕机时它仍须运行。在该 profile 中让 Agent 使用以下 prompt 创建 hourly schedule，并填入 checkout 路径：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 保留 intentional stop。仅 `stop` 写入 marker；`start` 清除它。`restart` 不写 marker，因此 check 可恢复 shutdown 超时的 restart。

## 设置

setup 成功前不会开始轮询或工作。Overview 列出缺失要求并链接到相应设置。设置变更会再次运行 setup。若在应用外修复（如 CLI 认证），选择 **Check again**。

1. **Sources。** 设置 GitHub 仓库（`owner/name`）与 agent CLI，例如 `/home/me/.local/bin/gh-agent`。请求通常通过 agent 账户的通知到达。新 Issue 或 @ 账户的评论在下次 poll 到达；通过编辑已有 Issue body 添加 @ 不会。请发新评论。Feishu 设置 bot CLI、聊天、每聊天的 receive-all 或 mentions-only 模式，以及是否接收 direct messages。
2. **Policy。** 两条示例规则可编辑或删除。在 **On each Source** 下添加可请求工作的 GitHub 账户与 Feishu 用户；列表为空时任何请求都无法成为 Ticket。Feishu 可选择某聊天在启动与 **Sync now** 时同步列表。
3. **Execution。** directory root 与 prepare command。为每个 Ticket 提供仓库 checkout：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   将个人 CLI（如以你自己账户登录的）列为 blocked commands。
4. **Prompts。** 设置 team sections：GitHub disclosure 行、review bots、commit author 与 auto-merge 条件。

### Management API

面板使用本地 HTTP API，Agent 也可调用。`SETUP.md` 用它保存设置，服务脚本读取 `GET state`。

向 `http://127.0.0.1:<port>/orchestrator/api/` 发送请求，并带 `x-orchestrator-request: 1`。host 须为 `127.0.0.1:<port>` 或 `localhost:<port>`。缺少 header 或来自其他站点的 `Origin` 返回 403。

API 保存具有面板权限。可变更 execution environment 与 setup 账户字段，与通过主 Session 工具保存不同。

Agent 可依赖以下路由与字段。`src/app/api.ts` 中的其他路由服务于面板并随面板变更。

| Request | Answer |
| --- | --- |
| `GET state` | 各页面读取的内容：`setup`，含 `problem` 与 `feishuProblem`（各为 `null` 或带 `place`、`code`、`values`、`detail` 的 setup problem），以及 setup 解析后的 `accounts`；`health`，每 part 一条，含 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 与 `at`；`load`，Executor Session `running`、Ticket `queued` 与 `maxWorkers`；`paused`；以及 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval`、`executor`，各含 `value` 与 `revision`；`sources`：Source 实例，各含 `id`、`kind`、`installed`、block 的 `value`、`{"enabled": …, "config": {…}}` 与 `revision`、kind 的 form，以及 setup `problem`（若有）；`policyVersion`；以及 `environment`，execution environment 的 `root` 与 `variables` |
| `GET sources` | 各 Source 状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、parts 的 `health`、setup `problem`、最近一天的 `stats`、该日 `filtered` 的 Trigger，以及该日 `dropped` 的消息（最新 drop 在前），各含 `channel`、`preview`、对应 Ticket 时的 `ticket`（`id` 与 `title`）、store 有则含 `thread`，以及 `dropped.at` 与 `dropped.reason`、`channelGone` 或 `channelIgnored` |
| `PUT settings/<block>` with `{"value": …, "revision": n}` | 保存一个 block：返回 `saved`、block 新 `revision`、`policy_changed` 与 `restarting`。`value` 为整个 block，同 `GET settings`；缺失键取默认 |
| `POST executor/pause` with `{"paused": true}` or `false` | 暂停或恢复处理：返回 `paused` |
| `POST setup/check` | 通过重启 dsh-superagent 再次运行 setup：返回 `restarting: true`。setup 无待修复项时拒绝 |
| `POST policy-sync/<source id>` | 立即同步该 Source 的 allowed list：返回 `policy`，该 Source 的 part 如已保存 |
| `GET prompts`, `PUT prompts` with `{"config", "baseRevision"}` | 读取 prompt 配置：`current`（含 `revision` 与 `config`）、`defaults`、`sections`（Session 读取顺序的 team sections）、`skillName`（team skill 名称 pattern）、`builtin`（team skill 不可用的名称）、`skillPrefix`（Session 加载 skill 名称的前缀）。或保存：返回新 `revision` |
| `POST tickets/<id>/retry`, `POST tickets/<id>/stop` | 与面板相同地重试或停止 Ticket |

Health 条目按职责分组：

| Entry | Reports |
| --- | --- |
| `runtime` | 应用本身 |
| `models` | 等待 model-budget retry 的 Session |
| `evals` | 过去一小时内不可用且之后无 judgement 或 dismissal 的 Eval |
| `assessment`, `launch`, `delivery`, `relay`, `notice`, `reply` | [003 Execution](003-execution.md) 中描述的存储工作；delivery 统计等待中的工作，reply 覆盖 Channel 消息 |
| `worker` | Executor Session 事件处理；首次失败后出现 |
| `github`, `feishu`, `history` | Source 轮询、事件与历史恢复 |
| `sync:<source id>` | Allowed-list 同步 |

拒绝响应含 status 与 `error`（英文文本）：

- 400：无效值（`issues`，各含 `path`）、账户检查失败（`setup`）、固定拒绝（`code` 与 `values`），或非 JSON 或过大的 body。
- 404：不存在。
- 409：revision 过期。
- 500：其他。

blocks：

- `source:github`, `source:feishu`：`{"enabled": true, "config": {…}}`。GitHub config 含 `repository`、`cli`，以及默认关闭的 `observeIssues`、`observePulls`、`observeComments`、`observeCi`，接收仓库中每个新 Issue、新 pull request 与评论及默认分支 CI；Feishu 含 `cli`、`chats` 为 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`（或 `"all"`）、`directMessages`，以及 `otherChats` 为 `"none"` 或 `"mentions"`。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`. `bots` 列出其消息可成为 inferred candidate 的 bot。`admission` 行以 chat id、`other` 或 `direct`（Feishu），以及 `issues`、`pulls`、`comments` 或 `ci`（GitHub）为键；未设置单元格取默认（见 [002 Authorization](002-authorization.md)）。
- `eval`：`mode`、`timeoutSeconds` 与 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 与 `blockedCommands`。processing pause 为独立设置 `paused`，由 `POST executor/pause` 设置。

变更 repository 或 CLI 会先运行对应 setup check。失败返回 400 及 `setup` 详情。成功的账户变更，或 setup pending 时的任何保存，返回 `restarting: true`。应用重启期间 API 暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## Setup checks

Setup 使用 DeepSeek Harness subprocess service，与 worker shell 相同。它移除名称含 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

Setup 验证 GitHub 仓库与 agent 的 push 访问，写入 worker environment，并检查 git 对 github.com 仅使用 agent 凭据。Feishu setup 失败仅禁用 Feishu Source。

Setup 写入的进程 environment：

- **Command shims** 位于 `<stateDir>/worker-bin`，在 `PATH` 最前：`gh` 运行 agent CLI；`lark-cli` 运行 bot CLI；blocked commands 失败。shim 运行目标前从 `PATH` 移除自身目录。
- **git configuration** 位于 `<stateDir>/worker-gitconfig`：替换操作者 global 与 system git 配置。它将 agent CLI 选为 github.com credential helper，将 GitHub SSH 地址重写为 HTTPS，并定义 Git LFS filters。个人 git 设置（如 HTTP proxy）不适用。此处不设置 commit identity，git 从不 prompt 凭据，并移除 `GH_TOKEN` 与 `GITHUB_TOKEN`。

dsh-superagent 未写入的 `worker-bin` 或 `worker-gitconfig` 文件会阻止 setup。git 仓库内且自身配置添加凭据的 Ticket 目录会导致 Ticket 失败。

## 试用

从 allowed GitHub 账户打开 Issue 或发含 `@<agent login> <request>` 的评论。在一个 polling interval 内（默认一分钟），Overview 应显示 Ticket。Agent 确认请求，在自有 Session 与目录中工作，并在原会话中交付。Feishu 在已配置聊天中 @ bot。

## 升级

1. 等待没有 Executor 处于 turn 中途。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 运行 `git pull && pnpm install && pnpm build`。
4. 运行 `scripts/dsh-superagent restart` 并检查 status。

Restart 结束运行中的 Assignment 并以 recovery notes 启动替代项。Team 文本与 overridden prompt parts 保留；无 override 的部分跟随新默认。升级后审查标记 **Default updated** 的部分。

较新版本迁移的数据库无法被较旧版本打开。因此回滚需要升级前备份。升级 DeepSeek Harness 本身后，重启运行中的实例以使面板加载匹配的 artifacts。

### Agent presets

worker 与 router preset 在 5 秒后将 shell 命令放到后台。Session 没有 preset 无法加载。在其 Session 使用中保留 `cordis.patch.yml` 中的 `dsh-superagent-worker` 与 `dsh-superagent-router` 行；否则 resume 失败并返回 `agent-preset/not-found`。

## 故障排查

Setup 问题在 overview 显示并链接到修复项，使用面板语言；日志与 API 显示下方英文文本。

| Symptom | Cause and fix |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片设置 agent CLI |
| `The agent GitHub CLI was not found.` | 设置 wrapper 绝对路径，或 `PATH` 上的命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 登录 wrapper |
| `@… cannot read …` or `@… cannot push to …` | 授予 agent 账户仓库写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片设置仓库 `owner/name` |
| A Ticket fails with `The prepare command failed: …` | 输出指明原因，如错误 clone 路径；修复命令后重试 Ticket |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高 |
| A Ticket fails with `The git configuration of the repository in … sets …` | 从 clone 的 `.git/config` 移除 named credential 或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 将 named 文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或选择其他 `stateDir` |
| `The bot lark-cli allows user identities` | 将 Feishu Source 指向仅 bot 配置，见上文 Feishu bot |
| `The bot lark-cli does not work.` or `The bot identity of app … is …` | 检查配置的 app id 与 secret |
| `The Feishu bot is not a member of the chat.` | 在聊天设置中将 bot 加入聊天；下次尝试成功 |
| `status` exits 2 | 某 part 报错，重启无法修复：`errors` 列出各 part（见 Management API）及 detail；part 恢复工作后清除其条目 |
| `status` exits 3 | Setup 等待：`setup` 说明缺失项，overview 链接到修复设置 |
| `status` exits 1 | 服务未运行、重复运行、无响应或 runtime 报错；输出与最新日志说明原因 |
| `… timed out after 90 s`, `… was stopped by …`, `… exited with code …` or `… could not run: …` | CLI 失败且未打印原因；以操作者用户手动运行 named 程序查看其消息 |
| The panel says dsh-superagent is not answering | 正在重启，或配置导致未启动；DeepSeek Harness 日志指明原因 |
| `dsh-superagent configuration has keys it does not use` (log) | 移除 named 键；消息说明各值现处位置 |
| The log says the plugin is incompatible with the running dsh | 安装 `engines.dsh` 范围内的 DeepSeek Harness 版本 |
| The panel shows "Failed to load plugins" after a DeepSeek Harness upgrade | 重启运行中的实例 |
| Nothing happens after a mention | 检查作者是否在 Source allowed list、Source 是否开启、处理是否未 pause。GitHub 上通过编辑 Issue body 添加 @ 不会收到；在评论中 @ agent |
