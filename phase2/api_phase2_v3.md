# Phase 2 API 명세서

**문서 식별자:** API-AIT-P2  
**버전:** 0.3  
**작성일:** 2026-05-10  
**대상 Phase:** Phase 2 (W11) — Walking Skeleton  
**범위:** FG-1 대화 작성 + FG-7 인증 + Phase 2 동작에 필요한 최소 보조 기능 (FR-4.1 루트 대화 생성, FR-6.2 대화 목록 제공, FR-10.1 LLM 선택, FR-2.3 턴 요약)  
**기준 ERD:** `ait_erd_v2`

---

## 0. 공통 컨벤션

### 0.1 Base URL

```
https://api.ait.example.com/api/v1
```

모든 엔드포인트는 `/api/v1/` 로 시작.

### 0.2 인증

- 인증 방식: **JWT Bearer Token (Access Token 단일 사용)**
- 헤더: `Authorization: Bearer <accessToken>`
- Access Token 만료: 12시간
- 인증 처리: Spring Security `SecurityFilterChain` + `JwtAuthenticationFilter`
- 미인증 엔드포인트 (permitAll): `/auth/signup`, `/auth/login`
- **Phase 2 결정:** Refresh Token 미사용. Token 만료 시 사용자가 재로그인. 후속 Phase 에서 도입 예정.

### 0.3 응답 형식 — `ApiResponse<T>` 단일 래퍼

성공/실패 모두 같은 구조로 응답한다. 클라이언트는 단일 타입(`ApiResponse<T>`)으로 모든 응답을 파싱 가능.

**스키마**

| 필드 | 타입 | 설명 |
|---|---|---|
| `isSuccess` | Boolean | 성공 여부 |
| `code` | String | 성공/에러 코드 (`BaseSuccessCode` / `BaseErrorCode` enum 값) |
| `message` | String | 사람이 읽을 수 있는 메시지 |
| `result` | T \| null | 실제 데이터 (실패 시 보통 null) |

**성공 응답 예시**
```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": { "memberId": 1, "email": "user@example.com" }
}
```

**에러 응답 예시**
```json
{
  "isSuccess": false,
  "code": "AUTH_4011",
  "message": "토큰이 유효하지 않습니다.",
  "result": null
}
```

**Java 측 사용**
```java
return ApiResponse.onSuccess(SuccessStatus.OK, dto);
return ApiResponse.onFailure(ErrorStatus.INVALID_TOKEN, null);
```

### 0.4 HTTP 상태 코드

모든 응답은 본문에 `ApiResponse<T>` 래퍼를 포함. (삭제 응답도 `result: null` 로 200 반환, 204 미사용)

| 코드 | 의미 | 사용 시점 |
|---|---|---|
| 200 | OK | 조회/수정/삭제 성공 |
| 201 | Created | 생성 성공 |
| 400 | Bad Request | 입력값 검증 실패 |
| 401 | Unauthorized | 토큰 없음/만료/변조 |
| 403 | Forbidden | 인증되었으나 권한 없음 |
| 404 | Not Found | 리소스 없음 |
| 409 | Conflict | 중복 (이메일 중복 등) |
| 500 | Internal Server Error | 서버 오류 |

### 0.5 공통 코드 목록

**SuccessStatus (성공 코드)**

| HTTP | code | message |
|---|---|---|
| 200 | `COMMON_200` | 요청에 성공하였습니다. |
| 201 | `COMMON_201` | 리소스가 생성되었습니다. |

**ErrorStatus (에러 코드)**

| HTTP | code | message |
|---|---|---|
| 400 | `COMMON_400` | 잘못된 요청입니다. |
| 400 | `COMMON_4001` | 입력값이 올바르지 않습니다. |
| 401 | `AUTH_4011` | 토큰이 유효하지 않습니다. |
| 401 | `AUTH_4012` | 이메일 또는 비밀번호가 일치하지 않습니다. |
| 401 | `AUTH_4013` | 토큰이 만료되었습니다. |
| 403 | `COMMON_403` | 접근 권한이 없습니다. |
| 404 | `COMMON_404` | 리소스를 찾을 수 없습니다. |
| 404 | `MEMBER_4041` | 존재하지 않는 회원입니다. |
| 404 | `CHAT_4041` | 존재하지 않는 대화입니다. |
| 404 | `MESSAGE_4041` | 존재하지 않는 메시지입니다. |
| 409 | `MEMBER_4091` | 이미 존재하는 이메일입니다. |
| 409 | `MESSAGE_4091` | 취소할 수 없는 상태의 메시지입니다. |
| 500 | `COMMON_500` | 서버 오류가 발생했습니다. |
| 500 | `LLM_5001` | LLM 서비스 호출에 실패했습니다. |

> 코드 체계: `<도메인>_<HTTP상태><일련번호>` (예: `AUTH_4011`, `MEMBER_4041`).  
> 일반 에러는 `COMMON_*`, 도메인 특화는 `MEMBER_*`, `AUTH_*`, `CHAT_*`, `LLM_*`, `MESSAGE_*`.

### 0.6 데이터 타입 표기

| 명세 표기 | ERD 컬럼 타입 | 직렬화 형식 |
|---|---|---|
| `Long` | `bigint` | JSON number |
| `String` | `varchar(N)`, `text` | JSON string |
| `String?` | nullable | string \| null |
| `DateTime` | `timestamp` | ISO 8601 UTC, `"2026-04-13T15:30:00Z"` |
| `Enum<...>` | `enum` | string (정해진 값 중 하나) |
| `Boolean` | `BOOLEAN` | `true` / `false` |
| `Integer` | `INT` | JSON number |

### 0.7 페이지네이션

본 명세는 두 가지 페이지네이션을 데이터 성격에 따라 다르게 사용한다.

**Offset 기반** — 대화 목록처럼 점프 이동이 의미 있는 경우
```
GET /chats?page=0&size=20&sort=updatedAt,desc
```

**Cursor 기반** — 채팅 메시지처럼 무한 스크롤 + 실시간 추가가 있는 경우
```
GET /chats/{id}/turns?lastTurnSequence=120&limit=20&direction=BACKWARD
```

- `lastTurnSequence` 는 마지막으로 받은 turn 의 `turn_sequence` 값. 첫 호출 시 생략.
- 첫 호출은 생략, 이후엔 응답의 `nextTurnSequence` 를 그대로 사용.
- `direction=BACKWARD` (과거로, 기본) / `direction=FORWARD` (미래로, 새 메시지 polling 용)

---

## 1. 인증 API (FG-7)

### 1.1 회원가입

```http
POST /api/v1/auth/signup
Content-Type: application/json
```

**인증 필요:** ❌

**요청 본문**

| 필드 | 타입 | 필수 | 제약 | 설명 |
|---|---|---|---|---|
| `email` | String | ✓ | varchar(100), 이메일 형식 | 중복 불가 |
| `password` | String | ✓ | 8자 이상, 영문+숫자 | 서버에서 hash 후 저장 |
| `name` | String | ✓ | varchar(20) | 표시 이름 |

```json
{
  "email": "user@example.com",
  "password": "secret1234",
  "name": "홍길동"
}
```

**응답 (201 Created)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `memberId` | Long | 생성된 회원 ID |
| `email` | String | |
| `name` | String | |
| `accessToken` | String | JWT 액세스 토큰 (12시간 유효) |

```json
{
  "isSuccess": true,
  "code": "COMMON_201",
  "message": "리소스가 생성되었습니다.",
  "result": {
    "memberId": 1,
    "email": "user@example.com",
    "name": "홍길동",
    "accessToken": "eyJhbGciOiJIUzI1NiJ9..."
  }
}
```

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `COMMON_4001` | 이메일 형식 잘못, 비밀번호 짧음 |
| 409 | `MEMBER_4091` | 이메일 이미 존재 |

**관련 FR:** FR-7.1

---

### 1.2 로그인

```http
POST /api/v1/auth/login
Content-Type: application/json
```

**인증 필요:** ❌

**요청 본문**

| 필드 | 타입 | 필수 |
|---|---|---|
| `email` | String | ✓ |
| `password` | String | ✓ |

```json
{
  "email": "user@example.com",
  "password": "secret1234"
}
```

**응답 (200 OK)**

| 필드 | 타입 | 설명 |
|---|---|---|
| `memberId` | Long | |
| `accessToken` | String | JWT (12시간 유효) |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "memberId": 1,
    "accessToken": "eyJhbGciOiJIUzI1NiJ9..."
  }
}
```

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4012` | 이메일/비번 불일치 또는 존재하지 않는 이메일 |

> 보안상 "존재하지 않는 이메일" 과 "비밀번호 틀림" 을 구분하지 않고 동일 메시지로 응답.

**관련 FR:** FR-7.1


---

### 1.3 회원 탈퇴

```http
DELETE /api/v1/members/me
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

**요청 본문:** 없음

**응답 (200 OK)**

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": null
}
```

> 소프트 삭제 — `member.deleted_at` 채움 (ERD: `null → 탈퇴 X, 값 존재 → 탈퇴 O`).  
> 일정 기간 후 배치잡으로 실제 삭제 (FR-7.3).

**관련 FR:** FR-7.3

---

### 1.4 내 정보 조회

```http
GET /api/v1/members/me
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

**응답 (200 OK)**

| 필드 | 타입 | 설명 |
|---|---|---|
| `memberId` | Long | |
| `email` | String | |
| `name` | String | |
| `profileUrl` | String? | 프로필 이미지 URL |
| `type` | Enum<"USER" \| "ADMIN"> | 회원 종류 |
| `createdAt` | DateTime | |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "memberId": 1,
    "email": "user@example.com",
    "name": "김도윤",
    "profileUrl": null,
    "type": "USER",
    "createdAt": "2026-04-13T10:00:00Z"
  }
}
```

---

## 2. 대화 API (FG-1, FR-4.1, FR-6.2)

### 2.1 새 대화 시작

```http
POST /api/v1/chats
Authorization: Bearer <accessToken>
Content-Type: application/json
```

**인증 필요:** ✓

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `title` | String? | | 미지정 시 "제목없음" (ERD default) |
| `llmProvider` | Enum<"OPENAI" \| "GOOGLE" \| "ANTHROPIC"> | ✓ | LLM 제공자 |
| `llmModel` | Enum<"gpt-4o-mini" \| "gemini-2.0-flash" \| "claude-3.5-sonnet"> | ✓ | 모델 식별자 |

```json
{
  "title": "Spring Security 학습",
  "llmProvider": "OPENAI",
  "llmModel": "gpt-4o-mini"
}
```

**응답 (201 Created)**

| 필드 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | `aichat_session_id` |
| `rootChatId` | Long | 분기 트리 묶음 ID (Phase 2 에선 자기 자신과 동일) |
| `parentId` | Long? | NULL → 루트 세션 (Phase 2 에선 항상 null) |
| `title` | String | |
| `llmProvider` | Enum | |
| `llmModel` | Enum | |
| `createdAt` | DateTime | |
| `updatedAt` | DateTime | |

```json
{
  "isSuccess": true,
  "code": "COMMON_201",
  "message": "리소스가 생성되었습니다.",
  "result": {
    "chatId": 42,
    "rootChatId": 42,
    "parentId": null,
    "title": "Spring Security 학습",
    "llmProvider": "OPENAI",
    "llmModel": "gpt-4o-mini",
    "createdAt": "2026-05-06T11:00:00Z",
    "updatedAt": "2026-05-06T11:00:00Z"
  }
}
```

> Phase 2 에선 루트 세션만 생성 가능 (`parent_id = null`, `branch_point_turn_id = null`).  
> `rootChatId` 는 자기 자신의 `chatId` 로 채움.  
> 분기 생성 API 는 Phase 3 에서 추가.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `COMMON_4001` | llmProvider / llmModel 미지원 값 |
| 401 | `AUTH_4011` | |

**관련 FR:** FR-4.1, FR-10.1

---

### 2.2 대화 목록 조회

```http
GET /api/v1/chats?page=0&size=20&sort=updatedAt,desc
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

**페이지네이션:** Offset 기반 (정렬 옵션 다양함)

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `page` | Integer | 0 | 페이지 번호 |
| `size` | Integer | 20 | 페이지 크기 (1~50) |
| `sort` | String | `updatedAt,desc` | 정렬 (FR-6.2) |

**정렬 옵션**
- `updatedAt,desc` (최근 활동순)
- `createdAt,desc` (생성순)
- `title,asc` (이름순)

**응답 (200 OK)**

`result.content` 의 각 항목:

| 필드 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |
| `title` | String | |
| `llmProvider` | Enum | |
| `llmModel` | Enum | |
| `createdAt` | DateTime | |
| `updatedAt` | DateTime | |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "content": [
      {
        "chatId": 42,
        "title": "Spring Security 학습",
        "llmProvider": "OPENAI",
        "llmModel": "gpt-4o-mini",
        "createdAt": "2026-05-06T11:00:00Z",
        "updatedAt": "2026-05-06T11:30:00Z"
      },
      {
        "chatId": 41,
        "title": "분기 그래프 설계 논의",
        "llmProvider": "ANTHROPIC",
        "llmModel": "claude-3.5-sonnet",
        "createdAt": "2026-05-05T14:20:00Z",
        "updatedAt": "2026-05-05T16:45:00Z"
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 47,
    "totalPages": 3
  }
}
```

> Phase 2 에선 루트 세션만 반환 (`parent_id IS NULL` 필터).  
> 자기 소유 대화만 (`member_id = currentMember`, `deleted_at IS NULL`).

**관련 FR:** FR-6.2, NFR-S-2

---

### 2.3 대화 메타정보 조회

```http
GET /api/v1/chats/{chatId}
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | `aichat_session_id` |

**응답 (200 OK)**

대화 자체의 메타정보만 반환. 턴 목록은 별도 엔드포인트 (`/chats/{id}/turns`) 사용.

| 필드 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |
| `rootChatId` | Long | |
| `parentId` | Long? | Phase 2 에선 항상 null |
| `title` | String | |
| `llmProvider` | Enum | |
| `llmModel` | Enum | |
| `createdAt` | DateTime | |
| `updatedAt` | DateTime | |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "chatId": 42,
    "rootChatId": 42,
    "parentId": null,
    "title": "Spring Security 학습",
    "llmProvider": "OPENAI",
    "llmModel": "gpt-4o-mini",
    "createdAt": "2026-05-06T11:00:00Z",
    "updatedAt": "2026-05-06T11:30:00Z"
  }
}
```

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 대화 |
| 404 | `CHAT_4041` | 존재하지 않는 chatId |

**관련 FR:** FR-6.1

---

### 2.4 대화 턴 목록 조회 (Cursor 기반 페이징)

```http
GET /api/v1/chats/{chatId}/turns?lastTurnSequence={seq}&limit=20&direction=BACKWARD
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

**페이지네이션:** Cursor 기반 (채팅 무한 스크롤 + 실시간 추가)

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `lastTurnSequence` | Integer? | null | 마지막으로 받은 turn 의 `turn_sequence` 값. 첫 호출 시 생략. |
| `limit` | Integer | 20 | 한 페이지 크기 (1~50) |
| `direction` | Enum<"BACKWARD" \| "FORWARD"> | BACKWARD | BACKWARD: 과거 턴 (무한 스크롤) / FORWARD: 미래 턴 (새 메시지 polling) |

**호출 패턴**

| 시나리오 | 호출 방식 |
|---|---|
| 채팅 첫 진입 | `cursor` 생략, `direction=BACKWARD` → 최신 20턴 받음 |
| 위로 스크롤 (이전 보기) | 받은 응답의 `nextTurnSequence` 를 `lastTurnSequence` 로 보냄 |
| 새 메시지 동기화 | 가장 최신 turn 의 `turnSequence` 를 `lastTurnSequence` 로 + `direction=FORWARD` |

**응답 (200 OK)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `turns` | Array<TurnDto> | 턴 목록 (BACKWARD 시 turn_sequence 내림차순) |
| `nextTurnSequence` | Integer? | 다음 호출 시 보낼 `lastTurnSequence` 값 (`hasMore=false` 면 null) |
| `hasMore` | Boolean | 더 가져올 페이지 존재 여부 |

**TurnDto**

| 필드 | 타입 | 설명 |
|---|---|---|
| `turnId` | Long | |
| `turnSequence` | Integer | 분기 내 순서 |
| `summary` | String | 턴 요약 (FR-2.3, default "제목없음") |
| `messages` | Array<MessageDto> | user 메시지 1 + ai 메시지 0~1 |
| `createdAt` | DateTime | |

**MessageDto**

| 필드 | 타입 | 설명 |
|---|---|---|
| `messageId` | Long | |
| `senderType` | Enum<"USER" \| "AI"> | |
| `status` | Enum<"STREAMING" \| "COMPLETED" \| "CANCELED" \| "FAILED"> | FR-1.3 |
| `content` | String? | 응답 취소/실패 시 부분 텍스트일 수 있음 |
| `promptToken` | Integer? | user 메시지에만 |
| `answerToken` | Integer? | AI 메시지에만 |
| `createdAt` | DateTime | |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "turns": [
      {
        "turnId": 150,
        "turnSequence": 150,
        "summary": "Spring Security 필터 체인 동작 원리",
        "messages": [
          {
            "messageId": 300,
            "senderType": "USER",
            "status": "COMPLETED",
            "content": "Spring Security 필터 체인이 어떻게 동작하나요?",
            "promptToken": 24,
            "answerToken": null,
            "createdAt": "2026-05-06T11:01:00Z"
          },
          {
            "messageId": 301,
            "senderType": "AI",
            "status": "COMPLETED",
            "content": "Spring Security는 ServletFilter 체인의 형태로...",
            "promptToken": null,
            "answerToken": 412,
            "createdAt": "2026-05-06T11:01:08Z"
          }
        ],
        "createdAt": "2026-05-06T11:01:00Z"
      },
      {
        "turnId": 149,
        "turnSequence": 149,
        "summary": "JWT 토큰 만료 처리",
        "messages": [...]
      }
    ],
    "nextTurnSequence": 149,
    "hasMore": true
  }
}
```

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `COMMON_4001` | lastTurnSequence 가 음수 / limit 범위 초과 |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 대화 |
| 404 | `CHAT_4041` | 존재하지 않는 chatId |

**관련 FR:** FR-6.1, NFR-P-1, NFR-P-3

---

### 2.5 메시지 송신 (스트리밍 응답)

```http
POST /api/v1/chats/{chatId}/messages
Authorization: Bearer <accessToken>
Accept: text/event-stream
Content-Type: application/json
```

**인증 필요:** ✓

**중요:** 이 엔드포인트는 **Server-Sent Events (SSE)** 로 응답.  
일반 JSON `ApiResponse<T>` 가 아니라 chunk 단위 스트림 (FR-1.2 점진적 표시).
AI 응답이 COMPLETED, CANCELED, FAILED 중 하나로 종료되면 해당 chat의 updatedAt을 갱신한다. 이를 통해 대화 목록의 최근 활동순 정렬이 유지된다.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |

**요청 본문**

| 필드 | 타입 | 필수 |
|---|---|---|
| `content` | String | ✓ |

```json
{
  "content": "Spring Security 필터 체인이 어떻게 동작하나요?"
}
```

**응답 (200 OK, `Content-Type: text/event-stream`)**

SSE 이벤트 시퀀스 — `message.status` 변화와 정확히 매핑됨:

```
event: turn_started
data: {"turnId": 100, "userMessageId": 200, "aiMessageId": 201}

event: chunk
data: {"text": "Spring Security는 "}

event: chunk
data: {"text": "ServletFilter "}

... (반복) ...

event: turn_completed
data: {"aiMessageId": 201, "summary": "Spring Security 필터 체인 동작 원리", "answerToken": 412}
```

**SSE 이벤트 타입 ↔ message.status 매핑**

| event | data 스키마 | DB 상태 변화 |
|---|---|---|
| `turn_started` | `{turnId: Long, userMessageId: Long, aiMessageId: Long}` | user 메시지 `COMPLETED`, AI 메시지 `STREAMING` 으로 생성 |
| `chunk` | `{text: String}` | content 누적 (status 그대로) |
| `turn_completed` | `{aiMessageId: Long, summary: String, answerToken: Integer}` | AI 메시지 `STREAMING` → `COMPLETED` |
| `error` | `{code: String, message: String}` | AI 메시지 `STREAMING` → `FAILED` |

**스트리밍 시작 전 에러**

인증 실패, chatId 없음 등 SSE 시작 *전* 에러는 일반 `ApiResponse<T>` 로 반환:

```json
{
  "isSuccess": false,
  "code": "CHAT_4041",
  "message": "존재하지 않는 대화입니다.",
  "result": null
}
```

**응답 취소:** 별도 cancel API 사용 (§2.6 참조). SSE abort 만으로는 처리하지 않음.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `COMMON_4001` | content 빈 문자열 |
| 401 | `AUTH_4011` | 인증 실패 |
| 403 | `COMMON_403` | 다른 사용자의 대화 |
| 404 | `CHAT_4041` | 존재하지 않는 chatId |
| stream | `LLM_5001` | LLM API 호출 실패 (event: error) |

**관련 FR:** FR-1.1, FR-1.2, FR-6.1
**기반 제공** FR-2.3, FR-5.1

---

### 2.6 응답 생성 취소

```http
POST /api/v1/chats/{chatId}/messages/{messageId}/cancel
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

스트리밍 중인 AI 메시지의 생성을 취소한다. 클라이언트 흐름:
1. 사용자가 취소 버튼 클릭
2. 이 API 호출 → 서버가 LLM stream cancel + 메시지 `status=CANCELED` 로 갱신
3. 클라이언트가 SSE 연결도 종료 (`AbortController.abort()`)
messageId가 실제로 chatId에 속한 메시지인지 확인 필요 

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |
| `messageId` | Long | 취소할 AI 메시지 ID (`turn_started` 이벤트에서 받은 `aiMessageId`) |

**요청 본문:** 없음

**응답 (200 OK)**

| 필드 | 타입 | 설명 |
|---|---|---|
| `messageId` | Long | |
| `status` | Enum | 취소 후 상태 (`CANCELED`) |
| `content` | String? | 취소 시점까지 누적된 부분 텍스트 |
| `answerToken` | Integer? | 취소 시점까지의 토큰 수 |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "messageId": 201,
    "status": "CANCELED",
    "content": "Spring Security는 ServletFilter 체인의 형태로 동작하며, 각 필터는...",
    "answerToken": 89
  }
}
```

**상태 전이 규칙**

| 현재 status | cancel 호출 시 동작 |
|---|---|
| `STREAMING` | → `CANCELED` 로 변경, 부분 content 보존 |
| `COMPLETED` | 409 `MESSAGE_4091` (이미 완료) |
| `FAILED` | 409 `MESSAGE_4091` (이미 실패) |
| `CANCELED` | 200 OK (idempotent — 이미 취소됨) |

**에러**

messageId가 존재하더라도 해당 chatId에 속하지 않으면 MESSAGE_4041 또는 COMMON_403으로 처리한다.

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 메시지 |
| 404 | `MESSAGE_4041` | 존재하지 않는 messageId |
| 409 | `MESSAGE_4091` | COMPLETED/FAILED 상태에서 cancel 시도 |

**관련 FR:** FR-1.3

---

### 2.7 대화 삭제

```http
DELETE /api/v1/chats/{chatId}
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

> Phase 2 보조 기능. SRS 상의 FR-2.5 분기 삭제와는 별개이며, 분기 삭제는 Phase 3에서 별도 설계한다. 
> Phase 2 선택 구현 API. 시간이 부족한 경우 구현 대상에서 제외 가능

**응답 (200 OK)**

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": null
}
```

> 소프트 삭제 — `chat.deleted_at` 채움.  
> Phase 3 에서 분기 도입 후 자손 분기까지 cascade 처리.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 대화 |
| 404 | `CHAT_4041` | |

---

## 3. 정리 — Phase 2 엔드포인트 목록

| Method | Path | 설명 | 인증 | FR |
|---|---|---|---|---|
| POST | `/api/v1/auth/signup` | 회원가입 | ❌ | FR-7.1 |
| POST | `/api/v1/auth/login` | 로그인 | ❌ | FR-7.1 |
| GET | `/api/v1/members/me` | 내 정보 조회 | ✓ | — |
| DELETE | `/api/v1/members/me` | 회원 탈퇴 | ✓ | FR-7.3 |
| POST | `/api/v1/chats` | 새 대화 시작 | ✓ | FR-4.1, FR-10.1 |
| GET | `/api/v1/chats` | 대화 목록 (offset 페이징) | ✓ | FR-6.2 |
| GET | `/api/v1/chats/{id}` | 대화 메타정보 | ✓ | FR-6.1 |  
| GET | `/api/v1/chats/{id}/turns` | 턴 목록 (cursor 페이징) | ✓ | FR-6.1 |
| POST | `/api/v1/chats/{id}/messages` | 메시지 송신 (SSE) | ✓ | FR-1.1, 1.2, 6.1 |
| POST | `/api/v1/chats/{id}/messages/{mid}/cancel` | 응답 취소 | ✓ | FR-1.3 |
| DELETE | `/api/v1/chats/{id}` | 대화 삭제 (Phase 2 보조) | ✓ | — |

총 11개 엔드포인트.

---

## 4. ERD ↔ API 필드 매핑 정리

검토자가 ERD와 API 명세 일관성을 확인할 수 있도록.

### chat 테이블 → 응답 DTO

| ERD 컬럼 | API 필드 | 비고 |
|---|---|---|
| `aichat_session_id` | `chatId` | API에선 외부 노출 이름 단순화 |
| `root_chat_id` | `rootChatId` | Phase 2 에선 자기자신=chatId |
| `parent_id` | `parentId` | Phase 2 에선 항상 null |
| `branch_point_turn_id` | (Phase 2 미사용) | Phase 3 분기 생성 후 사용 |
| `title` | `title` | |
| `llm_provider` | `llmProvider` | enum: OPENAI, GOOGLE, ANTHROPIC |
| `llm_model` | `llmModel` | enum: gpt-4o-mini, gemini-2.0-flash, claude-3.5-sonnet |
| `created_at` | `createdAt` | |
| `updated_at` | `updatedAt` | |
| `deleted_at` | (응답 미포함) | 서버 내부 필터링용 |
| `member_id` | (응답 미포함) | 인증으로 자동 처리 |

### turn 테이블 → 응답 DTO

| ERD 컬럼 | API 필드 |
|---|---|
| `turn_id` | `turnId` |
| `turn_sequence` | `turnSequence` |
| `Summary` | `summary` |
| `created_at` | `createdAt` |
| `aichat_session_id` | (부모 chat에 포함되어 응답에 미포함) |

### message 테이블 → 응답 DTO

| ERD 컬럼 | API 필드 |
|---|---|
| `message_id` | `messageId` |
| `sender_type` | `senderType` (USER, AI) |
| `status` | `status` (STREAMING, COMPLETED, CANCELED, FAILED) |
| `content` | `content` |
| `prompt_token` | `promptToken` |
| `answer_token` | `answerToken` |
| `created_at` | `createdAt` |
| `turn_id` | (부모 turn에 포함되어 응답에 미포함) |

### member 테이블 → 응답 DTO

| ERD 컬럼 | API 필드 |
|---|---|
| `member_id` | `memberId` |
| `email` | `email` |
| `name` | `name` |
| `profile_url` | `profileUrl` |
| `type` | `type` (USER, ADMIN) |
| `created_at` | `createdAt` |
| `password` | (응답에 절대 미포함) |
| `deleted_at` | (서버 내부 필터링용) |

---

## 5. Phase 3 에서 추가될 API (예고)

Phase 2 명세 범위 밖이지만 설계 일관성을 위해 미리 식별:

- `POST /api/v1/chats/{id}/branches` — 분기 생성 (FR-2.1)
- `DELETE /api/v1/chats/{id}/branches/{branchId}` — 분기 삭제 (FR-2.5)
- `POST /api/v1/chats/{id}/messages/{messageId}/regenerate` — 응답 재생성 (FR-1.4)
- `PATCH /api/v1/chats/{id}/messages/{messageId}` — 사용자 메시지 수정 (FR-1.5)
- `GET /api/v1/chats/{id}/graph` — 분기 그래프 구조 (FR-3.1)

이들은 Phase 3 시작 시 별도 명세 절로 추가.

---

## 부록 A. SSE + Cancel 클라이언트 구현 예시 (참고)

```javascript
import { fetchEventSource } from '@microsoft/fetch-event-source';

let currentAiMessageId = null;
const ctrl = new AbortController();

await fetchEventSource(`/api/v1/chats/42/messages`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ content: userInput }),
  signal: ctrl.signal,
  onmessage(ev) {
    const data = JSON.parse(ev.data);
    if (ev.event === 'turn_started') {
      currentAiMessageId = data.aiMessageId;
      initMessage(data.aiMessageId);
    } else if (ev.event === 'chunk') {
      appendText(data.text);
    } else if (ev.event === 'turn_completed') {
      finalizeMessage(data.aiMessageId, data.summary);
    } else if (ev.event === 'error') {
      showError(data.message);
    }
  },
});

// 사용자가 취소 버튼 클릭 (FR-1.3)
cancelButton.onclick = async () => {
  // 1. 서버에 명시적 cancel 호출
  await fetch(`/api/v1/chats/42/messages/${currentAiMessageId}/cancel`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${accessToken}` },
  });
  // 2. SSE 연결도 끊기
  ctrl.abort();
};
```

---


## 부록 C. 변경 이력

| 버전 | 날짜 | 변경 사유 |
|---|---|---|
| 0.1 | 2026-05-06 | 초안 |
| 0.2 | 2026-05-10 | 실제 ERD(`ait_erd_v2`) 반영 + `ApiResponse<T>` 단일 래퍼 패턴 |
| 0.3 | 2026-05-10 | 리뷰 반영 |

**v0.2 → v0.3 주요 변경:**

| # | 항목 | 변경 |
|---|---|---|
| 1 | 범위 표기 | 정직하게 "Phase 2 핵심 + 동작에 필요한 최소 보조 기능" 으로 확장 명시 |
| 2 | DELETE /chats/{id} | FR-2.5 매핑 제거. "Phase 2 보조, 분기 삭제는 Phase 3" 명시 |
| 3 | 대화 조회 분리 | `GET /chats/{id}` 메타만 + `GET /chats/{id}/turns` cursor 페이징 신설 |
| 4 | Refresh Token 제거 | `/auth/refresh` 삭제, Token Rotation 설명 제거, access token 만료 12시간으로 변경 |
| 5 | 204 No Content 제거 | ApiResponse 래퍼와 충돌 해소, 모든 응답을 200 + result null 로 통일 |
| 6 | 로그인 에러 통일 | 존재하지 않는 이메일도 401 AUTH_4012 로 통일 (보안) |
| 7 | Cancel API 추가 | `POST /chats/{id}/messages/{mid}/cancel` 신설. SSE abort 만 의존하지 않음 |
| 추가 | 엔드포인트 표기 | 모든 API 에 전체 URL + 헤더 명시 (복붙용) |
