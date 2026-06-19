# 리팩터링: ai/ 패키지를 domain/llm/ 으로 통합

## 배경

AI 서버 통합 코드가 `com.aistar.backend.ai` 패키지에 독립적으로 존재하여 도메인형 패키지 구조(`domain/` 하위)와 일관성이 없었다. 기존에 `domain/llm/` 패키지가 client/dto/service 구조로 이미 존재했으나 Phase 2의 MockLlmClient만 남아있었고, 실제 AI 서버 통합 코드는 Phase 3에서 `ai/` 패키지에 별도로 추가되었다.

## 변경 내용

### 파일 이동

| 이전 경로 | 이후 경로 |
|---|---|
| `ai/AiServerClient.java` | `domain/llm/client/AiServerClient.java` |
| `ai/AiServerWebClient.java` | `domain/llm/client/AiServerWebClient.java` |
| `ai/AiServerException.java` | `domain/llm/client/AiServerException.java` |
| `ai/AiServerConfig.java` | `domain/llm/config/AiServerConfig.java` |
| `ai/AiServerProperties.java` | `domain/llm/config/AiServerProperties.java` |
| `ai/dto/*.java` (7개) | `domain/llm/dto/*.java` |

### 삭제

| 파일 | 사유 |
|---|---|
| `domain/llm/client/LlmClient.java` | Phase 2 Mock 인터페이스, AiServerClient로 대체됨 |
| `domain/llm/client/MockLlmClient.java` | Phase 2 Mock 구현체, AiServerWebClient로 대체됨 |
| `domain/llm/dto/.gitkeep` | DTO 파일 이동으로 불필요 |
| `domain/llm/service/` (빈 디렉토리) | 사용하지 않는 빈 패키지 |
| `ai/` (전체 디렉토리) | domain/llm/으로 이동 완료 |

### Import 변경

| 파일 | 변경 내용 |
|---|---|
| `MessageService.java` | `com.aistar.backend.ai.*` → `com.aistar.backend.domain.llm.client.*`, `domain.llm.dto.*` |
| `HealthController.java` | `com.aistar.backend.ai.AiServerClient` → `com.aistar.backend.domain.llm.client.AiServerClient` |

### 최종 domain/llm/ 구조

```
domain/llm/
  client/
    AiServerClient.java        (인터페이스)
    AiServerWebClient.java     (WebClient 구현체)
    AiServerException.java     (예외)
  config/
    AiServerConfig.java        (WebClient Bean 설정)
    AiServerProperties.java    (application.yml 매핑)
  dto/
    AiChatRequest.java
    AiChatStreamChunk.java
    AiCompletionResponse.java
    AiBranchTitleRequest.java
    AiBranchTitleResponse.java
    AiSummaryRequest.java
    AiSummaryResponse.java
```

## 검증

- 빌드 성공
- 전체 테스트 69개 통과 (Phase 2: 19, Phase 3: 24, Phase 4: 25, Application: 1)
- 동작 변경 없음 (패키지 경로만 변경)
