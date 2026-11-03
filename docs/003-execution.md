# 003 执行

本文档跟踪已 accept 的工作项经执行、投递与恢复的全过程。基于 [001 领域模型](001-domain.md) 与 [002 授权](002-authorization.md)。

## 工作项阶段

每个 open 工作项有一个阶段，由等待什么决定。自上而下读表；首条匹配行生效。

| Stage | Condition | Panel label |
| --- | --- | --- |
| person / delivery | 向 Executor Session 投递 Input 失败；2 秒后重试，随后延迟加倍至最多一分钟；成功、新 Assignment 或工作项结束可清除 | Input undelivered |
| person / answer | 有 Attention | Awaiting answer |
| person / review | 最新 Report 为 `ready` | Awaiting review |
| person / blocked, failed | 最新 Report 为 `blocked` 或 `failed`，且之后无人员消息到达工作项 | Blocked · Failed |
| working / queued | 无 Assignment 或 pending | Queued |
| idle | 最新 Report 为 `waiting`，或 Assignment 已结束 | Waiting for a Trigger |
| working | 其余 | Working |

`person` 阶段表示无人工输入工作项无法继续。新人员消息将 `blocked` 或 `failed` 工作项释放给 Executor，故进入下一条匹配阶段，通常为 `working`。

`idle` 工作项在本轮结束且无未决问题。已订阅 Thread 上的 Trigger 可继续它。

## Executor

每个工作项获得名为 `worker-<ticket id>` 的 Executor Session 及以工作项短名命名的目录。目录创建于配置的执行根下，默认 `<stateDir>/work`。

Executor Session 以操作者 OS 用户身份无审批运行命令。其 `dsh-superagent-worker` preset 含工作所需工具，5 秒后将前台 shell 命令移到后台 job。这样 worker 下一步可收新消息，而不必等长命令结束。

主 Session 使用 `dsh-superagent-router`，同样 5 秒 shell 与 job-wait 限制，工具集更小用于路由。Session 保留创建时选的 preset。Worker 子智能体使用 DeepSeek Harness 的 `standard` preset。

### 后台工作与 worker 槽位

turn 结束时仍有 shell job 或 subagent 运行，工作项保留 worker 槽位并保持 working。不报告为 waiting，主 Session 也不收 waiting report。完成的 job 或 subagent 在同一槽位用结果唤醒 Executor。turn 结束且无后台工作后，Executor 正常报告 waiting。

每 turn 后，后台工作最多 30 分钟须完成。到限 shell job 被 kill，工作项报告 waiting。report 含 Executor 最后话语及被 kill 的 job 或仍活跃的 subagent。subagent 允许继续。Executor 下一 turn 读取被 kill job 的结果。

交给人员立即释放槽位，Report 仍有效。适用于 `ready` report 或提问后（除非此后有新 Input），以及任何 `blocked` 或 `failed` report 后。

handoff 时 subagent 停止。shell job（如为 review 提供的服务器）可继续至 Assignment 结束。不占槽位，免 30 分钟限制，含后续 turn 之后。完成不唤醒 Executor；下一 turn 读结果。

人员也可直接在 Executor Session 写以启动 turn。其后台工作遵循相同规则。

Assignment 结束时其 shell job 被 kill 且不唤醒 Session。subagent 释放，任何 completion turn 被丢弃。替换 Assignment 及其等待 Input 在该 shutdown 完成后启动。

在 DeepSeek Harness 进程内 stop 或 restart 插件走相同 shutdown 路径。尚未进入 Session history 的 Input 保留。shutdown 最多等每个 worker 后台工作 10 秒；进程 restart 直接结束该工作。busy 时被中断的 Assignment 以 recovery note 重启。

Worker subagent（含嵌套）使用 DeepSeek Harness `standard` preset 与 worker 权限。仅保留 worker 允许的工作工具，工作项结束时失去。`standard` 下 shell 命令 60 秒后移后台。subagent 结束时其 job 被 kill。

### 准备工作项目录

首次 Assignment 前，若工作项目录不存在，prepare 命令在 execution root 用 `sh` 运行一次。最多十分钟。无 prepare 命令时 worker 在空目录启动。

失败命令使工作项失败并记录输出。retry 或人员在其 Thread 上消息会再次尝试 prepare。典型命令用 `git worktree` 创建 checkout。PR 在其分支路径最后段与工作项短名匹配时属于该工作项。

### 并发

工作项从 launch 开始（含 prepare）或从唤醒 idle Executor 的 delivery 起持有 worker 槽位。活跃 turn 期间也持有。

配置的并发限制适用于这些槽位，另加一槽给人员消息或 status 请求。正常槽满时此类唤醒可用额外槽，并在新 launch 前取下一个空闲槽。新工作因此不能阻止人员继续已有工作项。

其他 Input（如 CI 结果）等正常槽。launch 在 `working / queued` 等到槽空闲。launch 并发运行，慢 prepare 仅延迟其自身工作项。

### 执行变量

prepare 命令与 Executor instruction 收到相同 execution variables。命令读环境变量；Executor 在 prompt 读 value 与 description。均不含凭证。

| Variable | Registered by | Value |
| --- | --- | --- |
| `DSH_SUPERAGENT_DIR`, `DSH_SUPERAGENT_NAME`, `DSH_SUPERAGENT_TICKET` | Executor | 工作项目录、短名与 id |
| `DSH_SUPERAGENT_BLOCKED_COMMANDS` | Executor | Executor 不得运行的命令 |
| `DSH_SUPERAGENT_GITHUB_REPOSITORY` | GitHub | 事件源监视的仓库 `owner/name` |
| `DSH_SUPERAGENT_GITHUB_ACCOUNT`, `DSH_SUPERAGENT_GITHUB_COMMIT_EMAIL` | GitHub | agent 账号及其 commit email |
| `DSH_SUPERAGENT_GITHUB_NUMBER`, `DSH_SUPERAGENT_GITHUB_PR_HEAD` | GitHub | 启动工作项的 Issue 或 PR，为 PR 时为其 head ref |
| `DSH_SUPERAGENT_FEISHU_BOT`, `DSH_SUPERAGENT_FEISHU_CHAT` | Feishu | bot 及启动工作项的聊天 |

工作项特定值来自其目录与 originating Trigger。其他值来自当前部署设置。仅面板可改 execution environment 设置；发给主 Session 的文本不能改配置的 shell 命令。

Worker 经 agent 账号的 `gh` 访问 GitHub，经 bot 的 `lark-cli` 访问飞书。blocked 命令按名调用时失败。

这些 wrapper 与目录建立默认，非安全边界。worker 仍可用绝对路径执行程序。

## Merge

Auto-merge 由 **Delivery requirements** prompt 管辖。宿主无 merge 专用决策逻辑。

默认 Executor 不以 agent 账号启用 auto-merge：GitHub 将 merge 归因于启用者。Delivery requirements 须指定另一账号 CLI 用于 merge。无则人员 merge PR。

默认 instruction 仅当 PR 为工作项打开、ready、所需批准覆盖 head commit、增删合计少于 200 行时允许 auto-merge。推送新 revision 前 Executor 撤回 auto-merge。

无 body 的 review（如 approve）也作为 Trigger 到达，可唤醒等待 review 的 Executor。这些 instruction 管辖智能体，非 GitHub merge 规则。要强制批准，在 branch protection 要求并在 stale 时 dismiss。省略 merge CLI 或删除 auto-merge 段则要求手动 merge。

## 工作项如何结束

- **无 PR 的 ready：** `ready` Report 以 completed 结束工作项。
- **无事可做：** Executor 发现无人员可行动的工作（如无法复现或被取代的请求）报告 `dropped`。工作项无消息结束，除非匹配规则要求；人员稍后在其 Thread 消息可恢复。
### Pull requests

工作项在其 Executor 报告后跟进其起始 PR。也从其分支打开的每个 PR 在 Executor 报告或 GitHub 事件源发现后跟进。reopen 的 PR 再次跟进。

当其打开的 PR 均不再 open 时工作项完成。若未打开任何 PR，其跟进的所有 PR 须 closed。然后以最后相关 close 的结果结束并保留 Subscription，允许后续人员评论恢复。

恢复、重试或恢复的工作未完成时 close 不结束工作项。若 Executor 已确认或回答其拥有的跟进但此后未做 non-progress Report 或未在读该消息后结束 turn，也等待。后续 `ready` Report 在相同 PR 完成规则满足后结束工作项。

Chat Channel 收到关于 PR 结果的一条最终通知。GitHub 已在 PR 上显示结果。移除工作项对 PR 的 Subscription 也停止跟进该 PR。

因此 “Awaiting review” 指带 PR 的 `ready` Report。

### 跟进与上下文 Thread

Executor 决定其工作项自有 Thread 上的人员跟进。处理 scope 内请求，交回超出 scope 或属另一工作项的请求。无 Executor 再拥有消息后，成为主 Session 候选。主 Session 否则仅可在人员请求时 stop 工作。

Input 在 Session 将其纳入 step 并随后 report 或 end turn 后计为已读。工作项结束时，未读且未确认、未回答或未交回的消息回到 Eval，见 [002 授权](002-authorization.md)。已读但未行动的消息（如感谢）不回到 Eval。

仅上下文 Thread 仍投递消息，但不赋予工作项对人员跟进的所有权，不恢复已结束工作项。那些消息可独立 assessment。

live 工作项订阅时事件源轮询已订阅 Thread，并在最后 Trigger 与订阅工作项结束中较晚者之后两周内继续。之后消息仅经普通 intake 到达：GitHub 通知或飞书实时事件。

## 失败与恢复

### Executor blocked 或 failed

Executor Session 报告 `blocked` 或 `failed`，或启动失败或 error（记 `failed`）时，工作项不自动收 Input。

模型 provider 或网络的 passing failure、HTTP 408/429/5xx 或 connection refused/reset/timeout 先重试三次递增 backoff：launch 再跑，turn 失败的 worker 被告知继续；失败的 prepare 命令不重试。

一旦 failed，CI 与 bot 消息记录在其上但不投递、不唤醒 Session。

工具仍可用，人员可直接在 Session 继续。

失败的 launch 不会自行再试。

人员消息、后续非 `blocked`/`failed` 的 Report 或 retry 结束 hold。

由存储的 Report 与 Input 推导，restart 后保持。
### Retry

open 工作项处于 `person / delivery`、`person / blocked` 或 `person / failed` 时可用：新 Assignment 再次启动 worker 并取等待 Input，分别为各情形补救。

人员消息到达 blocked 或 failed 工作项后 Executor 已有消息，工作项显示 working，或 failed launch 再启时为 queued，不提供 retry，除非投递该消息失败。

retry 结束当前 Assignment 并在同一 Executor Session、目录、分支与 PR 启动新 Assignment。

Executor 模型仅在 Session 创建时应用；已有 Session 保留其模型。
### Restart

运行中 Assignment 以 “host restarted” 结束，新的以 recovery note 启动，指示 Executor 在重复任何外部动作前检查目录、git history 与 GitHub，最旧工作项优先。

waiting、ready、blocked 或 failed 的 Assignment 保持，除非该 Report 之后有 Input 到达，则也再启动。

Session、目录、Subscription、pending 消息及下述存储工作 survive restart；每个 Executor Session 纳入 history 的消息记录也 survive：restart 切断的 turn 所 take 的消息在 Report 或 turn 结束后计已读；step 已 claim 但 cancel 或 error 在 history 前切断的计未读。

retry 或 restart 给新 Assignment 再次送达被替换 Assignment 所收但 Executor Session 未 take 的消息（cancel 或 cut-off step 可能丢弃）；Session 中仍等待的副本（restart 保留或 crash 持久化）在新 Assignment 首条消息前取出，故新 delivery 为唯一副本。
### 存储工作

dsh-superagent 在 decision 之后做的事（如 relay 候选、跟进或 report 到主 Session、launch Assignment 或 deliver Input）先存储后执行。

relay 或 delivery 失败（如 Session 不可达）2 秒后重试，随后延迟加倍至最多一分钟；launch 失败使工作项失败，除非为上述 passing failure。

每类工作仅在一项连续失败三次后在 overview 系统问题中报告错误；失败的 Input delivery 显示在工作项上。

到主 Session 的 relay 保持顺序，通知亦然；失败项阻塞同 kind 的后续项。
### 事件源失败

GitHub 事件源分别读每个 Issue、PR 与 Thread：失败读保留 error 于 Thread，下次 poll 再读，计入事件源 error；默认连续三次 failed poll 后报告；任一次成功 poll 重置计数。

GitHub 对私有仓库 agent 失去访问也返回 404，故 404 会重试。

通知的 Issue 或 PR 若读持续失败，首次失败读一日后丢弃，GitHub 报告 deleted 时立即丢弃；不再使 poll 失败，Thread 保留 error，事件源 state 命名它。

读取位置不变，故后续通知或 poll 其 Thread 可读失败读遗漏的内容。

飞书 event consumer 报告每次失败；已订阅主题轮询将每次失败保留于 Thread。

GitHub 列表或飞书页中 malformed 项跳过，事件源 state 计数。
### Channel 失败

外发消息存储并按存储工作发送，自 5 秒 backoff 重试，加倍至 300。

工作项发往同一会话的消息按排队顺序发出，故确认先于后续回答或结果：同工作项同会话较早消息未发出前消息等待，直到该条连续失败三次并显示为问题。

其他工作项消息及无工作项消息不等它。

有失败尝试的消息为系统问题，直至发出或 withdraw。

仅当事件源不再监听会话，或事件源回答会话永不可收时丢弃消息；事件源 off 时等待。

GitHub 上对 deleted Issue 或 locked 会话，飞书上对 recalled 或飞书不再找到的消息，事件源如此回答。

对已不存在 review comment 的回复改发到 PR 会话。

dropped 消息保留记录（何时为何 drop 及所属工作项），不再入队；dropped 确认显示为 dropped 而非 sent。
### Pause processing

停止 dispatch、新 Eval、Eval 超时、Input delivery 与回复；已在进行的 launch 等到 processing resume。

事件源继续记录 Trigger，运行中 Executor turn 不中断。

人员发给 ended 工作项的消息等待：pause 后首次 pass 恢复工作项（若 Mandate 当时仍成立）。

进行中的 Eval 在 pause 期间不过期；deadline 延后 pause 时长。
### 无效配置

其 kind 拒绝的存储事件源配置禁用该 Source 实例。
