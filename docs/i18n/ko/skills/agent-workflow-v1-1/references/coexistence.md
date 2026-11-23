<a name="pinned-runtime-binding-and-installation"></a>

# 고정된 런타임 연결과 설치

기술 식별자는 `agent-workflow-v1-1`, 표시 이름은 `agent-workflow-v1.1`입니다. 암시적 호출은 꺼져 있습니다. 후보는 명시적으로 선택해야 사용합니다. 주 클라이언트 모델은 사용자가 정합니다. 워크플로 역할은 `gpt-6.1-sol`, `gpt-6-astra`, `gpt-6.1-terra` 중 하나와 `medium` 이상의 추론 강도만 사용합니다. 내부 감시 담당 역할은 Terra/medium으로 고정돼 있습니다. 대체 모델은 허용되지 않습니다.

현재 명시적으로 선택하면 주 에이전트가 같은 턴 안에 지정된 범위의 실행을 시작하고 검증해야 합니다. 이전에 꺼진 상태라도 해당 실행을 다시 켤 수 있습니다. 과거의 선택이나 패키지 유지보수는 다시 켜는 근거가 아닙니다. 전체 활성화 근거가 없거나 Codex 목표 생성을 허용하지 않았으면 [운영 안내서](operations.md)의 수동 감시 시작 절차를 따르세요. 시작을 확인한 뒤 본 작업으로 넘어가세요. 감시 없이 작업을 몰래 계속하지 마세요. 이후 중단 지시가 우선합니다.

Python 3.11 이상과 표준 라이브러리로 빌드하세요.

```powershell
python -X utf8 -B tooling/build-candidate.py --output C:/staging/build-one
```

출력에는 `agent-workflow-v1-1/`, 결정적으로 만들어지는 ZIP, `build-receipt.json`이 들어갑니다. 빌드 기록의 `manifest_sha256`을 승인된 조율 또는 인계 근거에 넣고 변경 가능한 패키지 밖에 보관하세요. 호출자가 공급하는 외부 해시가 신뢰 기준입니다. 실행 연결, 중요한 워크플로 상태 쓰기, 조율 단계, 작업 프로세스의 각 측정 전에 매니페스트와 의존성을 모두 검사합니다. 상태 신호를 게시할 때는 작은 실행·세대 제어 기록만 확인합니다. 매초 패키지를 순회하거나 해시하지 않습니다. 수정한 매니페스트의 SHA를 다시 계산해 전달해도 외부 승인이 입증되지는 않습니다. Python은 바이트와 식별자 무결성을 검사하며, 사람의 승인, 공급자와 모델 메타데이터, 호출자가 통제하는 파일 시스템을 인증할 수 없습니다. 실행 파일이나 인터프리터가 손상됐거나 호출자가 실행 연결과 승인된 외부 해시를 모두 바꾸는 경우는 이 무결성 계약의 범위 밖입니다.

전역 옵션을 모드 앞에 둡니다. 소스에서 실행할 때는 `skills/agent-workflow-v1-1/scripts/candidate.py`와 빌드한 매니페스트를 사용합니다. 설치본에서는 설치된 `scripts/candidate.py`와 매니페스트를 같은 외부 보관 해시와 함께 사용합니다. 각 소스 또는 설치 연결에 별도의 실행 ID를 사용합니다. 변경할 수 없는 연결에는 실제 의존성 경로, 매니페스트 경로, 프로젝트, 실행, 버전 및 해시가 들어갑니다. 실행 중 다른 진입점으로 옮기거나 패키지를 바꾸거나, 의존성이 바뀌거나 누락되거나, 같은 실행 ID에 다른 해시를 쓰면 거부합니다. 승인된 업데이트 뒤에는 새 실행을 시작하세요.

```powershell
$pin = '<preserved manifest_sha256 from accepted build receipt>'
$entry = 'C:/staging/build-one/agent-workflow-v1-1/scripts/candidate.py'
$manifest = 'C:/staging/build-one/agent-workflow-v1-1/manifest.json'
$common = @('-X','utf8','-B',$entry,'--manifest',$manifest,'--manifest-sha256',$pin,'--version','1.1','--project','C:/project','--run-id','candidate-run-1')
python @common workflow validate
python @common workflow status
python @common coordination --input C:/project/batch.json
```

승인된 `workflow activate` 또는 `bind-active-goal`이 감시 프로세스를 시작하고 `--usage-input`으로 받은 첫 정상 측정값을 검증합니다. `resume`은 유효한 계속 요청 뒤 또는 기존 승인과 활성 목표 아래에서 만료된 측정값을 검증 절차로 복구한 뒤 새 실행 세대를 엽니다. 새 Codex 측정 기록이 필요합니다. [운영 안내서](operations.md)와 [Codex 사용량 기록](native-usage.md)을 참조하세요. 측정을 위해 Codex 서버나 CLI를 시작하지 않습니다. 활성 명령은 실행 유지 기한을 연장합니다. `complete`와 `halt`는 해당 Python 프로세스를 중단하고 종료를 검증합니다. 명시적 `watcher` 명령에도 같은 측정 기록 규칙이 적용됩니다.

`workflow`는 코어의 타입 지정 상태 명령과 `validate`를 받습니다. 명령은 `--input` 또는 표준 입력으로 JSON을 읽습니다. 이전 코어 CLI `native/install/export/migrate`는 지원되지 않습니다. `activate` 승인과 `bind-active-goal` 근거에는 선택한 실행 ID가 있어야 합니다. `validate`는 역할과 의존성을 검사하고 목표를 활성화하지 않은 채 이름 공간을 연결합니다. 배치 형식은 `{ "batch_id": "inspect", "steps": [{ "key": "status", "command": "status", "payload": {} }] }`입니다. 최대 64단계가 순차 실행되며 주 에이전트만 씁니다. 대기 중인 결과가 지속적으로 기록됩니다. 자동 재시도나 Codex 배정은 수행하지 않습니다. 실제 작업 명령에는 계속 현재의 타입 지정 승인과 결과 승인 근거가 필요합니다. 에이전트 목록의 종료 상태만으로 산출물을 승인할 수 없습니다. 더 이상 쓰지 않는 오래된 작업은 `finish accepted:false`로 버릴 수 있습니다. 작성 중이면 이 명령을 쓸 수 없습니다. 상태를 압축하려면 검증된 체크포인트가 필요하며 작업과 무효화 기록은 보관됩니다.

격리 루트는 `<project>/.agent-workflow-v1-1/runs/<run-id>/`입니다. 동작 상태 루트는 예약되어 작업 쓰기 경로로 지정할 수 없습니다. 내장 감시 프로세스는 해당 위치의 `watcher/`, 명시적으로 고정된 일정 도구, 변경 불가능한 소유권 기록, 후보 진입점으로 다시 실행할 인자를 사용합니다. 다른 실행의 제어나 내부 CLI 직접 호출로 다른 프로세스의 기한을 갱신하거나 종료할 수 없습니다. 채팅이 조용해도 제한된 기한이 연장되지는 않습니다. 작업 경계에서 활성 후보 명령을 사용하세요. 패키지를 업데이트해도 동작 중인 프로세스가 자동으로 옮겨지지 않습니다. 감시용 별도 스킬이나 실행 도구는 없습니다. 내부 의존성은 승인된 패키지에서 불러옵니다.

새 설치, 변경할 필요가 없는 확인, 관리되는 업데이트에는 이름이 `agent-workflow-v1-1`로 끝나는 명시적 대상이 필요합니다.

```powershell
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination C:/staging/skills/agent-workflow-v1-1 --update
```

업데이트에는 설치 도구가 관리하고 바이트가 바뀌지 않은 설치본이 필요합니다. 설치 도구는 기존 디렉터리를 교체하기 전에 준비된 패키지를 검증합니다. 배치 실패 시 이전 디렉터리를 복원합니다. 성공하면 트랜잭션 자료를 제거합니다. 영구 롤백 아카이브나 롤백 명령은 없습니다. 복원도 실패하면 트랜잭션 디렉터리에 이전의 유일한 사본이 남습니다. 수동 복구가 확인될 때까지 이 디렉터리를 지우지 마세요. `activated:false`가 나타나는 이 명령들은 파일만 준비합니다. 스킬 목록 표시, 실제 명시적 선택, 공급자가 사용한 실제 모델 메타데이터, 실제 제품이나 시범 실행의 승인은 설치 뒤에 별도로 검사합니다.

Windows에서는 패키지 파일을 복사하기 전에 준비 디렉터리의 ACL을 대상의 상위 디렉터리에서 상속하도록 초기화합니다. 특정 Windows 계정을 지정하지 않습니다. 다른 계정이나 이름을 바꾼 계정은 그 사용자의 상위 `skills`에 이미 부여된 권한을 받습니다. 상위 디렉터리는 대상 사용자에게 읽기와 실행 권한을 부여해야 합니다. 설치가 끝나면 권한을 높이지 않은 일반 Codex 세션에서 스킬 검색과 후보의 직접 실행을 확인하세요. 관리자 권한에서의 성공만으로는 충분하지 않습니다.
