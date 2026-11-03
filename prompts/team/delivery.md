对每个变更 UI 的工单，PR 描述必须同时包含 Before 与 After 视觉证据，清晰标注，并标识基线与变更版本。
捕获相同场景、视口、主题与相关 UI 状态以使对比有意义；截图或演示均可。
仅 After 证据不完整。
使用实际应用捕获，勿用 mockup 冒充先前实现，勿暴露私有 Feishu 内容或凭据。
后续 UI 变更使对比过时时刷新对比。
若无法获得真实基线捕获，诚实说明缺失证据与限制，保持交接不完整，勿声称 UI PR 已就绪。
Worker 捕获并验证对比；orchestrator 在交付就绪结果前检查所报告证据。

请求匹配规则、分配、本 prompt 或仓库说明所要求的 review。
切勿以智能体账号启用 auto-merge，以免智能体合并自己的工作：仅在本节命名的 merge CLI 下启用，即 `<merge CLI> pr merge <number> --auto --merge --match-head-commit <sha> --repo $DSH_SUPERAGENT_GITHUB_REPOSITORY`。
本节未命名 merge CLI 时，由人工合并每个 PR。
有 merge CLI 时，在工单分支打开的 PR 的 head commit 上，当以下全部成立时启用 auto-merge：PR 就绪；变更少于 200 行（增删合计）；那些说明要求的每项批准覆盖该 commit；且若变更用户可见内容，那些说明点名谁批准此类变更且该批准覆盖该 commit。
报告就绪勿等待批准，批准到达时启用 auto-merge。
否则由人工合并 PR；在报告中说明。
一旦任一条件不再成立（如 review 请求变更），以及 push 到 PR 前，用 `<merge CLI> pr merge <number> --disable-auto --repo $DSH_SUPERAGENT_GITHUB_REPOSITORY` 撤回 auto-merge；新 head 再次满足条件时重新启用。
切勿代表 reviewer 批准或合并。
对打开的 PR，每个审阅者每个 PR 仅记一次功劳，在 PR 分支的每个 commit 上：当人类 reviewer 对其反馈首次被评估时，重写分支使每个 commit 携带迄今每位被记功者一条 `Co-authored-by: <name or login> <id+login@users.noreply.github.com>` trailer，id 来自 `gh api users/<login> --jq .id`。
先按上撤回 auto-merge。
运行 `git fetch origin`，用 `gh pr view <number> --json baseRefName` 的 base 分支设 `base=$(git merge-base HEAD origin/<base branch>)` 与 `old=$(git rev-parse origin/<head branch>)`，再运行 `FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch -f --msg-filter 'git interpret-trailers --if-exists addIfDifferent --trailer "<trailer>"' $base..HEAD`，每位被记功者一条 `--trailer`，仅改 commit 消息，并用 `git push --force-with-lease=<head branch>:$old` 推送；若 push 被拒，停止并报告。
之后添加的 commit 携带相同 trailer。
对某 commit 的批准也覆盖其重写副本（树相同）。
若 PR 已合并，勿重写；在 orchestrator_report 记录未记功 reviewer。
切勿 push 空 commit 以记功 reviewer。
勿记功 bot（登录名以 [bot] 结尾及本指令点名的 review bot）、PR 作者、智能体账号或本指令点名的 commit 作者（已作为每个 commit 作者记功）。
在 orchestrator_report 记录已记功者，以免后续 worker 重复记功。
若人类在处理同一问题且其分支或 PR 含本工单代码（来自工单分支的 commit 或其变更被携带），立即用 gh 关闭本工单 PR 勿问任何人，留一句说明人类 PR。
仅相关或不含工单代码的工作不是关闭理由。
关闭 PR 结束工单；勿发其他消息。
