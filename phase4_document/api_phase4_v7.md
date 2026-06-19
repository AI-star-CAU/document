# Phase 4 API 명세서

**문서 식별자:** API-AIT-P4  
**버전:** 0.8  
**작성일:** 2026-05-26  
**대상 Phase:** Phase 4 (W14) — 대화 기록 파일 시스템 + 맥락 관리  
**범위:** FG-4 대화 기록 파일 시스템 (FR-4.2, FR-4.3), FG-5 맥락 관리 (FR-5.1, FR-5.2, FR-5.3)  
**기준 ERD:** `ait_erd_phase4` (Phase 3 ERD에 Phase 4 변경분 적용)  
**선행 Phase:** Phase 2 (Walking Skeleton — FG-1, FG-7), Phase 3 (FG-2, FG-3)

---

## 0. 공통 컨벤션

Phase 2 명세서(`API-AIT-P2`) §0 및 Phase 3 명세서(`API-AIT-P3`) §0 공통 컨벤션을 그대로 따른다. 본 절은 Phase 4에서 추가되거나 변경되는 항목만 명시한다.

### 0.1 변경 없음

다음 사항은 Phase 2·3과 동일하다.

- Base URL: `/api/v1`
- 인증: JWT Bearer Token
- 응답 래퍼: `ApiResponse<T>` (`isSuccess`, `code`, `message`, `result`)
- HTTP 상태 코드 정책: 204 미사용, 삭제도 200 + `result: null`
- 데이터 타입 표기 (`Long`, `String`, `DateTime`, `Enum`)
- Offset / Cursor 페이지네이션 사용 규칙

### 0.2 Phase 4에서 추가되는 에러 코드

| HTTP | code | message |
|---|---|---|
| 400 | `EXPLORER_4001` | 유효하지 않은 탐색기 조회 파라미터입니다. |
| 404 | `USAGE_4041` | 사용량 기록을 찾을 수 없습니다. |

기존 `CHAT_4041`, `TURN_4041`, `AUTH_4011`, `COMMON_403` 등은 Phase 2·3과 동일하게 사용한다. 맥락 압축 중 LLM 호출이 실패하면 Phase 2·3의 `LLM_5001` 또는 `COMMON_500`으로 처리한다.

### 0.3 본 명세에서 사용하는 핵심 용어

| 용어 | 정의 |
|---|---|
| 탐색기 (explorer) | 사용자가 소유한 root chat과 그 하위 분기를 파일 탐색기와 유사한 트리 구조로 보여주는 뷰. SRS §2.1.2 "대화 탐색 영역"에 대응. |
| 트리 노드 | 탐색기에서 한 줄로 표시되는 chat. root chat 또는 branch chat. |
| 깊이 (depth) | root chat을 0으로 두고, 자식 분기로 한 단계 내려갈 때마다 1씩 증가하는 값. |
| context | LLM에 전달되는 입력 메시지의 집합. (FR-5.1, FR-5.2) |
| ancestor path | branch chat에서 root chat까지의 turn 시퀀스를 시간순으로 이은 경로. 맥락 조립의 기본 단위. |
| context window | LLM 모델이 한 번에 처리할 수 있는 최대 토큰 수 (예: 128K). |
| 압축 (compression) | 맥락 길이가 한도에 근접할 때 오래된 turn을 해당 turn의 summary로 치환하는 동작. (FR-5.2) |
| usage period | 토큰 사용량을 집계하는 기간 단위. plan에 따라 월 단위가 기본. |

---

## 0.5 설계 원칙

Phase 4는 다음 다섯 가지 원칙 위에서 설계된다.

1. **탐색기와 그래프는 책임이 다르다** — 탐색기(FG-4)는 chat 단위 메타데이터의 트리 표현이며, 그래프(FG-3)는 turn 단위 노드의 윈도우 표현이다. 동일한 분기 구조를 다른 추상화 수준으로 보여준다. 탐색기는 chat이 적게는 수십 개, 많아도 수백 개 단위이므로 전체 트리를 한 번에 반환할 수 있다.

2. **탐색을 통한 이동은 기존 API의 재사용** — FR-4.3 "탐색기를 통한 이동"은 별도 엔드포인트를 만들지 않는다. 탐색기 응답에 포함된 `chatId`를 사용해 클라이언트가 Phase 2의 `GET /chats/{id}/turns` 또는 Phase 3의 `GET /chats/{id}/graph`를 호출하면 된다.

3. **맥락 조립은 서버 내부 동작, 외부에서는 결과만 관찰** — FR-5.1·FR-5.2는 LLM 요청 직전에 서버 내부에서 자동으로 적용된다. 별도의 "맥락 생성" 엔드포인트는 두지 않으며, 메시지 송신(`POST /messages`)·재생성·수정 API 안에 통합된다. 단, 투명성과 디버깅 편의를 위해 사용된 맥락의 요약 통계는 SSE `turn_completed` 이벤트로 노출한다.

4. **압축은 비동기 의존이 아닌 동기 fallback 보유** — FR-5.2 압축은 turn의 `summary`를 활용하는데, summary는 Phase 3에서 비동기 생성된다. 압축 시점에 summary가 아직 PENDING이면 fallback으로 turn content를 토큰 단위로 절단(truncation)한다. 사용자는 응답을 받기 위해 summary 생성을 기다리지 않는다.

5. **토큰 사용량은 누적, 노출은 집계** — message 단위의 `prompt_token`·`answer_token`은 이미 ERD에 존재한다. Phase 4에서는 `usage_record` 테이블을 활성화하여 회원·기간별 누적치를 집계하고, FR-5.3의 사용자 표시는 이 집계값을 사용한다. 실시간 정확성보다 일관성·캐시 친화성을 우선한다.

---

## 1. ERD 변경분 (Phase 3 → Phase 4)

Phase 4에서 Phase 3 ERD(`ait_erd_phase3`) 대비 다음 변경분이 적용된다. 엔티티 관계는 그대로이며 컬럼·인덱스만 추가된다. 통합된 전체 스키마는 `AIT_ERD_phase4.txt` 파일을 참조한다.

### 1.1 컬럼·인덱스 변경

```sql
-- chat에 마지막 활동 시각 캐시 추가 (탐색기 정렬 성능용)
-- 2단계 마이그레이션: 일단 NULL 허용으로 추가 → 백필 → NOT NULL로 변경
ALTER TABLE chat
    ADD COLUMN last_activity_at timestamp NULL
        COMMENT '최신 turn 또는 분기 활동 시각. 탐색기 정렬·메타 표시용 캐시',
    ADD INDEX idx_chat_member_last_activity (member_id, last_activity_at DESC);

-- 기존 row 백필
UPDATE chat
SET last_activity_at = COALESCE(
    (SELECT MAX(t.created_at) FROM turn t WHERE t.chat_id = chat.chat_id),
    chat.updated_at,
    chat.created_at
);

-- 최종적으로 NOT NULL로 강제
ALTER TABLE chat
    MODIFY COLUMN last_activity_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP;

-- usage_record에 압축 누적 카운터 추가
ALTER TABLE usage_record
    ADD COLUMN compressed_turn_count INT NOT NULL DEFAULT 0
        COMMENT 'FR-5.2 압축이 적용된 누적 turn 수 (관측용)';

-- usage_record 조회 인덱스
CREATE INDEX idx_usage_member_period
    ON usage_record(member_id, period_end DESC);
```

### 1.2 Phase 3 ERD 표기 정리 (스키마 정의는 동일)

Phase 4 ERD 파일을 만들면서 Phase 3 ERD에 남아 있던 표기·정책 모호성 두 건을 함께 정리한다. **데이터 의미 변경은 없으며, 정책을 명문화하는 변경이다.**

- **`chat.last_turn_id`** — `DEFAULT 0` 제거. `turn_id = 0`인 turn은 존재하지 않으므로 의미 없는 기본값이었다. `last_turn_id bigint NULL`로 변경. 신규 생성 chat은 NULL로 시작하고 첫 turn 저장 시 갱신한다.
- **`chat.root_chat_id`** — `NULL` 유지. root chat은 `initRootChatId()`로 save 직후 자기 `chat_id`를 저장하고, branch는 부모의 `root_chat_id`를 복사한다. `@GeneratedValue(IDENTITY)` 특성상 INSERT 시점에 ID를 알 수 없어 NOT NULL 제약을 걸면 임시값(0L) 우회가 필요하므로, JPA 컬럼은 nullable로 두고 앱 레벨에서 보장한다. 실질적으로 null인 row는 존재하지 않는다.

### 1.3 FK 제약 관리 방식

본 ERD에는 PK·인덱스만 명시되며 FK 제약은 명시하지 않는다. FK 관계(`turn.chat_id → chat`, `message.turn_id → turn`, `chat.parent_id → chat`, `chat.branch_point_turn_id → turn`, `usage_record.plan_id → plan` 등)는 **JPA 연관관계(`@ManyToOne`, `@JoinColumn`)로 관리**한다. 실제 DDL은 JPA가 생성하거나 별도의 마이그레이션 도구가 처리한다.

### 1.4 `last_activity_at` 초기값 및 갱신 정책

- chat 생성 시 `created_at`과 동일한 값으로 초기화한다. **null 허용하지 않음.** 빈 chat도 `recent` 정렬에서 자연스러운 위치(생성 시각 기준)에 노출되어야 한다. 본 컬럼이 null이면 Phase 2에서 발생할 수 있는 빈 채팅(`POST /chats` 성공 후 `POST /messages` 실패)이 정렬 맨 뒤로 영구히 밀린다.
- 다음 이벤트가 발생할 때마다 해당 chat과 그 **ancestor chain 전체**의 `last_activity_at`을 `now()`로 갱신한다 (`max(기존, now())` 가드, 동일 트랜잭션 내).
  - 메시지 송신 (Phase 2 `POST /messages`)
  - 응답 재생성 (Phase 3 `POST /messages/{id}/regenerate`)
  - 사용자 메시지 수정 (Phase 3 `PATCH /messages/{id}`)
  - **branch 생성** (Phase 3 `POST /chats/{id}/branches`) — 새 branch 자신은 `created_at`으로 초기화되고, 부모부터 root까지의 ancestor chain은 `now()`로 갱신된다. 이 갱신이 없으면 사용자가 방금 만든 분기가 사이드바 정렬에서 부모 트리째 아래로 묻히는 문제가 발생한다.
  - **branch 복구** (Phase 3 §2.3 cascade soft delete 반대 방향, 본 명세 범위 외이지만 구현 시 같은 정책)
  - **chat 제목 수정** (Phase 3 `PATCH /chats/{id}`) — 사용자가 의도적으로 정리 작업을 한 것이므로 정렬 위로 올린다.
- **branch 삭제는 갱신 대상이 아니다.** "더 이상 보지 않겠다"는 의도에 가깝고, 삭제 직후 부모가 정렬 위로 올라오는 동작이 사용자 모델과 어긋난다.
- 기존 row 백필은 위 SQL의 `UPDATE chat SET last_activity_at = COALESCE(...)`로 처리한다.

> 본 컬럼이 없으면 탐색기 응답이 chat 트리 전체에 대해 N+1 형태로 마지막 turn 시각을 조회해야 하므로 NFR-P-2 충족이 어렵다.

### 1.5 ERD ↔ API 매핑 (Phase 4 추가분)

| ERD 컬럼 | API 필드 | 비고 |
|---|---|---|
| `chat.last_activity_at` | `lastActivityAt` | 탐색기·정렬용. 사이드바 미리보기에서 사용 |
| `usage_record.tokens_used` | `tokensUsed` | FR-5.3 표시 |
| `usage_record.token_limit` | `tokenLimit` | FR-5.3 표시 |
| `usage_record.request_count` | `requestCount` | 운영 통계 |
| `usage_record.compressed_turn_count` | (응답 미포함) | 서버 관측용. 외부 노출 안 함 |

---

## 2. 대화 탐색기 (FG-4)

### 2.1 탐색기 트리 조회

```http
GET /api/v1/chats/explorer?sort=recent&page=0&size=20&includeDeleted=false
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

사용자가 소유한 root chat 목록을 페이지 단위로 반환하며, 각 root chat에 대해 **하위 분기 트리를 평탄화된 배열로 함께 포함한다**. 프론트엔드는 `depth`와 `parentChatId`로 트리를 조립한다.

> **설계 결정:** 한 root chat의 분기 트리는 사용자당 평균 수십 개 이하로 작고, 사이드바에서는 root 단위로 펼침/접힘이 일어난다. 따라서 root chat은 페이징하고, 트리는 한 root 단위로 완전한 형태로 반환한다. lazy-loading이 필요할 만큼 큰 트리(분기 100+ 개)는 NFR-P-2 측정 결과를 보고 별도 엔드포인트(`§2.2`)로 분리 검토.

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `sort` | Enum<"recent" \| "created" \| "name"> | `recent` | 정렬 기준. FR-6.2와 정합. `recent`는 `lastActivityAt DESC` |
| `page` | Integer | 0 | 0-based root chat 페이지 번호 |
| `size` | Integer | 20 | 페이지당 root chat 수 (1~50) |
| `includeDeleted` | Boolean | false | 삭제된 chat 포함 여부. 복구 UI에서만 true |

**응답 (200 OK)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `roots` | Array&lt;ExplorerTreeDto&gt; | root chat과 그 하위 분기 트리 |
| `page` | Integer | 현재 페이지 |
| `size` | Integer | 페이지 크기 |
| `totalRootCount` | Integer | 전체 root chat 수 (삭제 제외, 단 `includeDeleted=true`이면 포함) |
| `hasNext` | Boolean | 다음 페이지 존재 여부 |

**ExplorerTreeDto**

| 필드 | 타입 | 설명 |
|---|---|---|
| `rootChatId` | Long | 트리의 루트 chat ID |
| `nodes` | Array&lt;ExplorerNodeDto&gt; | 루트 포함 모든 chat의 평탄화 배열. 루트가 0번 인덱스에 위치하며, 이후는 부모-자식 관계를 따라 DFS 순서로 정렬 |

**ExplorerNodeDto**

| 필드 | 타입 | 설명 |
|---|---|---|
| `chatId` | Long | |
| `parentChatId` | Long \| null | 부모 chat ID. root이면 null |
| `branchPointTurnId` | Long \| null | 부모 chat에서 분기된 turn ID. root이면 null |
| `title` | String \| null | 분기 제목. null이면 클라이언트가 "제목없음"·"새 분기" 등 기본 문구로 표시 |
| `titleStatus` | Enum&lt;"PENDING" \| "GENERATED" \| "USER_EDITED"&gt; | Phase 3 정의와 동일 |
| `depth` | Integer | 0=root, 1=직접 자식, ... |
| `turnCount` | Integer | 해당 chat에 속한 turn 수 (자식 분기 미포함) |
| `lastActivityAt` | DateTime | 마지막 활동 시각. turn이 없는 chat은 `createdAt`과 동일한 값이 들어 있다 (§1 초기값 정책) |
| `createdAt` | DateTime | |
| `deletedAt` | DateTime \| null | soft delete 상태. `includeDeleted=true`일 때만 null이 아닐 수 있음 |
| `llmProvider` | Enum | Phase 2와 동일 |
| `llmModel` | Enum | Phase 2와 동일 |

**응답 예시**

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "roots": [
      {
        "rootChatId": 42,
        "nodes": [
          {
            "chatId": 42,
            "parentChatId": null,
            "branchPointTurnId": null,
            "title": "비기능 요구사항 정리",
            "titleStatus": "USER_EDITED",
            "depth": 0,
            "turnCount": 12,
            "lastActivityAt": "2026-05-22T14:30:00Z",
            "createdAt": "2026-05-17T12:50:00Z",
            "deletedAt": null,
            "llmProvider": "OPENAI",
            "llmModel": "gpt-4o-mini"
          },
          {
            "chatId": 87,
            "parentChatId": 42,
            "branchPointTurnId": 105,
            "title": "분기 그래프 깊이 토론",
            "titleStatus": "GENERATED",
            "depth": 1,
            "turnCount": 4,
            "lastActivityAt": "2026-05-22T10:15:00Z",
            "createdAt": "2026-05-17T13:35:00Z",
            "deletedAt": null,
            "llmProvider": "OPENAI",
            "llmModel": "gpt-4o-mini"
          },
          {
            "chatId": 91,
            "parentChatId": 87,
            "branchPointTurnId": 207,
            "title": null,
            "titleStatus": "PENDING",
            "depth": 2,
            "turnCount": 1,
            "lastActivityAt": "2026-05-22T10:20:00Z",
            "createdAt": "2026-05-22T10:18:00Z",
            "deletedAt": null,
            "llmProvider": "OPENAI",
            "llmModel": "gpt-4o-mini"
          }
        ]
      },
      {
        "rootChatId": 50,
        "nodes": [ /* ... */ ]
      }
    ],
    "page": 0,
    "size": 20,
    "totalRootCount": 7,
    "hasNext": false
  }
}
```

**처리 규칙**

1. `member_id = 인증사용자`이고 `parent_id IS NULL`인 chat을 root로 식별
2. `includeDeleted=false`인 경우 `deleted_at IS NULL` 조건 추가
3. 정렬:
   - `recent`: `ORDER BY last_activity_at DESC, chat_id DESC` (`last_activity_at`은 NOT NULL이므로 NULLS 처리 불필요)
   - `created`: `ORDER BY created_at DESC, chat_id DESC`
   - `name`: 제목 오름차순(대소문자 무시). `title`이 null인 항목은 마지막에 배치한다. MySQL/JPA 구현 예: `ORDER BY (title IS NULL) ASC, LOWER(title) ASC, chat_id DESC` (MySQL은 표준 SQL의 `NULLS LAST` 키워드를 지원하지 않으므로 `IS NULL` 표현식으로 대체).
4. 페이징된 root들의 `rootChatId` 목록을 먼저 구한 뒤, **`root_chat_id IN (:pagedRootIds)` 조건으로 해당 root들에 속한 모든 chat을 한 번의 쿼리로 일괄 조회**한다 (`idx_chat_root_deleted` 활용). root별 개별 쿼리를 반복하지 않는다.
5. 메모리에서 DFS로 정렬: 같은 부모를 가진 형제는 `created_at ASC`
6. `turnCount` 조회 방식 — **root별로 개별 count 쿼리를 날리지 않는다.** 처리 규칙 4에서 일괄 조회한 chat의 ID 목록을 사용하여 다음 한 번의 쿼리로 일괄 계산한다:
   ```sql
   SELECT chat_id, COUNT(*) AS turn_count
   FROM turn
   WHERE chat_id IN (:allChatIds)
   GROUP BY chat_id;
   ```
   `idx_turn_chat_sequence` 활용. 결과를 메모리 Map으로 변환 후 노드에 매핑. Phase 3에서 구현한 `TurnRepository.countByChatIds(chatIds)`를 재사용한다.
7. `lastActivityAt`은 ERD §1의 `chat.last_activity_at` 캐시 값을 그대로 사용
8. 소유 chat이 0개인 사용자: `roots: []`, `totalRootCount: 0`, `hasNext: false`를 반환한다. 에러가 아닌 정상 200 응답이다.

**FR-4.3 (탐색기를 통한 이동)**

본 응답의 `chatId`를 사용해 클라이언트가 직접 다음 API를 호출한다:
- 대화 내용 보기: `GET /api/v1/chats/{chatId}/turns` (Phase 2)
- 그래프 보기: `GET /api/v1/chats/{chatId}/graph` (Phase 3)

별도의 "이동" 엔드포인트는 두지 않는다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `EXPLORER_4001` | 잘못된 sort 값, size 범위 위반 |
| 401 | `AUTH_4011` | |

**관련 FR:** FR-4.2, FR-4.3

---

### 2.2 단일 트리 새로고침 (선택 구현)

```http
GET /api/v1/chats/explorer/{rootChatId}?includeDeleted=false
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

분기 생성·삭제 직후 해당 root의 트리만 다시 가져오기 위한 보조 엔드포인트. §2.1 응답의 `ExplorerTreeDto` 한 개를 반환한다.

> **선택 구현 사유:** Phase 4 발표 기준에서는 §2.1 전체 재조회로 충분하다. 트리가 큰 사용자를 대상으로 한 점진적 갱신이 필요할 때 도입.

**Path Parameters**

| 파라미터 | 타입 | 설명 |
|---|---|---|
| `rootChatId` | Long | root chat의 ID. `parent_id IS NULL`이고 본인 소유인 chat만 허용 |

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `includeDeleted` | Boolean | false | §2.1과 동일 |

**응답 (200 OK)**

`result` 필드: `ExplorerTreeDto` 한 개.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 400 | `EXPLORER_4001` | rootChatId가 root chat이 아님 (`parent_id IS NOT NULL`) |
| 401 | `AUTH_4011` | |
| 403 | `COMMON_403` | 다른 사용자의 chat |
| 404 | `CHAT_4041` | 존재하지 않는 chatId |

**관련 FR:** FR-4.2

---

## 3. 맥락 관리 (FG-5)

### 3.1 맥락 조립 규칙 (FR-5.1 분기별 격리)

본 절은 LLM 요청 시 서버가 적용하는 맥락 조립 규칙을 명세한다. **별도의 외부 API는 없으며**, `POST /messages`·재생성·수정 등 LLM을 호출하는 모든 엔드포인트에 공통으로 적용된다.

**조립 입력:** 현재 LLM 호출의 대상이 되는 chat ID(이하 `targetChatId`) 및 새 user 메시지 내용.

**조립 절차:**

1. `targetChatId`의 chat을 조회하고, `parent_id`를 따라 root chat까지 거슬러 올라가며 chain을 만든다. chain은 root부터 target 순으로 정렬한다: `[c₀=root, c₁, c₂, …, c_target]`.

2. chain의 각 chat에 대해 포함할 turn 범위를 다음 규칙으로 결정한다.

   - **target chat (`c_target`)**: 해당 chat의 모든 turn을 시간순(`turn_sequence ASC`)으로 포함한다.

   - **ancestor chat (`c_i`, `i < target`)**: chain의 바로 다음 chat `c_{i+1}`가 갖는 `branchPointTurnId`를 조회한다. 해당 turn이 `c_i`에 속해 있어야 하며 (불일치 시 데이터 무결성 오류), 그 turn의 `turn_sequence`를 `bp_seq`라 한다. `c_i`에서는 `turn_sequence ≤ bp_seq`인 turn만 시간순으로 포함한다.

   > 즉 ancestor chat은 "다음 자식이 분기되어 나간 지점까지의 흐름"만 맥락에 포함된다. 분기점 이후에 ancestor가 독립적으로 이어간 turn(자매 흐름)은 포함하지 않는다. 이것이 FR-5.1 분기별 맥락 격리의 핵심이다.

3. 위 절차로 선별된 모든 turn의 user/assistant 메시지를 chain 순서 + turn 순서대로 시간순으로 펼친다 (이를 `flat_messages`라 한다).

4. 새 user 메시지를 `flat_messages` 끝에 추가한다.

5. 토큰 한도 검사 후 필요 시 §3.2의 압축을 적용한다.

6. 최종 messages 배열을 LLM에 전달한다.

**FR-5.1 보장:** chain에 포함되지 않은 자매(sibling) 분기, 자식(descendant) 분기, **그리고 ancestor에서 분기점 이후에 이어진 turn**의 내용은 절대 포함되지 않는다.

**예시 — 잘못된 포함을 방지하는 규칙 확인:**

```
root chat 42
  turn 1
  turn 2
  turn 3   ← branch 87이 여기서 분기 (branchPointTurnId = turn 3)
  turn 4
  turn 5

branch chat 87 (parent_id=42, branchPointTurnId=turn 3)
  turn 1
  turn 2

사용자가 chat 87에서 새 메시지 송신 시 LLM에 전달되는 맥락:
  → chain = [42(root), 87(target)]
  → c₀ = 42: 다음 자식 87의 branchPointTurnId = turn 3, bp_seq = 3
        → turn_sequence ≤ 3 인 turn만 포함: turn 1, turn 2, turn 3
  → c₁ = 87 (target): 모든 turn 포함: turn 1, turn 2
  → 새 user 메시지 추가

결과: root.turn1, root.turn2, root.turn3, 87.turn1, 87.turn2, 새 메시지
(root.turn4, root.turn5는 분기 이후 자매 흐름이므로 제외)
```

**경계 케이스:**
- target이 root이면 chain은 `[root]` 하나, ancestor 규칙 적용 대상 없음. 모든 turn 포함.
- 분기점 turn 자체(`bp_seq`)는 부모 chat 쪽에 포함된다. target에는 이 turn이 없으며, target의 첫 turn부터 새 흐름을 쌓는다.
- chain 길이가 3 이상인 경우(손자 분기 등)에도 동일 규칙이 재귀적으로 적용된다. 각 ancestor는 *자신의 다음 chain 멤버*가 분기된 지점까지만 포함하므로 정확하다.

> 본 절차는 Phase 2 Sequence Diagram ① 단계 4(`buildContext`)와 Phase 3 Sequence Diagram ② 단계 12(`buildContext`)에 대응한다. Phase 4에서는 이를 명문화하고 ancestor 범위 규칙을 정확히 규정한다.

---

### 3.2 긴 대화 압축 규칙 (FR-5.2)

`flat_messages`의 누적 입력 토큰이 사용 중인 LLM 모델의 `contextWindow × compressionRatio` (기본값 `0.8`)를 초과하면 압축을 적용한다.

**모델별 임계값 (참고 기본값):**

| 모델 | `contextWindow` | 압축 시작 임계 (×0.8) |
|---|---|---|
| `gpt-4o-mini` | 128,000 | 102,400 |
| `gemini-2.0-flash` | 1,048,576 | 838,860 |
| `claude-3.5-sonnet` | 200,000 | 160,000 |

> 본 임계값은 운영 가능한 기본값이며, 별도의 런타임 설정(`application.yml`)으로 모델별로 조정 가능하도록 둔다. API 응답에는 노출되지 않는다.
>
> **[MVP 간소화]** Phase 4 구현에서는 `COMPRESSION_RATIO = 0.8`을 상수로 하드코딩하였다. `application.yml` 외부화는 운영 데이터 수집 후 조정이 필요한 시점에 진행한다.

**압축 절차:**

1. `flat_messages`를 시간순으로 정렬 (이미 그렇게 조립됨)
2. 가장 오래된 turn부터 차례로 다음 중 하나로 치환:
   - 해당 turn의 `summary`가 `GENERATED` 상태이면 → "이전 대화 요약: <summary>" 형태의 system 또는 user 보조 메시지 1건으로 치환
   - `summary`가 `PENDING`이면 → fallback으로 해당 turn의 user/assistant content를 각각 토큰 단위로 절단 (잘림 표시 포함, 예: "...[일부 생략]")
3. 1턴 치환 후 다시 토큰 합계를 측정. 임계 이하로 떨어지면 종료. 아니면 다음 오래된 turn에 대해 반복
4. **가장 최근 N턴은 절대 압축하지 않는다.** 기본 N=2 (현재 turn + 직전 turn). 이 보호 구간을 더 줄이지 못해 임계 초과가 계속되면, 더 이상 압축을 시도하지 않고 LLM 호출을 그대로 진행한다 (MVP 정책). 처리 규칙:
   - LLM provider가 `context_length_exceeded` 등의 에러를 반환하면 Phase 2·3과 동일하게 **`LLM_5001`**로 매핑한다. SSE 스트림 시작 전이면 JSON 에러 응답, 스트림 도중이면 `error` 이벤트로 통보한다.
   - 서버 로그에 `context_overflow_after_compression` 라벨로 경고를 남긴다. 운영 시 본 라벨의 빈도가 높으면 보호 구간 N 또는 압축 임계 비율을 조정한다.
   - 클라이언트는 사용자에게 "이번 대화의 맥락이 모델 한도를 초과했습니다. 분기를 만들어 흐름을 나누어 보세요" 식의 안내를 표시하도록 권장한다 (UI 정책, 본 명세 범위 외).

**결과 통보:**

SSE `turn_completed` 이벤트에 다음 세 필드를 **압축 여부와 무관하게 항상** 포함한다. 압축이 적용되지 않은 경우 `compressionApplied: false`, `compressedTurnCount: 0`이며, `contextTokens`는 실제 LLM 입력 토큰 수를 채운다:

| 필드 | 타입 | 설명 |
|---|---|---|
| `contextTokens` | Integer | 압축 적용 후 LLM에 실제로 전달된 입력 토큰 수 |
| `compressionApplied` | Boolean | 압축이 1턴 이상 적용되었는지 (summary 치환 또는 truncation 중 하나라도 발생) |
| `compressedTurnCount` | Integer | 본 호출에서 압축이 적용된 turn 수. **summary 치환된 turn과 truncation된 turn을 모두 합산.** `compressionApplied=false`이면 0 |

> Phase 4 단계에서는 단순화를 위해 summary 치환과 truncation을 하나의 카운터로 합산한다. 두 방식의 통계를 분리할 필요가 생기면 후속 Phase에서 `summarizedTurnCount` / `truncatedTurnCount` 필드를 추가한다.

> Phase 3 §4.1의 `turn_completed` 스키마에 위 세 필드가 추가된다. Phase 2의 일반 메시지 송신 API에도 동일 마이그레이션이 필요하다.

**관련 FR:** FR-5.1, FR-5.2

---

### 3.3 토큰 사용량 조회 (FR-5.3)

```http
GET /api/v1/usage/me
Authorization: Bearer <accessToken>
```

**인증 필요:** ✓

현재 사용자의 활성 기간(usage period) 토큰 사용량을 조회한다.

**Query Parameters**

없음 (현재 활성 기간이 자동 선택됨).

**응답 (200 OK)**

`result` 필드:

| 필드 | 타입 | 설명 |
|---|---|---|
| `usageRecordId` | Long | usage_record PK |
| `planId` | Long | 적용 중인 plan ID |
| `planName` | String | plan 이름 (예: "Free", "Pro") |
| `periodStart` | DateTime | 기간 시작 |
| `periodEnd` | DateTime | 기간 종료 |
| `tokensUsed` | Long | 누적 사용 토큰 |
| `tokenLimit` | Long | 기간 내 한도. `0`이면 무제한 |
| `requestCount` | Integer | 누적 요청 수 |
| `remainingTokens` | Long \| null | `tokenLimit - tokensUsed`. `tokenLimit=0`이면 null |
| `usageRatio` | Double \| null | `tokensUsed / tokenLimit`. 0~1. `tokenLimit=0`이면 null |
| `warningLevel` | Enum&lt;"NONE" \| "WARN" \| "CRITICAL"&gt; | `usageRatio` 기반. 0.8 이상이면 WARN, 0.95 이상이면 CRITICAL. `tokenLimit=0`이면 NONE |

```json
{
  "isSuccess": true,
  "code": "COMMON_200",
  "message": "요청에 성공하였습니다.",
  "result": {
    "usageRecordId": 1234,
    "planId": 2,
    "planName": "Pro",
    "periodStart": "2026-05-01T00:00:00Z",
    "periodEnd": "2026-05-31T23:59:59Z",
    "tokensUsed": 423500,
    "tokenLimit": 500000,
    "requestCount": 187,
    "remainingTokens": 76500,
    "usageRatio": 0.847,
    "warningLevel": "WARN"
  }
}
```

**처리 규칙**

1. `member_id = 인증사용자`이고 `period_start ≤ now() < period_end`인 usage_record를 조회
2. 활성 record가 없으면:
   - 사용자에게 plan이 있는 경우: 새 usage_record를 생성하고 0으로 초기화 후 반환
   - plan 자체가 없는 경우: 404 `USAGE_4041`

**토큰 누적 시점**

`usage_record.tokens_used`·`request_count`는 LLM 호출이 완료되는 시점(SSE `done` 이후 트랜잭션 안)에 다음 식으로 갱신한다:

```sql
UPDATE usage_record
SET tokens_used   = tokens_used + ?,        -- message.prompt_token + message.answer_token
    request_count = request_count + 1,
    compressed_turn_count = compressed_turn_count + ?
WHERE usage_record_id = ?;
```

**토큰 필드 의미 통일 (Phase 4 명문화):**

- `message.prompt_token` — **압축 적용 후 LLM에 실제로 전달된 입력 토큰 수**로 저장한다. SSE `turn_completed` 이벤트의 `contextTokens`와 동일한 값이다. 압축 전 원본 ancestor chain 전체의 토큰 수가 아니다.
- `message.answer_token` — LLM 응답의 출력 토큰 수.
- `usage_record.tokens_used` 누적 시 `message.prompt_token + message.answer_token`을 더한다.

> Phase 2 명세서에서 `prompt_token`의 정확한 의미가 명시되지 않아 구현자가 "사용자 메시지 토큰" 또는 "압축 전 전체 입력 토큰"으로 해석할 수 있었다. Phase 4부터는 **"실제 LLM에 전달된 입력 토큰 수"**로 통일한다. 기존 데이터 마이그레이션이 필요한 경우 별도 합의.

부분 응답(cancel·fail)도 그 시점까지의 토큰 수를 합산한다. 재생성(§Phase 3 §4.1)은 새 LLM 호출이므로 동일하게 누적한다.

**에러**

| HTTP | code | 시점 |
|---|---|---|
| 401 | `AUTH_4011` | |
| 404 | `USAGE_4041` | plan 없음 |

**관련 FR:** FR-5.3  
**SRS 우선순위:** 권장 (SRS §3.2.5)  
**Phase 4 구현 우선순위:** 필수 (FG-5가 Phase 4 범위에 포함되어 발표 시연에 사용량 표시가 필요)

---

### 3.4 SSE 이벤트 스키마 확장

Phase 3 §4.1의 SSE 이벤트에 다음을 추가·변경한다. Phase 2의 일반 메시지 송신(`POST /chats/{id}/messages`)에도 동일하게 적용한다.

**`turn_completed` (변경)**

| 필드 | 타입 | 설명 |
|---|---|---|
| `turnId` | Long | (기존) |
| `aiMessageId` | Long | (기존) |
| `answerToken` | Integer | (기존) |
| `summaryStatus` | Enum&lt;"PENDING"&gt; | (Phase 3에서 추가) |
| `contextTokens` | Integer | **Phase 4 신규. 항상 전송.** §3.2 참조 |
| `compressionApplied` | Boolean | **Phase 4 신규. 항상 전송.** §3.2 참조 |
| `compressedTurnCount` | Integer | **Phase 4 신규. 항상 전송.** §3.2 참조 |

```
event: turn_completed
data: {
  "turnId": 207,
  "aiMessageId": 415,
  "answerToken": 320,
  "summaryStatus": "PENDING",
  "contextTokens": 98430,
  "compressionApplied": true,
  "compressedTurnCount": 5
}
```

이 변경은 클라이언트가 `usageRatio` 등을 즉시 갱신하거나 압축 알림을 띄울 수 있게 한다. `GET /usage/me`를 매번 호출하지 않아도 된다.

**관련 FR:** FR-5.2, FR-5.3

---

## 4. 처리 시점 정리

다음 표는 Phase 4의 각 작업이 어느 시점에 수행되는지 정리한다.

| 작업 | 시점 | 비고 |
|---|---|---|
| 탐색기 트리 조립 | 조회 시점 | 페이징된 root 묶음 1쿼리 (`root_chat_id IN (:pagedRootIds)`) + turn count `GROUP BY` 1쿼리 |
| 트리 노드 정렬 | 조회 시점 | DFS 메모리 정렬 |
| **맥락 조립 (FR-5.1)** | **LLM 호출 직전** | ancestor chain 따라 turn 펼침 |
| **압축 (FR-5.2)** | **LLM 호출 직전** | 토큰 임계 초과 시에만. summary 캐시 활용 |
| `chat.last_activity_at` 갱신 | 메시지 송신·재생성·수정 트랜잭션 끝 | ancestor chain까지 max 갱신 |
| `usage_record` 누적 | SSE `done` 직전 트랜잭션 | promptToken+answerToken 합산 |
| Summary 생성 (Phase 3 잔류) | 저장 시점 비동기 | 압축에 활용되므로 신뢰성 중요 |

---

## 5. 시나리오 예시

### 5.1 사이드바 첫 진입 → 탐색기 렌더링 → 특정 분기 클릭

```
1. 로그인 직후 사이드바가 탐색기를 요청
   → GET /api/v1/chats/explorer?sort=recent&page=0&size=20
   ← roots: [...] (각 root별 분기 트리 포함)

2. 프론트가 트리를 렌더링. 사용자는 root chat 42 아래의 분기 chat 87 클릭

3. 클라이언트가 대화 내용 로딩 + 그래프 로딩을 병렬로 요청
   → GET /api/v1/chats/87/turns?lastTurnSequence=0&limit=20
   → GET /api/v1/chats/87/graph
```

### 5.2 매우 긴 대화에서 메시지 송신 → 압축 적용

```
1. 사용자가 chat 87(turn 30+개 누적, 부모 chat 42의 turn 105에서 분기)에 새 메시지 입력
   → POST /api/v1/chats/87/messages { content: "..." }

2. 서버 내부:
   a. ancestor chain 조립 (§3.1):
      - chain = [42(root), 87(target)]
      - chat 42: 다음 자식 87의 branchPointTurnId = turn 105
                 → turn_sequence ≤ 105인 turn만 포함 (turn 1~105)
      - chat 87 (target): 모든 turn 포함 (turn 1~30)
      - 새 user 메시지 끝에 추가
   b. flat_messages 누적 토큰 = 130,000 → gpt-4o-mini 한도(128K)의 80% = 102,400 초과
   c. 압축: 가장 오래된 turn(chat 42의 turn 1)부터 차례로 처리
      - turn 1~3: summary가 GENERATED → summary 메시지로 치환
      - turn 4~5: summary가 PENDING → content truncation 적용
   d. 5개 turn 압축 후 누적 토큰 = 95,000 → 임계 이하. 압축 종료
   e. LLM 호출 시작

3. SSE 이벤트:
   event: turn_started
   data: {chatId: 87, turnId: 200, ...}

   event: chunk × N

   event: turn_completed
   data: {
     turnId: 200, aiMessageId: 415, answerToken: 320,
     summaryStatus: "PENDING",
     contextTokens: 95320,
     compressionApplied: true,
     compressedTurnCount: 5
   }

   event: done

4. 트랜잭션 종료:
   - chat.last_activity_at 갱신 (87 → 42)
   - usage_record.tokens_used += 95320 + 320
   - usage_record.compressed_turn_count += 5
```

### 5.3 토큰 사용량 경고

```
1. 프론트는 매 turn_completed의 contextTokens를 누적치에 더해 캐시
2. 일정 간격(또는 사용자 요청 시) /usage/me 호출하여 정확치 동기화
   → GET /api/v1/usage/me
   ← warningLevel: "WARN" (usageRatio = 0.847)
3. 프론트가 상단에 노란 배너 표시 — "이번 달 토큰 사용량 84.7% (76,500 토큰 남음)"
```

### 5.4 삭제 분기 복구 UI

```
1. 사용자가 "휴지통" 메뉴 클릭
   → GET /api/v1/chats/explorer?includeDeleted=true
   ← roots에 deletedAt이 not null인 노드가 함께 반환됨

2. 사용자가 chat 87 복구 요청 (Phase 3 §2.3의 cascade soft delete 반대 방향)
   → 본 명세 범위 외. SRS FR-2.5의 복구 정책은 후속 결정
```

---

## 6. ERD ↔ API 필드 매핑 (Phase 2·3과의 연속)

Phase 3 §7의 매핑에 다음을 추가한다.

### chat 테이블 → 응답 DTO (Phase 4 추가분)

| ERD 컬럼 | API 필드 | 비고 |
|---|---|---|
| `last_activity_at` | `lastActivityAt` | 탐색기 응답 및 차기 Phase 2 `GET /chats` 응답에도 노출 검토 |

### usage_record 테이블 → 응답 DTO

| ERD 컬럼 | API 필드 | 비고 |
|---|---|---|
| `usage_record_id` | `usageRecordId` | |
| `member_id` | (응답 미포함) | 인증으로 자동 처리 |
| `plan_id` | `planId` | |
| `period_start` | `periodStart` | |
| `period_end` | `periodEnd` | |
| `tokens_used` | `tokensUsed` | |
| `token_limit` | `tokenLimit` | |
| `request_count` | `requestCount` | |
| `compressed_turn_count` | (응답 미포함) | 운영 관측용 |

### plan 테이블 → 응답 DTO

`/usage/me` 응답의 `planName`만 외부에 노출. 그 외 필드(`max_project`, `max_depth` 등)는 Phase 4 범위 외.

---

## 7. Phase 4 엔드포인트 목록

| Method | Path | 설명 | 인증 | FR | 우선순위 |
|---|---|---|---|---|---|
| GET | `/api/v1/chats/explorer` | 탐색기 트리 조회 | ✓ | FR-4.2, FR-4.3 | 필수 |
| GET | `/api/v1/usage/me` | 토큰 사용량 조회 | ✓ | FR-5.3 | 필수 |
| GET | `/api/v1/chats/explorer/{rootChatId}` | 단일 트리 새로고침 | ✓ | FR-4.2 | 선택 |

총 신규 엔드포인트 3개 (필수 2, 선택 1). Phase 2·3의 18개와 합쳐 누적 21개.

**Phase 4 변경 사항 (기존 API):**

- **SSE `turn_completed` 스키마 확장** — Phase 3에서 `summaryStatus` 추가에 이어, Phase 4에서 `contextTokens`, `compressionApplied`, `compressedTurnCount` 필드 추가. Phase 2의 일반 메시지 송신, Phase 3의 재생성·수정 API에 모두 적용.
- **`POST /messages`·재생성·수정 내부 동작** — §3.1·§3.2의 맥락 조립·압축 규칙 적용. 외부 호출 인터페이스는 변경 없음.
- **`message.prompt_token` 의미 명문화** — §3.3 참조. "압축 후 실제 LLM 입력 토큰 수"로 통일.
- **`chat.last_activity_at` 노출** — 차기 마이너 버전에서 Phase 2의 `GET /chats` 목록 응답과 Phase 3 graph 응답에도 노출 검토.

**우선순위 정책** (Phase 3과 동일):
- **필수**: Phase 4 발표·회귀 테스트 통과를 위해 반드시 구현되어야 하는 엔드포인트. FR-5.3은 SRS에서 "권장"이나, Phase 4가 FG-5 맥락 관리를 범위에 포함하므로 발표 시연을 위해 필수로 승격.
- **선택**: 동등한 기능을 다른 방식으로 충족 가능. 시간 부족 시 제외 가능.

---

## 8. 성능 고려 (NFR-P-2, NFR-P-4)

### 8.1 탐색기 응답 페이로드 예상

- `ExplorerNodeDto`: 약 200 B/개
- 평균 사용자: root 10개, root당 평균 분기 5개 = 노드 60개 = **12 KB** (gzip 시 3~4 KB)
- 헤비 사용자: root 50개, root당 분기 30개 = 노드 1,500개 = **300 KB** — 본 케이스는 §2.2 단일 트리 조회 + 페이징으로 대응

### 8.2 탐색기 쿼리 패턴

```sql
-- 1쿼리: 페이징된 root + 트리
SELECT * FROM chat
WHERE member_id = ?
  AND (deleted_at IS NULL OR :includeDeleted = true)
  AND root_chat_id IN (
    SELECT chat_id FROM chat
    WHERE member_id = ? AND parent_id IS NULL
      AND (deleted_at IS NULL OR :includeDeleted = true)
    ORDER BY last_activity_at DESC, chat_id DESC
    LIMIT :size OFFSET :page * :size
  );
-- idx_chat_member_last_activity + idx_chat_root_deleted 활용
-- last_activity_at은 NOT NULL이므로 NULLS 처리 불필요

-- 1쿼리: turn count 일괄 조회
SELECT chat_id, COUNT(*) FROM turn
WHERE chat_id IN (...)
GROUP BY chat_id;
-- idx_turn_chat_sequence 활용
```

총 2쿼리. N+1 없음.

### 8.3 맥락 조립·압축 비용

- ancestor chain 조회: chain 길이만큼의 chat row 단순 SELECT. 일반적으로 depth ≤ 5
- turn 펼침: chat_id IN (chain) + turn_sequence ≤ 분기점 조건. `idx_turn_chat_sequence` 활용
- 토큰 카운트: tokenizer 라이브러리(예: `tiktoken-java`) 활용. message 1건당 약 0.5ms
  > **[MVP 간소화]** Phase 4 구현에서는 tokenizer 라이브러리 대신 `characters / 4` 간이 추정을 사용하였다. 정밀한 토큰 카운팅이 필요한 시점에 tiktoken 등으로 교체한다.
- 압축 시 summary는 turn row에 이미 캐시되어 있으므로 추가 LLM 호출 없음

목표: ancestor chain 길이 5, turn 100건 기준 맥락 조립 ≤ 200ms.

### 8.4 캐시 정책

본 명세에서는 별도 캐시 계층을 도입하지 않는다. `last_activity_at` 컬럼화와 인덱스로 NFR-P-2(3초) 충족 가능. 사용자 수 증가 시 탐색기 응답을 Redis 등으로 캐싱하는 방안은 후속 검토.

---

## 9. 미해결 / 향후 결정 사항

1. **압축 임계값과 보호 구간 N의 정확한 값** — `compressionRatio=0.8`, 보호 구간 `N=2`는 초기값. 운영 데이터 수집 후 조정.
2. **압축 시 summary 형식** — system 메시지로 넣을지, user role 보조 메시지로 넣을지. LLM provider별로 효과 차이 가능. 통합 테스트에서 확정.
3. **`tokenLimit=0` (무제한 plan) 처리** — 현재는 `warningLevel="NONE"` 고정. 별도의 절대 한도 경고(예: 1M 토큰 초과 시 알림)는 후속.
4. **삭제 분기 복구 UI 및 API** — SRS FR-2.5의 "일정 기간 복구 가능"을 구체화한 복구 엔드포인트는 본 명세 범위 외. 휴지통 조회는 §2.1의 `includeDeleted=true`로 대체.
5. **`chat.last_activity_at` 갱신의 트랜잭션 비용** — depth가 깊은 chain에서 매 메시지 전송 시 ancestor 전체를 UPDATE하는 부하. 분기 깊이 ≤ 5라는 가정 하에 허용. 깊이가 깊어지면 비동기 갱신으로 분리 검토.
6. **Phase 2 `GET /chats` 응답에 `lastActivityAt` 노출** — 본 명세는 신규 엔드포인트에만 노출. Phase 2 응답 확장은 별도 결정.
7. **비동기 작업 완료 통보 (Phase 3 §10에서 이연)** — summary 생성 완료를 WebSocket/SSE push로 통보할지, polling 유지할지. Phase 4에서 결정하기로 한 사항이나, 본 명세는 polling 유지로 가정하고 작성. 추후 변경 시 별도 보강.

---

## 10. 변경 이력

| 버전 | 날짜 | 변경 사유 |
|---|---|---|
| 0.1 | 2026-05-23 | Phase 4 초안. FG-4 탐색기 트리 API, FG-5 맥락 조립·압축 규칙 명문화, FR-5.3 토큰 사용량 API. SSE `turn_completed` 스키마에 압축 결과 필드 추가. Phase 3 ERD 대비 `chat.last_activity_at` 컬럼·인덱스 추가, `usage_record` 활성화. Phase 2·3 명세서 컨벤션 준수. |
| 0.2 | 2026-05-24 | 리뷰 반영. (1) §3.1 맥락 조립 규칙 정정 — ancestor chat은 chain의 다음 child가 갖는 `branchPointTurnId`까지만 포함하도록 변경. 기존의 "root는 모든 turn 포함"은 FR-5.1 분기 격리 원칙을 위반하므로 제거. 예시와 시나리오 5.2도 함께 정정. (2) `CONTEXT_5031` 에러 코드 삭제 — 동기 fallback 압축 원칙과 충돌. (3) `compressedTurnCount` 정의 정정 — summary 치환과 truncation을 모두 합산. (4) `message.prompt_token` 의미를 "압축 후 실제 LLM 입력 토큰 수"로 명문화 (= `contextTokens`). (5) `last_activity_at` 초기값을 `created_at`으로 변경하여 not null로 강제. 빈 채팅이 정렬 맨 뒤로 영구히 밀리는 문제 해결. 기존 row 마이그레이션 규칙 명시. (6) 탐색기 `turnCount` 조회를 페이징된 root 묶음 기준 단일 `GROUP BY` 쿼리로 명시화 (root별 N+1 금지). (7) `GET /usage/me` 우선순위를 권장에서 필수로 승격. 우선순위 정책을 Phase 3 명세서와 동일하게 통일. |
| 0.3 | 2026-05-24 | 리뷰 2차 반영. (1) `last_activity_at` ALTER SQL을 NOT NULL 강제와 일관되도록 2단계 마이그레이션(ADD NULL → 백필 → MODIFY NOT NULL)으로 변경. v0.2의 본문/SQL 내부 충돌 해소. (2) §2.1 처리 규칙 4번의 "페이징된 root 각각" 표현 제거. `root_chat_id IN (:pagedRootIds)` 단일 쿼리로 일괄 조회한다는 §8.2 SQL 예시와 일치하도록 명시화. (3) `last_activity_at` 갱신 시점 확장 — 메시지 송신·재생성·수정에 더해 **branch 생성**, branch 복구, chat 제목 수정을 포함. branch 삭제는 갱신 대상에서 제외(사용자 모델과 정합). branch 생성 시 부모 root가 정렬에서 묻히는 문제 해결. (4) §3.3 `/usage/me`의 FR 우선순위 표기를 SRS 우선순위(권장)와 Phase 4 구현 우선순위(필수)로 분리. (5) §3.2 압축 실패 후 LLM 호출 정책 명확화 — provider의 `context_length_exceeded`는 Phase 2·3과 동일하게 `LLM_5001`로 매핑하고, 서버 로그에 `context_overflow_after_compression` 라벨로 경고 기록. MVP 정책으로 사전 차단 대신 시도 후 통과. |
| 0.4 | 2026-05-24 | 리뷰 3차 반영 — 문서/SQL 표현 일관성 정리. (1) §4 처리 시점 표의 "root당 1쿼리" 표현 정정 — v0.3에서 §2.1·§8.2를 IN 쿼리로 통일했으나 §4 표만 누락된 잔존 표현. "페이징된 root 묶음 1쿼리 (`root_chat_id IN (:pagedRootIds)`) + turn count `GROUP BY` 1쿼리"로 명시. (2) §8.2 SQL 예시의 `ORDER BY last_activity_at DESC NULLS LAST` 제거 — `last_activity_at`이 NOT NULL이므로 NULLS 처리 불필요하며 MySQL은 표준 `NULLS LAST` 키워드를 미지원. `ORDER BY last_activity_at DESC, chat_id DESC`로 변경. (3) §2.1 `name` 정렬 규칙을 MySQL/JPA 호환 표현으로 변경 — `title ASC NULLS LAST` 대신 자연어 설명("title이 null인 항목은 마지막에 배치")과 구현 예 `ORDER BY (title IS NULL) ASC, LOWER(title) ASC, chat_id DESC` 병기. |
| 0.5 | 2026-05-25 | ERD 표기 정리. (1) 기준 ERD 표기를 내부 버전 `ait_erd_v2.2`에서 파일명 기반 `ait_erd_phase4`로 변경. Phase 2·3 명세서가 사용한 `ait_erd_v2`, `ait_erd_v2.1` 표기 대신 phase 명을 직접 쓰는 방식으로 통일하여 ERD 파일과 명세서 간 매칭을 명확화. (2) §1 제목을 "ERD v2.2 변경분"에서 "ERD 변경분 (Phase 3 → Phase 4)"으로 변경. 본문에 통합 스키마 파일 `AIT_ERD_phase4.txt` 참조 명시. **스키마 정의 자체는 변경 없음** — v0.4와 동일하게 `chat.last_activity_at`, `usage_record.compressed_turn_count` 추가 및 인덱스 2개 유지. |
| 0.6 | 2026-05-25 | ERD 리뷰 반영 — Phase 3 ERD에 남아 있던 정책 모호성 정리. (1) `chat.last_turn_id`의 `DEFAULT 0` 제거. `turn_id = 0`은 존재하지 않는 의미 없는 기본값. `NULL`로 시작하고 첫 turn 저장 시 갱신한다. (2) `chat.root_chat_id`를 `NULL` → `NOT NULL`로 변경하고 **root chat도 자기 `chat_id`를 저장**하도록 정책 통일. 탐색기·그래프 쿼리가 `root_chat_id IN (:pagedRootIds)` 한 줄로 처리되도록 단순화. JPA 구현 시 생성 트랜잭션 내 self-update 흐름 명문화. (3) §1.3 FK 관리 방식 추가 — FK 제약은 ERD에 명시하지 않고 JPA 연관관계(`@ManyToOne`, `@JoinColumn`)로 관리한다는 점을 명문화. (4) ERD 파일의 SQL 표기 정리 — `String(20)` → `varchar(20)`, enum 컬럼들의 허용값을 `enum('VAL1', 'VAL2', ...)` 형식으로 명시. 스키마 정의 자체는 동일하며 표기만 SQL 친화적으로 변경. |
| 0.7 | 2026-05-26 | 구현 리뷰 반영. (1) `chat.root_chat_id`를 NOT NULL에서 **NULL 유지**로 변경. `@GeneratedValue(IDENTITY)` 특성상 INSERT 시점에 PK를 알 수 없어 NOT NULL 제약 시 임시값(0L) 우회가 필요하며, 이는 `initRootChatId()` 호출 누락 시 잘못된 쿼리 결과를 유발할 수 있다. 앱 레벨(`initRootChatId()`)에서 non-null을 보장하고 DDL은 nullable로 둔다. §1.2 및 ERD 파일 동시 수정. (2) §2.1 처리 규칙 6번 `turnCount` 쿼리를 서브쿼리 방식에서 Phase 3의 `countByChatIds(chatIds)` 재사용 방식으로 변경. 배치 조회된 chatId 목록을 직접 전달하여 동일 결과를 더 단순하게 얻는다. (3) §3.2 압축 비율 `application.yml` 외부화를 MVP 간소화 사항으로 주석 추가. (4) §8.3 토큰 추정을 `characters / 4` 간이 방식으로 MVP 간소화 사항 주석 추가. |
| 0.8 | 2026-05-26 | FE 팀 리뷰 반영 (P4F-1~4). (1) §3.2·§3.4 `turn_completed`의 `contextTokens`·`compressionApplied`·`compressedTurnCount`를 **압축 여부와 무관하게 항상 전송**으로 정정. §5.3 시나리오의 "매 턴 누적 캐시"와 일관성 확보. (2) §0.2에서 `CONTEXT_4001` 에러 코드 제거 — 맥락 조립은 서버 내부 동작이며 대응 엔드포인트가 없어 트리거 불가. `ErrorStatus.java`에서도 동시 제거. (3) §2.1 처리 규칙에 빈 사용자 응답 명시 — `roots: []`, `totalRootCount: 0`, `hasNext: false` (정상 200). (4) §3.3 `usageRatio` 타입을 `Number`에서 `Double`로 변경하여 타입 표기 컨벤션 통일. |
