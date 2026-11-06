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
| (안 씀) | `GET /api/auth/check` | 토큰 확인. 11/4 게이트웨이로 바꾼 뒤로는 게이트웨이가 JWT를 직접 확인해서 부르는 곳이 없어요 | FR-AUTH-002 | ✅ develop |
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
| 관리자 | `GET /api/reports?verdict=OPEN` | 신고 목록 | FR-RPT-103 | ✅ develop |
| 관리자 | `POST /api/reports/{id}/judge` | 신고 판정 + 경고·정지·알림 | FR-RPT-103·104 | ✅ develop |
| 관리자 | `POST /api/reservations/{id}/force-checkout` | 강제 퇴실 (퇴출) | FR-RPT-104 | ✅ develop |
| 관리자 | `GET /api/reports/{id}/evidence` | 판정용 증거 사진 두 장 | FR-WEB-101 | PR 대기 (core #76) |
| 관리자 | `GET /api/penalties?days=30` | 경고·정지 목록 | FR-WEB-101 | PR 대기 (core #75) |
| 관리자 | `GET /api/stats/usage?from=&to=` | 이용 통계 (이용률·노쇼율·시간대·센서) | FR-WEB-101 | PR 대기 (core #74) |
| 화면 | `GET /api/notifications/me` | 내 알림함 (최근 50개, 읽음 여부) | FR-NTF-103 | ✅ develop |
| 화면 | `GET /api/notifications/me/unread-count` | 안 읽은 알림 개수 (종 아이콘 점) | FR-NTF-103 | ✅ develop |
| 화면 | `POST /api/notifications/{id}/read` | 알림 하나 읽음 | FR-NTF-103 | ✅ develop |
| 화면 | `POST /api/notifications/read-all` | 내 알림 모두 읽음 | FR-NTF-103 | ✅ develop |
| 키패드 | `POST /api/access/verify` | 6자리 번호 확인 → 문 열기 | FR-ACC-001 | ✅ develop |
| 재실·CO2 센서 | `POST /api/sensor-events` | 방에 사람이 있는지, CO2 값 알리기 | FR-DEV-101~104 | ✅ develop |
| 화면 | `GET /api/rules` | 이용 규칙 | FR-WEB-003 | ✅ develop |
| 화면 | `POST /api/agent/chat` | 비서에게 말 보내기 (접수, `202`) | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `GET /api/agent/chat/{jobId}` | 비서 답 조회 (폴링) | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `DELETE /api/agent/chat/history` | 새 대화 (서버가 기억한 대화 지우기) | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `POST /api/agent/confirmations/{token}` | 확인 카드 [확인] | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `GET /api/agent/reservations/{id}/access-code` | 비서 입실 번호 카드 | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `GET`·`PUT`·`DELETE /api/agent/schedule` | 내 시간표 보기·통째로 바꾸기·지우기 | FR-AGT-102a | ✅ develop (assistant) |
| 화면 | `POST /api/agent/schedule/parse` | 시간표 캡처 읽기 (초안만, 저장 안 함) | FR-AGT-102a | ✅ develop (assistant) |
| 화면 | `GET /api/agent/briefing` | 오늘 브리핑 (시간표·예약·빈 방·수업 뒤 칸, 모델 안 거침) | FR-AGT-102b·103 | ✅ develop (assistant) |

`/api/agent/`로 시작하면 API 게이트웨이가 비서 서버(agent)로, 나머지 `/api/`는 core로 보내요. 게이트웨이가 로그인 토큰을 먼저 확인해요([아키텍처](../architecture/README.md)).

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

### `GET /api/auth/check` 토큰 확인 (지금은 안 씀)

`Authorization: Bearer` → 맞으면 `204`, 아니면 `401`. 예전 입구 nginx의 `auth_request`용이었고, 11/4 게이트웨이로 바꾼 뒤로는 게이트웨이가 토큰을 직접 확인해요.

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
{ "studentId": "20230002", "name": "홍*동", "me": false }
```

`me`는 찾은 사람이 나(로그인한 대표자)면 `true`예요. 대표자는 이미 들어 있으니 화면은 **추가** 버튼 대신 "추가됨"을 보여 줘요. 그래도 대표자 학번을 `participantStudentIds`에 넣으면 예약 만들기가 `400 PARTICIPANT_DUPLICATE`로 막아요.

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
  "status": "BOOKED", "headcount": 3, "accessCode": "482913", "qrText": "MSPACE:1:482913" }
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
| `USER_SUSPENDED` | 403 | 내가 이용 정지 중: 노쇼 2번(FR-RSV-106) 또는 경고 3번(FR-RPT-104, core #70). 메시지에 끝나는 날짜 |
| `PARTICIPANT_SUSPENDED` | 403 | 참여자 중 이용 정지 중인 사람이 있음 (누구인지는 안 알려 줌) |
| `PARTICIPANT_NOT_FRIEND` | 403 | 참여자 중 대표자와 친구가 아닌 사람이 있음 (누구인지는 안 알려 줌, 11/6) |
| `RESERVATION_TIME_TAKEN` | 409 | 그 방 그 시간에 이미 예약 |
| `OWNER_HAS_ACTIVE_RESERVATION` | 409 | 내가 대표자나 참여자로 들어간 끝나지 않은 예약이 이미 있음 (FR-RSV-003, 11/5) |
| `PARTICIPANT_HAS_ACTIVE_RESERVATION` | 409 | 참여자 중 끝나지 않은 예약이 이미 있는 사람이 있음 (누구인지는 안 알려 줌, core #64) |
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
    "participants": [ { "name": "김*영", "owner": true }, { "name": "홍*동", "owner": false } ],
    "qrText": "MSPACE:1:482913" }
]
```
`accessCode`는 대표자이고 끝나지 않은 예약(`BOOKED`·`IN_USE`)일 때만 값이 있어요. 참여자이거나, 퇴실·자동 퇴실·자리 비움 퇴실·취소·노쇼된 예약이면 `null`이에요.

`qrText`는 폰에 띄울 **입실 QR 안의 글자**예요(11/6, core #71). 형식은 `MSPACE:<방 번호>:<입실 번호>`이고, `accessCode`가 `null`이면 `qrText`도 `null`이에요. 예약 만들기·지금 바로 쓰기·연장하기 응답에도 있어요. core는 QR 그림을 만들지 않고, 화면이 이 글자로 QR을 그려요. 문 앞 웹캠(mspace-device `qr_door.py`)이 읽어서 키패드와 같은 `POST /api/access/verify`로 물어요(`deviceId` `qr-<방>`). 그래서 열림 규칙과 5번 틀리면 1분 잠금(방 단위)도 키패드와 같아요.

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
- `verdict`는 관리자 판정(FR-RPT-103) 전까지 `OPEN`이에요

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | `roomId`·`category` 빠짐, 없는 종류, 메모 500자 넘음 |
| `REPORT_NOISE_NOT_CONFIRMED` | 400 | 소음인데 `noiseConfirmed`가 `true`가 아님 |
| `REPORT_NOT_ALLOWED` | 403 | 지금 그 방이나 옆 방을 사용 중인 예약자가 아님 (입실 전·퇴실 후 포함) |
| `REPORT_SUSPENDED` | 403 | 허위 신고 경고가 쌓여 신고 정지 중 (메시지에 끝나는 날짜, FR-RPT-103, core #70) |

### `GET /api/reports?verdict=OPEN` 신고 목록 — 관리자만

최근 신고가 위. `verdict`를 빼면 전부. 관리자가 아니면 `403 ADMIN_ONLY`.
```json
[{ "id": 7, "roomId": 2, "category": "MESSY", "content": null, "verdict": "OPEN",
   "reporterReservationId": 12, "createdAt": "2026-11-12T14:20:05+09:00", "judgedAt": null }]
```

### `POST /api/reports/{id}/judge` 판정 — 관리자만

```json
{ "verdict": "CONFIRMED" }
```
→ `200` 판정이 반영된 신고(목록과 같은 모양). 대기(`OPEN`)인 신고에 한 번만.

| 판정 | 신고자 | 그 방 직전 이용 예약의 대표자 |
|---|---|---|
| 사실 `CONFIRMED` | 고맙다는 알림 | 경고 1회 + 알림 `WARNING` |
| 확인 불가 `UNVERIFIED` | 처리하지 않았다는 알림, 벌칙 없음 | 없음 |
| 허위 `FALSE` | 경고 1회 + 알림 | 없음 |

- 신고자 알림 종류는 `REPORT_RESULT`. 알림함(5-2)에 보여요
- 직전 이용 예약: 같은 방, 신고자 예약 제외, 신고 전에 입실한 예약 중 가장 최근. 노쇼(입실 안 함)는 빼요. 찾지 못하면 경고 없음
- 30일 안에 경고 3회면 14일 정지. 사용 위반은 예약·연장 정지(`403 USER_SUSPENDED`), 허위 신고는 신고만 정지(`403 REPORT_SUSPENDED`). 두 경고는 따로 세요
- 판정·경고·정지·알림은 한 번에 저장돼요. 하나라도 실패하면 판정도 안 돼요

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | `verdict`가 없거나 `CONFIRMED`·`UNVERIFIED`·`FALSE`가 아님 |
| `ADMIN_ONLY` | 403 | 관리자가 아님 |
| `REPORT_NOT_FOUND` | 404 | 없는 신고 |
| `REPORT_NOT_JUDGEABLE` | 409 | 소음 신고 (판정 대상 아님) |
| `REPORT_ALREADY_JUDGED` | 409 | 이미 판정함 |

### `POST /api/reservations/{id}/force-checkout` 강제 퇴실 — 관리자만

사용 중(`IN_USE`)인 예약을 관리자가 바로 끝내요(FR-RPT-104 퇴출, core #73). 본문은 빼도 돼요.
```json
{ "reason": "신고 확인, 소음 계속" }
```
→ `200 { "id": 12, "status": "CHECKED_OUT" }`

- 그 예약의 대표자·참여자 모두에게 알림 `FORCED_CHECKOUT`("관리자가 퇴실 처리했어요", 사유가 있으면 본문에 붙음)
- 사유는 200자까지. 알림에 그대로 보이니 이름·학번은 쓰지 않아요
- 보통 퇴실과 같은 조건부 UPDATE라 두 관리자가 동시에 눌러도 한 번만 돼요. 퇴실·알림 저장은 한 트랜잭션
- 경고·정지는 따로 걸지 않아요(벌칙은 신고 판정이 맡음)

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 사유가 200자를 넘음 |
| `ADMIN_ONLY` | 403 | 관리자가 아님 |
| `RESERVATION_NOT_FOUND` | 404 | 없는 예약 |
| `RESERVATION_NOT_IN_USE` | 409 | 아직 입실 전이거나 이미 끝난 예약 |

### 관리자 화면용 API (PR 대기, core #74~#76)

모두 관리자만(`403 ADMIN_ONLY`). 머지되면 모양을 자세히 적어요.

| 주소 | 하는 일 |
|---|---|
| `GET /api/reports/{id}/evidence` | 직전 이용 예약의 퇴실(`CHECK_OUT`) 사진, 신고자 예약의 입실(`CHECK_IN`) 사진. 사진마다 `exists`, 있으면 `url`(사진 보기 주소), 없으면 `note`(안 올림·30일 지나 지움·직전 예약 없음) |
| `GET /api/penalties?days=30` | 최근 days일(1–365) 경고(받은 사람 학번·이름, 이유 `VIOLATION`·`FALSE_REPORT`, 신고·예약 id)와 정지(이용 `USE`·신고 `REPORT`, 진행 중이면 `active`) |
| `GET /api/stats/usage?from=&to=` | 서울 날짜 기준, to 포함, 92일까지(빼면 오늘까지 7일). 방별·전체 예약 수(상태별), 노쇼율·실사용률·이용률, 평균 이용 시간, 시간대별(0–23시), CO2 평균·최고·1000ppm 초과 비율, 예약 없이 사람이 있던 시간 |

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

## 6-1. 이용 규칙

### `GET /api/rules` 이용 규칙 — 토큰 필요 없음 (core #57)

화면 "이용 규칙"(FR-WEB-003)과 비서 `usage_rules` 도구가 같은 내용을 받아요. 로그인 전에도 볼 수 있어요(수용 기준).
- `reservation`의 숫자는 core가 실제 검사에 쓰는 설정값을 그대로 내보내요. 설정을 바꾸면 화면·비서 안내도 같이 바뀌어요.
- `conduct`(이용 수칙)·`reminders`(한 줄 안내) 문구는 core 설정(`application.yml`의 `mspace.rules`)에 있어요. 원본은 [요구사항 3.1](../requirements/requirements.md).

```json
200
{
  "reservation": {
    "bookingWindowDays": 7, "slotMinutes": 30, "maxMinutes": 120,
    "checkinOpenBeforeMinutes": 10, "noShowMinutes": 15, "instantNoShowMinutes": 10,
    "extension": { "maxCount": 1, "openBeforeMinutes": 10, "maxMinutes": 120, "chainGapMinutes": 30 }
  },
  "conduct": [
    { "key": "FOOD", "title": "음식", "rule": "음식물 반입 금지예요. 먹을 것과 마실 것(음료 포함) 모두 가져올 수 없어요",
      "penalty": "신고가 사실로 판정되면 경고. 기기·비품이 고장 나면 배상을 요구해요" }
  ],
  "reminders": { "checkin": "음식물은 반입 금지예요", "checkout": "자리 정리하셨나요?" }
}
```

| 필드 | 뜻 |
|---|---|
| `reservation.bookingWindowDays` | 오늘 포함 며칠 안에 시작하는 예약만 (7) |
| `reservation.slotMinutes` · `maxMinutes` | 예약 단위 30분, 1회 최대 120분 |
| `reservation.checkinOpenBeforeMinutes` | 시작 몇 분 전부터 입실 번호가 맞나 (10, FR-ACC-001) |
| `reservation.noShowMinutes` · `instantNoShowMinutes` | 입실 기한: 미리 예약 15분, 지금 바로 쓰기 10분 |
| `reservation.extension` | 연장하기(FR-RSV-105): 1회, 종료 10분 전부터, 최대 120분, 끝난 뒤 30분 안의 새 예약은 이어진 사용 |
| `conduct[]` | 이용 수칙. `key`는 `NOISE` `FOOD` `TIDY` `NO_SHOW` `AWAY` `PHOTO` `DOOR` `BELONGINGS` (8개), `title`·`rule`·`penalty`는 화면에 그대로 보여 줄 문장 |
| `reminders.checkin` · `checkout` | 체크인 직후, 퇴실 확인 창에 보여 줄 한 줄 |

- 내 경고 횟수("내 경고 0회")는 이 응답에 없어요. 로그인한 사람마다 다른 값이라 신고 판정 쪽 API에서 따로 줘요(FR-RPT-104, 아직 없음).

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

## 7. 비서 (`/api/agent`) — 로그인 필요, assistant `develop`

> 자세한 입출력·오류는 비서 저장소 [도구 명세](https://github.com/AI-go-do/mspace-assistant/blob/develop/docs/도구-명세.md)·[설계](https://github.com/AI-go-do/mspace-assistant/blob/develop/docs/설계.md)가 기준이에요. 여기는 화면이 부르는 모양만 요약해요.

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
    "toolCalls": 2, "stoppedByLimit": false,
    "tools": ["find_available_rooms"], "hints": [], "suggestions": ["2시에 1번 방 2시간 잡아 줘"]
  },
  "error": null
}
```
실패하면 `200 { "status": "FAILED", "result": null, "error": { "code": "LLM_RATE_LIMITED", "message": "..." } }`. `error.message`를 그대로 보여 주면 돼요.

쓰기(예약·지금 바로 쓰기·취소·퇴실·연장)는 바로 하지 않고 확인 카드(`confirmations`)로 돌려줘요. 입실 번호는 모델에 보내지 않아서 `accessCodeCards`에는 예약 id만 있어요.

- `tools`: 이번 답에서 부른 도구 이름 (성공률 측정용)
- `hints`: 화면 신호. `OPEN_SCHEDULE`이면 시간표가 없어서 답한 것 → 답 아래 [내 시간표 넣기] 버튼
- `suggestions`: 답 아래 빠른 답장 버튼 글(최대 3개). 누르면 그 글을 그대로 채팅으로 보내요
- 대화 맥락은 서버가 학생별로 24시간 기억해요(Redis). 화면이 `history`를 보내지 않아도 이어져요

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

### `DELETE /api/agent/chat/history` 새 대화

서버가 기억한 앞 대화를 지워요 → `204`. 화면도 자기 대화 목록을 비워요. 그 학생의 채팅이 아직 도는 중이면 `409 CHAT_IN_PROGRESS`(끝날 때 옛 대화를 다시 저장하니까), Redis에 닿지 못하면 `503 MEMORY_UNAVAILABLE`.

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

### `GET`·`PUT`·`DELETE /api/agent/schedule` 내 시간표 (FR-AGT-102a)

학생이 화면에서 직접 넣는 시간표예요. 모델은 고치지 못하고 읽기 도구 `my_schedule`·`week_plan`으로만 봐요. 저장은 비서 Redis(`agent:schedule:<학생 키>`, TTL 없음).

```json
PUT { "classes": [ { "dayOfWeek": "MON", "startTime": "09:00", "endTime": "10:15", "name": "자료구조" } ] }
→ 200 { "classes": [ ... ], "updatedAt": "2026-11-06T10:00:00+09:00" }
```
`PUT`은 통째로 바꾸기. `GET`은 같은 모양(없으면 `404 NO_SCHEDULE`), `DELETE`는 `204`.

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 요일 `MON`~`SUN`, `HH:mm`, 시작 < 끝, 과목명 1~30자, 1~40칸이 아님 |
| `SCHEDULE_OVERLAP` | 400 | 같은 요일에 시간이 겹침 |
| `NO_SCHEDULE` | 404 | 넣은 시간표 없음 (`GET`) |
| `SCHEDULE_UNAVAILABLE` · `TOOLS_UNAVAILABLE` | 503 | Redis · mcp-server에 닿지 못함 |

### `POST /api/agent/schedule/parse` 시간표 캡처 읽기

`multipart/form-data`의 `image` 한 장(PNG·JPEG·WebP, 5MB까지) → `200 { "classes": [ ... ], "dropped": 1, "note": "..." }`. Gemini가 읽은 **초안**만 주고 저장하지 않아요. `classes`는 `PUT` 본문과 같은 모양이라 확인 카드에서 고친 뒤 그대로 `PUT`해요. 하나도 못 읽으면 `classes: []`(오류 아님). 사진이 없거나 형식·크기가 틀리면 `400 INVALID_IMAGE`, 모델 한도·장애는 `429 LLM_RATE_LIMITED`·`503 LLM_UNAVAILABLE`, 그림을 못 읽는 모델(Ollama)이면 `503 LLM_NOT_CONFIGURED`. 캡처에 이름·학번이 보이면 외부 AI로 나가니 잘라서 올리게 안내해요([개인정보 처리방침](../privacy/README.md)).

### `GET /api/agent/briefing` 오늘 브리핑

앱을 열 때 1번 불러요. 서버가 `my_schedule`(오늘)·`my_reservations`·`room_status`를 직접 부르고(모델 안 거침) 한 문단 `text`로 묶어요. [듣기]는 이 `text`를 브라우저 `speechSynthesis`로 읽어요.

```json
{ "date": "2026-11-06", "dayOfWeek": "금", "schedule": { "classes": [ ... ], "lastClassEnd": "14:50", "note": null },
  "afterClass": { "roomName": "2번 방", "start": "15:30", "end": "17:30", "suggestion": "15:30 2번 방 2시간 잡아 줘", "note": null },
  "reservations": { "items": [], "note": null }, "rooms": { "availableNames": ["2번 방"], "earliestLimit": "09:30", "note": null },
  "text": "11월 6일 금요일 브리핑이에요. ...", "hints": [], "suggestions": ["15:30 2번 방 2시간 잡아 줘"] }
```
- 도구 하나가 실패하면 그 칸만 `null` + `note`, 나머지는 그대로. 시간표가 없으면 `hints: ["OPEN_SCHEDULE"]`
- `afterClass`: 오늘 수업이 있고 대표자 예약이 없을 때만 수업 뒤 가장 이른 칸 하나. `suggestions` 버튼을 누르면 채팅으로 보내 확인 카드까지 가요
- `Cache-Control: no-store`. 오류는 `401 UNAUTHORIZED`만

## 8. 코드와 문서가 아직 다른 곳 (11/6 점검)

11/3~6에 적었던 줄(알림 읽음 #69, 판정 벌칙 #70, 강제 퇴실 #73, 연장 #51, 신고 #52, 노쇼 벌칙 #58, `created_via` #45 + 비서 #54, 자리 비움 `AWAY_RELEASED`, 센서 #60·게이트웨이 공개 경로, 비서 API)은 모두 develop에 머지돼 지웠어요.

| 무엇 | 지금 | 할 일 |
|---|---|---|
| springdoc `/swagger-ui` | 컨벤션에만 있고 core에 의존성 없음 | core에 추가하면 이 문서는 요약본으로 줄이기 |
| 센서용 장치 토큰 | 키패드와 같은 토큰 하나 (6-2장 "정할 것") | 장치마다 따로 둘지 정하기(지영·재웅). 11/6 2번 방 키패드 토큰 추가 예정 |
| 입실 방식 NFC · QR | 8차 회의(11/6): 시험용 방 1실에서 키패드 → NFC → QR. 지금 API는 키패드(6장)만 | 요구사항 Q-16이 정해지면 6장에 방식별 모양 추가(지영·수연) |
| 관리자 통계·경고 목록·증거 사진 | core #74~#76 PR 대기 (5장 "관리자 화면용 API") | 머지되면 0장 상태 ✅로, 응답 예시 추가 |
