| 마일스톤 | 이름                | 핵심 목표                | 주요 산출물                                            |
| ---- | ----------------- | -------------------- | ------------------------------------------------- |
| M1   | Schema            | Phase4 기반 스키마/엔티티 준비 | Chat 수정, Plan/UsageRecord 추가, ErrorCode 추가        |
| M2   | lastActivityAt 전파 | 최근 활동 시각 갱신 정책 연결    | ChatService/MessageService 일부 수정                  |
| M3   | Explorer API      | 대화 탐색기 트리 조회 구현      | ExplorerController, ExplorerService, Explorer DTO |
| M4   | Context Assembly  | LLM 요청용 분기별 맥락 조립    | ContextService, AiChatRequest.messages 연결         |
| M5   | Compression       | 긴 맥락 압축 및 SSE 확장     | 압축 로직, turn_completed 필드 추가                       |
| M6   | Usage Tracking    | 토큰 사용량 조회/누적         | UsageController, UsageService, Usage DTO          |
| M7   | Integration       | 전체 E2E 검증            | 테스트, 회귀 검증                                        |




# Phase 4 구현 마일스톤 정리

## 전체 의존성 흐름

```text
M1 Schema
 ├─> M2 lastActivityAt 전파 ──> M3 Explorer API
 ├─> M4 Context Assembly ──> M5 Compression ──> M6 Usage Tracking
 └─> M7 Integration
```

핵심은 **DB/엔티티 기반을 먼저 만들고**, 그다음 **탐색기 계열**과 **LLM 맥락 계열**을 나눠서 구현한 뒤, 마지막에 전체 통합 검증하는 흐름.

---

# M1. Schema

## 목표

Phase 4 구현을 위한 **기반 스키마·엔티티·에러 코드 준비**.

## 주요 작업

* `Chat` 엔티티 수정

  * `lastActivityAt` 추가
  * `rootChatId NOT NULL` 정책 반영
  * `lastTurnId DEFAULT 0` 제거 확인
* `Plan` 엔티티 추가
* `UsageRecord` 엔티티 추가
* `PlanRepository` 추가
* `UsageRecordRepository` 추가
* Phase 4 에러 코드 추가

  * `EXPLORER_4001`
  * `CONTEXT_4001`
  * `USAGE_4041`
* `LlmModel` 또는 모델 메타데이터에 `contextWindow` 추가
* 인덱스/컬럼 설정 반영

## 하지 말 것

* Explorer API 구현
* Context Assembly 구현
* Compression 구현
* Usage Tracking 구현
* SSE 수정
* `MessageService` LLM 호출 흐름 수정

## 검증 기준

* 컴파일 통과
* 기존 테스트 통과
* 엔티티 매핑 문제 없음

---

# M2. lastActivityAt 전파

## 목표

Phase 4 탐색기 정렬 기준인 `lastActivityAt`을 실제 서비스 흐름에 연결.

## 주요 작업

* root chat 생성 시 `lastActivityAt = createdAt`
* root chat 생성 후 `rootChatId = 자기 chatId` self-update
* branch 생성 시

  * branch의 `rootChatId = parent.rootChatId`
  * branch의 `lastActivityAt = createdAt`
  * 부모부터 root까지 ancestor chain의 `lastActivityAt` 갱신
* 메시지 송신 시 해당 chat과 ancestor chain 갱신
* 재생성 시 해당 chat과 ancestor chain 갱신
* 사용자 메시지 수정 시 해당 chat과 ancestor chain 갱신
* chat 제목 수정 시 해당 chat과 ancestor chain 갱신
* branch 삭제 시에는 갱신하지 않음

## 주의점

`lastActivityAt`은 단순히 현재 chat만 갱신하면 안 됨.
분기 내부 활동이 root 탐색기 정렬에도 반영되어야 하므로 **ancestor chain 전체 갱신** 필요.

## 검증 기준

* root chat 생성 후 `rootChatId = chatId`
* branch 생성 후 `branch.rootChatId = root.chatId`
* branch 생성 시 부모/root의 `lastActivityAt` 갱신
* branch 삭제 시 `lastActivityAt` 미갱신

---

# M3. Explorer API

## 목표

파일 탐색기 형태의 대화 트리 조회 API 구현.

## 구현 API

```http
GET /api/v1/chats/explorer
```

## 주요 작업

* `ExplorerController` 추가
* `ExplorerService` 추가
* `ExplorerResDto` 추가

  * `ExplorerTreeDto`
  * `ExplorerNodeDto`
* root chat 페이지네이션
* `sort=recent | created | name` 처리
* `includeDeleted=false` 기본값 처리
* 페이징된 root들의 `rootChatId` 목록 조회
* `root_chat_id IN (:pagedRootIds)`로 하위 branch 전체 일괄 조회
* 메모리에서 DFS 정렬
* `turnCount`는 `TurnRepository.countByChatIds(chatIds)` 재사용
* root별 N+1 count 쿼리 금지

## 쿼리 원칙

```text
1. root 목록 페이징
2. root_chat_id IN (:pagedRootIds)로 트리 전체 일괄 조회
3. 조회된 chatId 목록으로 turnCount GROUP BY 일괄 계산
```

## 하지 말 것

* `GET /chats/explorer/{rootChatId}` 구현

  * 명세상 선택 API이므로 이번 필수 구현에서는 제외

## 검증 기준

* root 페이지네이션 정상 동작
* recent/created/name 정렬 정상 동작
* `includeDeleted=false`일 때 삭제 chat 제외
* DFS 순서 정상
* `turnCount` 정확
* root별 개별 count 쿼리 없음

---

# M4. Context Assembly

## 목표

LLM 호출 시 단일 user message만 보내던 기존 구조를, Phase 4 명세의 **분기별 맥락 조립 방식**으로 변경.

## 주요 작업

* `ContextService` 추가 권장
* target chat 기준 ancestor chain 생성
* chain을 root → target 순서로 정렬
* target chat은 모든 turn 포함
* ancestor chat은 다음 child의 `branchPointTurnId`까지만 포함
* sibling branch 제외
* descendant branch 제외
* ancestor의 분기점 이후 turn 제외
* 조립된 `flat_messages`를 `AiChatRequest.messages`에 전달
* 기존 `messages=null` 호출 제거

## 핵심 규칙

```text
target chat:
  모든 turn 포함

ancestor chat:
  chain의 다음 child가 분기된 branchPointTurnId까지만 포함
```

## 예시

```text
root: turn1, turn2, turn3, turn4, turn5
branch가 turn3에서 생성됨

branch에서 LLM 호출 시:
root turn1, turn2, turn3
+ branch의 모든 turn
+ 새 user message

root turn4, turn5는 제외
```

## 검증 기준

* ancestor chain 정확히 생성
* 분기점 이후 parent turn 제외
* sibling branch 제외
* target chat turn 전체 포함
* `AiChatRequest.messages`에 실제 맥락 배열 전달

---

# M5. Compression

## 목표

LLM context window 초과 시 오래된 turn을 summary 또는 truncation으로 압축.

## 주요 작업

* 모델별 `contextWindow` 기준 적용
* `compressionRatio = 0.8` 기본값 적용
* 오래된 turn부터 압축
* `summary`가 `GENERATED`이면 summary 메시지로 치환
* `summary`가 `PENDING`이면 content truncation fallback
* 최근 N턴 보호

  * 기본 N=2
* 압축 후에도 초과하면 MVP 정책대로 LLM 호출 시도
* `context_length_exceeded`는 `LLM_5001`로 매핑
* `context_overflow_after_compression` 로그 기록
* `turn_completed` SSE에 필드 추가

  * `contextTokens`
  * `compressionApplied`
  * `compressedTurnCount`

## compressedTurnCount 의미

```text
summary 치환된 turn 수 + truncation 적용된 turn 수
```

## 검증 기준

* summary GENERATED turn은 summary로 치환
* summary PENDING turn은 truncation fallback
* 최근 2턴은 압축하지 않음
* `contextTokens` 계산
* `compressionApplied` 정확
* `compressedTurnCount` 정확
* SSE `turn_completed`에 신규 필드 포함

---

# M6. Usage Tracking

## 목표

사용자의 토큰 사용량 조회 및 누적 관리 구현.

## 구현 API

```http
GET /api/v1/usage/me
```

## 주요 작업

* `UsageController` 추가
* `UsageService` 추가
* `UsageResDto` 추가
* 현재 활성 usage period 조회
* 활성 `usage_record` 없으면:

  * plan 있음 → 새 usage_record 생성 후 0으로 반환
  * plan 없음 → `USAGE_4041`
* LLM 호출 완료 후 사용량 누적

  * `tokens_used += promptToken + answerToken`
  * `request_count += 1`
  * `compressed_turn_count += compressedTurnCount`
* `warningLevel` 계산

  * `NONE`
  * `WARN`
  * `CRITICAL`
* `tokenLimit=0`이면 무제한 처리

  * `remainingTokens = null`
  * `usageRatio = null`
  * `warningLevel = NONE`

## promptToken 의미

```text
message.promptToken = 압축 후 실제 LLM에 전달된 입력 토큰 수
```

즉 SSE의 `contextTokens`와 같은 값.

## 검증 기준

* `/usage/me` 정상 응답
* 활성 record 없을 때 자동 생성
* plan 없을 때 `USAGE_4041`
* remainingTokens 계산
* usageRatio 계산
* warningLevel 계산
* LLM 완료 후 tokensUsed/requestCount/compressedTurnCount 누적

---

# M7. Integration

## 목표

Phase 4 전체 기능의 E2E 검증.

## 주요 검증 시나리오

### 1. Explorer 시나리오

```text
root chat 생성
→ branch 생성
→ GET /chats/explorer 호출
→ root와 branch가 DFS 순서로 반환되는지 확인
```

### 2. lastActivityAt 시나리오

```text
branch에서 메시지 송신
→ branch/root의 lastActivityAt 갱신
→ explorer recent 정렬에 반영
```

### 3. Context Assembly 시나리오

```text
root turn3에서 branch 생성
root turn4, turn5 추가
branch에서 메시지 송신
→ LLM context에 root turn1~3만 포함
→ root turn4~5는 제외
```

### 4. Compression 시나리오

```text
긴 대화 생성
→ contextWindow * 0.8 초과
→ 오래된 turn 압축
→ turn_completed에 compressionApplied=true
```

### 5. Usage 시나리오

```text
LLM 응답 완료
→ usage_record.tokens_used 증가
→ GET /usage/me에서 반영 확인
```

## 검증 명령

```bash
./gradlew test
```

필요 시:

```bash
./gradlew clean build
```

## 완료 기준

* 컴파일 성공
* 기존 테스트 통과
* Phase 4 신규 테스트 통과
* 필수 API 2개 동작

  * `GET /api/v1/chats/explorer`
  * `GET /api/v1/usage/me`
* 기존 메시지 송신/재생성/수정 흐름 깨지지 않음


---

# 진행 원칙

```text
각 마일스톤은 독립적으로 완료하고 멈춘다.
다음 마일스톤으로 넘어가기 전에 변경 파일과 검증 결과를 보고한다.
명세와 충돌하면 구현하지 말고 먼저 보고한다.
선택 API는 명시적으로 지시하기 전까지 구현하지 않는다.
관련 없는 리팩토링은 하지 않는다.
```

