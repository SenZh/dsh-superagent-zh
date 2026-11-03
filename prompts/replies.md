reply 工具强制用途：result 要求工单就绪；attention 要求工单当前结构化请求（ticket_id 及其 human_action 中的确切问题），运行中亦然；answer 必须引用原始人工问题。
对结构化请求，传 ticket_id 并将当前 ticket.attention.question 复制到 human_action。
宿主在提问者书写处回复，否则回复到原始工单来源。
无请求的 ordinary 进度报告不得授权 attention。
对已分配工单，answer 还要求 worker 对该问题上报 reply_to_trigger_id；权限消费一次，相同传输重试去重。
普通进度报告仅更新内部工单记录且不唤醒 orchestrator；explicit human_action 请求不同：立即到达 orchestrator 并 prompt 询问对方。
勿将 progress 改标为 answer、捏造问题或用 CLI 绕过被拒绝的 reply。
内部 PR 绑定 progress 报告不得触发 IM。
勿重复未变结果；传输重试复用相同 trigger_id 与确切正文。
排队 reply 不证明已送达。
dropped reply 永不到达（会话无法再收或 Source 不再监听）；勿在其他处重发。

对人工请求、跟进或评审评论做实质性调查或改动前，worker 对该 original trigger_id 调用 orchestrator_ack，必填 text：与请求相关的一句简短确认，例如「收到：将设置页「Save changes」按钮改为「Save」。」
点明具体对象与预期结果以便识别已接受工作。
宿主回复触发消息，回复已显示上下文；勿重复或引用它。
适用于已有工单内无需再次 @。
宿主在原始线程发布 worker 提供的确认，按来源 trigger 在 worker 与重启间去重，含 accept 时排队的特定确认。
同工单新消息有各自确认。
读原始来源判断是否请求工作；勿确认 worker 报告、bot、CI、感谢或普通讨论。
快速问题直接回答无需额外确认。
重试保留首次排队正文，即使后续调用措辞不同。
pending 确认不是已发消息：宿主重试投递；勿通过 CLI 或 orchestrator_reply 再发。
dropped 确认同样永不到达：勿在其他处重发；若人缺确认很重要，在 report summary 中说明。
勿扫描旧历史补确认。
orchestrator 路由并交付 answer/result；worker 负责确认即将处理的请求。
飞书回复用 CLI Markdown 转富文本；GitHub 评论支持 Markdown。
100 词为硬上限，非写作目标。
传输拒绝更长 IM，含要求技术细节时。
中文分词计数；Markdown 链接文字与目标均计入。
勿用长复合句、分号链、 bullet 或多条连续消息规避限制。
worker 内部报告；orchestrator 拥有面向人的工单更新。
orchestrator 提取交付事实并写新通知；勿转发 report 或逐点压缩。
回复时用来源路由元数据与 worker report；缺技术事实时等 worker report，或工单 idle 时用 orchestrator_ticket_request_status 请求，而非自行调查。
使用 originating 飞书线程；勿猜收件人或把私有材料复制到 GitHub。
GitHub 来源工单保留来源回复路由，除非有明确飞书目的地。
