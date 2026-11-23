<a name="native-usage-receipts"></a>

# 原生用量回执

仅在已明确授权并启用的 AGWF 运行中使用。用户在当前会话显式调用技能后，主代理需要在授权范围内启动。如果不允许创建原生目标，按[运行指南](operations.md)启动独立监视器。维护软件包、讨论技能或写入用量记录不会自动重启已关闭的工作流。

主代理通过现有 Codex 连接调用 `mcp__codex_app__get_usage_limits`。Python 不会启动第二个 app-server、选择其他 CLI、读取身份验证文件、连接私有通道或调用模型。后台监视器校验传入回执的有效期。主代理仍需在工作阶段边界调用原生工具。

从实际工具结果建立私有 JSON 输入。

```json
{
  "source": "mcp__codex_app__get_usage_limits",
  "sourceId": "actual-tool-call-receipt-id",
  "observedAt": 1800000000,
  "result": {
    "rateLimits": {
      "limitId": "codex",
      "primary": {"usedPercent": 25, "windowDurationMins": 300, "resetsAt": 1800010000},
      "secondary": {"usedPercent": 30, "windowDurationMins": 10080, "resetsAt": 1800600000}
    }
  }
}
```

上面的数字只是示例。保留工具的实际观测时间、回执标识和值。不要虚构，也不要用文件写入时间代替观测时间。实际结果包含多组配额时优先使用 `rateLimitsByLimitId.codex`。如果多组映射缺少 Codex，就拒绝该输入。两组时间窗口的长度与重置时间均必填。私有输入只复制必要的额度字段，不需要账号、个人资料或积分信息。生成的 `watcher/usage-source.json` 仅保留允许的窗口字段、观测与来源标识以及绑定的运行和监视轮次。

`workflow activate --input AUTH.json --usage-input USAGE.json` 和 `bind-active-goal` 在启动进程前验证授权。`resume` 也要用 `--usage-input` 提供新的回执，不能重新使用旧轮次的回执。手动 `watcher start --usage-input USAGE.json` 采用相同的预检。缺少回执或回执已过期时不会创建进程。

工作中通过 `watcher ingest --usage-input USAGE.json` 或工作流、协调命令的 `--usage-input` 发布新观测。`ingest` 仅写入数据，不会启用工作流、启动进程、延长授权或清除 SAVE 记录。新源文件会触发评估，无需等待原期限。所属监视器读取文件期间，状态可能仍显示上一次评估。派发重要任务前，确认新样本和进程状态均正常。

主代理需在动态期限前刷新读数，最长间隔 300 秒。接近阈值或消耗加快时，间隔缩短。重新读取旧文件不会产生新观测。若没有新回执，监视器再次核验一次，然后锁定 SAVE_AND_STOP。对于记录为 `usage_receipt_expired` 的停止，可在同一个已授权、仍活动的目标上用 `workflow resume` 和 `usage_recovery: true` 走[技术恢复](operations.md#expired-telemetry-recovery-under-existing-authority)。仅有新数据不足以清除停止记录。满足技术恢复条件不需要再发一条用户消息。额度耗尽和用户明确停止，仍按各自的继续规则处理。应在工作阶段边界留足下一次工具调用时间，期限不是建议等待时长。不可中断的调用过长时，主代理可能来不及取新观测。后台进程没有自动调用原生工具的能力。

拒绝格式错误、运行或轮次不匹配、未来或已过期的观测、倒退时间戳，以及内容不同的重放。重复提交完全相同的回执不会改变状态。时间和来源由调用方提供，JSON 结构不能认证提供者。私下保存原始证据。来源标签本身不是独立证明。

测量失败会在样本和状态中给出允许的 `diagnosticCode` 和错误类别。`watcher/diagnostics.json` 最多保留 20 条结构化记录，排除异常原文、认证信息和工具正文。意外的 Python 进程错误另写入本地 `worker-stderr.log`。不再启动第二个 Codex 服务，因此没有该服务的标准错误输出。

每次 candidate 命令和监视器评估都会完整校验包，包括新发布的输入。心跳只校验运行和监视轮次的控制信息并写入状态，不再遍历和哈希整个包。依赖变化在下一个命令或测量边界遭拒绝，无法保证每次心跳都检测到。

单元测试覆盖实际回执文件的写入与读取、严格 JSON、过期、重放、身份匹配、阈值、有界诊断和观测路径不会创建进程。实时 Remote 仍需单独验收。用户再次明确启用前，已安装的工作流保持禁用。维护期间不要用真实 AGWF 运行来制造验收结果。
