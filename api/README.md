# API 명세

> 기준: 2026-11-06 core `develop`(d9fc2a0, 친구 #68까지). 알림 읽음은 core PR #69. 비서는 assistant 브랜치 기준.
> 근거 요구사항: [요구사항 명세 7.2](../requirements/requirements.md#72-기능-요구사항-목록).
> 컨벤션에 적힌 springdoc 자동 문서(`/swagger-ui`)가 붙기 전까지는 이 문서가 기준이에요. 코드를 바꾸면 이 문서도 같은 PR에서 고쳐 주세요.

## 0. 한눈에

| 누가 부르나 | 메서드 · 주소 | 하는 일 | 요구사항 | 상태 |
|---|---|---|---|---|
| 화면 | `POST /api/auth/email-codes` | 학교 메일로 인증번호 보내기 | FR-AUTH-001 | ✅ develop |
| 화면 | `POST /api/auth/login` | 인증번호 확인 → 토큰 받기 | FR-AUTH-001 | ✅ develop |
| 화면 | `POST /api/auth/refresh` | 자동 로그인 (토큰 새로 받기) | FR-AUTH-002 | ✅ develop |
| 화면 | `POST /api/auth/logout` | 로그아웃 | FR-AUTH-002 | ✅ develop |
| nginx | `GET /api/auth/check` | 비서로 넘기기 전 토큰 확인 | FR-AUTH-002 | ✅ develop |
| 화면 · 비서 | `GET /api/rooms` | 방 목록 | FR-ROOM-001 | ✅ develop |
| 화면 · 비서 | `GET /api/rooms/availability?date=` | 하루 30분 칸별 빈방 | FR-WEB-004 | ✅ develop |
| 화면 · 비서 | `GET /api/rooms/status` | 방 현황 (지금 비었는지, 다음 예약, 센서 재실) | FR-DEV-102 | ✅ develop |
| 화면 · 비서 | `GET /api/members?studentId=` | 학번 검색 (친구 요청 전에 이름 확인) | FR-RSV-004 | ✅ develop |
| 화면 | `GET /api/friends` | 친구 창: 친구·받은 요청·보낸 요청 | FR-RSV-004 | ✅ develop |
| 화면 | `POST /api/friends/requests` | 학번으로 친구 요청 | FR-RSV-004 | ✅ develop |
| 화면 | `POST /api/friends/requests/{id}/accept`·`decline`·`cancel` | 수락·거절(받은 사람), 취소(보낸 사람) | FR-RSV-004 | ✅ develop |
| 화면 | `DELETE /api/friends/{studentId}` | 친구 삭제 | FR-RSV-004 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations` | 예약 만들기 | FR-RSV-001~005 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/instant` | 지금 바로 쓰기 (누른 시각부터) | FR-RSV-103 | ✅ develop |
| 화면 · 비서 | `GET /api/reservations/me` | 내 예약 | FR-RSV-006 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/cancel` | 시작 전 취소 | FR-RSV-007 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/checkout` | 퇴실 | FR-RSV-008 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/continue` | 연장하기 (사용 중 이어 쓰기, 1회) | FR-RSV-105 | ✅ develop |
| 화면 | `POST /api/reservations/{id}/still-here` | 아직 있어요 (자리 비움 타이머 다시, 예약당 1번) | FR-DEV-103 | ✅ develop |
| 관리자 | `GET /api/reservations/stats` | WEB/AGENT 예약 건수 | NFR-COM-008 | ✅ develop |
| 화면 | `POST /api/reports` | 신고 (그 방·옆 방) | FR-RPT-101 | ✅ develop |
| 화면 | `GET /api/notifications/me` | 내 알림함 (최근 50개, 읽음 여부) | FR-NTF-103 | ✅ develop |
| 화면 | `GET /api/notifications/me/unread-count` | 안 읽은 알림 개수 (종 아이콘 점) | FR-NTF-103 | PR 대기 (core #69) |
| 화면 | `POST /api/notifications/{id}/read` | 알림 하나 읽음 | FR-NTF-103 | PR 대기 (core #69) |
| 화면 | `POST /api/notifications/read-all` | 내 알림 모두 읽음 | FR-NTF-103 | PR 대기 (core #69) |
| 키패드 | `POST /api/access/verify` | 6자리 번호 확인 → 문 열기 | FR-ACC-001 | ✅ develop |
| 재실·CO2 센서 | `POST /api/sensor-events` | 방에 사람이 있는지, CO2 값 알리기 | FR-DEV-101~104 | ✅ develop |
| 화면 | `GET /api/rules` | 이용 규칙 | FR-WEB-003 | ✅ develop |
| 화면 | `POST /api/agent/chat` | 비서에게 말 보내기 (접수, `202`) | FR-AGT | 브랜치 (assistant) |
| 화면 | `GET /api/agent/chat/{jobId}` | 비서 답 조회 (폴링) | FR-AGT | 브랜치 (assistant) |
| 화면 | `POST /api/agent/confirmations/{token}` | 확인 카드 [확인] | FR-AGT | 브랜치 (assistant) |
| 화면 | `GET /api/agent/reservations/{id}/access-code` | 비서 입실 번호 카드 | FR-AGT | 브랜치 (assistant) |

`/api/agent/`로 시작하면 nginx가 비서 서버(agent)로, 나머지는 core로 보내요.

## 1. 공통 규칙

| 항목 | 규칙 |
|---|---|
| 로그인 | `Authorization: Bearer <accessToken>`. 없거나 틀리면 `401 UNAUTHORIZED` |
| 개발용 임시 헤더 | `X-User-Id: <회원 id>`. core를 `MSPACE_DEV_HEADER=true`로 띄웠을 때만 받아요. 운영에서는 꺼요. 토큰이 있으면 토큰만 봐요 |
| 만든 곳 | 비서는 예약 만들기(`POST /api/reservations`, `/instant`)에 `X-Created-Via: AGENT`를 붙여요. 그러면 `created_via = AGENT`, 없으면 `WEB`. 대소문자와 앞뒤 공백은 무시해요(` agent `도 AGENT). `WEB`·`AGENT` 말고 다른 값이면 `400 INVALID_REQUEST`. 건수를 따로 세는 표시일 뿐 권한과는 상관없어요 (core #45) |
| 장치 | 키패드·센서는 사람 토큰이 아니라 장치 토큰(`Authorization: Bearer <장치 토큰>`)을 써요 |
| 시각 | 서울 시간 ISO-8601 `+09:00` (예: `2026-11-03T14:00:00+09:00`). 토큰 만료 시각(`expiresAt`)만 UTC `Z`. 보낼 때 오프셋이 없으면(`2026-11-03T14:00`) `400 INVALID_REQUEST`, `Z`로 보내면 받고 응답은 `+09:00`으로 줘요 |
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
| `MAIL_SEND_FAILED` | 503 | 메일을 못 보냄 → 잠시 후 다시 |

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
| `SIGNUP_INFO_REQUIRED` | 400 | 처음 로그인인데 `name`·`studentId`가 없음 |
| `INVALID_SIGNUP_INFO` | 400 | 이름 50자 넘음, 학번이 숫자 20자 이내가 아님 |
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

### 친구 — 모두 로그인 필요 (FR-RSV-004, 11/6)

참여자는 **서로 수락한 친구**만 넣을 수 있어요. 이름은 친구끼리도 가린 이름(김*영)이에요.

`GET /api/friends` 친구 창
```json
{
  "friends":  [{ "studentId": "20230002", "name": "홍*동", "since": "2026-11-06T10:20:00+09:00" }],
  "received": [{ "requestId": 7, "studentId": "20230003", "name": "박*연", "requestedAt": "2026-11-06T10:21:00+09:00" }],
  "sent":     []
}
```

`POST /api/friends/requests` `{ "studentId": "20230002" }`
→ `201` `{ "requestId": 7, "studentId": "20230002", "name": "홍*동", "status": "PENDING" }`
상대가 이미 나에게 요청을 보내 뒀으면 바로 친구가 되어 `200`, `"status": "ACCEPTED"`.

`POST /api/friends/requests/{requestId}/accept` (받은 사람) → `200` `{ "studentId", "name", "since" }`
`POST /api/friends/requests/{requestId}/decline` (받은 사람) → `204`. 상대는 다시 보낼 수 있어요
`POST /api/friends/requests/{requestId}/cancel` (보낸 사람) → `204`
`DELETE /api/friends/{studentId}` → `204`. 양쪽에서 빠지고, 이미 잡은 예약의 참여자는 그대로예요

| 오류 | 상태 | 언제 |
|---|---|---|
| `FRIEND_SELF` | 400 | 나에게 요청 |
| `MEMBER_NOT_FOUND` | 404 | 없는 학번 또는 학생 인증 전 |
| `FRIEND_ALREADY` | 409 | 이미 친구 |
| `FRIEND_REQUEST_ALREADY_SENT` | 409 | 이미 요청을 보냄 (수락 대기 중) |
| `FRIEND_REQUEST_NOT_FOUND` | 404 | 내가 받은(취소는 보낸) 대기 중 요청이 아님. 두 번 누른 경우도 |
| `FRIEND_NOT_FOUND` | 404 | 친구가 아닌 학번을 삭제 |

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
| `USER_SUSPENDED` | 403 | 내가 노쇼 2번으로 이용 정지 중 (메시지에 끝나는 날짜, FR-RSV-106) |
| `PARTICIPANT_SUSPENDED` | 403 | 참여자 중 이용 정지 중인 사람이 있음 (누구인지는 안 알려 줌) |
| `PARTICIPANT_NOT_FRIEND` | 403 | 참여자 중 대표자와 친구가 아닌 사람이 있음 (누구인지는 안 알려 줌, 11/6) |
| `RESERVATION_TIME_TAKEN` | 409 | 그 방 그 시간에 이미 예약 |
| `OWNER_HAS_ACTIVE_RESERVATION` | 409 | 내가 대표인 끝나지 않은 예약이 이미 있음 |
| `PARTICIPANT_TIME_TAKEN` | 409 | 참여자가 같은 시간 다른 방 예약 중 |
| `IDEMPOTENCY_KEY_REUSED` | 409 | 같은 키를 다른 내용에 씀 |
| `RESERVATION_BUSY` | 503 | 동시 요청이 몰림 (드묾). 잠시 후 다시 |

만들기 직전에 지난 예약 정리(노쇼·자동 퇴실)를 한 번 돌려요. 그래서 입실 마감이 지난 예약 때문에 `409`가 나지 않아요.

### `POST /api/reservations/instant` 지금 바로 쓰기

시작은 서버가 정해요(누른 시각, 초 아래 버림). 끝만 보내요. `Idempotency-Key`도 똑같이 받아요(같은 키로 다시 보내면 시작 시각이 달라도 처음 예약을 `200`).

```json
{ "roomId": 1, "endAt": "2026-11-03T16:30:00+09:00", "participantStudentIds": ["20230002", "20230003"] }
```
→ `201`, 응답 모양은 `POST /api/reservations`와 같아요.

- 끝은 30분 칸이고, 최대 (지금 + 2시간)을 30분 칸으로 올린 시각까지예요. 14:17에 누르면 16:30까지 고를 수 있어요.
- 다음 예약이 있으면 그 전까지만 돼요(아니면 `409 RESERVATION_TIME_TAKEN`).
- 10분 안에 키패드로 입실하지 않으면 노쇼예요.
- 오류는 `POST /api/reservations`와 같아요. 다만 끝이 30분 칸이 아니면 `RESERVATION_NOT_ON_SLOT`("끝 시각은 정각 또는 30분이어야 합니다"), 길이가 틀리면 `RESERVATION_INVALID_LENGTH`("지금 바로 쓰기는 지금부터 16:30까지 고를 수 있습니다")가 나가요.
- 미리 예약에 쓴 `Idempotency-Key`를 여기 다시 쓰면 `409 IDEMPOTENCY_KEY_REUSED`예요.

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
`accessCode`는 대표자이고 끝나지 않은 예약(`BOOKED`·`IN_USE`)일 때만 값이 있어요. 참여자이거나, 퇴실·자동 퇴실·자리 비움 퇴실·취소·노쇼된 예약이면 `null`이에요.

예약 상태(`status`): `BOOKED` 예약됨 → `IN_USE` 입실 → `CHECKED_OUT` 퇴실 / `AUTO_CHECKED_OUT` 시간 끝나 자동 퇴실 / `AWAY_RELEASED` 자리 비움 자동 퇴실(재실 센서 15분 비어 있음, v0.2 FR-DEV-103) / `NO_SHOW` 입실 마감까지 안 옴(미리 예약은 시작 + 15분, 지금 바로 쓰기는 누른 뒤 10분. 끝 시각이 먼저 오면 끝 시각) / `CANCELED` 취소. 노쇼·자동 퇴실은 core가 1분마다 바꿔요.

### `POST /api/reservations/{id}/cancel` 시작 전 취소

본문 없음 → `200 {"id": 12, "status": "CANCELED"}`

| 오류 | 상태 | 언제 |
|---|---|---|
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내 예약이 아님 |
| `NOT_RESERVATION_OWNER` | 403 | 참여자가 취소하려 함 |
| `RESERVATION_NOT_CANCELABLE` | 409 | 이미 시작·끝남·취소됨. `message`가 이유마다 달라요: "이미 취소된 예약입니다" / "이미 시작 시각이 지난 예약은 취소할 수 없습니다" / "사용 중인 예약은 취소할 수 없습니다. 퇴실해 주세요" / "이미 끝난 예약입니다" |

### `POST /api/reservations/{id}/checkout` 퇴실

본문 없음 → `200 {"id": 12, "status": "CHECKED_OUT"}`. 방이 바로 비어요. 조기 퇴실 확인 창은 화면이 띄워요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내 예약이 아님 |
| `NOT_RESERVATION_OWNER` | 403 | 참여자가 퇴실하려 함 |
| `RESERVATION_NOT_IN_USE` | 409 | 입실 전이거나 이미 끝남 |

### `POST /api/reservations/{id}/still-here` 아직 있어요

본문 없음 → `200 {"id": 12, "status": "IN_USE"}`. 대표자, 사용 중, **예약당 1번**. 누르면 자리 비움 시계를 누른 시각부터 다시 재고, 누른 뒤에는 **10분 비면 알림 없이 바로 퇴실**해요(FR-DEV-103, 11/5 지영). 화면은 `AWAY_SOON` 알림에만 이 버튼을 붙여요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `STILL_HERE_DISABLED` | 404 | 서버 설정으로 꺼 둠 (기본은 켜짐) |
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내 예약이 아님 |
| `NOT_RESERVATION_OWNER` | 403 | 참여자가 누름 |
| `RESERVATION_NOT_IN_USE` | 409 | 사용 중이 아님 |
| `STILL_HERE_ALREADY_USED` | 409 | 이 예약에서 이미 씀 |

### `POST /api/reservations/{id}/photos` 입실·퇴실 사진 올리기

`multipart/form-data`로 `kind`(`CHECK_IN` 또는 `CHECK_OUT`)와 `file`(사진 1장)을 보내요 → `201 {"reservationId": 12, "kind": "CHECK_IN", "uploadedAt": "...+09:00", "sizeBytes": 812345}`. 선택 기능이라 안 올려도 입실·퇴실은 그대로 돼요(FR-RPT-102).

- 그 예약의 대표자·참여자 누구나. 종류별 1장(먼저 올린 사진)
- 입실 사진: 사용 중(`IN_USE`)일 때. 퇴실 사진: 사용 중이거나 퇴실한 뒤 30분 안
- JPEG·PNG만, 최대 2MB. 화면에서 줄여서 보내 주세요(nginx 본문 크기 제한도 2MB보다 커야 해요)
- 올린 지 30일 지나면 지워져요

| 오류 | 상태 | 언제 |
|---|---|---|
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내가 들어간 예약이 아님 |
| `PHOTO_INVALID` | 400 | `kind`가 틀리거나, 파일이 없거나 JPEG·PNG가 아님 |
| `PHOTO_TOO_LARGE` | 400 | 2MB 넘음 |
| `PHOTO_NOT_ALLOWED_NOW` | 409 | 입실 사진인데 사용 중이 아님, 퇴실 사진인데 사용 전이거나 퇴실 30분 지남 |
| `PHOTO_ALREADY_UPLOADED` | 409 | 그 종류 사진이 이미 있음 |

### `GET /api/reservations/{id}/photos/{kind}` 사진 보기

사진 파일 그대로(`image/jpeg`·`image/png`). 그 예약의 대표자·참여자와 관리자만. 없거나 볼 수 없으면 `404 PHOTO_NOT_FOUND`. 신고 판정 화면(FR-WEB-101)은 "앞 예약의 `CHECK_OUT`"과 "신고자 예약의 `CHECK_IN`"을 이 주소로 불러요.

### `POST /api/reservations/{id}/continue` 연장하기

형식은 [요구사항 FR-RSV-105](../requirements/requirements.md#72-기능-요구사항-목록) 그대로 만들어요. 1회를 넘기면 `409 RESERVATION_EXTENSION_LIMIT`(11/3 지영 확정). 구현되면 이 절을 채워요.

### `GET /api/reservations/stats?from=2026-11-01&to=2026-11-30` 예약 건수 — 관리자만

예약을 **만든 날**(서울 날짜, 양 끝 포함) 기준으로 WEB/AGENT를 따로 세요. 노쇼·취소된 예약도 만든 건수에 들어가요. `from`·`to`를 빼면 처음부터·지금까지 전부이고, 응답의 `from`·`to`도 `null`이에요. 숫자만 나오고 누가·어느 방인지는 안 나와요.

→ `200`
```json
{ "from": "2026-11-01", "to": "2026-11-30", "total": 42, "web": 35, "agent": 7 }
```

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 날짜 모양이 틀림(`YYYY-MM-DD` 아님), `from`이 `to`보다 늦음, 연도가 2000–2100 밖 |
| `UNAUTHORIZED` | 401 | 로그인 안 함 |
| `ADMIN_ONLY` | 403 | 관리자(`users.role = ADMIN`)가 아님 |

관리자 지정은 아직 화면이 없어 SQL로 해요: `UPDATE users SET role = 'ADMIN' WHERE email = '...'` (core V6).

### `POST /api/reports` 신고

"방에 문제가 있나요?" 버튼 뒤의 신고(FR-RPT-101). 지금 **사용 중(`IN_USE`)인 예약의 대표자나 참여자**만 보낼 수 있고, 그 방이거나 **옆 방**일 때만 돼요. 옆 방은 번호가 이웃한 방(1–2, 2–3, 3–4, 4–5, 요구사항 Q-05)이에요.

```json
{ "roomId": 2, "category": "NOISE", "content": "벽 너머로 계속 소리 질러요", "noiseConfirmed": true }
```
→ `201`
```json
{ "id": 7, "roomId": 2, "category": "NOISE", "verdict": "OPEN", "createdAt": "2026-11-12T14:20:05+09:00" }
```

- `category`: `BROKEN`(비품 고장) · `WINDOW_OPEN`(창문 열림) · `MESSY`(어질러짐) · `NOISE`(소음)
- `noiseConfirmed`: "정말 소란인가요?" 체크. 소음일 때만 필수
- `content`: 메모, 선택, 500자까지
- `verdict`는 관리자 판정(FR-RPT-103) 전까지 `OPEN`이에요. 사진·결과 알림·판정 API는 아직 없어요

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | `roomId`·`category` 빠짐, 없는 종류, 메모 500자 넘음 |
| `REPORT_NOISE_NOT_CONFIRMED` | 400 | 소음인데 `noiseConfirmed`가 `true`가 아님 |
| `REPORT_NOT_ALLOWED` | 403 | 지금 그 방이나 옆 방을 사용 중인 예약자가 아님 (입실 전·퇴실 후 포함) |

## 5-2. 알림함 — 모두 로그인 필요 (FR-NTF-103)

`GET /api/notifications/me` 최근 50개, 최근 순
```json
[{ "id": 3, "reservationId": 12, "kind": "END_SOON", "title": "종료 10분 전입니다",
   "body": "창작실 1 11:30에 끝나요. ...", "createdAt": "2026-11-06T11:20:00+09:00", "read": false }]
```

`GET /api/notifications/me/unread-count` 종 아이콘 점 → `200 { "count": 3 }`

`POST /api/notifications/{id}/read` 알림 하나 읽음 → `204`
- 이미 읽었어도 `204` (처음 읽은 시각이 남아요)
- 없거나 남의 알림이면 `404 NOTIFICATION_NOT_FOUND` (남의 알림이 있는지도 안 알려 줘요)

`POST /api/notifications/read-all` 모두 읽음 → `200 { "count": 2 }` (이번에 읽음으로 바뀐 개수)

## 6. 출입 (키패드)

### `POST /api/access/verify` — 장치 토큰 필요

```json
{ "eventId": "1-48213-18", "roomId": 1, "deviceId": "keypad-1", "code": "123456" }
```
`eventId`(1~64자)는 장치가 누를 때마다 새로 만드는 값이라, 같은 값이 다시 와도 한 번만 처리하고 처음 결과를 그대로 돌려줘요. `deviceId`는 50자 이하(선택).

→ 거절도 `200`이에요. 장치는 `result`만 보고 LED를 켜요.

```json
{ "result": "OPENED" }
{ "result": "DENIED", "reason": "WRONG_CODE" }
```
`reason`: `WRONG_CODE` 틀림, `NOT_YET` 아직 이름(시작 10분 전부터 열림), `LOCKED` 5번 연속 틀려 1분 잠김.

`WRONG_CODE`가 나가는 경우는 이것도 포함해요.
- 6자리 숫자가 아닌 `code`
- 끝 시각이 지난 예약, 퇴실·자동 퇴실·취소·노쇼된 예약의 번호
- 아직 입실 안 했는데 입실 마감(미리 예약 시작 + 15분, 지금 바로 쓰기 누른 뒤 10분)이 지난 번호. 이미 입실한 사람은 끝 시각 전까지 같은 번호로 다시 열 수 있어요

| 오류 | 상태 | 언제 |
|---|---|---|
| `DEVICE_UNAUTHORIZED` | 401 | 장치 토큰이 없거나 틀림 |
| `INVALID_REQUEST` | 400 | `eventId`·`roomId` 빠짐, 길이 넘음 |

## 6-2. 재실 · CO2 센서

### `POST /api/sensor-events` — 장치 토큰 필요

방마다 붙은 장치(ESP32, 시험용은 젯슨+우노)가 "지금 어떤지"만 알려요. 10·15분 자리 비움(FR-DEV-103), 환기 알림(FR-DEV-104), 방 현황(FR-DEV-102) 판단은 모두 core가 해요. 형식은 11/5 지영·수연이 맞췄어요. 장치 쪽 자세한 설명은 mspace-device `docs/sensor-events-api.md`.

재실:
```json
{ "eventId": "2f1c6b9e-6d0a-4a77-9a52-0c1f3c1d8e41", "roomId": 2, "deviceId": "sensor-2",
  "kind": "OCCUPANCY", "occupied": false, "sources": { "radar": false },
  "observedAt": "2026-11-05T14:05:00+09:00" }
```

CO2:
```json
{ "eventId": "9a0e3f55-1b2c-4d6e-8f90-a1b2c3d4e5f6", "roomId": 2, "deviceId": "sensor-2",
  "kind": "CO2", "ppm": 1080, "source": "scd40", "observedAt": "2026-11-05T14:06:00+09:00" }
```

| 필드 | 형식 | 뜻 |
|---|---|---|
| `eventId` | 글자 1~64자 (UUID) | 장치가 이벤트마다 새로 만들어요. 다시 보낼 때만 같은 값이라, 같은 값은 한 번만 저장해요 |
| `roomId` · `deviceId` | 정수 · 글자 50자 이하 | 방 번호, 보낸 장치(`sensor-방번호`) |
| `kind` | `OCCUPANCY` 또는 `CO2` | 재실 / CO2 |
| `occupied` | 참/거짓 (재실) | 장치가 센서들을 합친 판단. 하나라도 "있음"이면 `true` (FR-DEV-103) |
| `sources` | 객체, 선택 (재실) | 센서별 값(`pir`, `radar`, `light`). core는 판단에 안 쓰고 기록만 해요 (PIR·레이더 정확도 비교용) |
| `ppm` · `source` | 정수 0~10000 · 글자, 선택 (CO2) | CO2 농도, 센서 이름 |
| `observedAt` | 시각 `+09:00` | 센서가 본 시각. 10·15분 타이머와 CO2 3분 판단은 받은 시각이 아니라 이 시각으로 재요. 장치는 인터넷 시계(NTP)를 맞춘 뒤에만 보내요 |

**언제 보내나**
- 재실: 있음↔없음이 **바뀔 때 바로**, 바뀌지 않아도 **10초마다** 지금 상태를 다시 보내요(살아 있다는 신호). 같은 상태가 다시 오면 core는 "마지막 받은 시각"만 바꿔요.
- 한 방에서 **1분 동안** 재실 이벤트가 안 오면 core는 그 방을 **"있음"**으로 봐요(끊김 때문에 자동 퇴실되지 않게, FR-DEV-101 "틀리면 있음 쪽으로").
- CO2: 1분마다. **1000ppm 넘는 값이 3분 연속**이면 환기 알림(같은 방 30분에 한 번).

→ 처리했으면 `200`.

```json
{ "result": "STORED" }
{ "result": "DUPLICATE" }
```

| 오류 | 상태 | 언제 | 장치 동작 |
|---|---|---|---|
| `INVALID_REQUEST` | 400 | 빠진 값, 없는 방 번호(`roomId`), 모르는 `kind`, `ppm` 범위 밖, 오프셋 없는 시각, 5분 넘게 미래인 시각 | 그 이벤트만 버려요 |
| `DEVICE_UNAUTHORIZED` | 401 | 장치 토큰 없음·틀림 | 이벤트를 모아 둔 채 간격을 늘려 다시 보내요(설정 문제라 고치면 이어서 보냄) |
| `5xx`, 연결 실패 | - | 서버 문제 | 같은 `eventId`로 다시 보내요 (1→2→4…30초) |

정할 것:
- 장치 토큰: 지금 core는 키패드와 같은 토큰 하나뿐이에요. 센서용 토큰을 장치마다 따로 두고 토큰에 방·장치를 묶을지(다르면 `403 DEVICE_ROOM_MISMATCH`) 정하기 (지영·재웅)
- API 게이트웨이: 장치 토큰은 JWT가 아니라서 `/api/sensor-events`를 `/api/access/verify`처럼 공개 경로로 넣기 (재웅)

## 7. 비서 (`/api/agent`) — 로그인 필요, 아직 브랜치

비서 서버는 받은 `Authorization`을 그대로 core에 넘겨요. 그래서 비서도 학생 본인 권한만큼만 할 수 있어요.

### `POST /api/agent/chat` 대화 접수 → `GET /api/agent/chat/{jobId}` 결과 조회 (폴링)

비서는 답을 만드는 데 몇 초~45초가 걸려서, **보내면 바로 접수 번호를 주고 화면이 결과를 조회**해요 (5차 회의 결정, 웹소켓 대신 폴링). 요청 하나가 짧아 nginx 제한 시간(60초)에 걸리지 않아요.

```json
{ "message": "내일 2시에 빈방 있어?", "history": [ { "role": "user", "text": "..." }, { "role": "assistant", "text": "..." } ] }
```
→ `202`, `Location: /api/agent/chat/j_…`

```json
{ "jobId": "j_Q2x...", "status": "PENDING", "pollAfterMs": 1000 }
```

화면은 `pollAfterMs`마다 `GET /api/agent/chat/{jobId}`를 불러요. 아직이면 `200 { "jobId": "...", "status": "PENDING", "pollAfterMs": 1000 }`, 끝나면:

```json
{
  "jobId": "j_Q2x...",
  "status": "DONE",
  "result": {
    "reply": "내일 14:00에 1번 방, 2번 방이 비어 있어요. 예약할까요?",
    "confirmations": [ { "tool": "create_reservation", "token": "c_9f2...", "summary": "1번 방 · 11/4 14:00–16:00",
                         "notes": [], "needsParticipants": 2, "expiresAt": "2026-11-03T16:10:00+09:00" } ],
    "accessCodeCards": [ { "reservationId": 12 } ],
    "toolCalls": 2, "stoppedByLimit": false
  },
  "error": null
}
```
실패하면 `200 { "status": "FAILED", "result": null, "error": { "code": "LLM_RATE_LIMITED", "message": "..." } }`. `error.message`를 그대로 보여 주면 돼요.

쓰기(예약·취소·퇴실)는 바로 하지 않고 확인 카드(`confirmations`)로 돌려줘요. 입실 번호는 모델에 보내지 않아서 `accessCodeCards`에는 예약 id만 있어요.

- 결과에는 확인 토큰이 들어 있어서 **접수한 학생만** 볼 수 있어요. 남의 `jobId`는 없는 것과 같은 `404`. 응답은 `Cache-Control: no-store`
- 끝난 결과는 5분 보관해요. 그 뒤에는 `404`라 다시 보내야 해요
- 화면은 90초 넘게 `PENDING`이면 조회를 멈추고 다시 보내 달라고 안내해요 (비서 기한 45초 + 여유)
- 학생 한 명은 동시에 한 건만 보내요. 앞 답이 오기 전에 보내면 `409`

| 오류 | 상태 | 어디서 | 언제 |
|---|---|---|---|
| `UNAUTHORIZED` | 401 | 접수·조회 | 로그인 안 함 |
| `INVALID_REQUEST` | 400 | 접수 | 빈 말, 너무 긴 말(1,000자), 깨진 대화 기록 |
| `CHAT_IN_PROGRESS` | 409 | 접수 | 같은 학생의 앞 질문에 아직 답하는 중 |
| `AGENT_BUSY` | 503 | 접수 | 비서 서버 대기열이 가득 참. 잠시 뒤 다시 |
| `CHAT_JOB_NOT_FOUND` | 404 | 조회 | 없는 번호, 남의 번호, 보관 시간(5분) 지남 |
| `LLM_RATE_LIMITED` | `error.code` | 조회(`FAILED`) | Gemini 무료 한도 |
| `LLM_UNAVAILABLE`, `LLM_NOT_CONFIGURED` | `error.code` | 조회(`FAILED`) | LLM 응답 없음, 키 설정 안 됨 |
| `TOOLS_UNAVAILABLE` | `error.code` | 조회(`FAILED`) | MCP 서버 연결 안 됨 |
| `AGENT_FAILED` | `error.code` | 조회(`FAILED`) | 그 밖의 오류 |

### `POST /api/agent/confirmations/{token}` 확인 카드 [확인]

```json
{ "participantStudentIds": ["20230002", "20230003"] }
```
예약 만들기 카드일 때만 본문을 보내요. 학번 수는 카드의 `needsParticipants`와 같아야 해요. 취소·퇴실은 본문 없음.

→ `200` (예약·취소·퇴실 모두 같은 모양. core의 `201` 본문을 그대로 주지 않아요)

```json
{ "action": "CREATE", "reservationId": 12, "status": "BOOKED", "accessCode": "482913",
  "replayed": false, "message": "예약했어요." }
```
`action`: `CREATE` · `CANCEL` · `CHECKOUT`. `accessCode`는 예약 만들기일 때만 값이 있고 나머지는 `null`. `replayed`가 `true`면 같은 확인이 이미 처리돼 core가 처음 예약을 돌려준 경우예요(멱등 키).

| 오류 | 상태 | 언제 |
|---|---|---|
| `CONFIRMATION_NOT_FOUND` | 404 | 없는 토큰이거나 다른 학생의 토큰 |
| `CONFIRMATION_USED` | 409 | 이미 [확인]한 토큰 (한 번만 쓸 수 있어요) |
| `CONFIRMATION_EXPIRED` | 410 | 2분 지남 → 비서에게 다시 말하기 |
| `INVALID_REQUEST` | 400 | 학번 수가 인원과 다르거나 빈 학번. **토큰은 그대로라** 학번만 고쳐 다시 [확인]할 수 있어요 |
| core 오류 그대로 | 400·403·404·409 | 5장 표의 코드·문장 그대로 (예: `RESERVATION_TIME_TAKEN`, `PARTICIPANT_NOT_FOUND`). 취소 409 `RESERVATION_NOT_CANCELABLE`의 문장은 이유마다 달라요 |
| `RESERVATION_UNKNOWN` | 503 | 예약 만들기에서 core 응답을 못 받음(같은 키로 한 번 다시 보낸 뒤에도). 됐을 수 있으니 "내 예약에서 확인" 안내 |
| `STATUS_CHANGE_UNKNOWN` | 503 | 취소·퇴실에서 core 응답을 못 받음. 다시 누르기 전에 내 예약 확인 |
| `TOOLS_UNAVAILABLE` | 503 | 비서 도구 서버(mcp-server)에 연결 안 됨 |

참여자 학번을 카드에서 받을 때 `GET /api/members?studentId=`(4장)로 이름(가린 이름)을 미리 보여 주면 잘못 넣는 일이 줄어요.

### `GET /api/agent/reservations/{id}/access-code` 입실 번호 카드

대표자에게만 번호를 줘요. `Cache-Control: no-store`. → `200`

```json
{ "reservationId": 12, "roomName": "1번 방", "startAt": "2026-11-03T14:00:00+09:00",
  "endAt": "2026-11-03T16:00:00+09:00", "accessCode": "482913" }
```

| 오류 | 상태 | 언제 |
|---|---|---|
| `UNAUTHORIZED` | 401 | 로그인 안 함 |
| `RESERVATION_NOT_FOUND` | 404 | 없거나 내 예약이 아님 |
| `NOT_RESERVATION_OWNER` | 403 | 참여자 → "대표자에게 문의하세요" |
| `RESERVATION_FINISHED` | 409 | 이미 끝났거나 취소된 예약 |
| `ACCESS_CODE_UNAVAILABLE` | 502 | core가 번호를 주지 않음 → 예약 화면에서 확인 |

## 8. 코드와 문서가 아직 다른 곳

| 무엇 | 지금 | 할 일 |
|---|---|---|
| `GET /api/rules` | 요구사항에만 있고 코드 없음 | core에 추가 |
| springdoc `/swagger-ui` | 컨벤션에만 있고 core에 의존성 없음 | core에 추가하면 이 문서는 요약본으로 줄이기 |
| 비서 API | 도구(MCP) 쪽 #26·#29, 에이전트 쪽 #31은 develop에 머지됨. #30·#33·#34·#40·#41은 리뷰 중 | 모두 머지되면 0장 상태 ✅로 |
| `created_via` | core PR #45: `X-Created-Via: AGENT`면 AGENT로 저장 | #45 머지 뒤 비서(MCP)가 core 호출에 `X-Created-Via: AGENT`를 붙이기(영준) |
| 연장 `POST /api/reservations/{id}/continue` | core PR #51 리뷰 중 (#45 다음 머지) | 머지되면 0장 상태 ✅로, 비서 `continue_reservation` 도구 추가(영준) |
| 신고 `POST /api/reports` | core PR #52 리뷰 중 (#51 다음 머지). 사진·결과 알림·관리자 판정(FR-RPT-103) 없음 | 머지되면 0장 상태 ✅로. 판정 API·화면은 다음 PR(지영) |
| 자리 비움 `AWAY_RELEASED` (v0.2) | 요구사항 FR-DEV-103(11/3 회의 결정)에만 있음 | core에 상태 추가, 10·15분 판단 스케줄러, 30일 3번 = 노쇼 1번 계산(지영·수연) |
| 노쇼 벌칙 `USER_SUSPENDED`·`PARTICIPANT_SUSPENDED` | core 브랜치 `feat/rsv-no-show-penalty`(V9 `user_suspension`), PR 대기 | 머지되면 이 줄 지우기 |
| 재실 센서 `POST /api/sensor-events` | 형식은 6-2장(11/5 지영·수연 확정). core #60 리뷰 중. 장치 코드는 mspace-device #24 (지금은 보내지 않고 화면에만 출력) | core #60 머지(지영), 센서용 장치 토큰(지영·재웅), API 게이트웨이 공개 경로(재웅) |
