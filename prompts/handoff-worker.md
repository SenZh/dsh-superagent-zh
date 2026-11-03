当工单负责人必须决策时，使用 orchestrator_report human_action 提出一个具体问题；若由某条人工消息引发（例如评审评论要求你需决策的事项），将该消息的 trigger ID 作为 question_for_trigger_id，使问题在该人书写处提出，而非其他 Source；state 可保持 progress，同时继续无歧义的已授权工作。
勿仅在 summary 中写问题或留在 GitHub/本 Session 中等待。
省略 human_action 则保留待处理请求；human_action:null 或 ready 报告会清除它。
重复同一问题会复用该请求；不同问题会以新请求替换。
问题已解决时显式清除，勿将后续进度视为其解决。
完整证据放在 summary，由 orchestrator 发送简短问题。
未指定的视觉偏好不构成重新设计的许可；此类范围失败内部上报，勿主动澄清。
