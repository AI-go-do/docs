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
| 화면 | `GET /api/members/me` | 내 정보 (학번·이름, 마이 화면) | 11/9 지영 | ✅ develop (core #102) |
| 화면 | `GET /api/friends` | 친구 창: 친구·받은 요청·보낸 요청 | FR-RSV-004 | ✅ develop |
| 화면 | `POST /api/friends/requests` | 학번으로 친구 요청 | FR-RSV-004 | ✅ develop |
| 화면 | `POST /api/friends/requests/{id}/accept`·`decline`·`cancel` | 수락·거절(받은 사람), 취소(보낸 사람) | FR-RSV-004 | ✅ develop |
| 화면 | `DELETE /api/friends/{studentId}` | 친구 삭제 | FR-RSV-004 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations` | 예약 만들기 | FR-RSV-001~005 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/instant` | 지금 바로 쓰기 (누른 시각부터) | FR-RSV-103 | ✅ develop |
| 화면 · 비서 | `GET /api/reservations/me` | 내 예약 | FR-RSV-006 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/cancel` | 시작 전 취소 (즉시 시작은 누른 뒤 5분 안) | FR-RSV-007 | ✅ develop · 즉시 시작 예외 core #84 |
| 화면 · 비서 | `POST /api/reservations/{id}/checkout` | 퇴실 | FR-RSV-008 | ✅ develop |
| 화면 · 비서 | `POST /api/reservations/{id}/continue` | 연장하기 (사용 중 이어 쓰기, 1회) | FR-RSV-105 | ✅ develop |
| 화면 | `GET /api/relay/waits/me`, `POST /api/relay/waits`, `POST /api/relay/waits/{id}/cancel` | 빈방 릴레이: 차 있는 칸 기다리기 (1인 3개) | Q-18 | ✅ develop (core #101) |
| 화면 | `GET /api/relay/offers/me`, `POST /api/relay/offers/{id}/decline` | 빈방 릴레이: 나에게 넘어온 자리, 넘길게요 | Q-18 | ✅ develop (core #101) |
| 화면 | `POST /api/reservations/{id}/still-here` | 아직 있어요 (자리 비움 타이머 다시, 예약당 1번) | FR-DEV-103 | ✅ develop |
| 관리자 | `GET /api/reservations/stats` | WEB/AGENT 예약 건수 | NFR-COM-008 | ✅ develop |
| 화면 | `POST /api/reports` | 신고 (그 방·옆 방) | FR-RPT-101 | ✅ develop |
| 관리자 | `GET /api/reports?verdict=OPEN` | 신고 목록 | FR-RPT-103 | ✅ develop |
| 관리자 | `POST /api/reports/{id}/judge` | 신고 판정 + 경고·정지·알림 | FR-RPT-103·104 | ✅ develop |
| 관리자 | `POST /api/reservations/{id}/force-checkout` | 강제 퇴실 (퇴출) | FR-RPT-104 | ✅ develop |
| 관리자 | `GET /api/reports/{id}/evidence` | 판정용 증거 사진 두 장 + 앞 예약 방 카메라 결과(`previous.tidyCheck`) | FR-WEB-101 | ✅ develop (core #76, #110) |
| 관리자 | `GET /api/tidy-checks?days=7` | 방 카메라 정리 확인 목록 (`phase`, 물건별 확신도) | FR-DEV-105 | ✅ develop (core #110, #112) |
| 방 카메라 | `POST /api/rooms/{roomId}/tidy-photos` | 정리 확인 사진 올리기 (`FIRST`·`RECHECK` 1장씩, 30일 보관) | FR-DEV-105 · Q-27 | ✅ develop (core #115) |
| 화면 · 관리자 | `GET /api/reservations/{id}/tidy-photos/{phase}` | 정리 확인 사진 보기 (그 예약 사람들·관리자) | FR-DEV-105 · Q-27 | ✅ develop (core #115) |
| 관리자 | `GET /api/penalties?days=30` | 경고·정지 목록 | FR-WEB-101 | ✅ develop |
| 관리자 | `GET`·`POST /api/admin/room-blocks` | 기간 막기 목록·만들기 (공사·점검·대관) | Q-23 | ✅ develop (core #98) |
| 관리자 | `POST /api/admin/room-blocks/{id}/release`, `GET …/conflicts`, `POST …/cancel-conflicts` | 풀기, 겹친 예약 보기, 겹친 예약 취소 + 알림 | Q-23 | ✅ develop (core #98) |
| 관리자 | `GET /api/stats/usage?from=&to=` | 이용 통계 (이용률·노쇼율·시간대·센서·알림 효과) | FR-WEB-101 | ✅ develop · 알림 효과 core #93 |
| 화면 | `GET /api/notifications/me` | 내 알림함 (최근 50개, 읽음 여부) | FR-NTF-103 | ✅ develop |
| 화면 | `GET /api/notifications/me/unread-count` | 안 읽은 알림 개수 (종 아이콘 점) | FR-NTF-103 | ✅ develop |
| 화면 | `POST /api/notifications/{id}/read` | 알림 하나 읽음 | FR-NTF-103 | ✅ develop |
| 화면 | `POST /api/notifications/read-all` | 내 알림 모두 읽음 | FR-NTF-103 | ✅ develop |
| 화면 | `GET /api/room-watches/me` | 오늘 켜 둔 빈방 알림 | 11/9 지영 | ✅ develop (core #109) |
| 화면 | `POST /api/room-watches`, `POST /api/room-watches/{roomId}/cancel` | 빈방 알림 켜기·끄기 (그 방이 비면 알림 한 번) | 11/9 지영 | ✅ develop (core #109) |
| QR | `POST /api/access/verify` | 입실 QR 안 번호 확인 → 문 열기 (11/10부터 QR만, 키패드·NFC 뺌). 찾으러 온 QR이면 `purpose: "RETRIEVE"` | FR-ACC-001 | ✅ develop · `purpose` core #114 |
| 화면 | `POST /api/reservations/{id}/retrieve-pass` | "찾으러 갈게요": 두고 간 물건 가지러 들어가는 10분짜리 QR | FR-DEV-105 | ✅ develop (core #114) |
| 재실·CO2 센서 | `POST /api/sensor-events` | 방에 사람이 있는지, CO2 값 알리기 (CO2에 온도·습도 선택) | FR-DEV-101~104 | ✅ develop · 온도·습도 core #97 |
| 관리자 | `GET /api/admin/rooms/environment?minutes=60` | "지금" 탭: 방별 CO2·온도·습도·재실 + 지난 N분 1분 값 | Q-21 | ✅ develop (core #97, #106) |
| QR | `POST /api/device-faults` | 장치 오류 구간·그동안 읽힌 번호 알리기 → 장치 탓 노쇼 자동 무르기 | Q-19 | ✅ develop (core #99) |
| 화면 | `GET /api/rules` | 이용 규칙 | FR-WEB-003 | ✅ develop |
| 화면 | `GET /api/penalties/me` | 내 경고 횟수·정지 ("내 경고 0회") | FR-WEB-003 · FR-RPT-104 | ✅ develop (core #92) |
| 화면 | `GET /api/penalties/me/warnings` | 내 경고 목록 + 철회 요청 상태 (내 예약 "경고 철회 요청 (N일 남음)") | FR-RPT-105 | ✅ develop (core #116) |
| 화면 | `POST /api/warnings/{id}/appeal` | 경고 철회 요청 (받은 뒤 7일 안, 1번) → 판단기가 다시 비교 | FR-RPT-105 | ✅ develop (core #116) |
| 화면 | `POST /api/agent/chat` | 비서에게 말 보내기 (접수, `202`) | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `GET /api/agent/chat/{jobId}` | 비서 답 조회 (폴링) | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `DELETE /api/agent/chat/history` | 새 대화 (서버가 기억한 대화 지우기) | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `POST /api/agent/confirmations/{token}` | 확인 카드 [확인] | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `GET /api/agent/reservations/{id}/access-code` | 비서 입실 번호 카드 | FR-AGT-101 | ✅ develop (assistant) |
| 화면 | `GET`·`PUT`·`DELETE /api/agent/schedule` | 내 시간표 보기·통째로 바꾸기·지우기 | FR-AGT-102a | ✅ develop (assistant) |
| 화면 | `POST /api/agent/schedule/parse` | 시간표 캡처 읽기 (초안만, 저장 안 함) | FR-AGT-102a | ✅ develop (assistant) |
| 화면 | `GET /api/agent/briefing` | 오늘 브리핑 (시간표·예약·빈 방·수업 뒤 칸, 모델 안 거침) | FR-AGT-102b·103 | ✅ develop (assistant) |
| 화면 | `GET`·`PUT /api/agent/persona` | 비서 성향 보기·고르기 (조용히·알아서·잔소리) | 8차 회의(요구사항 ID 대기) | ✅ develop (assistant #85) |

`/api/agent/`로 시작하면 API 게이트웨이가 비서 서버(agent)로, 나머지 `/api/`는 core로 보내요. 게이트웨이가 로그인 토큰을 먼저 확인해요([아키텍처](../architecture/README.md)).

## 1. 공통 규칙

| 항목 | 규칙 |
|---|---|
| 로그인 | `Authorization: Bearer <accessToken>`. 없거나 틀리면 `401 UNAUTHORIZED` |
| 개발용 임시 헤더 | `X-User-Id: <회원 id>`. core를 `MSPACE_DEV_HEADER=true`로 띄웠을 때만 받아요. 운영에서는 꺼요. 토큰이 있으면 토큰만 봐요 |
| 만든 곳 | 비서는 예약 만들기(`POST /api/reservations`, `/instant`)에 `X-Created-Via: AGENT`를 붙여요. 그러면 `created_via = AGENT`, 없으면 `WEB`. 대소문자와 앞뒤 공백은 무시해요(` agent `도 AGENT). `WEB`·`AGENT` 말고 다른 값이면 `400 INVALID_REQUEST`. 건수를 따로 세는 표시일 뿐 권한과는 상관없어요 (core #45) |
| 장치 | QR 장치·센서는 사람 토큰이 아니라 장치 토큰(`Authorization: Bearer <장치 토큰>`)을 써요. 토큰은 장치마다 따로 두고 방·장치 이름에 묶여요(core #91, 아래 6장) |
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
{ "email": "a@chosun.ac.kr", "code": "123456", "name": "테스트", "studentId": "20230001" }
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
      "occupiedNow": false, "blockedNow": null,
      "slots": [ { "startAt": "2026-11-03T14:00:00+09:00", "endAt": "2026-11-03T14:30:00+09:00", "available": true,
                   "block": null } ] }
  ]
}
```

- 관리자가 막은 칸(Q-23)은 `available: false`에 `block: { "reason": "INSPECTION", "memo": "에어컨 점검", "endAt": "…" }`이 붙어요. 지금 막혀 있는 방은 `blockedNow`에 같은 모양, `occupiedNow`도 `true` (core #103)
- 방 현황 `GET /api/rooms/status`도 걸린 예약 없이 막혀 있으면 `status: "BLOCKED"`와 `block`이 오고, `availableCount`에서 빠져요 (core #103)

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

### `GET /api/members/me` 내 정보 — 로그인 필요 (core #102)

```json
{ "studentId": "20230001", "name": "홍길동" }
```

마이 화면에 쓰는 내 학번·이름이에요. 이메일·회원 번호는 화면에 안 써서 내보내지 않아요. `Cache-Control: no-store`라 브라우저에 남지 않아요. 회원이 없으면 `404 MEMBER_NOT_FOUND`.

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
| `ROOM_BLOCKED` | 409 | 관리자가 막아 둔 칸 (Q-23, core #98). 메시지 예: "점검 중이라 3/5 14:00까지 예약할 수 없습니다". 즉시 시작·연장도 같아요 |
| `OWNER_HAS_ACTIVE_RESERVATION` | 409 | 내가 대표자나 참여자로 들어간 끝나지 않은 예약이 이미 있음 (FR-RSV-003, 11/5) |
| `PARTICIPANT_HAS_ACTIVE_RESERVATION` | 409 | 참여자 중 끝나지 않은 예약이 이미 있는 사람이 있음 (누구인지는 안 알려 줌, core #64) |
| `PARTICIPANT_TIME_TAKEN` | 409 | 참여자가 같은 시간 다른 방 예약 중 |
| `IDEMPOTENCY_KEY_REUSED` | 409 | 같은 키를 다른 내용에 씀 |
| `RELAY_HELD` | 409 | 빈방 릴레이로 기다리던 학생에게 5분 동안 넘겨 둔 자리 (Q-18). 지금 바로 쓰기·연장하기도 같음 |
| `RELAY_OFFER_INVALID` | 409 | `relayOfferId`가 내 것이 아님·5분 지남·이미 받음·방이 다름·넘어온 구간 밖 |
| `RESERVATION_BUSY` | 503 | 동시 요청이 몰림 (드묾). 잠시 후 다시 |

빈방 릴레이로 나에게 넘어온 자리를 받을 때는 본문에 `"relayOfferId": 7`을 더 붙여요([5-3](#5-3-빈방-릴레이--모두-로그인-필요-q-18)). 넘어온 구간 안이면 일부만 잡아도 돼요. 참여자·인원·친구·정지·대표자 1건 규칙은 보통 예약과 같아요.

만들기 직전에 지난 예약 정리(노쇼·자동 퇴실)를 한 번 돌려요. 그래서 입실 마감이 지난 예약 때문에 `409`가 나지 않아요.

### `POST /api/reservations/instant` 지금 바로 쓰기

시작은 서버가 정해요(누른 시각, 초 아래 버림). 끝만 보내요. `Idempotency-Key`도 똑같이 받아요(같은 키로 다시 보내면 시작 시각이 달라도 처음 예약을 `200`).

```json
{ "roomId": 1, "endAt": "2026-11-03T16:30:00+09:00", "participantStudentIds": ["20230002", "20230003"] }
```
→ `201`, 응답 모양은 `POST /api/reservations`와 같아요.

- 끝은 30분 칸이고, 최대 (지금 + 2시간)을 30분 칸으로 올린 시각까지예요. 14:17에 누르면 16:30까지 고를 수 있어요.
- 다음 예약이 있으면 그 전까지만 돼요(아니면 `409 RESERVATION_TIME_TAKEN`).
- 10분 안에 QR로 입실하지 않으면 노쇼예요.
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
    "qrText": "MSPACE:1:482913", "cancelableUntil": "2026-11-03T14:00:00+09:00" }
]
```
`cancelableUntil`(core #84)은 이 시각 **전까지** 대표자가 취소할 수 있다는 뜻이에요. 미리 예약은 시작 시각, 지금 바로 쓰기는 누른 뒤 5분(설정값, 입실 마감을 넘지 않음). 화면은 `owner`이고 `BOOKED`이고 지금이 이 시각 전일 때만 취소 버튼을 보여요.
`accessCode`는 대표자이고 끝나지 않은 예약(`BOOKED`·`IN_USE`)일 때만 값이 있어요. 참여자이거나, 퇴실·자동 퇴실·자리 비움 퇴실·취소·노쇼된 예약이면 `null`이에요.

`qrText`는 폰에 띄울 **입실 QR 안의 글자**예요(11/6, core #71). 형식은 `MSPACE:<방 번호>:<입실 번호>`이고, `accessCode`가 `null`이면 `qrText`도 `null`이에요. 예약 만들기·지금 바로 쓰기·연장하기 응답에도 있어요. core는 QR 그림을 만들지 않고, 화면이 이 글자로 QR을 그려요. 문 앞 웹캠(mspace-device `qr_door.py`)이 읽어서 키패드와 같은 `POST /api/access/verify`로 물어요(`deviceId` `qr-<방>`). 그래서 열림 규칙과 5번 틀리면 1분 잠금(방 단위)도 키패드와 같아요.

예약 상태(`status`): `BOOKED` 예약됨 → `IN_USE` 입실 → `CHECKED_OUT` 퇴실 / `AUTO_CHECKED_OUT` 시간 끝나 자동 퇴실 / `AWAY_RELEASED` 자리 비움 자동 퇴실(재실 센서 15분 비어 있음, v0.2 FR-DEV-103) / `NO_SHOW` 입실 마감까지 안 옴(미리 예약은 시작 + 15분, 지금 바로 쓰기는 누른 뒤 10분. 끝 시각이 먼저 오면 끝 시각) / `CANCELED` 취소. 노쇼·자동 퇴실은 core가 1분마다 바꿔요.

### `POST /api/reservations/{id}/cancel` 시작 전 취소

미리 예약은 시작 전까지, 지금 바로 쓰기는 누른 뒤 5분 안이고 입실 전일 때 취소돼요(11/6 팀 결정, core #84).

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
- **어질러짐(`MESSY`)만 더 좁게 받아요(11/8, core #96):** 자기 방만, 입실 뒤 5분 안에만, 그 예약에 입실 사진(`CHECK_IN`)이 있어야 해요. 연장으로 생긴 예약은 안 돼요. 화면은 어질러짐을 고르면 입실 사진을 먼저 올리게 안내해요

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | `roomId`·`category` 빠짐, 없는 종류, 메모 500자 넘음 |
| `REPORT_NOISE_NOT_CONFIRMED` | 400 | 소음인데 `noiseConfirmed`가 `true`가 아님 |
| `REPORT_NOT_ALLOWED` | 403 | 지금 그 방이나 옆 방을 사용 중인 예약자가 아님 (입실 전·퇴실 후 포함) |
| `REPORT_MESSY_OWN_ROOM_ONLY` | 403 | 어질러짐인데 옆 방을 신고함 |
| `REPORT_MESSY_TOO_LATE` | 409 | 어질러짐인데 입실 뒤 5분이 지남, 또는 연장으로 생긴 예약 |
| `REPORT_CHECKIN_PHOTO_REQUIRED` | 409 | 어질러짐인데 그 예약에 입실 사진이 없음 |
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

### 관리자 화면용 API

모두 관리자만(`403 ADMIN_ONLY`). 관리자 화면이라 학번·이름을 가리지 않고 그대로 보여 줘요.

| 주소 | 상태 | 하는 일 |
|---|---|---|
| `GET /api/penalties?days=30` | ✅ develop (core #75) | 경고·정지 목록 |
| `GET /api/stats/usage?from=&to=` | ✅ develop (core #74, 알림 효과 #93) | 이용 통계 + 센서 값 + 알림 효과 |
| `GET /api/reports/{id}/evidence` | ✅ develop (core #76) | 직전 이용 예약의 퇴실(`CHECK_OUT`) 사진, 신고자 예약의 입실(`CHECK_IN`) 사진. 사진마다 `exists`, 있으면 `url`(사진 보기 주소), 없으면 `note`(안 올림·30일 지나 지움·직전 예약 없음). 방 카메라 자동 신고는 `reporter: null`. `previous.tidyCheck`는 그 예약이 끝난 뒤 방 카메라 첫 확인(`FIRST`) 결과 `{result, items, leftItems, observedAt}`, 없으면 `null` (core #110, #112) |

#### 기간 막기 `/api/admin/room-blocks` (Q-23, core #98)

공사·점검·대관으로 방을 일정 기간 예약 못 하게 막아요. 막힌 칸은 미리 예약·즉시 시작·연장이 모두 `409 ROOM_BLOCKED`예요.

`POST /api/admin/room-blocks` → `201`
```json
{ "roomId": 5, "startAt": "2026-11-10T09:00:00+09:00", "endAt": "2026-11-10T12:00:00+09:00",
  "reason": "INSPECTION", "memo": "에어컨 점검" }
```
```json
{ "block": { "id": 1, "roomId": 5, "startAt": "2026-11-10T09:00:00+09:00", "endAt": "2026-11-10T12:00:00+09:00",
             "reason": "INSPECTION", "memo": "에어컨 점검", "createdAt": "…", "releasedAt": null },
  "conflicts": [{ "reservationId": 31, "roomId": 5, "startAt": "…", "endAt": "…", "people": 6,
                  "status": "BOOKED", "cancelable": true }] }
```

| 주소 | 하는 일 |
|---|---|
| `GET /api/admin/room-blocks` | 안 끝났거나 끝난 지 30일 안인 막기, 시작 빠른 순 (풀린 것도 `releasedAt`과 함께) |
| `POST /api/admin/room-blocks/{id}/release` | 풀기 → `block`. 이미 풀렸으면 그대로 `200` |
| `GET /api/admin/room-blocks/{id}/conflicts` | 겹친 끝나지 않은 예약 (풀렸거나 끝난 막기면 `[]`) |
| `POST /api/admin/room-blocks/{id}/cancel-conflicts` | 겹친 예약 중 시작 전(`cancelable: true`)만 취소 + 대표자·참여자 알림(`ROOM_BLOCKED`) → `{ "canceled": 1, "reservationIds": [31] }` |

- `reason`: `CONSTRUCTION` 공사 · `INSPECTION` 점검 · `RENTAL` 대관 · `OTHER` 기타
- 이미 잡힌 예약은 막을 때 지우지 않아요. 관리자가 목록을 보고 취소를 눌러야 취소돼요. 사용 중인 예약은 `cancelable: false`라 필요하면 강제 퇴실로
- `memo`는 학생에게도 보여서 이름·학번은 적지 않아요

| 오류 | 상태 | 언제 |
|---|---|---|
| `ROOM_BLOCK_INVALID` | 400 | 30분 칸이 아님, 끝 ≤ 시작, 이미 지난 기간, 31일 넘음, 메모 100자 넘음 |
| `ADMIN_ONLY` | 403 | 관리자가 아님 |
| `ROOM_NOT_FOUND` | 404 | 없는 방 |
| `ROOM_BLOCK_NOT_FOUND` | 404 | 없는 막기 |
| `ROOM_BLOCK_OVERLAP` | 409 | 같은 방의 풀리지 않은 막기와 겹침 |

#### `GET /api/penalties?days=30` 경고·정지 목록

- `days`: 1–365, 빼면 30. 틀리면 `400`
- 경고는 최근 `days`일 안에 받은 것, 정지는 그 사이 끝났거나 아직 진행 중인 것. 둘 다 최근 것부터

```json
{ "days": 30,
  "warnings": [{ "id": 3, "member": { "id": 12, "studentId": "20240001", "name": "테스트" },
                 "reason": "VIOLATION", "reportId": 7, "reservationId": 11,
                 "createdAt": "2026-11-06T14:00:00+09:00" }],
  "suspensions": [{ "scope": "USE", "id": 1, "member": { "id": 12, "studentId": "20240001", "name": "테스트" },
                    "reason": "WARNINGS", "startsAt": "2026-11-06T14:00:00+09:00",
                    "endsAt": "2026-11-20T14:00:00+09:00", "active": true }] }
```

| 칸 | 값 |
|---|---|
| `warnings[].reason` | `VIOLATION`(사실 판정, 이용한 방) · `FALSE_REPORT`(허위 신고) |
| `warnings[].reservationId` | 경고 근거 예약(이용한 방). 허위 신고 경고는 `null` |
| `suspensions[].scope` | `USE`(예약·참여 정지) · `REPORT`(신고만 정지) |
| `suspensions[].reason` | `NO_SHOW`(노쇼 누적) · `WARNINGS`(위반 경고 누적) · `FALSE_REPORT`(허위 신고 경고 누적) |
| `suspensions[].active` | 지금 진행 중이면 `true` |

#### `GET /api/stats/usage?from=2026-11-01&to=2026-11-30` 이용 통계

- 날짜는 서울 기준, `to` 포함. 둘 다 빼면 오늘까지 7일. 92일 넘거나 `from`이 `to`보다 늦으면 `400`
- `total`은 전체(`roomId`·`roomName`은 `null`), `rooms[]`는 방별, `byHour[]`는 서울 시작 시각 기준 0–23시(예약은 취소 뺀 수, `used`는 입실한 수, 예약 없는 시간은 빠짐)

```json
{ "from": "2026-11-01", "to": "2026-11-07", "days": 7,
  "total": { "roomId": null, "roomName": null,
             "reservations": { "total": 40, "canceled": 3, "pending": 2, "noShow": 4, "used": 31, "forced": 1,
                               "awayReleased": 2, "autoCheckedOut": 10 },
             "rates": { "noShow": 0.114, "actualUse": 0.886, "utilization": 0.231 },
             "minutes": { "booked": 3600, "used": 2900, "avgUsed": 94 },
             "shortUses": 5, "uniqueUsers": 18,
             "sensor": { "co2Readings": 1200, "co2AvgPpm": 720, "co2MaxPpm": 1350, "co2OverRate": 0.08,
                         "thresholdPpm": 1000, "ventilationAlerts": 3,
                         "occupiedMinutes": 2800, "occupiedWithoutReservationMinutes": 40 },
             "alerts": { "noShowWarned": 6, "noShowCheckedIn": 4, "noShowCheckedInRate": 0.667,
                         "ventilationChecked": 3, "ventilationResolved": 2, "ventilationResolvedRate": 0.667,
                         "resolvedWithinMinutes": 15 } },
  "rooms": [{ "roomId": 1, "roomName": "창작실 1", "...": "total과 같은 모양" }],
  "byHour": [{ "hour": 14, "reservations": 6, "used": 5 }] }
```

| 칸 | 뜻 |
|---|---|
| `reservations.total` | 시작 시각이 기간 안인 예약 (취소 포함) |
| `reservations.pending` | 아직 시작 안 했거나 입실 기다리는 예약 (`BOOKED`) |
| `reservations.used` | 입실한 예약 (사용 중 포함). `forced`는 그중 관리자 강제 퇴실 |
| `rates.*` | 0–1, 소수 셋째 자리. 분모가 0이면 `null`. `noShow`·`actualUse` = 노쇼·입실 / (취소·대기 뺀 예약), `utilization` = 실제 이용 시간 / 기간 중 지난 시간(하루 24시간) |
| `minutes` | `booked` 취소 뺀 예약 시간 합, `used` 입실 ~ 퇴실(사용 중이면 지금) 합, `avgUsed` = used / 입실 예약 수 |
| `shortUses` | 예약 시간의 절반도 안 쓰고 끝난 예약 수 |
| `uniqueUsers` | 기간 안에 입실한 예약의 서로 다른 회원 수 (대표자·참여자 모두) |
| `sensor.*` | CO2 평균·최고·기준(`thresholdPpm`) 넘은 비율, 환기 알림 수. 재실 분은 센서가 바뀔 때만 기록해서, 바뀐 기록 사이를 그 상태로 본 근삿값 |
| `sensor.occupiedWithoutReservationMinutes` | 예약 없이 사람이 있던 시간 (무단 사용 의심) |
| `alerts.noShowWarned` · `noShowCheckedIn` | 노쇼 직전 알림(`NO_SHOW_SOON`)을 받은 예약 수(시작 시각이 기간 안), 그중 입실한 수. 아직 입실 마감 전이라 결과가 안 난 예약은 빼요 |
| `alerts.ventilationChecked` · `ventilationResolved` | 환기 알림(`VENTILATE`, 같은 방·같은 시각은 한 번) 수, 그중 알림 뒤 `resolvedWithinMinutes`분 안에 CO2가 기준(`thresholdPpm`) 이하로 내려간 수. 알림 뒤 그 시간이 아직 안 지난 알림은 빼요 |
| `alerts.*Rate` | 입실 수 / 알림 받은 예약 수, 내려간 수 / 환기 알림 수. 0–1, 소수 셋째 자리, 분모가 0이면 `null` |
| `alerts.resolvedWithinMinutes` | 환기 결과를 보는 시간 (15분, 설정값 `mspace.sensor.co2.resolved-within-minutes`) |

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

## 5-3. 빈방 릴레이 — 모두 로그인 필요 (Q-18)

차 있는 30분 칸에 "기다리기"를 걸어 두면, 그 칸을 덮는 예약이 취소될 때 먼저 기다린 사람에게 놓인 구간을 넘겨요(11/9 지영 확정).

- 기다리기는 한 사람 동시에 **3개**까지. 칸마다 기다리는 팀 수는 제한 없고 선착순이에요
- 취소되면 줄 첫 사람에게 **5분** 동안 잡아 둬요. 알림함에 `kind: "RELAY_OFFER"`, 제목 "기다리던 자리가 났어요"
- 그 5분 동안은 넘겨받은 사람만 그 구간을 예약할 수 있어요(다른 사람은 `409 RELAY_HELD`)
- [넘길게요]를 누르거나 5분이 지나면 다음 사람에게 가요(5분 지남은 1분 정리가 처리)
- 받기는 `POST /api/reservations`에 `relayOfferId`를 붙여서. 받은 사람이 대표자이고 보통 예약 규칙 그대로예요. 하나를 받으면 내 다른 기다리기·넘김은 풀려요
- 시작 **30분 전**부터 취소된 자리는 넘기지 않고 그냥 빈 칸이에요. 기다리기도 시작 30분 전까지만 걸 수 있어요
- 받은 예약도 노쇼 규칙은 보통과 같아요
- 숫자 3·5·30은 설정값(`mspace.reservation.relay`)

`GET /api/relay/waits/me` 내 기다리기 (줄 서 있는 것, 시작 전) → `200`
```json
[{ "id": 3, "roomId": 1, "startAt": "2026-11-12T14:00:00+09:00", "order": 2 }]
```
`order`는 그 칸 줄에서 내 순서(1 = 맨 앞)예요.

`POST /api/relay/waits` 기다리기 걸기 → `200` (위 한 줄 모양. 같은 칸에 이미 걸었으면 그것을 그대로)
```json
{ "roomId": 1, "startAt": "2026-11-12T14:00:00+09:00" }
```

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 빠진 값, 칸 시작(정각·30분)이 아님 |
| `RESERVATION_OUT_OF_RANGE` | 400 | 7일 밖 |
| `ROOM_NOT_FOUND` | 404 | 없거나 쓸 수 없는 방 |
| `RELAY_SLOT_FREE` | 409 | 그 칸이 비어 있음(바로 예약하면 됨), 또는 내가 들어 있는 예약 |
| `RELAY_WAIT_LIMIT` | 409 | 기다리기가 이미 3개 |
| `RELAY_TOO_LATE` | 409 | 시작 30분 전부터 |

`POST /api/relay/waits/{id}/cancel` 그만 기다리기 → `204` (없거나 이미 끝났으면 `404 RELAY_WAIT_NOT_FOUND`)

`GET /api/relay/offers/me` 나에게 넘어온 자리 (5분이 남은 것) → `200`
```json
[{ "id": 7, "roomId": 1, "startAt": "2026-11-12T14:00:00+09:00", "endAt": "2026-11-12T15:00:00+09:00",
   "expiresAt": "2026-11-12T10:15:00+09:00", "order": 1 }]
```
화면은 `expiresAt`까지 남은 시간을 보여 주고, [받기]는 이 구간으로 예약 만들기 화면을 채워 `relayOfferId: 7`과 함께 보내요.

`POST /api/relay/offers/{id}/decline` 넘길게요 → `204` (내 것이 아니거나 이미 끝났으면 `404 RELAY_OFFER_NOT_FOUND`)

## 6. 출입 (QR)

> **11/10 재웅 결정: 입실은 QR로만 해요.** 키패드·NFC는 빼요. 장치는 QR에서 읽은 번호를 `code`로 보내요. 지금 core 코드는 `keypad-` 장치와 `KEYPAD`·`KEYPAD_DOWN` 값을 아직 받아요. 지울지는 요구사항 Q-29(지영·수연)에서 정해요.

### `POST /api/access/verify` — 장치 토큰 필요

```json
{ "eventId": "1-48213-18", "roomId": 1, "deviceId": "qr-1", "code": "123456" }
```
문 앞 웹캠 QR 입실(device #26)도 이 API를 그대로 불러요. 폰 화면의 입실 QR(`MSPACE:<방>:<6자리>`, 예약 응답 `qrText`)에서 6자리만 꺼내 `code`로 보내요. 다른 방 QR은 장치가 서버에 묻지 않고 거절해요.

`eventId`(1~64자)는 장치가 누를 때마다 새로 만드는 값이라, 같은 값이 다시 와도 한 번만 처리하고 처음 결과를 그대로 돌려줘요. `deviceId`는 50자 이하(선택).

→ 거절도 `200`이에요. 장치는 `result`만 보고 LED를 켜요.

```json
{ "result": "OPENED" }
{ "result": "OPENED", "purpose": "RETRIEVE" }
{ "result": "DENIED", "reason": "WRONG_CODE" }
```
`purpose`(core #114): "찾으러 갈게요" QR(`POST /api/reservations/{id}/retrieve-pass`)로 열렸을 때만 `"RETRIEVE"`, 보통 입실은 빠져요. 이때는 **체크인하지 않고**(예약 상태·노쇼 그대로), 틀린 횟수도 되돌리지 않아요(찾으러 온 QR로 잠금을 풀 수 없게).
`reason`: `WRONG_CODE` 틀림, `NOT_YET` 아직 이름(시작 10분 전부터 열림), `LOCKED` 5번 연속 틀려 1분 잠김.

`WRONG_CODE`가 나가는 경우는 이것도 포함해요.
- 6자리 숫자가 아닌 `code`
- 끝 시각이 지난 예약, 퇴실·자동 퇴실·취소·노쇼된 예약의 번호
- 아직 입실 안 했는데 입실 마감(미리 예약 시작 + 15분, 지금 바로 쓰기 누른 뒤 10분)이 지난 번호. 이미 입실한 사람은 끝 시각 전까지 같은 번호로 다시 열 수 있어요

| 오류 | 상태 | 언제 |
|---|---|---|
| `DEVICE_UNAUTHORIZED` | 401 | 장치 토큰이 없거나 틀림 |
| `DEVICE_ROOM_MISMATCH` | 403 | 토큰에 등록된 방과 `roomId`가 다름 |
| `DEVICE_ID_MISMATCH` | 403 | 토큰에 등록된 장치 이름과 `deviceId`가 다름 |
| `INVALID_REQUEST` | 400 | `eventId`·`roomId` 빠짐, 길이 넘음 |

장치 토큰 (core #91, 11/7): core `.env`의 `MSPACE_DEVICES`에 `deviceId:roomId:토큰`을 쉼표로 이어 적어요(예: `qr-1:1:<토큰>,qr-2:2:<토큰>`). 같은 방 장치끼리는 토큰 하나를 같이 써도 되고, 다른 방끼리는 안 돼요. 센서(6-2장)와 방 카메라(6-3장)도 같은 규칙이에요. 예전 공용 토큰 `MSPACE_DEVICE_TOKEN`은 장치를 하나씩 옮기는 동안만 모든 방에 받아요.

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

- 내 경고 횟수("내 경고 0회")는 이 응답에 없어요. 로그인한 사람마다 다른 값이라 아래 `GET /api/penalties/me`에서 따로 줘요.

### `GET /api/penalties/me` 내 경고 횟수 — 로그인 필요 (core #92)

이용 규칙 화면 위쪽 "내 경고 0회"(FR-WEB-003)에 쓰는 값이에요. 로그인한 본인 것만 줘요. 로그인 안 하면 `401`.
- 숫자는 판정 때 정지를 거는 셈과 같은 방식으로 세요(FR-RPT-104). 그래서 화면 숫자와 실제 정지가 어긋나지 않아요. 정지가 한 번 걸리면 그때까지의 경고는 빼고 다시 세요.

```json
200
{ "warnings": 1, "falseReportWarnings": 0, "limit": 3, "windowDays": 30,
  "suspendedUntil": null, "reportSuspendedUntil": null }
```

| 필드 | 뜻 |
|---|---|
| `warnings` | 최근 `windowDays`일 안 위반 경고(`VIOLATION`) 수 |
| `falseReportWarnings` | 최근 `windowDays`일 안 허위 신고 경고(`FALSE_REPORT`) 수 |
| `limit` · `windowDays` | 위반 경고·허위 신고 경고 각각 `windowDays`일 안 `limit`회면 정지 (3회 · 30일, 설정값) |
| `suspendedUntil` | 이용 정지(예약·참여, 노쇼 누적 포함) 중이면 끝나는 시각(`+09:00`), 아니면 `null` |
| `reportSuspendedUntil` | 신고 정지 중이면 끝나는 시각, 아니면 `null` |

### 경고 철회 요청 — 로그인 필요 (FR-RPT-105, ✅ develop core #116)

억울한 경고를 받은 학생이 7일 안에 철회를 요청하면 **판단기가 근거를 다시 비교**해 철회·유지를 정해요(11/10 재웅: 관리자는 정하지 않음). 세부 규칙은 요구사항 Q-24.

- 판단은 core의 `WarningReviewer` 포트 뒤에서 해요(A-6). 지금은 **규칙 판단기**(`rules`, 지시문 버전 `rules-v1`)가 1분마다 대기 중인 요청을 최대 10개씩 봐요(11/10 지영 "규칙 판단기 먼저"). Gemini 판단기는 다음 PR이고, 키가 없으면 규칙 판단기로 돌아요.
- 규칙 판단기 기준: 방 카메라 경고는 다시 확인(`RECHECK`) 때 가장 높은 확신도가 경고 기준(0.8, `mspace.tidy.warn-min-confidence`) 이상이면 유지, 아니면 철회. 사람 신고로 받은 경고는 지금 자동으로 다시 볼 수 없어서 철회해요(확인할 수 없으면 벌칙 없음).
- Q-24 ①–③(11/10 지영): ① 요청 중에도 경고 횟수에 세요. ② 철회되면 그 경고는 횟수에서 빠지고, 그 경고로 걸린 정지도 바로 풀려요. ③ 판단기가 유지라 해도 확신도가 0.8(`mspace.appeal.min-confidence`) 아래면 철회해요.

#### `GET /api/penalties/me/warnings?days=30` 내 경고 목록

본인 것만, 최근 것부터. `days`는 1–90(기본 30), 벗어나면 `400 INVALID_REQUEST`. 내 예약 화면은 `reservationId`로 지난 예약 카드에 버튼을 붙여요.

```json
200
[
  { "id": 41, "reason": "VIOLATION", "reportType": "MESSY", "roomId": 3,
    "reservationId": 812, "startAt": "2026-11-08T14:00:00+09:00", "endAt": "2026-11-08T16:00:00+09:00",
    "createdAt": "2026-11-08T19:12:00+09:00", "why": "퇴실 뒤 방 카메라가 다시 봤을 때도 두고 간 물건이 남아 있었어요",
    "appealableUntil": "2026-11-15T19:12:00+09:00",
    "appeal": null }
]
```

| 필드 | 뜻 |
|---|---|
| `reason` | `VIOLATION`(수칙 위반) · `FALSE_REPORT`(허위 신고) |
| `reportType` | 걸린 신고 종류 `MESSY`·`WINDOW_OPEN`·`BROKEN`·`NOISE` |
| `reservationId` | 경고가 걸린 예약. 허위 신고 경고는 신고할 때 쓰던 예약(없으면 `null`) |
| `startAt` · `endAt` | 그 예약 시간(예약이 없으면 `null`) |
| `why` | 경고 이유 한 줄. 방 카메라 "퇴실 뒤 방 카메라가 다시 봤을 때도 두고 간 물건이 남아 있었어요", 사람 신고 "이용하신 방의 ○○ 신고가 사실로 판정됐어요", 허위 신고 "○○ 신고가 허위로 판정됐어요" |
| `appealableUntil` | 요청 마감 = `createdAt` + 7일(`mspace.appeal.days`) |
| `appeal` | 아직 안 보냈으면 `null`. 보냈으면 `{ "status": "PENDING" \| "WITHDRAWN" \| "KEPT", "text", "createdAt", "decidedAt", "answer" }` (`answer` = 판단기 이유를 학생용 한 문단으로, 판단 전엔 `decidedAt`·`answer`가 `null`) |

#### `POST /api/warnings/{id}/appeal` 철회 요청 보내기

```json
{ "text": "퇴실할 때 컵은 다 버렸어요. 다시 봐 주세요.", "evidence": ["CHECKOUT_PHOTO"] }
```

- `201` → 위 목록의 한 줄(`appeal.status = PENDING`). 판단이 끝나면 `WITHDRAWN`·`KEPT`로 바뀌고 알림 `WARNING_APPEAL_RESULT`("경고가 철회됐어요" / "경고가 유지됐어요", 정지가 풀리면 본문에 함께 적어요)
- `text` 10–500자(이모지도 1자). 7자리 넘는 숫자(사이 공백·`-` 포함, 전화번호·학번)는 `400 INVALID_REQUEST` (나중에 외부 AI로 넘어갈 수 있어서)
- `evidence`(선택): 다시 봐 달라는 근거 `CHECKIN_PHOTO`·`CHECKOUT_PHOTO`·`TIDY_CHECK` 중에서. 다른 값이나 `null`이 섞이면 `400 INVALID_REQUEST`
- 없는 경고·남의 경고 `404 WARNING_NOT_FOUND`, 7일 지남 `409 WARNING_APPEAL_EXPIRED`, 이미 보냄 `409 WARNING_APPEAL_DUPLICATE`
- core는 판단마다 판단기 이름·지시문 버전·결정·확신도·쓴 근거를 남겨요(`warning_appeal` 표). 정답 세트로 정확도를 재서 표로 내려고요(요구사항 FR-RPT-105 "신뢰도 기록")

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
  "kind": "CO2", "ppm": 1080, "source": "scd40", "temperatureC": 25.4, "humidityPct": 47,
  "observedAt": "2026-11-05T14:06:00+09:00" }
```

| 필드 | 형식 | 뜻 |
|---|---|---|
| `eventId` | 글자 1~64자 (UUID) | 장치가 이벤트마다 새로 만들어요. 다시 보낼 때만 같은 값이라, 같은 값은 한 번만 저장해요 |
| `roomId` · `deviceId` | 정수 · 글자 50자 이하 | 방 번호, 보낸 장치(`sensor-방번호`) |
| `kind` | `OCCUPANCY` 또는 `CO2` | 재실 / CO2 |
| `occupied` | 참/거짓 (재실) | 장치가 센서들을 합친 판단. 하나라도 "있음"이면 `true` (FR-DEV-103) |
| `sources` | 객체, 선택 (재실) | 센서별 값(`pir`, `radar`, `light`). core는 판단에 안 써요. 재실이 바뀐 이벤트는 기록하고, `light`(0~1023 숫자)·`pir`(참/거짓)의 최신 값은 매번 관리자 "지금" 탭에 보여 줘요(서버 메모리, core #106). 카메라 방은 카메라 재실 이벤트에 실려 와요 |
| `ppm` · `source` | 정수 0~10000 · 글자, 선택 (CO2) | CO2 농도, 센서 이름 |
| `temperatureC` · `humidityPct` | 숫자, 선택 (CO2만) | SCD40이 같이 재는 온도(-40~85℃)·습도(0~100%). 소수 한 자리로 저장. 안 보내도 돼요(관리자 "지금" 탭, Q-21) |
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
| `INVALID_REQUEST` | 400 | 빠진 값, 없는 방 번호(`roomId`), 모르는 `kind`, `ppm`·온도·습도 범위 밖, 재실 이벤트에 온도·습도, 오프셋 없는 시각, 5분 넘게 미래인 시각 | 그 이벤트만 버려요 |
| `DEVICE_UNAUTHORIZED` | 401 | 장치 토큰 없음·틀림 | 이벤트를 모아 둔 채 간격을 늘려 다시 보내요(설정 문제라 고치면 이어서 보냄) |
| `DEVICE_ROOM_MISMATCH` · `DEVICE_ID_MISMATCH` | 403 | 토큰에 등록된 방·장치와 `roomId`·`deviceId`가 다름 (6장 "장치 토큰") | 401과 같음 (설정 실수) |
| `5xx`, 연결 실패 | - | 서버 문제 | 같은 `eventId`로 다시 보내요 (1→2→4…30초) |

정할 것:
- API 게이트웨이: 장치 토큰은 JWT가 아니라서 `/api/sensor-events`를 `/api/access/verify`처럼 공개 경로로 넣기 (재웅)

### `GET /api/admin/rooms/environment?minutes=60` 관리자 "지금" 탭 — 관리자만

관리자 화면 "지금" 탭(Q-21)이 10초마다 불러요. 방마다 지금 값과 지난 `minutes`분(기본 60, 1~180)의 1분 평균을 줘요. 관리자가 아니면 `403 ADMIN_ONLY`, `minutes` 범위 밖이면 `400 INVALID_REQUEST`.

```json
{ "now": "2026-11-09T14:41:10+09:00", "minutes": 60,
  "rooms": [
    { "roomId": 1, "roomName": "창작실 1",
      "presence": "OCCUPIED", "lastSignalAt": "2026-11-09T14:41:05+09:00", "offline": false,
      "co2": { "ppm": 915, "at": "2026-11-09T14:40:30+09:00" },
      "temperatureC": 25.4, "humidityPct": 47.0,
      "light": { "raw": 512, "at": "2026-11-09T14:41:05+09:00" },
      "pir": { "detected": true, "at": "2026-11-09T14:41:05+09:00" },
      "series": [ { "at": "2026-11-09T13:42:00+09:00", "ppm": 620, "temperatureC": 24.9, "humidityPct": 45.0 },
                  { "at": "2026-11-09T13:43:00+09:00", "ppm": null, "temperatureC": null, "humidityPct": null } ],
      "current": { "reservationId": 12, "startAt": "…14:00+09:00", "endAt": "…16:00+09:00", "people": 4 },
      "next": { "startAt": "…16:30+09:00", "people": 3 } } ] }
```

| 필드 | 뜻 |
|---|---|
| `presence` | 재실 `OCCUPIED`·`EMPTY`. 재실 신호가 1분(`offline-after-seconds`) 넘게 없거나 한 번도 없으면 `UNKNOWN` (방 현황과 같은 기준) |
| `lastSignalAt` · `offline` | 재실·CO2 중 마지막으로 받은 시각, 그게 1분보다 오래됐거나 없으면 `offline: true` |
| `co2` | 가장 최근 CO2 값 (창 밖이어도 마지막 값). 한 번도 없으면 `null` |
| `temperatureC` · `humidityPct` | 값이 있는 가장 최근 온도·습도. 없으면 `null` |
| `light` | 조도 센서 **원시값 0~1023**(lux 아님. 우노 `analogRead(A0)`, CDS + 10kΩ 분압, 밝을수록 큼)과 받은 시각. 재실 이벤트 `sources.light`의 최신 값, 지난 `minutes`분 안에 없으면 `null`. 보여 주기만 해요 |
| `pir` | PIR 움직임 감지(`detected`)와 받은 시각. 재실 이벤트 `sources.pir`의 최신 값, 없으면 `null` |
| `series` | 1분마다 한 줄, 오래된 것이 앞, 지금 진행 중인 분까지 `minutes`줄. 그 분에 값이 없으면 `null`. ppm은 정수, 온도·습도는 소수 한 자리 평균 |
| `current` | 지금 시각을 낀 예약(입실 전 포함). `people`은 예약 인원. 없으면 `null` |
| `next` | 지금 뒤 가장 가까운 예약. 없으면 `null` |

- 젯슨 예측(`forecast`)은 아직 없어요. 화면은 없으면 최근 추세로 그려요
- 방 안 사람 수(카메라)는 Q-17 결정 전이라 주지 않아요

## 6-3. 퇴실 뒤 정리 확인 (방 카메라, FR-DEV-105)

방 안 카메라(젯슨)가 **예약이 실제로 끝났을 때만** 남은 물건을 봐요. "방이 30초 비면" 기준은 다 같이 화장실만 가도 걸려서 쓰지 않아요(11/6 지영·수연).
**사람이 찍힌 화면은 저장하지 않아요.** 젯슨은 1초에 한 번 화면에서 사람 수와 물건 이름만 뽑고 화면을 바로 버려요. 개발용 미리보기는 기본 꺼짐, 켜도 팀 Tailscale 안에서만 열리고 저장하지 않아요.
**11/11 수연 결정(Q-27):** 사람이 0명일 때 찍은 정리 확인 사진 3장(입실 직전 빈 방 `ENTRY`, 퇴실 뒤 `FIRST`, 다시 확인 `RECHECK`)은 core에 30일 두고, 관리자 화면 정리 확인 목록에서 입실 전·퇴실 뒤를 나란히 봐요. 아래 `tidy-photos`.
비교 기준은 사람이 처음 보이기 직전 3초 동안 0명인 빈 방 화면 1장이에요(11/9, Q-22). 젯슨 안의 비교용 파일은 정리 확인 결과를 보내면 지워요. 같은 순간의 컬러 사진은 `ENTRY`로 core에 올려요(11/11). 방이 한 번도 안 비어 기준이 없으면 그 예약은 결과를 보내지 않아요(어질러짐 신고가 받음).
10차 회의(11/10)로 사람이 0명인 빈 방·물건 사진은 저장할 수 있게 됐어요(Q-27, 지영 "core에 30일"). 젯슨이 확인 결과와 별도로 `FIRST`·`RECHECK` 사진을 1장씩 core에 올리고, core가 30일 뒤 지워요(아래 `tidy-photos`, core #115). 물건을 가져가러 들어가는 방법은 "찾으러 갈게요" 10분 QR이에요(`POST /api/reservations/{id}/retrieve-pass`, core #114).

**두고 간 물건 흐름 (11/10 지영·수연, core #112).** 예약 하나에 확인이 **두 번**까지 있어요.
1. 첫 확인(`FIRST`): 예약이 끝나면 바로 봐요. 남은 물건이 있으면(`LEFT_ITEMS`, 쓰레기 포함, 종류 구분 없음) 대표자·참여자에게 **알림만** 가요. 신고·경고는 없어요.
2. 다시 확인(`RECHECK`): 첫 확인이 `LEFT_ITEMS`였으면 **퇴실 3시간 뒤**에 다시 봐요(`current.recheck`가 알려 줘요). 그때도 확신도 0.8 이상 물건이 남아 있으면 core가 바로 어질러짐 신고를 사실로 정하고 **대표자에게 경고**를 줘요.
3. 퇴실 뒤 3시간 안에 **다음 팀이 입실하면 다시 보지 않고 경고도 없어요**(누구 물건인지 갈리지 않음). 지금 core 코드 기준이에요. 11/10 지영이 "다음 팀이 써도 3시간 안에 안 찾아가면 경고"를 원해서, 수연과 방법(다음 팀 입실 때 다시 확인 등)을 정한 뒤 바꿀 예정이에요.
4. 찾으러 갈게요: 첫 확인이 `LEFT_ITEMS`면 대표자·참여자가 퇴실 뒤 3시간 안에 10분짜리 QR을 받아 들어갈 수 있어요(다음 팀이 쓰는 중이어도). 아래 `retrieve-pass`.

### `GET /api/rooms/{roomId}/current` 지금 예약·방금 끝난 예약 — 장치 토큰 필요

젯슨이 **10초마다** 불러요. `lastEnded.reservationId`가 바뀌면 그 예약이 방금 끝난 것이라 첫 확인을 시작해요. 시간 종료·직접 퇴실·자리 비움 퇴실이 모두 잡히고, 다음 예약이 바로 입실해도 놓치지 않아요. `recheck`가 있으면 `at` 시각이 지난 뒤 다시 확인을 보내요.

```json
200
{ "roomId": 2,
  "current": { "reservationId": 41, "status": "IN_USE",
               "startAt": "2026-11-09T15:00:00+09:00", "endAt": "2026-11-09T17:00:00+09:00" },
  "lastEnded": { "reservationId": 40, "status": "CHECKED_OUT", "endedAt": "2026-11-09T14:52:10+09:00" },
  "recheck": { "reservationId": 40, "at": "2026-11-09T16:52:10+09:00" } }
```

| 필드 | 뜻 |
|---|---|
| `current` | 지금 시간에 걸린 예약(`BOOKED` 입실 전 또는 `IN_USE`). 없으면 `null` |
| `lastEnded` | 이 방에서 가장 최근에 끝난 예약. `status`는 `CHECKED_OUT`(직접 퇴실·강제 퇴실) `AUTO_CHECKED_OUT`(시간 종료) `AWAY_RELEASED`(자리 비움). 입실 안 한 예약(`NO_SHOW`·`CANCELED`)은 방에 아무도 없었으니 넣지 않아요. 없으면 `null` |
| `endedAt` | 실제로 나간 시각(`checked_out_at`) |
| `recheck` | 다시 확인할 예약. `lastEnded` 예약의 첫 확인이 `LEFT_ITEMS`이고, 아직 다시 확인이 없고, 그 뒤 이 방에 아무도 입실하지 않았을 때만 줘요. 아니면 `null`. `at` = 퇴실 시각 + 3시간(`mspace.tidy.recheck-after-minutes`, 기본 180) |

- 예약자 이름·학번은 주지 않아요(장치에 필요 없음).
- 오류: `401` 토큰 문제, `404 ROOM_NOT_FOUND`.

### `POST /api/rooms/{roomId}/tidy-checks` 정리 확인 결과 — 장치 토큰 필요

방에 **사람이 0명일 때 보인 물건 목록**으로 판단한 결과를 보내요. 예약 하나에 `FIRST` 1개, `RECHECK` 1개까지예요.

```json
{ "eventId": "5b0e1c2a-…", "reservationId": 40, "deviceId": "camera-2", "phase": "FIRST",
  "result": "LEFT_ITEMS", "items": [ { "name": "umbrella", "confidence": 0.93 }, { "name": "bottle", "confidence": 0.71 } ],
  "observedAt": "2026-11-09T14:53:00+09:00" }
→ 200 { "result": "STORED", "reportId": null } 또는 { "result": "DUPLICATE", "reportId": null }
```

`RECHECK`에서 경고를 줬으면 `reportId`에 그 신고 번호가 와요(아니면 `null`).

| 필드 | 형식 | 뜻 |
|---|---|---|
| `eventId` | 글자 1~64자 | 센서와 같아요. 같은 값을 다시 보내면 `DUPLICATE`로 한 번만 처리 |
| `reservationId` | 정수 | `current`의 `lastEnded.reservationId` |
| `deviceId` | 글자 50자 이하 | 보낸 장치(`camera-방번호`) |
| `phase` | `FIRST` · `RECHECK`, 없으면 `FIRST` | 퇴실 직후 첫 확인 / 3시간 뒤 다시 확인 |
| `result` | `CLEAN` · `LEFT_ITEMS` | 남은 것 없음 / 남은 물건 있음(쓰레기 포함, 종류 구분 없음). 예전 값 `NEEDS_CLEANING`은 400 |
| `items` | **필수**, 0~20개. `CLEAN`이면 빈 배열, `LEFT_ITEMS`면 1개 이상 | 깨끗한 빈 방보다 늘어난 물건. `name`은 모델이 내는 영어 이름 그대로(1~50자), `confidence`는 확신도 0~1(필수). 방에 원래 있는 물건(리모컨 등)은 빠져요 |
| `observedAt` | 시각, 오프셋 필수(`+09:00` 권장, `Z`도 받음) | 확인한 시각. 오프셋이 없으면 400 |

**core가 하는 일**
- `FIRST` + `LEFT_ITEMS` → 그 예약의 대표자·참여자 모두에게 알림 `LEFT_ITEM`(제목 "두고 간 물건이 있어요", 내용 예 "2번 방에 umbrella·bottle을(를) 두고 가신 것 같아요. 3시간 안에 챙겨 가 주세요."). 물건이 많으면 "외 N개"로 줄여 300자 안에 맞춰요. 신고·경고는 없어요. 예전처럼 어질러짐 신고를 자동으로 만들지 않아요.
- `RECHECK` + 확신도 0.8 이상(`mspace.tidy.warn-min-confidence`, 모델 비교 뒤 조정) 물건이 하나라도 있음 → 그 방에 어질러짐(`MESSY`) 신고를 만들고(신고자 없음, `source: CAMERA`, 내용 "방 카메라 자동 확인: 퇴실 뒤 다시 봐도 남은 물건 umbrella(0.93)") **같은 트랜잭션에서 사실(`CONFIRMED`)로 정해요**(판정한 관리자 없음). 이 예약의 **대표자에게 경고**가 가요(FR-RPT-104 규칙 그대로, 이유 "두고 간 물건을 3시간 안에 챙겨 가지 않았어요."). 0.8 아래 물건만 남았으면 기록만 해요.
- 판정 위치(Q-28)의 AI 어댑터가 아직 없어서, 지금은 core가 위 기준으로 바로 정해요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 형식이 틀림 |
| `ROOM_NOT_FOUND` | 404 | 없는 방 |
| `RESERVATION_NOT_FOUND` | 404 | 이 방의 예약이 아님 |
| `RESERVATION_NOT_ENDED` | 409 | 아직 안 끝난 예약이거나 입실 안 한 예약 |
| `TIDY_CHECK_NOT_LATEST` | 409 | 이 방에서 **가장 최근에 끝난 예약**(`lastEnded`)이 아님. 장치 토큰이 공용이라 지난 예약에 신고를 만들지 못하게 막아요 |
| `TIDY_CHECK_EXISTS` | 409 | 이 예약에 같은 `phase` 결과가 다른 `eventId`로 이미 있거나, 같은 `eventId`를 다른 예약·`phase`에 씀 |
| `TIDY_RECHECK_NOT_NEEDED` | 409 | `RECHECK`인데 다시 볼 것이 없음(첫 확인이 없거나 `CLEAN`), 또는 퇴실 뒤 이 방에 다음 팀이 입실함 |
| `TIDY_RECHECK_TOO_EARLY` | 409 | `RECHECK`인데 지금 시각이나 `observedAt`이 `recheck.at`(퇴실 + 3시간)보다 이름 |

- API 게이트웨이: 장치 토큰은 JWT가 아니라서 `/api/rooms/{roomId}/current`, `/api/rooms/{roomId}/tidy-checks`도 `/api/sensor-events`처럼 공개 경로로 넣어야 해요. 장치 토큰은 core가 확인해요 (재웅)

### `POST /api/reservations/{id}/retrieve-pass` 찾으러 갈게요 QR — 로그인 필요 (✅ develop, core #114)

두고 간 물건 알림(`LEFT_ITEM`)을 받은 **대표자·참여자**가 누르면 문 앞에서 쓸 10분짜리 QR을 줘요. 본문 없음.

```json
200
{ "reservationId": 40, "roomId": 2, "code": "482913", "qrText": "MSPACE:2:482913",
  "expiresAt": "2026-11-09T15:12:10+09:00" }
```

- 조건: 그 예약의 첫 확인(`FIRST`)이 `LEFT_ITEMS`이고, 퇴실 뒤 3시간(`recheck-after-minutes`) 안. 다음 팀이 방을 쓰는 중이어도 받아요.
- 10분 안에 다시 누르면 **같은 QR**을 줘요. 10분 안에는 여러 번 열 수 있어요.
- 문 앞 장치는 보통 입실 QR처럼 `POST /api/access/verify`로 보내고, 답이 `{"result":"OPENED","purpose":"RETRIEVE"}`예요. 체크인은 하지 않아요.

| 오류 | 상태 | 언제 |
|---|---|---|
| `RESERVATION_NOT_FOUND` | 404 | 없는 예약이거나 내가 대표자·참여자가 아님 |
| `RESERVATION_NOT_ENDED` | 409 | 아직 안 끝났거나 입실 안 한 예약 |
| `RETRIEVE_NOT_NEEDED` | 409 | 방 카메라가 두고 간 물건을 찾지 않음(첫 확인이 없거나 `CLEAN`) |
| `RETRIEVE_TOO_LATE` | 409 | 퇴실 뒤 3시간 지남 |

### `POST /api/rooms/{roomId}/tidy-photos` 정리 확인 사진 올리기 — 장치 토큰 필요 (✅ develop, core #115)

`multipart/form-data`로 보내요. 정리 확인 결과(`tidy-checks`)와 따로 올려요.

| 칸 | 뜻 |
|---|---|
| `reservationId` | `current`의 `lastEnded.reservationId` |
| `phase` | `ENTRY` · `FIRST` · `RECHECK` (`ENTRY` = 입실 직전 빈 방, **core 추가 예정**) |
| `deviceId` | 보낸 장치(`camera-방번호`), 장치 토큰과 맞아야 해요 |
| `file` | JPEG·PNG 1장, 최대 2MB. 확장자가 아니라 파일 앞부분으로 확인해요 |

→ `201 { "reservationId": 40, "phase": "FIRST", "uploadedAt": "2026-11-09T14:53:05+09:00", "sizeBytes": 412345 }`

- 예약 하나에 `ENTRY`·`FIRST`·`RECHECK` 1장씩. 이 방에서 가장 최근에 끝난 예약에만 올릴 수 있어요. 그래서 젯슨은 `ENTRY`도 입실 때가 아니라 퇴실 뒤 `FIRST` 결과를 보낸 다음 같이 올려요. 30일 뒤 자동으로 지워요(퇴실 사진과 같은 규칙, FR-RPT-102).
- 사람이 0명일 때 찍은 사진만 올려요(10차 회의).
- nginx가 요청을 1MB쯤에서 막을 수 있어요. 젯슨이 1MB 아래로 줄여 보내고, 서버 한도는 재웅 확인 필요.
- API 게이트웨이: `current`·`tidy-checks`처럼 공개 경로로 넣어야 해요(장치 토큰은 core가 확인, 재웅).

| 오류 | 상태 | 언제 |
|---|---|---|
| `PHOTO_INVALID` | 400 | `reservationId`·`phase`·`file` 빠짐, JPEG·PNG가 아님 |
| `PHOTO_TOO_LARGE` | 400 | 2MB 넘음 |
| `DEVICE_UNAUTHORIZED` · `DEVICE_ROOM_MISMATCH` · `DEVICE_ID_MISMATCH` | 401 · 403 · 403 | 장치 토큰 문제(6장과 같음) |
| `RESERVATION_NOT_FOUND` | 404 | 이 방의 예약이 아님 |
| `TIDY_CHECK_NOT_LATEST` | 409 | 이 방의 마지막 퇴실 예약이 아님 |
| `PHOTO_ALREADY_UPLOADED` | 409 | 같은 `phase` 사진이 이미 있음 |

### `GET /api/reservations/{id}/tidy-photos/{phase}` 정리 확인 사진 보기 — 로그인 필요 (✅ develop, core #115)

`phase`는 `ENTRY` · `FIRST` · `RECHECK`(`ENTRY`는 core 추가 예정). 관리자 화면은 `GET /api/tidy-checks`의 `reservationId`로 입실 전(`ENTRY`)·퇴실 뒤(`FIRST` 또는 `RECHECK`) 사진을 불러 나란히 보여 줘요(웹 추가 예정). 그 예약의 대표자·참여자와 관리자만 볼 수 있어요. 사진 바이트를 그대로(`image/jpeg`·`image/png`) 주고, 저장하지 않게 `Cache-Control: no-store`예요. 사진이 없거나 볼 수 없는 사람이면 모두 `404 PHOTO_NOT_FOUND`(있는지도 알리지 않아요).

### `GET /api/tidy-checks?days=7` 정리 확인 목록 — 관리자만 (✅ develop, core #110)

최근 N일(1~92, 기본 7) 결과를 최근 순으로 줘요. 관리자가 아니면 `403 ADMIN_ONLY`.

```json
200
{ "items": [ { "reservationId": 40, "roomId": 2, "roomName": "2번 방", "endedAt": "2026-11-09T14:52:10+09:00",
               "phase": "RECHECK", "result": "LEFT_ITEMS", "items": [ { "name": "umbrella", "confidence": 0.93 } ],
               "leftItems": [], "reportId": 12, "observedAt": "2026-11-09T16:53:00+09:00" } ] }
```

- 예약 하나에 `FIRST`·`RECHECK`가 따로 한 줄씩 나와요. 11/10 전 기록은 `confidence`가 `null`이고, `result`가 `NEEDS_CLEANING`, `leftItems`(`BAG` 등)가 남아 있을 수 있어요. 새 기록의 `leftItems`는 늘 빈 배열이에요.

- 신고 목록(`GET /api/reports`)에 자동 신고는 `source: "CAMERA"`, 사람 신고는 `source: "USER"`로 나와요(core #81). 카메라 신고는 신고자가 없어서 `reporterReservationId`가 `null`이에요.
- 신고 증거(`GET /api/reports/{id}/evidence`)의 앞 예약 칸에도 **첫 확인(`FIRST`)** 결과를 `tidyCheck`로 붙여요(없으면 `null`, core #110). 다음 팀 신고의 근거는 퇴실 직후 모습이라서요. 이 칸의 `items`는 이름 배열 그대로예요(확신도 없음). 예: `"tidyCheck": {"result": "LEFT_ITEMS", "items": ["umbrella"], "leftItems": [], "observedAt": "2026-11-09T14:53:00+09:00"}`. 사람이 어질러짐을 신고하면 관리자가 앞 사람 퇴실 사진과 카메라 결과를 같이 봐요.

## 6-4. 장치 오류 · 방의 사과 (Q-19, core #99)

문 장치(QR·키패드 게이트웨이)가 끊겨서 못 들어간 학생이 노쇼 벌칙을 받지 않게 해요. 장치는 끊긴 동안 오류 구간과 그동안 눌린 번호를 모아 뒀다가, 다시 연결되면 같은 `eventId`로 보내요(재실 이벤트처럼 1→2→4…30초 간격 재전송).

### `POST /api/device-faults` — 장치 토큰 필요

```json
{ "eventId": "qr-1-0005", "roomId": 1, "deviceId": "qr-1", "kind": "SERVER_UNREACHABLE",
  "startedAt": "2026-11-10T14:00:00+09:00", "endedAt": "2026-11-10T14:20:00+09:00",
  "attempts": [{ "at": "2026-11-10T14:02:00+09:00", "code": "123456", "via": "QR" }] }
```
→ `200 { "result": "STORED" }` · 같은 `eventId`를 다시 보내면 `"result": "DUPLICATE"` (아무것도 안 바뀜)

| 칸 | 값 |
|---|---|
| `kind` | `SERVER_UNREACHABLE` 서버 연결 안 됨(와이파이 끊김 포함) · `SERVER_ERROR` 서버 5xx · `DEVICE_RESTART` 꺼졌다 켜진 구간 · `CAMERA_DOWN` QR 카메라 못 읽음 · `KEYPAD_DOWN` 키패드 끊김 |
| `attempts[]` | 오류 구간 안에 눌린(읽힌) 번호. 없으면 빈 목록. `via`는 `KEYPAD`·`QR`, 최대 100개 |

- 번호는 방마다 24시간에 200개까지 받아요. 넘으면 `429 DEVICE_FAULT_TOO_MANY` (번호 없는 오류 기록은 계속 받아요)
- 번호가 예약과 맞았는지는 응답에 넣지 않아요. 장치 토큰이 새도 번호를 넣어 보고 맞는지 알아내는 통로가 되지 않게요(키패드 5번 잠금을 거치지 않으니까요). 무른 결과는 학생 알림으로 가요
- **번호는 저장하지 않아요.** core는 같은 방·그 번호·번호가 맞을 수 있던 시간(시작 10분 전 ~ 입실 마감)의 예약을 찾는 데만 쓰고 예약 id만 남겨요. 키패드 5번 잠금에도 세지 않아요. 장치도 번호를 메모리에만 둬요
- 찾은 예약이 노쇼면 **노쇼 벌칙 셈에서 빼고**, 그 노쇼 때문에 걸린 이용 정지를 풀고, 대표자·참여자에게 사과 알림(`DEVICE_APOLOGY`)을 보내요. 예약 상태는 기록용으로 `NO_SHOW` 그대로예요
- 무르기는 이 요청 안에서 하지 않고, 응답을 보낸 뒤 따로 **곧바로**(몇 초 안) 해요. 실패하면 1분 정리가 다시 해요. 번호가 맞을 때만 요청 안에서 일이 늘면 응답 시간 차이로 맞은 번호를 짐작할 수 있어서, 이 요청은 맞든 안 맞든 저장만 하고 응답해요(11/9 지영)
- 그래서 오류가 노쇼 처리보다 먼저 와도(입실 마감 전에 다시 연결) 장치는 순서를 신경 쓰지 않아도 돼요
- 번호 없이 시간만 겹치는 노쇼는 누구 예약인지 몰라서 자동으로 무르지 않아요 (관리자 목록은 다음 PR)
- "문 안 열림"은 지금 문에 잠금장치가 없어(LED만) 보내지 않아요

| 오류 | 상태 | 언제 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 빠진 값, 모르는 `kind`·`via`, 번호가 6자리 숫자가 아님, 끝 < 시작, 24시간 넘음, 끝이 지금보다 5분 넘게 뒤, 번호 시각이 구간 밖 |
| `DEVICE_UNAUTHORIZED` | 401 | 장치 토큰 없음·틀림 |
| `DEVICE_ROOM_MISMATCH` · `DEVICE_ID_MISMATCH` | 403 | 다른 방·다른 장치 토큰 |

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
- **화면 쪽 음성·추천 말 (web #43, 11/6):** API는 그대로예요. 마이크는 브라우저 음성 인식(`SpeechRecognition`, ko-KR)이 글로 바꿔 같은 `POST /api/agent/chat`으로 보내고, [듣기]는 `reply`를 `speechSynthesis`로 읽어요(서버·비용 없음). 입력 칸 위 추천 메뉴(오늘 브리핑·방 예약·지금 빈 방·시간표·내 예약·사용 규칙)도 정해진 문장을 그대로 보내요. 추천 말이 바로 예약하지 않고 확인 카드를 거쳐요
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

### AI로 간단히 예약하기 (FR-AGT-105, 11/9) — 새 API 없음

홈의 **[AI로 간단히 예약하기]** 버튼 → 비서 화면이 "언제, 나 포함 몇 명이서 쓰실 건가요?"로 시작해요. 말 몇 마디로 확인 카드까지 가고, 함께할 친구는 카드에서 이름으로 골라요. 지금 있는 API만 써요.

| 차례 | 누가 | 무엇을 | API |
|---|---|---|---|
| 1 | 화면 | 홈 버튼 → `/assistant?quick=1`. 첫 질문 말풍선은 화면이 만들고 소리로 읽어요(모델 호출 없음) | 없음 |
| 2 | 학생 → 비서 | "내일 3시부터 2시간, 나 포함 4명" (말 또는 글). 빠진 것만 비서가 하나씩 물어요 | `POST /api/agent/chat` |
| 3 | 비서 | 인원을 묻는 답이면 `suggestions`에 `["나 포함 3명","나 포함 4명","나 포함 5명"]` | (같은 응답) |
| 4 | 비서 | 모이면 빈방 찾기 → 예약 확인 카드 `needsParticipants: 3` | (같은 응답) |
| 5 | 화면 | 카드가 친구 목록을 받아 버튼으로 보여 주고, "지영이랑 재웅이랑 수연이"처럼 말하면 가린 이름(김*영)과 맞춰 학번 칸을 채워요 | `GET /api/friends` (core 4장) |
| 6 | 학생 | [확인] | `POST /api/agent/confirmations/{token}` |
| 7 | core | 참여자에게 "OO님이 함께 예약했어요" 알림 | ✅ develop (core #108, `RESERVATION_INVITED`, 아래) |

- **친구 이름은 외부 AI로 가지 않아요.** 5번 이름 맞추기는 브라우저 안에서 해요. 비서 지시문도 "이름·학번을 묻지 않는다"를 그대로 둬요
- 이름 맞추기 규칙: 가린 이름의 보이는 글자(성·끝 글자)만 비교해요. "김지영"(3글자)은 `김*영`과, "지영"(이름만)은 `?*영`과 맞아요. 끝의 "이·아·야"("지영이")는 떼고 한 번 더 봐요. 후보가 둘 이상이면 채우지 않고 후보 버튼을 보여 줘요
- 친구가 아니면 채우지 않고 "친구 탭에서 먼저 친구를 맺어 주세요"라고 안내해요. 학번 칸 직접 입력도 그대로 돼요

**"누구랑?" (11/9 영준):** 비서가 함께할 사람을 물으면 결과에 `hints: ["PICK_FRIENDS"]`가 와요. 화면은 친구 버튼으로 고르게 하고 비서에게는 "나 포함 N명"만 보내요. 고른 친구는 다음 확인 카드 칸에 미리 채워요.

**내 팀 (11/9 영준, core 변경 없음)** — "팀 A랑 내일 3시"

| API | 하는 일 |
|---|---|
| `GET /api/agent/teams` | `200 { "teams": [ { "name": "팀 A", "members": [ { "studentId": "20230002", "name": "홍*동" } ] } ] }` |
| `PUT /api/agent/teams/{name}` `{ "members": [ ... ] }` | 같은 이름이면 바꾸고 없으면 추가 → 전체 목록. 친구 1~7명, 이름 1~12자, 팀 10개까지 |
| `DELETE /api/agent/teams/{name}` | `204` (없어도 204) |

- 오류: `400 INVALID_TEAM`(빈 멤버·중복 학번·긴 이름), `400 TOO_MANY_TEAMS`, `401 UNAUTHORIZED`, `503 MEMORY_UNAVAILABLE`(저장 실패)
- 저장은 비서 Redis `agent:teams:<학생 키>`(TTL 없음). 모델 지시문에는 **팀 이름과 인원만** 들어가요. 팀원은 화면이 확인 카드에 채워요
- 친구인지는 비서가 보지 않아요. 예약할 때 core가 `403 PARTICIPANT_NOT_FRIEND`로 막아요

**참여자 알림 (core #108 머지, 11/9 지영 확인):** 예약이 만들어지면(웹·비서·즉시 시작 모두) 대표자를 뺀 참여자마다 알림함에 한 줄.

```json
{ "kind": "RESERVATION_INVITED", "reservationId": 31, "title": "함께 쓰는 예약이 생겼어요",
  "body": "홍*동님이 11/10(화) 15:00–17:00 2번 방에 함께 예약했어요." }
```
- 예약 저장과 같은 트랜잭션에서 알림 줄을 넣어요(예약이 실패하면 알림도 없음). 멱등 키로 같은 예약을 다시 돌려줄 때(`200`)는 다시 넣지 않아요
- 푸시는 이미 있는 알림 보내기 경로를 그대로 타요. 입실 번호는 넣지 않아요(대표자만 받음)

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

### `GET`·`PUT /api/agent/persona` 비서 성향 (assistant #85, 8차 회의)

학생이 비서 성향을 셋 중 하나로 골라요. 성향은 지시문 끝 한 토막만 바꾸고, 두 단계 확인·개인정보 규칙은 그대로예요(assistant `docs/설계.md` 5.v).

| `style` | 이름 | 하는 일 |
|---|---|---|
| `QUIET` | 조용히 | 물은 것만 짧게, 제안 없음, 빠른 답장 버튼 없음 |
| `ACTIVE` | 알아서 (기본) | 답 끝에 다음 행동 하나 제안 |
| `NAGGING` | 잔소리 | 입실·퇴실·연장·노쇼·시간표를 매번 챙기고 재촉 |

```json
GET  → { "style": "ACTIVE", "label": "알아서", "description": "...", "options": [{ "style": "QUIET", "label": "조용히", "description": "..." }, ...] }
PUT  { "style": "NAGGING" } → 같은 모양
```
- 채팅으로 "잔소리 모드로 해 줘"라고 해도 모델 없이 같은 저장을 해요(`tools: ["persona"]`)
- 저장은 학생(토큰 `sub`)별, 기간 없음. "새 대화"로 안 지워져요
- 오류: `400 INVALID_PERSONA`(틀린 `style`), `503 MEMORY_UNAVAILABLE`(저장 실패. 읽기 실패는 기본 성향으로 넘어가요), `401 UNAUTHORIZED`

## 8. 코드와 문서가 아직 다른 곳 (11/6 점검)

11/3~6에 적었던 줄(알림 읽음 #69, 판정 벌칙 #70, 강제 퇴실 #73, 연장 #51, 신고 #52, 노쇼 벌칙 #58, `created_via` #45 + 비서 #54, 자리 비움 `AWAY_RELEASED`, 센서 #60·게이트웨이 공개 경로, 비서 API)과 11/10에 확인한 참여자 알림 `RESERVATION_INVITED`(#108)는 모두 develop에 머지돼 지웠어요.

| 무엇 | 지금 | 할 일 |
|---|---|---|
| springdoc `/swagger-ui` | 컨벤션에만 있고 core에 의존성 없음 | core에 추가하면 이 문서는 요약본으로 줄이기 |
| 입실 방식 | **11/10 결정: QR만**(키패드·NFC 뺌). API는 6장 하나 그대로 | core에 남은 키패드 값 정리(요구사항 Q-29, 지영·수연) |
| 증거 사진 | core #76 머지됨, 응답 예시는 아직 없음 (5장 "관리자 화면용 API") | 응답 예시 추가 |
