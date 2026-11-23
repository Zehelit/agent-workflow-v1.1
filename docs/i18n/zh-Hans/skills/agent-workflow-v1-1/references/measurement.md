<a name="scoped-native-usage-measurement"></a>

# 有范围的原生用量测量

N08 自动记录可发布的元数据。它既不调用模型，也不分摊配额或估算价格。产物测试和元数据重建需要经过独立审查。新旧上下文、缓存行为、对照的经济实验和安装采用情况要分别验收。

从源码或安装里的技能目录运行 `python -X utf8 -B tools/extract-usage.py --manifest MANIFEST --output NEW_JSON`。该工具与 `tools/telemetry.schema.json` 由清单固定。API 为 `extract(manifest)` 和 `publish(document, new_path, canaries=())`。输出为规范 UTF-8 JSON，键排序、分隔符紧凑，结尾只有一个 LF。已有文件不会覆盖。`--canary-file` 读取包含需确切匹配的字符串的私有 JSON 数组；不会把它复制到证据。打开输出前核验所有待导出的字符串与键。错误只用固定的理由代码。

清单只有 `runs`、`comparison_pairs`、`checks`、`evidence_pins`。每个运行有 `run_id`、`sources`、`phases`、`coverage_gaps`。来源列出 `source_id`、`path`、`thread_id`、`session_id`、`task_id`、`agent_path`、两端均包含的 `ranges`、`requested_model`、`requested_effort`、`main_client_selected`，以及可选的 `prefix_bytes`、`sha256`。未指定固定值时在打开文件时捕获，指定则须相符。文件继续追加后，用既定字节数和 SHA-256 重现。只接受由完整记录构成的 JSONL 前缀，拒绝重复键和非有限数值。范围有序且不重叠，显式标明来源与记录头身份，响应时间不倒退。缺少范围则拒绝清单。中间缺失或重启令 token 总量为 null，并记录理由。

每个响应的权威元数据来自 `token_usage_record.usage`。按对话、会话、轮次和来源计算，每个响应 ID 只计一次。重复冲突、超出范围、缺字段或无效子集合计会置 null 并保留理由。旧响应跨选定边界重用不会变成新完成。固定前缀其他位置同 ID 的无效观测也会使所选重复失效，包括较早记录。选定的每个时间戳都推动时间水位线，同 ID 的重复也在内。去重不会跳过时间核对。Token 是非负整数，布尔不算。`total_tokens` 为输入加输出。缓存输入与写入缓存的输入是输入的子集，推理输出是输出的子集。不会把子集额外加进原始总量。排除缓存的输入等于输入减去缓存输入。它只是 token 数量。缓存原因缺失则未知。

`event_msg.info.total_token_usage` 与 `token_usage_record.thread_token_usage` 是独立累计流。两者仅用于诊断，差值为 null，说明起点、重置或缺失原因，不能成为归因替代。不会合并或相减。重置诊断不会使完整独立的逐响应记录失效。响应数量统计观测到的完成计量 ID，不是计费的请求数量。

私有转录、推理、工具输入输出正文、账户、资料、配置和不透明消息不在导出范围。读取本地私有工具值仅为计算表示长度和识别原生边界，不导出、不归为隐藏推理、不解密。工具输出需要唯一调用 ID、匹配的范围内调用和有效时间顺序。字符串用原始 UTF-8 长度，结构化值用原插入顺序的紧凑 UTF-8 JSON，设置 `ensure_ascii=False`。这些长度符合已接受的 N05 表示约定，不能当作外层 JSONL、网络或计费字节。

请求的模型和推理力度来自精确的原生请求回执。主客户端由用户选择，主代理的请求字段保持 null。轮次上下文中的模型和力度是观测设置，不能证明提供者实际采用的模型。缺失观测、有效模型、成本和币种保留 null 和原因。工具不引入费率、Standard 的代理估算、缓存未命中解释或将五小时窗口消耗归因到单个操作。比较保留原始 token 和耗时的变化，实际费用仍未知。合成费用反例的统一币种只用于核验更快但更贵的方案仍可见，与真实账单无关。

阶段有 `implementation`、`repair`、`independent_review`、`external_wait` 和 `closure`，各自包含任务和来源、记录的起止时间与授权标识。观测缺口产生 null 时长。生命周期窗口计入等待及轮询，不能证明实际运算时长，`agent_active_seconds_sum` 保持 null。`observed_agent_interval_seconds_sum` 对每个代理的区间先合并再求和，`observed_wall_union_seconds` 跨代理合并区间。外部等待不计入代理时段。区间重叠时两者不同。`coverage_complete` 要求没有声明缺口、token 范围失败、工具覆盖失败或阶段采集和类型缺陷。覆盖不全时 `complete_wall_union_seconds` 为 null，理由包含 `run_coverage_incomplete`。这与可核对的局部 token 总量、已观测的墙钟和生命周期并集独立。阶段与任务、来源的关联必须匹配。另一端缺失时，已知端点也必须是有效时间，缺失仍使时长为 null。观测区间的并集不能代表尚未完整观测的项目总时长。

未来如对照同一工作的新建精简上下文和带公开长历史的运行，主代理建立含两个不同运行 ID 的清单。各实验组须有主会话标记范围、准确的子会话头来源，及实施、修正、审查、等待和关闭区间，再设 `comparison_pairs: [["fresh_compact", "public_long_history"]]`。从已完成的原生观测中提取 `prefix_bytes` 和 `sha256`。回执提供请求的模型和力度，逐响应记录提供原始输入、缓存输入、排除缓存的输入和输出。子代理完整前缀可表示为 `ranges: [[1, RECORD_COUNT]]`。计工具字节时主标记范围必须含调用和结果。缺失关闭与阶段记录用 `coverage_gaps` 和 null 端点表示，不填估算值。`extract(manifest)` 生成各组对齐后的元数据与 token 差。`publish` 在通过封闭 schema、语义和配置的敏感串检查后写入。语义验证重算唯一响应和、子集、原始运算、来源与运行计数、工具字节与关联、来源与范围、仅诊断流、阶段时间和并集、覆盖与 null 与理由的关系，以及对照结果。修改文档后即使形式合法也不能绕过这些关系。验证完才创建文件。工具核对内部关系，不能认证原始元数据来源。任务是否等价、如何处理上下文和解释原生缓存要由主协调者用独立方案验证。提取器不创建原生读取会话，也不宣布实验通过。

`telemetry.schema.json` 封闭所有公开对象。无依赖的验证器实现其声明的 JSON Schema 子集，拒绝未知关键字。普通与优化模式测试均走真实提取与发布路径，生产验证不依赖断言。证据固定值认准实际扫描的文件前缀，不是可变整份文件。元数据固定值证明字节，而不是账单、回执身份或真正运行的模型。

历史实验不构成本工具的依赖。每次新测量需提供自己的固定清单并独立验收。
