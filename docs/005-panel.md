# 005 面板

在 DeepSeek Harness Web 侧边栏中打开 **Superagent**，用于监督 Agent。面板优先展示需要人工介入的工作，其次是系统健康与历史记录。本文档在 [000 理念](000-philosophy.md) 至 [004 提示词](004-prompts.md) 之上展开。

## 三个问题

打开面板时，按以下顺序回答三个问题：

1. 现在是否有需要我处理的事项？
2. 系统是否健康、是否在执行正确的工作？
3. 某一项工作从 Trigger 经 Eval 到 Ticket 的完整历史是什么？

## 标签页

标签页沿工作流排列：**Overview**、**Source**、**Trigger**、**Eval**、**Execution** 与 **Settings**。在仓库与账户配置成功之前，Overview 会显示缺失项并链接到相关设置。

| Tab | 展示内容 |
| --- | --- |
| Overview | 需要处理的 Ticket、系统问题、过去 24 小时的活动计数以及 Executor 负载。每个区块链接到其详情页 |
| Source | Source 状态、配置问题与非零计数。已过滤的 Trigger 与丢弃的消息与 Overview 使用相同的 24 小时窗口 |
| Trigger | 按 Subscribed、Evaluating、New Triggers in 24 h 或 All Threads 筛选的 Thread。每个 Thread 显示其 Trigger（最新在前）、Decision 与 Subscription |
| Eval | 带原因与匹配规则的 Decision |
| Execution | Executor 状态、Pause processing，以及按 Open、Pending、Working、Waiting for a Trigger 或 Ended 筛选的 Ticket。Ticket 详情展示 actions、authorization、等待中的 Input、Subscription 与 Assignment |
| Settings | Sources、Eval、Execution、Prompts 与 Deployment。每组包含其保存历史 |

### 阅读单条记录

被丢弃的消息链接到其 Ticket；若无 Ticket 则链接到其 Thread。Source 页面说明其会话是否消失，或 Source 是否已停止监听该会话。

仍由 Executor 持有的人工跟进标记为 **Delivered to Ticket**，不经过 Eval。Thread 还会显示交还原因以及仅作上下文的 Subscription。**Evaluate again** 可将该跟进送入 Eval，或重试没有有效判定的 Trigger：被其 Source 过滤的、Eval 不可用的，或已投递但既无 non-stop Decision 也无 Executor 确认的消息。

Ticket 详情展示 originating Trigger、Decision 与原因、scope 与 Policy 版本。Assignment 与次要详情折叠在即时操作下方。**Retry** 可用于失败的 Input 投递，或 Report 被阻塞或失败后等待人工的 Ticket。已结束的 Ticket 显示结果，例如 **Pull request merged** 或 **Stopped by a person**。

## 设置分组

- **Sources：** 每个 Source 一张卡片，含开关、字段与已验证的账户信息。GitHub 还提供新 Issue、pull request、评论与默认分支 CI 的 receive-all 开关。轮询与读取限制在 Advanced 下。
- **Eval：** Policy 版本与规则，随后是 **On each Source** 与 Eval 设置。每个 Source 标签页包含 allowed list、membership sync、admission 表与 bot 列表。Sync 在启动时及 **Sync now** 时运行。admission 表区分接收范围、发送者成员资格以及消息是否 @ Agent。Eval 设置选择 accept mode、timeout 与 assessment model。
- **Execution：** worker model、concurrency、directory root、prepare command 与 blocked commands。点击可用变量会将其插入 prepare command。
- **Prompts：** **Edit** 修改 team sections、built-in parts 与 skills。**Full prompt** 展示组装后的文本，并与已保存文本或默认值对比。**History** 列出版本。已编辑的 built-in parts 标记为已更改并可重置；模板列出其必填占位符。

  升级导致被覆盖的默认值变更时，会标记 **Default updated**（默认内容已更新）。面板在团队文本上方对比新旧默认值。若未记录旧默认值，则对比团队文本与新默认值。选择 **Keep my text** 或 **Use the new default**。已移除部分的覆盖项列在标签栏下方，下次保存时丢弃。
- **Deployment：** 启动问题及 **Check again**、state directory 与 machine limits。已保存的限制无需重启即生效。DeepSeek Harness Plugin Manager 暴露同一页面。

## 搜索

页眉搜索按文本、id、Feishu message id 或存储的链接（例如 GitHub comment URL）查找 Trigger。原始消息中缺失的 Feishu 链接，仅在主机于当前进程生命周期内读取后才可搜索。

搜索还可按引用（如 `#12` 或 GitHub URL）查找 Thread，按标题或匹配的 Thread 查找 Ticket。

## 原则

1. **聚焦重要事项。** 每个元素回答三个问题之一，否则移除。数字仅在改变监督者行为时出现。
2. **精确定义。** 分类有可核查的定义，互斥且覆盖所有情况，如同 Ticket 阶段。标签说明一行等待谁、需要什么，而非内部状态码。
3. **概念一处归属。** Source 状态在 Source；Thread 与 Trigger 在 Trigger；Decision 在 Eval；Ticket 在 Execution。设置按所配置的概念分组。Overview 汇总这些页面并链接到它们，而不是成为另一套详情视图。
4. **领域结构。** 面板使用领域术语。内部标识符仅出现在折叠详情中，在可解析处用名称替代标识符。
5. **诚实状态。** 计数等于其链接列表的长度，且没有状态与另一事实矛盾。
6. **默认安静。** 健康状态不占用额外空间。问题必须可行动；一次性网络错误不够。需要人工的行直接提供其操作。页面说明保持一句，进一步细节放在 (?) 工具提示中。破坏性操作需要确认。
7. **一致控件。** 使用 DeepSeek Harness 设计 token 与控件行为。相同信息在面板中应看起来一致。每个设置块一条 save bar，仅在存在变更、正在保存或保存失败时可见。
