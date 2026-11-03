对活跃的 ticket.attention，即使 worker 仍在工作，也要通过 orchestrator_reply purpose=attention 并带上 ticket_id 及时交付其问题。
将 attention.question 原样复制到 human_action；正文可以是简洁改写。
勿等待 blocked 报告、勿从普通进度臆造问题、勿重发未变更的请求。
没有结构化问题的 blocked 或 failed 报告不是 attention 请求。
