<a name="install-verify-and-maintain-agwf"></a>

# 安装、验证与维护 AGWF

需要 Python 3.11 或更新版本和 Git。构建和运行程序只用 Python 标准库。安装到目标用户可写的技能目录。以下命令使用 PowerShell。在 Linux 或 macOS 上选用相应的 Codex 根目录，并调整 shell 语法。Windows ACL 检查只适用于 Windows。

<a name="build-from-the-selected-checkout"></a>

## 从选定的工作副本构建

在 AGWF 仓库根目录运行。接受构建前记录源提交并检查本地改动。输出目录必须是新的。

```powershell
python --version
git rev-parse HEAD
git status --short
$build = Join-Path $env:TEMP ('agwf-build-' + [guid]::NewGuid().ToString('N'))
python -X utf8 -B tooling/build-candidate.py --output $build
if ($LASTEXITCODE -ne 0) { throw 'AGWF build failed.' }
$receipt = Get-Content -LiteralPath (Join-Path $build 'build-receipt.json') -Raw | ConvertFrom-Json
$pin = $receipt.manifest_sha256
$package = Join-Path $build 'agent-workflow-v1-1'
$entry = Join-Path $package 'scripts/candidate.py'
$manifest = Join-Path $package 'manifest.json'
```

构建器生成软件包目录、同一输入可复现的 ZIP 和构建回执。将认可的回执保存在可变软件包之外，并与源提交证据一起保留。安装程序需要认可的 `manifest_sha256`。意外变更后重新计算的哈希是一次新观测。

[软件包绑定与安装](../skills/agent-workflow-v1-1/references/coexistence.md)

<a name="choose-a-human-or-agent-path"></a>

## 手动安装或请代理安装

下面的命令以当前用户的 Codex 技能目录为目标。做一次性试验时，把目标设为隔离目录，末尾为 `agent-workflow-v1-1`。首次安装和验证未改动的同一软件包使用相同命令。

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` 表示仅安装或验证文件，没有启动工作流。在 Windows 上，暂存目录继承目标父目录的权限。请以实际使用安装的普通用户验证读取和运行权限。

代理安装步骤：

1. 核对用户指定的安装位置，确认是首次安装、更新还是隔离试验。更改现有目标之前检查已安装软件包、用户改动和归属明确的进程。
2. 检查工作副本身份和 Python 版本。在新目录构建并保留获认可的外部清单固定值。
3. 用固定版本的 candidate 安装到指定目标。更新前只停止属于该安装的旧进程，并确认实际退出。安装程序拒绝时保留现有目标供检查。
4. 验证安装的清单和文件。用安装后的入口执行下面的被动检查。报告源提交、清单哈希、安装位置、结果，以及执行的是被动验证还是明确启动的实际工作流。

<a name="verify-the-installed-entrypoint"></a>

## 验证安装后的入口

使用全新的临时项目和运行 ID。命令在该项目写入运行绑定，不启动监视器或目标。

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

检查响应中的 `identity`、`version`、`run_id`、`manifest_sha256` 和 `missing_dependencies`。本文件的隔离试验中安装返回 `activated:false`，验证产生 `run-binding.json`，没有工作流状态文件或监视器目录。错误的清单哈希被拒绝，退出码为 2。

实际的 Codex 调用是后续操作。当前会话可能需要刷新技能目录。显式选择 `$agent-workflow-v1-1` 并给出任务范围。获授权的启动和监视健康检查见[运行指南](../skills/agent-workflow-v1-1/references/operations.md)。

<a name="update-recover-or-remove"></a>

## 更新、恢复和移除

- 更新前用原固定入口停止归属明确的相关监视器，确认进程退出。单独构建和认可新包。只有归属已验证且未被修改的安装可使用 `--update`。先保存并核对用户改动。
- 安装程序替换目录前验证暂存包。替换失败时尝试恢复原目录。如果恢复也失败，保留事务目录。成功更新不保留永久回滚归档。[安装程序细节](../skills/agent-workflow-v1-1/references/coexistence.md)
- 技术标识变更时，在新目录首次安装。旧运行绑定与原清单一起保存；Codex 技能发现配置另行处理。使用获认可的新包时开新的运行。[迁移边界](../skills/agent-workflow-v1-1/references/migration.md)
- 没有卸载命令。移除前确认精确目录、停止其所属进程，并在技能扫描范围外保留已验证备份。保留用户项目、运行状态、检查点和外部回执。移除后另行验证技能发现。
