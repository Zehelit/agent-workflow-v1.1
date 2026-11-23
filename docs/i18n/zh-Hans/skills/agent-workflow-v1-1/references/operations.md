<a name="operation"></a>

# 运行

需要 Python 3.11+ 和标准库。可用 `python -X utf8 -B scripts/candidate.py --help`。启动前查看[运行绑定](coexistence.md)，保存包外构建回执的清单 SHA。若系统找不到 `python`，使用宿主附带的 Python。命令通过 `--input path` 或 stdin 读取 JSON。状态在 `<project>/.agent-workflow-v1-1/runs/<run-id>/state.json`。检查点、锁、临时文件、监视器归属和协调日志也使用同一隔离目录，仅主协调者写入。文件、测试产物和快照不作日常提示词输入。

<a name="start-and-dispatch"></a>

## 启动与派发

显式调用后应在同一轮启动，再做实质工作，包括准备和小任务。仅阅读指令不算启动。无权创建原生目标时，使用独立监视；不要编造目标或绑定证据。讨论或维护技能不会恢复已停止的工作。

在完整工作流之外监视时，选择新的运行 ID，用固定入口调用 `watcher start --usage-input USAGE.json`。仅启动监视，不激活工作流，也不授权代理。用 `watcher status` 确认该运行和轮次有存活的所属进程及近期健康样本，或读到拒绝。用 `watcher ingest` 提交后续实际观测，工作边界用 `watcher renew`，结束后用 `watcher stop` 并确认进程退出。匹配的健康运行可复用。停止后按认可的继续、新轮次和恢复条件处理，不能另开进程越过停止。开始实质工作前报告结果。启动失败或出现 `SAVE_AND_STOP` 时报告原因并暂停新工作。

启用或恢复前，查看[原生用量回执](native-usage.md)。主代理从现有 `mcp__codex_app__get_usage_limits` 取得读数，用 `--usage-input` 提交。后台只读回执，不启动 Codex app-server，也无法主动调用宿主工具。须在动态期限前获得新观测，复用文件不会增加有效期。缺失、过期、重放或不属于当前运行的数据都会停止工作。独立监视也适用。`watcher ingest --usage-input PATH` 仅发布数据，不启动进程也不启用 AGWF。

1. `validate` 检查定义。`activate` 接收 `objective`、下文的类型化授权、`models`（实际看到的模型与推理力度）、`slot_limit`（含主代理的活动槽数）、`inventory`（真实子代理标识和状态）、`observed_at`（Unix）和可选 `session_limit`。不要编造可用性证据，启用不分配或报告主会话模型。
2. 已授权的 `activate`/`bind-active-goal` 由固定入口启动私有监视器，在提交活动状态前验首个健康样本。有效 `resume` 开新轮次。活动工作流或协调命令续租 600 秒。在较长步骤之间用 `workflow status` 获取健康状态并续租。租约到期或测量不健康即拒绝新工作，恢复命令仍可用。`complete`/`halt` 请求停止并验证进程退出。手动 `watcher start/status/renew/stop` 用于独立监视和诊断，采用新 ID。Codex 没有自动启动钩子，显式选择后主代理自己执行和验证启动。过期及恢复按固定监视角色规则处理。采样不调用模型、控制聊天或出于例行检查启动代理。`usage-due` 返回 `REUSE_MEASUREMENT` 或 `READ_USAGE`。重要或并行工作之前可通过 `substantial_work: true` 要求重新测量。`usage` 接收 `observed_at` 和真实工具的 `rateLimits`。两组窗口均需存在，未知时重试一次再恢复。剩余 >40%、>25%、>20%、≤20% 时的期限分别为 300、120、60、30 秒，并可按每窗口实际消耗速度缩短至预计到 20% 剩余时间的四分之一。旧样本与重置不能建立消耗速度。剩余 ≤25% 或预计 ≤10 分钟时，`CHECKPOINT_SOON` 表示应验证已完成工作的检查点。完成一批写入后刷新，无须每条消息刷新。剩余 ≤20% 或预计 ≤120 秒到达该线，`SAVE_AND_STOP` 冻结派发、提前恢复。`prepare` 拒绝超过动态期限的观测。没有空闲轮询，不保证跨越无法中断的工具调用或其他会话的用量突增。
3. `task` 接收 `id`、`role`、`objective`、`file_inputs`、`context_refs`、`review_targets`、`criteria`、`writes`（准确项目相对文件，包括将删除的文件）、`depends_on` 与布尔 `independent_review`。只读任务写入列表为空，不能占用整目录。每次派发前用 `inventory` 刷新真实子代理状态。
4. `prepare` 接收任务 `id`、可选 `effort` 和固定的可选 `current_context`。检查模型、用量、槽位、依赖和文件哈希，预留容量并返回含 `fork_turns: none` 的实际 `collaboration.spawn_agent` 参数。消息只携带分配角色的 `SKILL.md` 和 `MEMORY.md`。缺失或为空会被拒绝，文件均在包清单内。按返回的参数调用。随后 `record` 接收 `id`、`session_id`、`submitted_arguments`、实际 `receipt`，以及观察到的可选 `effective_model`/`effective_effort`。失败或不确定记录为 `dispatch_failed`，包括 `certain_not_started`。不确定会话在实际清单核对前阻塞进一步派发，不能盲目重试。
5. `finish` 接收 `id`、`accepted`、`evidence`，及需要独立审查时的 `review_task`。先通过 `inventory` 观测会话 idle、stopped 或 done。写入者还在运行时不能释放归属。验证者在 `review_targets` 指明产物作者的任务和修订。作者应停止写入，产物尚待验收。审查期间哈希必须稳定。拒绝的工作保持拒绝，修正后另开任务。`cancel` 用 `id` 和 `reason` 取消从未派发的就绪任务。集成前检查用户改动，重试外部操作前先解决未知结果。

工作流角色限 `gpt-6.1-sol`、`gpt-6-astra` 或 `gpt-6.1-terra`，推理至少为 `medium`。具体模型和默认力度固定在角色文件，不自动替代。主会话沿用用户设置，不改变已运行的模型。准备的派发参数保持精确的角色配置。原 `astra_coordinator` 不可用。

用量监视顾问固定 `gpt-6.1-terra`/`medium`，没有备用模型。启用时要记录实际观测到的模型和推理支持。找不到准确设置就拒绝准备。

<a name="recovery-and-stop"></a>

## 停止与恢复

<a name="initial-activation-retry-v11"></a>

### 首次启用重试（v1.1）

candidate 在首次启用前写入 `activation-attempt.json`，只在监视器健康启动并已提交核心状态后标为提交。失败可能返回 `startup_retry.attempt_sha256`，应在可修改的运行目录外保留。用真实新用量、同目标新观测、原授权和 `--startup-attempt-sha256 PRESERVED_FAILURE_PIN` 重试同一 `activate` 或 `bind-active-goal`。修改日志后重新算哈希不算批准。

| 情况 | 支持的处理 |
| --- | --- |
| 有首次启用用量过期记录，身份与状态未变，已核实进程不存在 | 用包外的失败记录固定值和失败后的新目标及用量观测重试同一命令。 |
| 已有 halted 工作流，记录原因是测量过期 | 按 `usage_recovery: true` 执行核验过的 `resume`，仍需所有停止后检查。 |
| 额度、提前保留、用户停止、其他失败、日志改变或未完成、进程未知 | 保留证据，并遵循适用的继续授权或核对规则。 |

启动、提交、回退、停止和授权切换由所属 candidate 锁保护。遇到不明的 `lifecycle.lock`，先查持有者和未完成操作。与 Codex 原生工具之间没有通用原子事务。协调批次按序记日志，各步骤并非整体原子执行。

先准备，再取得原生测量。可按 `sample_at + interval - min(15, interval / 4)` 提前刷新，但这不会延长回执。分别确认新样本时间、存活进程与轮次、心跳、新鲜度及租约。单独 ingest 不会续租。周窗口例外用真实用户原话和有限期限。核心与监视器使用同样的决策窗口，并保留两个窗口读数。

<a name="initial-native-goal-binding-n02"></a>

### 首次绑定原生目标（N02）

原生目标已活动且准备超过人类来源有效期时可用 `bind-active-goal`。它只启动完全初始的非活动 schema2 辅助程序，不能有任务、历史操作字段、停止、halt、完成、迁移或重放记录。已有运行使用原授权与新观测。原生绑定的 halted 运行可观测停止后 `paused`/`blocked`，再观测同目标较新的 `active`。只有 `updated_at` 变化不能证明用户操作。同一授权活动目标的过期中断另有下文的技术恢复。

`native_binding` 提供 `binding_id`、目标/对话/运行 ID、含新 ID 的目标范围、准确原生 `objective`、原生 `goal_created_at`、当前 `observed_at`。命令 `objective` 必须与原生目标字面相同，并提供常规模型、容量和清单证据。另给 `app_goal_observation`，包括相同目标、对话、目标文字、创建和观测时间、`status: active`、`source_kind: get_goal` 与真实回执 ID。观测不得超过 60 秒，不能用未来时间，创建时间可较早且保留原值。目标文字不要加说明前缀。

`interpretation` 与 `procedural_check` 分别为主代理撰写的非空证据。绑定前核对当前授权和更新的人类限制。不从旧停止、报告或政策消息构建绑定，不虚构 `exact_quote`、`source_message`、`original_turn_id` 或用户来源创建时间。不得有存活或状态未知的子代理。工具校验格式和一致性，主代理取得真实来源证据。

原生记录及防重放历史在人类继续或启用后保留。原生依据仍有效时，每次 `app-goal` 刷新均须包含原目标创建时间、确切目标、目标与对话身份。近期用量、清单、容量、未知派发和分工检查仍适用。原生目标仍活动也不解除辅助程序停止。正常停止、快照、halt 和核验恢复仍可用。非过期的 `resume` 要有 halt 后直接的 `explicit_goal_continuation`，或 `native_ui_resume: true`：halt 后通过 `app-goal` 观测 `paused`/`blocked`，之后用新的 `get_goal` 看到同一目标 `active`，Unix `updated_at` 须严格晚于 stop 和 halt。同秒歧义拒绝。均需检查点、用量和清单。回执只用一次，不能证明操作者是谁。

<a name="expired-telemetry-recovery-under-existing-authority"></a>

### 现有授权下恢复用量观测过期

`usage_receipt_expired` 表示错过动态有效期，不能推出配额已尽。停止实质工作，保存并 halt。candidate 在 `halt` 记录实际的同运行、同轮次监视原因。调用方不要传 `usage_failure`。恢复用批次可停止并保存快照；生命周期 `halt` 和 `resume` 仍为直接命令。

取得 halt 后的真实新用量、同活动目标的 `get_goal`、所属子代理清单，验证检查点、当前文件哈希和进程退出。记录 `workflow usage` 和 `inventory`，再运行 `workflow resume --input RECOVERY.json --usage-input USAGE.json`。`RECOVERY.json` 只含 `usage_recovery: true` 和 `app_goal_observation`。原生绑定观测保留目标原文和创建时间。不需修改 `updated_at`、制造非活动或点击观测，也不用另造用户引语；使用原授权。开启新轮次前校验保存的失败原因与实际 latch 一致，并等健康监视。有限周例外和以往 SAVE 保留。旧任务被拒绝，不自动重启。

额度、提前保留、缺失或无效以及其他测量失败、无原因记录、目标不匹配、用量过时、目标非活动、仍有活跃或未知子代理、检查点改变、尚未被有效继续取代的显式停止，均不走此路。不会改原生目标状态。用户用 Continue 重新激活 paused 或 blocked 目标后，获取实际活动观测再做检查。目标仍获授权且活动时，常规自动目标轮也可修复这一技术中断。不要仅因过期要用户重发继续或设置 `blocked`，遵守原生 `update_goal` 条件。活动的时间戳不能证明谁点击。显式用户停止只能由接受的直接继续取代，或已绑定运行通过确认的非活动到活动转换路径取代；`usage_recovery` 没有这个作用。

没有绑定原生目标的独立监视运行不使用工作流 `usage_recovery`，仍按显式继续和新轮次处理。旧包的 halted 运行如果没有记录原因，不能自动升级或替换标签。保留原固定证据并遵循更新和绑定规则。该技术路径只适用于由此包记录的原因。来源不明就报告缺失。

<a name="authorization-lifecycle-a26--auth01"></a>

### 授权生命周期（A26 / AUTH01）

`authorization` 是明确的事件对象。单纯文本，包括非空的历史 `authorization`，也被拒绝。不导入旧状态；新工作使用当前软件包、全新运行和当前授权。`interpretation` 是独立顶层字符串。保存用户原话、标点不改，不附加主代理解释。独立的 `source_message` 投影须与事件原话及来源完全一致。

继续参数示例。ID 与时间为合成数据，实际操作使用真实观测。

```json
{
  "authorization": {
    "event_id": "user-event-42",
    "kind": "explicit_goal_continuation",
    "exact_quote": "Continue this goal now.",
    "source_kind": "user_goal_continuation",
    "source_id": "user-message-42",
    "goal_id": "goal-7",
    "thread_id": "thread-3",
    "run_id": "run-5",
    "scope": {"kind": "objective", "id": "scope-5", "objective": "Deliver the authorized project change"},
    "observed_at": 1791230401,
    "source_created_at": 1791230401, "original_turn_id": "turn-42", "observed_turn_id": "turn-42"
  },
  "interpretation": "Main classified this source as immediate continuation of the matching objective.",
  "source_message": {
    "source_kind": "user_goal_continuation",
    "source_id": "user-message-42",
    "exact_quote": "Continue this goal now.",
    "source_created_at": 1791230401, "original_turn_id": "turn-42"
  },
  "app_goal_observation": {
    "goal_id": "goal-7", "thread_id": "thread-3", "status": "active",
    "source_kind": "get_goal", "source_id": "actual-goal-tool-receipt",
    "observed_at": 1791230402
  }
}
```

来源类型为 `user_message` 或 `user_goal_continuation`。ID 和引语为非空文本。事件观测期限 300 秒，不得在未来。事件 ID 和来源消息只能消费一次，即使原运行已完成。来源身份是对话 ID 和来源 ID 对两种来源类型统一确定；改类型或事件 ID 不会允许重放。应用目标观测需要相同目标和对话 ID、`get_goal` 来源、非空回执 ID、合法且在 60 秒内的时间，并且不早于事件。实质操作前用 `app-goal` 记录真实近期观测，`task` 和 `prepare` 以当前时间和授权身份重新核验。60 秒仍可用，61 秒、未来值和身份不符均拒绝且不改状态字节。刷新用量或清单不刷新目标。所有字段由调用方提供。Python 检查格式、等值、时间和状态转换，不联系提供者，也不认证转录、分类自然语言或验真回执。主代理取得并保留真实证据，正确分类事件。两条虚构的引语可以相同，不能据此独立证明来源。

| 事件类型 | 新启用 | 已停止旧目标 | 对派发的影响 |
| --- | --- | --- | --- |
| `explicit_task_authorization` | 独立运行和范围，目标匹配且观察到活动 | 不恢复 | 只允许新的授权运行 |
| `explicit_goal_continuation` | 有实际活动证据时可为独立运行和范围 | 所有后续检查通过才恢复 | 成功恢复后可发新任务 |
| `future_goal_intent` | 拒绝 | 拒绝 | 不授权旧任务 |
| `policy_change` | 拒绝 | 拒绝 | 不授权旧任务 |
| `independent_report_request` | 只允许独立 `kind: report` 范围及其准确目标 | 不恢复，也不借原 halted 运行重新启用 | 独立报告运行，旧任务保持停止 |
| `explicit_stop` | 拒绝 | 用 `observe-authorization` 记录相同身份与范围 | 立刻冻结新任务与准备 |

`observe-authorization` 记录未来意向、政策变更、独立报告或匹配范围的停止，不改变当前授权，不启用或恢复目标。对 halted 目标的报告必须用另一套项目和状态，不能覆盖原运行以获取任务。`app-goal` 单独记新观测。暂停、阻塞、停止或已完成的观测冻结活动辅助程序。之后仅看到活动不能证明用户继续；原生界面路径需先看到 halt 后非活动状态。已记录的回执过期可用原授权下的 `usage_recovery`。`status` 分别输出辅助程序 `status` 和 `app_goal_observation`。分工提示带授权范围与目标、对话、运行身份。

直接通过用户消息继续，需要在 stop 和 halt 之后实际观察到新的 `explicit_goal_continuation`，与所有保存身份及范围对应。先读 halt 后真实用量与代理清单。用量须新鲜、留有足够额度、在动态期限内。清单不得有活跃或未知的所属子代理。复核试恢复和当前文件哈希。在 halt 后、不早于继续事件时确认目标活动。降级停止必须先修检查点。恢复成功后拒绝旧未完成任务，允许重新分工，不自动重启旧代理。缺失、过时、重用、损坏或非继续类授权拒绝且不改变状态。

显式停止时，主代理立刻暂停新工作和派发。仅做已有恢复所需的简短所属写入者停止、确认、保存与 halt。不要新增规划或长篇交接，不要在停止阶段搭建新的恢复系统。应用观测与辅助转换分别记录。它不拦截原生工具，来源判断和停止纪律由主代理执行。A26 真正的原生继续或停止回执，以及对未经授权的原生操作的独立审计，需要临时 Python 测试之外的证据。

`stop` 冻结派发，向仍写入的任务返回 SAVE_ACK 请求。每位写入者收一个简短确认。最多等 30 秒；无响应时如有必要只中断所属写入者，并说明内存里可能有未存内容。快照前确认磁盘不再被写。辅助程序本身不发消息、不停进程、不持续轮询。

监视器不健康或已锁定 `SAVE_AND_STOP` 时，固定入口仍提供 `stop`。恢复不要求先取得健康测量。直接 `stop` → `snapshot` → `halt` 和仅恢复批次使用相同恢复白名单。不能借此发新任务、清除 SAVE、续不健康的租约或恢复工作。`halt` 仍校验静止清单、检查点和所属进程退出。

确认写入停止后，`snapshot` 接收 `files`（全部改动、新建、删除的项目相对路径）、`next_action` 和 `writers_quiescent: true`。任务占用路径自动纳入；主代理写过和未跟踪的文件要明确加入。复制时哈希须稳定。清单记录本应不存在或删除的路径和状态副本。在新的内部目录试恢复并对比哈希。不上传凭据。需恢复的外部产物先用对应工具存到本地。

通过宿主工具停止所属代理和进程。`halt` 接收近期 `inventory`、`observed_at`、`owned_processes_stopped: true`，以及实际观测的 `stop_evidence`，再验检查点和当前文件哈希。快照或恢复失败时提供 `degraded: true`、`reason`、`next_action` 和真实停止证据，不声明恢复已验证。`resume` 要符合上述三种路径之一及各项检查。降级 halt 先修检查点。不自动兑换重置或修改原生状态；技术恢复只恢复已经授权、仍活动的目标。

`restore` 把核验过的检查点复制到项目之外的全新目标，项目内容位于 `files/`，恢复元数据在该子树外。清单保留应不存在的路径。不删除或覆盖工作中的文件。检查点持久提交后保留当前与上一个可用版本，只清理归属明确的旧目录，拒绝链接并保留未知数据。`compact` 要求无活跃代理或未完成任务以及已验证快照，再归档已结束任务。`complete` 关闭已交付运行。未来工作重新启用。

<a name="mechanical-coordination-and-durable-dispatch-n05"></a>

## 机械协调与持久派发（N05）

`scripts/candidate.py <global pins/options> coordination --input <request.json>` 是不使用模型的标准库适配器。请求格式 `{ "batch_id": "unique_id", "steps": [{ "key": "unique_step", "command": "task", "payload": { ... } }], "seen": [] }`。最多 64 步，顺序执行，每条命令各自校验、加锁与提交，整个批次非原子。独立读文件用 `read` 及精确项目相对 `payload.paths`。日志保存文本、字节数和 SHA-256，缺失固定值为 null。响应为紧凑决定，列出阻塞、已见会话身份、源码固定值、日志路径、每步结果及结果 token。原始数据、回执和结果留在 `.agent-workflow-v1-1/runs/<run-id>/coordination/<batch_id>.json`。将已消费 token 送入 `seen` 可省略未改变的已交付结果。此适配器不处理原生工具自己的输出。

执行前各步以旧状态哈希记 `pending`，执行后独立记录 `applied` 及新哈希和结果，或 `failed` 及错误和新哈希。写状态后记录结果前崩溃会保留 `pending`，重新执行该批次只返回阻塞，不运行下一步。已应用步骤不重做，失败停止批次并保留之前结果。不得用同 ID 换请求或删除日志。状态没变也不能证明可重试，快照或恢复可能已经写入。查看持久日志、当前状态和实际外部回执，明确解决未知副作用后才开新的有限核对批次。崩溃不自动认作成功。各锁保护自己的提交，仅主代理写入，与其他工具没有全局事务。删除旧锁前检查归属。

日志化的原生路径：

1. 派发前取得并记录实际 `inventory`。`prepare` 返回确切原生参数及在任务中持久保留的 `intent_id`。
2. 即将在原生工具中派发时，用 `id` 和 `intent_id` 运行 `dispatch-start`。复核授权、目标新鲜度、用量与 SAVE、清单、任务和输入、写入哈希。先记录 `uncertain`，再给精确的 `collaboration.spawn_agent` 参数。超时、遗失返回或重启因此会阻止重复派发。调用前崩溃也算结果未知，必须以真实清单解决。原有的 `prepare`、原生调用、`record` 还可用，但没有调用前未知标记。
3. 主代理直接通过支持的工具调用 `collaboration.spawn_agent`，不用 Python 或 `functions.exec`。保留真实结果。近期清单使用 `observation: { "source_kind": "collaboration.list_agents", "source_id": "caller-capture-correlation-id", "evidence_path": "operations/raw-native-inventory.json", "sha256": "<actual-file-hash>" }`。准确原文只连同哈希和项目相对路径保存一次，状态与日志只存引用。替代方式是 `receipt: <actual-native-output>`，原文会在状态和日志重复。两方式只能选一。`dispatch-start` 和迟到成功的核对须有来源投影，仍遵循 60 秒时限和身份、状态检查。核心每次读取复验固定值。`native_inventory(raw)` 规范真实 `{agents:[{agent_name,agent_status}]}`。`running`、`idle` 不变，`interrupted` 转为 `stopped`，确切的 `{completed: <final text string>}` 转为 `done`。拒绝未知、畸形、歧义、重复项。调用方的标准清单与原生投影必须相同，不计排序。最终全文留在证据。
4. 用 `id`、`evidence`、`no_owned_work_running: false`、确切 `submitted_arguments`、真实 `session_id`、实际 `receipt` 解决未知任务，可附观察到的模型和力度。回执须是对象，其 `id`、`agent_id`、`session_id` 或 `task_name` 与清单会话身份一致。新清单须有已知会话状态，观测时间不得早于派发意图。准备参数和不可变输入仍匹配，重用会话拒绝。`Coordination.receipt(...)` 在转换前写真实原回执，CLI 可用同样的 `reconcile`。不合法的来源、格式、身份，或缺失、过时观测均不能把任务改成 running。停止过程中出现的迟到会话以 `stopping` 记录供恢复，不能因此恢复授权或派发。输入失效时保留回执及日志，并用原生工具停止真实所属子代理。
5. 只有验收关口新取得实际终态清单后才能完结，独立审查和结果固定值条件仍有效。关闭和恢复需要时再实际检查。如果实际观测表明没有所属活跃或未知工作，以 `reconcile` 和 `no_owned_work_running: true` 拒绝未知任务，不重试，也不在没有回执时认作成功。

辅助程序核验格式、来源标签、有效时间和内部一致性，不能认证调用者、原生回执或转录。真实证据由主代理取得并保留。如果提供者未给调用或回执 ID，`observation.source_id` 是主代理生成并指向原工具结果的捕获关联号；不能说是提供者发给或认证的。实际派发返回的身份仍须匹配。身份一致不是元数据认证。Prepare、spawn、record 非原子，适配器不拦截工具或隐藏输出。日志 `source_pin` 对 workflow.py 和 coordination.py 的 SHA-256 规范映射取哈希。

`collaboration.list_agents` 支持已知规范任务前缀的 `path_prefix`。每个需要的所属组只取关口观测。未知根先查清单。前缀查询仍可能带完成会话的全部内容。原生工具没有 `compact`、`exclude_completed` 或虚构的筛选参数。本地结果 token 只过滤重复交付的辅助程序结果，不能证明提供者过滤了输出字节。

从仓库根目录运行 `python -X utf8 -B -m unittest discover -s skills/agent-workflow-v1-1/tests -p test_candidate_coexistence.py -v`，测试受支持的源码与暂存入口、受保护批次及生命周期。历史核心直调不能证明 candidate 就绪。两条路径都做相同的任务与输入固定、准备、合成回执、终态、验收、快照验证及关闭。报告真实子进程命令数与退出码、stdout 和 stderr 字节。这只是模拟主代理决策边界，不是实测模型调用数。合成数据不是原生派发证据，串行和批次原生使用还需用实际回执测量。时间、临时路径长度和序列化会影响字节数。

<a name="install-and-update"></a>

## 安装与更新

用 `tooling/build-candidate.py` 输出可复现的软件包、ZIP 和外部构建回执。固定入口的 `install --destination <explicit-directory>/agent-workflow-v1-1` 检查全部字节。相同且归属明确的安装不改动。`--update` 只替换归属明确且未更改的安装，拒绝已编辑或无归属的内容。发布失败会尝试恢复旧目录。成功不会留下旧包或长期回滚档案。具体命令和外部信任依据见[运行绑定](coexistence.md)。

禁用隐式选择。固定的源码、暂存或已安装入口是各运行的执行身份，中途不得交换。监视引擎、角色和说明都在 `private/limit-watcher` 内打包，只有固定入口控制。更新后应启动新获认可运行，不能迁移或重写旧来源。构建或安装不激活 AGWF，也不注册代理。用户为项目准备显式调用时，以上启动要求仍限于相应范围。主客户端模型由用户决定。
