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
- macOS, iOS와 iPadOS 지원

KNU 웹사이트를 WebView나 iframe으로 열지 않습니다. `surface.json`이 화면 구조와
데이터 바인딩을 선언하고, Codmes Apple 클라이언트가 SwiftUI로 렌더링합니다.

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

`MCP_AUTH_TOKEN`은 KNU 서버 운영자가 loopback MCP 게이트웨이를 점검할 때만 쓰는
내부용 비밀값입니다. 일반 사용자 설치 정보나 plugin manifest에 포함하지 않습니다.

## Marketplace 배포

Marketplace에는 이 Git 저장소 자체가 아니라 `plugin.json`, `surface.json`,
`tools.json`을 묶어 Publisher 키로 서명한 `.codmes-plugin` 파일을 배포합니다.

배포 순서는 다음과 같습니다.

1. KNU API와 플러그인의 데이터 계약을 로컬에서 확인합니다.
2. 플러그인 버전을 올리고 설치 테스트를 실행합니다.
3. Publisher 키로 `.codmes-plugin` package를 서명합니다.
4. package와 Registry 변경을 Codmes Marketplace에 Pull Request로 제출합니다.
5. Marketplace의 공식 Actions가 manifest, 권한, SHA-256, Publisher 서명을
   최종 검증합니다.

이 저장소의 CI는 Codmes 저장소를 checkout하지 않습니다. Community plugin은
개인이나 외부 조직이 소유할 수 있으며, 공식 검증 책임은 Marketplace PR에
있습니다.

일반 사용자는 소스 저장소를 복제하지 않고 Codmes Marketplace에서 KNU를
설치하거나 업데이트합니다.
