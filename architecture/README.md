# 시스템 아키텍처

> 2026-11-02 기준. 회의에서 정한 구조를 각 저장소 develop 코드와 맞춰 그렸어요. **점선은 아직 코드가 없는 부분**이에요.

![M-SPACE 시스템 아키텍처](mspace-architecture.png)

## 한 줄 요약

- 밖에서 들어오는 문은 **우리 nginx 하나**예요. `/api/*`는 core로, `/`는 화면(web)으로 보내요.
- DB에는 **core만** 닿아요. 비서도 DB를 직접 보지 않고 MCP 서버를 거쳐 core API만 불러요.
- 키패드는 번호를 판단하지 않아요. 입력을 서버로 보내고 결과(LED)만 보여 줘요.

## 구성 요소

| 상자 | 저장소 | 기술 | 하는 일 |
|---|---|---|---|
| 휴대폰 (PWA) · 화면 web | `mspace-web` | PWA | 로그인, 빈방, 예약, 내 예약, 6자리 번호, 비서 채팅 |
| 학교 nginx | (학교 서버) | nginx | 도메인(`mspace.java21.net`, `dev-mspace.java21.net`)을 우리 포트(8412, 8411)로 넘김 |
| 우리 입구 nginx | `infra` | nginx | 주소별로 core·web·agent에 나눠 보냄 |
| mspace-core | `mspace-core` | Java 21 · Spring Boot 4 · 모듈러 모놀리스 | 예약 규칙, 로그인, 출입 번호 확인. 포트와 어댑터 구조 |
| PostgreSQL | `infra` (compose) | PostgreSQL 16 · Flyway | 겹침 금지(EXCLUDE) 제약, 테이블 버전 관리. 밖으로 포트 안 열림 |
| 비서 agent | `mspace-assistant` | Java · Spring Boot | 사용자 말 → 모델 → 도구. 쓰기 작업은 확인 버튼 뒤에만 |
| MCP 서버 | `mspace-assistant` | Spring AI MCP | core API를 MCP 도구로 감쌈 |
| 키패드 + 우노 | `mspace-device` | Arduino | 6자리 + `#` 입력, 초록·빨강 LED |
| 중계 gateway.py | `mspace-device` | Python | 입력을 `POST /api/access/verify`로 전달. 나중에 Jetson 보드 B로 옮김 |
| 재실 센서 (v0.2) | `mspace-device` | PIR · 조도 · 레이더 · CO2 | 빈 방 판단 보조 |
| Gemini / Ollama | (외부) | `LlmClient` 포트 + 어댑터 2개 | Gemini는 시연용, Ollama는 로컬(보드 B 후보). 학번은 모델로 보내지 않음 |
| GitHub Actions → GHCR | 각 저장소 `.github` | Actions · GHCR | 빌드·테스트 → 이미지 → SSH로 학교 서버에 배포 ([인프라 · CI/CD](../infra/infra-cicd.md)) |

## 누가 누구를 어떻게 부르나

| 보내는 쪽 → 받는 쪽 | 방법 |
|---|---|
| web(브라우저) → core | `fetch('/api/...')`. 같은 주소라 CORS 필요 없음. 타입은 OpenAPI에서 자동 생성 |
| 중계 → core | HTTP `POST /api/access/verify` |
| agent → MCP 서버 | MCP (Streamable HTTP, `/mcp`) |
| MCP 서버 → core | Spring `RestClient` (REST). OpenFeign은 쓰지 않음 |
| core → DB | JDBC · SQL |
| agent → 모델 | `LlmClient` 포트 (Gemini API / Ollama) |

## 그림과 코드가 아직 다른 곳 (2026-11-02)

| 어디 | 지금 | 맞추는 법 |
|---|---|---|
| 로그인 | core에 Security·JWT 없음. MCP는 임시로 `X-User-Id` 헤더를 넘김 | 메일 발송은 가짜 어댑터로 두고 JWT까지 먼저 |
| 출입 확인 | 중계는 `/api/access/verify`를 부르는데 core에 그 API가 없음 | 요청·응답 모양을 OpenAPI에 먼저 정하고 맞추기 |
| 화면 | `mspace-web` 비어 있음, compose에 web 없음 | web 이미지가 생기면 compose·nginx에 추가 |
| 비서 경로 | nginx가 `/api/`를 전부 core로 보내서 비서에 닿을 수 없음 | `/api/agent/` → agent 경로 추가 |
| 비서 채널 | 1차 회의는 텔레그램, 비서 설계는 웹 채팅 + 확인 카드 | 시연할 채널 하나 정하기 |

## 고치는 규칙

- 새 상자나 화살표가 생기면 이 문서와 그림을 같이 고쳐요. 버전(v0.2, v0.3)마다 상자를 덧붙여요.
- 새 기능 PR에는 "그림의 어느 상자에 붙나"를 한 줄 적어요.
