# Install, verify and maintain AGWF

Use Python 3.11 or later and Git. The build and candidate use the Python standard library. Install into a writable skill directory for the intended user. The commands below use PowerShell. On Linux or macOS, choose the equivalent directory under the intended Codex root and adapt shell syntax. The Windows ACL checks are platform-specific.

## Build from the selected checkout

Run this from the AGWF repository root. Record the source commit and inspect local changes before accepting the build. The output directory must be new.

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

The builder produces a package directory, a deterministic ZIP and a build receipt. Preserve the accepted receipt outside the mutable package and keep it with the source commit evidence. The installer expects the accepted `manifest_sha256`. A hash recomputed after an unexpected edit is a different observation. [Package binding and installation](../skills/agent-workflow-v1-1/references/coexistence.md).

## Choose a human or agent path

For the current user's Codex skill root, run the following. To install into a disposable test environment, set the destination to an isolated directory ending in `agent-workflow-v1-1`. A fresh install or a safe no-op uses the same command.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false` means this operation installed or verified package files without starting a workflow. On Windows the staged directory inherits access from the destination's parent. Use the intended normal user to check that the installed files can be read and executed.

For an agent installation:

1. Confirm the requested destination and whether this is a fresh install, an update or an isolated trial. Inventory the current package, user edits and owned workers before changing an existing target.
2. Inspect the checkout identity and Python version. Build into a new directory and retain the accepted manifest pin as above.
3. Use the pinned candidate installer with the requested destination. Stop only the relevant, owned old workers before an update and verify that they exited. An installer refusal leaves the existing target for inspection.
4. Verify the installed manifest and payload, then run the passive check below from the installed entrypoint. Report the source commit, manifest hash, destination, resulting status and whether it was a passive check or an explicitly launched live workflow.

## Verify the installed entrypoint

Use a new disposable project and run identifier. This writes a run binding in the disposable project without starting a worker or activating a goal.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

Check `identity`, `version`, `run_id`, `manifest_sha256` and `missing_dependencies` in the response. In an isolated check for these docs, installation reported `activated:false`, validation created `run-binding.json` without a workflow state file or watcher directory, and the wrong manifest hash was refused with exit code 2.

The next real Codex invocation is separate. The currently open session may need a refreshed skill catalog. Select `$agent-workflow-v1-1` explicitly and supply the intended task scope. Use the [operations guide](../skills/agent-workflow-v1-1/references/operations.md) for authorized startup and watcher health.

## Update, recover or remove

- For an owned update, stop the relevant worker through its existing pinned entrypoint and verify exit. Build and accept the new package separately. Add `--update` to the install command only for an owned, unchanged installation. User edits need reconciliation first.
- The installer verifies the staged package before replacement and attempts to restore the previous directory if publication fails. If restoration also fails, retain the transaction directory. A successful update does not keep a permanent rollback archive. [Installer details](../skills/agent-workflow-v1-1/references/coexistence.md).
- A different technical identity requires a fresh install in the new directory. Keep old run bindings with their original manifests and resolve the installed skill discovery configuration separately. Start a new run after an accepted package update. [Migration boundaries](../skills/agent-workflow-v1-1/references/migration.md).
- There is no uninstall command. Before removing an installation, confirm the exact skill directory, stop its owned workers, and keep a verified copy outside skill discovery. Preserve user projects, their run state, checkpoints and external receipts. Check skill discovery after the change.
