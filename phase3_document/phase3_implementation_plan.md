# Phase 3 구현 계획

## 1. 현재 프로젝트 구조 분석 결과

| 계층 | 파일 | 비고 |
|---|---|---|
| Entity | `Chat.java`, `Turn.java`, `Message.java` | `Chat` PK가 `aichat_session_id`, `Turn` FK도 동일 |
| Repository | `ChatRepository`, `TurnRepository`, `MessageRepository` | 기본 CRUD + cursor 페이징 |
| Service | `ChatService`, `MessageService`, `TurnService` | 대화 CRUD, SSE 스트리밍, 턴 조회 |
| Controller | `ChatController`, `MessageController` | `/chats/`, `/chats/{id}/messages/` |
| Converter | `ChatConverter`, `TurnConverter` | Entity → DTO 변환 |
| DTO | `ChatReqDto`, `ChatResDto`, `MessageReqDto`, `MessageResDto`, `TurnResDto` |  |
| Enum | `SenderType`, `MessageStatus`, `LlmModel`, `LlmProvider`, `CursorDirection` |  |

---

## 2. ERD 변경분 반영 필요 파일

### 2.1 컬럼명 변경

| 변경 | 현재 코드 | 수정 필요 파일 |
|---|---|---|
| `aichat_session_id` → `chat_id` (`Chat` PK) | `Chat.java:24 @Column(name = "aichat_session_id")` | `Chat.java` |
| `aichat_session_id` → `chat_id` (`Turn` FK) | `Turn.java:37 @JoinColumn(name = "aichat_session_id")` | `Turn.java` |
| `Summary` → `summary` | `Turn.java:28`에서 이미 소문자 `summary`로 매핑됨 | 변경 불필요 |

### 2.2 신규 컬럼 추가

| 컬럼 | 엔티티 | 현재 상태 | 수정 내용 |
|---|---|---|---|
| `chat.last_turn_id` | `Chat.java` | 없음 | `Long lastTurnId` 필드 추가 |
| `chat.title_status` | `Chat.java` | 없음 | `TitleStatus titleStatus` 필드 추가 + 새 enum 생성 |
| `turn.summary` nullable | `Turn.java:28` | `@Builder.Default private String summary = "제목없음"` | default 제거, nullable 유지 (`null = PENDING`) |

### 2.3 새 Enum

```java
public enum TitleStatus {
    PENDING,
    GENERATED,
    USER_EDITED
}
```

### 2.4 인덱스

JPA `@Table(indexes = ...)` 또는 SQL 마이그레이션으로 추가.

```sql
CREATE INDEX idx_chat_root_deleted
ON chat(root_chat_id, deleted_at);

CREATE INDEX idx_chat_parent
ON chat(parent_id);

CREATE INDEX idx_chat_branch_point
ON chat(branch_point_turn_id);

CREATE INDEX idx_turn_chat_sequence
ON turn(chat_id, turn_sequence);

CREATE INDEX idx_message_turn
ON message(turn_id);
```

### 2.5 수정 파일 목록 — ERD 관련

| 파일 | 변경 |
|---|---|
| `Chat.java` | PK 컬럼명 변경, `lastTurnId`/`titleStatus` 추가, `@Table(indexes = ...)` 추가 |
| `Turn.java` | FK 컬럼명 변경, `summary` default 제거 (`null` 허용) |
| `Message.java` | `@Table(indexes = ...)` 추가 (`idx_message_turn`) |
| `TitleStatus.java` | 신규 enum 생성 |

---

## 3. Phase 2 기존 API와의 충돌 확인

### 3.1 충돌 없음 — 유지

- `SenderType`: `USER` / `ASSISTANT` 유지
- `DELETE /chats/{chatId}`: 기존 soft delete 로직 존재 → cascade 로직만 추가

### 3.2 변경 필요

| 항목 | 현재 Phase 2 | Phase 3 변경 |
|---|---|---|
| SSE `turn_completed` | `{ aiMessageId, summary, answerToken }` | `summary` 제거, `summaryStatus: "PENDING"` 추가 |
| `MessageResDto.TurnCompleted` | `summary` 필드 있음 | `summary` → `summaryStatus` (`String "PENDING"`) |
| `MessageService.streamMessage()` 완료 처리 | summary를 동기 생성 후 `turn_completed`에 포함 | summary 생성은 비동기 worker로 분리. `turn_completed`에는 `summaryStatus: "PENDING"`만 전송 |
| `Turn.summary` 초기값 | `"제목없음"` (`Builder.Default`) | `null` (`요약 미생성 상태`) |
| `Chat` 생성 시 | `titleStatus` 없음 | `titleStatus = PENDING` 기본값 설정 필요 |
| `ChatResDto.Detail` | `titleStatus` 없음 | `titleStatus`, `branchPointTurnId` 필드 추가 |

### 3.3 수정 파일 목록 — 호환성

| 파일 | 변경 |
|---|---|
| `MessageResDto.java` | `TurnCompleted` record에서 `summary` → `summaryStatus` |
| `MessageService.java` | `streamMessage()` 완료 처리: summary 동기 생성 제거, `summaryStatus: "PENDING"` 전송, summary 비동기 생성 호출 |
| `ChatResDto.Detail` | `titleStatus`, `branchPointTurnId` 필드 추가 |
| `ChatConverter.java` | `toDetail()`에 `titleStatus`, `branchPointTurnId` 매핑 |

---

## 4. Phase 3 API 구현 순서

### 단계 0. 기반 작업 — ERD 변경 + 에러 코드

모든 API의 전제 조건.

- Entity 변경
  - `Chat`
  - `Turn`
  - `Message` 인덱스
- `TitleStatus` enum 생성
- `ErrorStatus`에 Phase 3 에러 코드 추가
  - `BRANCH_4001`
  - `GRAPH_4001`
  - `BRANCH_4041`
  - `TURN_4041`
  - `BRANCH_4091`
  - `BRANCH_4092`
  - `MESSAGE_4092`
- Phase 2 SSE `turn_completed` 스키마 변경
  - `summary` → `summaryStatus`
- `ChatResDto.Detail`에 `titleStatus`, `branchPointTurnId` 추가
- Chat 생성 시 `lastTurnId` 갱신 로직 추가
  - `MessageService`에서 `Turn` 생성 후 `chat.lastTurnId` 세팅
- 검증
  - 기존 테스트/빌드가 깨지지 않는지 확인

---

### 단계 1. `POST /chats/{chatId}/branches` — 분기 생성

분기 구조의 핵심. 이후 API들이 이 기능에 의존.

#### 수정/추가 대상

- `ChatReqDto`
  - `BranchCreate` record 추가
  - 필드: `branchPointTurnId`, `title`
- `ChatResDto`
  - `Detail`에 필요한 필드 추가 완료 상태 전제
- `ChatRepository`
  - `findAllByRootChatIdAndDeletedAtIsNull()` 추가
- `TurnRepository`
  - `findByIdAndChatId()` 추가
  - `branchPointTurn` 검증용
- `ChatService`
  - `createBranch()` 메서드 추가
  - turn 존재 검증
  - 소유권 검증
  - 새 chat row 생성
- `ChatController`
  - `POST /chats/{chatId}/branches` 엔드포인트 추가

#### 검증

- 분기 생성 후 DB 확인
  - `parent_id`
  - `branch_point_turn_id`
  - `root_chat_id`
  - `title_status`

---

### 단계 2. `PATCH /chats/{chatId}` — 분기/채팅 제목 수정

#### 수정/추가 대상

- `ChatReqDto`
  - `UpdateTitle` record 추가
  - 필드: `title`
- `Chat` entity
  - `updateTitle()` 메서드 추가
- `ChatService`
  - `updateChatTitle()` 메서드 추가
- `ChatController`
  - `PATCH /chats/{chatId}` 엔드포인트 추가

#### 검증

- 제목 수정 후 `titleStatus = USER_EDITED` 확인

---

### 단계 3. `DELETE /chats/{chatId}` — cascade 삭제

기존 삭제 API 업그레이드.

#### 수정/추가 대상

- `ChatRepository`
  - `findAllByParentId()` 또는 재귀 자손 조회 쿼리 추가
- `ChatService`
  - `deleteChat()` 수정
  - 자손 chat도 함께 soft delete
  - 이미 삭제된 경우 `BRANCH_4091` 반환
- `ChatController`
  - 기존 `DELETE` 엔드포인트 재사용

#### 검증

- 부모 삭제 시 자손도 `deleted_at` 설정 확인

---

### 단계 4. `GET /chats/{chatId}/graph` — 그래프 조회

가장 복잡한 API. 새 DTO와 서비스 로직 다수 필요.

#### 새 DTO

`GraphResDto` 또는 `ChatResDto` 내부 nested DTO로 구성.

- `GraphResult`
  - `rootChatId`
  - `center`
  - `chats`
  - `turns`
  - `frontier`
- `CenterDto`
  - `turnId`
  - `chatId`
- `ChatNodeDto`
  - `chatId`
  - `title`
  - `titleStatus`
  - `parentChatId`
  - `branchPointTurnId`
  - `depth`
  - `isDeleted`
  - `lastTurnId`
  - `updatedAt`
- `TurnNodeDto`
  - `turnId`
  - `chatId`
  - `turnSequence`
  - `summary`
  - `summaryStatus`
  - `isBranchPoint`
  - `isCurrent`
  - `createdAt`
- `FrontierDto`
  - `up`
  - `down`
- `FrontierPoint`
  - `fromTurnId`
  - `hasMore`

#### 수정/추가 대상

- `GraphConverter`
  - Entity → `GraphResDto` 변환
  - `isBranchPoint` 계산
  - `isCurrent` 계산
  - `depth` 계산
  - `summaryStatus` 계산
- `ChatRepository`
  - `findAllByRootChatId()`
  - `includeDeleted` 옵션 포함
- `TurnRepository`
  - `findByChatIdAndTurnSequenceBetween()`
  - `findByChatIdIn()` 등 window 탐색 쿼리
- `GraphService`
  - `centerTurnId` fallback 규칙
  - `UP` ancestor path 탐색
  - `DOWN` BFS 탐색
  - frontier 계산
- `ChatController` 또는 신규 `GraphController`
  - `GET /chats/{chatId}/graph` 엔드포인트 추가

#### 검증

- `centerTurnId` 기준 window 범위
- `frontier.hasMore`
- `isBranchPoint` 마킹
- fallback 규칙

---

### 단계 5. `GET /chats/{chatId}/graph/expand` — 윈도우 확장

#### 수정/추가 대상

- `GraphResDto`
  - `ExpandResult`
    - `direction`
    - `turns`
    - `frontier`
- `GraphService`
  - `expandWindow()`
  - `UP`: ancestor path 추적
  - `DOWN`: BFS
- `GraphController`
  - `GET /chats/{chatId}/graph/expand` 엔드포인트 추가

#### 검증

- expand 후 새 frontier가 정확한지 확인

---

### 단계 6. `POST /chats/{chatId}/messages/{messageId}/regenerate` — 응답 재생성

#### 수정/추가 대상

- `MessageResDto`
  - `BranchCreated` record 추가
  - 필드: `newChatId`, `branchPointTurnId`, `title`, `titleStatus`
- `MessageService`
  - `regenerateMessage()` 추가
  - §4.1.1 분기점 결정 규칙 구현
  - 자동 분기 생성
  - SSE `branch_created` 이벤트 전송
  - 기존 스트리밍 로직 재사용
- `MessageController`
  - `POST /chats/{chatId}/messages/{messageId}/regenerate` 엔드포인트 추가

#### 검증

- 분기 자동 생성 확인
- SSE 이벤트 시퀀스 확인

```text
branch_created → turn_started → chunk* → turn_completed → done
```

---

### 단계 7. `PATCH /chats/{chatId}/messages/{messageId}` — 메시지 수정

#### 수정/추가 대상

- `MessageReqDto`
  - `Edit` record 추가
  - 필드: `content`
- `MessageService`
  - `editMessage()` 추가
  - regenerate와 유사하나 수정된 `content` 사용
- `MessageController`
  - `PATCH /chats/{chatId}/messages/{messageId}` 엔드포인트 추가

#### 검증

- 수정된 content로 새 분기 생성
- SSE 흐름 확인

```text
branch_created → turn_started → chunk* → turn_completed → done
```

---

## 5. 테스트 전략

### 5.1 단위 테스트

#### `GraphService`

- window 계산
  - `UP`
  - `DOWN`
- frontier 계산
- `centerTurnId` fallback 규칙

#### 분기점 결정 규칙 (§4.1.1)

- `turnSequence > 1`
- `turnSequence = 1 + branch`
- `turnSequence = 1 + root`

---

### 5.2 통합 테스트 — API 레벨

| API | 주요 검증 |
|---|---|
| 분기 생성 | `parent`/`root` 관계 세팅, `titleStatus = PENDING`, 소유권 검증 |
| 제목 수정 | `titleStatus = USER_EDITED`로 전환 |
| cascade 삭제 | 자손도 함께 `deleted_at` 세팅, 이미 삭제된 chat이면 409 |
| 그래프 조회 | `chats` 전체 반환, `turns` window 범위, frontier, `isBranchPoint` |
| 윈도우 확장 | 증분 로딩, frontier 갱신 |
| 재생성 | 자동 분기 생성 + SSE 이벤트 시퀀스 |
| 메시지 수정 | 수정된 content로 새 분기 + SSE |

---

### 5.3 Edge Case

- 빈 분기(turn 없음)에서 그래프 조회
  - fallback 규칙 3
- root chat의 첫 turn 재생성 시도
  - `MESSAGE_4092`
- 삭제된 chat에서 분기 시도
  - `BRANCH_4001`
- `up`/`down` 파라미터 범위 위반
  - `GRAPH_4001`

---

## 6. 보류 사항

- `POST /chats/{chatId}/restore`
  - 복구 API는 우선순위 낮음
  - Phase 4 이연 가능
- `usage_record` 테이블 보강
  - `token_limit`, `plan_id`
  - Phase 3 API에서 직접 사용하지 않으므로 ERD 반영만 하고 서비스 로직은 보류
- 비동기 summary/title 생성 worker
  - 실제 LLM 호출은 `MockLlmClient` 기준으로 간단히 구현
  - 실 LLM 연동은 별도
