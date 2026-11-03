# dsh-superagent（中文文档）

本目录为 [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) 文档与提示词的中文译本，相对路径与官方仓库一致。

- 代码与配置键、CLI 命令、API 路径保持英文原文。
- 叙述性正文与标题为简体中文。
- 阅读顺序仍建议按 `docs/` 编号从 000 起依次阅读。

---

# dsh-superagent

> **仅供参考的研究原型，并非 DeepSeek 官方产品。** 本代码源于团队日常开发，其中大量工程工作由编程智能体完成。请将其视为研究代码，而非可直接上线的生产软件。仍可能存在缺陷，编码风格与工程实践也可能不一致。在你自己的环境中使用前，请审阅并自行适配。
>
> 为公开发布我们做了适配，因此与内部版本或论文 *Exploring Super-agent Design: Towards a More Autonomous Agent System*（探索 Super-agent 设计：迈向更自主的智能体系统）中所述实现并不完全一致。文中讨论的部分设计是未来方向，**未**在本原型中实现。

**dsh-superagent** 是针对**单个 GitHub 仓库**的**主动式编程智能体**，打包为 [DeepSeek Harness](https://www.deepseek.com/en/harness/) 的 bundle。它监听 GitHub，并可选择监听飞书（Feishu）。当**被授权**的人向智能体提出工作时，它会创建一条工作项，在**独立的智能体会话与工作目录**中执行，并在**原对话**中回复（例如通过 Pull Request）。Web 面板展示**需要人工处理**的事项。

这也是一种与智能体协作的方式：**尽可能少占用人的注意力，尽可能多产出价值。** 能为整支团队行动的智能体，也可能浪费整支团队的注意力，因此它**仅在被人请求时**，或**团队编写的规则允许时**才行动，并且**仅在必须有人介入或结果可被使用时**才发言。[docs/000-philosophy.md](docs/000-philosophy.md) 说明了该原则以及对运维者的要求。

## 安装与配置

步骤较多，因此由智能体**与你一起**完成。对你的编程智能体说：

> Read SETUP.md in this repository and set up dsh-superagent with me.
> （阅读本仓库中的 SETUP.md，并和我一起配置 dsh-superagent。）

它会**一次只问一个简短问题**，**展示完整计划一次**，然后安装并完成全部配置。具体流程见 [SETUP.md](SETUP.md)。

## 文档

请**按编号顺序**阅读；每一篇只依赖其前面的文档。

- [000 理念](docs/000-philosophy.md)
- [001 领域模型](docs/001-domain.md)
- [002 授权](docs/002-authorization.md)
- [003 执行](docs/003-execution.md)
- [004 提示词](docs/004-prompts.md)
- [005 面板](docs/005-panel.md)
- [006 部署](docs/006-deployment.md)：手动安装、配置、升级与排障
- [007 架构](docs/007-architecture.md)
- [008 手册](docs/008-cookbook.md)：新增一个 Source（事件源）

## 开发

`pnpm check` 会运行类型检查、测试与构建。

## 许可证

[MIT](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/LICENSE)
