# Codmes KNU Plugin

공주대학교 KNU AI Assistant를 Codmes의 네이티브 Surface와 AI 도구로 연결하는
공식 커뮤니티 플러그인입니다.

이 저장소에는 KNU 웹이나 백엔드 서버가 포함되지 않습니다. 공지 수집, 포털·LMS
로그인, 데이터 동기화와 검색은
[knu-uic/knu-ai-assistant](https://github.com/knu-uic/knu-ai-assistant)의
KNU API/MCP 서버가 담당합니다.

## 제공 기능

- 공지, LMS, 포털, 설정으로 구성된 Codmes 네이티브 Surface
- 공주대 포털 계정 연결과 로그인 상태 표시
- 공지 검색 및 상세 근거 조회 MCP 도구
- 로그인 학적정보를 이용한 학교 공통·사용자 학과 공지 자동 범위 설정
- macOS, iOS와 iPadOS 지원

KNU 웹사이트를 WebView나 iframe으로 열지 않습니다. `surface.json`이 화면 구조와
데이터 바인딩을 선언하고, Codmes Apple 클라이언트가 SwiftUI로 렌더링합니다.
공지와 LMS는 standalone React 웹의 정보 계층을 반영해 출처·학과, 게시일·마감일,
상태 badge, 요약과 tag가 구분된 native card로 표시합니다. React의 CSS를 복사하는
방식이 아니라 Codmes 공용 card 규격을 사용하므로 macOS와 iPhone/iPad에서 같은
내용을 각 플랫폼에 맞는 UI로 보여줍니다.

## 파일 구성

```text
plugin.json          플러그인, Surface, 인증과 MCP 연결 manifest
surface.json         공지·LMS·포털·설정 화면 선언
tools.json           AI가 사용할 공지 검색·상세 조회 도구 선언
plugin.docker.json   Docker/Caddy 배포 주소를 사용하는 대체 manifest
```

## 개발 환경

기본 manifest는 KNU API와 MCP가 `http://127.0.0.1:8000`에서 실행된다고
가정합니다. 먼저 KNU AI Assistant 저장소에서 API 서버를 실행합니다.

```sh
cd /path/to/knu-ai-assistant/services/api
source ../../.venv/bin/activate
python -m uvicorn api.main:app --host 127.0.0.1 --port 8000
```

이 저장소의 루트를 로컬 플러그인으로 설치합니다.

```sh
cd /path/to/Codmes
node bin/codmes.mjs plugin install \
  /path/to/codmes-plugin-knu \
  --root /path/to/CodmesWorkspace
```

Docker/Caddy를 통해 KNU API를 로컬 `80` 포트에 노출하는 환경에서는 대체
manifest를 설치할 수 있습니다.

```sh
node bin/codmes.mjs plugin install \
  /path/to/codmes-plugin-knu/plugin.docker.json \
  --root /path/to/CodmesWorkspace
```

설치 후 KNU 설정에서 포털 학번과 비밀번호로 로그인하면 KNU 서버가 사용자별
세션 토큰을 발급합니다. Codmes Workspace 서버는 이 토큰을 Surface와 MCP에
공유하므로 일반 사용자가 `MCP_AUTH_TOKEN`을 알거나 직접 입력할 필요가 없습니다.
로그아웃하면 KNU 서버의 현재 세션을 폐기한 뒤 Codmes 서버에 저장된 두 credential도
함께 삭제합니다.

MCP 호출에서 학과를 생략하면 KNU 서버가 해당 session의 학적정보를 확인해 학교
공통 공지와 사용자의 학과 공지를 함께 조회합니다. 다른 학과 공지는 기본 결과에서
제외되며, 사용자가 다른 학과를 명시적으로 물었을 때만 AI가 `department` 인자로
범위를 바꿉니다.

`MCP_AUTH_TOKEN`은 KNU 서버 운영자가 loopback MCP 게이트웨이를 점검할 때만 쓰는
내부용 비밀값입니다. 일반 사용자 설치 정보나 plugin manifest에 포함하지 않습니다.

## Marketplace 배포

Marketplace에는 이 Git 저장소 자체가 아니라 `plugin.json`, `surface.json`,
`tools.json`을 묶어 Publisher 키로 서명한 `.codmes-plugin` 파일을 배포합니다.

초기 설정으로 저장소 Actions secret 두 개를 등록합니다.

- `CODMES_PLUGIN_SIGNING_KEY`: KNU Publisher Ed25519 개인키 PEM
- `MARKETPLACE_AUTOMATION_TOKEN`: 같은 조직의 `Codmes-Marketplace` 저장소에
  Contents와 Pull requests 쓰기 권한을 가진 fine-grained token 또는 GitHub App token

개인키와 token은 package나 Git 기록에 포함되지 않고 Actions 실행 중에만 임시
파일로 복원한 뒤 삭제됩니다. Release workflow는 fork의 Pull Request에서는 실행되지
않고 이 공식 저장소에서 관리자가 Release를 발행할 때만 secret을 사용합니다.

초기 설정 후 배포 순서는 다음 세 단계뿐입니다.

1. `plugin.json`의 버전을 올리고 변경사항을 tag로 만듭니다.
2. 해당 tag로 GitHub Release를 발행하고 릴리스 노트를 작성합니다.
3. 자동 생성된 Marketplace Pull Request를 검수하고 병합합니다.

Release가 발행되면 `publish-marketplace.yml`이 Publisher 키로 package를 서명하고
`registry/packages/knu-plugin/<version>.codmes-plugin`에 배치합니다. 이어서 SHA-256,
서명, 버전, 경로, 릴리스 노트와 Registry 시간을 자동 갱신하고 전체 검증을 통과한
경우에만 `release/knu-plugin-<version>` 브랜치와 Pull Request를 생성합니다. 동일한
package를 GitHub Release asset에도 첨부하므로 Marketplace와 Release가 완전히 같은
byte를 보관합니다. `index.json`을 사람이 직접 편집할 필요가 없습니다.

KNU 자동 발행 workflow는 서명과 Registry 생성을 위해 검증된 Codmes Publisher CLI
commit을 읽습니다. Marketplace PR의 별도 Actions가 결과물을 다시 검증하므로 자동화
token만으로 검수 절차를 우회하거나 `main`에 직접 배포할 수는 없습니다. 일반
Community plugin은 Marketplace 쓰기 token을 받지 않으며 자신의 fork에서 PR을
제출하는 기존 흐름을 사용합니다.

일반 사용자는 소스 저장소를 복제하지 않고 Codmes Marketplace에서 KNU를
설치하거나 업데이트합니다.
