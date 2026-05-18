# Phase 3 API 명세서

**문서 식별자:** API-AIT-P3  
**버전:** 0.3  
**작성일:** 2026-05-18 
**대상 Phase:** Phase 3 (W12~W13) — 분기 관리 + 그래프 시각화  
**범위:** FG-2 분기 관리, FG-3 그래프 시각화, FR-1.4 응답 재생성, FR-1.5 사용자 메시지 수정  
**기준 ERD:** `ait_erd_v2.1` (Phase 3 보강 반영)  
**선행 Phase:** Phase 2 (Walking Skeleton — FG-1, FG-7)

---

## 0. 공통 컨벤션

Phase 2 명세서(`API-AIT-P2`)의 §0 공통 컨벤션을 그대로 따른다. 본 절은 Phase 3에서 추가되거나 변경되는 항목만 명시한다.

### 0.1 변경 없음

다음 사항은 Phase 2와 동일하다.

- Base URL: `/api/v1`
- 인증: JWT Bearer Token
- 응답 래퍼: `ApiResponse<T>` (`isSuccess`, `code`, `message`, `result`)
- HTTP 상태 코드 정책: 204 미사용, 삭제도 200 + `result: null`
- 데이터 타입 표기 (`Long`, `String`, `DateTime`, `Enum`)
- Offset / Cursor 페이지네이션 사용 규칙

### 0.2 Phase 3에서 추가되는 에러 코드

| HTTP | code | message |
|---|---|---|
| 400 | `BRANCH_4001` | 분기를 생성할 수 없는 turn입니다. |
| 400 | `GRAPH_4001` | 유효하지 않은 그래프 조회 파라미터입니다. |
| 404 | `BRANCH_4041` | 존재하지 않는 분기입니다. |
| 404 | `TURN_4041` | 존재하지 않는 turn입니다. |
| 409 | `BRANCH_4091` | 이미 삭제된 분기입니다. |
| 409 | `BRANCH_4092` | 복구 기간이 만료된 분기입니다. |
| 409 | `MESSAGE_4092` | 수정/재생성할 수 없는 상태의 메시지입니다. |

기존 `CHAT_4041`, `MESSAGE_4041`, `MESSAGE_4091` 등은 Phase 2와 동일하게 사용한다.

### 0.3 본 명세에서 사용하는 핵심 용어

| 용어 | 정의 |
|---|---|
| chat (session) | 하나의 분기 또는 루트 대화. ERD `chat` 엔티티에 대응. |
| turn | 사용자 입력 1건 + LLM 응답 1건의 단위. |
| message | turn 안의 개별 발화. `USER` 또는 `ASSISTANT`. |
| center | 그래프 조회 시 윈도우의 기준이 되는 turn. 사용자가 현재 보고 있는 turn. |
| frontier | window 경계에서 더 펼칠 노드가 남아 있는 지점. |
| branch point | 자식 chat의 시작 지점이 되는 부모 chat의 turn. |

---

## 0.5 설계 원칙

Phase 3은 다음 네 가지 원칙 위에서 설계된다.

1. **chat 골격은 항상 전체 반환** — 그래프 조회 시 root chat에 속한 모든 분기 메타데이터(`parent`, `branchPointTurn`)를 전체 포함한다. 사이드바·미니맵의 일관성과 FR-3.4(현재 위치 표시)를 위함이며, 메타만이라 페이로드는 작다.
2. **turn 노드는 center 기준 window만 반환** — `centerTurnId`를 중심으로 위/아래 N홉까지만 직렬화한다. `UP` 방향은 root까지의 ancestor path 추적(단일 경로), `DOWN` 방향은 BFS로 자손 분기까지 탐색(다중 경로).
3. **구조 가공은 조회 시점, AI 가공은 저장 시점 또는 비동기** — DTO 변환·window 계산·frontier 계산·`isBranchPoint` 마킹은 조회 시점에 수행한다. LLM 호출이 필요한 작업(turn summary, 분기 제목 자동 명명)은 저장 직후 비동기로 수행하며, 조회 응답 시간(NFR-P-2)을 지연시키지 않는다.
4. **본문은 lazy loading** — graph API는 노드 메타와 summary까지만 반환한다. 메시지 본문은 사용자가 해당 turn으로 이동할 때 Phase 2의 `GET /chats/{id}/turns`로 별도 조회한다.

---

## 1. ERD v2.1 변경분

Phase 3에서 ERD v2 대비 다음 변경분이 적용된다. 엔티티 관계는 그대로이며 컬럼·인덱스만 추가된다.

```sql
-- chat 테이블 보강
ALTER TABLE chat
  ADD COLUMN last_turn_id BIGINT NULL,
  ADD COLUMN title_status ENUM('PENDING', 'GENERATED', 'USER_EDITED')
    NOT NULL DEFAULT 'PENDING';

-- turn 요약 상태 정책 정리
-- Summary는 요약 생성 전 상태를 표현하기 위해 NULL 허용.
-- 화면 기본 문구("제목없음", "요약 생성 중")는 클라이언트/DTO에서 처리하고 DB에는 저장하지 않는다.
ALTER TABLE turn
  MODIFY COLUMN Summary TEXT NULL;

-- usage_record 테이블 보강 (이력 보존)
-- 신규 컬럼은 마이그레이션 호환성을 위해 NULL/DEFAULT 허용, 신규 row 생성 시에는 반드시 채움.
ALTER TABLE usage_record
  ADD COLUMN token_limit BIGINT NOT NULL DEFAULT 0,
  ADD COLUMN plan_id BIGINT NULL,
  ADD CONSTRAINT fk_usage_record_plan
    FOREIGN KEY (plan_id) REFERENCES plan(plan_id);

-- 인덱스 (NFR-P-2 보장)
CREATE INDEX idx_chat_root_deleted  ON chat(root_chat_id, deleted_at);
CREATE INDEX idx_chat_parent        ON chat(parent_id);
CREATE INDEX idx_chat_branch_point  ON chat(branch_point_turn_id);
CREATE INDEX idx_turn_chat_sequence ON turn(aichat_session_id, turn_sequence);
CREATE INDEX idx_message_turn       ON message(turn_id);
```

| 변경 | 근거 |
|---|---|
| `chat.last_turn_id` | FR-6.2 정렬 최적화. 사이드바 미리보기는 last_turn의 summary 재사용 (별도 preview 컬럼 미도입). |
| `chat.title_status` | FR-2.2 + 비동기 자동 명명 도입에 따른 상태 구분. |
| `turn.Summary` nullable 정책 | summary가 NULL이면 요약 생성 전(PENDING) 상태. DB 기본 문구 저장 금지. |
| `usage_record.token_limit` | 그 기간 시점의 토큰 한도 스냅샷. 플랜 정책 변경·사용자 플랜 변경 이력 보존. |
| `usage_record.plan_id` | 해당 기간에 적용된 플랜 추적. subscription 이력을 거치지 않고 직접 조회. |
| `idx_turn_chat_sequence` | window 조회의 핵심 경로. 필수. |
| 나머지 인덱스 | NFR-P-2 3초 이내 보장. |

> frontier 및 edge는 DB에 저장하지 않는다. 둘 다 조회 시점 계산값이다.

> `turn.Summary`는 nullable로 관리한다. `summary IS NULL`이면 요약 생성 전 상태이며, API 응답의 `summaryStatus`는 `PENDING`이다. "제목없음" 같은 화면용 기본 문구는 클라이언트 또는 DTO 조립 단계에서만 사용하고 DB에는 저장하지 않는다.

---

## 2. 분기 관리 API (FG-2)

### 2.1 분기 생성

```http
POST /api/v1/chats/{chatId}/branches
Authorization: Bearer <accessToken>
Content-Type: application/json
```

**인증 필요:** ✓

지정한 turn에서 새 분기를 생성한다. 생성 응답은 즉시 반환되며, 분기 제목 자동 명명(FR-2.2)은 비동기로 수행된다.

> Phase 2 §5 예고와 동일한 경로. `chatId`는 분기점이 속한 chat의 ID이며, root chat이 아닐 수도 있다 (분기에서 또 분기 가능).

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | 분기점이 속한 부모 chat의 ID |

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `branchPointTurnId` | Long | ✓ | 분기 시작 지점 turn |
| `title` | String? | | 사용자가 직접 지정하면 즉시 적용 (`titleStatus = USER_EDITED`). null이면 임시명 "새 분기" + 비동기 명명 |

```json
{
  "branchPointTurnId": 105,
  "title": null
}
```

**응답 (201 Created)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | 새로 생성된 분기의 chatId |
| `rootChatId` | Long | 분기 트리의 루트 (부모로부터 상속) |
| `parentId` | Long | 부모 chat의 ID (= 요청 path의 `chatId`) |
| `branchPointTurnId` | Long | 분기점 turn |
| `title` | String | `titleStatus`에 따라 임시명 또는 사용자 지정명 |
| `titleStatus` | Enum<"PENDING" \| "GENERATED" \| "USER_EDITED"> | |
| `llmProvider` | Enum | 부모 chat에서 상속 |
| `llmModel` | Enum | 부모 chat에서 상속 |
| `createdAt` | DateTime | |
| `updatedAt` | DateTime | |

```json
{
  "isSuccess": true,
  "code": "COMMON_201",
  "message": "리소스가 생성되었습니다.",
  "result": {
    "chatId": 87,
    "rootChatId": 42,
    "parentId": 42,
    "branchPointTurnId": 105,
    "title": "새 분기",
    "titleStatus": "PENDING",
    "llmProvider": "OPENAI",
    "llmModel": "gpt-4o-mini",
    "createdAt": "2026-05-17T13:35:00Z",
    "updatedAt": "2026-05-17T13:35:00Z"
  }
}
```

**처리 흐름**

동기 처리(응답 전):
1. `branchPointTurnId`의 존재·소유권 검증
2. 새 `chat` row 생성 — `parent_id`, `branch_point_turn_id`, `root_chat_id` 설정
3. `title_status = PENDING`, `title = "새 분기"` (사용자 지정 시 즉시 `USER_EDITED`)
4. 응답 반환

비동기 처리(응답 후):
1. 분기점 turn의 summary·문맥을 LLM에 전달
2. 분기 제목 생성
3. 업데이트 직전 `title_status`를 다시 확인
4. `title_status = PENDING`인 경우에만 `title`, `title_status = GENERATED`로 갱신
5. 이미 `USER_EDITED`이면 사용자가 직접 수정한 제목을 보호하기 위해 아무 작업도 수행하지 않음

```sql
UPDATE chat
SET title = ?, title_status = 'GENERATED'
WHERE aichat_session_id = ?
  AND title_status = 'PENDING';
```

> 첫 메시지 송신은 Phase 2 `POST /api/v1/chats/{chatId}/messages`를 그대로 사용한다 (`chatId`에 새 분기 ID 전달). 분기 생성과 메시지 송신을 하나의 API로 합치지 않는다 — Phase 2의 "대화 생성 + 첫 메시지" 흐름과 동일한 2단계 패턴.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `BRANCH_4001` | 삭제된 chat에서 분기 시도, 또는 branchPointTurnId가 해당 chat에 속하지 않음 |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 chat |
| 404 | `CHAT_4041` | 존재하지 않는 chatId |
| 404 | `TURN_4041` | 존재하지 않는 branchPointTurnId |

**관련 FR:** FR-2.1, FR-2.2

---

### 2.2 분기 정보 수정

```http
PATCH /api/v1/chats/{chatId}
Authorization: Bearer <accessToken>
Content-Type: application/json
```

**인증 필요:** ✓

분기 제목을 사용자가 직접 수정한다. 루트 chat의 제목 수정에도 사용 가능.

> Phase 2에서는 `PATCH /chats/{id}`가 정의되지 않았으므로 Phase 3에서 신설.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `title` | String | ✓ | 새 제목 (1~100자) |

```json
{
  "title": "비기능 요구사항 심화 - v2"
}
```

**응답 (200 OK)**

`result` 필드는 §2.1의 응답 본문과 동일한 chat 정보. `titleStatus`는 항상 `USER_EDITED`로 잠긴다.

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "chatId": 87,
    "rootChatId": 42,
    "parentId": 42,
    "branchPointTurnId": 105,
    "title": "비기능 요구사항 심화 - v2",
    "titleStatus": "USER_EDITED",
    "llmProvider": "OPENAI",
    "llmModel": "gpt-4o-mini",
    "createdAt": "2026-05-17T13:35:00Z",
    "updatedAt": "2026-05-17T13:40:00Z"
  }
}
```

> `titleStatus`가 `USER_EDITED`인 chat은 비동기 명명 워커가 건너뛴다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `COMMON_4001` | title 길이 위반 |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 chat |
| 404 | `CHAT_4041` | |

**관련 FR:** FR-2.2

---

### 2.3 분기 삭제

```http
DELETE /api/v1/chats/{chatId}
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

소프트 삭제를 수행한다. `chat.deleted_at`을 기록하며, **자손 분기도 cascade 처리**된다.

> Phase 2에서 동일 경로가 보조 기능으로 정의되어 있었다. Phase 3에서 FR-2.5의 정식 구현으로 승격. 가장 큰 변경은 **자손 cascade 처리**.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |

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

**처리 규칙**

1. 대상 chat 및 모든 자손 chat에 대해 `deleted_at`을 동일한 시각으로 채운다.
2. 삭제된 chat의 turn·message는 별도 삭제하지 않는다 (FR-6.1 데이터 보존).
3. 복구 가능 기간(FR-2.5)을 위한 정책은 별도 배치잡으로 관리한다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | |
| 404 | `CHAT_4041` | |
| 409 | `BRANCH_4091` | 이미 삭제된 chat |

**관련 FR:** FR-2.5

---

### 2.4 분기 복구

```http
POST /api/v1/chats/{chatId}/restore
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

소프트 삭제된 분기를 복원한다. 자손 분기도 함께 복원된다.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |

**요청 본문:** 없음

**응답 (200 OK)**

`result` 필드는 §2.1의 응답 본문과 동일한 chat 정보.

**처리 규칙**

1. 대상 chat과 모든 **자손 chat**을 함께 복원한다 (cascade 복구).
2. **부모 chat이 삭제 상태이면 단독 복구를 거부한다**. `409 Conflict` + `BRANCH_4091` 반환. 사용자는 더 상위의 삭제된 조상부터 복구해야 한다.
   ```
   예시:
     A 삭제 → B 삭제 → C 삭제 인 상황에서
     C만 복구 요청 → BRANCH_4091 (부모 B가 삭제 상태)
     A 복구 요청 → 성공. A, B, C 모두 함께 복구됨.
   ```
3. 복구 가능 기간(예: 30일) 안에만 가능. 만료 시 `BRANCH_4092` 반환. 정확한 기간 정책은 별도 결정.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | |
| 404 | `CHAT_4041` | |
| 409 | `BRANCH_4091` | 부모 chat이 삭제 상태 (부모부터 복구 필요) |
| 409 | `BRANCH_4092` | 복구 기간 만료 |

**관련 FR:** FR-2.5

---

## 3. 그래프 시각화 API (FG-3)

### 3.1 그래프 조회

```http
GET /api/v1/chats/{chatId}/graph?centerTurnId={turnId}&up=30&down=30
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

지정한 root chat의 분기 구조를 반환한다. chat 골격은 전체 반환되고, turn 노드는 `centerTurnId` 기준 window만 포함된다.

> 본 명세의 핵심 엔드포인트. NFR-P-2(3초 이내)는 인덱스와 window 크기 제한으로 보장한다.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | 분기 트리에 속한 임의의 chat ID. root든 자손이든 무관하며, 서버는 해당 chat의 `root_chat_id`를 기준으로 트리 전체를 반환한다. |

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `centerTurnId` | Long? | path의 chatId에 해당하는 chat의 `lastTurnId`, 없으면 §3.1.1의 fallback 규칙 적용 | window 중심 turn. 클라이언트가 명시 전달 권장 |
| `up` | Integer | 30 | centerTurnId에서 ancestor 방향 최대 홉 수 (1~100) |
| `down` | Integer | 30 | centerTurnId에서 자손 방향 BFS 최대 홉 수 (1~100) |
| `includeDeleted` | Boolean | false | 삭제된 분기 포함 여부 |

**응답 (200 OK)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `rootChatId` | Long | |
| `center` | CenterDto? | window 중심 정보. fallback 규칙(§3.1.1)에 따라 null일 수 있음 |
| `chats` | Array<ChatNodeDto> | 트리에 속한 chat 골격 (전체) |
| `turns` | Array<TurnNodeDto> | window 안의 turn 노드 |
| `frontier` | FrontierDto | window 경계 정보 |

**CenterDto**

| 필드 | 타입 | 설명 |
|---|---|---|
| `turnId` | Long | |
| `chatId` | Long | |

**ChatNodeDto** (그래프용 chat 골격 — message 본문 없음)

| 필드 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |
| `title` | String | |
| `titleStatus` | Enum<"PENDING" \| "GENERATED" \| "USER_EDITED"> | |
| `parentChatId` | Long? | 루트면 null |
| `branchPointTurnId` | Long? | 루트면 null |
| `depth` | Integer | root로부터의 깊이. 응답 시점 계산 |
| `isDeleted` | Boolean | 소프트 삭제 여부 |
| `lastTurnId` | Long? | 가장 최근 turn |
| `updatedAt` | DateTime | |

**TurnNodeDto** (그래프용 turn 노드 — message 본문 없음)

| 필드 | 타입 | 설명 |
|---|---|---|
| `turnId` | Long | |
| `chatId` | Long | 소속 chat |
| `turnSequence` | Integer | chat 내 순서 (1부터) |
| `summary` | String? | LLM 생성 요약. 미생성 시 null |
| `summaryStatus` | Enum<"PENDING" \| "GENERATED"> | `summary IS NULL`이면 PENDING |
| `isBranchPoint` | Boolean | 다른 chat의 `branchPointTurnId`로 참조되는지 (응답 시점 계산) |
| `isCurrent` | Boolean | center인지 여부 |
| `createdAt` | DateTime | |

**FrontierDto**

| 필드 | 타입 | 설명 |
|---|---|---|
| `up` | Array<FrontierPoint> | UP 방향 경계 |
| `down` | Array<FrontierPoint> | DOWN 방향 경계 (분기마다 별도 항목 가능) |

**FrontierPoint**

| 필드 | 타입 | 설명 |
|---|---|---|
| `fromTurnId` | Long | 더 펼칠 수 있는 시작 지점 |
| `hasMore` | Boolean | 너머에 추가 노드 존재 여부 |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "rootChatId": 42,
    "center": { "turnId": 152, "chatId": 87 },
    "chats": [
      {
        "chatId": 42,
        "title": "소프트웨어 공학 SRS",
        "titleStatus": "USER_EDITED",
        "parentChatId": null,
        "branchPointTurnId": null,
        "depth": 0,
        "isDeleted": false,
        "lastTurnId": 110,
        "updatedAt": "2026-05-17T13:24:51Z"
      },
      {
        "chatId": 87,
        "title": "비기능 요구사항 심화",
        "titleStatus": "GENERATED",
        "parentChatId": 42,
        "branchPointTurnId": 105,
        "depth": 1,
        "isDeleted": false,
        "lastTurnId": 152,
        "updatedAt": "2026-05-17T13:30:11Z"
      }
    ],
    "turns": [
      {
        "turnId": 105,
        "chatId": 42,
        "turnSequence": 5,
        "summary": "DB 설계 질문",
        "summaryStatus": "GENERATED",
        "isBranchPoint": true,
        "isCurrent": false,
        "createdAt": "2026-05-17T13:05:00Z"
      },
      {
        "turnId": 152,
        "chatId": 87,
        "turnSequence": 1,
        "summary": null,
        "summaryStatus": "PENDING",
        "isBranchPoint": false,
        "isCurrent": true,
        "createdAt": "2026-05-17T13:30:00Z"
      }
    ],
    "frontier": {
      "up":   [ { "fromTurnId": 105, "hasMore": true  } ],
      "down": [ { "fromTurnId": 152, "hasMore": false } ]
    }
  }
}
```

**프론트엔드 활용 (FR-3.1, FR-3.2, FR-3.3, FR-3.4)**

- 노드 = 각 `TurnNodeDto`
- 순서 간선 = 같은 `chatId`에서 `turnSequence` 인접 항목 연결
- 분기 간선 = 각 `ChatNodeDto.branchPointTurnId` → 해당 chat의 `turnSequence=1`인 turn
- 분기 시작 지점 강조 (FR-3.3) = `isBranchPoint: true`
- 현재 위치 강조 (FR-3.4) = `isCurrent: true`
- frontier 위치에 "더 보기" 핸들 표시 (FR-3.5의 보조)
- child chat의 첫 turn이 현재 `turns` window에 포함되지 않은 경우, 클라이언트는 `branchPointTurnId` 위치에 접힌 분기(collapsed branch) 핸들을 표시한다. 해당 핸들을 클릭하면 child chat의 `lastTurnId` 또는 분기점 주변 turn을 `centerTurnId`로 하여 graph를 재조회한다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `GRAPH_4001` | up/down 범위 위반, centerTurnId가 chatId의 트리에 속하지 않음 |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | |
| 404 | `CHAT_4041` | 존재하지 않는 chatId |
| 404 | `TURN_4041` | 존재하지 않는 centerTurnId |

**관련 FR:** FR-3.1, FR-3.3, FR-3.4  
**관련 NFR:** NFR-P-2

#### 3.1.1 centerTurnId fallback 규칙

`centerTurnId`가 생략되었거나 path의 chat에 `lastTurnId`가 null인 경우(분기 생성 직후 등) 다음 규칙을 적용한다.

1. **path chatId의 lastTurnId가 존재하면** → 그것을 centerTurnId로 사용.
2. **lastTurnId가 null이고 path chat이 branch이면** → 해당 chat의 `branchPointTurnId`를 centerTurnId로 사용 (부모 chat의 turn을 기준으로 그래프 표시).
3. **lastTurnId가 null이고 path chat이 root이면** → 빈 응답 반환:
   ```json
   {
     "rootChatId": <chatId>,
     "center": null,
     "chats": [ <root chat 골격 1개> ],
     "turns": [],
     "frontier": { "up": [], "down": [] }
   }
   ```

이 fallback 덕분에 분기 생성 직후나 신규 가입 직후처럼 turn이 없는 상태에서도 그래프 API가 안정적으로 동작한다.

---

### 3.2 그래프 윈도우 확장

```http
GET /api/v1/chats/{chatId}/graph/expand?fromTurnId={id}&direction=UP&limit=30&includeDeleted=false
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

frontier 지점에서 윈도우를 추가로 펼친다. 전체 graph 응답을 다시 받지 않고 증분 로딩하기 위함이다.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | §3.1과 동일. 트리에 속한 임의의 chat ID |

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `fromTurnId` | Long | (필수) | 확장 시작 turn |
| `direction` | Enum<"UP" \| "DOWN"> | (필수) | 확장 방향 |
| `limit` | Integer | 30 | 추가 노드 최대 수 (1~100) |
| `includeDeleted` | Boolean | false | 삭제된 분기 포함 여부 (§3.1과 일관성 유지). DOWN 방향 확장 시 의미 있음 |

**응답 (200 OK)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `direction` | Enum<"UP" \| "DOWN"> | 요청 방향 그대로 |
| `turns` | Array<TurnNodeDto> | 추가된 turn 노드. `isCurrent`는 모두 false |
| `frontier` | FrontierDto | 갱신된 frontier (해당 방향만 유효) |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "direction": "UP",
    "turns": [
      {
        "turnId": 102,
        "chatId": 42,
        "turnSequence": 2,
        "summary": "프로젝트 시작",
        "summaryStatus": "GENERATED",
        "isBranchPoint": false,
        "isCurrent": false,
        "createdAt": "2026-05-17T12:55:00Z"
      },
      {
        "turnId": 101,
        "chatId": 42,
        "turnSequence": 1,
        "summary": "...",
        "summaryStatus": "GENERATED",
        "isBranchPoint": false,
        "isCurrent": false,
        "createdAt": "2026-05-17T12:50:00Z"
      }
    ],
    "frontier": {
      "up":   [ { "fromTurnId": 101, "hasMore": false } ],
      "down": []
    }
  }
}
```

**처리 규칙**

- `UP` 방향은 ancestor path를 단일 경로로 추적한다.
- `DOWN` 방향은 BFS로 펼쳐지며, 새로운 분기를 만나면 그 chat의 첫 turn까지 포함한다.
- `fromTurnId`는 path의 `chatId`가 속한 root tree 내부의 turn이어야 한다.
- `includeDeleted=false`인 경우 삭제된 chat에 속한 `fromTurnId`는 확장할 수 없다.
- `fromTurnId`는 본 응답을 반환하면 위치가 옮겨가므로, 클라이언트는 응답의 새 `frontier`를 그대로 사용한다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `GRAPH_4001` | 잘못된 direction, limit 범위 위반, fromTurnId가 chatId의 트리에 속하지 않음 |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | |
| 404 | `TURN_4041` | 존재하지 않는 fromTurnId |

**관련 FR:** FR-3.2, FR-3.5

---

## 4. 분기 메커니즘 기반 메시지 작업 (FG-1 확장)

다음 두 엔드포인트는 분기 시스템이 도입된 후에야 의미를 갖는 기능이다 (SRS: "수정/재생성 결과는 기존 대화를 바꾸지 않고 같은 지점에서 새로 갈라진 대화로 저장"). 따라서 Phase 3에 배정한다.

### 4.1 응답 재생성

```http
POST /api/v1/chats/{chatId}/messages/{messageId}/regenerate
Authorization: Bearer <accessToken>
Accept: text/event-stream
```

**인증 필요:** ✓

직전 assistant 응답을 재생성한다. 기존 응답은 보존되며, 같은 분기점에서 자동으로 새 분기가 생성되고 그 안에서 첫 메시지로 새 응답이 스트리밍된다.

> Phase 2 §5 예고와 동일한 경로. 본 명세에서 SSE 흐름을 확정한다.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | assistant 메시지가 속한 chat |
| `messageId` | Long | 재생성 대상 assistant 메시지 ID |

**요청 본문:** 없음

**응답 (200 OK, `Content-Type: text/event-stream`)**

처리 흐름:
1. `messageId`가 속한 turn을 식별
2. 그 turn의 user 메시지를 입력으로 사용
3. **분기점 결정 규칙** (§4.1.1 참조)에 따라 branchPoint turn 선택
4. 해당 branchPoint를 기준으로 새 분기 자동 생성
5. 새 분기 안에서 user 메시지 재기록 + LLM 응답 스트리밍

SSE 이벤트 시퀀스는 Phase 2 §2.5의 메시지 송신과 동일한 패턴을 따르되, 시작 이벤트에 새 분기 정보가 포함된다.

```
event: branch_created
data: {"newChatId": 88, "branchPointTurnId": 104, "title": "새 분기", "titleStatus": "PENDING"}

event: turn_started
data: {"turnId": 200, "userMessageId": 400, "aiMessageId": 401}

event: chunk
data: {"text": "..."}

... (반복) ...

event: turn_completed
data: {"turnId": 200, "aiMessageId": 401, "answerToken": 350, "summaryStatus": "PENDING"}

event: done
data: {}
```

> **Phase 2 SSE와의 차이:** Phase 2의 `turn_completed`에는 `summary`가 즉시 포함되었으나, Phase 3에서는 turn summary 생성이 비동기로 분리된다(§0.5 원칙 3 및 §5 처리 시점 표). 따라서 `turn_completed`에는 `summaryStatus: "PENDING"`만 포함되며, summary 본문은 비동기 워커가 채운 후 다음 graph/turns 조회 시 반영된다. **이 변경은 Phase 2의 일반 메시지 송신 API에도 후속 마이그레이션으로 적용 권장.**

**SSE 이벤트 타입 (Phase 2 §2.5 대비 변경/추가)**

| event | data 스키마 | 의미 |
|---|---|---|
| `branch_created` | `{newChatId: Long, branchPointTurnId: Long, title: String, titleStatus: Enum}` | **신규.** 분기 자동 생성 알림. 클라이언트는 그래프 갱신 |
| `turn_completed` | `{turnId: Long, aiMessageId: Long, answerToken: Integer, summaryStatus: Enum<"PENDING">}` | **변경.** summary 필드 제거, summaryStatus 추가 |
| 이외 | (Phase 2와 동일) | |

#### 4.1.1 분기점(branchPoint) 결정 규칙

재생성/수정 대상 turn의 **논리적 직전 turn**을 branchPoint로 사용한다.

1. **대상 turn의 turnSequence > 1** → 같은 chat의 `turnSequence - 1` turn을 branchPoint로 사용.
2. **대상 turn의 turnSequence = 1이고 현재 chat이 branch** → 현재 chat의 `branchPointTurnId`를 branchPoint로 사용 (부모 chat의 turn으로 거슬러 올라감).
3. **대상 turn의 turnSequence = 1이고 현재 chat이 root** → 더 이전 맥락이 없으므로 재생성/수정을 거부한다. `409 Conflict` + `MESSAGE_4092` 반환.

> 규칙 3은 단순성을 위한 정책 선택이다. 향후 "root의 첫 turn 수정 시 sibling root 생성" 같은 확장은 별도 논의가 필요하다.

**프론트엔드 권장 흐름**

1. `branch_created` 수신 → 그래프에 새 노드 즉시 추가, 현재 분기를 새 분기로 전환
2. `turn_started`부터는 Phase 2와 동일하게 처리
3. `done` 수신 후 그래프 재조회는 선택사항 (이미 새 노드는 반영됨)
4. summary는 별도 polling 또는 다음 graph 조회 시 채워짐

**에러 응답 규칙**

Phase 2 §2.5와 동일. 스트리밍 시작 전 에러는 `ApiResponse<T>` JSON, 시작 후 에러는 SSE `event: error`.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | |
| 404 | `CHAT_4041` | |
| 404 | `MESSAGE_4041` | 존재하지 않는 messageId |
| 409 | `MESSAGE_4092` | senderType이 ASSISTANT가 아니거나 STREAMING 상태, 또는 root chat의 첫 turn(§4.1.1 규칙 3) |
| stream | `LLM_5001` | LLM API 호출 실패 |

**관련 FR:** FR-1.4, FR-2.1, FR-2.2

---

### 4.2 사용자 메시지 수정

```http
PATCH /api/v1/chats/{chatId}/messages/{messageId}
Authorization: Bearer <accessToken>
Accept: text/event-stream
Content-Type: application/json
```

**인증 필요:** ✓

과거 user 메시지를 수정한다. 기존 메시지는 보존되며, **§4.1.1 분기점 결정 규칙**에 따라 선정된 branchPoint를 기준으로 새 분기가 자동 생성되고 그 안에서 수정된 메시지로 첫 응답이 스트리밍된다.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |
| `messageId` | Long | 수정 대상 user 메시지 ID |

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `content` | String | ✓ | 수정된 메시지 내용 |

```json
{
  "content": "다시 묻겠습니다. Spring Security 필터 체인의 순서는 어떻게 되나요?"
}
```

**응답 (200 OK, `Content-Type: text/event-stream`)**

SSE 흐름은 §4.1 응답 재생성과 동일. `branch_created` → `turn_started` → `chunk*` → `turn_completed` → `done`.

차이점은 `turn_started`의 user 메시지가 수정된 내용으로 기록된다는 점이다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `COMMON_4001` | content 빈 문자열 |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | |
| 404 | `CHAT_4041` | |
| 404 | `MESSAGE_4041` | |
| 409 | `MESSAGE_4092` | senderType이 USER가 아님, 또는 root chat의 첫 turn(§4.1.1 규칙 3) |
| stream | `LLM_5001` | |

**관련 FR:** FR-1.5, FR-2.1

---

## 5. 처리 시점 정리

다음 표는 Phase 3의 각 작업이 어느 시점에 수행되는지를 명확히 한다. NFR-P-2(그래프 3초)와 NFR-P-1(응답 시작 3초)을 동시에 만족시키기 위한 핵심 분리다.

| 작업 | 시점 | 비고 |
|---|---|---|
| chat/turn → DTO 변환 | 조회 시점 | 단순 매핑 |
| window 계산 (UP ancestor path / DOWN BFS) | 조회 시점 | 인메모리 탐색 |
| frontier 계산 | 조회 시점 | window 경계 확인 |
| `isBranchPoint` 마킹 | 조회 시점 | chats 결과로 boolean 계산 |
| `depth` 계산 | 조회 시점 | 부모 체인 따라 계산 |
| edge 구성 | 프론트 | nodes만 받아 프론트가 조립 |
| **turn summary 생성** | **저장 시점 비동기** | LLM 호출. assistant 메시지 저장 직후 워커 트리거 |
| **분기 제목 자동 명명** | **저장 시점 비동기** | fork 직후 워커 트리거. 임시명으로 즉시 응답 |
| **context summary (FR-5.2)** | **비동기 배치** | 토큰 임계 초과 시. Phase 4로 이연 가능 |

---

## 6. 시나리오 예시

### 6.1 분기 생성 → 첫 메시지 → 그래프 갱신

> SRS Sequence Diagram ②와 대응. 분기 생성과 첫 메시지 송신은 별도 API 두 단계.

```
1. 사용자가 turn 105 위에서 "여기서 분기" 클릭
   → POST /api/v1/chats/42/branches { branchPointTurnId: 105 }
   ← 201 { chatId: 87, title: "새 분기", titleStatus: "PENDING", ... }

2. 프론트가 그래프 갱신을 위해 재조회
   → GET /api/v1/chats/42/graph?centerTurnId=105
   ← chats[42, 87] + turns[...] + frontier

3. 사용자가 새 분기에서 첫 메시지 입력
   → POST /api/v1/chats/87/messages { content: "..." }
   ← SSE 스트림

4. 백그라운드 워커가 분기 제목 LLM 생성 완료
   → DB: chat.title = "분기 그래프 깊이 토론", title_status = GENERATED

5. 다음 graph 조회 시 갱신된 제목 반영
   → GET /api/v1/chats/42/graph?centerTurnId=200
```

### 6.2 분기 간 이동

```
1. 사용자가 그래프에서 다른 분기의 노드 turn 300 클릭
   → GET /api/v1/chats/42/graph?centerTurnId=300
   ← chats 골격 + turn 300 주변 window 반환

2. 대화 영역 채우기 (Phase 2 API 재사용)
   → GET /api/v1/chats/{turn 300이 속한 chatId}/turns?lastTurnSequence=...
   ← message 본문 포함된 turn 목록
```

### 6.3 윈도우 확장

```
1. 사용자가 그래프 위쪽 frontier 핸들 클릭
   → GET /api/v1/chats/42/graph/expand?fromTurnId=103&direction=UP&limit=30
   ← turns 추가 + 새 frontier
```

### 6.4 응답 재생성 (자동 분기)

```
1. 사용자가 assistant 메시지 401에서 "재생성" 클릭
   → POST /api/v1/chats/42/messages/401/regenerate
   ← SSE 스트림 시작

2. SSE 이벤트 시퀀스:
   event: branch_created  → 그래프에 새 노드 즉시 추가
   event: turn_started    → 새 분기에서 첫 응답 시작
   event: chunk           → 점진적 표시
   ...
   event: turn_completed
   event: done

3. 백그라운드 워커가 분기 제목 비동기 생성 (§6.1과 동일)
```

---

## 7. ERD ↔ API 필드 매핑 (Phase 2와의 연속)

Phase 2 §4 ERD ↔ API 매핑에 다음을 추가한다.

### chat 테이블 → 응답 DTO (Phase 3 추가분)

| ERD 컬럼 | API 필드 | 비고 |
|---|---|---|
| `branch_point_turn_id` | `branchPointTurnId` | Phase 3에서 활성화 |
| `parent_id` | `parentChatId` | 그래프 응답에서 사용 |
| `last_turn_id` | `lastTurnId` | 신규 컬럼. 사이드바 미리보기·정렬에 활용 |
| `title_status` | `titleStatus` | 신규 컬럼. PENDING/GENERATED/USER_EDITED |

### turn 테이블 → 응답 DTO (Phase 3 추가분)

| ERD 컬럼 | API 필드 | 비고 |
|---|---|---|
| `Summary` | `summary` | Phase 2와 동일. 그래프 노드 라벨로도 사용 |
| (응답 시점 계산) | `isBranchPoint` | DB 컬럼 없음. chats 결과로 계산 |
| (응답 시점 계산) | `isCurrent` | DB 컬럼 없음. center 비교로 계산 |
| (응답 시점 계산) | `summaryStatus` | `summary IS NULL`이면 PENDING. DB 기본 문구 저장 금지 |

---

## 8. Phase 3 엔드포인트 목록

| Method | Path | 설명 | 인증 | FR | 우선순위 |
|---|---|---|---|---|---|
| POST | `/api/v1/chats/{chatId}/branches` | 분기 생성 | ✓ | FR-2.1, FR-2.2 | 필수 |
| PATCH | `/api/v1/chats/{chatId}` | 분기 제목 수정 | ✓ | FR-2.2 | 필수 |
| DELETE | `/api/v1/chats/{chatId}` | 분기 삭제 (cascade) | ✓ | FR-2.5 | 필수 |
| GET | `/api/v1/chats/{chatId}/graph` | 그래프 조회 (window) | ✓ | FR-3.1, FR-3.3, FR-3.4 | 필수 |
| GET | `/api/v1/chats/{chatId}/graph/expand` | 윈도우 확장 | ✓ | FR-3.2, FR-3.5 | 필수 |
| POST | `/api/v1/chats/{chatId}/messages/{messageId}/regenerate` | 응답 재생성 (자동 분기) | ✓ | FR-1.4 | 필수 |
| PATCH | `/api/v1/chats/{chatId}/messages/{messageId}` | 메시지 수정 (자동 분기) | ✓ | FR-1.5 | 필수 |
| POST | `/api/v1/chats/{chatId}/restore` | 분기 복구 | ✓ | FR-2.5 | 보류 가능 |

총 8개 엔드포인트. Phase 2의 11개와 합쳐 누적 19개.

**우선순위 정책:**
- **필수**: Phase 3 발표·통합 테스트 통과를 위해 반드시 구현되어야 하는 엔드포인트.
- **보류 가능**: 시간이 부족하면 Phase 4로 이연 가능. 단 `DELETE`는 soft delete로 구현되므로 데이터 손실은 없음.

**Phase 2와의 변경:**
- `DELETE /chats/{chatId}` — Phase 2의 보조 기능이 Phase 3에서 cascade 처리 추가로 정식 승격. 경로는 동일.
- **`turn_completed` SSE 이벤트 스키마 변경** — Phase 2의 `summary` 필드 제거, `summaryStatus` 추가. Phase 2의 일반 메시지 송신 API에도 후속 마이그레이션 권장 (§4.1 참조).

## 9. 성능 고려 (NFR-P-2)

### 9.1 응답 페이로드 예상 크기

- `ChatNodeDto`: 약 250 B/개. 20개 = 5 KB
- `TurnNodeDto`: 약 280 B/개 (summary 60자 컷 가정). 60개 = 17 KB
- 전체: 25 KB 이하 (gzip 시 6~8 KB)

### 9.2 쿼리 패턴

- chat 골격: `WHERE root_chat_id = ? AND deleted_at IS NULL` — `idx_chat_root_deleted` 활용
- window 탐색: `WHERE aichat_session_id = ? AND turn_sequence BETWEEN ? AND ?` — `idx_turn_chat_sequence` 활용
- 분기점 마킹: chats 결과를 `Map<turnId, List<childChatId>>`로 변환하여 O(1) 조회

### 9.3 캐시 정책

본 명세에서는 별도 캐시 계층을 도입하지 않는다. NFR-P-2(3초)는 인덱스만으로 충분히 만족 가능하다. 사용자 수 증가 시 chat 골격에 대한 read-through 캐시(Redis 등)를 검토할 수 있다.

---

## 10. 미해결 / 향후 결정 사항

1. **비동기 작업 완료 통보 방식** — 현재는 polling 기반(다음 graph 조회 시 반영). WebSocket/SSE push는 Phase 4에서 결정.
2. **summary 길이 컷** — DB에는 전체 summary를 저장하고 API 응답 시 60자 컷을 적용. 정확한 임계값은 통합 테스트로 조정.
3. **window 기본 크기** — `up=30, down=30`은 초기값. NFR-P-2 측정 결과로 조정.
4. **분기 복구 기간** — 정확한 일수는 Phase 4 정책 수립 시 결정.
5. **`POST /chats` 응답 필드 확장** — Phase 2 응답에 `titleStatus`, `lastTurnId`를 추가할지 여부. 본 명세는 Phase 3에서 신설되는 응답에서만 신규 컬럼을 노출한다. Phase 2의 기존 응답 확장은 별도 결정.
6. **FR-2.4 (부모 분기로 돌아가기) 구현 방식** — 별도 엔드포인트 없이 graph 응답의 `chats[].parentChatId` / `branchPointTurnId`를 활용해 클라이언트에서 처리. 빈번한 사용 패턴이 확인되면 후속 Phase에서 편의 API 추가 검토.
7. **Phase 2 `turn_completed` SSE 마이그레이션** — Phase 3에서 비동기 summary 원칙을 도입하면서 SSE 스키마가 변경됨. Phase 2의 일반 메시지 송신 API에도 동일 스키마 적용이 필요하며, Phase 2 코드 수정 일정은 별도 합의.

---

## 11. 변경 이력

| 버전 | 날짜 | 변경 사유 |
|---|---|---|
| 0.1 | 2026-05-17 | Phase 3 초안. graph window 방식, 비동기 LLM 가공 분리, ERD v2.1 반영. Phase 2 명세서 컨벤션(`ApiResponse<T>`, 에러 코드 체계, 경로 패턴) 준수. |
| 0.2 | 2026-05-18 | 리뷰 반영. centerTurnId 명칭 통일, 재생성/수정 첫 turn edge case 정의, SSE summary 비동기화, expand includeDeleted 추가, cascade 복구 정책 보강, parent API 제거, endpoint 우선순위 추가. |
| 0.3 | 2026-05-18 | 추가 리뷰 반영. Summary NULL/PENDING 정책 명시, window 밖 branch의 collapsed handle 렌더링 규칙 추가, 비동기 title worker의 USER_EDITED 덮어쓰기 방지 조건 추가, expand fromTurnId 검증 조건 보강. |
