# 008 手册

本手册说明如何扩展 dsh-superagent。术语见 [001 领域模型](001-domain.md) 与 [002 授权](002-authorization.md)，组件见 [007 架构](007-architecture.md)。

## 添加 Source

本示例实现名为 `webhook` 的 Source 种类。它通过虚构 CLI `hookq` 读取假想的消息队列，并将消息转换为 Trigger。其余接口均由 dsh-superagent 提供。

实现承担四项职责：转换事件、定义接收与授权规则、提供回复与上下文、维护轮询循环。下文编号步骤按顺序构建这些部分。

示例假定如下 CLI 约定：

- `hookq pull --after <n> --json` 输出位置 `n` 之后的消息，按时间从旧到新：`{"me": "<队列上智能体的 id>", "items": […], "next": <position>}`。每条 item 含 `id`、`conversation`、`thread`（发起该线程的消息 id，该消息本身无此字段）、`sender`（含 `id` 与 `bot`）、`mentions_me`、`text`、`at` 与 `url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在某条消息下回复，每个 key 仅一次。对话已删除时在标准错误输出 `{"error": {"code": 410, "message": "…"}}`。
- `hookq thread --conversation <id> --root <message id> --json`，后续页用 `--after <cursor>`，按时间从旧到新输出线程的一页：`{"items": […], "next": "<cursor>"}`，最后一页无 `next`。

将 kind 及其插件入口放在 `src/sources/webhook.ts`，参照 `src/sources/github.ts`。若 Source 变大，可像 Feishu Source 那样把转换、Channel 与轮询代码拆到独立模块。

### 1. 将事件映射到领域

写代码前先选定稳定映射。Trigger id、Thread id 与 Channel 地址会被持久化，事后更改可能使已有记录脱节。

| 概念 | 对 webhook 队列 |
| --- | --- |
| Trigger | 一条消息 |
| Thread | 同一对话中某条消息及其回复：`<conversation>:<首条消息 id>` |
| Actor | 队列上发送者 id，与允许名单中的命名一致 |
| Channel 地址 | 对话及回复所挂靠的消息 |
| 接收范围 | 一个对话，在准入表中有独立行 |

### 2. 配置：schema、字段、facts 与健康

模块导入后续步骤所用内容：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是稳定的 Source 标识。它出现在 Trigger 的 `source`、Thread 键前缀、`source:webhook` 设置块与 Source Policy 中。
- `schema` 校验已存配置并供给默认值。在操作员保存实例前，注册表会用这些默认值创建已启用实例。默认值应不接收任何内容；本示例默认无对话。
- `fields` 生成 Source 设置表单，无需自定义面板代码。`rows` 字段编辑以首列为主键的条目；`advanced` 隐藏较少使用的设置。若 Source 能将 id 解析为显示名，实现 `refreshLabels`，并将名称存于 `KV.sources.labels(kind)`。带 `choose` 与 `refresh` 的列即可提供这些名称。
- 可选 `facts(deployment)` 展示由 setup 推导的值，例如 CLI 背后的账号。setup 目前仅检查 `SETUP_FIELDS` 中列出的 GitHub 与 Feishu 字段，故本示例不提供 facts。
- `health` 声明 Source 的健康项，初始为 `starting`。注册时占用每个名称，并与 `runtime`、`reply`、`worker` 或 effect kind 等项冲突则拒绝。使用 kind 名称，并为 Policy 同步保留 `sync:<id>`。`error` 会使状态脚本以 2 退出、每小时检查通知操作员，因此仅在可能需要人工介入时上报。

### 3. 将消息转换为 Trigger

转换前先校验 CLI 响应：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定 id。** 通过 `key` 从队列 id 派生 Trigger id。`Engine.receive` 对已存 id 返回 `duplicate`，且 duplicate 接受仅创建一张 Ticket，因此重读消息是安全的。若将编辑视为新事件，在 id 中包含编辑版本，如 GitHub 对评论使用 `updated_at`。
- **Threads。** 第一个 Thread 为主 Thread 并拥有 Eval。额外 Thread 增加 Correlation 上下文。Source 选择 id；完整键为 `webhook:<id>`。
- **发送者。** 在 `actor` 中使用 Source 原生 id。对 bot 与 CI 设 `human: false`。名单中的 bot 只能成为 inferred 候选。空 actor id（如 CI）使用准入单元处理未列入、未 @ 的发送者。仅当消息确实 @ 智能体时将 `addressed` 设为 `mention` 或 `direct`，这赋予允许用户消息的 explicit 强度。切勿为智能体自己的消息产生 Trigger。
- **回复目标与元数据。** `channel` 标识回复去向；`url` 链接源消息。`channel.thread` 将消息分组以实现有序投递：Ticket 的后续消息会排在更早消息之后，直到更早消息失败三次；其他 Ticket 不等待。将 Source 特有字段放在 `meta` 中，如接收对话与 Thread 根。`meta` 与 `text` 均为不可信数据，不是授权依据。

智能体可通过 `orchestrator_read trigger/<id>` 与 `thread/<key>` 读取新 kind。这些接口暴露消息文本、处置及相邻 Trigger。新 kind 不会自动获得内置 skill、原始服务记录的资源或 PR 绑定。

全文搜索无需额外代码。当人们使用 Source 特有引用（如消息 id、URL 或 `#12`）时实现 `find(query)`。返回这些引用所标识的 Thread 键与搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

此处查询中的队列消息 id（如 `m42`）会搜索对应 Trigger。

### 4. Source 接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

Engine 通过 `SourceRegistry` 对每个入站 Trigger 调用 `scope`。范围外的消息以 filtered 及原因存储，不创建 Thread。将消息交给 `receive`，而非在轮询循环中静默丢弃，以便面板解释过滤原因。

`observed: true` 表示 Source 在该范围内接收未 @ 的消息，从而允许准入表对其决策。`row` 标识对应的准入行。

重新校验时用 Mandate 存储的 metadata 调用 `scope`。应仅依赖 `meta` 与配置。恢复需要 `listened`，而非 `observed`。无 `scope` 时，kind 接收其产生的每个 Trigger 但不 observe 任何一条。

### 5. 回复：Channel

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **幂等发送。** Outbox 调用 `SourceRegistry.send`，进而调用 `channel.send`。Source 插件卸载时消息会等待。发送可能在远程已收到后仍失败，因此重试不得重复消息。用 `message.id` 作幂等键：`hookq` 支持 `--key`；GitHub 存隐藏评论标记；Feishu 接受幂等键。
- **永久失败与可重试失败。** 成功返回 `sent`；仅当投递永久不可能（如本示例 410）时返回 `gone`。gone 消息保留为 dropped 且不再入队；出现在 `GET sources` 与 Source 页。其他失败应抛出，以便 Outbox 在 5 秒后重试，倍增至 300 秒。`CliError.code` 为 CLI 输出报告的错误码。
- **允许的目标。** `allowed(address, config)` 仅检查已存地址与当前配置。将所需事实全部放入 address；例如 Feishu 记录是否为私聊。不再监听的目标被丢弃；禁用的 Source 使消息等待。无 `allowed` 时，Source 启用则允许所有地址。
- **链接。** `link(address, links)` 返回面板通过 `SourceRegistry.link` 展示的 URL 与标签。

### 6. 评估用的 Thread 上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动评估需要 Trigger 周围的对话。`orchestrator_eval` 读取初始窗口；`orchestrator_eval_read` 扩展分页、根或父消息。实现可选的 `threadContext` 接口以提供这些读取：

- `read(commands, trigger, part, cursor, config)` 返回 `ContextPart`。其 `messages` 使用 `ContextMessage`：`{id, from, at, text}` 及适用的 `bot`、`deleted`、`reply_to`、`url`、`path`、`line`。最后一页省略 `next`。`body` 是展示给评估 Session 的内容，此处与 messages 相同。可选 `status` 可描述 Feishu `chat_neighbours` 等特殊结果。`part` 为 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 在其中定位 Trigger 自身消息。
- `images(text)` 与 `image(…)` 用于消息带图片的 kind（如 Feishu）；本示例省略。

无 `threadContext` 的 kind 不向评估提供 Thread 上下文；Trigger 自身文本仍可读。

若消息回复更早消息，添加 `repliesTo(trigger)`。它返回对应 Trigger id，使跟进通知与 worker Input 在已存储时展示被回复消息。例如若 `hookq` 提供 `reply_to` 且转换存为 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁可发起：Policy 中的 Source 部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 校验 Source Policy，含 `allowed` 发送者 id。id 比较不区分大小写。准入表单独校验并在各 kind 间共享。此 Policy 部分存于 `policy.sources.webhook` 并计入 Policy 版本。
- `fields` 在 **Eval → On each Source** 下生成表单，以 `allowed` 开头。`explicit` 说明如何 @ 智能体。可选 `unnamed: {person, bot}` 为 Feishu `ou_…` 等不透明 id 提供显示标签；本示例省略，因队列 id 为可读名称。
- `rows(config)` 列出准入范围及可用单元。每行 id 须与 `scope` 返回的 `row` 一致。未 @ 消息单元仅在 Source observe 这些消息处可用。无 `rows` 的 kind 使用默认单元。
- 可选 `bots` 列出可 inferred 准入的 bot，条目形状与 `allowed` 相同。无则不准入任何 bot。
- 可选 `sync: {label, ready(part), run(host, part, signal)}` 在启动与 **Sync now** 时刷新成员。Feishu 使用群成员。失败出现在 `sync:<id>` 下，成功后前每分钟重试，倍增至一小时。

### 8. Executor 用的变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

执行设置中，即使 Source 禁用也会展示声明的 `variables`（在 prepare 命令旁）。启用时 `values` 用 deployment、配置与 originating Trigger 供给 Ticket 特有值。

prepare 命令读环境变量；worker 在指令中读其值。使用 `DSH_SUPERAGENT_<KIND>_…` 前缀，对来自其他 Source 的 Ticket 返回空值，且永不包含凭据。

### 9. 插件入口及其循环

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

插件注入 `orchestrator`。第一个 effect 注册 kind，销毁时注销。第二个用 `SourceContext` 运行轮询循环。销毁时 abort 循环并 await 完成，包括应用停止或重启时。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```


- **Setup 与开关。** 等待 `ctx.ready`；`ctx.configured()` 为 false 时不读取。每轮读取 `ctx.config(kind)`：Source 关闭时为 undefined。
- **游标。** 用 `KV`（`src/kv.ts`）中的类型化项在 `ctx.state` 存读取位置，如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键字面量仅出现在该注册表。外部包可用导出的 `record` 与 `family` 定义键。设 `undefined` 删除记录；`ctx.atomic` 在事务中分组写入。

  仅在每个相关 Trigger 已存储后推进游标。中断的一轮游标不变；稳定 id 使重放安全。对时间戳游标，`src/sources/poll.ts` 的 `cursorAt(started)` 减 5 秒以容纳延迟的远程写入。
- **Thread 元数据。** 接收 Trigger 后调用 `ctx.threads.describe(key, {label, url, facts})` 填充展示信息。`ctx.threads.patch(key, meta)` 委托 `threads.patchMeta`，设置 Source 拥有的键或在值为 `undefined` 时删除。两者均忽略不存在的 Thread，含 filtered Trigger 的 Thread。
- **逐条错误。** 统计畸形项并继续，而非整批失败。`src/infra/commands.ts` 的 `validItems(item, list, tally)` 支持此模式。对独立轮询的 Thread 使用 `src/sources/poll.ts` 的 `pollThreads`；将每次失败记为 `pollError`。其 `polled` 谓词保留 live 订阅及最后 Trigger 与 Ticket 结束较晚者之后两周窗口。
- **健康。** 仅通过 `ctx.report` 写入已声明项。`health(state, code, values)` 提供 `setupPending`、`sourceOff`、`pollFailures` 等固定消息。连续失败达到阈值再报 `error`。
- **Abort。** 将 signal 传给 `ctx.commands.withSignal(signal)` 与 `delay`，并在条目间检查，以便停止或重启时立即结束循环。

本示例不发送 Source 发起的通知。需要时 Source 调用 `ctx.notify`；Outbox 与其他回复一样投递。

也不将 Thread 绑定到 Ticket 完成。仅当 Source 的 Thread 生命周期决定工作是否完成（如 GitHub PR）时使用 `ctx.tickets.bindThread` 与 `threadClosed`。Engine 仍决定 Ticket 何时结束。

### 10. 注册插件

在 `tsdown.config.ts` 中将模块建为独立入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 的 `exports` 中导出构建文件，添加 `"./webhook-watch": "./lib/webhook-watch.js"`，并将插件加入 bundle 的 `cordis.patch.yml` 的 `insert` 列表，默认禁用，与 Feishu 相同：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在 profile 的 `cordis.patch.yml` 中以 `id: dsh-superagent-webhook` 与 `disabled: false` 启用插件。构建并重启。新 Source 出现在 Sources 设置与 **Eval → On each Source** 下。

将其模块与 spec 加入 [007 架构](007-architecture.md) 的模块与测试表。`tests/docs.spec.ts` 检查两表完整。

### 11. 测试

按 `tests/sources.spec.ts` 模式创建 `tests/webhook.spec.ts`：临时目录中的真实 Store、带 `HealthBoard` 的 `SourceRegistry`、带替身端口的 Engine，以及来自 `sourceContextOf` 的 `SourceContext`。

通过 spy `Commands.run` 模拟 CLI。亦 spy `withSignal`，因其通常返回新的 `Commands` 实例。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 构造队列项。至少覆盖：

- 转换：稳定 id、Thread、actor、`human` 与 `addressed`，以及智能体自身消息无 Trigger；
- 一轮：Trigger 各存一次、范围外一条 `filtered`、游标在其后移动、允许用户 @ 评估为 `explicit`；
- 被截断的一轮保留游标；
- `send`：成功 `sent`、仅 410 为 `gone`、其他失败抛出，CLI 错误由 `commandError` 构造；
- Thread 上下文：一页 messages 与下一页 cursor，如 `tests/registry.spec.ts` 通过 `SourceRegistry.threadContext` 读 kind 的 Thread；
- 插件：`apply` 注册 kind、setup 等待时报 `setupPending`、disposer 注销 kind、逃出循环的失败成为 Source 健康而非未处理 rejection；
- Policy 的 Source 部分与准入行，如 `tests/registry.spec.ts` 与 `tests/policy.spec.ts` 检查 Feishu Source。

### 检查清单

1. 映射 Trigger、Thread、actor、Channel 地址与接收范围。
2. 编写 kind：`schema`、`fields`、`health`，若 setup 推导任何值则加 `facts`。
3. 以稳定 id 将事件转为 Trigger，并丢弃智能体自身消息。
4. 在 `scope` 中决定 Source 接收什么，仅依赖 `meta`。
5. 提供 Channel：`send` 每条消息只发一次，仅当对话永远无法接收时返回 `gone`。
6. 若可读 Thread，为评估提供 Thread 上下文。
7. 提供含 `allowed` 与准入行的 Policy 部分。
8. 注册执行环境变量。
9. 编写插件入口与循环：等待 setup、跟随开关、在 `receive` 后移动游标、逐条保留错误、上报健康、abort 时停止；在 `KV` 中命名游标。
10. 添加构建入口、export 与 `cordis.patch.yml` insert，并在 007 表中列出模块与 spec。
11. 按 `tests/sources.spec.ts` 测试其他 Source 的方式测试。
