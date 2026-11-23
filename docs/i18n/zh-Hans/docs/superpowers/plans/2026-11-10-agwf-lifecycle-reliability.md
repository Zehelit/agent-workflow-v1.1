<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# 生命周期启动与恢复实施计划

> 2026 年 11 月 10 日的历史实施计划。清单描述当时拟议的工作。授权及私有证据仅适用于原任务。当前行为见[运行约定](../../../skills/agent-workflow-v1-1/references/operations.md)。

<a name="goal-and-constraints"></a>

## 目标与约束

计划覆盖首次启动失败后的安全重试、及时刷新测量、已验证恢复和正常关闭。保留固定 candidate、类型化核心和无模型监视器。生命周期工具区分准备、真实宿主观测、运行与关闭。计时测试仍为显式模式。参见[历史规范](../specs/2026-11-10-agwf-lifecycle-reliability.md)。

原计划授权相关 v1.1 源提交和推送，不包括 GitHub Release 或发布标签。保留 Python 3.11+、标准库、unittest、现有 Codex 目标与用量工具和 Windows 进程、文件共享行为。保留在做的工作、老证据、包与运行绑定。包、目标、测量、用户引语及进程身份须实际观测。合成数据仅用于隔离测试。过期、额度、提前保留和用户停止各有授权条件。正常运行不注入 SAVE，不改其他会话或用户模型设置。

<a name="review-focus"></a>

## 审查重点

1. 首次启动确因过期留下 SAVE 时，尚无活动状态也应能按条件重试。
2. 额度或用户停止、新目标和未知进程不能走技术恢复。
3. 半启动或并发调用须在证据明确前保持未知。
4. 较新文件不能刷新旧观测。样本新鲜时租约也可能过期。
5. 迟到的原生事件无法通过计时。Windows 暂时共享故障不能证明进程退出。

<a name="files-and-responsibilities"></a>

## 文件与职责

| 路径 | 计划职责 |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | 启动尝试、重试决定、监视器归属与生命周期结果 |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | SAVE、轮次和回执，只为约定做必要改动 |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | 保留 active/halted/complete 与停止规则 |
| `tooling/run-lifecycle-check.py` | 利用主代理提供的真实原生证据运行预备好的工具 |
| `tooling/tests/test_lifecycle_check.py` | 测试准备、迟到事件和结果分类 |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | 源码及安装入口与真实所属 Python 进程集成 |
| `test_native_usage_source.py`、`test_goal_binding.py`、`test_watcher_engine.py`、`test_monitor.py`、`test_watcher_atomic_write.py` | 回执、目标及授权、计时和 Windows 回归 |
| `SKILL.md`、`references/operations.md`、`references/native-usage.md`、`references/coexistence.md` | 刷新、重试、安装及宿主独立验收说明 |

<a name="task-1-retry-a-failed-first-activation"></a>

## 任务 1：重试首次启动失败

计划接口有 `activation_identity(command: str, payload: dict) -> dict`、`record_activation_attempt(runtime, record: dict) -> None`、`startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`。每次运行的 `activation-attempt.json` 记录来源、`schema_version`、`command`、`pre_state_sha256`、`phase` 和监视故障。`prepared`、`failed`、`committed` 不同，未结束的 `prepared` 需要核对。

- 用旧回执验证首次失败，再以同一目标和新数据重试 `bind-active-goal` 和 `activate`。保留最初 SAVE 哈希及授权，验证进程健康且状态活动。
- `preflight_activation` 后持久记录意图，绑定命令、运行、目标、会话、范围、目标文字及授权。新观测不自动扩大范围。
- 在启动与核心步骤之间用所属单写入者锁。遇到未知持有者拒绝。原生界面与本地状态并非同一事务。
- 只有同运行确认 `usage_receipt_expired`、新活动目标观测及真实回执时使用技术轮次。调用方的重试标志不授权。写入成功前先验进程与核心提交。
- 拒绝额度、提前保留、其他诊断、新 SAVE、用户停止、目标变更或非活动、存活或未知进程、变更或未完成日志、包不匹配及并发启动。测试无效重试回执后的有效重试。回退仅恢复获验证的原控制并保留更新的 SAVE。
- 执行 candidate 共存和原生回执测试，检查活动状态、进程身份、样本新鲜度、保留的历史和实际清理。

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## 任务 2：先准备，再及时刷新

工具提供 `prepare`、`run --input PATH`、`observe --input PATH` 和 `close`。准备应核验包、固定值、角色和命令，不启动。`refresh_by(at: float, interval: float) -> float` 建议 `at + interval - min(15, interval / 4)`，60 秒间隔即 45 秒刷新，真实期限不变。

- 测试准备及帮助不依赖实际输入；缺失或陈旧数据不产生活动状态；重放不刷新时间。
- 准备后才调用真实工具，收集目标及用量，只复制所需字段、两个窗口和重置时间，排除账号、身份与信用字段。
- 从输出区分 `sample_at`、`next_check_at`、`check_interval_seconds`、`refresh_by`。发布回执或心跳不证明近期健康。
- 要求 `status.at == usage.observedAt`、存活身份、轮次匹配、`sampleFresh` 与 `leaseValid`。数据传入后另验续租。不能保证不可中断调用期间一定取到新回执。
- 正常运行不启用计时和故障注入。要求检查点时验证恢复。正常关闭核实 complete、进程退出及无人工 SAVE。
- 生命周期和回执测试采用不同的观测及文件发布时间。

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## 任务 3：计时诊断正确处理迟到事件

- 仅明确请求才用 `timed-test`，先准备健康进程，再提供真实事件及 `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`。目标时间为 `event_at + 10`，等待用单调时钟。
- 晚到 45 秒返回 `INVALID_TIMING`，历史 98.74 和 45.05 秒结果保持失败。
- 使用官方目标证据。无支持的原生事件钩子时把晚到或缺失观测如实分类。不要借私有传输或凭据测量界面。
- 分别记录 `native_event_at`、`observed_at`、`worker_healthy_at`、`save_requested_at`、`save_returned_at`、`snapshot_verified_at`、`helper_halted_at` 和 `native_paused_at`。9–11 秒窗口只衡量已健康的进程发出 SAVE，原生暂停单独证明。
- 两轮使用同一目标和运行。先证明实际活动、检查点及 halted，再验证获授权的真实继续。首次绑定不可算恢复。
- 测试提前、迟到、重放和篡改的合成事件、缺暂停证据、用户停止及双轮流程。实时计时与正常验收分别处理。

<a name="task-4-verify-recovery-and-regressions"></a>

## 任务 4：恢复与回归验收

- 在源码和成品入口验证首次健康启动、首次过期后重试、已活动运行过期后停止/快照/halt/恢复、错误后正确重试、原生暂停到继续和自然关闭。
- 恢复之前核实 `halted` 和已核验快照。之后确认同一目标及运行、新轮次、旧 SAVE 哈希、原授权与真实进程健康。
- 拒绝真实额度、提前保留、新额度 SAVE、用户停止、不同目标、过时或重用回执、坏检查点及活跃或未知子代理。原生 `blocked` 按自己的条件。
- Windows 测试仅复制必要归属记录。进程身份与终态健康一起确认。共享故障不能当退出，持续权限问题须在有限时间内返回。
- 重点测试后运行两个全量测试集，从发现结果取数量。实时正常验收需要真实目标与预备工具。过期和计时按单独授权，未测试可显示 `not_tested`。

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## 任务 5：记录约定并验证发布

- 分开记录首次重试、已停止恢复、额度与用户停止，解释刷新及续租顺序和无效计时。
- 清点包、工作副本和现有运行。保护用户改动及正在使用的安装。不能从旧固定值断言活动，也不迁移现有运行。
- 同源构建两次，对比清单及 ZIP 哈希。测试变动文件。已安装 candidate 在临时目录被动验证，并核验固定值和 Windows 权限。说明此检查范围。
- 分别记录 `startup_retry`、`receipt_refresh`、`halted_resume`、`native_continue`、`natural_complete`、`timed_test`、`worker_exit` 和 `package_match` 的证据，保留历史反例。
- 对当时授权的发布，独立审查授权、回滚和 SAVE 保留，排除私有报告、回执与运行记录。

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## 任务 6：对齐已有 v1.1 的身份和展示

- 显示名、调用身份、安装路径和新运行空间一起调整。旧运行身份与固定值保留。
- `CHANGELOG.md` 记录真实 v1.0 及 v1.1 改动。README 解释目的、限制、架构、工作流和编排。
- 人工与代理安装需说明干净工作副本、外部哈希、核验和显式调用。安装不启动目标或改变模型。
- 对当时已授权的发布，使 About 和主题匹配源码。保留访问权限及分支保护，核验所展示的命令和图。
- 只发布获授权文件，重读远端提交、树、版本和文档，核实没有创建发布或标签。

<a name="acceptance-boundary"></a>

## 验收范围

原计划先做核心任务 1 至 5，再做最后发布。要求验证首次重试、真正 halted 后继续的矩阵、回归、包检查和范围正确的观测。原生界面只凭实际宿主证据验收。十秒计时需要真实事件与独立观测时刻。
