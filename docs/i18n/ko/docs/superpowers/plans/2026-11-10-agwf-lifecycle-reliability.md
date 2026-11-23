<a name="lifecycle-startup-and-recovery-implementation-plan"></a>

# 수명 주기 시작과 재개 구현 계획

> 2026년 11월 10일의 과거 구현 계획입니다. 체크리스트는 당시 제안한 작업을 기록합니다. 아래 승인과 언급된 비공개 자료는 당시 작업에 해당합니다. 현재 동작은 [운영 안내서](../../../skills/agent-workflow-v1-1/references/operations.md)를 참조하세요.

<a name="goal-and-constraints"></a>

## 목표와 제약

계획은 첫 시작의 안전한 재시도, 제때 사용량 갱신, 검증된 재개, 정상 종료를 다뤘습니다. 해시로 고정한 후보, 타입이 지정된 워크플로 코어, 모델을 쓰지 않는 감시 프로세스를 유지했습니다. 검사 도구는 준비, 실제 호스트 관측, 동작, 종료를 구분하도록 했습니다. 시간 시험은 명시적으로 선택하는 모드로 두었습니다. [과거 명세](../specs/2026-11-10-agwf-lifecycle-reliability.md)를 참조하세요.

당시 계획은 v1.1의 해당 소스 커밋과 푸시를 허용하고 GitHub 릴리스와 릴리스 태그는 제외했습니다. Python 3.11 이상, 표준 라이브러리와 unittest, 기존 Codex 목표·사용량 도구, Windows 프로세스 및 파일 공유 동작을 사용했습니다. 진행 중인 작업, 기존 런타임 근거, 패키지 연결을 보존했습니다. 기존 패키지, 목표, 측정값, 사용자 발언, 프로세스 식별자를 직접 관측해야 했습니다. 합성 데이터는 격리 테스트에만 사용했습니다. 측정 기록 만료, 실제 한도, 조기 중단, 사용자 중단에는 각각 다른 승인 요건을 뒀습니다. 정상 동작에는 인위적 SAVE를 추가하지 않았고, 다른 세션 작업과 사용자의 모델 설정은 건드리지 않았습니다.

<a name="review-focus"></a>

## 검토할 핵심 사항

1. 첫 시작 실패 뒤 측정 만료로 실제 SAVE가 기록됐다면 보조 프로그램 상태가 생기기 전에도 요건을 충족하는 재시도를 허용합니다.
2. 한도나 사용자 중단, 새 목표, 식별되지 않은 작업 프로세스는 기술 복구 검사를 통과할 수 없습니다.
3. 부분 시작 또는 동시 시작은 근거가 확인될 때까지 결과가 불확실합니다.
4. 기록 파일을 새로 쓰더라도 관측은 새로워지지 않습니다. 측정값이 최신이어도 실행 유지 기한은 만료될 수 있습니다.
5. 늦은 Codex 이벤트는 시간 검사를 통과할 수 없습니다. Windows 임시 공유 오류만으로 프로세스 종료를 확정하지 않습니다.

<a name="files-and-responsibilities"></a>

## 파일과 책임

| 경로 | 계획된 역할 |
| --- | --- |
| `skills/agent-workflow-v1-1/scripts/candidate.py` | 시작 시도, 재시도 판단, 감시 프로세스 소유권, 수명 주기 결과 |
| `skills/agent-workflow-v1-1/private/limit-watcher/scripts/monitor.py` | 계약을 뒷받침하는 최소 변경으로 SAVE, 실행 세대, 측정 기록 동작 구현 |
| `skills/agent-workflow-v1-1/scripts/workflow.py` | 기존 활성·중단 확정·완료 상태, 사용자 중단 규칙 |
| `tooling/run-lifecycle-check.py` | 주 에이전트의 Codex 근거로 동작하는 검사 도구 |
| `tooling/tests/test_lifecycle_check.py` | 준비, 늦은 이벤트, 결과 분류 테스트 |
| `skills/agent-workflow-v1-1/tests/test_candidate_coexistence.py` | 해당 실행의 실제 Python 프로세스를 사용하는 소스 및 설치본 통합 테스트 |
| `test_native_usage_source.py`, `test_goal_binding.py`, `test_watcher_engine.py`, `test_monitor.py`, `test_watcher_atomic_write.py` | 기록 경계, 목표와 권한, 감시 시간, Windows 동작의 스킬 테스트 |
| `SKILL.md`, `references/operations.md`, `references/native-usage.md`, `references/coexistence.md` | 갱신, 재시도, 설치 및 별도 호스트 승인 지침 |

<a name="task-1-retry-a-failed-first-activation"></a>

## 작업 1. 실패한 최초 활성화 재시도

계획된 인터페이스는 `activation_identity(command: str, payload: dict) -> dict`, `record_activation_attempt(runtime, record: dict) -> None`, `startup_retry_options(runtime, engine, command: str, payload: dict) -> tuple[str, ...]`입니다. 실행별 `activation-attempt.json`은 소스 식별자, `schema_version`, `command`, `pre_state_sha256`, `phase`, 관측한 감시 및 실패 정보를 기록했습니다. `prepared`, `failed`, `committed`는 별개 상태입니다. `prepared`에서 멈춘 기록은 실제 상태와 대조해야 합니다.

- 먼저 오래된 기록으로 실패를 입증한 뒤 `bind-active-goal`과 `activate`에서 같은 목표에 새 기록으로 재시도합니다. 원래 SAVE 해시와 사용자 승인을 보존하고 정상 프로세스와 활성 상태를 확인합니다.
- `preflight_activation` 뒤에 시작 의도를 기록합니다. 명령, 실행, 목표, 채팅, 범위, 목표 문구, 승인을 연결합니다. 관측 시각만 달라져도 범위는 그대로입니다.
- 실행 도구와 코어 단계 사이에 이 실행이 소유하는 단일 작성자 잠금을 사용합니다. 알 수 없는 잠금 소유자는 거부합니다. 기본 UI와 Python 상태는 별도 작업입니다.
- 같은 실행에서 `usage_receipt_expired`가 확인됐고 활성 목표와 실제 측정값을 새로 관측한 경우에만 기술적 새 실행 세대를 사용합니다. 호출자의 재시도 플래그는 승인 근거가 될 수 없습니다. 프로세스 정상과 코어 상태 기록 후 성공을 남깁니다.
- 한도 소진, 조기 중단, 다른 진단, 새 SAVE, 사용자 중단, 변경되거나 비활성인 목표, 동작 중이거나 알 수 없는 프로세스, 수정됐거나 미완료인 저널, 틀린 패키지, 동시 시작을 거부하는 경로를 검사합니다. 잘못된 재시도 기록 뒤에도 복구를 시험합니다. 롤백은 해당 실패 범위 안에서 검증된 이전 제어 정보만 되돌리고 이후 SAVE는 보존합니다.
- 후보 공존 및 Codex 측정 기록 테스트를 실행합니다. 활성 상태, 현재 식별자와 유효성, 이전 기록 보존, 실제 프로세스 정리를 검사합니다.

<a name="task-2-prepare-before-measuring-and-refresh-early"></a>

## 작업 2. 측정 전 준비와 조기 갱신

검사 도구는 `prepare`, `run --input PATH`, `observe --input PATH`, `close`를 제공하도록 계획했습니다. 준비 단계에서 시작 전에 패키지, 고정 해시, 역할, 명령을 확인합니다. `refresh_by(at: float, interval: float) -> float`는 `at + interval - min(15, interval / 4)`를 권장합니다. 간격이 60초이면 45초에 갱신하라는 제안입니다. 실제 기한은 달라지지 않습니다.

- 준비와 도움말에 실제 입력이 필요 없는지, 입력이 오래됐거나 없으면 활성 상태가 생기지 않는지, 기록 재사용으로 관측 시각이 바뀌지 않는지 확인합니다.
- 준비 뒤 Codex 목표와 사용량을 읽습니다. 실제 두 구간과 초기화 시각 등 승인된 필드만 복사하고 계정·개인 식별·크레딧은 포함하지 않습니다.
- `sample_at`, `next_check_at`, `check_interval_seconds`, `refresh_by`를 구분합니다. 측정값 파일 쓰기 성공이나 상태 신호 하나로 최신 정상 상태가 입증되지 않습니다.
- `status.at == usage.observedAt`, 실제 프로세스 식별자, 실행 세대 일치, `sampleFresh`, `leaseValid`를 확인합니다. 반영 뒤 기한 연장을 검사합니다. 중단할 수 없는 호출 중에도 기록이 최신이라고 약속할 수 없습니다.
- 정상 작업에서는 타이머를 삽입하지 않습니다. 지시에 따라 스냅샷을 검증하고 정상 종료의 완료 상태와 실제 프로세스 종료를 확인합니다. 정상 종료에서 인위적 SAVE가 생기지 않았는지 검사합니다.
- 기록 관측 시각과 게시 시각이 다를 때 수명 주기 및 Codex 측정 기록 테스트를 실행합니다.

<a name="task-3-diagnose-timing-without-confusing-late-events-with-success"></a>

## 작업 3. 늦은 이벤트를 구별하는 시간 진단

- `timed-test`는 명시적으로 요청됐을 때만, 준비된 정상 프로세스와 실제 이벤트 및 `--event-input PATH --delay-seconds 10 --tolerance-seconds 1`로 실행합니다. 목적 시각은 `event_at + 10`이며 단조 시계로 기다립니다.
- 45초 늦게 관측한 이벤트는 `INVALID_TIMING`이어야 합니다. 과거 98.74초와 45.05초 시험은 실패로 기록합니다.
- 기존 공식 목표 근거를 사용합니다. 지원되는 Codex 이벤트 연결이 없으면 관측을 늦거나 불완전한 것으로 분류합니다. 비공개 전송 경로나 자격 증명 저장소로 UI를 측정하지 않습니다.
- `native_event_at`, `observed_at`, `worker_healthy_at`, `save_requested_at`, `save_returned_at`, `snapshot_verified_at`, `helper_halted_at`, `native_paused_at`을 각각 기록합니다. 9~11초 합격 범위는 이미 정상인 프로세스로 SAVE를 요청할 때 적용합니다. Codex 일시 정지에는 자체 근거가 필요합니다.
- 두 실환경 시험 모두 같은 목표와 실행을 씁니다. 활성 상태, 검증된 체크포인트, 중단을 입증한 뒤 실제 승인된 계속을 확인합니다. 첫 연결만으로는 재개에 성공한 것이 아닙니다.
- 이른 이벤트, 늦은 이벤트, 재사용 또는 수정된 합성 이벤트, 일시 정지 근거 부재, 명시적 중단, 두 차례 흐름을 테스트합니다. 실제 시간 측정은 선택이며 정상 승인과는 별개입니다.

<a name="task-4-verify-recovery-and-regressions"></a>

## 작업 4. 재개와 회귀 검증

- 첫 시작, 첫 시도 중 만료와 새 측정으로 재시도, 활성 실행에서 만료 뒤 중단·스냅샷·중단 확정·재개, 잘못된 시도 다음의 유효한 재시도, Codex 일시 정지 후 활성 전이, 자연 종료를 시험합니다. 해당 소스와 패키지 진입점을 사용합니다.
- 재개 전 `halted`를 관측하고 스냅샷을 검사합니다. 재개 후에는 원래 목표와 실행, 새 실행 세대, 보존된 SAVE 해시, 권한과 실제 프로세스 정상 상태를 확인합니다.
- 한도, 조기 중단, 더 새로 기록된 한도 SAVE, 사용자 중단, 다른 목표, 이전 기록이나 재사용 기록, 손상된 체크포인트, unresolved 상태로 동작 중인 하위 에이전트를 거부합니다. Codex의 `blocked`에는 별도의 기준이 있습니다.
- Windows에서는 필요한 해당 실행 소유 파일만 복사하고 알려진 프로세스 식별자와 최종 상태를 함께 검사합니다. 일시적 공유 오류만으로 종료로 판단하지 않습니다. 지속적인 권한 오류는 정해진 시간 안에 표시합니다.
- 집중 검사가 통과하면 저장소의 두 테스트 모음을 실행합니다. 테스트 수는 실제 실행에서 확인합니다. 실환경 정상 수명 주기에는 실제 목표와 준비된 검사 도구가 필요합니다. 만료와 시간 진단에는 별도 승인이 필요하므로 `not_tested`로 남을 수 있습니다.

<a name="task-5-document-the-contract-and-verify-distribution"></a>

## 작업 5. 계약 문서화 및 배포 형태 검증

- 첫 재시도, 중단 확정 이후 재개, 한도 및 사용자 중단의 개별 경로를 설명합니다. 측정 갱신과 기한 연장 순서, 늦은 이벤트의 진단 범위를 적습니다.
- 패키지와 기존 작업을 확인합니다. 기존 수정과 실행 중인 설치본을 보존합니다. 이전 해시 소유권만으로 활성 상태를 추정하거나 동작 중인 실행을 이전하지 않습니다.
- 동일 소스로 두 번 빌드해 매니페스트 및 ZIP 해시를 비교합니다. 변경한 패키지 파일은 빌드 및 설치 테스트에서 확인합니다. 설치된 후보를 임시 실행에서 활성화 없이 검사하고 해시와 Windows 접근을 검증한 뒤 검사 범위를 보고합니다.
- `startup_retry`, `receipt_refresh`, `halted_resume`, `native_continue`, `natural_complete`, `timed_test`, `worker_exit`, `package_match`의 근거를 따로 기록합니다. 예전 실패 근거도 보존합니다.
- 당시 승인된 소스 공개 전에 권한, 롤백, SAVE 보존을 검토합니다. 비공개 보고서, 도구 기록, 런타임 상태는 Git 밖에 둡니다.

<a name="task-6-align-the-existing-v11-identity-and-presentation"></a>

## 작업 6. 기존 v1.1 식별자와 저장소 소개 정리

- 표시 이름, 호출 이름, 패키지 경로, 새 이름 공간을 함께 이전합니다. 이전 런타임 식별자와 승인 해시는 보존합니다.
- v1.0의 실제 이력과 v1.1의 변경을 `CHANGELOG.md`에 적습니다. README에는 목적, 한계, 아키텍처, 워크플로, 오케스트레이션을 설명합니다.
- 깨끗한 체크아웃에서 해시 보존, 검증, 명시적 호출로 이어지는 수동 및 에이전트 설치를 설명합니다. 설치 준비만으로는 Codex 목표를 시작하거나 모델 설정을 바꾸지 않습니다.
- 당시 허용한 공개에서는 About과 Topics을 소스에 맞춥니다. 저장소 접근과 브랜치 보호를 보존합니다. README 명령과 다이어그램의 소스 일치를 검증합니다.
- 승인된 파일만 공개하고 원격 커밋과 트리, 버전과 문서를 확인합니다. 릴리스나 태그가 생성되지 않았는지 확인합니다.

<a name="acceptance-boundary"></a>

## 승인 범위

원래 계획은 최종 공개 전에 핵심 작업 1~5를 진행하도록 했습니다. 최초 재시도와 실제 중단 뒤 재개의 시험 조합, 회귀 검사, 패키지 검사와 정확한 범위의 실제 관측이 필요했습니다. Codex UI 동작에는 실제 호스트 근거가 필요했습니다. 10초 결과는 별도의 주장으로, 실제 Codex 이벤트와 별도로 관측한 시간 근거를 요구했습니다.
