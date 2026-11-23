# Changelog

This is the existing version history. The source checkout and accepted build receipt identify the exact package.

## 1.1 — 2026-11-10

### Added

- Durable evidence for the first activate or bind-active-goal attempt. Safe retry uses the failure receipt SHA-256 preserved outside the package.
- An owned single-writer lock around candidate lifecycle operations.
- tooling/run-lifecycle-check.py for preparation, startup, observation refresh, post-halt resume and normal completion.
- An explicit timed-test mode, with separate SAVE, checkpoint and halt timestamps and classification of late events. The tool produces no native pause proof.
- Actual measurement time, deadline, interval and an early refresh suggestion in status summaries.
- Purpose, architecture, workflow and orchestration diagrams, manual and agent installation instructions, and the lifecycle guide.

### Fixed

- Controlled recovery under the same authorized active goal after usage evidence expires. Actual quota and user stops remain outside this recovery path.
- Retry after a failed first startup preserves scope, past SAVE evidence, package binding and verified worker absence.
- A time-limited weekly-window exception applies consistently in the core and watcher without rewriting observed values.
- Bounded retry handles identified temporary Windows sharing and lock read failures. Permanent and unclassified errors remain visible.
- Clarified measurement refresh, lease renewal and post-halt freshness requirements.
- Updated old test-helper call signatures and process/file waits.

### Compatibility and evidence

- The callable skill, install directory and runtime namespace move together to 1.1: `$agent-workflow-v1-1`, `agent-workflow-v1-1/` and `.agent-workflow-v1-1/runs/`.
- New manifests reject the previous technical identity. Old runs remain historical evidence and cannot be automatically rebound. Old runtime namespaces remain reserved for ownership and restore.
- An update does not rewrite a run's immutable package binding. Use a new run after accepting a new package.
- Python 3.11+ and standard library only. The watcher has no independent access to native usage tooling and starts no second Codex server.
- Isolated tests use synthetic host observations and real Python worker processes. Live Remote, Continue and interface timing require distinct evidence.
- This version was recorded through source commits. A GitHub release or tag was outside the recorded publication scope.

## 1.0 — previous source version

- Manifest-pinned candidate entrypoint, typed workflow state, role documents, file ownership and capacity checks.
- Integrated model-free watcher, checkpoint and restore, a durable coordination journal and deterministic packaging.
- The historical record makes no claim that a matching GitHub Release or tag exists.
