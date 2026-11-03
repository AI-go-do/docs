# API 명세

> 기준: 2026-11-03 코드. core `develop`(8740166), 로그인은 core PR #37 브랜치 `feat/34-email-login`, 비서는 assistant `feat/39-access-code-card` 브랜치.
> 근거 요구사항: [요구사항 명세 7.2](../requirements/requirements.md#72-기능-요구사항-목록).
> 컨벤션에 적힌 springdoc 자동 문서(`/swagger-ui`)가 붙기 전까지는 이 문서가 기준이에요. 코드를 바꾸면 이 문서도 같은 PR에서 고쳐 주세요.

## 0. 한눈에

| 누가 부르나 | 메서드 · 주소 | 하는 일 | 요구사항 | 상태 |
|---|---|---|---|---|
| 화면 | `POST /api/auth/email-codes` | 학교 메일로 인증번호 보내기 | FR-AUTH-001 | PR 대기 (core #37) |
| 화면 | `POST /api/auth/login` | 인증번호 확인 → 토큰 받기 | FR-AUTH-001 | PR 대기 (core #37) |
| 화면 | `POST /api/auth/refresh` | 자동 로그인 (토큰 새로 받기) | FR-AUTH-002 | PR 대기 (core #37) |
| 화면 | `POST /api/auth/logout` | 로그아웃 | FR-AUTH-002 | PR 대기 (core #37) |
| nginx | `GET /api/auth/check` | 비서로 넘기기 전 토큰 확인 | FR-AUTH-002 | PR 대기 (core #37) |
| 화면 · 비서 | `GET /api/rooms` | 방 목록 | FR-ROOM-001 | ✅ develop |
| 화면 · 비서 | `GET /api/rooms/availability?date=` | 하루 30분 칸별 빈방 | FR-WEB-004 | ✅ develop |
| 화면 · 비서 | `GET /api/members?studentId=` | 참여자 학번 검색 | FR-RSV-004 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations` | 예약 만들기 | FR-RSV-001~005 | ✅ develop |
| 화면 · 비서 | `GET /api/reservations/me` | 내 예약 | FR-RSV-006 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/cancel` | 시작 전 취소 | FR-RSV-007 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/checkout` | 퇴실 | FR-RSV-008 | ✅ develop |
| 키패드 | `POST /api/access/verify` | 6자리 번호 확인 → 문 열기 | FR-ACC-001 | ✅ develop |
| 화면 | `GET /api/rules` | 이용 규칙 | FR-WEB-003 | ❌ 아직 없음 |
| 화면 | `POST /api/agent/chat` | 비서와 대화 | FR-AGT | 브랜치 (assistant) |
| 화면 | `POST /api/agent/confirmations/{token}` | 확인 카드 [확인] | FR-AGT | 브랜치 (assistant) |
| 화면 | `GET /api/agent/reservations/{id}/access-code` | 비서 입실 번호 카드 | FR-AGT | 브랜치 (assistant) |

`/api/agent/`로 시작하면 nginx가 비서 서버(agent)로, 나머지는 core로 보내요.

## 1. 공통 규칙

| 항목 | 규칙 |
|---|---|
| 로그인 | `Authorization: Bearer <accessToken>`. 없거나 틀리면 `401 UNAUTHORIZED` |
| 개발용 임시 헤더 | `X-User-Id: <회원 id>`. core를 `MSPACE_DEV_HEADER=true`로 띄웠을 때만 받아요. 운영에서는 꺼요. 토큰이 있으면 토큰만 봐요 |
| 장치 | 키패드는 사람 토큰이 아니라 장치 토큰(`Authorization: Bearer <장치 토큰>`)을 써요 |
| 시각 | 서울 시간 ISO-8601 `+09:00` (예: `2026-11-03T14:00:00+09:00`). 토큰 만료 시각(`expiresAt`)만 UTC `Z` |
| 예약 칸 | 정각·30분 단위, 1회 최대 2시간, 오늘부터 7일 안 |
| 오류 본문 | 모두 `{"code": "RESERVATION_TIME_TAKEN", "message": "이미 예약된 시간입니다"}`. 화면은 `message`를 그대로 보여 줘도 돼요 |
| 형식 오류 | 빠진 값, 숫자가 아닌 id, 깨진 JSON은 `400 INVALID_REQUEST` |
| 개인정보 | 다른 사람 이름은 가려서(`김*영`) 줘요. 입실 번호는 대표자에게만 |

## 2. 로그인 (`/api/auth`) — 토큰 필요 없음

### `POST /api/auth/email-codes` 인증번호 보내기

```json
{ "email": "a@chosun.ac.kr" }
```
→ `204` (본문 없음). 번호는 6자리, 10분 유효.

| 오류 | 상태 | 언제 |
|---|---|---|
| `SCHOOL_EMAIL_ONLY` | 400 | `@chosun.ac.kr`이 아님 |
| `EMAIL_CODE_TOO_SOON` | 429 | 1분 안에 다시 요청 |

### `POST /api/auth/login` 번호 확인 → 토큰

```json
{ "email": "a@chosun.ac.kr", "code": "123456", "name": "김지영", "studentId": "20230001" }
```
`name`·`studentId`는 처음 가입할 때만 넣어요. → `200`

```json
{
  "accessToken": "eyJ...", "tokenType": "Bearer", "expiresAt": "2026-11-03T08:00:00Z",
  "memberId": 1, "newMember": true,
  "refreshToken": "...", "refreshExpiresAt": "2026-11-17T07:00:00Z"
}
```
액세스 토큰 1시간, refresh 토큰 14일.

| 오류 | 상태 | 언제 |
|---|---|---|
| `SCHOOL_EMAIL_ONLY` | 400 | 학교 메일 아님 |
| `EMAIL_CODE_NOT_FOUND` | 400 | 번호를 받은 적 없음 |
| `EMAIL_CODE_EXPIRED` | 400 | 10분 지남 |
| `EMAIL_CODE_MISMATCH` | 400 | 번호 틀림 |
| `EMAIL_CODE_TOO_MANY_ATTEMPTS` | 400 | 5번 틀림 → 다시 받기 |
| `STUDENT_ID_TAKEN` | 409 | 다른 메일로 이미 가입된 학번 |
| `EMAIL_ALREADY_SIGNED_UP` | 409 | 같은 메일로 동시에 가입됨 → 번호 다시 받기 |

### `POST /api/auth/refresh` 자동 로그인

```json
{ "refreshToken": "..." }
```
→ `200`, 로그인과 같은 모양(새 refresh 토큰 포함). 보낸 refresh 토큰은 더는 못 써요(회전). 이미 쓴 토큰이 다시 오면 그 사람의 모든 로그인을 끊어요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `REFRESH_TOKEN_INVALID` | 401 | 없음·만료·이미 씀 → 화면은 로그인 화면으로 |

### `POST /api/auth/logout` 로그아웃

```json
{ "refreshToken": "..." }
```
→ 항상 `204`. 화면도 들고 있던 두 토큰을 지워요.

### `GET /api/auth/check` 토큰 확인 (nginx 전용)

`Authorization: Bearer` → 맞으면 `204`, 아니면 `401`. nginx `auth_request`가 `/api/agent/` 앞에서 불러요.

## 3. 방

### `GET /api/rooms` 방 목록 — 토큰 필요 없음

```json
[ { "id": 1, "name": "1번 방", "minPeople": 3, "maxPeople": 5 } ]
```

### `GET /api/rooms/availability?date=2026-11-03` 하루 빈방 — 토큰 필요 없음

```json
{
  "date": "2026-11-03", "slotMinutes": 30,
  "rooms": [
    { "roomId": 1, "roomName": "1번 방", "minPeople": 3, "maxPeople": 5,
      "slots": [ { "startAt": "2026-11-03T14:00:00+09:00", "endAt": "2026-11-03T14:30:00+09:00", "available": true } ] }
  ]
}
```

| 오류 | 상태 | 언제 |
|---|---|---|
| `ROOM_DATE_OUT_OF_RANGE` | 400 | 오늘 전이거나 7일 밖 |

## 4. 회원

### `GET /api/members?studentId=20230002` 참여자 학번 검색 — 로그인 필요

```json
{ "studentId": "20230002", "name": "홍*동" }
```

| 오류 | 상태 | 언제 |
|---|---|---|
| `MEMBER_NOT_FOUND` | 404 | 없는 학번 또는 학생 인증 전 (둘을 구별해 알려 주지 않아요) |

## 5. 예약 — 모두 로그인 필요

### `POST /api/reservations` 예약 만들기

헤더(선택): `Idempotency-Key: <1~64자>`. 같은 키로 다시 보내면 새로 만들지 않고 처음 예약을 `200`으로 돌려줘요(비서 재시도용). 화면은 안 붙여도 돼요.

```json
{
  "roomId": 1,
  "startAt": "2026-11-03T14:00:00+09:00", "endAt": "2026-11-03T16:00:00+09:00",
  "participantStudentIds": ["20230002", "20230003"]
}
```
대표자(나)는 `participantStudentIds`에 넣지 않아요. 참여자가 없으면 `[]`.

→ `201` (`Location: /api/reservations/12`), 같은 키 재요청이면 `200`

```json
{ "id": 12, "roomId": 1, "startAt": "2026-11-03T14:00:00+09:00", "endAt": "2026-11-03T16:00:00+09:00",
  "status": "BOOKED", "headcount": 3, "accessCode": "482913" }
```

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 빠진 값, 빈 학번, 키가 비었거나 64자 넘음 |
| `ROOM_NOT_FOUND` | 404 | 없거나 쓸 수 없는 방 |
| `RESERVATION_NOT_ON_SLOT` | 400 | 정각·30분이 아님 |
| `RESERVATION_INVALID_LENGTH` | 400 | 끝이 시작보다 앞이거나 2시간 넘음 |
| `RESERVATION_IN_PAST` | 400 | 지난 시각 |
| `RESERVATION_OUT_OF_RANGE` | 400 | 7일 밖 |
| `PARTICIPANT_DUPLICATE` | 400 | 같은 학번 두 번 |
| `PARTICIPANT_NOT_FOUND` | 400 | 학생 인증 안 한 학번 |
| `HEADCOUNT_OUT_OF_RANGE` | 400 | 방 인원(대표자 포함) 범위 밖 |
| `RESERVATION_TIME_TAKEN` | 409 | 그 방 그 시간에 이미 예약 |
| `OWNER_HAS_ACTIVE_RESERVATION` | 409 | 내가 대표인 끝나지 않은 예약이 이미 있음 |
| `PARTICIPANT_TIME_TAKEN` | 409 | 참여자가 같은 시간 다른 방 예약 중 |
| `IDEMPOTENCY_KEY_REUSED` | 409 | 같은 키를 다른 내용에 씀 |
| `RESERVATION_BUSY` | 503 | 동시 요청이 몰림 (드묾). 잠시 후 다시 |

### `GET /api/reservations/me` 내 예약

대표자·참여자로 들어간 예약 모두. 끝나지 않은 것이 위(시작 빠른 순), 끝난 것은 최근 순.

```json
[
  { "id": 12, "room": { "id": 1, "name": "1번 방" },
    "startAt": "2026-11-03T14:00:00+09:00", "endAt": "2026-11-03T16:00:00+09:00",
    "status": "BOOKED", "owner": true, "accessCode": "482913",
    "participants": [ { "name": "김*영", "owner": true }, { "name": "홍*동", "owner": false } ] }
]
```
`accessCode`는 대표자일 때만 값이 있고, 참여자면 `null`.

예약 상태(`status`): `BOOKED` 예약됨 → `IN_USE` 입실 → `CHECKED_OUT` 퇴실 / `AUTO_CHECKED_OUT` 시간 끝나 자동 퇴실 / `NO_SHOW` 15분 안 옴 / `CANCELED` 취소.

### `POST /api/reservations/{id}/cancel` 시작 전 취소

본문 없음 → `200 {"id": 12, "status": "CANCELED"}`

| 오류 | 상태 | 언제 |
|---|---|---|
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내 예약이 아님 |
| `NOT_RESERVATION_OWNER` | 403 | 참여자가 취소하려 함 |
| `RESERVATION_NOT_CANCELABLE` | 409 | 이미 시작·끝남·취소됨 |

### `POST /api/reservations/{id}/checkout` 퇴실

본문 없음 → `200 {"id": 12, "status": "CHECKED_OUT"}`. 방이 바로 비어요. 조기 퇴실 확인 창은 화면이 띄워요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내 예약이 아님 |
| `NOT_RESERVATION_OWNER` | 403 | 참여자가 퇴실하려 함 |
| `RESERVATION_NOT_IN_USE` | 409 | 입실 전이거나 이미 끝남 |

## 6. 출입 (키패드)

### `POST /api/access/verify` — 장치 토큰 필요

```json
{ "eventId": "1-48213-18", "roomId": 1, "deviceId": "keypad-1", "code": "123456" }
```
`eventId`(1~64자)는 장치가 누를 때마다 새로 만드는 값이라, 같은 값이 다시 와도 한 번만 처리해요. `deviceId`는 50자 이하(선택).

→ 거절도 `200`이에요. 장치는 `result`만 보고 LED를 켜요.

```json
{ "result": "OPENED" }
{ "result": "DENIED", "reason": "WRONG_CODE" }
```
`reason`: `WRONG_CODE` 틀림, `NOT_YET` 아직 이름, `LOCKED` 5번 틀려 잠김.

| 오류 | 상태 | 언제 |
|---|---|---|
| `DEVICE_UNAUTHORIZED` | 401 | 장치 토큰이 없거나 틀림 |
| `INVALID_REQUEST` | 400 | `eventId`·`roomId` 빠짐, 길이 넘음 |

## 7. 비서 (`/api/agent`) — 로그인 필요, 아직 브랜치

비서 서버는 받은 `Authorization`을 그대로 core에 넘겨요. 그래서 비서도 학생 본인 권한만큼만 할 수 있어요.

### `POST /api/agent/chat` 대화

```json
{ "message": "내일 2시에 빈방 있어?", "history": [ { "role": "user", "text": "..." }, { "role": "assistant", "text": "..." } ] }
```
→ `200`

```json
{
  "reply": "내일 14:00에 1번 방, 2번 방이 비어 있어요. 예약할까요?",
  "confirmations": [ { "tool": "create_reservation", "token": "c_9f2...", "summary": "1번 방 · 11/4 14:00–16:00",
                       "notes": [], "needsParticipants": 2, "expiresAt": "2026-11-03T16:10:00+09:00" } ],
  "accessCodeCards": [ { "reservationId": 12 } ],
  "toolCalls": 2, "stoppedByLimit": false
}
```
쓰기(예약·취소·퇴실)는 바로 하지 않고 확인 카드(`confirmations`)로 돌려줘요. 입실 번호는 모델에 보내지 않아서 `accessCodeCards`에는 예약 id만 있어요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `UNAUTHORIZED` | 401 | 로그인 안 함 |
| `INVALID_REQUEST` | 400 | 빈 말, 깨진 대화 기록 |
| `LLM_RATE_LIMITED` | 429 | Gemini 무료 한도 |
| `LLM_UNAVAILABLE`, `LLM_NOT_CONFIGURED` | 503 | LLM 응답 없음, 키 설정 안 됨 |
| `TOOLS_UNAVAILABLE` | 503 | MCP 서버 연결 안 됨 |

### `POST /api/agent/confirmations/{token}` 확인 카드 [확인]

```json
{ "participantStudentIds": ["20230002", "20230003"] }
```
(예약 만들기 카드일 때만. 나머지는 본문 없음) → core 응답을 상태 코드까지 그대로 돌려줘요(예: 예약 만들기면 5장의 `201` 본문).

### `GET /api/agent/reservations/{id}/access-code` 입실 번호 카드

대표자에게만 번호를 줘요. `Cache-Control: no-store`. 오류는 core의 `GET /api/reservations/me`와 같은 기준.

## 8. 코드와 문서가 아직 다른 곳

| 무엇 | 지금 | 할 일 |
|---|---|---|
| `GET /api/rules` | 요구사항에만 있고 코드 없음 | core에 추가 |
| springdoc `/swagger-ui` | 컨벤션에만 있고 core에 의존성 없음 | core에 추가하면 이 문서는 요약본으로 줄이기 |
| 로그인 API | core PR #37 리뷰 중 | 머지되면 0장 상태 ✅로 |
| 비서 API | assistant 브랜치(#8 등)만 | develop 머지되면 0장 상태 ✅로 |
