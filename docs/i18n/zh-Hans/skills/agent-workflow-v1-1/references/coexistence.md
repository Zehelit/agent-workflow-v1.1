<a name="pinned-runtime-binding-and-installation"></a>

# 固定运行绑定与安装

技术标识为 `agent-workflow-v1-1`，显示名为 `agent-workflow-v1.1`。隐式调用已禁用。只有本次明确选择才使用这个 candidate。主客户端模型由用户选择。工作流角色限 `gpt-6.1-sol`、`gpt-6-astra` 或 `gpt-6.1-terra`，推理至少为 `medium`。私有监视顾问固定用 Terra/medium，无自动替代。

当前明确选择后，主代理需要在同一轮启动并验证所选范围的运行。这可以重新启用此前关闭的同一运行范围。过去的选择和维护包不算新的启用。如果缺少完整启用证据或禁止创建原生目标，使用[运行指南](operations.md)中的独立监视启动。确认启动前不要开始实质工作或改为无监视执行。之后的用户停止指令优先。

使用 Python 3.11+ 和标准库构建。

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

输出为 `agent-workflow-v1-1/`、可复现 ZIP 和 `build-receipt.json`。在可修改的软件包之外，把已接受回执的 `manifest_sha256` 保存在协调或交接证据中，由调用方提供的这个固定值是信任依据。运行绑定、重要状态写入、协调步骤和每次测量前都会检查清单及依赖。心跳只核对小型的运行与轮次控制，不会每秒遍历整个包。重新计算被修改的清单哈希并提交新值，不证明获得外部批准。Python 核验字节与身份一致性，无法认证用户同意、提供者和模型元数据，也无法保护完全由调用者控制的文件系统。若解释器已被篡改，或调用者同时改动绑定和获认可的外部哈希，超出此完整性约定。

全局选项写在模式前。源码执行使用 `skills/agent-workflow-v1-1/scripts/candidate.py` 和构建清单；已安装执行使用安装目录下的 `scripts/candidate.py`、对应清单以及同一保留的外部哈希。不同来源或安装入口各用自己的运行 ID。不可变绑定包含解析后的依赖路径、清单路径、项目、运行、版本和哈希。更换入口对应关系、替换包、更改或缺少依赖、用不同固定值复用同一运行均会被拒绝。接受更新后开启新运行。

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

已授权的 `workflow activate` 或 `bind-active-goal` 启动监视器，并验证由 `--usage-input` 提供的首个健康样本。有效的继续，或在原有活动目标授权下核实回执过期后的恢复，可以通过 `resume` 使用新的原生用量回执开启新轮次。详见[运行指南](operations.md)和[原生用量](native-usage.md)。测量不启动第二个 Codex 服务或 CLI。活动命令续租，`complete` 和 `halt` 停止所属 Python 进程并验证退出。独立 `watcher` 命令使用同一个回执约定。

`workflow` 接受核心类型化命令和 `validate`。命令从 `--input` 或 stdin 读取 JSON。旧核心 `native/install/export/migrate` 入口不受支持。`activate` 的授权与 `bind-active-goal` 的证据必须包含所选运行 ID。`validate` 核验角色与依赖并绑定命名空间，但不会激活目标。批次示例为 `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`。一个批次至多 64 步，仅主代理写入，持久记录未决结果，不会自动重试或通过宿主派发。实质命令仍需当前类型化授权与验收。终态清单并不验收文件。写入已停止的过时成果可用 `finish accepted:false` 丢弃；有活跃写入者则不能。压缩需要已验证的检查点，并会归档任务和失效历史。

隔离根目录是 `<project>/.agent-workflow-v1-1/runs/<run-id>/`。运行状态目录是保留路径，不属于任务写入范围。集成监视器在其中使用 `watcher/`、固定的调度辅助程序、不可变的进程归属和 candidate 入口重启参数。其他运行的控制记录或绕开固定入口的私有 CLI 无权续租或停止别的进程。静置的聊天不会续租。在工作边界调用活动 candidate 命令。包更新不会迁移正在运行的进程。没有另一个监视技能或启动器，私有依赖从已认可的包加载。

首次安装、未变更安装的验证和归属明确的更新，都要求明确提供以 `agent-workflow-v1-1` 结尾的目标。

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

更新要求现有安装归属明确且文件没有被改过。安装器先验证暂存软件包，再替换目录。发布失败会尝试恢复原目录；更新成功会清理事务材料。既无长期回滚归档，也无回滚命令。恢复若也失败，事务目录保存原软件包的唯一副本，核实恢复之前应保留。命令仅准备文件，返回 `activated:false`。技能目录可见性、原生显式选择、提供者实际使用模型的元数据和真实工作验收，均需安装后分别验证。

Windows 上，安装器在复制软件包前重设暂存根目录的 ACL，使其继承目标父目录的访问权限。它不硬编码账号。新账号或改名账号采用所在 `skills` 父目录的权限。父目录必须允许目标用户读取及执行。安装后用普通、非提权的 Codex 会话验证技能发现和直接运行 candidate；只在提权会话成功不够。
