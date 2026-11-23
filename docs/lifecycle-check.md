# Repeatable lifecycle check

`tooling/run-lifecycle-check.py` processes real goal, usage and child-inventory observations through the pinned candidate. The main agent obtains those observations from the host. Keep the session and inputs private, outside Git, and give the session a single writer.

## Prepare before measuring

Build the candidate and preserve the accepted external pin. `prepare` checks the package and records a new session without launching the candidate or watcher. Finish preparation before collecting the real first usage reading.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256` comes from the accepted build receipt. Use an existing project and a new session file. After a package change, do not rebind an old session to a different pin.

## Host input object

`run`, `observe`, `resume`, `close` and `timed-test` accept a JSON object through `--input`.

| Field | Contents |
| --- | --- |
| `activation` | First `run` only. The full typed candidate payload for `bind-active-goal`, including `native_binding`, matching `app_goal_observation`, objective, interpretation, procedural check, observed models and efforts, slot limit, actual inventory and observation time. |
| `app_goal_observation` | A current `get_goal` observation of goal and thread identity, objective, creation time and status, with `source_kind: get_goal`, a real source reference and `observed_at`. Include actual `updated_at` when available. |
| `usage` | The `source`, `sourceId`, `observedAt` and `result` fields from [native usage receipts](../skills/agent-workflow-v1-1/references/native-usage.md). Retain only the required windows. |
| `inventory` | Real current child identities and statuses. Use `[]` only if there are no child agents. |
| `checkpoint_files` | Project-relative changed, added and deleted paths to recover, including the main agent's files. The core also includes task file claims. |
| `writers_quiescent` | `true` only after observing that all writers stopped. The harness cannot create this observation. |
| `next_action` | Meaningful next step after recovery or normal close. |
| `weekly_exception` | Optional on first `run`. Actual user `exact_quote` and bounded Unix expiry `valid_until`. No exception is configured by default. |
| `resume` | Optional full candidate resume payload. Otherwise the harness tries `usage_recovery: true` only for recorded measurement expiry. Quota and user stops keep their separate requirements. |

If the native API supplies no independent goal ID, use a consistently derived identity such as `threadId:createdAt` and label it as derived. See [operations](../skills/agent-workflow-v1-1/references/operations.md) and [commands](../skills/agent-workflow-v1-1/references/commands.md) for the typed contracts.

## Run, refresh and close

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Record the actual observation time as soon as the native call returns. The harness checks `status.at == usage.observedAt`, owned worker identity, epoch, heartbeat, sample and lease. `observe` also renews the lease. `refresh_by` is an earlier refresh suggestion and leaves the actual deadline unchanged.

At `CHECKPOINT_SOON`, verify a checkpoint for the finished work. Keep `checkpoint_files` current. Natural `close` requires fresh monitoring, verified snapshot, complete and observed worker exit. Normal operation does not inject a SAVE event.

A first `run` with `weekly_exception` installs the bounded watcher policy and returns `NEED_POLICY_OBSERVATION` before workflow activation. Take a fresh goal and usage observation, then rerun the same session. Keep every previous receipt's original timestamp.

## Halt and resume after expiry

When the watcher requests SAVE, `observe` applies stop, snapshot and halt, then returns `NEED_POST_HALT_OBSERVATION`. Obtain new goal, usage and inventory observations after halt.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

The core uses the observed watcher cause. Technical recovery requires the same authorized active goal, unchanged files and checkpoint, fresh post-halt evidence and verified owned-worker exit. Quota, early-reserve and explicit user stops require the applicable continuation. The harness does not change the native goal to paused, blocked or active.

For a failed first activation, the session retains the candidate's startup-attempt hash. Use the same `run` with new evidence only under the supported retry conditions. A modified session is not accepted evidence. Completed and uncertain attempts are not automatically restarted.

## Run a timed diagnosis only when selected

Prepare a healthy worker before the event and use `timed-test` only for the explicitly requested diagnosis. Event JSON carries `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at` and `observed_at` from a real native observation.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

A late event produces `INVALID_TIMING`. `SAVE_REQUEST_ON_TIME` means the SAVE request met the timing condition. Native pause needs separate host evidence, so the harness leaves `native_pause_verified: false` and `native_paused_at: null`. A complete live two-round check captures active, halted and then active after an actual authorized continuation on the same goal/run. A synthetic clock test exercises logic only.
