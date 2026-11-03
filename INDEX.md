# dsh-superagent-prototype 中文文档包

| 项目 | 说明 |
| --- | --- |
| 英文原文 | 与上游仓库目录结构一致，可单独克隆 |
| **中文文档（本包）** | 与本 `INDEX.md` 同目录下的 `.md` 与 `locale/` |
| 上游仓库 | https://github.com/deepseek-ai/dsh-superagent-prototype |

## 内容

- **28** 个 `.md` 文件：与英文仓库一一对应（`README.md`、`SETUP.md`、`docs/000`–`008`、`prompts/**`、`skills/**/SKILL.md`）。
- `locale/zh.json`：面板元数据（与上游一致）。

## 阅读顺序

1. `README.md` → `docs/000-philosophy.md` → … → `docs/008-cookbook.md`
2. 安装：按 `SETUP.md` 与 `docs/006-deployment.md`
3. 实现扩展： `docs/008-cookbook.md` + `docs/007-architecture.md`

## 翻译约定

- 命令、配置键、TypeScript/API 标识符、文件路径：**保留英文**
- 叙述、标题、表格说明：**简体中文**
- 术语：**Source** → 事件源（文中常保留 Source）；**Ticket** → 工作项；**Harness** → Harness
- GitHub 自动化声明 `This is by an automated agent, not a human.`：**按产品要求保留英文**

## 生成说明

由上游英文文档翻译而来；目录结构与官方仓库中的 Markdown 路径一致。许可证以官方仓库为准。
