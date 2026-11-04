# 시스템 아키텍처

> 2026-11-02 처음 그림, 11-04 갱신(학교 nginx를 compose 밖으로, web 연결). 회의에서 정한 구조를 각 저장소 develop 코드와 맞춰 그렸어요. **점선은 아직 코드가 없는 부분**이에요.

![M-SPACE 시스템 아키텍처](mspace-architecture.png)

## 한 줄 요약

- 밖에서 들어오면 학교 nginx를 지나 **우리 입구 nginx 하나**로 들어와요. `/api/*`는 core로, `/`는 화면(web)으로 보내요.
- DB에는 **core만** 닿아요. 비서도 DB를 직접 보지 않고 MCP 서버를 거쳐 core API만 불러요.
- 키패드는 번호를 판단하지 않아요. 입력을 서버로 보내고 결과(LED)만 보여 줘요.

## nginx가 두 개인 이유

요청은 `휴대폰 → 학교 nginx → 우리 입구 nginx → web 또는 core` 순서로 들어와요. 이름은 둘 다 nginx지만 하는 일이 달라요.

| | 학교 nginx | 우리 입구 nginx (`gateway`) |
|---|---|---|
| 어디에 | 학교 서버, **우리 Docker 밖** | 우리 compose 안 컨테이너 (`infra/nginx/default.conf`) |
| 누가 관리 | 강사님 (학교 서버 공용) | 우리 팀 (재웅) |
| 하는 일 | 도메인 → 우리 포트. `mspace.java21.net` → 8412(시연), `dev-mspace.java21.net` → 8411(개발). HTTPS 처리 | 주소 → 컨테이너. `/` → web, `/api/*` → core, `/actuator/health` → core |
| 비유 | 건물 정문 안내 데스크 ("M-space 팀은 8412호") | 우리 사무실 안내 데스크 ("화면은 왼쪽, API는 오른쪽") |

- 우리 입구 nginx는 **길만 나눠 주는 리버스 프록시**예요. API 게이트웨이처럼 토큰을 검사하지는 않아요. **로그인·토큰 확인은 core가 해요.**
- 둘 다 거치기 때문에 화면과 API가 같은 주소를 써요. 그래서 CORS 설정이 필요 없어요.
- 배포로 컨테이너가 바뀌어도 502가 나지 않게, 우리 nginx는 Docker 내부 DNS에 10초마다 주소를 다시 물어요.
- 비서 채팅을 열 때 `/api/agent/` → agent 경로를 우리 nginx에 추가해요(그림의 점선).

## MVP (v0.1) 범위

v0.1은 아래 다섯 가지를 **순서대로** 만들어요. 위 그림 중 비서·센서·Jetson 쪽은 v0.1 다음이에요.

| 순서 | 기능 | 핵심 규칙 | 그림의 상자 | 상태 (11-04) |
|---|---|---|---|---|
| 1 | 배포 파이프라인 | PR → CI → GHCR 이미지 → 서버 compose | GitHub Actions · GHCR · nginx 2개 | ✅ CI/CD·GHCR 완료. 학교 서버 배포는 강사님 안내 대기 |
| 2 | 학번 로그인 | 학교 메일 인증 → JWT + refresh | web · core | ✅ core 머지. 화면은 web PR 리뷰 중 |
| 3 | 방 목록 · 예약 | 2시간 단위, 7일 이내, 대표자 1건, 시간 겹침 금지(DB 제약) | web · core · PostgreSQL | 진행 중 |
| 4 | 내 예약 · 퇴실 | 시작 전 취소, 퇴실 버튼 = 바로 퇴실(대표자). 시간이 끝나면 자동 퇴실은 v0.2 | web · core | 진행 중 |
| 5 | 6자리 번호 · 키패드 입실 | 잠깐 나갔다 오면 같은 번호 다시 사용. 노쇼(15분)는 v0.2 | 키패드 · 중계 · core | core `/api/access/verify` 머지 |

**v0.1에 안 넣는 것**: 비서 에이전트(agent·MCP), 재실 센서, Jetson 보드 B. 다음 버전(v0.2~)에 상자를 덧붙여요.

## 버전별 계획

자세한 기능 목록은 [요구사항 3장 버전별 범위](../requirements/requirements.md#3-버전별-범위)와 8장에 있어요. 여기서는 **그림에 무엇이 붙는지**만 봐요.

| 버전 | 기간 | 한 줄 | 들어가는 것 | 그림에 새로 붙는 상자 · 길 |
|---|---|---|---|---|
| **v0.1** | 11/2 – 11/5 | 예약 한 바퀴 | 위 MVP 표 5가지 | 학교 nginx, 우리 입구 nginx, web, core, PostgreSQL, 키패드 · 중계 |
| **v0.2** | 11/6 – 11/10 | 사람 없이 돌아감 | 자동 퇴실, 노쇼, 지금 바로 쓰기, 연장하기(1회), 알림(시작 · 퇴실 안내, 앱 알림함), 방 현황 화면, 재실 · 환기 센서, 자리 비움 자동 퇴실 | 재실 센서(ESP32 · 아두이노) → core, core 안 스케줄러(자동 퇴실 · 노쇼) |
| **v0.3** | 11/11 – 11/13 | 운영 · AI | 신고, 사진 체크인 · 아웃, 허위 신고 판정 · 관리자 화면, AI 비서 스페이시(MCP), 내 시간표(선택) | agent · MCP 서버, 모델(Gemini · Ollama), 우리 nginx에 `/api/agent/` 길 |
| 그 뒤 | 필수 기능이 다 된 뒤 | 더 똑똑하게 | Jetson 보드 B(후보: 환기 예측, 로컬 모델 Ollama, 이용 규칙 질문 답변). 학과 일정 앱은 설계도만 | Jetson 보드 B, 학사 정보 서버(설계만) |

- **11/13 기능 동결**, 11/14 – 11/19 발표 준비, 11/20 발표.
- 버전 안에서 밀리면 v0.1부터 지켜요. v0.1 예약 한 바퀴는 자르지 않아요.
- **하지 않는 것**: 카메라 · 영상, 키박스, 학교 학사 DB 직접 연동, 실제 도어락 · QR(어댑터 설계만).
- 버전이 하나 끝나면 위 그림에 그 버전 상자를 실선으로 바꿔요.

## 구성 요소

| 상자 | 저장소 | 기술 | 하는 일 |
|---|---|---|---|
| 휴대폰 (PWA) · 화면 web | `mspace-web` | PWA | 로그인, 빈방, 예약, 내 예약, 6자리 번호, 비서 채팅 |
| 학교 nginx | (학교 서버, compose 밖) | nginx | 도메인(`mspace.java21.net`, `dev-mspace.java21.net`)을 우리 포트(8412, 8411)로 넘김. HTTPS |
| 우리 입구 nginx | `infra` | nginx | `/` → web, `/api/*` → core. 인증은 안 함(core가 함) |
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

## 그림과 코드가 아직 다른 곳 (2026-11-02, core 줄은 11/3 갱신)

| 어디 | 지금 | 맞추는 법 |
|---|---|---|
| 로그인 | ✅ 11/3 core에 메일 인증·JWT·refresh 들어감(core #37). `X-User-Id`는 `MSPACE_DEV_HEADER=true`일 때만 받는 개발용 | 비서(MCP)가 학생 토큰을 넘기는지는 영준 확인 필요 |
| 출입 확인 | ✅ 11/3 core에 `/api/access/verify` 들어감(core #38). 형식은 [API 명세 6장](../api/README.md#6-출입-키패드) | — |
| 화면 | ✅ 11/3 compose·nginx에 web 연결(infra #5). 화면 PR은 리뷰 중 | — |
| 비서 경로 | nginx가 `/api/`를 전부 core로 보내서 비서에 닿을 수 없음 | `/api/agent/` → agent 경로 추가 |
| 비서 채널 | 1차 회의는 텔레그램, 비서 설계는 웹 채팅 + 확인 카드 | 시연할 채널 하나 정하기 |

## 고치는 규칙

- 새 상자나 화살표가 생기면 이 문서와 그림을 같이 고쳐요. 버전(v0.2, v0.3)마다 상자를 덧붙여요.
- 새 기능 PR에는 "그림의 어느 상자에 붙나"를 한 줄 적어요.
