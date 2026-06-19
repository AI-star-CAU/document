# Phase 5 구현 요약

## 1. 개요

Phase 4(탐색기, 맥락 조립/압축, 사용량 추적)까지의 기능을 기반으로, 발표 전 MVP 완성도를 높이기 위한 **리팩터링**과 **누락 기능 구현**을 진행하였다.

주요 변경:
- `ai/` 패키지를 `domain/llm/`으로 통합하여 도메인형 패키지 구조 일관성 확보
- 로컬 LLM 전환에 따른 모델/제공자 구조 개선
- AI 서버를 활용한 실제 요약(Summary) 생성
- 비동기 제목 자동 생성 구현

---

## 2. 변경 내역

### 2.1 패키지 리팩터링 — ai/ → domain/llm/

기존에 AI 서버 통합 코드가 `com.aistar.backend.ai` 패키지에 독립적으로 존재하여, `domain/` 하위의 도메인형 패키지 구조와 일관성이 없었다. 이를 `domain/llm/` 아래로 통합했다.

**이동 내역:**

| 이전 위치 | 이후 위치 |
|---|---|
| `ai/AiServerClient.java` | `domain/llm/client/AiServerClient.java` |
| `ai/AiServerWebClient.java` | `domain/llm/client/AiServerWebClient.java` |
| `ai/AiServerException.java` | `domain/llm/client/AiServerException.java` |
| `ai/AiServerConfig.java` | `domain/llm/config/AiServerConfig.java` |
| `ai/AiServerProperties.java` | `domain/llm/config/AiServerProperties.java` |
| `ai/dto/*.java` (7개) | `domain/llm/dto/*.java` |

**삭제:** Phase 2의 `LlmClient` 인터페이스와 `MockLlmClient`는 `AiServerClient`로 대체되어 삭제.

**영향 범위:** `MessageService`, `HealthController`의 import 경로 변경. 동작 변경 없음.

### 2.2 로컬 LLM 기본 지원

외부 AI API(OpenAI, Google, Anthropic)가 아닌 로컬 LLM을 사용하는 방향으로 전환됨에 따라, 모델/제공자 구조를 개선했다.

**변경 내용:**

- `LlmProvider`에 `LOCAL` 추가
- `LlmModel`에 `LOCAL_DEFAULT` 추가 (contextWindow: 128,000)
- 채팅 생성 시 `llmProvider`/`llmModel` 미지정이면 자동으로 `LOCAL` / `LOCAL_DEFAULT` 적용
- 기존 외부 모델(OPENAI, GOOGLE, ANTHROPIC)은 enum에 유지하여 추후 확장 가능

**ChatReqDto.Create 변경:**

채팅 생성 요청에서 `llmProvider`/`llmModel`이 선택 사항으로 변경됐다.
- 미지정 시 → `LOCAL` / `LOCAL_DEFAULT` 자동 적용
- 명시적 지정 시 → 지정된 값 사용
- 기존 프론트엔드 코드와 호환됨 (보내든 안 보내든 동작)

### 2.3 contextWindow 프로퍼티 기반 전환

맥락 압축 임계값 계산 시 `LlmModel.contextWindow`에 의존하던 구조를 `application.yml` 프로퍼티로 변경했다. 로컬 LLM 환경에서 모델 enum에 종속되지 않고 설정으로 관리할 수 있다.

**변경 전:** `ContextAssembler`가 `LlmModel.getContextWindow()`를 직접 참조
**변경 후:** `AiServerProperties.contextWindow()`를 주입받아 사용

```yaml
# application.yml
ai:
  server:
    context-window: 128000
```

### 2.4 AI 요약(Summary) 실제 호출

Phase 3~4에서 `generateSummaryAsync()`가 단순히 응답의 앞 50자를 잘라서 저장하던 것을, 실제 AI 서버에 요약을 요청하도록 변경했다.

**동작 흐름:**
1. 메시지 스트리밍 완료 후 비동기로 `aiServerClient.generateSummary()` 호출
2. AI 서버가 대화 내용을 50자 이내 한국어로 요약하여 반환
3. 요약 결과를 `turn.summary`에 저장

**AI 서버 실패 시:** 기존과 동일하게 응답 앞 50자를 fallback으로 저장. 요약이 아예 없는 상태는 발생하지 않음.

### 2.5 비동기 제목 자동 생성

채팅/분기 생성 시 제목을 지정하지 않으면 `titleStatus = PENDING`으로 남는데, 이전까지는 제목이 자동 생성되지 않았다. 이제 메시지 완료 시 제목이 PENDING이면 AI 서버에 제목 생성을 요청한다.

**동작 흐름:**
1. 메시지 스트리밍 완료 후 `titleStatus == PENDING` 확인
2. 사용자 질문 + AI 응답을 context로 전달하여 `aiServerClient.generateBranchTitle()` 호출
3. AI 서버가 20자 이내 한국어 제목을 생성하여 반환
4. `titleStatus`를 `GENERATED`로 변경하고 제목 저장

**AI 서버 실패 시:** 사용자 질문의 앞 20자를 fallback 제목으로 사용.

**안전 장치:**
- 저장 시점에 `titleStatus == PENDING` 재확인 → 사용자가 이미 수동 편집한 경우 덮어쓰지 않음
- `turnSequence` 제한 없음 → 첫 메시지가 실패/취소되어도 이후 메시지 성공 시 제목 생성

**프론트엔드 연동:**
- `useConversations` — Explorer에서 PENDING 노드 존재 시 3초 polling
- `useGraph` — 그래프에서 PENDING title/summary 존재 시 3초 polling
- `useChatTitle` — PENDING이면 "제목 생성 중…" 표시, GENERATED 시 실제 제목 반영
- 모든 PENDING 해소 시 polling 자동 중지

### 2.6 AI 서버 응답 토큰 확대

`max_new_tokens`을 512 → 1024로 변경하여 AI 서버가 더 긴 응답을 생성할 수 있도록 했다.

---

## 3. 커밋 이력

| 커밋 | 설명 |
|------|------|
| `2209985` | ai/ 패키지를 domain/llm/으로 통합 및 테스트 보고서 추가 |
| `5233279` | LOCAL LLM 기본 지원 및 AI 요약 실제 호출 구현 |
| `0471fc7` | 첫 턴 완료 시 비동기 AI 제목 자동 생성 |
| `53b807b` | 제목 생성 조건에서 turnSequence 제한 제거 |

---

## 4. 파일 변경 총정리

### 신규 파일 (1개)

| 파일 | 역할 |
|---|---|
| `docs/refactoring/ai_package_to_domain_llm.md` | ai → domain/llm 리팩터링 문서 |

### 이동 파일 (12개, ai/ → domain/llm/)

| 이전 | 이후 |
|---|---|
| `ai/AiServerClient.java` | `domain/llm/client/AiServerClient.java` |
| `ai/AiServerWebClient.java` | `domain/llm/client/AiServerWebClient.java` |
| `ai/AiServerException.java` | `domain/llm/client/AiServerException.java` |
| `ai/AiServerConfig.java` | `domain/llm/config/AiServerConfig.java` |
| `ai/AiServerProperties.java` | `domain/llm/config/AiServerProperties.java` |
| `ai/dto/Ai*.java` (7개) | `domain/llm/dto/Ai*.java` |

### 삭제 파일 (3개)

| 파일 | 사유 |
|---|---|
| `domain/llm/client/LlmClient.java` | AiServerClient로 대체 |
| `domain/llm/client/MockLlmClient.java` | AiServerWebClient로 대체 |
| `domain/llm/dto/.gitkeep` | DTO 파일 이동으로 불필요 |

### 수정 파일 (13개)

| 파일 | 변경 내용 |
|---|---|
| `domain/chat/entity/Chat.java` | `updateGeneratedTitle()` 메서드 추가 |
| `domain/chat/enums/LlmProvider.java` | `LOCAL` 추가 |
| `domain/chat/enums/LlmModel.java` | `LOCAL_DEFAULT` 추가 |
| `domain/chat/dto/ChatReqDto.java` | provider/model 선택 사항 변경, default 메서드 추가 |
| `domain/chat/service/ChatService.java` | `llmProviderOrDefault()` / `llmModelOrDefault()` 사용 |
| `domain/chat/service/ContextAssembler.java` | `AiServerProperties.contextWindow()` 기반으로 변경 |
| `domain/chat/service/MessageService.java` | AI 요약 호출, 비동기 제목 생성, max_new_tokens 1024 |
| `domain/llm/config/AiServerProperties.java` | `contextWindow` 필드 추가 |
| `global/config/DataInitializer.java` | LOCAL / LOCAL_DEFAULT 사용 |
| `global/config/HealthController.java` | import 경로 변경 |
| `src/main/resources/application.yml` | `context-window` 프로퍼티 추가 |
| `src/test/resources/application.yml` | 테스트용 ai.server 프로퍼티 추가 |
| `Phase2/3/4ApiTest.java` | LOCAL / LOCAL_DEFAULT로 변경 |

---

## 5. 검증

- 빌드 성공
- 전체 테스트 통과 (Phase 2: 19개, Phase 3: 24개, Phase 4: 25개, Application: 1개)
- 동작 변경: 요약/제목이 AI 서버를 통해 실제 생성됨 (기존: 단순 substring)
- 하위 호환: 프론트엔드 수정 없이 기존 API 계약 유지
