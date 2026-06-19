# Phase 4 구현 요약

## 1. 개요

Phase 3에서 구현한 분기 트리 구조와 그래프 시각화 위에, **대화 탐색기(FG-4)**, **맥락 조립/압축(FG-5)**, **토큰 사용량 추적(FR-5.3)** 을 추가하여 대화 이력을 파일 탐색기 형태로 관리하고, LLM 호출 시 분기 경로에 맞는 맥락을 자동으로 조립할 수 있도록 확장하였다.

**명세서**: `api_phase4_v7.md` (v0.8)
**ERD**: `AIT_ERD_phase4 v2.txt`

**진행 상태**: M1~M7 전체 완료 (2026-05-26)

---

## 2. 구현 API 목록

### 2.1 Phase 3까지의 기존 API

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
| POST | `/chats/{chatId}/branches` | 분기 생성 |
| PATCH | `/chats/{chatId}` | 대화/분기 제목 수정 |
| DELETE | `/chats/{chatId}` | 대화 삭제 (cascade soft delete) |
| POST | `/chats/{chatId}/messages/{messageId}/regenerate` | 응답 재생성 |
| PATCH | `/chats/{chatId}/messages/{messageId}` | 메시지 수정 (자동 분기) |
| GET | `/chats/{chatId}/graph` | 그래프 조회 |
| GET | `/chats/{chatId}/graph/expand` | 그래프 윈도우 확장 |

### 2.2 Phase 4 신규 API

| Method | Endpoint | 설명 | 명세서 | 상태 |
|--------|----------|------|--------|------|
| GET | `/chats/explorer` | 탐색기 트리 조회 | §2.1 | 완료 |
| GET | `/chats/explorer/{rootChatId}` | 단일 트리 새로고침 | §2.2 | 완료 |
| GET | `/usage/me` | 토큰 사용량 조회 | §3.3 | 완료 |

### 2.3 Phase 4에서 변경되는 기존 API 동작

| Endpoint | 변경 내용 | 상태 |
|----------|----------|------|
| `POST /chats/{chatId}/messages` | LLM 호출 시 ancestor chain 맥락 조립 + 압축, `turn_completed`에 압축 필드 추가, 사용량 누적 | 완료 |
| `POST /chats/{chatId}/messages/{messageId}/regenerate` | 동일 맥락 조립 + 압축 + 사용량 누적 적용 | 완료 |
| `PATCH /chats/{chatId}/messages/{messageId}` | 동일 맥락 조립 + 압축 + 사용량 누적 적용 | 완료 |
| `POST /chats/{chatId}/branches` | branch 생성 시 부모 ancestor chain `lastActivityAt` 갱신 | 완료 |
| `PATCH /chats/{chatId}` | 제목 수정 시 ancestor chain `lastActivityAt` 갱신 | 완료 |

---

## 3. 핵심 구현 상세

### 3.1 lastActivityAt — 탐색기 정렬 캐시

탐색기의 `recent` 정렬을 위해 `chat.last_activity_at` 컬럼을 추가했다. 이 컬럼은 해당 chat과 하위 분기에서 마지막으로 활동이 발생한 시각을 캐시한다.

```
root(lastActivityAt=14:30)
├── branch A (lastActivityAt=14:30)  ← 여기서 메시지 전송됨
└── branch B (lastActivityAt=10:15)
```

branch A에서 메시지를 전송하면 branch A뿐 아니라 **부모인 root의 lastActivityAt도 함께 갱신**된다. 이렇게 해야 탐색기에서 root 목록을 `lastActivityAt DESC`로 정렬할 때, 하위 분기에서 활동이 있었던 트리가 위로 올라온다.

**갱신 정책:**

```
갱신 O: 메시지 송신, 응답 재생성, 메시지 수정, branch 생성, chat 제목 수정
갱신 X: branch 삭제 ("더 이상 보지 않겠다"는 의도이므로 정렬 위로 올리지 않음)
```

**ancestor chain 전파:** `ChatService.touchAncestorChain(chatId)`가 `parentId`를 따라 root까지 순회하며 각 chat의 `lastActivityAt`을 갱신한다.

```java
// ChatService.java — ancestor chain 전체 갱신
public void touchAncestorChain(Long chatId) {
    Long currentId = chatId;
    while (currentId != null) {
        Chat chat = chatRepository.findById(currentId).orElse(null);
        if (chat == null) break;
        chat.touchLastActivityAt();
        currentId = chat.getParentId();
    }
}
```

**연결 지점:**
- `ChatService.createBranch()` — branch 저장 후 `touchAncestorChain(parentChatId)`
- `ChatService.updateChatTitle()` — 제목 변경 후 `touchAncestorChain(chatId)`
- `MessageService.streamMessage()` — 완료 트랜잭션에서 `chatService.touchAncestorChain(chatId)`
  - 송신/재생성/수정 모두 `streamMessage()`를 통과하므로 한 곳에서 처리

**DDL 마이그레이션:** 기존 row가 있는 테이블에 NOT NULL 컬럼을 추가하면 `ALTER TABLE` 실패. `columnDefinition = "timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP"`를 사용하여 MySQL이 기존 row에 자동으로 CURRENT_TIMESTAMP를 채우도록 했다.

```java
// Chat.java — DDL 안전한 NOT NULL 컬럼 추가
@Column(name = "last_activity_at", nullable = false,
        columnDefinition = "timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP")
@Builder.Default
private LocalDateTime lastActivityAt = LocalDateTime.now();
```

### 3.2 대화 탐색기 API (FG-4)

사용자가 소유한 root chat 목록을 페이지 단위로 반환하며, 각 root chat에 대해 하위 분기 트리를 **평탄화된 배열**로 함께 포함한다.

```http
GET /api/v1/chats/explorer?sort=recent&page=0&size=20&includeDeleted=false
```

**응답 구조:**

```
ExplorerPage
├── roots: ExplorerTreeDto[]
│   ├── rootChatId: Long
│   └── nodes: ExplorerNodeDto[]      ← DFS 순서로 정렬된 평탄 배열
│       ├── chatId, parentChatId, depth
│       ├── title, titleStatus
│       ├── turnCount, lastActivityAt
│       └── llmProvider, llmModel
├── page, size, totalRootCount, hasNext
```

**정렬 옵션:**
- `recent`: `lastActivityAt DESC, chatId DESC` — 최근 활동순
- `created`: `createdAt DESC, chatId DESC` — 생성일순
- `name`: `title IS NULL`로 null 마지막 배치 후 `LOWER(title) ASC` — 이름순

```java
// ChatRepository.java — name 정렬 (MySQL은 NULLS LAST 미지원, CASE WHEN으로 대체)
@Query("SELECT c FROM Chat c WHERE c.member.id = :memberId AND c.parentId IS NULL " +
        "AND (:includeDeleted = true OR c.deletedAt IS NULL) " +
        "ORDER BY CASE WHEN c.title IS NULL THEN 1 ELSE 0 END ASC, " +
        "LOWER(c.title) ASC, c.id DESC")
Page<Chat> findRootsByName(...);
```

**쿼리 전략 — N+1 방지:**

root별로 개별 쿼리를 반복하지 않는다. 3단계로 일괄 처리한다.

```
1. root 목록 페이징 (1 쿼리)
2. root_chat_id IN (:pagedRootIds)로 하위 chat 전체 배치 조회 (1 쿼리)
3. 조회된 chatId 목록으로 turnCount GROUP BY 일괄 계산 (1 쿼리)
```

```java
// ExplorerService.java — 배치 조회 흐름
List<Long> rootIds = rootPage.getContent().stream().map(Chat::getId).toList();
List<Chat> allChats = chatRepository.findAllByRootChatIdIn(rootIds, includeDeleted);

List<Long> allChatIds = allChats.stream().map(Chat::getId).toList();
Map<Long, Long> turnCounts = turnRepository.countByChatIds(allChatIds).stream()
        .collect(Collectors.toMap(r -> (Long) r[0], r -> (Long) r[1]));
```

`turnCount`는 Phase 3에서 만든 `TurnRepository.countByChatIds()`를 재사용한다.

**DFS 정렬:** 배치 조회한 chat 리스트를 메모리에서 DFS로 재정렬한다. 같은 부모를 가진 형제 chat은 `createdAt ASC` 순서.

```java
// ExplorerConverter.java — DFS로 트리 평탄화
private static void dfs(Chat chat, int depth,
                        Map<Long, List<Chat>> childrenMap,
                        Map<Long, Long> turnCounts,
                        List<ExplorerResDto.ExplorerNodeDto> nodes) {
    nodes.add(toNode(chat, depth, turnCounts));
    childrenMap.getOrDefault(chat.getId(), List.of()).stream()
            .sorted(Comparator.comparing(Chat::getCreatedAt))
            .forEach(child -> dfs(child, depth + 1, childrenMap, turnCounts, nodes));
}
```

**단일 트리 새로고침 (§2.2):**

```http
GET /api/v1/chats/explorer/{rootChatId}?includeDeleted=false
```

분기 생성/삭제 후 해당 root의 트리만 다시 가져오는 보조 엔드포인트. `ExplorerTreeDto` 한 개를 반환한다. root chat이 아닌 ID가 들어오면 `EXPLORER_4001` 에러.

### 3.3 Plan / UsageRecord 엔티티

Phase 4 토큰 사용량 추적(FR-5.3)을 위한 기반 엔티티를 추가했다.

```java
// Plan.java — 요금제 엔티티
@Entity
public class Plan {
    private Long id;
    private String name;          // varchar(20), 예: "Free", "Pro"
    private BigDecimal price;
    private Integer maxProject;
    private Integer maxDepth;
    private Long maxFileSize;
    private Long aiLimit;         // 토큰 한도
    private String description;
}

// UsageRecord.java — 기간별 토큰 사용량 기록
@Entity
@Table(indexes = {
    @Index(name = "idx_usage_member_period", columnList = "member_id, period_end")
})
public class UsageRecord {
    private Long id;
    private LocalDateTime periodStart;
    private LocalDateTime periodEnd;
    private Long tokensUsed;              // default 0
    private Long tokenLimit;              // default 0 (0 = 무제한)
    private Integer requestCount;         // default 0
    private Integer compressedTurnCount;  // default 0
    @ManyToOne Member member;
    @ManyToOne Plan plan;

    // LLM 호출 완료 시 사용량 누적
    public void accumulate(long promptTokens, long answerTokens, int compressedTurns) {
        this.tokensUsed += promptTokens + answerTokens;
        this.requestCount += 1;
        this.compressedTurnCount += compressedTurns;
    }
}
```

`DataInitializer`에 Free 플랜(aiLimit=500,000)과 당월 UsageRecord 시드 데이터를 추가했다.

### 3.4 LlmModel contextWindow

압축(FR-5.2) 임계값 계산을 위해 각 모델의 context window 크기를 enum에 추가했다.

```java
// LlmModel.java
GPT_4O_MINI("gpt-4o-mini", LlmProvider.OPENAI, 128_000),
GEMINI_2_0_FLASH("gemini-2.0-flash", LlmProvider.GOOGLE, 1_048_576),
CLAUDE_3_5_SONNET("claude-3.5-sonnet", LlmProvider.ANTHROPIC, 200_000);

private final String modelId;
private final LlmProvider provider;
private final int contextWindow;
```

### 3.5 맥락 조립 (FG-5, FR-5.1 — Context Assembly)

LLM 호출 시, target chat에서 root chat까지의 ancestor chain을 따라가며 분기 경로에 해당하는 turn만 선별하여 messages 배열을 조립한다. **별도의 외부 API 없이** 메시지 송신/재생성/수정 시 내부에서 자동 적용된다.


맥락 조립

분기 트리에서 root부터 현재 chat까지의 대화만 골라서 시간순으로 이어 붙인다.

root (turn 1,2,3) → 분기A (turn 1,2) → 분기B (turn 1) ← 현재 위치

- root: 분기A가 갈라진 branchPointTurn 시점까지만 (예: turn 1,2)
- 분기A: 분기B가 갈라진 시점까지만 (예: turn 1)
- 분기B: 현재 turn 이전까지
- 각 turn의 user/assistant 메시지를 [{role, content}, ...] 배열로 만듦

→ 관련 없는 가지의 turn은 전송 X

압축

전체 토큰이 모델 context window × 80% 를 넘으면 발동합니다.

- 오래된 turn부터 하나씩 처리
- summary 있으면 → "이전 대화 요약: {summary}" 한 줄로 교체
- summary 없으면 → 메시지를 200자로 잘라서 "...[일부 생략]" 붙임
- 마지막 2턴은 절대 건드리지 않음 (최근 맥락 보존)
- 임계치 아래로 떨어지면 중단

토큰 추정은 글자수 / 4 로 단순 계산합니다 (MVP 간소화).
**조립 절차:**

```
사용자가 chat 87(parent=42)에서 메시지 전송 시:

1. ancestor chain 구축: [42(root), 87(target)]
2. chat 42: 다음 자식 87의 branchPointTurnId = turn 3
   → turn_sequence ≤ 3인 turn만 포함 (turn 1, 2, 3)
3. chat 87 (target): 현재 turn 이전까지의 모든 turn 포함
4. 새 user 메시지를 끝에 추가

결과: root.turn1, root.turn2, root.turn3, 87.turn1, 87.turn2, 새 메시지
(root의 turn4, turn5는 분기 이후 자매 흐름이므로 제외)
```

```java
// ContextAssembler.java — ancestor chain 순회
private List<Chat> buildAncestorChain(Chat targetChat) {
    List<Chat> chain = new ArrayList<>();
    chain.add(targetChat);
    Long parentId = targetChat.getParentId();
    while (parentId != null) {
        Chat parent = chatRepository.findById(parentId).orElse(null);
        if (parent == null) break;
        chain.add(parent);
        parentId = parent.getParentId();
    }
    Collections.reverse(chain);  // root → target 순으로 정렬
    return chain;
}

// 각 ancestor에서 branchPointTurnId 기준으로 turn 범위 결정
for (int i = 0; i < chain.size(); i++) {
    Chat chat = chain.get(i);
    if (i < chain.size() - 1) {
        Chat nextChild = chain.get(i + 1);
        Turn bpTurn = turnRepository.findById(nextChild.getBranchPointTurnId()).orElseThrow();
        turns = turnRepository.findByChatIdAndTurnSequenceLteWithMessages(
                chat.getId(), bpTurn.getTurnSequence());
    } else {
        // target: 현재 turn 이전까지
        turns = turnRepository.findByChatIdAndTurnSequenceLteWithMessages(
                chat.getId(), currentTurn.getTurnSequence() - 1);
    }
}
```

**MessageService 연결:** `streamMessage()` 내에서 SSE 스트리밍 시작 전에 `contextAssembler.buildContext()`를 호출하여 조립된 messages 배열을 AI 서버에 전달한다.

```java
// MessageService.java — 맥락 조립 결과를 AI 서버에 전달
ctxHolder[0] = contextAssembler.buildContext(
        ctx.chat(), ctx.turn(), ctx.userMessage().getContent(),
        ctx.chat().getLlmModel());

aiServerClient.streamChatCompletion(
        new AiChatRequest(content, 512, 0.7, null, ctxHolder[0].messages()),
        chunk -> { ... }
);
```

### 3.6 긴 대화 압축 (FR-5.2 — Compression)

맥락 조립 후 총 토큰이 `모델의 contextWindow × 0.8`을 초과하면, 오래된 turn부터 차례로 압축한다.

**압축 전략:**
- summary가 `GENERATED` 상태인 turn → `"이전 대화 요약: <summary>"` 1건으로 치환
- summary가 `PENDING`인 turn → content를 200자로 truncation + `"...[일부 생략]"`
- **마지막 2턴은 절대 압축하지 않음** (현재 맥락 보호)

```
모델별 압축 임계값:
┌──────────────────┬───────────────┬──────────────────┐
│ 모델             │ contextWindow │ 임계(×0.8)       │
├──────────────────┼───────────────┼──────────────────┤
│ gpt-4o-mini      │ 128,000       │ 102,400          │
│ gemini-2.0-flash │ 1,048,576     │ 838,860          │
│ claude-3.5-sonnet│ 200,000       │ 160,000          │
└──────────────────┴───────────────┴──────────────────┘
```

```java
// ContextAssembler.java — 압축 루프
int threshold = (int) (model.getContextWindow() * COMPRESSION_RATIO);  // 0.8
int compressibleCount = Math.max(0, orderedTurns.size() - PROTECTED_TURN_COUNT);  // 마지막 2턴 보호

for (int i = 0; i < compressibleCount && totalTokens > threshold; i++) {
    Turn turn = orderedTurns.get(i);
    if (turn.getSummary() != null) {
        // summary 치환
        replacement = List.of(Map.of("role", "user",
                "content", "이전 대화 요약: " + turn.getSummary()));
    } else {
        // truncation fallback
        replacement = truncateMessages(original);  // 200자 절단
    }
    totalTokens -= (oldTokens - newTokens);
    compressionApplied = true;
    compressedTurnCount++;
}
```

**결과 통보:** `turn_completed` SSE 이벤트에 압축 결과 필드를 포함한다.

```java
// MessageResDto.java — Phase 4 확장 필드
public record TurnCompleted(
        Long turnId, Long aiMessageId, Integer answerToken, String summaryStatus,
        Integer contextTokens,           // 압축 후 LLM에 전달된 입력 토큰 수
        Boolean compressionApplied,      // 압축 적용 여부
        Integer compressedTurnCount      // 압축된 turn 수
) {}
```

`message.prompt_token`에도 `contextTokens` 값을 저장하여, 이후 사용량 누적에 활용한다.

### 3.7 토큰 사용량 추적 (FR-5.3 — Usage Tracking)

```http
GET /api/v1/usage/me
```

현재 사용자의 활성 기간 토큰 사용량을 조회한다.

**응답 구조:**

```
UsageInfo
├── usageRecordId, planId, planName
├── periodStart, periodEnd
├── tokensUsed, tokenLimit, requestCount
├── remainingTokens        ← tokenLimit - tokensUsed (무제한이면 null)
├── usageRatio             ← tokensUsed / tokenLimit (무제한이면 null)
└── warningLevel           ← NONE / WARN(≥0.8) / CRITICAL(≥0.95)
```

**활성 record 자동 생성:** 조회 시점에 활성 record가 없으면, 가장 최근 record의 plan을 기준으로 새 기간을 자동 생성한다.

```java
// UsageService.java — 활성 record 조회 또는 자동 생성
public UsageResDto.UsageInfo getMyUsage(Long memberId) {
    LocalDateTime now = LocalDateTime.now();
    return usageRecordRepository.findActiveByMemberId(memberId, now)
            .map(UsageConverter::toUsageInfo)
            .orElseGet(() -> createNewPeriod(memberId, now));
}
```

**사용량 누적:** LLM 호출 완료 시 `MessageService`의 완료 트랜잭션에서 호출된다.

```java
// MessageService.java — 성공/취소/에러 세 경로 모두에서 누적
// 성공 시
usageService.accumulateUsage(memberId,
        ctxHolder[0].contextTokens(), answerToken,
        ctxHolder[0].compressedTurnCount());

// 취소/에러 시 (ctxHolder[0]이 null이 아닌 경우에만)
usageService.accumulateUsage(memberId,
        ctxHolder[0].contextTokens(), partialToken,
        ctxHolder[0].compressedTurnCount());
```

**WarningLevel:** `usageRatio` 기반으로 프론트엔드에 경고 수준을 전달한다.

```java
// WarningLevel.java
public enum WarningLevel {
    NONE, WARN, CRITICAL;

    public static WarningLevel from(Double usageRatio) {
        if (usageRatio == null) return NONE;
        if (usageRatio >= 0.95) return CRITICAL;
        if (usageRatio >= 0.8) return WARN;
        return NONE;
    }
}
```

---

## 4. ERD 변경분 (Phase 3 → Phase 4)

```sql
-- chat에 마지막 활동 시각 캐시 추가
ALTER TABLE chat
    ADD COLUMN last_activity_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP;
CREATE INDEX idx_chat_member_last_activity ON chat(member_id, last_activity_at DESC);

-- usage_record에 압축 누적 카운터 추가
ALTER TABLE usage_record
    ADD COLUMN compressed_turn_count INT NOT NULL DEFAULT 0;
CREATE INDEX idx_usage_member_period ON usage_record(member_id, period_end DESC);
```

**표기 정리 (스키마 동일):**
- `chat.last_turn_id`: `DEFAULT 0` 제거 → `NULL`로 시작
- `chat.root_chat_id`: `NULL` 유지 (IDENTITY PK 제약으로 INSERT 시점에 ID 불가, 앱 레벨 non-null 보장)

---

## 5. 패키지 구조 (Phase 4 추가분)

```
com.aistar.backend
├── domain
│   ├── chat
│   │   ├── controller/
│   │   │   └── ExplorerController.java    ─ 탐색기 트리 조회, 단일 트리 새로고침
│   │   ├── service/
│   │   │   ├── ExplorerService.java       ─ 탐색기 페이징, 배치 조회
│   │   │   └── ContextAssembler.java      ─ ancestor chain 맥락 조립 + 압축
│   │   ├── converter/
│   │   │   └── ExplorerConverter.java     ─ DFS 정렬, depth 계산
│   │   ├── dto/
│   │   │   └── ExplorerResDto.java        ─ ExplorerPage, TreeDto, NodeDto
│   │   └── enums/
│   │       └── ExplorerSort.java          ─ RECENT, CREATED, NAME
│   └── usage
│       ├── entity/
│       │   ├── Plan.java                  ─ 요금제
│       │   └── UsageRecord.java           ─ 기간별 토큰 사용량
│       ├── repository/
│       │   ├── PlanRepository.java
│       │   └── UsageRecordRepository.java
│       ├── enums/
│       │   └── WarningLevel.java          ─ NONE, WARN, CRITICAL
│       ├── dto/
│       │   └── UsageResDto.java           ─ UsageInfo 응답
│       ├── converter/
│       │   └── UsageConverter.java        ─ UsageRecord → UsageInfo 변환
│       ├── service/
│       │   └── UsageService.java          ─ 사용량 조회, 누적
│       └── controller/
│           └── UsageController.java       ─ GET /usage/me
```

---

## 6. 개발 중 발생한 이슈 및 해결

### 6.1 lastActivityAt NOT NULL 컬럼 추가 시 DDL 실패

**증상**: 서버 기동 시 `ALTER TABLE chat ADD COLUMN last_activity_at datetime(6) not null` 실패. 기존 row에 값이 없어 NOT NULL 제약 위반.

**원인**: `ddl-auto: update`에서 NOT NULL 컬럼을 추가하면 MySQL이 기존 row에 값을 채울 수 없음. `@CreatedDate`나 `@LastModifiedDate`는 JPA 엔티티 레벨에서만 동작하므로 DDL에 반영되지 않음.

**해결**: `columnDefinition = "timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP"` 지정. MySQL은 NOT NULL + DEFAULT가 있는 컬럼 추가 시 기존 row에 DEFAULT 값을 자동으로 채움.

### 6.2 rootChatId NOT NULL 시도 실패

**증상**: root chat 생성 시 `could not execute statement` — `root_chat_id`가 NOT NULL인데 INSERT 시점에 값이 없음.

**원인**: `@GeneratedValue(IDENTITY)` 특성상 INSERT 후에야 PK를 알 수 있음. root chat은 `rootChatId = 자기 chatId`여야 하는데, INSERT 전에는 chatId가 없으므로 NOT NULL 제약을 만족시킬 수 없음. `0L` 같은 임시값을 넣는 방법도 시도했으나 추가 복잡성만 발생.

**해결**: nullable 유지. `initRootChatId()`로 save 직후 자기 chatId를 저장. 앱 레벨에서 null인 row가 존재하지 않도록 보장. 명세서(§1.2)와 ERD에 이 결정을 반영.

### 6.3 contextResult 스코프 문제 (Virtual Thread + try/catch)

**증상**: `MessageService.streamMessage()` 내 Virtual Thread에서 `var contextResult = contextAssembler.buildContext(...)`를 try 블록에 선언하면, catch 블록에서 접근 불가.

**원인**: Java의 블록 스코프 규칙. try 내 지역 변수는 catch/finally에서 보이지 않는다. catch 블록에서도 `contextResult`의 압축 정보를 사용하여 사용량을 누적해야 한다.

**해결**: effectively final 배열 홀더 패턴 사용. `final ContextAssembler.ContextResult[] ctxHolder = {null}`을 try 밖에 선언하고, try 내에서 `ctxHolder[0] = ...`으로 할당. catch에서 `ctxHolder[0] != null` 체크 후 사용.

### 6.4 취소/에러 경로에서 detached entity 접근

**증상**: cancel/error catch 블록에서 `ctx.chat().getMember().getId()` 호출 시, 원래 트랜잭션이 끝난 후라 lazy loading이 될 수 있는 우려.

**원인**: `createTurnAndMessages()`가 `findByIdWithMemberAndDeletedAtIsNull()`로 member를 fetch join하여 영속 컨텍스트에 로딩해두므로, 프록시가 아닌 실제 객체가 TurnContext에 전달됨.

**해결**: 실제로 문제 없음. fetch join으로 이미 로딩된 상태이므로 detached 상태에서도 getter 호출 가능. 단, TransactionTemplate 콜백 내에서 사용하므로 안전.

---

## 7. 파일 변경 총정리

### 신규 파일 (15개)

| 파일 (`src/main/java/com/aistar/backend/` 기준) | 역할 |
|---|---|
| `domain/chat/controller/ExplorerController.java` | 탐색기 API 엔드포인트 |
| `domain/chat/service/ExplorerService.java` | 탐색기 페이징, 배치 조회 |
| `domain/chat/service/ContextAssembler.java` | ancestor chain 맥락 조립 + 압축 |
| `domain/chat/converter/ExplorerConverter.java` | DFS 정렬, depth 계산 |
| `domain/chat/dto/ExplorerResDto.java` | ExplorerPage, TreeDto, NodeDto |
| `domain/chat/enums/ExplorerSort.java` | RECENT, CREATED, NAME |
| `domain/usage/entity/Plan.java` | 요금제 엔티티 |
| `domain/usage/entity/UsageRecord.java` | 기간별 토큰 사용량 |
| `domain/usage/repository/PlanRepository.java` | Plan CRUD |
| `domain/usage/repository/UsageRecordRepository.java` | 활성 record 조회, 최근 record 조회 |
| `domain/usage/enums/WarningLevel.java` | NONE / WARN / CRITICAL |
| `domain/usage/dto/UsageResDto.java` | UsageInfo 응답 DTO |
| `domain/usage/converter/UsageConverter.java` | UsageRecord → UsageInfo 변환 |
| `domain/usage/service/UsageService.java` | 사용량 조회, 누적 |
| `domain/usage/controller/UsageController.java` | GET /usage/me |

### 수정 파일 (10개)

| 파일 | 변경 내용 |
|---|---|
| `domain/chat/entity/Chat.java` | `lastActivityAt` 필드 + 인덱스 + `touchLastActivityAt()` |
| `domain/chat/entity/Message.java` | `updatePromptToken()` 메서드 추가 |
| `domain/chat/enums/LlmModel.java` | `contextWindow` 필드 추가 |
| `domain/chat/repository/ChatRepository.java` | Explorer 페이징 쿼리 4개 추가 |
| `domain/chat/repository/TurnRepository.java` | 맥락 조립용 `findByChatIdAndTurnSequenceLteWithMessages` 추가 |
| `domain/chat/dto/MessageResDto.java` | TurnCompleted에 `contextTokens`, `compressionApplied`, `compressedTurnCount` 추가 |
| `domain/chat/service/ChatService.java` | `touchAncestorChain()` + createBranch/updateChatTitle 연결 |
| `domain/chat/service/MessageService.java` | ContextAssembler 연동, 압축 결과 turn_completed, 사용량 누적 |
| `global/apiPayload/code/ErrorStatus.java` | `EXPLORER_4001`, `CONTEXT_4001`, `USAGE_4041` 추가 |
| `global/config/DataInitializer.java` | Plan + UsageRecord 시드 데이터 추가 |
