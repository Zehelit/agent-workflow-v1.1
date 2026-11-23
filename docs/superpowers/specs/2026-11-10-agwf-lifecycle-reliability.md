# Lifecycle reliability specification

> Historical specification from 10 November 2026. The evidence and authorization below apply to that earlier work. See the [current operations contract](../../../skills/agent-workflow-v1-1/references/operations.md) for present behavior.

## Intended result

Make first activation, usage refresh, recovery and normal completion reliable. Prepare the test before the target runs. Report startup, recovery from a halted workflow and timing as separately observed outcomes.

## Evidence available at the time

- The original private timed report recorded an expired measurement before activation. Both ten-second trials failed at 98.74 and 45.05 seconds. A fresh epoch and first binding in the second trial did not demonstrate resume from a previously halted helper.
- A separate private normal-flow report recorded healthy startup, a second real reading 43.423 seconds later, a verified checkpoint, normal completion and worker exit. That run did not exercise a SAVE latch.
- At the time, `scripts/candidate.py` could open a new technical epoch and restore controls after a failed `resume`. First `activate` or `bind-active-goal` lacked the same retry journal. The implementation needed separate regression evidence.
- Earlier maintenance corrected authority and instruction conflicts as well as four test helper and wait faults. Those behaviors were to remain covered by regression tests.

## Requirements

1. Retry a first startup under existing authority only for a recorded `usage_receipt_expired` failure in the same goal/run scope, with a genuine new usage reading, an actual active goal observation and verified worker absence. This path should need no new user quote, run identity or rewritten usage data.
2. Reject quota, early reserve, user stop, a nonactive goal, uncertain process ownership, changed package or evidence, and a different goal. Python checks consistency and format. It does not independently authenticate the person or tool identified by the caller.
3. Keep failed attempts observable. A partially initialized run must not appear active. Preserve subsequent real SAVE evidence.
4. Prepare the package and test before taking the first measurement. Store the actual observation timestamp. A newer file write cannot refresh an older reading. Keep `ingest` and lease renewal distinct.
5. Normal operation should let real decisions drive `CHECKPOINT_SOON` and `SAVE_AND_STOP`. It should not inject a stop.
6. Demonstrate native Continue and recovery from a genuinely active then halted workflow apart from first-time binding.
7. Run the timed diagnosis only when selected. Prepare its harness before the real native event. Late observations or preparation that misses the deadline cannot pass. The proposed tolerance was 10 ± 1 seconds for requesting SAVE, with the actual timing reported.
8. The main agent continues to call native goal tools on the host. Do not add undocumented host access, a private Codex transport or credential reading for the background worker. Exact timing from a UI click would require an actual native event source.
9. Verify checkpoint, worker exit, helper completion and native goal completion independently. Pause only with the user's authorization and use `blocked` only when the native goal tool's criteria apply.
10. Keep Python 3.11+ and the standard library. A Windows sharing failure is not evidence of process exit or success. Preserve existing local edits and old runtime records. Do not rewrite their bindings.

## Original authorization

This document began as a planning task. The user later authorized the implementation, v1.1 source commit and push, and documentation refresh for that earlier task. A GitHub Release and tag were outside its scope. The live timed stop remained a separate diagnostic choice. Old run bindings were preserved.
