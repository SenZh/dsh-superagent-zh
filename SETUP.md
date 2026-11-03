# SETUP

供**与运维者一起安装 dsh-superagent 的智能体**使用的说明。若你是人工操作，请让你的编程智能体阅读本文件并按其执行。

## 如何运行安装流程

安装流程遵循 [docs/000-philosophy.md](docs/000-philosophy.md)：**尽可能少占用运维者的注意力**。

- **在第一个问题之前，** 只阅读本文件与 `docs/000-philosophy.md`，并**运行一次**下方探测命令。在收齐全部答案前，不要阅读其他文档。
- **每条消息只问一个问题，** 只涉及一件事。等到答案后再问下一个。探测已能回答的问题则跳过。
- **每条消息控制在 300 词以内，** 除非运维者要求更详细。每个问题给出**可用一个词接受的推荐答案**，并用一句话说明为何重要。
- **提问阶段不要执行安装。** 收齐所有答案后，发**一条**包含完整计划的消息，仅在运维者确认后再执行。
- **永远不要在聊天里索要密钥。** 需要运维者亲手完成的步骤（例如创建账号、CLI 登录、填写应用 Secret）合并为**一条**消息，给出在其**本机终端**运行的确切命令。
- **通过问题逐句传递理念，** 不要长篇宣讲。

## 1. 探测（Probe）

在本仓库检出目录中**运行一次**，结果自行保留：

```sh
node -v; pnpm -v; git --version; tmux -V; python3 --version; gh --version | head -1; dsh --version
gh api user --jq .login
command -v lark-cli
grep -l -E 'id: dsh-superagent\b' "${DSH_HOME:-$HOME/.dsh}"/profiles/*/cordis.patch.yml 2>/dev/null
```

- **前置条件：** Node.js 22.19+ 或 24+、pnpm、git 2.32+、tmux、Python 3.8+、GitHub CLI 2.48+，以及 `package.json` 中 `engines.dsh` 范围内的 DSH。缺项写入计划，不作为提问。DSH 还需要模型提供商；若运维者从未用模型跑过 DSH，配置提供商属于运维者步骤。
- **运维者登录：** `gh api user` 的结果是运维者本人的 GitHub 登录名，用作问题 5 的推荐值。
- **已有 profile：** 若某 profile 的 `cordis.patch.yml` 中已有 `dsh-superagent` 条目，说明曾安装过：询问是否继续、是否保留配置，并阅读 `docs/006-deployment.md` 中的升级（Upgrade）章节。

## 2. 问题

按此顺序**逐条**提问，工作流顺序见 [docs/001-domain.md](docs/001-domain.md)：先 Trigger 从哪来，再谁可在该处请求工作，最后 Executor 如何处理 Ticket。方括号内为推荐答案。

**Sources：请求从哪来。**

1. **Sources。** 智能体只接 GitHub，还是也接飞书会话？ [默认仅 GitHub，除非团队主要在飞书协作。]
2. **Repository。** 智能体要处理的 github.com 仓库是哪个？格式 `owner/name`。
3. **Agent account。** 智能体以哪个 GitHub 用户身份行动？ [专用账号，不要用运维者本人：智能体写的一切都会显示在该账号名下。] 账号可能尚未存在，创建属于运维者步骤。
4. 若启用飞书：**哪些会话，** 按名称？机器人在配置完成后会查找其 id。

**Policy：谁可以请求工作。**

5. **GitHub 上。** 哪些 GitHub 登录名可以向智能体要工作？ [建议仅 `<operator login>`：智能体只为名单上的人行动；短名单在运维者熟悉行为前能降低团队噪音。]
6. 若启用飞书：**飞书中谁可以请求？** [默认第一个会话中的所有人，与其成员同步。]

**Execution：智能体拿到 Ticket 后做什么。**

7. **Checkout。** 每个 Ticket 从哪个本地 clone 检出？ [默认 `~/src/<name>` 下新建 clone。]
8. **Commit identity。** 智能体的 commit 作者应显示为谁？ [智能体账号，使其工作可识别。] 若要用其他身份（如运维者），写在 Delivery requirements 提示词段落中。
9. **Auto-merge。** Pull Request 就绪且已批准后，是否由智能体自行开启 auto-merge？ [默认否：在 PR 长期正确之前由人工合并。] 智能体不能以智能体账号开启 auto-merge；若选「是」，需要另一个有合并权限的账号的 GitHub CLI：在运维者步骤中让其登录，并在 Delivery requirements 中写明。若选「否」，从 Delivery requirements 中删除 auto-merge 段落。

不要询问有良好默认值的项；在计划中列出即可：

- profile 名 `superagent`，从 DSH Web profile 复制并与运维者日常 profile 分离，以便重启而不关闭其 Session；端口 `3082`；状态目录 `<home>/.dsh/state/superagent`（绝对路径）；
- 在运维者本人 profile 上的每小时检查，用于在 dsh-superagent 未运行时修复；
- 分支前缀 `agent/`；
- accept 模式 `observe`（主动工作只记录不执行）及默认规则与 prompt 段落；
- 飞书：机器人仅读 @ 提及，并开启私信。

## 3. 计划（Plan）

发**一条** 300 词以内的消息：将安装什么、账号与 CLI、要保存的设置、运维者步骤、默认值。请求**一次**确认。

## 4. 执行（Execute）

1. **先做运维者步骤。** 阅读 `docs/006-deployment.md` 的 Accounts 章节，自行编写 CLI wrapper，再发一条消息列出仅运维者能做的事：创建智能体账号；若 auto-merge 则登录可开启它的账号；智能体账号接受仓库邀请；对每个 wrapper 运行 `auth login`；飞书则创建并发布机器人应用，用 Secret 运行 `config init`。等运维者说完成，再用 `<wrapper> api user --jq .login` 检查各登录。
2. **安装并启动。** 按 `docs/006-deployment.md` 的 Install、Machine configuration、Start and stop 执行。跳过对本仓库的 `git clone`：你已在检出目录中。若问题 7 要求新 clone，则 clone 问题 2 的仓库。若已有 profile，跳过 `dsh superagent --from-default-profile`（会替换 profile 并删除其 `cordis.patch.yml`），改为编辑该文件而非覆盖。
3. **Settings。** 按 `docs/006-deployment.md` 的 Management API 顺序保存：`github` Source、若用的 `feishu` Source、`policy` 块（允许名单；飞书先用机器人 CLI 查 chat id）、`executor` 块（问题 7 的 prepare 命令）。再保存 prompt，在 Delivery requirements 中写入问题 8 的 commit 身份与问题 9 的 auto-merge 答案。若保存返回 `restarting: true`，dsh-superagent 会重启：在 `scripts/dsh-superagent status` 不再以 1 退出后再发下一请求，并用 `GET settings` 重新读取 revision。
4. **失败。** 被拒绝的 save 会在响应体中说明安装问题。在 `docs/006-deployment.md` 的 Troubleshooting 表中查找、修复并继续。仅在你无法自行修复时问运维者。

## 5. 验证与交接

`scripts/dsh-superagent status` 以 0 退出且无 setup 问题。若你在运维者本人的 DSH profile 中运行，用调度工具创建 `docs/006-deployment.md` Hourly check 章节中的每小时检查；否则把对应 prompt 交给运维者在其 profile 中调度。再发一条 300 词以内的消息：

- 面板 URL，以及侧边栏 **Superagent** 入口。
- 如何试用：用允许名单上的账号，在仓库某 issue 评论 `@<agent login> <request>`。约一分钟内概览会出现 Ticket。
- 给运维者的三条规则，来自 `docs/000-philosophy.md` 的 Responsibility：从窄起步、为降噪调优、对发言负责。
- 延伸阅读：`docs/` 下编号文档，按序阅读。
