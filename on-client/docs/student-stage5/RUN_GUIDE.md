# Keshev 5단계 실행 안내

기준일: 2026-10-31 · 현재 Git 작업 폴더의 앱: `git-stage6/on-client` · 시작 틀 버전: `0.2.21`.

이 문서는 이번 PC에서 발생한 실행 오류와 확인 결과를 정리한 학생 문서입니다. 기존 문서와 제출물은 보존했습니다. 5단계 실행 확인은 원래 `main/on-client`에서 했고, 이후 같은 기준 커밋을 복제한 `git-stage6`에 승인한 두 파일을 복사했습니다. 예전 `STUDENT_RUNBOOK.md`와 `STUDENT_REPRODUCTION.md`의 `0.2.11`, 이전 저장소·브랜치 정보는 과거 기록이며 이번 실행 기준과 구분합니다.

## 1. 처음 막힌 원인

| 현상 | 확인한 원인 | 이번에 한 조치 |
|---|---|---|
| `node --version`에서 `CommandNotFoundException` | 현재 PowerShell이 Node.js 실행 파일을 찾지 못함. 일반 설치 경로에도 없다는 사용자 확인 | 프로젝트 밖에 설치하지 않고 허용 루트 아래에 공식 ZIP 배포본 준비 |
| 앱의 Windows 기록만 `확인 불가(query_failed)` | 앱이 실행한 Windows PowerShell에서 `CouldNotAutoloadMatchingModule` 발생. 일반 사용자 권한의 직접 조회는 성공 | 앱을 시작하는 프로세스에서만 `PSModulePath` 상속을 비우자 조회 성공 |
| 브라우저 `ERR_CONNECTION_TIMED_OUT` | 처음 실행한 임시 서버 세션이 사라진 것을 확인. 당시 네트워크 상태 전체를 검사한 것은 아님 | 별도 서버 프로세스로 다시 실행하고 로컬 HTTP 200 확인 |
| MinGit의 `diff`에서 `cannot spawn less` | 출력 페이지 도구 `less`를 실행하지 못함 | 차이 조회 명령에 Git 전역 옵션 `--no-pager` 사용 |

처음 두 문제는 이번 실행 환경의 준비·상속 문제로 진단했습니다. 앱 서버나 제공 공통 화면 코드는 고치지 않았습니다. 기존 안내에는 Node.js 준비 조건이 있었지만, 이 PC의 설치 상태 확인·ZIP 사용·모듈 경로 충돌 대응이 필요했습니다.

## 2. 폴더와 준비 조건

이 PC에서 허용된 루트는 다음입니다. 다른 PC에서는 실제 프로젝트 위치에 맞춰 `$keshevRoot`만 바꾸고, 접근 허용 범위를 먼저 정합니다.

```text
C:\Users\K_501\Desktop\ALEPH\Keshev
├─ main\on-client                 원래 작업 폴더, 그대로 보존
├─ git-stage6                     현재 Git 작업 폴더 (.git 포함)
│  └─ on-client                   앱 소스와 package.json
│     ├─ student\extension.mjs     학생 화면 코드
│     └─ docs\student-stage5\RUN_GUIDE.md
└─ stage5-runtime
   ├─ node-v24.19.0-win-x64.zip
   └─ node-v24.19.0-win-x64\node.exe
```

- 이번에 확인한 CPU 종류: `AMD64`. Windows x64 배포본 사용.
- 프로젝트 `package.json`이 지정한 Node.js 버전: `24.19.0`.
- 앱 `package.json`에는 외부 npm 의존성이 없습니다. 이 앱 실행에 `npm install`은 필요하지 않습니다. 저장소 최상위의 별도 `package.json`과 혼동하지 않습니다.
- 실행·다운로드는 일반 권한으로 진행했습니다. 관리자 권한은 사용하지 않습니다.
- ZIP 배포본은 `main` 밖의 `stage5-runtime`에 두며 앱 소스에 함께 올리지 않습니다.

## 3. Node.js 준비

이번 PC에서는 다운로드·해시 확인·압축 해제·버전 확인까지 완료했습니다. 이미 준비됐다면 4절로 갑니다.

처음 준비하는 사람은 PowerShell에서 아래 경로 변수를 설정합니다.

```powershell
$keshevRoot = 'C:\Users\K_501\Desktop\ALEPH\Keshev'
$keshevRuntime = Join-Path $keshevRoot 'stage5-runtime'
$keshevZip = Join-Path $keshevRuntime 'node-v24.19.0-win-x64.zip'
$keshevNodeFolder = Join-Path $keshevRuntime 'node-v24.19.0-win-x64'
Set-Location -LiteralPath $keshevRoot
if (-not (Test-Path -LiteralPath $keshevRuntime)) {
    New-Item -ItemType Directory -Path $keshevRuntime | Out-Null
}
```

공식 ZIP을 다운로드합니다. 기존 ZIP이 있으면 덮어쓰지 않고 다음 검증으로 넘어갑니다.

```powershell
if (-not (Test-Path -LiteralPath $keshevZip)) {
    Invoke-WebRequest -UseBasicParsing -Uri 'https://nodejs.org/download/release/v24.19.0/node-v24.19.0-win-x64.zip' -OutFile $keshevZip
}
$keshevExpectedHash = '57F71AB3652E797D84ACDDC79C81CC9FF1C6DDB2A1974CDB83F00FEE9BFF4C73'
$keshevActualHash = (Get-FileHash -Algorithm SHA256 -LiteralPath $keshevZip).Hash
if ($keshevActualHash -ne $keshevExpectedHash) {
    throw '공식 SHA256 값과 다릅니다. 압축을 풀거나 실행하지 말고 다운로드 결과를 확인하세요.'
}
if (-not (Test-Path -LiteralPath $keshevNodeFolder)) {
    Expand-Archive -LiteralPath $keshevZip -DestinationPath $keshevRuntime
}
& (Join-Path $keshevNodeFolder 'node.exe') --version
```

정상 기대 결과는 `v24.19.0`입니다. 이 버전은 이번 PC에서도 실제 출력으로 확인했습니다. SHA256 기준은 [Node.js 공식 검증 목록](https://nodejs.org/download/release/v24.19.0/SHASUMS256.txt)입니다. 기존 압축 해제 폴더의 상태를 알 수 없다면 덮어쓰지 말고 먼저 확인합니다.

## 4. 앱 실행

### 4.1 실행 위치와 문법 확인

새 PowerShell 창에서는 아래 변수를 다시 설정합니다. 전체 경로로 Node.js를 호출하므로 전역 PATH를 바꿀 필요가 없습니다.

```powershell
$keshevRoot = 'C:\Users\K_501\Desktop\ALEPH\Keshev'
$keshevApp = Join-Path $keshevRoot 'git-stage6\on-client'
$keshevNode = Join-Path $keshevRoot 'stage5-runtime\node-v24.19.0-win-x64\node.exe'
Set-Location -LiteralPath $keshevApp
& $keshevNode --version
& $keshevNode --check .\student\extension.mjs
```

`--check`에서 오류 없이 입력 줄로 돌아오면 문법 확인이 끝난 것입니다. 화면 동작까지 검증하는 명령은 아닙니다.

### 4.2 서버 켜기

이 안내는 SDP 어댑터·AI·계정 호스트를 별도로 설정하지 않은 PC 조회 실행 기준입니다. 기존 담당자 설정이 있는 환경은 그 설정의 경로와 사용 범위를 먼저 확인합니다.

5189 포트에서 이미 이 앱을 실행 중이라면 그 서버로 확인하거나 실행한 주체에게 종료를 요청합니다. 이번 대화에서 도구가 켠 백그라운드 서버는 이 대화에서 종료를 요청할 수 있습니다. 다른 프로그램을 자동으로 종료하지 않습니다.

```powershell
$keshevPreviousModulePath = $env:PSModulePath
$keshevPreviousPort = $env:ON_PORT
try {
    $env:PSModulePath = $null
    $env:ON_PORT = '5189'
    & $keshevNode .\src\server.mjs
}
finally {
    $env:PSModulePath = $keshevPreviousModulePath
    $env:ON_PORT = $keshevPreviousPort
}
```

`PSModulePath` 조정은 이 실행과 하위 프로세스에만 적용됩니다. 영구 환경변수나 Windows 보안 설정을 변경하지 않습니다. 이 PC에서 발생한 Windows PowerShell 모듈 자동 로딩 실패를 피하기 위한 조치입니다.

정상 시작 시 다음 줄이 나옵니다.

```text
케셰브 연결 확인: http://127.0.0.1:5189
```

서버를 실행한 창을 열어 둔 채 브라우저에서 [Keshev](http://127.0.0.1:5189/)를 엽니다. 서버 시작은 SDP·AI 연결 성공을 뜻하지 않습니다.

기존 `tools/start-alekov.ps1`의 기본 실행은 사용자 홈에 계정 연결 보관소를 준비하는 경로가 있습니다. 이번 폴더 제한에 맞춘 PC 조회 확인에서는 위의 서버 직접 실행을 사용합니다.

### 4.3 종료와 재실행

- 이 문서의 명령으로 켰다면 해당 PowerShell 창에서 `Ctrl+C`를 누릅니다. 그 서버가 단독으로 실행 중이었다면 이후 브라우저 요청은 실패해야 합니다.
- 재실행은 같은 창에서 4.2의 명령 블록을 다시 실행합니다.
- `EADDRINUSE`가 나오면 5189 포트가 사용 중입니다. 어떤 서버인지 확인하거나 실행 블록의 포트를 `5190`으로 바꾸고 브라우저도 `http://127.0.0.1:5190/`으로 엽니다.
- 강제 프로세스 종료, 방화벽 변경, 관리자 재실행으로 해결하려 하지 않습니다.

## 5. 화면에서 확인하는 순서

1. 기본 화면 열림을 확인합니다. SDP 미연결 실행에서는 로그인 정보·업무 앱·보안 기록·기기·세션 조회가 연결 오류를 표시할 수 있습니다. 예시 값으로 채워 연결 성공처럼 보이면 안 됩니다.
2. 제공 카드 `내 PC 상태`에서 `내 PC 확인`을 한 번 누릅니다. CPU·메모리·디스크·네트워크·방화벽·Windows 기록의 결과와 측정 시각을 봅니다. 자동 조회 확인란은 처음에는 선택하지 않습니다.
3. 학생 영역 `PC 상태와 변화 비교`에서 `비교용 조회`를 누릅니다. 실측 항목 수와 시각을 확인합니다. 첫 조회에는 이전 값이 없으므로 비교가 나오지 않을 수 있습니다.
4. 잠시 뒤 `다시 조회`를 누릅니다. 이전·이번 시각과 증가·감소·변화 없음 표시를 확인합니다. 대기 중에는 조회 중 안내가 보이는지 확인합니다.
5. `PC 보안 관찰 요약`에서 네트워크·방화벽·Windows 기록, 조회 범위, 확인 불가 이유를 봅니다. 경고·오류 건수는 침해나 공격 탐지 결과가 아니라는 문구를 확인합니다.
6. 학생 영역의 `SDP 미연결` 표시를 확인합니다. PC 실측과 SDP 상태는 별개입니다. 이번 제출 캡처에는 이 표시 자체가 포함되지 않았으므로 캡처 확인 완료로 쓰지 않습니다.
7. 키보드 확인은 `Tab`으로 조회 버튼에 이동하고 `Enter` 또는 `Space`로 실행합니다. 접힌 상세도 키보드로 열 수 있는지 확인합니다. 이 절차의 전체 키보드 실사용 검증은 이번 기록에서 미확인입니다.

읽기에 실패한 항목은 `확인 불가`와 이유를 보여야 합니다. 이전 값이나 예시 값을 최신 실측처럼 표시하면 안 됩니다. 실패 안내 뒤에 사용자가 직접 다시 조회합니다. 공격·침투 시험은 하지 않습니다.

## 6. 값과 제공 기능의 구분

| 구분 | 제공 기능 | 학생 구현 |
|---|---|---|
| PC 측정 | `src/pc-status.mjs`, `/api/pc/status`, 제공 `내 PC 상태` 카드 | 같은 API 응답의 두 시점 비교·원형 그래프·보안 관찰 요약 |
| 데이터 출처 | `source: local-pc`, `measuredAt`, 항목별 상태와 단위 | `[실측]` 표시, 조회 시각·변화량·실패 이유 표시 |
| 용량 단위 | 제공 카드 문구는 원본 그대로 유지 | 1024³바이트를 `GiB`로 표시. 계산식은 그대로이며 단위 이름을 바로잡음 |
| SDP·AI | 실제 어댑터·권한·운영 설정이 있어야 연결 확인 가능 | 실측 성공과 연결 상태를 분리해 안내 |

CPU 사용률은 `%`, 두 시점의 차이는 `%p`, 메모리·디스크는 `GiB`, 기록 수는 `건`입니다. CPU는 200ms 표본입니다. 두 번의 관찰만으로 지속 추이·변화 원인·침해 여부를 판단하지 않습니다.

예제 모드의 사건·사용자·기기·세션은 설명용 자료입니다. 기본 PC 조회 확인에서는 예제 모드를 사용하지 않았습니다. 이 문서의 오류 문구와 실행 예시 명령은 설명이며 PC 실측값이 아닙니다.

PC 조회에는 일반 권한으로 Windows PowerShell 실행과 Windows 정보 읽기가 필요합니다. 이번 사용자가 허용한 범위는 다음과 같습니다.

- 실행 파일: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`.
- CPU·메모리·네트워크 개수, 앱 폴더가 있는 드라이브의 용량 정보.
- `Get-NetFirewallProfile`의 프로필 설정 읽기.
- `Get-WinEvent`의 `System`·`Application` 최근 24시간 경고·오류 집계. 화면 집계는 최대 5,000건이며 잘리면 일부 집계로 표시합니다.

방화벽·보안 설정은 바꾸지 않습니다. 일반 권한으로 읽지 못한 값은 확인 불가로 남깁니다. 새 PC에서도 프로젝트 폴더 밖 Windows 정보 읽기가 허용되는지 먼저 확인합니다.

## 7. 이번에 확인한 근거와 한계

시각은 한국 표준시(KST, UTC+9)입니다. 아래 수치는 이번 PC에서 관찰한 과거 값이며 현재 값이나 다른 PC의 예시 값으로 대신 사용하지 않습니다.

| 근거 | 시각·내용 | 확인 수준 |
|---|---|---|
| Node.js | 공식 ZIP의 SHA256 일치, `v24.19.0` 출력 | 도구로 직접 확인 |
| PC API | 2026-10-31 14:36:17 KST, HTTP 200, `source: local-pc`, 6개 항목 모두 `ok` | 도구로 직접 확인 |
| 실측 일부 | 위 시각 CPU 47.5%(200ms 표본), 최근 24시간 경고 38건·오류 29건 | 실제 API 값. 공격 탐지 결과 아님 |
| 직접 실행 재검증 | 2026-10-31 14:46:38 KST, `src/server.mjs`를 5190 포트에서 실행. 페이지·PC API HTTP 200, 6개 항목 `ok`, `mode: disconnected`. 검증용 서버 종료 | 도구로 직접 확인. 기존 5189 서버는 별도 유지 |
| 제공 PC 카드 | 사용자가 화면에서 동작한다고 보고 | 사용자 확인 |
| 학생 비교 화면 | 첨부 캡처에 실측 3/3, 확인 불가 0개, 이전 14:37:28·이번 14:37:48, 변화 방향 표시 | 캡처 확인. 모든 변화량의 계산 정확성을 별도로 대조한 것은 아님 |
| 보안 관찰 | 캡처에 3/3 조회, 확인 불가 0개, 침해·공격 탐지 결과가 아니라는 안내 | 캡처에 보이는 범위 확인 |
| 문구 수정 후 배포 원문 | 로컬 서버가 수정된 학생 모듈을 HTTP 200으로 제공하며 `PC 상태와 변화 비교`·`GiB`가 포함됨 | HTTP 원문 확인. 수정 후 실제 브라우저 표시는 도구로 직접 관찰하지 않음 |
| 종료·재실행 | 2026-10-31 15:15:48 KST, 도구가 켠 서버를 종료하자 로컬 HTTP 연결 실패. 기존 `src/server.mjs`로 다시 실행한 뒤 페이지·PC API HTTP 200, 6개 항목 `ok`, `mode: disconnected` | 도구로 직접 확인. 재실행한 5189 서버는 유지 |

파일 수정 후 학생 코드의 `node --check`가 통과했고, 이 문서의 PowerShell 코드 블록 5개의 구문을 검사했습니다. 제공 화면·서버·실행 도구·꾸밈 설정 등 53개 파일의 해시가 수정 전과 일치했습니다. 변경한 두 파일에서 일반적인 토큰·JWT·개인키 패턴은 발견되지 않았습니다.

미확인: 실제 SDP·AI 연결, 실제 업무 앱 접속·조치 집행, 전체 키보드 동선, 서버 중지 상태에서 학생 화면이 이전 값을 제거하는 동작(HTTP 연결 실패와는 별도), 다른 PC에서의 처음부터 전체 재현, 현재 복사본의 커밋 일치 여부.

기준 커밋은 `3449089dafb5871b2e97a42a76897b34c06b1a13`입니다. 원래 `main`에는 `.git`이 없었으므로 별도 폴더 `git-stage6`에 저장소를 복제했고, 이 폴더에서 `git rev-parse HEAD`가 기준 커밋과 일치함을 도구와 사용자가 확인했습니다. 이 값은 이번 변경을 커밋하기 전의 기준점이며 새 제출 커밋이 아닙니다. 저장소는 [msk4905/sktaleph_Keshev](https://github.com/msk4905/sktaleph_Keshev)입니다.

## 8. 이번 변경과 수동 커밋

- 수정: `on-client/student/extension.mjs`의 학생 화면 문구와 용량 단위 이름.
- 새로 작성: `on-client/docs/student-stage5/RUN_GUIDE.md`.
- 제공 공통 화면·서버·실행 도구·꾸밈 설정은 유지합니다. 공식 포장 명령과 새 포장 스크립트는 사용하지 않습니다.
- 로컬 Node.js ZIP·실행 파일, 서버 로그·검토용 사본은 두 프로젝트 폴더 밖의 `stage5-runtime`에 둡니다. Git은 `stage6-runtime`의 공식 MinGit 배포본을 사용하며 이 도구도 커밋 대상에 넣지 않습니다.

커밋·푸시는 사용자가 직접 합니다. Git 작업은 `.git`이 있는 `git-stage6`에서 합니다. 이 안내 작성과 파일 복사는 커밋·푸시 완료를 뜻하지 않습니다.

아래는 수동으로 실행할 명령입니다. 명령은 한 줄씩 실행하며 오류가 나면 다음으로 넘어가지 않습니다. 이 PC에서는 `git`도 처음에 실행 파일을 찾지 못해, 공식 MinGit `2.56.0.windows.4`를 허용 루트에 준비하고 공식 SHA256 및 버전을 확인했습니다. 시스템 PATH를 바꾸는 대신 전체 경로로 호출합니다.

```powershell
Set-Location -LiteralPath 'C:\Users\K_501\Desktop\ALEPH\Keshev\git-stage6'
if (-not (Test-Path -LiteralPath .\.git)) { throw 'Git 이력이 없는 복사본입니다. 기존 Git 작업 폴더를 먼저 확인하세요.' }
$keshevGit = 'C:\Users\K_501\Desktop\ALEPH\Keshev\stage6-runtime\MinGit-2.56.0.4-64-bit\cmd\git.exe'
$env:GIT_CONFIG_NOSYSTEM = '1'
$env:GIT_CONFIG_GLOBAL = 'C:\Users\K_501\Desktop\ALEPH\Keshev\stage6-runtime\gitconfig'
& $keshevGit status --short
& $keshevGit remote -v
& $keshevGit --no-pager diff -- on-client/student/extension.mjs
& $keshevGit add -- on-client/student/extension.mjs on-client/docs/student-stage5/RUN_GUIDE.md
& $keshevGit --no-pager diff --cached --check
& $keshevGit --no-pager diff --cached --stat
& $keshevGit --no-pager diff --cached -- on-client/student/extension.mjs on-client/docs/student-stage5/RUN_GUIDE.md

# 작성자가 아직 설정되지 않은 경우에만 입력합니다. 이메일은 채팅에 보내지 않습니다.
$keshevAuthorName = Read-Host '커밋 표시 이름 (GitHub 사용자명 권장)'
$keshevAuthorEmail = Read-Host 'GitHub 이메일 설정에서 확인한 noreply 이메일'
& $keshevGit config --local user.name $keshevAuthorName
& $keshevGit config --local user.email $keshevAuthorEmail

& $keshevGit commit -m "PC 조회 실행 안내와 오류 대응 개선"
& $keshevGit push
& $keshevGit rev-parse HEAD
```

커밋 전에는 포함 파일과 공개하면 안 되는 값이 없는지 직접 확인합니다. `git remote -v`의 대상이 본인 저장소인지 확인하고, 다른 작업이 이미 스테이징돼 있다면 함께 커밋하지 않도록 먼저 범위를 검토합니다. push가 거절되면 오류를 확인하고 원인을 해결합니다. 강제 푸시는 하지 않습니다.
