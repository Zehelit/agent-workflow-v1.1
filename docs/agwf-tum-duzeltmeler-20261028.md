# AGWF monitoring follow-up on 28 October 2026

> Historical maintenance record. This describes the follow-up to the [first maintenance pass](agwf-duzeltmeler-20261028.md). It provides no current activation authority.

| Finding | Change and evidence recorded at the time |
| --- | --- |
| Six role skills could be selected automatically | Each role received its own `allow_implicit_invocation: false` setting and a prepared-task boundary. The package tests checked all seven skills. |
| Source, remote and install differed | Required tools moved into the package. The installed files were compared with the manifest and ZIP built from a clean source commit. |
| Windows read/write collisions | Shared readers and replacement changes reached the relevant components. Temporary Windows file tests exercised locks, retries and preservation of old contents. |
| A second Codex server was started for every measurement | That code was removed. The main agent reads the existing host usage tool and Python reads the receipt. Measurement tests block process-launch calls. |
| PATH could select a different Codex CLI | Executable selection was removed from the measurement path. |
| Error output was discarded | The child server was removed. Measurement failures retain at most 20 records with diagnostic code, error class and stage. Unexpected Python worker failures keep local stderr. |
| Two loops rechecked the package every second | Heartbeat is limited to run/epoch identity. The full check occurs at command and measurement boundaries. |
| Tests previously used a mock server | Tests moved to the usage receipt contract. The maintenance record also reports checking a real Codex usage response through a temporary file. |
| Role policy still named the wrong threshold | The old `%10` text was aligned with the actual `%20` stop threshold. A refused startup with missing data clears its active marker. |

The new data path uses the original observation time and source identity through `--usage-input`. The watcher binds the receipt to its run and epoch. Missing, future, expired, regressed or wrong-run readings are refused. Publishing the same reading again cannot advance its observation time. Without fresh input the watcher retries validation, then records SAVE_AND_STOP.

The main agent obtains new observations before the deadline. Python does not call the host tool. `ingest` publishes the data only. Activation, worker launch, authorization, renewed goal evidence and clearing a stop are separate operations.

The record reports 111 targeted tests in standard and `-O` modes. Actual AGWF/Codex launches were blocked in this suite, while the package builder and read-only tool tests could use allowed subprocesses. The source checks, installed hashes, disabled skill settings and clean-source reproduction were recorded separately.

The maintenance also reports passing a real host usage result through the receipt path without starting another process or changing its observation time. No account or identity data was added to the package. The exact cause of the historical Remote problem remained unverified and live AGWF/Remote coexistence was outside the test. See the [native usage contract](../skills/agent-workflow-v1-1/references/native-usage.md).
