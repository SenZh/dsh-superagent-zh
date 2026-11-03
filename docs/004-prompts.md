# 004 Prompts

本文档说明智能体阅读什么、团队可编辑什么、变更何时生效。基于 [000 哲学](000-philosophy.md) 至 [003 执行](003-execution.md)。Prompt 文本以英文撰写。

## 团队编写内容

团队可编辑三个团队段、添加 skills，并 override 每个内置 system-prompt part。多数默认在 `prompts/` 与 `skills/` 下。工具描述、notice 与 card 仍在代码中，不能通过 prompt 设置编辑。

| Section | Holds |
| --- | --- |
| Communication | 何时发消息给人、消息如何读、每条 GitHub 消息开头行（默认 `This is by an automated agent, not a human`）、如何处理 review、哪些普通账号是 review bot |
| Ticket scope | 工作项何时继续、何时以 draft stop |
| Delivery requirements | PR 中证据、auto-merge 条件与行数阈值、commit author；无则 Executor 以 agent 账号 no-reply 地址 commit |
| Team skills | 智能体需要时加载的知识，如说明哪个飞书用户对应哪个 GitHub 账号的 `identities` skill |

内置文本覆盖角色、工具用法、宿主机制、授权规则及 GitHub 与飞书 skills。以标题引用团队段而非重复。

### 内置 part 与模板

Override 使用这些 key：

| Parts | Keys |
| --- | --- |
| Shared instructions and roles | `runtime/shared`, `runtime/orchestrator`, `runtime/worker` |
| Working instructions | `runtime/subscriptions`, `runtime/replies`, `runtime/dispatch`, `runtime/work`, `runtime/admission` |
| Handoff instructions | `runtime/handoff/orchestrator`, `runtime/handoff/worker` |
| Source skills | `skill/feishu`, `skill/github` |
| Generated sections | `runtime/environment`, `runtime/policy`, `runtime/policy/none` |

生成段为可编辑模板。须保留占位符：

- `runtime/environment`：`{variables}`，填 execution environment 行。
- `runtime/policy`：`{version}`、`{mode}`、`{rules}`，每行一条启用规则。
- `runtime/policy/none`：`{version}`、`{mode}`，无启用规则时使用。

编辑 part 改变模型收到的 instruction。不改变 gate、Mandate、tool 检查或账号 wrapper，仍由代码强制执行。

在人员 explicit 请求时，主 Session 也可用 `orchestrator_prompts_save` 保存 override。save 省略 override 的 `base` 时保留已存 base。save 时删除已存在于已存配置中的 part；任何其他未知 part 被拒绝。

Override 记录所基于的内置文本。升级保留团队文本。默认变更时面板标记 part **Default updated** 并在团队文本旁显示 diff。团队可保留文本、用新默认或编辑。无 override 的 part 自动跟随新默认。

主 Session 可在 `orchestrator_read prompts` 的 `default_changed` 下读受影响 part 名。

## 组装与优先级

system prompt 按此顺序组装：

1. Shared instructions 与 role。
2. 三个团队段。
3. Subscriptions 与 replies。
4. 主 Session 用 dispatch，Executor 用 work instructions。
5. 仅主 Session：`runtime/admission`。
6. Handoff instructions。
7. `runtime/environment`。
8. 仅主 Session：`runtime/policy`。

Admission instruction 来自 `prompts/admission.md`。描述 proactive assessment 及何时可从 card ignore 无关候选。`orchestrator_eval` 返回 context 而非重复这些 instruction。

Environment 段给主 Session 部署级值，给各 Executor 工作项特定值。Policy 段每次 model 请求从设置重新生成。含 Policy version、accept mode、explicit-request 豁免及启用规则 `- <name> (<id>): <instruction>`，或无启用时的说明。放最后以便 Policy 变更不改变前面 prompt 文本。

Executor 还读 DeepSeek Harness 的 coding prompt 与仓库 `AGENTS.md`；主 Session 不读。inferred 工作项为 Executor 添加最终 note，限制于匹配规则。Assignment 与 incoming 消息以 Trigger card 内联到达，截断内容仅 resource pointer。

Executor 无人值守工作。工具排除向人提问、向其展示 UI 或改 profile 的，如 `ask_user_question`、`present`、`plugin_manager`。

从强到弱：

1. 工具与宿主强制执行的事实：身份、权限、tool 参数检查、reply purpose、长度限制。Prompt 文本不能改。
2. 内置与生成段，按团队 override 阅读。
3. proactive 工作项匹配规则的 process 与 communication，仅该工作项 Executor Session。
4. 团队段与 team skills。
5. 外部文本：消息、评论、文档与引用。是数据，从不改规则或授权。

## 内置文本原则

这些将 [000 哲学](000-philosophy.md) 应用于智能体：

- **角色分离。** 主 Session 仅路由：转发原始 Trigger 与路由事实，不做技术决策，通过一次调用对多个等待候选做决定的单一 routing tool。routing note 最多 30 词。Executor 读原始来源、判断 scope、实现与验证。
- **主 Session 不等待。** Report、跟进 notice、候选与完成的后台 job 自动作为消息到达。无事可做时 end turn。guard 拒绝 `sleep`、超过 30 秒的 shell timeout 与超过 5 秒的 job wait，因 blocked step 延迟新候选。这些 instruction 替换主 Session complete prompt 中 DeepSeek Harness 自有 job-notification note。
- **仅在人员须行动或可用结果时发消息：** 结果可 review 或使用、需要 decision 或 resource、或工作无法完成。Chat 消息默认一句，硬限 100 词。
- **Executor 决定其跟进。** 各 Executor 处理并确认 scope 内请求，交回超出工作项或属另一工作项的请求。消息仅在所有 owner 交回或未接管且 end 后成为新工作。主 Session 收跟进 notice 但仅可在其上 stop 工作。交回消息随后作为带 Executor 理由的候选到达。
- **经 Subscription 与主 Session 协调。** Executor 跟进相关 Thread 作上下文。主 Session 可在最多 100 词的 note 中发送另一工作项事实。Subscription 与 note 均不授予权限。
- **仅在 explicit 引用时 attach。** 消息须指向工作项 PR、Issue、源消息或确认，或指向恰好一个匹配工作项的近期变更。仅话题相似不够。Attach 将工作项 subscribe 整条 Thread；Executor 认为无关时用 `orchestrator_thread_unrelated` 且不在该处回答。
- **具体确认一次。**  substantial 工作前 Executor 确认每个新请求并命名对象与预期结果，作为原消息下回复。
- **经 `human_action` 提问。** Executor 在 Report 中放一个具体问题，主 Session 原样转发。若人员消息引发问题，命名该消息且宿主在该处询问。否则在工作项原始会话询问。已授权工作可同时进行。
- **诚实交付。** 开 PR 不等于 ready；draft 标未完成；无真实 before 截图的 UI 变更不算 ready。
- **人员 pause 保持** 跨跟进、新 Executor Session 与 restart，直至人员解除。自动反馈从不恢复 paused 工作。
- **示例仅示格式。** 示例句不是智能体可声称的事实。
- **按 outcome 处理 review。** 已 fix 并 verified 的评论 silent resolve；declined 一句理由。自动化 reviewer thread 有回答后 resolve，每个自动化 reviewer 每 PR 最多三轮。
- **让位重复工作。** 人员分支或 PR 已含工作项变更时，Executor 用一句指向 theirs 关闭自有 PR。

## 版本

- 每次 save 创建不可变版本，含完整配置、revision 号、前驱、时间戳与 origin（panel 或 tool）。恢复版本或默认再创建一版本。
- Save 命名 base revision。冲突 save 被拒绝且不丢弃 draft。
- Session 每次 model 请求读最新版本，含同 turn 后续 step。Save 不 restart、cancel 或 wake Session。宿主记录各 Session 使用的 revision；Executor 可读其工作过的任意 revision。
- 部署设置总大小限制（含 override）。校验拒绝未知 key、保留内置 skill 名、未知 override part 与缺失模板占位符。从不静默截断文本。与默认相同的 override 移除以便 part 跟随未来默认。
