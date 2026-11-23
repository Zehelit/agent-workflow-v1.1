# Lifecycle startup and recovery implementation plan

> Historical implementation plan from 10 November 2026. Its checklists describe work proposed at the time. Its authorization and any private artifacts mentioned here belong to that work. Current behavior is documented in the [operations contract](../../../skills/agent-workflow-v1-1/references/operations.md).

## Goal and constraints

The plan covered safe first-start retry, timely measurement refresh, verified recovery and normal close. It retained the pinned candidate, typed workflow core and model-free watcher. The lifecycle harness would separate preparation, real host observation, operation and close. The timed test would remain an explicit mode. See the [historical specification](../specs/2026-11-10-agwf-lifecycle-reliability.md).

The original plan authorized the relevant source commit and push for v1.1 and excluded GitHub Release and release tag creation. It required Python 3.11+, standard library and unittest, the existing Codex goal/usage tools and Windows process/file-share behavior. It preserved work in progress, old runtime evidence and package bindings. Existing package, goal, measurement, user quotes and process identity had to be observed. Synthetic data belonged in isolated tests. Receipt expiry, real quota, early reserve and user stops had separate authority requirements. Normal operation introduced no artificial SAVE. The plan left other sessions' work and the user's model settings alone.

## Review focus

1. A real expired-usage SAVE after a failed initial start should allow an eligible retry even before helper state exists.
2. A quota or user stop, new goal or unknown worker must not pass the technical recovery check.
3. A half-start or concurrent start must remain uncertain until supported evidence resolves it.
4. A newer receipt file does not make its observation newer. A fresh sample can coexist with an expired lease.
5. A late native event cannot pass timing. Windows temporary sharing failures cannot establish worker exit.

## Files and responsibilities

| Path | Planned responsibility |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | Startup attempt, retry decision, watcher ownership and lifecycle outcomes |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | SAVE, epoch and receipt behavior, with only narrow changes to support the contract |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | Existing active/halted/complete and user stop rules |
| `tooling/run-lifecycle-check.py` | Prepared harness driven by native evidence supplied by the main agent |
| `tooling/tests/test_lifecycle_check.py` | Preparation, late-event and result-classification tests |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | Source and installed package integration against real owned Python workers |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py` and `test_watcher_atomic_write.py` | Skill test suites for receipt boundaries, goals and authority, watchdog timing and Windows behavior |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md` and `references/coexistence.md` | Instructions for refresh, retry, installation and separate host acceptance |

## Task 1. Retry a failed first activation

The planned interfaces included `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None` and `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`. The per-run `activation-attempt.json` recorded source identity, `schema_version`, `command`, `pre_state_sha256`, `phase` and observed watcher/failure details. `prepared`, `failed` and `committed` are distinct phases. An unfinished `prepared` record requires reconciliation.

- First prove failure with an old receipt, then exercise same-goal fresh retry on `bind-active-goal` and `activate`. Preserve the original SAVE hash and user authority. Verify a healthy worker and active state.
- Persist the startup intent after `preflight_activation`. Bind command, run, goal, thread, scope, objective and authority. A new observation time alone does not create a new scope.
- Use an owned single-writer lifecycle lock between launcher and core steps. Refuse an unknown lock owner. Native UI and Python state are separate operations.
- Use the technical epoch path only for confirmed same-run `usage_receipt_expired` and a new active goal observation and real receipt. A caller-provided retry flag grants no authority. Record success after worker health and core commit.
- Cover negative paths for quota, early reserve, another diagnostic, a new SAVE, user stop, changed or inactive goal, live or unknown workers, modified or unfinished journal, incorrect package and concurrent startup. Test recovery after an invalid retry receipt. Rollback restores verified old controls only within the applicable failure boundary and preserves any newer SAVE.
- Run the candidate coexistence and native receipt tests. Inspect active state, current identity and freshness, retained history and actual worker cleanup.

## Task 2. Prepare before measuring and refresh early

The harness would expose `prepare`, `run --input PATH`, `observe --input PATH` and `close`. Preparation verifies package, pin, roles and commands before any startup. `refresh_by(at: float, interval: float) -> float` suggests `at + interval - min(15, interval / 4)`. For a 60-second interval this suggests a refresh at 45 seconds. It does not move the actual deadline.

- Test that prepare and help need no live input, that old or missing input creates no active state and that replay cannot change observation time.
- Get native goal and usage after preparation. Copy only approved fields including both real windows and reset times. Keep account, identity and credit data out.
- Distinguish `sample_at`, `next_check_at`, `check_interval_seconds` and `refresh_by`. A successful receipt publication or heartbeat does not demonstrate fresh health.
- Require `status.at == usage.observedAt`, live identity, epoch match, `sampleFresh` and `leaseValid`. Verify renewal after ingestion. Do not promise fresh receipts through an uninterruptible call.
- Leave timer injection off in normal work. Verify snapshots when directed and confirm complete plus worker exit at normal close. Check that normal close created no artificial SAVE latch.
- Exercise the lifecycle and native receipt suites with unequal receipt and publication times.

## Task 3. Diagnose timing without confusing late events with success

- Run `timed-test` only when explicitly requested, with a prepared healthy worker, a genuine event and `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`. The intended deadline was `event_at + 10` and the wait uses a monotonic clock.
- An event observed 45 seconds late should return `INVALID_TIMING`. The historical 98.74 and 45.05 second trials remain failures.
- Use existing official goal evidence. If no supported native event hook exists, classify late or incomplete observation accordingly. Do not use private transports or credential stores to measure the UI.
- Record `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at` and `native_paused_at` separately. The 9–11 second success window applies to requesting SAVE with a previously healthy worker. Native pause requires its own evidence.
- For both live rounds, use the same goal and run. Prove active then checkpointed and halted, then verify an eligible real continuation. A pristine binding is not successful resume.
- Exercise early, late, replayed and changed synthetic events, absent pause evidence, explicit stop and the two-round flow in tests. Keep live timing optional and separate from normal acceptance.

## Task 4. Verify recovery and regressions

- Exercise first startup, expiry during first attempt and fresh retry, expiry of an active run followed by stop/snapshot/halt/resume, invalid then valid retry, native paused-to-active continuation and natural close. Test the applicable source and packaged entrypoints.
- Before resumed work, observe `halted` and verify the snapshot. After resuming, verify original goal and run, new epoch, retained SAVE hash, authority and actual worker health.
- Refuse quota, early reserve, a newer quota SAVE, user stop, wrong goal, stale or replayed receipts, corrupt checkpoints and unresolved live children. Native `blocked` retains its own criteria.
- In Windows tests, copy only needed owned files and check known process identity together with terminal health. Temporary sharing errors do not imply exit. Persistent permission errors must surface within a bounded time.
- Run both repository test suites after focused cases pass. Test counts are discovered, not assumed from an older run. Live normal lifecycle uses a real goal and a prepared harness. Expiry and timed diagnosis have their own authorization and may remain `not_tested`.

## Task 5. Document the contract and verify distribution

- Document first retry, halted resume and separate quota/user stop paths. Explain refresh and lease order and what a late-event diagnostic can show.
- Inventory the package and existing work. Preserve edits and running installations. Do not infer activity from old pin ownership or migrate active runs.
- Build twice from identical source. Compare manifest and ZIP hashes. Verify any changed payload in the build and installation tests. Check the installed candidate with a disposable passive run, verify the pin and Windows access, and report the passive scope.
- Record proof for `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit` and `package_match` independently. Retain old negative evidence.
- Review authority, rollback and SAVE preservation before any previously authorized source publication. Keep private reports, receipts and runtime state outside Git.

## Task 6. Align the existing v1.1 identity and presentation

- Migrate visible name, callable identity, package path and new namespace together. Preserve earlier runtime identities and accepted pins.
- Record actual v1.0 history and v1.1 changes in `CHANGELOG.md`. Explain purpose, limitations, architecture, workflow and orchestration in the README.
- Describe human and agent installation from a clean checkout through preserved pin, verification and explicit invocation. Setup does not itself start a native goal or change model settings.
- For the publication authorized at that time, make About and topics match the source. Preserve repository access and branch protection. Verify README commands and source fidelity of the diagrams.
- Publish only authorized files, verify the remote commit/tree, version and documentation, and verify that no release or tag was created.

## Acceptance boundary

The original plan ordered core tasks 1 through 5 before final publication work. It required the first retry and genuinely halted resume matrix, regression tests, package checks and correctly scoped real observations. Native UI behavior required actual host evidence. The timed 10-second result was a separate claim that required an actual native event and separately observed timing.
