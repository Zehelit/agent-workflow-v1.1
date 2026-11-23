<a name="command-contract-a02"></a>

# 명령 명세 (A02)

표준 라이브러리로 만든 보조 프로그램은 `Workflow.execute`와 `Workflow.apply`를 거치는 명령을 검증합니다. Codex 기본 도구를 가로채거나 호출 기록이나 사용자 메시지의 진위를 인증하거나 프로세스를 중단하거나 실제 모델을 독자적으로 관측하지 않습니다. 주 에이전트가 해당 동작을 수행하고 근거를 제공합니다. A26 승인과 체크포인트, 소유권, 슬롯에 관한 기존 규칙이 적용됩니다.

상태는 정수 `schema_version: 2`를 사용합니다. 버전이 없거나 구버전·미래 버전이거나 형식이 잘못된 상태는 변경 전에 거부합니다. 이전 상태를 가져오거나 이전하는 도구는 없습니다. 현재 타입이 지정된 승인으로 새 AGWF 실행을 시작하세요. 체크포인트 스냅샷과 복원은 지원됩니다. 과거 근거는 작업을 승인하지 않습니다. 오류 주입 테스트는 임시 테스트 사본만 사용합니다.

<a name="json-and-error-policy"></a>

## JSON과 오류 정책

상태 파일, `--input` 파일과 표준 입력, 체크포인트 메타데이터, 설치 도구의 소유권 JSON에 같은 엄격한 파서를 사용합니다. 중첩 깊이와 관계없이 같은 키가 두 번 나오면 값이 같아도 거부합니다. `NaN`, `Infinity`, `-Infinity`와 유한하게 보이지만 무한대로 넘치는 리터럴(예: `1e999`)도 거부합니다. 입력 파일은 BOM이 있거나 없는 UTF-8을 지원합니다. 명령의 JSON 루트는 객체여야 합니다. 빈 표준 입력은 `{}`을 주며 명령이 허용할 때만 성공합니다. `write_json`은 디렉터리나 임시 파일을 만들기 전에 `allow_nan=False`로 검증하고 직렬화합니다.

런타임 입력은 문자열 키의 JSON 객체입니다. 목록은 목록이고 정수나 실수는 불리언으로 쓰지 않습니다. 제공된 모든 플래그는 정확히 Python `bool` 타입(JSON의 `true`·`false`)이어야 합니다. 문자열, 0/1, null, 목록, 객체는 플래그를 대신하지 못합니다. 필수 플래그를 생략하면 거부하며 선택 플래그는 필드가 없을 때만 false로 기본 설정합니다. null은 잘못된 값입니다. 숫자는 유한해야 합니다. 개수는 1 이상의 정수, 시각은 0부터 sys.float_info.max까지의 정수나 실수, 백분율은 0부터 100까지입니다. 시각의 상한은 시간 계산에서 넘칠 정도로 큰 정수를 거부합니다. 명령별 최신성 조건도 지켜야 합니다. 새로운 슬롯 최대치는 없습니다. 배정에서 주 에이전트를 포함한 기존 개수 계산과 하위 에이전트 상한을 유지합니다.

입력 검증은 잠금과 디렉터리 생성 및 명령 실행 전에 합니다. `apply`는 전용 JSON 사본에서 처리하고 검사 통과 뒤 반영합니다. 거부된 명령은 아래의 지속적인 INPUT_INVALIDATED 기록과 contract_invalid 플래그를 제외하면 기존 상태 바이트와 SHA-256을 보존합니다. 최초 명령을 거부하면 상태를 생성하지 않습니다. 이전과 마찬가지로 실패한 스냅샷이 확정되지 않은 소유 체크포인트 디렉터리를 남길 수 있습니다. 형식은 유효하지만 없는 사용량은 아래에 설명하는 의도된 상태 전이입니다.

보조 프로그램은 오류 시 종료 코드 1을 반환하고 다음 객체를 표준 오류에 씁니다.

```json
{"error":"exact boolean required","code":"FIELD_TYPE","field":"$.accepted"}
```

| 코드 | 뜻 | 필드 |
| --- | --- | --- |
| `COMMAND_UNKNOWN` | 알 수 없는 엔진 명령 | `$.command` |
| `FIELD_REQUIRED` | 필수 입력 필드 또는 필수 유틸리티 옵션 누락 | 정확히 누락된 필드 |
| `FIELD_UNKNOWN` | 알 수 없는 입력 필드 또는 명령에 사용할 수 없는 CLI 옵션 | 문제의 필드 |
| `FIELD_TYPE` | 호출자가 잘못된 타입을 전달 | 정확한 필드나 목록 인덱스 |
| `FIELD_VALUE` | 잘못된 식별자, 비어 있는 필수 텍스트·목록, 범위를 벗어난 수치, 잘못된 목록 상태 | 정확한 필드나 목록 인덱스 |
| `JSON_DUPLICATE_KEY` | JSON의 중복 멤버 | 중첩을 포함한 정확한 중복 멤버 |
| `JSON_NONFINITE` | 원본 메타데이터를 포함한 유한하지 않은 숫자 또는 토큰 | 정확한 필드나 목록 인덱스 |
| `JSON_SYNTAX` | 잘못된 JSON 구문 또는 인코딩 | `$` |
| `JSON_TYPE` | JSON이 아닌 런타임 값 또는 문자열이 아닌 객체 키 | 정확한 값이나 객체 |
| `COMMAND_REFERENCE` | 참조한 작업이나 필요한 저장 필드가 없음 | 조회 위치 `$.id`. 오류 본문에 누락된 키를 표시합니다. |
| `COMMAND_CONSTRAINT` | 기존 승인·상태·소유권·검토·최신성 검사 | 관련 필드가 있으면 그 필드, 전체 상태 검사는 `$.command` |
| `STATE_VERSION` 또는 `STATE_TYPE` | 지원되지 않는 상태 버전 또는 객체가 아닌 상태 루트 | `$.schema_version` 또는 `$` |
| `IO_ERROR` 또는 `INVALID_VALUE` | 파일 시스템 오류 또는 다른 값 오류 | `$` |

단순 키의 경로에는 점, 그 밖의 키에는 대괄호, 목록에는 `[index]`를 씁니다. 알 수 없는 CLI 스위치, 알 수 없는 명령, 스위치 인자 누락 등 일반 argparse 오류는 argparse 표준 오류와 코드 2를 유지합니다. 알아볼 수 있으나 해당 명령에 적용할 수 없는 옵션에는 위의 구조화된 `FIELD_UNKNOWN`을 사용합니다. 두 방식 모두 `assert`를 쓰지 않으므로 `python -O`에서도 검증합니다.

<a name="payload-types"></a>

## 입력 타입

`text`는 검증할 때 양끝 공백을 제거한 뒤 비어 있지 않은 문자열입니다. 원문을 보존합니다. `string`은 비어 있을 수 있습니다. `task_id`는 `[a-z][a-z0-9_]{0,63}`에 일치합니다. `strings`는 비어 있지 않은 문자열로 된 목록이며 목록 자체는 비어 있어도 됩니다. `criteria`는 항목이 최소 하나 있어야 합니다. `task_ids`는 작업 ID 목록입니다. `timestamp`, `count`, `percent`는 위 숫자 규칙을 따릅니다. 명시적 null은 `?`가 붙은 타입만 허용합니다.

아래 명령과 구조화된 중첩 객체는 필수·선택 목록에 없는 필드를 거부합니다. 호출 기록 내용과 원시 `rateLimits` 또는 구간의 측정 이외 메타데이터에 한해 공급자 원본을 해석하지 않는 JSON으로 유지합니다. 이 경우도 중복 키와 유한하지 않거나 JSON으로 표현할 수 없는 값은 거부합니다. 알 수 없는 메타데이터를 명령 플래그나 유효한 측정의 근거로 쓰지 않습니다.

| 중첩 타입 | 필수 필드 | 선택 필드 및 제약 |
| --- | --- | --- |
| `authorization` | `event_id`, `exact_quote`, `kind`, `source_kind`, `source_id`, `goal_id`, `thread_id`, `run_id`는 text. `scope`는 scope. `observed_at`, `source_created_at`은 timestamp. `original_turn_id`, `observed_turn_id`는 text | 추가 필드 없음. A26에서 허용하는 종류, 직접 사용자 출처, 범위 일치, 재사용 및 시각 검사를 적용합니다. |
| `scope` | `kind`, `id`, `objective`는 text | A26에 따라 `kind`는 objective 또는 report입니다. |
| `source` | `source_kind`, `source_id`, `exact_quote`, `original_turn_id`는 text. `source_created_at`은 timestamp | 제공된 승인의 출처 필드와 일치해야 합니다. |
| `native_binding` | `binding_id`, `goal_id`, `thread_id`, `run_id`, `objective`는 text. `scope`는 scope. `goal_created_at`, `observed_at`은 timestamp | N02 최초 연결 전용입니다. objective 범위에서 목표가 정확히 일치하고 최초 생성은 관측보다 앞서야 합니다. 사용자 출처 필드는 없습니다. |
| `app_goal` | `goal_id`, `thread_id`, `status`, `source_kind`, `source_id`는 text. `observed_at`은 timestamp | `objective`(text)와 `goal_created_at`(timestamp)은 선택입니다. Codex 연결을 근거로 삼으면 갱신할 때마다 둘 다 필요하며 연결과 일치해야 합니다. A26은 get_goal 출처, 정확한 식별자, 60초 이하의 관측, 해당하는 활성 상태를 요구합니다. |
| `inventory` | `id`, `status`(text) 객체의 목록 | 항목에 다른 필드는 없습니다. 식별자는 고유하며 상태는 running, interrupting, unknown, idle, stopped, done입니다. 관측은 0~60초 이내여야 합니다. |
| `models` | 비어 있지 않은 모델 이름을 `strings` 추론 강도 목록에 매핑하는 객체 | 빈 객체와 목록을 허용합니다. prepare에는 정확히 관측한 모델과 강도가 필요합니다. |
| `arguments` | 구조상 필수 필드 없음 | `task_name`(task_id), `fork_turns`(text), `message`(string), `model` 및 `reasoning_effort`(text)만 받습니다. record가 준비한 인자와 완전히 일치하는지 검사하므로 필드가 빠지면 배정할 수 없습니다. |
| `receipt` | 비어 있지 않은 객체, 목록 또는 문자열 | 실제 도구 결과의 본문을 해석하지 않는 JSON으로 취급합니다. 진위를 인증하는 것은 아닙니다. |
| `rate_limits?` | 객체 또는 null | 선택 primary·secondary는 window?이고 다른 공급자 원본 필드는 해석하지 않습니다. |
| `window?` | 객체 또는 null | `usedPercent`(percent?), `windowDurationMins`(count?), `resetsAt`(timestamp?)은 선택입니다. 그 밖의 메타데이터는 해석하지 않습니다. |

<a name="engine-commands"></a>

## 엔진 명령

아래 필수 및 선택 목록에 있는 필드만 받습니다. 선택 필드가 없으면 필수 필드만 받습니다. 필드가 없는 명령은 `{}`을 받습니다.

| 명령 | 필수 입력 필드 | 선택 입력 필드·기본값 |
| --- | --- | --- |
| `activate` | `objective`(text), `authorization`(authorization), `interpretation`(string), `source_message`(source), `app_goal_observation`(app_goal), `models`(models), `slot_limit`(count), `inventory`(inventory), `observed_at`(timestamp) | `session_limit`(count?). 없거나 null이면 열린 세션에 추가 상한이 없습니다. |
| `bind-active-goal` | `objective`(text), `native_binding`(native_binding), `interpretation` 및 `procedural_check`(text), `app_goal_observation`(app_goal), `models`(models), `slot_limit`(count), `inventory`(inventory), `observed_at`(timestamp) | `session_limit`(count?). 작업이나 다른 필드와 기록이 없는 비활성 초기 schema2만 허용합니다. |
| `observe-authorization` | `authorization`(authorization), `interpretation`(string), `source_message`(source) | `app_goal_observation`(app_goal)을 전달할 수 있으나 저장된 목표 관측을 갱신하지 않습니다. |
| `app-goal` | `app_goal_observation`(app_goal) | 없음 |
| `usage` | 없음 | `observed_at`(timestamp), `rateLimits`(rate_limits?). 생략은 측정할 수 없다는 뜻입니다. |
| `usage-due` | 없음 | `substantial_work`(bool, false) |
| `inventory` | `inventory`(inventory), `observed_at`(timestamp) | 없음 |
| `task` | `id`(task_id), `role`(text), `objective`(text), `criteria`(criteria) | `file_inputs`(file_inputs, []), `context_refs`(strings, []), `review_targets`(review_targets, []), `revision`(text, 생략하면 UUID 생성), `writes`(strings, []), `depends_on`(task_ids, []), `independent_review`(bool, false) |
| `prepare` | `id`(task_id) | `effort`(text, 생략하면 역할 기본값), `current_context`(N06에 쓰는 context_pin) |
| `record` | `id`(task_id), `submitted_arguments`(arguments), `session_id`(text), `receipt`(receipt) | `effective_model`, `effective_effort`(text?). 생략하거나 null이면 검증하지 않았음을 뜻합니다. |
| `dispatch_failed` | `id`(task_id), `evidence`(text) | `certain_not_started`(bool, false). 없거나 false면 시작 여부가 불확실합니다. |
| `reconcile` | `id`(task_id), `evidence`(text), `no_owned_work_running`(bool) | true와 활성 하위 에이전트가 없는 최신 목록이 필요합니다. |
| `finish` | `id`(task_id), `accepted`(bool), `evidence`(text) | `review_task`(task_id). 독립 검토가 필요한 작업을 승인할 때 필수입니다. |
| `cancel` | `id`(task_id), `reason`(text) | 준비됐고 배정한 적 없는 작업만 가능합니다. |
| `stop` | 없음 | 없음 |
| `snapshot` | `writers_quiescent`(bool), `next_action`(text) | `files`(strings, []). 불리언 값은 true여야 합니다. |
| `restore` | `destination`(text) | 프로젝트 밖의 새 대상 경로여야 합니다. |
| `halt` | `inventory`(inventory), `observed_at`(timestamp), `owned_processes_stopped`(bool), `stop_evidence`(text) | `degraded`(bool, false). true라면 `reason`과 `next_action`(text)이 모두 필요합니다. 프로세스 중단 플래그는 true여야 합니다. |
| `resume` | `authorization`(authorization), `interpretation`(string), `source_message`(source), `app_goal_observation`(app_goal) | 없음. 중단 확정 후의 일치하는 계속 요청, 최신 사용량과 목록, 현재 검증된 체크포인트가 필요합니다. |
| `compact` | 없음 | 없음. 작업이 승인 또는 거부로 종료됐고 동작 중인 하위 에이전트가 없으며 검증된 최신 체크포인트가 필요합니다. |
| `complete` | 없음 | 없음. compact 조건에 더해 활성 실행이 필요합니다. |
| `status` | 없음 | 없음. 이전에 활성화한 상태가 필요합니다. |

파일 쓰기 권한은 기존의 Windows 별칭, 범위 및 심볼릭 링크 검사 아래 정확한 프로젝트 상대 파일로 지정합니다. 역할과 추론 강도는 기존 목록과 가용성 검사를 따릅니다. 읽기 전용인 검토·조율 역할은 쓰기 목록이 비어 있어도 됩니다. 알 수 없는 작업 참조는 거부하며 작업을 암시적으로 만들지 않습니다. `accepted: false`는 쓰기를 멈춘 작업을 거부하고, `accepted: true`는 해당 검토 검사를 통과한 뒤에만 승인합니다.

<a name="raw-usage-lifecycle"></a>

## 원시 사용량 수명 주기

N06의 `context_pin`에는 정확히 `{path, sha256, manifest_path, manifest_sha256}`가 들어갑니다. 경로는 비어 있지 않은 프로젝트 상대 경로이고 SHA-256은 소문자입니다. 매니페스트는 기존의 변경 불가능한 작업 입력 파일이어야 합니다. `prepare`는 최신 버전 1 요약을 현재 상태 및 별도로 고정한 매니페스트와 비교합니다. 요약을 활용한 준비 메시지는 이 절차에 묶입니다. `dispatch-start`는 `current_context`도 받아 준비 이후 상태와 원래 매니페스트로 다시 생성한 요약을 요구합니다. Codex 관련 모든 검사가 계속 적용됩니다. 생성 도구, 근거 장애 요인, 작성자 검사, 제한 사항은 [컨텍스트](context.md)를 보세요.

N07의 `status`는 원래 작업 상태 맵을 유지하고 `task_observations: {contract_version: 1, tasks: ...}`를 추가합니다. Codex 실행과 산출물 승인은 별개입니다. 실행이 done이어도 finish 검사를 통과하기 전에는 승인 대기 상태입니다. 독립 검토 후 finish는 기존 선택적 `review` 연결을 적고 서로 다르고 비어 있지 않은 두 Codex 세션 식별자를 요구합니다. 배정한 검토 해시와 기록한 산출물 해시도 일치해야 합니다. schema2는 바뀌지 않습니다. 검토 작업은 고정한 자기 작성자의 승인에 직간접적으로 의존할 수 없습니다. 검토자·작성자 식별자 또는 검토 해시가 불완전한 과거 독립 승인은 `historical_unrecorded`로 표시됩니다. 새로 배정하는 작업의 승인 의존 조건을 prepare, record, dispatch-start에서 충족할 수 없습니다. 요약과 보조 프로그램에서 같은 독립성 조건을 씁니다. 과거 기록은 자동으로 이전하지 않습니다.

호출자의 타입이나 범위가 잘못되면 오류가 나고 상태 바이트는 바뀌지 않습니다. 예를 들어 `usedPercent: "10"`, 불리언 개수나 시각, 101%, 음수 초기화 시각, 목록으로 된 구간, 유한하지 않은 메타데이터가 이에 해당합니다. `rateLimits`나 구간·측정값이 없거나 null인 경우, 타입은 맞으나 300분·10080분 중 한 구간이 없거나 다른 점은 유효한 오래된 관측이나 미래 관측은 사용할 수 없는 사용량입니다. 형식이 맞는 `usage`는 의도적으로 과거 측정값을 지우고 `usage_errors`를 올립니다. 첫 시도는 `RETRY_USAGE_ONCE`, 두 번째는 `SAVE_AND_STOP`을 반환하며 이미 중단 확정이나 완료 상태인 경우를 제외하고 중단 중 상태가 됩니다. 이미 중단된 실행을 조용히 되살리지 않습니다. 0%, 100%, 유한한 소수 백분율은 유효한 측정값입니다. 100%는 임계값 복구를 시작합니다. 잘못된 형식은 알 수 없는 측정 횟수에 들어가지 않습니다. 원시 메타데이터는 보존하지만 필요한 유효 구간의 대체가 되지 않습니다.

<a name="supported-pinned-entry"></a>

## 지원되는 고정 진입점

모든 동작은 `scripts/candidate.py --manifest <manifest> --manifest-sha256 <accepted-pin> --version 1.1`로 실행합니다. 실행 모드에는 `--project <absolute-project>`와 `--run-id <fresh-run>`도 필요합니다.

| 모드 | 입력 | 동작 |
| --- | --- | --- |
| `workflow validate` | 입력 없음 | 승인된 패키지와 역할 정책을 검증한 뒤 비활성 이름 공간을 연결합니다. |
| `workflow <typed-command>` | `--input <json-file>` 또는 표준 입력 | 위에 설명한 타입 지정 계약대로 실행합니다. |
| `coordination` | `--input <json-file>` | 모델 없이 범위가 정해진 배치를 실행합니다. Codex 작업을 자동 배정하지 않습니다. |
| `watcher` | `start`, `renew`, `stop`, `status`, `once` | 내부 고정 감시 프로세스와 해당 실행의 상태를 사용합니다. |
| `install` | 명시적 `--destination .../agent-workflow-v1-1`, 선택적 `--update` | 패키지 내용과 설치 소유권을 검사합니다. |

`workflow.py` CLI를 직접 호출하면 거부됩니다. 설치, 내보내기, Codex 역할 생성 진입점이 없습니다. `tooling/build-candidate.py`로 배포 파일을 만들고 Codex에는 고정 후보 설치 도구만 사용하세요. 이전 스키마 변환은 후보 CLI 명령이 아니며 폐기한 설치본의 실행을 재개하는 데 쓰면 안 됩니다.

<a name="input-and-state-boundary"></a>

## 입력과 상태의 경계

올바른 불리언 명령은 임시 환경에서 재현할 때도 같은 판단을 냅니다. 선택 플래그는 안전한 false 기본값을 쓸 수 있습니다. 참으로 해석될 수 있더라도 타입이 틀리면 거부합니다. 엄격한 상태 파싱으로 JSON 구문과 수치의 유한성을 검사합니다. 문자열 승인과 구버전 상태는 거부합니다. 테스트는 임시 프로젝트만 사용합니다.

<a name="typed-assignment-and-source-chronology-a04--gi01"></a>

## 타입 지정 작업 배정과 출처 시각 순서 (A04 / GI01)

N02의 `bind-active-goal`은 초기화가 오래 걸렸을 때 사용자 출처를 꾸미거나 시간을 바꾸지 않고 현재 활성 Codex 목표를 연결합니다. 상태가 없거나 정확히 `{schema_version: 2, status: inactive, tasks: {}}`여야 합니다(실제 JSON에서는 문자열에 따옴표 사용). 이전됐거나 완료·중단·중단 확정됐거나 이미 쓰인 등 초기 상태가 아닌 보조 프로그램에는 연결할 수 없습니다. 최초 관측은 0~60초 이내의 실제 `get_goal` 근거이며 목표·채팅·목표 문구·원래 생성 시각·관측 시각이 연결과 일치해야 합니다. 목표 생성은 오래전이어도 되지만 관측 뒤일 수는 없습니다. 명령 목표와 목표 범위가 같아야 합니다. 보고서 범위나 넓어진 범위는 거부합니다. 주 에이전트가 현재 컨텍스트와 이후 제한을 확인했다는 별개의 비어 있지 않은 해석과 절차 검사 근거를 남깁니다. 보조 프로그램은 그 확인을 인증할 수 없습니다.

상태는 현재 근거를 `native_binding`에 저장하고 `native_binding_events`와 `native_binding_ids`를 사용자 승인 원장과 구분합니다. 각 이벤트에는 `binding`, `interpretation`, `procedural_check`, 원래 `app_goal_observation`이 있습니다. 연결 ID를 예약해 이후 사용자 이벤트 ID와 충돌하지 않게 합니다. 실행·범위 ID도 기존 기록과 함께 예약합니다. 이 연결을 사용해 작업하려면 본문과 생성 시각이 같은 최신 목표 관측이 필요합니다. 사용자 승인은 기존의 정확한 출처 스키마와 원래 출처 300초 규칙을 따릅니다. 사용자 재개는 현재 Codex 연결 근거를 대체하면서 구분된 Codex 기록을 보존합니다. 이후 사용자 활성화도 같은 기록을 보존합니다. 잘못되거나 누락·중복·모순된 Codex 기록이나 예약은 사용자 계속 요청 이후에도 거부합니다. 중단, 스냅샷, 중단 확정 규칙은 계속 적용되며 최신 활성 목표 관측만으로 정지된 보조 프로그램을 자동 재개하지 않습니다.

Schema 2는 폐기된 작업 필드 `inputs`를 거부합니다. 다시 검증할 때 예전 문자열을 명시적으로 `context_refs`로 보존할 수 있습니다. 컨텍스트 문자열을 파일이나 작업 참조로 해석하지 않습니다. `file_inputs`는 정확히 `{path, revision}` 객체의 목록입니다. 배정은 프로젝트 상대 파일 경로를 정규화하고 대소문자 무시 중복, 범위 이탈, 예약 경로, 링크를 거부한 뒤 `sha256`과 `present`를 고정합니다. 파일 부재는 null과 false로 기록합니다. 변경 불가능한 입력 파일은 같은 작업의 쓰기 경로와 겹칠 수 없습니다. 쓰기 파일의 배정 전 기준 해시는 별도로 보존합니다. 각 작업에는 명시적 리비전과 전체 약정의 해시가 있으므로 입력 또는 작업 리비전이나 해시를 바꾼 뒤 조용히 해시를 다시 계산해도 이전 결과를 승인할 수 없습니다.

`review_targets`는 검토자에게만 허용되는 정확한 `{producer_id, revision}` 객체 목록입니다. 각 작성자가 그 리비전에 존재해야 하고 배정과 검토 시 모두 최신 관측에서 idle, done, stopped 중 하나여야 합니다. 배정 시 산출물 파일의 정확한 해시와 존재 여부를 고정합니다. 컨텍스트 참조는 검토 권한이 아닙니다. 준비된 인자에 타입 지정된 고정 정보가 모두 포함됩니다. prepare, record, 검토자의 finish와 작성자 결과 승인 시 입력 파일, 작성자 약정 및 결과를 다시 확인합니다. 승인된 의존 관계도 다시 검사합니다. A05의 accepted/running 상태 머신은 그대로입니다. 독립 검토를 받는 작성자는 검토 중 쓰기를 멈춘 것으로 관측돼도 상태는 running으로 남을 수 있습니다.

작업 약정이 오래되면 `INPUT_INVALIDATED`와 `invalidation` 기록을 반환합니다. 이 기록은 task_id, 배정 리비전, 사유, 해당 시 정확한 변경 경로, 기대 및 실제 해시나 리비전, at, 정렬된 affected_task_ids를 담습니다. 같은 경로의 소비 작업은 자체 고정 해시나 파일 존재 또는 배정 약정이 관측값과 맞지 않을 때만 무효화의 출발점이 됩니다. 현재 바이트나 파일 부재에 맞게 고정한 더 새 리비전은 이전 해시가 무효여도 유지됩니다. 그 뒤 실제 의존과 검토 관계를 따라 무효화합니다. 컨텍스트 문자열이나 경로 일치만으로는 관계가 생기지 않습니다. 동작 상태는 그대로지만 `contract_invalid: true`와 지속적인 기록이 이전 산출물이나 검토의 승인을 막습니다. 이전 바이트를 복원해도 이전 작성자나 승인된 검토를 살릴 수 없습니다. 검사를 반복해도 기록과 상태 바이트를 보존합니다.

입력, 검토 결과, 의존 대상 결과를 관측할 때 해시를 계산하기 전에 심볼릭 링크, 정션, 하드 링크 파일, 디렉터리 대체, 범위 이탈을 거부합니다. 위험한 산출물은 검토 배정, prepare, record, finish, 작성자 승인에서 오류를 반환하기 전에 작성자 및 영향받는 검토·소비 작업을 무효화합니다. 기록에는 기대 해시 또는 null, 정확한 상대 경로, 통제된 관측 사유를 담아 `unsafe: `로 시작하는 `actual` 문자열을 보존합니다. 파일이 없으면 null을 기록합니다. EIO 등 무관한 I/O 오류는 약정 변경으로 오인하지 않고 그대로 전달합니다. 확인하는 것은 관측한 경로입니다. 외부 동시 교체에 대해 파일 시스템 전체의 원자적 스냅샷을 제공하지 않습니다.

무관한 부분 변경은 모두 버립니다. 직접 `apply`를 호출하면 전달된 상태에 무효화 기록만 남깁니다. execute는 오류를 내기 전에 이 기록을 원자적으로 저장합니다. 초기화하거나 해시를 다시 계산하는 명령은 없습니다. 기록을 확인하고 새 작업 ID와 리비전을 최신 해시와 함께 명시적으로 배정하세요. 다른 검사 실패는 기존 바이트 보존 정책을 따릅니다.

GI01은 승인과 정확한 출처 필드 복사본인 source_message 모두에 source_created_at과 original_turn_id를 요구합니다. observed_at과 observed_turn_id는 관측을 따로 설명합니다. 출처 생성이 관측보다 앞서야 하고 새로 받아들일 때 300초 이내여야 합니다. 재개에는 출처 생성 시각이 중단, 중단 확정 또는 완료보다 뒤여야 합니다. 관측 시각이 뒤라는 것만으로 충분하지 않습니다. 마지막 승인 출처의 시각 기준은 재개와 새 활성화 뒤에도 남습니다. 생성 시각은 역행할 수 없으며 같은 시각이면 원래 턴이 같아야 합니다. 이벤트 ID 및 원본 출처 ID의 재사용 금지는 종류, 관측 시각, 실행·채팅, 계속 요청이 달라져도 유지됩니다. 관측 이벤트는 현재 목표·실행·채팅에 맞아야 하며 명시적 중단은 목표 범위에도 맞아야 합니다. 향후 의도, 정책, 보고서 관측은 근거로 남지만 현재 목표의 실행이나 재개를 승인하지 않습니다. 사용량 임계값은 보조 프로그램을 기계적으로 중단시키며 앱 목표 관측을 바꾸거나 사용자 일시 정지 승인을 만들지 않습니다.

이 필드는 호출자가 제공한 근거이며 공급자를 인증하지 않습니다. 보조 프로그램은 Codex 목표 도구의 직접 호출을 가로채지 않고 update_goal을 직접 호출하지도 않습니다. 과거 중단 뒤 새 계속 요청과 같은 턴의 컨텍스트 재사용을 순서대로 적용한 통제 회귀 검사는 보조 프로그램이 거부하고 활성 목표 관측이 그대로임을 보여 줍니다. N07은 위에서 설명한 실행과 승인에 대한 버전 지정 뷰를 추가합니다. 새 Codex 하위 세션의 컨텍스트·캐시 측정과 A06/GI02 일시 정지 통합은 여전히 별도 근거가 필요합니다.
