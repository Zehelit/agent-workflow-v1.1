# AGWF maintenance on 28 October 2026

> Historical maintenance record. The observations and authorization below belong to the original work. See the current [README](../README.md) for current operation.

Each of the six role skills received its own `agents/openai.yaml` with `allow_implicit_invocation: false`. Role descriptions apply to prepared AGWF tasks. A role stops without the assignment and evidence of the actual model and reasoning effort. Role skills do not start the workflow or watcher. The main skill removed language implying that a previous project selection remained valid.

The source and installation were aligned on shared reads, permission-preserving replacement and bounded retries for Windows state files. `candidate.py` uses the same shared reader for changing status files. An external reader's lock preserves the old file. A persistent lock remains an error.

Context and usage tools moved into the packaged `tools/` folder so the user installation would not depend on the project checkout. The source, package and installed copy matched on 41 files in this historical check. The old root copy of the context tool was removed and its test paths were updated.

The maintenance record reports 63 package/tool tests and nine Windows file-access tests. The nine Windows tests were also exercised with `-O` and installed files. Seven skill definitions and file hashes were checked. The package test for the six missing role policies failed before the correction and passed afterward.

AGWF was disabled during the maintenance. The seven skill settings stayed off. The tests exercised core file access on temporary files. They did not test live Remote coexistence.

At that point the remaining risks included starting a second Codex server per reading, selecting another CLI from PATH, discarding child-server stderr and performing repeated full-package checks. The first pass did not establish the cause of the Remote issue. The [follow-up maintenance](agwf-tum-duzeltmeler-20261028.md) describes the subsequent changes. The current watcher no longer starts a second Codex server.
