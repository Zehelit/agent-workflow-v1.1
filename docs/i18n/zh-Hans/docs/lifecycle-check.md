<a name="repeatable-lifecycle-check"></a>

# 可重复的生命周期核验

`tooling/run-lifecycle-check.py` 接收实际的目标、用量和子代理清单，通过固定版本的 candidate 处理。观测由主代理从宿主获取。会话与输入文件应私下保存，放在 Git 之外。一个会话仅由一方写入。

<a name="prepare-before-measuring"></a>

## 先准备，再读取用量

构建软件包并保留获认可的外部固定值。`prepare` 校验软件包并写入新的会话记录，不启动 candidate 或监视器。完成准备再取第一条真实用量读数。

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` 来自认可的构建回执。项目应已存在，会话文件必须是新的。更换包后，不要用另一个固定值重新绑定旧会话。

<a name="host-input-object"></a>

## 宿主输入对象

`run`、`observe`、`resume`、`close` 和 `timed-test` 通过 `--input` 接收 JSON 对象。

| 字段 | 内容 |
| --- | --- |
| `activation` | 仅首次 `run`。`bind-active-goal` 完整类型化参数，包含 `native_binding`、匹配的 `app_goal_observation`、目标、解释、程序性检查、实际观测的模型和推理力度、槽位限制、真实会话清单和观测时间。 |
| `app_goal_observation` | 最新 `get_goal`，包括目标、对话标识、目标文字、创建时间和状态。填入 `source_kind: get_goal`、真实来源引用与 `observed_at`，有实际 `updated_at` 时也保留。 |
| `usage` | [原生用量回执](../skills/agent-workflow-v1-1/references/native-usage.md)中的 `source`、`sourceId`、`observedAt`、`result`。只保留必要窗口。 |
| `inventory` | 实际、当前的子代理标识及状态。没有子代理才用 `[]`。 |
| `checkpoint_files` | 待恢复的项目相对路径，含新增、变更、删除及主代理文件。核心还会加入任务登记的路径。 |
| `writers_quiescent` | 真实观测到所有写入者停止后才填 `true`。工具不会产生这个观测。 |
| `next_action` | 恢复或正常关闭后可执行的下一步。 |
| `weekly_exception` | 首次 `run` 的可选字段。使用用户真的说过的 `exact_quote` 和有限 Unix 有效期 `valid_until`。默认没有例外。 |
| `resume` | 可选的完整 candidate 恢复输入。未提供时，只对已记录的测量过期尝试 `usage_recovery: true`。额度或用户停止继续适用各自规则。 |

如果原生 API 没有单独的目标 ID，可一致地使用 `threadId:createdAt` 等派生值并明确说明来源。详细类型约定见[运行指南](../skills/agent-workflow-v1-1/references/operations.md)和[命令](../skills/agent-workflow-v1-1/references/commands.md)。

<a name="run-refresh-and-close"></a>

## 运行、刷新和关闭

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

原生调用返回后立即记录真实观测时间。工具分别检查 `status.at == usage.observedAt`、所属进程、轮次、心跳、样本和租约。`observe` 还会续租。`refresh_by` 提醒提前刷新，不修改真实期限。

收到 `CHECKPOINT_SOON` 时验证已完成工作的检查点，保持 `checkpoint_files` 最新。正常 `close` 要求新的健康观测、已验证快照、完成和已观测进程退出。正常操作不会人为注入 SAVE。

若首次 `run` 有 `weekly_exception`，工具先设置有限的监视规则，返回 `NEED_POLICY_OBSERVATION`，工作流尚未启用。重新取得真实目标和用量观测，再运行同一会话。不要改旧回执时间。

<a name="halt-and-resume-after-expiry"></a>

## 测量过期后的停止与恢复

监视器请求 SAVE 时，`observe` 执行 stop、snapshot 和 halt，返回 `NEED_POST_HALT_OBSERVATION`。停止以后重新获取目标、用量和会话清单。

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

核心使用观测到的停止原因。技术恢复要求同一个获授权、仍活动的目标，文件与检查点未改变，停止后的真实新证据及所属进程退出。额度、提前保留与用户停止需满足各自继续规则。辅助工具不会将原生目标设为暂停、阻塞或活动。

若首次启动失败，会话保留 candidate 的启动尝试哈希。满足重试条件时，可用新证据再次运行同一 `run`。改过的会话不能用作原始证据。已结束和结果不明的尝试不会自动重启。

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## 只在明确要求时做计时诊断

先准备好健康的监视器。`timed-test` 仅用于选定的诊断。事件 JSON 包括真实原生观测中的 `event_id`、`goal_id`、`thread_id`、`source_id`、`event_at` 和 `observed_at`。

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

迟到的事件得到 `INVALID_TIMING`。`SAVE_REQUEST_ON_TIME` 表示 SAVE 请求符合时间条件。原生暂停需要单独的宿主证据，工具仍返回 `native_pause_verified: false` 和 `native_paused_at: null`。完整的两轮实测需在同一目标和运行上观测活动、已停止、经授权继续后的活动。合成时钟只验证逻辑。
