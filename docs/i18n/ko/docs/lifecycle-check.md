<a name="repeatable-lifecycle-check"></a>

# 반복 가능한 수명 주기 검사

`tooling/run-lifecycle-check.py`는 실제 목표, 사용량, 하위 세션 목록을 고정된 후보 프로그램에 전달합니다. 주 에이전트가 호스트에서 이 관측값을 얻습니다. 세션과 입력은 Git 밖에 비공개로 보관하고 한 작성자만 세션에 쓰도록 하세요.

<a name="prepare-before-measuring"></a>

## 측정 전에 준비하기

후보 패키지를 빌드하고 승인된 외부 해시를 보관하세요. `prepare`는 패키지를 검사하고 새 세션을 기록합니다. 후보 프로그램이나 감시 프로세스는 시작하지 않습니다. 첫 실제 사용량 측정 전에 준비를 마치세요.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py prepare --candidate C:/build/agent-workflow-v1-1/scripts/candidate.py --manifest C:/build/agent-workflow-v1-1/manifest.json --manifest-sha256 PRESERVED_SHA256 --version 1.1 --project C:/project --run-id check-1 --session C:/private/check-1.json
```

`PRESERVED_SHA256`은 승인된 빌드 기록에서 가져옵니다. 기존 프로젝트와 새로운 세션 파일을 쓰세요. 패키지를 변경했다면 이전 세션을 다른 해시에 다시 연결하지 마세요.

<a name="host-input-object"></a>

## 호스트 입력 객체

`run`, `observe`, `resume`, `close`, `timed-test`는 `--input`으로 JSON 객체를 받습니다.

| 필드 | 내용 |
| --- | --- |
| `activation` | 최초 `run` 전용. `bind-active-goal`의 전체 타입 지정 입력으로, `native_binding`과 일치하는 `app_goal_observation`, 목표, 해석, 절차 확인, 관측한 모델 및 추론 강도, 슬롯 제한, 실제 세션 목록과 관측 시각을 포함합니다. |
| `app_goal_observation` | 최신 `get_goal` 관측으로 목표 및 채팅 ID, 목표 본문, 생성 시각, 상태, `source_kind: get_goal`, 실제 출처 참조, `observed_at`을 포함합니다. 실제 `updated_at`이 제공되면 추가합니다. |
| `usage` | [Codex 사용량 측정 기록](../skills/agent-workflow-v1-1/references/native-usage.md)의 `source`, `sourceId`, `observedAt`, `result` 필드입니다. 필요한 구간만 보존합니다. |
| `inventory` | 최신 실제 하위 에이전트 ID와 상태입니다. 하위 에이전트가 없을 때만 `[]`로 둡니다. |
| `checkpoint_files` | 복구할 변경, 추가, 삭제 경로입니다. 프로젝트 상대 경로이며 주 에이전트가 쓴 파일도 포함합니다. 코어는 작업에 배정된 파일도 넣습니다. |
| `writers_quiescent` | 모든 작성자가 실제로 쓰기를 멈춘 뒤에만 `true`를 설정합니다. 이 관측은 검사 도구가 직접 할 수 없습니다. |
| `next_action` | 복구 또는 정상 종료 뒤에 할 구체적인 다음 작업입니다. |
| `weekly_exception` | 첫 `run`에서 선택할 수 있습니다. 실제 사용자 인용 `exact_quote`와 유한한 Unix 만료 시각 `valid_until`을 제공합니다. 기본값은 예외 없음입니다. |
| `resume` | 선택적 후보 프로그램 재개 입력 전체입니다. 생략하면 기록된 측정 만료일 때만 `usage_recovery: true`를 시도합니다. 한도에 따른 중단과 사용자가 요청한 중단은 각각 별도 요건을 따릅니다. |

Codex API가 별도의 목표 ID를 주지 않으면 `threadId:createdAt`처럼 일관되게 만든 식별자를 사용하고 파생 ID임을 명시하세요. 타입 지정 규칙은 [운영](../skills/agent-workflow-v1-1/references/operations.md)과 [명령](../skills/agent-workflow-v1-1/references/commands.md)에 있습니다.

<a name="run-refresh-and-close"></a>

## 실행, 갱신 및 종료

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py run --session C:/private/check-1.json --input C:/private/initial-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py observe --session C:/private/check-1.json --input C:/private/new-observation.json
python -X utf8 -B tooling/run-lifecycle-check.py close --session C:/private/check-1.json --input C:/private/final-observation.json
```

Codex 호출이 반환되는 즉시 실제 관측 시각을 적습니다. 검사 도구는 `status.at == usage.observedAt`, 해당 실행의 작업 프로세스 식별자, 실행 세대, 상태 신호, 측정값, 실행 유지 기한을 확인합니다. `observe`는 실행 유지 기한도 연장합니다. `refresh_by`는 여유 있게 갱신하라는 권장 시각이며 실제 기한을 바꾸지 않습니다.

`CHECKPOINT_SOON`에서 완성된 작업의 체크포인트를 확인하세요. `checkpoint_files`를 최신 상태로 유지하세요. 정상 `close`는 최신 감시, 검증된 스냅샷, 완료 상태 및 관측된 프로세스 종료를 요구합니다. 정상 동작에서 SAVE 이벤트를 인위적으로 넣지 않습니다.

`weekly_exception`을 준 첫 `run`은 기간이 정해진 감시 정책을 적용하고 활성화 전에 `NEED_POLICY_OBSERVATION`을 반환합니다. 새 목표 및 사용량 관측값을 얻고 같은 세션에서 재실행하세요. 이전 측정 기록의 원래 시각은 그대로 둡니다.

<a name="halt-and-resume-after-expiry"></a>

## 측정 만료 뒤 중단 및 재개

감시 프로세스가 SAVE를 요청하면 `observe`는 중단, 스냅샷, 중단 확정을 수행하고 `NEED_POST_HALT_OBSERVATION`을 반환합니다. 중단 뒤의 목표, 사용량, 세션 목록을 새로 관측하세요.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py resume --session C:/private/check-1.json --input C:/private/after-halt-observation.json
```

코어는 감시 프로세스가 실제로 관측한 원인을 사용합니다. 기술 복구에는 동일한 승인된 활성 목표, 그대로인 파일과 체크포인트, 중단 뒤 최신 근거, 해당 프로세스가 종료됐다는 확인이 필요합니다. 한도 소진, 조기 중단, 사용자의 명시적 중단에는 각각 맞는 재개 절차가 있습니다. 검사 도구가 Codex 목표를 paused, blocked, active로 바꾸지는 않습니다.

첫 활성화가 실패하면 세션은 후보 프로그램의 시작 시도 해시를 보관합니다. 지원되는 재시도 조건에서만 같은 `run`을 새 근거와 함께 사용하세요. 수정된 세션은 근거로 인정되지 않습니다. 완료됐거나 결과가 불확실한 시도는 자동으로 다시 시작하지 않습니다.

<a name="run-a-timed-diagnosis-only-when-selected"></a>

## 명시적으로 선택한 경우에만 시간 진단

이벤트 전에 정상적으로 동작하는 프로세스를 준비하고, 명시적으로 요청한 진단에만 `timed-test`를 사용하세요. 이벤트 JSON에는 실제 Codex 관측의 `event_id`, `goal_id`, `thread_id`, `source_id`, `event_at`, `observed_at`을 넣습니다.

```powershell
python -X utf8 -B tooling/run-lifecycle-check.py timed-test --session C:/private/check-1.json --input C:/private/current-observation.json --event-input C:/private/native-event.json --delay-seconds 10 --tolerance-seconds 1
```

늦게 도착한 이벤트는 `INVALID_TIMING`을 반환합니다. `SAVE_REQUEST_ON_TIME`은 SAVE 요청이 시간 조건에 맞았음을 뜻합니다. Codex 일시 정지는 호스트의 별도 근거가 있어야 하므로 이 도구에서는 `native_pause_verified: false`와 `native_paused_at: null`로 남습니다. 실제 두 차례 검사에서는 같은 목표와 실행의 활성 상태, 중단 상태, 실제 승인된 계속 요청 이후의 활성 상태를 관측해야 합니다. 합성 시계 테스트는 로직만 확인합니다.
