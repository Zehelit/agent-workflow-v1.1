<a name="command-contract-a02"></a>

# 命令约定（A02）

标准库辅助程序仅校验经由 `Workflow.execute` 和 `Workflow.apply` 的命令。主代理负责调用原生工具、识别回执和用户来源、停止进程及观测实际模型，并提供证据。A26 授权、检查点、归属与容量检查仍生效。

状态使用整数 `schema_version: 2`。在写入前拒绝无版本、旧版、未来版本和损坏状态。没有旧格式迁移或导入命令。用当前类型化授权开启新运行。仍支持保存和恢复检查点，历史证据本身不授权。故障注入仅对临时测试副本使用。

<a name="json-and-error-policy"></a>

## JSON 与错误处理

状态、`--input`、stdin、检查点元数据和安装归属 JSON 共用严格解析器。任何深度的重复键均拒绝，即使值完全相同。拒绝 `NaN`、`Infinity`、`-Infinity`，以及表示形式合法但溢出成无穷值的数，例如 `1e999`。输入采用 UTF-8，可带 BOM。命令的 JSON 根必须是对象。空 stdin 提供 `{}`，仅在命令允许时有效。`write_json` 在创建目录或临时文件前以 `allow_nan=False` 校验并序列化。

参数对象的键必须是字符串，列表保持列表，整数和浮点数均不是布尔值。已提供的标志要精确为 Python `bool`，即 JSON 的 `true` 或 `false`。字符串、0/1、null、列表和对象均不能替代。必填标志缺失会被拒绝。可选标志仅在省略时默认为 false，null 无效。数值必须有限，计数为大于等于 1 的整数，时间戳为 [0, sys.float_info.max] 内的整数或浮点，百分比在 [0,100] 之间。时间上限避免在时钟运算中溢出，还须满足新鲜度。容量规则不增设上限，原有槽位仍包括主代理，并保留子代理上限。

在建目录、加锁和执行前校验输入。`apply` 在私有 JSON 副本上操作，通过检查后才提交。除了专门持久保存的 INPUT_INVALIDATED 与 contract_invalid 标记，命令被拒绝时保持原状态字节和 SHA-256，首次拒绝不创建状态。失败的快照仍可能留下未提交的所属目录。格式合法但不可用的用量按专门规则变更状态。

辅助程序命令错误以 1 退出，并在 stderr 返回类似对象。

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| 代码 | 含义 | 字段 |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | 核心命令未知 | `$.command` |
| `FIELD_REQUIRED` | 缺少必填参数或工具选项 | 具体缺失字段 |
| `FIELD_UNKNOWN` | 未知参数或该命令不支持该 CLI 选项 | 具体字段 |
| `FIELD_TYPE` | 输入类型不符 | 字段或列表下标 |
| `FIELD_VALUE` | 标识、必填内容、数值范围或清单状态非法 | 字段或下标 |
| `JSON_DUPLICATE_KEY` | JSON 成员重复 | 具体成员，含嵌套 |
| `JSON_NONFINITE` | 出现非有限值，包含原始元数据 | 字段或下标 |
| `JSON_SYNTAX` | JSON 格式或编码错误 | `$` |
| `JSON_TYPE` | 非 JSON 值或非字符串键 | 具体值或对象 |
| `COMMAND_REFERENCE` | 所引用的任务或必需状态字段不存在 | `$.id` 用于查找，错误文本给出缺少的键 |
| `COMMAND_CONSTRAINT` | 授权、状态、归属、审查或新鲜度限制 | 能定位时给出字段，整体状态检查为 `$.command` |
| `STATE_VERSION` / `STATE_TYPE` | 版本不支持或状态根不是对象 | `$.schema_version` / `$` |
| `IO_ERROR` / `INVALID_VALUE` | 文件系统错误或其他值错误 | `$` |

简单键用点号，否则用方括号；列表用 `[index]`。argparse 的未知选项、命令或缺值错误沿用自身 stderr 和退出码 2。已识别但不适用的选项给出结构化 `FIELD_UNKNOWN`。校验均不依赖 `assert`，在 `python -O` 下也执行。

<a name="payload-types"></a>

## 参数类型

`text` 验证去除首尾空白后非空，保存原文；`string` 可空。`task_id` 符合 `[a-z][a-z0-9_]{0,63}`。`strings` 是非空字符串的列表，列表可空；`criteria` 至少一项。`task_ids` 是任务 ID 列表。`timestamp`、`count`、`percent` 遵循上述数值规则。只有标为 `?` 的类型接受显式 null。

各命令与嵌套对象仅接受声明的字段。两类原始数据不解释：回执正文及 `rateLimits`、窗口内的非测量元数据。仍拒绝重复键、非有限数值及非 JSON 值。未知元数据不能当作标志或有效用量。

| 类型 | 必填字段 | 可选字段或限制 |
| --- | --- | --- |
| `authorization` | `event_id`、`exact_quote`、`kind`、`source_kind`、`source_id`、`goal_id`、`thread_id`、`run_id` 为文本；`scope` 为范围；`observed_at`、`source_created_at` 为时间；`original_turn_id`、`observed_turn_id` 为文本 | 无额外字段。A26 限制事件类型、直接用户来源、范围一致性、重放及年龄。 |
| `scope` | `kind`、`id`、`objective` 为文本 | `kind` 按 A26 为 objective 或 report。 |
| `source` | `source_kind`、`source_id`、`exact_quote`、`original_turn_id` 为文本；`source_created_at` 为时间 | 与授权的来源投影一致。 |
| `native_binding` | `binding_id`、`goal_id`、`thread_id`、`run_id`、`objective` 为文本；`scope` 为范围；`goal_created_at`、`observed_at` 为时间 | 只供 N02 首次绑定。范围是 objective 且与目标完全相同。创建时间在观测前，不含人类来源字段。 |
| `app_goal` | `goal_id`、`thread_id`、`status`、`source_kind`、`source_id` 为文本，`observed_at` 为时间 | 可填 `objective` 文本及 `goal_created_at` 时间；以原生绑定为当前依据时，每次刷新都须提供且匹配。A26 另要求 get_goal 来源、身份匹配、年龄 ≤60 及适当的活动状态。 |
| `inventory` | 对象列表，每项 `id`、`status` 为文本 | 无额外字段，ID 唯一。状态可为 running、interrupting、unknown、idle、stopped、done；观测年龄 0 至 60 秒。 |
| `models` | 模型名称映射到推理力度 `strings` 列表 | 对象或列表可空；prepare 仍要求观测到精确模型及力度。 |
| `arguments` | 结构无必填字段 | 只接受 `task_name` 任务 ID、`fork_turns` 文本、`message` 字符串、`model` 和 `reasoning_effort` 文本。记录时必须与准备好参数完全相同，缺项会被拒绝并不能派发。 |
| `receipt` | 非空对象、列表或字符串 | 实际工具回执为不解释的 JSON，不声明其真实性。 |
| `rate_limits?` | 对象或 null | 可有 primary、secondary 窗口，额外元数据保持原样。 |
| `window?` | 对象或 null | `usedPercent` 百分比、`windowDurationMins` 计数和 `resetsAt` 时间可选，其余元数据不解释。 |

<a name="engine-commands"></a>

## 核心命令

这里列出全部必填及可选字段。没有列出可选字段时，只接受必填项。空命令用 `{}`。

| 命令 | 必填参数 | 可选值、默认项与条件 |
| --- | --- | --- |
| `activate` | `objective` 文本，`authorization` 授权，`interpretation` 字符串，`source_message` 来源，`app_goal_observation` 目标观测，`models` 模型，`slot_limit` 计数，`inventory` 清单，`observed_at` 时间 | `session_limit` 可选计数，缺失或 null 不加开放会话上限。 |
| `bind-active-goal` | `objective` 文本，`native_binding` 原生绑定，`interpretation`、`procedural_check` 文本，`app_goal_observation` 目标观测，`models` 模型，`slot_limit` 计数，`inventory` 清单，`observed_at` 时间 | `session_limit` 可选计数。只接受完全初始且未激活的 schema2，没有任务或历史。 |
| `observe-authorization` | `authorization` 授权，`interpretation` 字符串，`source_message` 来源 | 接受 `app_goal_observation` 投影，但不刷新已保存的观测。 |
| `app-goal` | `app_goal_observation` | 无。 |
| `usage` | 无。 | `observed_at` 时间和 `rateLimits` 可空测量对象，省略表示测量不可用。 |
| `usage-due` | 无。 | `substantial_work` 布尔，默认 false。 |
| `inventory` | `inventory` 清单、`observed_at` 时间 | 无。 |
| `task` | `id` 任务 ID，`role` 文本，`objective` 文本，`criteria` 条件 | `file_inputs` 文件输入 []，`context_refs` 文本列表 []，`review_targets` 审查目标 []，`revision` 文本，如无则生成 UUID；`writes` 文本列表 []，`depends_on` ID 列表 []，`independent_review` 布尔 false。 |
| `prepare` | `id` 任务 ID | `effort` 缺省使用角色默认值，`current_context` 为 N06 的 context_pin。 |
| `record` | `id` ID，`submitted_arguments` 参数，`session_id` 文本，`receipt` 回执 | `effective_model`、`effective_effort` 可空文本，缺失或 null 表示未核验。 |
| `dispatch_failed` | `id` ID，`evidence` 文本 | `certain_not_started` 布尔 false，缺失或 false 表示结果不明。 |
| `reconcile` | `id` ID，`evidence` 文本，`no_owned_work_running` 布尔 | 要求 true 和没有活动子代理的近期清单。 |
| `finish` | `id` ID，`accepted` 布尔，`evidence` 文本 | 验收需要独立审查的任务，须填 `review_task` 任务 ID。 |
| `cancel` | `id` ID，`reason` 文本 | 仅可取消尚未派发的就绪任务。 |
| `stop` | 无。 | 无。 |
| `snapshot` | `writers_quiescent` 布尔，`next_action` 文本 | `files` 文本列表 []。布尔须为 true。 |
| `restore` | `destination` 文本 | 项目以外的新目标。 |
| `halt` | `inventory` 清单，`observed_at` 时间，`owned_processes_stopped` 布尔，`stop_evidence` 文本 | `degraded` 布尔 false，为 true 时须有 `reason` 和 `next_action` 文本。进程退出标志必须为 true。 |
| `resume` | `authorization` 授权，`interpretation` 字符串，`source_message` 来源，`app_goal_observation` 目标观测 | 无附加项。需要匹配的停止后继续授权、近期用量及清单、当前已验证检查点。 |
| `compact` | 无。 | 任务须已验收或拒绝，没有活动代理，有当前检查点。 |
| `complete` | 无。 | 在 compact 的限制外，还要求活动运行。 |
| `status` | 无。 | 必须存在已激活状态。 |

文件占用声明始终是精确的项目相对文件，沿用 Windows 别名、路径边界和链接检查。角色及推理设置沿用目录与可用性检查。空写入清单支持只读验证者及协调者。未知任务会报错，不会自动创建。`accepted: false` 拒绝已停止写入的结果。`accepted: true` 仅在通过适用的审查限制时接受。

<a name="raw-usage-lifecycle"></a>

## 原始用量与运行状态

N06 `context_pin` 只有 `{path, sha256, manifest_path, manifest_sha256}`。路径是非空项目相对路径，哈希用小写。清单须是现有不可变任务输入。`prepare` 用当前状态与从独立固定的清单核验新的 version1 摘要。通过摘要形成的准备参数绑定此约定，`dispatch-start` 可接收 `current_context`，但必须根据准备后的状态和原清单重新生成。原生限制始终有效。参见[上下文](context.md)。

N07 `status` 保留原任务状态表，并添加 `task_observations: {contract_version: 1, tasks: ...}`。原生执行完毕与产物验收不同，done 要待 finish 验证。独立 finish 保存可选 `review` 关联，要求不同且非空的原生会话标识、匹配的已分配和已记录产物哈希。schema2 不变。审查不能直接或间接依赖其固定作者先获验收。历史独立验收缺完整标识或固定值时，显示 `historical_unrecorded`，不能让新依赖通过 prepare、record 或 dispatch-start。摘要阻塞与辅助程序用同一独立条件，不自动迁移历史。

错误类型或范围不写状态，例如 `usedPercent: "10"`、布尔计数或时间、百分比 101、负重置时间、列表窗口或非有限元数据。缺失或 null 的 `rateLimits`、窗口和测量值，少 300/10080 分钟窗口之一，或其他格式合法但过时或未来的观测，表示用量不可用。这样类型有效的 `usage` 会清除旧测量、增加 `usage_errors`，第一次是 `RETRY_USAGE_ONCE`、第二次是 `SAVE_AND_STOP`。未 halted 或 complete 时进入停止状态，绝不会自动恢复。百分比 0、100 和有限小数都是有效测量；100 触发阈值处理。格式错误不增加未知读数计数。保留原始元数据，但不可替代真实窗口。

<a name="supported-pinned-entry"></a>

## 受支持的固定入口

操作均使用 `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1`。运行模式另需 `--project <absolute-project>` 和 `--run-id <fresh-run>`。

| 模式 | 输入 | 行为 |
| --- | --- | --- |
| `workflow validate` | 无 | 校验认可的软件包与角色政策，绑定未激活的运行。 |
| `workflow <typed-command>` | `--input <json-file>` 或 stdin | 应用上述类型化约定。 |
| `coordination` | `--input <json-file>` | 执行有界无模型批次，不自动调用原生派发。 |
| `watcher` | `start`、`renew`、`stop`、`status`、`once` | 运行固定的私有采样器，操作所属运行。 |
| `install` | 显式 `--destination .../agent-workflow-v1-1`，可选 `--update` | 验证包与安装归属条件。 |

直接调用 `workflow.py` CLI 会被拒绝。它没有安装、导出或原生角色生成接口。用 `tooling/build-candidate.py` 构建，只用固定安装入口安装。旧 schema 转换不是 candidate 命令，不能用来重新启动退役安装。

<a name="input-and-state-boundary"></a>

## 输入与状态边界

正确布尔输入在临时重放中保留决策。可选 false 默认值可用，但不接受将错误类型解释为真。状态解析要求合法 JSON 与有限值。旧状态和字符串授权不接受。测试用临时项目。

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## 类型化分工与来源时序（A04 / GI01）

N02 `bind-active-goal` 在长时间准备后绑定此刻的活动原生目标，不生成或重写用户来源时间。状态必须不存在，或恰为 `{schema_version: 2, status: inactive, tasks: {}}`，实际 JSON 文本用引号。已有迁移、完成、停止、halted 或发生过操作的辅助程序均不接受。首条真实 `get_goal` 必须在 0 至 60 秒内，目标、会话、目标文字、原创建及观测时间匹配。原创建时间可以很早，但不能在观测之后。命令目标等于目标范围，拒绝报告或扩大的范围。非空、独立的解释与程序性检查说明主代理看过当前上下文和更新的限制，工具不能认证该检查。

状态保存当前依据 `native_binding`，以及独立于人类授权的 `native_binding_events`、`native_binding_ids`。事件记录 `binding`、`interpretation`、`procedural_check` 和原 `app_goal_observation`。绑定 ID 为以后的人类事件保留，运行和范围 ID 由已有历史保留。实际操作需要目标和创建时间均匹配的近期观测。人类授权保留准确来源 schema 和原来源 300 秒期限。人类继续授权替换当前原生依据但不删除历史，之后的启用也保留历史。缺失、重复、损坏或矛盾的原生记录与保留项，在继续后仍拒绝。Stop、snapshot 和 halt 保持原规则。新的活动观测不会自动恢复已停止的辅助程序。

Schema2 拒绝旧任务字段 `inputs`。旧文本可明确保存在 `context_refs`，其内容不作为任务或文件权限。`file_inputs` 是精确的 `{path, revision}` 列表。准备任务时规范化项目相对路径，拒绝忽略大小写后的重复、越界、保留路径与链接，并固定 `sha256` 与 `present`，null/false 表示不存在。不可变输入不得与任务负责的写入文件重叠；写入文件另保留派发前的基准哈希。任务具有修订和覆盖完整约定的分工哈希，不能靠重新计算哈希来验收旧结果。

`review_targets` 是只允许验证者使用的 `{producer_id, revision}` 列表。作者必须存在于指定修订，分配及审查时均有近期 idle、done 或 stopped 原生观测。分配固定作者输出的哈希和存在状态。上下文引用不是审查授权。准备参数保留全部固定值。Prepare、record、验证者 finish 和作者验收时重新检查输入、作者约定、输出及已验收依赖。A05 运行与验收状态机不变；需独立审查的作者仍可能在状态上 running，但已观测为停止写入。

过时约定抛出 `INPUT_INVALIDATED`，并记录 `invalidation` 回执，包含任务 ID、分工修订、原因、适用时的具体路径、预期和实际哈希或修订、时间及排序后的受影响任务 ID。同一路径的其他任务只有在自己的固定值、文件存在或分工约定与观测不符时才被纳入。固定当前字节或当前不存在的较新合法修订不受旧固定值失效连带影响。再沿实际依赖和审查关系传播。单有上下文或相同路径不建立关系。任务工作状态不变；`contract_invalid: true` 和持久记录阻塞旧产物或审查的验收。恢复旧字节不能复活验收，重复检查保留原字节与回执。

输入和所审查、依赖的输出在哈希前拒绝符号链接、联接、硬链接、目录替换和越界。在审查分配、prepare、record、finish 和作者验收中，如观测到不安全的产物，先标记作者与受影响的审查和依赖任务失效再返回。回执保存预期哈希（或 null）、相对路径，`actual` 以 `unsafe: ` 和受控原因开头。文件缺失为 null。EIO 等独立 I/O 故障直接传播，不说成约定漂移。此措施只检查观测路径，防不了跨文件系统的并行外部替换。

舍弃无关的中间更改。直接 `apply` 只在传入状态留下失效记录，execute 在报错前原子持久化这些记录。没有重置或重新计算哈希命令。检查回执，再分配新的任务 ID、修订和固定值。其他失败遵循原状态字节保护规则。

GI01 要求 authorization 和精确来源投影 source_message 都含 source_created_at 和 original_turn_id，observed_at 与 observed_turn_id 单独描述观测。来源早于观测，新接受时不超过 300 秒。继续时来源的创建时间须晚于 stop、halt 或 complete，不能用观测时间替代。最后获认可的来源时间水位线跨启用和继续保留。创建时间不能倒退，相同时须是同一原始轮次。事件 ID 和原来源 ID 跨类型、观测时间、运行、会话和继续保留防重放。观测匹配当前目标、运行和会话；显式停止也匹配目标范围。未来意图、政策更改、报告请求只留证据，不授权现有目标执行或继续。用量阈值只使本地辅助程序进入停止，不改变应用目标观测，也不生成用户暂停授权。

这些字段由调用方提供，不是提供者认证。工具既不拦截原生目标工具，也不调用 update_goal。历史停止、新继续、同轮上下文重放的回归测试说明辅助程序拒绝违规且保留活动观测。N07 添加上述版本化的执行与验收投影。原生新子代理的上下文与缓存测量、A06/GI02 暂停集成，需要各自的真实证据。
