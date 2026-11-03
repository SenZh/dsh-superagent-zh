# 002 授权

授权决定谁可以开始工作以及工作可包含什么。本文档基于 [001 领域模型](001-domain.md)。

宿主强制执行允许名单、规则与 Mandate。消息、评论与引用文档是决策输入；不能改变这些权限。

## Trigger 的两条路径

### Eval：决定是否开始工作

事件源记录消息是否指向智能体：`mention` 表示 @ bot 或 agent 账号，`direct` 表示私信。gate 检查发送者与事件源的 Policy：

- 允许名单中的人员 @ 智能体构成 explicit 请求。
- 人员在已有工作项的 Thread 上的跟进遵循下列规则。
- 其他人员消息对其接收 scope 使用 admission 表。
- 事件源 bot 列表上的 bot 可产生 inferred 候选，永不构成 explicit 请求。其他 bot 不准入。

### Subscription：延续已有工作

已订阅 Thread 上的每条消息作为上下文到达其订阅工作项，含 bot 与 CI 消息。该投递 bypass gate，不授予新权限。每个工作项仍受 Mandate 限制。

人员在工作项自有 Thread 上的消息由该工作项 Executor 决定。主 Session 将消息挂到工作项时同样适用。Executor 拥有该消息期间不走 Eval、无 Decision。各 Executor 在 work item scope 内处理请求，或交回以便别处路由。

只要工作项仍在 live，所有权持续，即使人员移除了 Subscription。也覆盖人员未 stop、且消息等待恢复它的已结束工作项。

主 Session 收到跟进通知。Executor 拥有消息期间，主 Session 仅可在人员请求时 stop 工作；accept、attach、ignore 与 requeue 被拒绝。stop 仍经 gate，使用下文 follow-up strength。

### 交回与释放跟进

Executor 在请求超出工作项 scope 或属于另一工作项时交回消息一次。工作项保留 Subscription。若挂接 Thread 无关，Executor 交回其在该处拥有的消息。

仅当没有工作项再拥有该消息时，消息进入新的 Eval。其候选包含 Executor 的理由。gate 将其作为该 Thread 上的普通消息判断，主 Session 可 accept、挂到别处或 ignore。交回工作项未拥有的消息（如 bot 消息或人员已送 Eval 的消息）不入队。

工作项结束时，释放 Executor 未读或未接管的跟进。包括从未送达 Executor 的消息，以及在最后一次 Report 或 turn 结束之后送达、且既未确认、未回答也未交回的消息。每条释放消息在无其他工作项拥有时进入 Eval。stop 仅结束被 stop 工作项的工作。

被 stop 的工作项不会再次将其 stop 消息送 Eval，除非另一工作项交回。stop 它的主 Session 可将消息其余部分 accept 为新工作。

人员可从面板将拥有的跟进送 Eval。主 Session 再挂回 live 工作项时，Executor 再次收到。面板在无有效 judgement 覆盖消息时也提供重新评估。

### 仅上下文 Subscription

Executor 可跟进相关 Thread（如另一 Issue 或 PR）作上下文。其消息到达工作项但不授予权限或 follow-up strength。该处人员消息不会恢复已结束工作项，除非主 Session 挂接该 Thread 上的消息。主 Session 关于其他工作的 note 也仅作上下文。

未到达任何工作项的人员消息（含被人员 stop 的工作项所在 Thread）走 Eval 路径。bot 消息无论是否投递到订阅工作项，独立走该路径。

## Strength

| Strength | 情形 |
| --- | --- |
| explicit | 名单中人员 @ 智能体，或在已有工作项的 Thread 上跟进；admission 表单元格设为 explicit 请求 |
| inferred | 非名单人员在已有工作项 Thread 上跟进（除非其 admission 单元格为 explicit）；admission 单元格设为 inferred 候选；事件源 bot 列表中 bot 的消息 @ 智能体或到达事件源接收全部消息的 scope |
| not admitted | 任何其他 bot 消息、admission 表 ignore 的一切，以及 accept 模式为 `off` 时所有 inferred 情形 |

对未列出人员，explicit admission 单元格优先，即使在有工作项的 Thread 上，且在任意 accept 模式生效。否则其跟进为 inferred，在 `off` 模式下不准入。仅作上下文跟进的 Thread 使用 admission 表而非 follow-up strength。

## Admission

各事件源 Policy 有 admission 表。行是接收 scope：

- **Feishu：** 各配置群聊、bot 收到 @ 的其他群聊，以及私信。其他群聊仅在飞书卡片将 **其他群** 设为 **仅 @** 时接收。
- **GitHub：** 新 Issue、新 PR、评论及默认分支 CI。

一行最多三格。每格决定消息成为 explicit 请求、inferred 候选或无：

| Cell | 选项 | 默认 |
| --- | --- | --- |
| 在名单上，未 @ 智能体 | inferred, ignore | inferred |
| 不在名单上，@ 智能体 | explicit, inferred, ignore | ignore |
| 不在名单上，未 @ 智能体 | inferred, ignore | ignore |

未 @ 消息的单元格仅在事件源接收全部消息处可用，如设为 **全部消息** 的飞书群或开启 receive-all 的 GitHub scope。CI 结果无发送者，其行仅提供最后一格。bot 消息使用 bot 列表而非这些单元格。

将未列出且 @ 的单元格设为 explicit 允许该 scope 内任何人 @ 智能体请求工作。对专用支持群有用。inferred 候选仍依赖 accept 模式。跟进通常 bypass 此表，但 explicit 单元格仍授予 explicit strength。

admission 表是 Policy 版本的一部分。弱化单元格不会立即停止在其下 accept 的 open 工作项。恢复前 revalidation 时可撤销 Mandate。

### Bot 与自动化 reviewer

宿主将 GitHub Apps、以 `[bot]` 结尾的 login 与飞书 bot 识别为 bot。事件源可列出如 `dependabot[bot]` 等 bot，其消息可成为 inferred 候选。它们须 @ 智能体或发布在接收全部消息的 scope。不能发 explicit 请求或 stop 工作项。

使用普通账号的自动化 reviewer 对宿主算人员。其评论到达订阅工作项并产生跟进通知。团队 prompt 须将这些账号标为自动化 reviewer。

## 路由候选

主 Session 用 `orchestrator_route` 路由 explicit 候选。一次调用可含多个 accept、attach、stop 或 ignore。各项独立处理：拒绝项或缺字段不阻止其他项运行。

inferred 候选通常需完整 assessment。一种快捷：若卡片明显不涉及任何启用规则（如闲聊或无关讨论），主 Session 可直接 ignore。Decision 记录 `ignoredFromCard` 以区别于规则 assessment。主 Session 在其 Policy prompt 段阅读启用规则。

accept inferred 候选始终需 assessment。direct accept 拒绝 gate 未准入为 explicit 的 Trigger。

## Accept 模式与规则

accept 模式是 Policy 的一部分：

| Mode | 效果 |
| --- | --- |
| `off` | 无 inferred 成为候选，如未 @ 消息、bot 消息或非名单人员跟进 |
| `observe`（默认） | 评估 inferred 候选；positive assessment 记为 ignore 并标 “would accept”，不 accept |
| `auto` | inferred 候选 positive assessment 则 accept |

explicit 请求在任意模式生效。先在 `observe` 启动，审阅记录的 Decision 后切到 `auto`。每次 assessment 对每个启用规则恰好判断一次，无论是否匹配。proactive accept 至少需一条匹配规则。

每条规则描述一类 proactive 工作。工作项 Mandate 记录匹配规则，其 Executor 收到规则名与 instruction。

对该工作项，规则的 process 与 communication 要求覆盖默认。silence 规则可 suppress 确认；review 规则可指定 merge 前所需 reviewer 与批准。适用于每个工作项的要求应放在团队 prompt 段。

## Policy 版本

规则、accept 模式与各事件源 Policy 构成 Policy 版本。每个 Mandate 记录 accept 时的版本。

版本变更丢弃进行中的 Eval 并重新入队其 Trigger。也向每个受影响 live 工作项发送 policy-changed Input。Input 说明：

- 匹配规则变更（含新文本）或不再适用；
- 请求者可能不再能开始工作；或
- 不再 accept proactive 工作（如模式从 `auto` 变更）。

未受影响工作项无通知。变更不会立即撤销 live 工作项 Mandate。

事件源配置与 Policy 分离。控制进入系统的内容：读哪些聊天、scope 接收全部消息或仅 @。Mandate 记录其事件源与接收 scope。恢复需事件源仍监听 Thread，但不要求仍接收该处全部消息。

事件源关闭时回复等待。事件源不再监听目标时丢弃回复。

关于 Executor 对自身结果可做什么的条件（如何时可 merge）不决定谁可授权工作。属于智能体 instruction，非 Policy 部分。

## 恢复与 stop

已结束工作项保留 Mandate。人员在其自有 Thread 上的消息可在无新 Eval 或 Decision 下恢复同一工作，不论发送者。bot 与 CI 不唤醒已结束工作项。Executor 判断请求是否 fit scope，否则结束本轮。

仅三种操作改变 Mandate：accept 创建；人员 stop 撤回；revalidation 失败撤销。人员 stop 的工作项保持 ended 且不再接收。该处请求的工作需新 accept。

### 恢复前 revalidation

事件源仍须监听原始 Thread。其余检查依赖 Mandate 依据：

- explicit follow-up Mandate 通过。
- inferred Mandate 需要 `auto` 模式且匹配规则仍启用。
- admission 表单元格准入的 Mandate 须在当前表与发送者成员关系下仍至少获得原 strength。从名单移除发送者可因此撤销经 listed-sender 单元格 accept 的 Mandate。
- bot 的 Mandate 要求 bot 仍在事件源 bot 列表。inferred 表与 bot Mandate 还需 `auto` 与启用匹配规则。
- 其他 Mandate 要求发送者仍在事件源允许名单。

revalidation 失败则工作项保持 ended 且不取任何等待消息。每条消息视为未到达任何工作项；若无其他工作项拥有则进入 Eval。主 Session 不会被告知到达 worker。处理 pause 后同样适用。挂到该工作项被拒绝，进一步工作需新 accept。

- **人员在 Thread 中说 “stop” 是普通 Trigger，** Eval 决定 stop。Executor 不能声明人员要求其 stop。
- **面板 stop 是指令。** 直接作用于工作项，记为 manual stop。
- **等待条件属于事件源。** 如 “N 小时无 assignee” 应是自行等待并再查的事件源；Eval 从不等待。

## Eval 未完成时

时限从候选到达主 Session 起算。等待投递的时间不计，未投递候选不会过期。

过期 Eval 将其 Trigger 重新入队再试。连续两次重试后，第三次过期记录 `unavailable`。assessment 失败也记录 `unavailable`；后续 Trigger 可启动新 Eval。Policy 变更或对过时 Policy 的 decision 记录 `unavailable` 并立即在当前版本下重新入队该批。

`evals` 健康项报告过去一小时内未解决的 unavailable Decision。`scripts/dsh-superagent status` 包含该错误。

面板仅在 unavailable Decision 不足 24 小时、仍是 Thread 上最新 Decision、未 dismiss、且 Thread 既未 queued 也未 evaluating 时请求人工关注。人员可再次评估 Trigger 或 dismiss 问题。
