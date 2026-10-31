# 팀 컨벤션

> 최종 수정 2026-10-31 · 초안. 팀 확인 후 확정 · 모든 저장소(`core`, `front`, `infra`, `agent`, `docs`)에 적용

**한 줄 요약**: 브랜치는 `feat/…`, 커밋은 `feat: 한국어 설명`, PR에는 요구사항 ID, 오류는 `{code, message}`, DB는 `snake_case`, 시간은 서울 시간 `+09:00`, 비밀값은 절대 커밋하지 않는다.

| 영역 | 핵심 | 자동으로 지키게 하는 것 |
|---|---|---|
| 1. Git | 브랜치·커밋·PR·리뷰 | 브랜치 보호, PR 템플릿 |
| 2. 이슈 | 제목·라벨·요구사항 ID | 이슈 템플릿 |
| 3. 백엔드 (Java) | 패키지·이름·예외·테스트 | `.editorconfig`, IntelliJ 포맷터 |
| 4. API | 주소·JSON·시간·오류 | OpenAPI 자동 문서 |
| 5. DB | 이름·시간·마이그레이션 | Flyway |
| 6. 프론트 (TS) | 파일·이름·API 타입 | ESLint, Prettier |
| 7. Jetson (Python) | 이름·포맷 | ruff |
| 8. 문서 | 마크다운 규칙 | — |

---

## 1. Git

### 브랜치

| 브랜치 | 용도 |
|---|---|
| `main` | 시연용. 버전이 끝났을 때만 합친다 |
| `develop` | 매일 합치는 곳 |
| `feat/…` `fix/…` `docs/…` `chore/…` `refactor/…` `test/…` | `develop`에서 따서 1–2일 안에 합친다 |

- 이름: `<타입>/<이슈번호>-<짧은-영어>` · 예: `feat/12-reservation-create`
- `main`, `develop`에 직접 push 금지.
- 자세한 흐름은 [인프라 문서](infra/infra-cicd.md) 3절(간소화 Git Flow).

### 커밋 메시지

```
<타입>: <무엇을 했는지, 한국어>

(필요하면 본문: 왜 바꿨는지)

Refs #12
```

| 타입 | 언제 |
|---|---|
| `feat` | 기능 추가 |
| `fix` | 버그 수정 |
| `refactor` | 동작은 그대로, 코드 구조만 |
| `test` | 테스트만 |
| `docs` | 문서만 |
| `chore` | 빌드·설정·의존성 |

- 예: `feat: 예약 생성 API 추가`, `fix: 동시 퇴실 시 500 오류 수정`
- 제목은 50자 안, 마침표 없음. 커밋 하나에 한 가지 일.

### PR

| 항목 | 규칙 |
|---|---|
| 제목 | 커밋 형식 (`feat: 예약 생성 API 추가`) |
| 본문 | 요구사항 ID(`FR-RSV-001`), 이슈(`Closes #12`), 한 일, 확인 방법, 화면이면 스크린샷 |
| 크기 | 변경 400줄 안. 넘으면 나눈다 |
| 마감 | 매일 15시까지 PR |
| 리뷰 | 다른 팀원 1명 승인. 당일 안에 리뷰. CodeRabbit은 참고용 |
| 합치기 | `develop`은 Squash merge 후 브랜치 삭제, `main`은 Merge commit |
| 통과 조건 | 빌드·테스트 통과. `main`은 새 코드 커버리지 80% 이상 |

**PR 본문 양식** (`.github/pull_request_template.md`)

```markdown
## 요구사항
FR-RSV-001 · Closes #12

## 한 일
-

## 확인 방법
- [ ] `./mvnw test` 통과
- [ ] 개발 서버에서 확인 (화면이면 스크린샷)
```

### 리뷰 댓글

| 머리말 | 뜻 |
|---|---|
| `[필수]` | 고치지 않으면 승인하지 않는다 |
| `[제안]` | 고치면 좋다. 작성자가 판단 |
| `[질문]` | 이해가 안 돼서 묻는다 |

---

## 2. 이슈

| 항목 | 규칙 |
|---|---|
| 제목 | `[FR-RSV-001] 예약 생성` 처럼 요구사항 ID를 앞에 |
| 라벨 | 종류(`feat`, `fix`, `docs`, `chore`) + 버전(`v0.1`, `v0.2`, `v0.3`) |
| 본문 | 요구사항 문서의 수용 기준을 체크리스트로 복사 |
| 담당 | 작업을 시작할 때 본인 지정 |

---

## 3. 백엔드 (Java · Spring Boot)

### 패키지 — 기능별로 나누고, 안은 포트·어댑터

```
kr.ac.chosun.mspace
├── reservation          ← 기능(모듈) 하나
│   ├── domain           ← 엔티티, 규칙, 상태
│   ├── application      ← 서비스, 포트(인터페이스)
│   └── adapter
│       ├── in/web       ← 컨트롤러, 요청·응답 DTO
│       └── out          ← DB, 메일, 키패드 구현
├── member
├── room
├── access
└── common               ← 공통 오류, 설정
```

- 기능끼리는 서로의 `application`만 부른다. 남의 `domain`이나 `adapter`를 직접 쓰지 않는다.

### 이름

| 대상 | 규칙 | 예 |
|---|---|---|
| 클래스 | PascalCase | `ReservationService` |
| 메서드·변수 | camelCase, 동사로 시작 | `createReservation()` |
| 상수 | UPPER_SNAKE | `MAX_DURATION` |
| 요청·응답 | `…Request` / `…Response` | `CreateReservationRequest` |
| 포트 | 하는 일 + 없음 | `IdentityVerifier`, `AccessDevice` |
| 어댑터 | 방식 + 포트 이름 | `EmailIdentityVerifier`, `KeypadAccessDevice` |

### 코드 규칙

| 항목 | 규칙 |
|---|---|
| 포맷 | IntelliJ 기본 + `.editorconfig` (공백 4칸, UTF-8, LF, 줄 끝 공백 제거) |
| 엔티티 | 컨트롤러 밖으로 내보내지 않는다. 항상 DTO로 바꿔서 응답 |
| 의존성 주입 | 생성자 주입 (`@RequiredArgsConstructor`). 필드 `@Autowired` 금지 |
| 숫자 규칙 | 7일·2시간 같은 값은 코드에 박지 않고 설정(`application.yml`)으로 |
| 예외 | `BusinessException(ErrorCode)` 하나로 던지고, `@RestControllerAdvice` 한 곳에서 `{code, message}`로 바꾼다 |
| 로그 | `System.out` 금지, `@Slf4j`. 개인정보(메일·학번) 로그에 남기지 않기 |
| 시간 | `OffsetDateTime`/`Instant` + `Clock` 주입 (테스트에서 시간 고정) |

### 테스트

| 항목 | 규칙 |
|---|---|
| 이름 | `@DisplayName("같은 방 시간이 겹치면 409")` 한국어로 |
| 모양 | given / when / then 주석으로 구분 |
| DB | Testcontainers PostgreSQL (H2 쓰지 않음, `EXCLUDE` 제약이 없어서) |
| 범위 | 요구사항 수용 기준 1개 = 테스트 1개 이상 |

---

## 4. API

| 항목 | 규칙 | 예 |
|---|---|---|
| 주소 | `/api/` + 복수 명사 + kebab-case | `/api/reservations`, `/api/access/verify` |
| 동작 | 상태를 바꾸는 동작은 `POST /{id}/<동사>` | `POST /api/reservations/7/cancel` |
| JSON 키 | camelCase | `startAt`, `roomId` |
| 시간 | ISO-8601, 서울 시간 오프셋 포함 | `2026-11-03T14:00:00+09:00` |
| 성공 | 생성 `201`, 나머지 `200` | |
| 오류 형식 | `{ "code": "...", "message": "..." }` | |
| 오류 코드 | UPPER_SNAKE, 영역으로 시작 | `RESERVATION_OVERLAP`, `AUTH_CODE_EXPIRED` |
| 문서 | springdoc OpenAPI 자동 생성. 프론트 타입과 MCP 도구가 이 문서를 근거로 한다 | `/swagger-ui` |

**상태 코드**

| 코드 | 언제 |
|---|---|
| `400` | 입력이 규칙에 안 맞음 (30분 단위 아님, 2시간 초과) |
| `401` | 로그인 안 함 |
| `403` | 권한 없음 (남의 예약 퇴실) |
| `404` | 없음 |
| `409` | 지금 상태와 충돌 (겹침, 이미 퇴실, 대표자 1건) |

---

## 5. DB (PostgreSQL)

| 항목 | 규칙 | 예 |
|---|---|---|
| 이름 | snake_case. 테이블 이름은 `schema-v2.sql`을 따른다 | `reservation_participant` |
| 기본 키 | `id` (`bigint` identity) | |
| 외래 키 | `<대상>_id` | `room_id`, `owner_id` |
| 시각 | `timestamptz`, 이름은 `…_at` | `start_at`, `checked_out_at` |
| 참·거짓 | `is_…` | `is_active` |
| 상태 | 대문자 문자열 + CHECK | `'BOOKED'`, `'IN_USE'` |
| 제약 이름 | 무엇을 막는지 드러나게 | `no_room_overlap`, `one_active_per_owner` |
| 마이그레이션 | Flyway `V<번호>__<설명>.sql` | `V3__add_participant.sql` |

- **이미 합쳐진 마이그레이션 파일은 절대 고치지 않는다.** 바꿀 게 있으면 새 번호 파일을 만든다.
- JPA `ddl-auto`는 `validate`만 쓴다.

---

## 6. 프론트 (React · TypeScript)

| 항목 | 규칙 | 예 |
|---|---|---|
| 폴더 | 기능별 | `src/features/reservation/` |
| 컴포넌트 | PascalCase 파일·이름 | `ReservationForm.tsx` |
| 훅 | `use…` | `useMyReservations.ts` |
| API 타입 | OpenAPI에서 자동 생성(openapi-typescript). 손으로 쓰지 않는다 | `src/api/schema.d.ts` |
| 서버 상태 | TanStack Query | |
| 포맷 | Prettier + ESLint, 저장할 때 자동 | |
| 문구 | 사용자에게 보이는 말은 "퇴실", "취소"처럼 요구사항 용어 그대로 | |

---

## 7. Jetson (Python)

| 항목 | 규칙 |
|---|---|
| 버전 | Python 3.10 이상 |
| 이름 | 함수·변수 snake_case, 클래스 PascalCase (PEP 8) |
| 포맷 | `ruff format` + `ruff check` |
| 설정 | 서버 주소·토큰은 `.env`에서. 코드에 박지 않는다 |
| 서버 호출 | core API만 부른다. DB에 직접 붙지 않는다 |

---

## 8. 문서

| 항목 | 규칙 |
|---|---|
| 형식 | 마크다운. 그림은 Mermaid, 나머지는 표 |
| 분량 | 핵심만. 긴 설명보다 표 |
| 시간 범위 | `14:00–16:00` (`~`는 취소선으로 바뀌므로 쓰지 않는다) |
| 요구사항 | ID로 부른다 (`FR-RSV-001`) |
| 상태 표시 | `초안` / `확정` |
| 비밀값 | 비밀번호·계정·학번 목록은 문서와 저장소에 넣지 않는다 |

---

## 공통 — 절대 하지 않는 것

- 비밀번호·키·`.env`를 커밋하지 않는다. 저장소에는 `.env.example`만.
- `main`, `develop`에 직접 push하지 않는다.
- 합쳐진 Flyway 파일을 고치지 않는다.
- 테스트를 끄거나 건너뛰어서 빌드를 통과시키지 않는다.
