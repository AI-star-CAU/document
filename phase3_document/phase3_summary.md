# Phase 3 구현 요약

## 1. 개요

Phase 2(Walking Skeleton)에서 구현한 기본 대화 기능 위에, **분기(Branch) 트리 구조**와 **그래프 시각화 API**를 추가하여 대화 이력을 트리 형태로 관리할 수 있도록 확장하였다.

**명세서**: `api_phase3_v4.md` (v0.8)  
**ERD**: `AIT ERD phase3.txt / .png`

---

## 2. 구현 API 목록

### 2.1 Phase 2에서 이어지는 API (기존)

| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | `/auth/signup` | 회원가입 |
| POST | `/auth/login` | 로그인 (JWT 발급) |
| GET | `/members/me` | 내 정보 조회 |
| POST | `/chats` | 새 대화 시작 |
| GET | `/chats` | 대화 목록 조회 (Offset 페이징) |
| GET | `/chats/{chatId}` | 대화 메타정보 조회 |
| GET | `/chats/{chatId}/turns` | 턴 목록 조회 (Cursor 페이징) |
| POST | `/chats/{chatId}/messages` | 메시지 송신 (SSE 스트리밍) |
| POST | `/chats/{chatId}/messages/{messageId}/cancel` | 스트리밍 취소 |

### 2.2 Phase 3 신규 API

| Method | Endpoint | 설명 | 명세서 |
|--------|----------|------|--------|
| POST | `/chats/{chatId}/branches` | 분기 생성 | §2.8 |
| PATCH | `/chats/{chatId}` | 대화/분기 제목 수정 | §2.9 |
| DELETE | `/chats/{chatId}` | 대화 삭제 (cascade soft delete) | §2.10 |
| POST | `/chats/{chatId}/messages/{messageId}/regenerate` | 응답 재생성 (덮어쓰기) | §4.1 |
| PATCH | `/chats/{chatId}/messages/{messageId}` | 메시지 수정 (자동 분기) | §4.2 |
| GET | `/chats/{chatId}/graph` | 그래프 조회 | §3.1 |
| GET | `/chats/{chatId}/graph/expand` | 그래프 윈도우 확장 | §3.2 |

---

## 3. 핵심 구현 상세

### 3.1 분기(Branch) 트리 구조

`chat` 테이블의 `parent_id`, `root_chat_id`, `branch_point_turn_id` 3개 컬럼으로 트리 구조를 표현한다.

```
root(chat_id=1)
├── turn1 ── turn2 ── turn3
│                      └── branch(chat_id=2, parent_id=1, branch_point_turn_id=turn2)
│                           └── turn1 ── turn2
└── ...
```

- **root_chat_id**: 어떤 chat이든 자신이 속한 트리의 루트 ID를 저장. 루트 자신은 `root_chat_id = chat_id`.
- **parent_id**: 직접 부모 chat의 ID. 루트는 `null`.
- **branch_point_turn_id**: 부모 chat에서 분기가 시작된 turn의 ID.

**ChatService.createBranch()** — 부모 chat의 특정 turn에서 새 분기 chat을 생성한다. LLM 설정(provider, model)은 부모에서 상속한다.

### 3.2 응답 재생성 / 메시지 수정

두 기능은 서로 다른 전략을 사용한다.

**응답 재생성** (`regenerateMessage`): 새 분기를 생성하지 않고, **기존 Turn의 AI 메시지를 초기화하여 새 LLM 응답으로 덮어쓴다**. Turn 수도, Chat 수도 변하지 않는다. 마지막 턴뿐 아니라 **모든 ASSISTANT 메시지**에 대해 재생성이 허용되며, 어떤 메시지를 재생성할 수 있는지에 대한 UX 제한은 프론트엔드에서 처리한다.

```
regenerateMessage(messageId)
         │
         ▼
   대상 메시지 검증 (ASSISTANT만)
   STREAMING 중이면 기존 스트리밍 cancel 후 진행
         │
         ▼
   같은 Turn의 원본 USER 메시지 내용 추출
         │
         ▼
   기존 AI 메시지 초기화 (content=null, status=STREAMING)
         │
         ▼
   TurnContext(같은 chat, 같은 turn, 기존 user msg, 기존 ai msg) → streamMessage()
```

**메시지 수정** (`editMessage`): 기존 대화의 특정 지점에서 **새로운 분기를 만들어** SSE 스트리밍을 시작한다.

```
editMessage(messageId, newContent)
         │
         ▼
   대상 메시지 검증 (USER만 허용)
         │
         ▼
   resolveBranchPoint()
         │
         ▼
   createBranchTurn()
   ├── 새 branch chat 생성
   ├── 새 turn + user/ai message 생성
   └── TurnContext 반환 → streamMessage()
```

**분기점 결정 규칙 (§4.1.1 resolveBranchPoint)** — 메시지 수정에만 적용:
1. 대상 turn의 sequence > 1 → 같은 chat의 직전 turn
2. 대상 turn이 첫 turn이고 현재 chat이 branch → 부모의 branchPointTurn
3. root의 첫 turn → 거부 (더 이상 올라갈 곳 없음)

### 3.3 그래프 시각화 API

`GraphService`가 대화 트리를 윈도우 기반으로 탐색하여 시각화 데이터를 제공한다.

**getGraph()** — center turn을 기준으로 UP/DOWN 방향으로 윈도우를 열어 turn 노드를 수집한다.

- **UP 탐색 (ancestor, 단일 경로)**: center turn에서 turn_sequence를 역순으로 따라가며, chat 경계에서는 `branch_point_turn_id`를 통해 부모 chat으로 점프한다.
- **DOWN 탐색 (BFS, 다중 경로)**: center turn에서 순방향으로 진행하며, 해당 turn에서 파생된 branch들도 큐에 넣어 너비 우선 탐색한다.
- **Frontier**: 윈도우 경계에서 추가 데이터 존재 여부(`hasMore`)를 반환한다.

**expandWindow()** — 프론트에서 frontier 지점을 클릭하면 호출되어 한 방향으로만 추가 turn을 가져온다.

### 3.4 SSE 스트리밍 흐름

`MessageService.streamMessage()`이 `SseEmitter`를 반환하고, 내부에서 Virtual Thread를 사용하여 비동기로 LLM 호출 및 이벤트 전송을 수행한다.

```
일반 메시지 / 재생성:
  turn_started → chunk × N → turn_completed → done

메시지 수정 (분기 생성):
  branch_created → turn_started → chunk × N → turn_completed → done
```

```
Controller → createTurnAndMessages() 또는 regenerateMessage() 또는 editMessage()
                                  │
                            streamMessage()
                                  │
                           Virtual Thread 시작
                                  │
                ┌─────────────────┤
                │ (수정 시만)      │
                │ branch_created  │
                ├─────────────────┤
                │ turn_started    │
                ├─────────────────┤
                │ chunk × N       │  ← LlmClient.streamCompletion()
                ├─────────────────┤
                │ turn_completed  │  ← DB 저장 (TransactionTemplate)
                ├─────────────────┤
                │ done            │
                └─────────────────┘
                          │
                 generateSummaryAsync()  ← 별도 Virtual Thread
```

**`turn_started` 이벤트**: `{chatId, turnId, userMessageId, aiMessageId}` — 프론트가 어떤 chat의 메시지인지 즉시 식별할 수 있도록 `chatId`를 포함한다.

**취소 흐름**: `cancelMessage()`가 `StreamingContext.canceled` 플래그를 set → 스트리밍 루프에서 `CancelException` throw → 부분 content 저장 후 `cancelled` 이벤트 전송.

### 3.5 title nullable 정책

Phase 3에서 `chat.title`을 **nullable**로 변경했다.

- **이전**: `NOT NULL DEFAULT '제목없음'` — UI 텍스트가 DB에 저장됨
- **이후**: `NULL` — title이 없으면 DB에 `null` 저장, UI 표시는 프론트에서 처리

관련 변경:
- `Chat.java`: `@Column(nullable=false)` + `@Builder.Default "제목없음"` 제거 → nullable
- `ChatService.createChat()`: `"제목없음"` 기본값 제거 → `dto.title()` 그대로 전달
- `ChatService.createBranch()`: `"새 분기"` 기본값 제거 → title 미지정 시 `null`
- `MessageService.createBranchTurn()`: `.title(null)` 사용
- `DataInitializer.java`: 테스트 데이터도 `.title(null)` 적용
- ERD: `chat.title TEXT NULL`, `turn.summary TEXT NULL`

### 3.6 N+1 쿼리 최적화

채팅 1건 전송 시 30+개 SQL이 발생하는 N+1 문제를 **fetch join**으로 해결하여 **12개**(모두 필수 CRUD)로 줄였다.

| 레이어 | 변경 | 효과 |
|--------|------|------|
| `ChatRepository` | `findByIdWithMemberAndDeletedAtIsNull` (JOIN FETCH member) 추가 | chat 조회 시 member LAZY 로딩 제거 |
| `MessageRepository` | `findByIdWithTurnAndChat` (JOIN FETCH turn→chat) 추가 | message→turn→chat 체인 LAZY 로딩 제거 |
| `TurnRepository` | `WithMessages` fetch join 변형 3개 추가 | cursor 페이징 시 turn→messages LAZY 로딩 제거 |
| 모든 Service | 소유권 검증이 필요한 조회를 fetch join 버전으로 교체 | 불필요한 단건 SELECT 제거 |

### 3.7 Message 응답에 chatId 포함

턴 목록 조회(`/chats/{chatId}/turns`) 응답의 `MessageItem`에 **`chatId` 필드**를 추가하여, 각 메시지가 실제로 속한 chat ID를 반환하도록 했다.

**배경**: 분기(branch) 뷰에서는 부모 chat의 메시지도 함께 표시되지만, 해당 메시지들의 실제 소유 chat은 부모 chat이다. 프론트엔드가 재생성/수정 요청 시 branch의 chatId를 사용하면 `MESSAGE_4041`(메시지 없음) 에러가 발생한다.

**변경 파일**:
- `TurnResDto.MessageItem`: `chatId` 필드 추가
- `TurnConverter.toMessageItem()`: `message.getTurn().getChat().getId()`로 매핑

```java
// TurnResDto.java — MessageItem에 chatId 추가
@Builder
public record MessageItem(
        Long chatId,        // 메시지가 실제로 속한 chat ID
        Long messageId,
        SenderType senderType,
        MessageStatus status,
        String content,
        Integer promptToken,
        Integer answerToken,
        LocalDateTime createdAt
) {}
```

프론트엔드는 이 `chatId`를 사용하여 재생성/수정 API의 path parameter를 정확하게 설정해야 한다.

### 3.8 Spring Security 비동기 디스패치 수정

`SseEmitter`의 비동기 디스패치 시 Spring Security가 `SecurityContext`를 찾지 못해 `AuthorizationDeniedException`이 발생하는 문제를 수정했다.

```java
// SecurityConfig.java — ASYNC/ERROR 디스패치에 대해 인가 생략
.dispatcherTypeMatchers(DispatcherType.ASYNC, DispatcherType.ERROR).permitAll()
```

---

## 4. 코드 구현 방식

### 4.1 레이어드 아키텍처 (Controller → Service → Repository)

모든 API가 동일한 3계층 흐름을 따른다.

```
Client 요청
  → Controller: 파라미터 바인딩, 입력 검증(@Valid), 인증 객체 추출
    → Service: 비즈니스 로직, 소유권 검증, 트랜잭션 관리
      → Repository: JPA 쿼리 실행
    → Converter: Entity → DTO 변환
  → ApiResponse 래핑 후 응답
```

**Controller**는 비즈니스 로직을 포함하지 않는다. `@AuthenticationPrincipal`로 인증 정보를 받아 Service에 `memberId`만 전달한다.

```java
// MessageController.java — Controller는 조합만 담당
MessageService.TurnContext ctx = messageService.createTurnAndMessages(memberId, chatId, dto.content());
return messageService.streamMessage(ctx);
```

**Service**에서 모든 비즈니스 규칙을 처리한다. 소유권 검증, 상태 전이, 분기점 결정 등 도메인 로직이 여기에 집중된다.

```java
// ChatService.java — 소유권 검증 패턴 (모든 Service에서 동일하게 사용)
private void validateOwner(Chat chat, Long memberId) {
    if (!chat.getMember().getId().equals(memberId)) {
        throw new ProjectException(ErrorStatus.FORBIDDEN);
    }
}
```

### 4.2 공통 응답 구조 (ApiResponse)

모든 API 응답은 `ApiResponse<T>` 래퍼로 통일된다.

```java
// 성공: {"isSuccess": true, "code": "COMMON_200", "message": "...", "result": {...}}
return ApiResponse.onSuccess(SuccessStatus.OK, result);

// 실패: {"isSuccess": false, "code": "CHAT_4041", "message": "존재하지 않는 대화입니다.", "result": null}
return ApiResponse.onFailure(ErrorStatus.CHAT_NOT_FOUND, null);
```

에러는 `ProjectException` → `GlobalExceptionHandler`에서 잡아 `ApiResponse.onFailure()`로 변환한다. 에러 코드는 `ErrorStatus` enum에 도메인별로 그룹화되어 있다 (AUTH_40xx, CHAT_40xx, MESSAGE_40xx 등).

### 4.3 엔티티 설계 패턴

**Builder 패턴**: 모든 엔티티가 Lombok `@Builder`를 사용한다. 생성 시점에 필요한 값을 명시적으로 전달하고, 이후 변경은 의미 있는 이름의 메서드(`updateTitle`, `softDelete`, `touchUpdatedAt`)로만 허용한다.

```java
// Chat.java — 상태 변경 메서드
public void updateTitle(String title) {
    this.title = title;
    this.titleStatus = TitleStatus.USER_EDITED;  // title 변경 시 상태도 함께 전이
}

public void softDelete() {
    this.deletedAt = LocalDateTime.now();  // 물리 삭제가 아닌 soft delete
}
```

**BaseEntity 상속**: `createdAt`, `updatedAt`을 공통으로 관리한다. `@PrePersist`, `@PreUpdate`로 자동 세팅.

**Soft Delete**: `deleted_at` 컬럼이 null이면 활성, 값이 있으면 삭제 상태. 조회 시 항상 `deletedAtIsNull` 조건을 붙인다.

```java
// ChatRepository.java — soft delete 적용된 조회
Optional<Chat> findByIdAndDeletedAtIsNull(Long chatId);
Page<Chat> findByMemberIdAndParentIdIsNullAndDeletedAtIsNull(Long memberId, Pageable pageable);
```

### 4.4 트리 구조의 DB 표현 (Adjacency List)

chat 간의 부모-자식 관계를 `parent_id` FK로 표현하는 **인접 리스트(Adjacency List)** 방식이다.

```java
// Chat.java — 트리 관련 필드
private Long parentId;           // 부모 chat ID (null = root)
private Long rootChatId;         // 트리의 최상위 root ID
private Long branchPointTurnId;  // 부모 chat에서 분기한 turn ID
```

**root_chat_id**를 별도로 저장하는 이유: 같은 트리에 속하는 모든 chat을 한 번의 쿼리로 조회하기 위해서다. `findAllByRootChatId()`로 트리 전체를 가져온 후, 메모리에서 Map으로 구성하여 탐색한다.

```java
// GraphService.java — 트리를 메모리에 로드하여 탐색
List<Chat> allChats = chatRepository.findAllByRootChatIdAndDeletedAtIsNull(rootChatId);
Map<Long, Chat> chatMap = allChats.stream()
        .collect(Collectors.toMap(Chat::getId, c -> c));
Map<Long, List<Chat>> branchPointMap = allChats.stream()
        .filter(c -> c.getBranchPointTurnId() != null)
        .collect(Collectors.groupingBy(Chat::getBranchPointTurnId));
```

**Cascade soft delete**는 재귀로 구현한다.

```java
// ChatService.java — 자손 chat을 재귀적으로 soft delete
private void softDeleteCascade(Chat chat) {
    chat.softDelete();
    List<Chat> children = chatRepository.findAllByParentId(chat.getId());
    for (Chat child : children) {
        if (child.getDeletedAt() == null) {
            softDeleteCascade(child);
        }
    }
}
```

### 4.5 Cursor 기반 페이징

턴 목록 조회에서 Offset이 아닌 **Cursor 방식**을 사용한다. `turn_sequence`를 커서로 사용하여 "이 sequence 이전/이후 N개"를 조회한다.

```java
// TurnService.java — limit + 1 조회로 hasMore 판단
PageRequest pageRequest = PageRequest.of(0, limit + 1);
List<Turn> turns = turnRepository
    .findByChatIdAndTurnSequenceLessThanOrderByTurnSequenceDesc(chatId, lastTurnSequence, pageRequest);

boolean hasMore = turns.size() > limit;
if (hasMore) {
    turns = turns.subList(0, limit);
}
```

Offset 페이징 대비 장점: 데이터 추가/삭제 시 페이지가 밀리지 않고, 인덱스를 활용한 범위 조회라 대량 데이터에서도 성능이 일정하다.

### 4.6 SSE 비동기 처리 (Virtual Thread + TransactionTemplate)

Spring MVC의 `SseEmitter`를 반환하면 HTTP 응답이 열린 채로 유지된다. LLM 호출은 **Virtual Thread**에서 비동기로 수행하며, Virtual Thread 내에서는 `@Transactional`이 동작하지 않으므로 **`TransactionTemplate`**으로 수동 트랜잭션을 관리한다.

```java
// MessageService.java — Virtual Thread 내 수동 트랜잭션
Thread.startVirtualThread(() -> {
    // ... SSE 스트리밍 중 ...

    // 완료 시 DB 저장 (TransactionTemplate으로 별도 트랜잭션)
    transactionTemplate.executeWithoutResult(status -> {
        Message message = messageRepository.findById(aiMessageId).orElseThrow();
        message.updateContent(fullContent);
        message.updateStatus(MessageStatus.COMPLETED);
    });
});
```

**취소 동시성 처리**: `ConcurrentHashMap<Long, StreamingContext>`에 스트리밍 중인 메시지를 등록하고, `AtomicBoolean` 플래그로 cancel 시그널을 전달한다.

```java
// StreamingContext — cancel 경합 해결
record StreamingContext(AtomicBoolean canceled, StringBuffer contentBuffer) {}

// cancel 요청 시
streamCtx.canceled().set(true);

// 스트리밍 루프에서 매 chunk마다 확인
if (streamCtx.canceled().get()) {
    throw new CancelException();
}
```

### 4.7 LLM 클라이언트 인터페이스 분리

```java
// LlmClient.java — 인터페이스
public interface LlmClient {
    void streamCompletion(String model, String userMessage, Consumer<String> onChunk);
}

// MockLlmClient.java — 개발용 Mock 구현체 (@Component)
@Component
public class MockLlmClient implements LlmClient {
    @Override
    public void streamCompletion(String model, String userMessage, Consumer<String> onChunk) {
        for (String chunk : CHUNKS) {
            Thread.sleep(200);
            onChunk.accept(chunk);
        }
    }
}
```

인터페이스로 분리하여 실제 LLM API 연동 시 `MockLlmClient` 대신 새 구현체를 `@Component`로 등록하면 된다. Service 코드 변경 없이 구현체만 교체 가능하다 (DIP).

### 4.8 Converter 분리

Entity → DTO 변환 로직을 별도 `Converter` 클래스로 분리한다. Service에서 비즈니스 로직과 응답 변환을 섞지 않는다.

```java
// ChatConverter.java — Entity를 DTO로 변환
public static ChatResDto.Detail toDetail(Chat chat) {
    return ChatResDto.Detail.builder()
            .chatId(chat.getId())
            .title(chat.getTitle())
            .titleStatus(chat.getTitleStatus())
            // ...
            .build();
}
```

대화 목록 조회 시, N+1 문제를 방지하기 위해 턴 수와 최신 메시지를 **batch 쿼리**로 미리 조회한 후 Converter에 Map으로 전달한다.

```java
// ChatService.getChatList() — batch 쿼리로 N+1 방지
Map<Long, Long> turnCounts = turnRepository.countByChatIds(chatIds).stream()
        .collect(Collectors.toMap(r -> (Long) r[0], r -> (Long) r[1]));
Map<Long, Message> latestMessages = messageRepository.findLatestByChatIds(chatIds).stream()
        .collect(Collectors.toMap(m -> m.getTurn().getChat().getId(), m -> m));
return ChatConverter.toPageInfo(page, turnCounts, latestMessages);
```

### 4.9 DTO 설계 (record + inner class)

요청/응답 DTO는 Java `record`로 정의하고, 하나의 도메인에 대한 DTO를 outer class 안에 inner record로 그룹화한다.

```java
// ChatReqDto.java — 요청 DTO 그룹
public class ChatReqDto {
    public record Create(String title, LlmProvider llmProvider, LlmModel llmModel) {}
    public record BranchCreate(Long branchPointTurnId, String title) {}
    public record UpdateTitle(String title) {}
}

// ChatResDto.java — 응답 DTO 그룹
public class ChatResDto {
    public record Detail(Long chatId, Long rootChatId, ...) {}
    public record ListItem(Long chatId, String title, ...) {}
    public record PageInfo(List<ListItem> content, int page, ...) {}
}
```

### 4.10 JPA 인덱스 선언

엔티티 클래스의 `@Table(indexes = {...})`에 인덱스를 선언하여, `ddl-auto: update` 시 자동으로 인덱스가 생성된다.

```java
// Chat.java
@Table(name = "chat", indexes = {
    @Index(name = "idx_chat_root_deleted", columnList = "root_chat_id, deleted_at"),
    @Index(name = "idx_chat_parent", columnList = "parent_id"),
    @Index(name = "idx_chat_branch_point", columnList = "branch_point_turn_id")
})
```

각 인덱스의 목적:
- `idx_chat_root_deleted`: 같은 트리의 활성 chat 전체 조회 (GraphService)
- `idx_chat_parent`: 자손 chat 조회 (cascade 삭제)
- `idx_chat_branch_point`: 특정 turn에서 분기된 chat 검색
- `idx_turn_chat_sequence`: chat 내 turn 순서 조회 (cursor 페이징, 그래프 탐색)
- `idx_message_turn`: turn에 속한 메시지 조회

---

## 5. 패키지 구조

```
com.aistar.backend
├── domain
│   ├── auth
│   │   ├── controller/AuthController.java
│   │   ├── service/AuthService.java
│   │   ├── dto/AuthReqDto.java, AuthResDto.java
│   │   └── converter/AuthConverter.java
│   ├── chat
│   │   ├── controller/
│   │   │   ├── ChatController.java      ─ 대화 CRUD + 분기 생성
│   │   │   ├── MessageController.java   ─ 메시지 송신/취소/재생성/수정
│   │   │   └── GraphController.java     ─ 그래프 조회/확장
│   │   ├── service/
│   │   │   ├── ChatService.java         ─ 대화 생성/목록/삭제, 분기 생성
│   │   │   ├── MessageService.java      ─ SSE 스트리밍, 취소, 재생성, 수정, 분기점 결정
│   │   │   ├── TurnService.java         ─ 턴 cursor 페이징 조회
│   │   │   └── GraphService.java        ─ 그래프 UP/DOWN 탐색, frontier 계산
│   │   ├── entity/
│   │   │   ├── Chat.java                ─ 트리 구조 (parentId, rootChatId, branchPointTurnId)
│   │   │   ├── Turn.java                ─ 대화 턴 (chat 소속, sequence 순서)
│   │   │   └── Message.java             ─ 메시지 (turn 소속, USER/ASSISTANT)
│   │   ├── enums/
│   │   │   ├── TitleStatus.java         ─ PENDING, GENERATED, USER_EDITED
│   │   │   ├── MessageStatus.java       ─ STREAMING, COMPLETED, CANCELED, FAILED
│   │   │   ├── SenderType.java          ─ USER, ASSISTANT
│   │   │   ├── LlmProvider.java         ─ OPENAI, GOOGLE, ANTHROPIC
│   │   │   ├── LlmModel.java           ─ gpt-4o-mini, gemini-2.0-flash, claude-3.5-sonnet
│   │   │   └── CursorDirection.java     ─ BACKWARD, FORWARD
│   │   ├── dto/
│   │   │   ├── ChatReqDto.java          ─ Create, BranchCreate, UpdateTitle
│   │   │   ├── ChatResDto.java          ─ Detail, ListItem, PageInfo
│   │   │   ├── MessageReqDto.java       ─ Send
│   │   │   ├── MessageResDto.java       ─ TurnStarted, Chunk, TurnCompleted, Cancelled, ...
│   │   │   ├── TurnResDto.java          ─ TurnPage, TurnDetail, MessageDetail
│   │   │   └── GraphResDto.java         ─ GraphResult, ExpandResult, TurnNodeDto, ChatNodeDto
│   │   ├── converter/
│   │   │   ├── ChatConverter.java
│   │   │   └── TurnConverter.java
│   │   └── repository/
│   │       ├── ChatRepository.java
│   │       ├── TurnRepository.java
│   │       └── MessageRepository.java
│   ├── llm
│   │   └── client/
│   │       ├── LlmClient.java           ─ 인터페이스 (streamCompletion)
│   │       └── MockLlmClient.java       ─ 테스트용 Mock 구현체
│   └── member
│       ├── controller/MemberController.java
│       ├── service/MemberService.java
│       ├── entity/Member.java
│       └── repository/MemberRepository.java
├── global
│   ├── apiPayload/                      ─ 공통 응답(ApiResponse), 에러 코드, 예외 처리
│   ├── config/
│   │   ├── AppConfig.java
│   │   ├── SwaggerConfig.java
│   │   └── DataInitializer.java         ─ 개발용 초기 데이터
│   ├── entity/BaseEntity.java           ─ createdAt, updatedAt 공통
│   └── security/
│       ├── SecurityConfig.java          ─ CORS, JWT 필터, 인가 설정
│       ├── JwtTokenProvider.java
│       ├── JwtAuthenticationFilter.java
│       ├── JwtAuthenticationEntryPoint.java
│       ├── CustomUserDetails.java
│       └── CustomUserDetailsService.java
└── BackendApplication.java
```

---

## 6. Phase 3 커밋 이력

| 커밋 | 설명 |
|------|------|
| `2bd1404` | 분기 생성, 채팅 수정, 채팅 삭제 API 추가 |
| `152fc18` | 그래프 조회 API — GraphService UP/DOWN 탐색, frontier 계산 |
| `09f6a37` | 그래프 확장 API — expandWindow, 방향별 frontier |
| `da25c3d` | 응답 재생성 API — regenerateMessage, resolveBranchPoint |
| `2d6cd01` | 메시지 수정 API — editMessage, createBranchTurn 공통화 |
| `18fa720` | title nullable 정책 변경 — DB 기본 문구 제거, ERD/명세 동기화 |
| `2cca516` | N+1 쿼리 최적화 (fetch join), Security async dispatch 수정 |
| `3691d94` | 명세서 v0.6 반영 |
| `a0f0174` | 재생성 시 브랜치 생성 제거 — 기존 AI 메시지 덮어쓰기 방식으로 변경 |
| `f8874f0` | AI 서버와 통합 — 실제 LLM API 연동 |

---

## 7. ERD 주요 변경 (Phase 2 → Phase 3)

```sql
-- chat.title: NOT NULL DEFAULT '제목없음' → NULL
ALTER TABLE chat MODIFY COLUMN title TEXT NULL;

-- title_status 컬럼 추가
-- enum('PENDING','GENERATED','USER_EDITED') NOT NULL DEFAULT 'PENDING'

-- turn.summary: TEXT NULL (DEFAULT 제거)

-- 인덱스 추가
CREATE INDEX idx_chat_root_deleted  ON chat(root_chat_id, deleted_at);
CREATE INDEX idx_chat_parent        ON chat(parent_id);
CREATE INDEX idx_chat_branch_point  ON chat(branch_point_turn_id);
CREATE INDEX idx_turn_chat_sequence ON turn(chat_id, turn_sequence);
CREATE INDEX idx_message_turn       ON message(turn_id);
```

---

## 8. 개발 중 발생한 이슈 및 해결

### 8.1 Spring Security 비동기 디스패치 Access Denied

**증상**: SSE 스트리밍 도중 `AuthorizationDeniedException: Access Denied` 발생. 채팅 응답이 완료되기 전에 연결이 끊김.

**원인**: `SseEmitter`가 응답을 비동기로 전송할 때, Spring Security가 `DispatcherType.ASYNC` 재디스패치에 대해 `SecurityContext`를 찾지 못해 인가를 거부함. 스택 트레이스의 `AsyncContextImpl$AsyncRunnable`에서 확인.

**해결**: `SecurityConfig`에 ASYNC/ERROR 디스패치에 대한 permitAll 추가.
```java
.dispatcherTypeMatchers(DispatcherType.ASYNC, DispatcherType.ERROR).permitAll()
```

### 8.2 N+1 쿼리 문제 (30+ → 12 쿼리)

**증상**: 채팅 메시지 1건 전송에 Hibernate SQL이 30개 이상 발생. `show-sql: true` 로그로 확인.

**원인**: JPA LAZY 로딩 체인이 여러 곳에서 개별 SELECT을 유발.
- `chat.getMember()` — 소유권 검증 시마다 member 단건 조회
- `message.getTurn().getChat()` — message → turn → chat 체인
- `turn.getMessages()` — 턴 목록 조회 시 각 turn의 messages 개별 로딩

**해결**: fetch join 쿼리 도입.
- `ChatRepository.findByIdWithMemberAndDeletedAtIsNull` — `JOIN FETCH c.member`
- `MessageRepository.findByIdWithTurnAndChat` — `JOIN FETCH m.turn t JOIN FETCH t.chat`
- `TurnRepository` — cursor 페이징에 `LEFT JOIN FETCH t.messages` 변형 3개

모든 Service의 조회를 fetch join 버전으로 교체하여 **30+ → 12 쿼리**(전부 필수 CRUD)로 감소.

### 8.3 SSE 엔드포인트 에러 응답 500

**증상**: 재생성/수정 API에서 비즈니스 에러(예: STREAMING 상태 메시지) 발생 시, 프론트에 `409`가 아닌 `500`이 반환됨.

**원인**: Controller의 `produces = MediaType.TEXT_EVENT_STREAM_VALUE`만 선언되어 있어, `GlobalExceptionHandler`가 JSON 에러 응답을 생성하려 해도 `HttpMediaTypeNotAcceptableException`이 발생하여 500으로 변환됨.

**해결**: SSE 엔드포인트의 `produces`에 `APPLICATION_JSON_VALUE`를 추가하여 에러 시 JSON 응답이 가능하도록 수정.
```java
@PostMapping(value = "/{messageId}/regenerate",
    produces = {MediaType.TEXT_EVENT_STREAM_VALUE, MediaType.APPLICATION_JSON_VALUE})
```

### 8.4 프론트엔드 Zod 스키마 불일치 (title nullable)

**증상**: 백엔드에서 채팅 생성 API가 정상 응답(200)을 반환하지만, 프론트에서 응답을 무시하고 아무 동작도 하지 않음. 에러 메시지도 없음.

**원인**: Phase 3에서 `chat.title`을 nullable로 변경했으나, 프론트의 Zod 스키마가 `title: z.string()`으로 되어 있어 `null` 값이 validation에 실패. catch 블록에서 에러가 조용히 무시됨.

**해결**: 프론트 Zod 스키마에서 `title: z.string()` → `title: z.string().nullable()`로 수정.

### 8.5 turn_started에 chatId 누락

**증상**: 프론트에서 재생성 버튼을 눌러도 SSE 스트리밍은 정상 수신되지만, 화면에 표시되지 않음. 3번 연속 누르면 동작.

**원인**: 프론트의 `useRegenerate` 훅이 `turn_started` 이벤트의 `chatId` 필드로 대상 chat을 식별하는데, 백엔드 `TurnStarted` DTO에 `chatId`가 없어서 `null` → 이후 모든 이벤트(chunk, done)가 무시됨. 3번째 시도에서 타이밍상 동작하는 것은 이전 시도의 상태가 누적된 부작용.

**해결**: `MessageResDto.TurnStarted`에 `chatId` 필드를 추가하고, `streamMessage()`에서 `ctx.chat().getId()`를 전달.

### 8.6 재생성 시 STREAMING 상태 충돌

**증상**: 재생성 중인 메시지에 다시 재생성을 요청하면 `MESSAGE_4092` 에러(→ §8.3에 의해 500) 발생.

**원인**: `regenerateMessage()`에서 `message.getStatus() == MessageStatus.STREAMING`이면 무조건 거부하도록 되어 있었음. 재생성은 기존 AI 메시지를 STREAMING으로 변경하므로, 스트리밍 완료 전에 다시 재생성하면 항상 거부됨.

**해결**: STREAMING 상태에서는 기존 스트리밍을 cancel(`StreamingContext.canceled.set(true)`)한 후 새로 시작하도록 변경. 에러 대신 이전 응답을 중단하고 새 응답을 시작.

### 8.7 분기 뷰에서 재생성 시 chatId 불일치

**증상**: 분기(branch) chat 화면에서 부모 chat의 메시지를 재생성하면 `MESSAGE_4041` (프론트에서 500으로 표시) 에러 발생.

**원인**: 분기 chat(예: chatId=77)은 부모 chat(chatId=65)의 branchPointTurn까지의 메시지를 함께 표시한다. 프론트엔드가 모든 메시지에 대해 현재 활성 branch의 chatId(77)로 재생성 요청을 보내지만, 실제 메시지(예: messageId=70)는 부모 chat(65)에 속해 있다. 백엔드에서 `message.getTurn().getChat().getId() != chatId` 검증에 걸려 404가 반환된다.

**해결**: `TurnResDto.MessageItem`에 `chatId` 필드를 추가하여 각 메시지의 실제 소유 chat ID를 반환 (§3.7). 프론트엔드는 branch의 chatId 대신 메시지 응답의 `chatId`를 사용하여 API를 호출해야 한다.

---

## 9. 미구현/보류 사항

| 항목 | 사유 |
|------|------|
| `UsageRecord` 엔티티 | ERD에 정의되어 있으나 Phase 3까지 필요한 기능 없음 |
