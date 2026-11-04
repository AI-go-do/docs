# MVP (v0.1) 서비스별 구조 · API

> 2026-11-04 기준 코드: core `develop` d9724f7 (Flyway V1~V5), device `main` 3d4d6b5, infra `main`, web `develop` 2ef77db + 화면 PR 사슬(#4~#10).
> 요청·응답 본문과 오류 코드 전체는 [API 명세](../api/README.md)에 있어요. 여기서는 **서비스마다 무엇을 열고(제공) 무엇을 부르는지(호출)** 만 봐요.
> 표의 상태: ✅ develop/main에 있음 · 🔶 PR·브랜치에만 있음 · ❌ 코드 없음.

![MVP 서비스 구조](mvp-services.png)

## 0. 한눈에

```mermaid
flowchart LR
    phone["휴대폰 (PWA)"] -- HTTPS --> school["학교 nginx<br/>(compose 밖)"]
    school -- ":8411 개발 / :8412 시연" --> gw["우리 입구 nginx<br/>gateway"]
    gw -- "/" --> web["web<br/>정적 파일"]
    gw -- "/api/*" --> core["core<br/>Spring Boot 4"]
    core -- JDBC --> db[("PostgreSQL 16")]
    core -. "SMTP (MAIL_ENABLED)" .-> mail["Gmail"]
    kp["키패드 + 우노"] -- "USB 시리얼 JSON" --> relay["중계 gateway.py<br/>(노트북)"]
    relay -- "POST /api/access/verify<br/>장치 토큰" --> school
```

| 서비스 | 저장소 | 실행 위치 | 여는 곳 |
|---|---|---|---|
| 학교 nginx | (학교 서버, 강사님 관리) | 학교 서버, Docker 밖 | 443 → 8411·8412 |
| 우리 입구 nginx `gateway` | `infra` | compose 컨테이너 | `${PORT}:80` (밖으로 열린 유일한 포트) |
| web | `mspace-web` | compose 컨테이너 (nginx가 정적 파일) | 안쪽 `web:80` |
| core | `mspace-core` | compose 컨테이너 | 안쪽 `core:8080` |
| PostgreSQL | `infra` | compose 컨테이너 | 안쪽 `db:5432`, 밖으로 안 염 |
| 키패드 + 중계 | `mspace-device` | 우노 + 노트북 | 열지 않음 (부르기만) |

**MVP에서 빠지는 것**: agent · MCP 서버 · 모델(Gemini/Ollama) · 재실 센서 · Jetson. compose에 `mcp`·`agent` 컨테이너가 떠 있어도 입구 nginx에 `/api/agent/` 길이 없어서 v0.1에는 닿지 않아요.

## 1. 학교 nginx

```mermaid
flowchart LR
    a["mspace.java21.net"] --> p2[":8412 시연 (main 이미지)"]
    b["dev-mspace.java21.net"] --> p1[":8411 개발 (develop 이미지)"]
```

| 받는 주소 | 보내는 곳 | 하는 일 |
|---|---|---|
| `https://mspace.java21.net` | 학교 서버 `:8412` | 시연용. HTTPS를 여기서 풀어요 |
| `https://dev-mspace.java21.net` | 학교 서버 `:8411` | 개발용 |

- **제공 API: 없음.** 길만 넘겨요. 설정은 강사님이 학교 nginx 관리자에서 해요.

## 2. 우리 입구 nginx (`gateway`)

```mermaid
flowchart LR
    in[":80"] --> r1{"주소"}
    r1 -- "/api/" --> core["core:8080"]
    r1 -- "= /actuator/health" --> core
    r1 -- "/ (나머지)" --> web["web:80"]
```

| 경로 | 보내는 곳 | 비고 | 상태 |
|---|---|---|---|
| `/api/` | `core:8080` | Host·X-Real-IP·X-Forwarded-* 헤더 붙임 | ✅ |
| `= /actuator/health` | `core:8080` | 배포 직후 살아 있는지 확인 | ✅ |
| `/` | `web:80` | 화면. 새로고침 시 화면 주소(`/me` 등)는 web 안 nginx가 처리 | ✅ |
| `/api/agent/` | agent | v0.3 (비서). MVP 아님 | ❌ |

- **제공 API: 없음.** 토큰도 검사하지 않아요. 인증은 core가 해요.
- Docker 내부 DNS(`127.0.0.11`)에 10초마다 다시 물어서, 배포로 컨테이너가 바뀌어도 502가 안 나요.

## 3. web (PWA)

```mermaid
flowchart LR
    subgraph web["web (Vite + React + TS)"]
      s1["로그인"] --- s2["예약"] --- s3["입실 번호 · 홈"] --- s4["내 예약"]
    end
    web -- "fetch('/api/...')<br/>Authorization: Bearer" --> core["core"]
```

- **제공 API: 없음.** 정적 파일(HTML·JS)만 내요. 같은 주소라 CORS가 필요 없어요.
- 액세스 토큰은 메모리에만 들고, `Authorization: Bearer`로 보내요.

| 화면 | 부르는 core API | 화면 상태 |
|---|---|---|
| 홈 (빈방) | `GET /api/rooms/availability` | ✅ develop |
| 로그인 | `POST /api/auth/email-codes` · `/login` · `/refresh` · `/logout` | 🔶 web #4 |
| 예약 | `GET /api/rooms` · `GET /api/rooms/availability` · `GET /api/members` · `POST /api/reservations` | 🔶 web #5 |
| 입실 번호 · 홈 내 예약 | `GET /api/reservations/me` | 🔶 web #6 |
| 내 예약 · 취소 · 퇴실 | `GET /api/reservations/me` · `POST .../{id}/cancel` · `POST .../{id}/checkout` | 🔶 web #7 |
| 이용 규칙 | (core API 없음 → 화면 안 문구 + 브라우저 저장) | 🔶 web #8 |
| 지금 바로 쓰기 (v0.2) | `POST /api/reservations/instant` | 🔶 web #9 |
| 배치도 | `GET /api/rooms` | 🔶 web #10 |

## 4. core (예약 서버)

```mermaid
flowchart TB
    subgraph core["core · 모듈러 모놀리스 (포트·어댑터)"]
      direction LR
      m["member<br/>로그인 · 회원"]
      r["room<br/>방 · 빈방"]
      v["reservation<br/>예약 · 취소 · 퇴실<br/>+ 1분 정리 스케줄러"]
      a["access<br/>6자리 번호 확인<br/>5회 잠금"]
    end
    core -- "JDBC (Flyway V1~V5)" --> db[("PostgreSQL")]
    m -. "SMTP · MAIL_ENABLED=true일 때" .-> mail["Gmail"]
```

| 모듈 | 하는 일 | 쓰는 테이블 |
|---|---|---|
| `member` | 학교 메일 인증번호, JWT 발급, refresh 회전, 학번 검색 | `users` · `email_verification` · `refresh_token` |
| `room` | 방 목록, 30분 칸별 빈방 | `rooms` · `reservation` |
| `reservation` | 예약 규칙(30분 칸 · 2시간 · 7일 · 대표자 1건 · 겹침 금지), 내 예약, 취소, 퇴실, 입실 번호 발급 | `reservation` · `reservation_participant` |
| `access` | 키패드 번호 확인, 같은 `eventId` 한 번만 처리, 5번 틀리면 1분 잠금 | `access_event` · `keypad_lock` |

### 4.1 core가 여는 API

인증 칸: **없음** = 토큰 필요 없음, **학생** = `Authorization: Bearer <액세스 토큰>`, **장치** = `Authorization: Bearer <MSPACE_DEVICE_TOKEN>`.

| 메서드 · 주소 | 인증 | 보내는 것 | 받는 것 | 버전 | 상태 |
|---|---|---|---|---|---|
| `POST /api/auth/email-codes` | 없음 | `email` | `204` | v0.1 | ✅ |
| `POST /api/auth/login` | 없음 | `email`, `code`, (처음만) `name`, `studentId` | `accessToken`, `expiresAt`, `memberId`, `newMember`, `refreshToken` | v0.1 | ✅ |
| `POST /api/auth/refresh` | 없음 | `refreshToken` | 로그인과 같은 모양 (새 refresh) | v0.1 | ✅ |
| `POST /api/auth/logout` | 없음 | `refreshToken` | `204` | v0.1 | ✅ |
| `GET /api/rooms` | 없음 | — | `[{id, name, minPeople, maxPeople}]` | v0.1 | ✅ |
| `GET /api/rooms/availability?date=` | 없음 | `date` (오늘~7일) | 방별 30분 칸 `available` | v0.1 | ✅ |
| `GET /api/members?studentId=` | 학생 | `studentId` | `{studentId, name(가림)}` | v0.1 | ✅ |
| `POST /api/reservations` | 학생 | `roomId`, `startAt`, `endAt`, `participantStudentIds`, (선택) `Idempotency-Key` 헤더 | `201` `{id, status, headcount, accessCode}` | v0.1 | ✅ |
| `GET /api/reservations/me` | 학생 | — | 내 예약 목록 (`accessCode`는 대표자 · 진행 중만) | v0.1 | ✅ |
| `POST /api/reservations/{id}/cancel` | 학생 | — | `{id, status: CANCELED}` | v0.1 | ✅ |
| `POST /api/reservations/{id}/checkout` | 학생 | — | `{id, status: CHECKED_OUT}` | v0.1 | ✅ |
| `POST /api/access/verify` | 장치 | `eventId`, `roomId`, `deviceId`, `code` | 늘 `200` `{result: OPENED}` 또는 `{result: DENIED, reason}` | v0.1 | ✅ |
| `GET /actuator/health` | 없음 | — | `{status: UP}` | 인프라 | ✅ |
| `POST /api/reservations/instant` | 학생 | `roomId`, `endAt`, `participantStudentIds` | `POST /api/reservations`와 같음 | v0.2 | ✅ (먼저 들어옴) |
| `GET /api/auth/check` | 학생 | — | `204` / `401` | v0.3 비서용 | ✅ |
| `GET /api/rules` | — | — | — | v0.1 요구사항 | ❌ 없음 |

- 오류는 모두 `{"code", "message"}` 모양이에요. 코드 목록은 [API 명세](../api/README.md) 2~6장.
- 시각은 서울 시간 `+09:00`. 오프셋 없이 보내면 `400`.
- 노쇼 · 자동 퇴실은 요구사항상 v0.2인데, 코드(1분마다 도는 정리 스케줄러, `MSPACE_SWEEP_ENABLED` 기본 `true`)는 이미 develop에 있어요.

### 4.2 core가 부르는 것

| 대상 | 방법 | 설정 (값은 `.env`에만) |
|---|---|---|
| PostgreSQL | JDBC, 켤 때 Flyway V1~V5 | `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` |
| Gmail SMTP | 인증번호 메일. `false`면 번호를 로그에만 찍음 | `MAIL_ENABLED`, `MAIL_USERNAME`, `MAIL_PASSWORD` |
| (토큰 서명) | JWT HS256 서명. 비면 켤 때마다 임시값 | `JWT_SECRET` |
| (장치 토큰) | 비면 `/api/access/verify`를 전부 `401` | `MSPACE_DEVICE_TOKEN` |

## 5. PostgreSQL

```mermaid
flowchart LR
    core["core"] -- "JDBC" --> db[("mspace DB")]
    fw["Flyway V1~V5"] -. "켤 때 자동" .-> db
```

| Flyway | 만든 테이블 | v0.1에서 씀 |
|---|---|---|
| V1 | `users`, `rooms`, `reservation`, `reservation_participant` | ✅ |
| V1 | `room_adjacency`, `reservation_photo`, `report`, `notification_log` | 아직 안 씀 (v0.2~v0.3용 자리) |
| V2 | `reservation`에 멱등 키 추가 | ✅ |
| V3 | `access_event`, `keypad_lock` | ✅ |
| V4 | `email_verification` | ✅ |
| V5 | `refresh_token` | ✅ |

- **제공 API: 없음.** core만 닿아요. 겹침 금지는 DB의 `EXCLUDE` 제약이 마지막으로 막아요.
- 데이터는 `db-data` 볼륨에 남아요. 컨테이너를 다시 만들어도 지워지지 않아요.

## 6. 키패드 + 중계 (`mspace-device`)

```mermaid
sequenceDiagram
    participant K as 키패드 · 우노
    participant G as 중계 gateway.py (노트북)
    participant C as core
    K->>G: 시리얼 한 줄 JSON {eventId, roomId, sensorType:"keypad", code}
    G->>C: POST /api/access/verify (Bearer 장치 토큰, 3초 · 재시도)
    C-->>G: 200 {result: OPENED | DENIED, reason}
    G-->>K: "OK" 또는 "NO" 한 줄
    K->>K: 초록 · 빨강 LED 1.5초
```

| 구간 | 형식 | 상태 |
|---|---|---|
| 우노 → 중계 | USB 시리얼 115200, JSON 한 줄. `eventId`는 `방-부팅번호-순번` | ✅ |
| 중계 → core | `POST {SERVER}/api/access/verify`. `eventId`는 uuid5로 바꿔 보냄(재전송해도 같은 값), `deviceId`=`keypad-{방}` | ✅ |
| 중계 → 우노 | `OK` (열림) / `NO` (거절 · 서버 응답 없음) | ✅ |

- **제공 API: 없음.** 번호는 서버만 판단해요. 장치는 번호를 저장하지 않아요.
- 중계 설정: `SERVER`, `DEVICE_TOKEN`, `PORT`(시리얼, 비면 자동 찾기), `DRY_RUN=1`(서버 없이 `123456`만 통과).
- 배포는 노트북에서 `git pull` 후 다시 실행이에요. 이미지(GHCR)를 만들지 않아요.

## 7. MVP 마감 전에 맞춰야 할 곳

| 무엇 | 지금 | 할 일 | 저장소 |
|---|---|---|---|
| compose → core 환경값 | infra `main`의 compose가 `MSPACE_DEVICE_TOKEN`·`JWT_SECRET`·`MAIL_*`를 core에 안 넘김. 이대로 서버에 올리면 키패드가 전부 `401`, 메일이 안 감 | infra 브랜치 `chore/device-token`·`chore/app-env` 머지 | infra |
| 중계 `SERVER` 기본값 | `http://localhost:8080` (core를 맥에서 따로 켤 때만 맞음). compose는 core 포트를 밖으로 안 열어요 | 서버 쓸 때 `SERVER=https://dev-mspace.java21.net`, 맥 compose면 `http://localhost:8411` | device |
| 화면 | 로그인~배치도가 web PR #4~#10에 있음 | 순서대로 머지 | web |
| refresh 토큰 쿠키 | core `feat/refresh-cookie` · web #4가 짝. 둘 다 미머지 | core 먼저, 그다음 web #4 | core · web |
| `GET /api/rules` | 요구사항에만 있음. 화면은 자체 문구로 대신 | MVP는 화면 문구로 가고, core API는 다음 버전 | core |
| 학교 서버 배포 | 강사님 안내 대기 | 그 전까지 맥 compose로 시험 | infra |
