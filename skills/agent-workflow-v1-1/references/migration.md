# State compatibility and recovery boundaries

The v1.1 runtime accepts current schema2 state only in an approved, immutable run namespace. Unknown, damaged and legacy state is refused without changing operational state. Old authorization text, archived task receipts and stale goal observations do not grant authority.

The pinned `scripts/candidate.py` entrypoint does not expose a migration command. The core has no legacy state converter or old authorization validator. Automatic import, bootstrap or package rollback from another installation is unsupported. Start a new run after an approved package update. The current run's manifest or helper mapping must remain unchanged.

Current recovery uses the verified checkpoint of the accepted run and new, valid user authority. Preserve source timestamps, replay reservations, file integrity and the observation that writers are stopped. Old stop or continuation text stays historical. The helper does not automatically activate the recovered goal.

The owner of new checkpoints is `agent-workflow-v1-1`. The core's independent default state directory is `.agent-workflow-v1-1`. The supported pinned entrypoint uses `.agent-workflow-v1-1/runs/<run-id>`. Earlier `.agent-workflow` and `.agent-workflow-v1-0` directories are not imported. They remain reserved and cannot be task paths or restore destinations. Current snapshot cleanup leaves their ownership markers intact. Direct core CLI invocation is refused.

## Move the user installation to the 1.1 technical identity

1. Inventory existing workers and writers. Stop only work you own. Before moving an older installation, verify the owned stop and actual process exit.
2. Verify the old package against its ownership record. Check the hashes of a backup outside every discovered skill directory. Preserve any edits before proceeding.
3. Fresh-install the new package to `skills/agent-workflow-v1-1` with its new manifest and external pin. Do not use `--update` against an earlier technical identity.
4. Before changing discovery paths, run `workflow validate` from the new installed entrypoint with a new, unique verification run ID. Compare the returned identity, version, workspace and manifest pin. This creates a `run-binding.json`. It does not start the goal, workflow state or watcher.
5. The new invocation is `$agent-workflow-v1-1`. Once direct validation passes, update Codex's main and role skill paths while preserving existing enabled/disabled and explicit invocation settings. Protect concurrent configuration edits. Recheck owned processes and preserve any active installation another session uses. Archive the old package outside skill discovery after the safe transition, then verify the configuration and backup.
6. Keep identities, paths and hashes in old checkpoints, run bindings and receipts intact. Start a new run with the new package. For relevant unfinished work, supply verified earlier outputs and actual current authority to the new run separately.
