<a name="install-verify-and-maintain-agwf"></a>

# AGWF 설치, 검증 및 유지보수

Python 3.11 이상과 Git이 필요합니다. 빌더와 후보 프로그램은 Python 표준 라이브러리를 사용합니다. 설치할 사용자가 쓸 수 있는 스킬 디렉터리를 선택하세요. 아래 명령은 PowerShell 기준입니다. Linux나 macOS에서는 해당 사용자의 Codex 루트 아래 알맞은 디렉터리를 고르고 셸 문법을 맞추세요. ACL 검사는 Windows 전용입니다.

<a name="build-from-the-selected-checkout"></a>

## 선택한 체크아웃에서 빌드

AGWF 저장소 루트에서 실행하세요. 빌드를 승인하기 전에 소스 커밋을 기록하고 로컬 변경을 살펴보세요. 출력 디렉터리는 새 경로여야 합니다.

```powershell
python --version
git rev-parse HEAD
git status --short
$build = Join-Path $env:TEMP ('agwf-build-' + [guid]::NewGuid().ToString('N'))
python -X utf8 -B tooling/build-candidate.py --output $build
if ($LASTEXITCODE -ne 0) { throw 'AGWF build failed.' }
$receipt = Get-Content -LiteralPath (Join-Path $build 'build-receipt.json') -Raw | ConvertFrom-Json
$pin = $receipt.manifest_sha256
$package = Join-Path $build 'agent-workflow-v1-1'
$entry = Join-Path $package 'scripts/candidate.py'
$manifest = Join-Path $package 'manifest.json'
```

빌더는 패키지 디렉터리, 결정적으로 만들어지는 ZIP, 빌드 기록을 생성합니다. 승인한 빌드 기록을 변경 가능한 패키지 외부에 보관하고 소스 커밋 근거와 함께 관리하세요. 설치 도구에는 승인된 `manifest_sha256`이 필요합니다. 예상하지 못한 변경 뒤에 다시 계산한 해시는 별개의 관측입니다. [패키지 연결과 설치](../skills/agent-workflow-v1-1/references/coexistence.md).

<a name="choose-a-human-or-agent-path"></a>

## 직접 설치 또는 에이전트를 통한 설치

현재 사용자의 Codex 스킬 루트에 설치하려면 아래 명령을 사용하세요. 임시 환경에 설치할 때는 대상이 `agent-workflow-v1-1`로 끝나는 격리 디렉터리를 지정하세요. 첫 설치와 동일한 설치본 확인에 같은 명령을 사용합니다.

```powershell
$codexRoot = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $env:USERPROFILE '.codex' }
$destination = Join-Path $codexRoot 'skills/agent-workflow-v1-1'
python -X utf8 -B $entry --manifest $manifest --manifest-sha256 $pin --version 1.1 install --destination $destination
if ($LASTEXITCODE -ne 0) { throw 'AGWF installation failed.' }
```

`activated:false`는 워크플로를 시작하지 않고 패키지 파일을 설치 또는 검증했다는 뜻입니다. Windows에서는 임시 디렉터리가 설치 대상의 상위 디렉터리 권한을 상속합니다. 실제 사용자의 일반 권한으로 설치 파일을 읽고 실행할 수 있는지 확인하세요.

에이전트로 설치하려면 다음과 같이 진행하세요.

1. 요청된 대상과 설치 유형을 확인합니다. 첫 설치, 업데이트, 격리 시험을 구분하세요. 기존 대상을 변경하기 전에 현재 패키지, 사용자의 수정 사항, 해당 설치본의 프로세스를 확인합니다.
2. 체크아웃 식별자와 Python 버전을 확인합니다. 새 디렉터리에 빌드하고 위와 같이 승인된 매니페스트 해시를 보관합니다.
3. 요청된 대상에 맞춰 고정 후보의 설치 도구를 사용합니다. 업데이트 전에는 관련된 이전 프로세스만 멈추고 종료됐는지 확인합니다. 설치 도구가 거부하면 기존 대상은 조사할 수 있도록 보존됩니다.
4. 설치된 매니페스트와 파일을 검사한 뒤 설치된 진입점에서 아래 수동 검사를 실행합니다. 소스 커밋, 매니페스트 해시, 대상, 결과 상태를 적고 워크플로를 띄우지 않은 검사인지 명시적으로 요청받아 실제로 띄운 실행인지 밝힙니다.

<a name="verify-the-installed-entrypoint"></a>

## 설치된 진입점 검증

새 임시 프로젝트와 고유한 실행 ID를 사용하세요. 이 작업은 임시 프로젝트에 실행 연결만 쓰며 프로세스를 시작하거나 목표를 활성화하지 않습니다.

```powershell
$probe = Join-Path $env:TEMP ('agwf-probe-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $probe -ErrorAction Stop | Out-Null
$installedEntry = Join-Path $destination 'scripts/candidate.py'
$installedManifest = Join-Path $destination 'manifest.json'
python -X utf8 -B $installedEntry --manifest $installedManifest --manifest-sha256 $pin --version 1.1 --project $probe --run-id install-check workflow validate
if ($LASTEXITCODE -ne 0) { throw 'AGWF binding validation failed.' }
```

응답의 `identity`, `version`, `run_id`, `manifest_sha256` 및 `missing_dependencies`를 확인하세요. 이 문서를 위한 격리 검사에서 설치는 `activated:false`를 반환했고, 검증은 워크플로 상태 파일이나 감시 디렉터리 없이 `run-binding.json`을 생성했습니다. 잘못된 매니페스트 해시는 종료 코드 2로 거부됐습니다.

실제 Codex 호출은 별도입니다. 이미 열려 있는 세션에서는 스킬 목록을 새로 고쳐야 할 수 있습니다. `$agent-workflow-v1-1`을 명시적으로 선택하고 작업 범위를 전달하세요. 승인된 시작과 감시 상태 확인은 [운영 안내서](../skills/agent-workflow-v1-1/references/operations.md)에 있습니다.

<a name="update-recover-or-remove"></a>

## 업데이트, 복구 및 제거

- 관리되는 설치본을 업데이트할 때는 기존 고정 진입점으로 관련 프로세스를 중단하고 종료를 확인하세요. 새 패키지는 별도로 빌드하고 승인하세요. 기존 설치본이 해당 설치 도구로 관리되고 수정되지 않았을 때만 설치 명령에 `--update`를 추가하세요. 사용자의 수정은 먼저 확인하고 반영 방법을 정합니다.
- 설치 도구는 교체 전에 준비된 패키지를 검사하고, 배치에 실패하면 이전 디렉터리 복원을 시도합니다. 복원도 실패하면 트랜잭션 디렉터리를 보존하세요. 업데이트가 성공하면 영구 롤백 아카이브를 남기지 않습니다. [설치 도구 세부 정보](../skills/agent-workflow-v1-1/references/coexistence.md).
- 기술 식별자가 다르면 새 디렉터리에 새로 설치해야 합니다. 이전 실행 연결은 원래 매니페스트와 함께 보관하고 설치된 스킬 검색 설정은 별도로 처리하세요. 패키지 업데이트가 승인되면 새 실행을 시작하세요. [마이그레이션 범위](../skills/agent-workflow-v1-1/references/migration.md).
- 제거 명령은 없습니다. 설치를 제거하기 전 정확한 스킬 디렉터리를 확인하고 해당 프로세스만 중단한 뒤 스킬 검색 경로 밖에 검증된 사본을 보관하세요. 사용자 프로젝트, 실행 상태, 체크포인트, 외부 기록을 보존하세요. 변경 뒤 스킬 검색 결과를 확인하세요.
