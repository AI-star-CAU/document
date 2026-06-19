# AI 서버 연동 README 실행 문제 해결 기록

## 배경

BE는 README에 적힌 기본 방식으로 실행되어야 한다.

```bash
./gradlew clean build
java -jar build/libs/ai-star-server.jar
```

이번 검증에서는 별도의 `source <(ssh HomeServer ...)` 명령 없이, 프로젝트 루트의 `.env`와 README 실행 명령만으로 AI 서버 응답까지 확인하는 것을 목표로 했다.

## 문제점

### 1. README 실행 확인 URL이 현재 서버 설정과 맞지 않음

현재 서버는 `server.servlet.context-path=/api/v1` 기준으로 동작한다. 하지만 README의 실행 확인 URL은 `/health`, `/hello` 등 이전 경로 기준으로 남아 있었다.

그 결과 README만 따라 실행하면 Swagger와 health check를 어떤 경로로 확인해야 하는지 불명확했다.

### 2. AI 서버 접속 환경 변수가 BE 실행 환경에 없었음

AI 서버는 `https://ait.hanbin5.com`으로 항상 떠 있지만, BE가 Cloudflare Access 뒤의 AI 서버로 요청하려면 다음 값이 필요하다.

- `AI_SERVER_BASE_URL`
- `AISTAR_AI_API_KEY`
- `CF_ACCESS_CLIENT_ID`
- `CF_ACCESS_CLIENT_SECRET`

기존에는 실행 전에 원격 AI repo의 `.env`에서 값을 가져와 shell에 주입하는 방식으로 검증했다. 이 방식은 동작은 하지만 README의 단순 실행 방식과 맞지 않았다.

### 3. Chat/Turn 엔티티 컬럼명이 로컬 DB 스키마와 달랐음

phase3 병합 후 `Chat` 엔티티의 PK와 `Turn`의 FK가 `chat_id` 기준으로 바뀌어 있었다. 하지만 현재 로컬 MySQL `AIT` 스키마의 `chat` PK는 `aichat_session_id`이고, `turn`도 이 컬럼을 FK로 사용한다.

이 mismatch 때문에 메시지 전송 API 검증 중 `turn` insert에서 다음 문제가 발생했다.

```text
Field 'chat_id' doesn't have a default value
```

즉, 서버는 떠도 실제 메시지 생성과 AI 응답 저장 흐름이 끝까지 통과하지 못했다.

## 해결 내용

### 1. README 실행 확인 경로 수정

README의 실행 확인 URL을 현재 context path 기준으로 수정했다.

- `http://localhost:8080/api/v1/health`
- `http://localhost:8080/api/v1/swagger-ui`
- `http://localhost:8080/api/v1/api-docs`

### 2. README 환경 변수 목록에 AI 서버 접속값 추가

README의 `.env` 예시에 AI 서버 연동에 필요한 환경 변수 이름을 추가했다. 실제 secret 값은 README나 Git에 기록하지 않는다.

```properties
AI_SERVER_BASE_URL=https://ait.hanbin5.com
AISTAR_AI_API_KEY=AI 서버 API key
CF_ACCESS_CLIENT_ID=Cloudflare Access Client ID
CF_ACCESS_CLIENT_SECRET=Cloudflare Access Client Secret
```

BE 로컬 `.env`에는 위 값들을 추가했고, 파일 권한은 `600`으로 제한했다.

### 3. 엔티티 매핑을 현재 DB 스키마에 맞춤

현재 DB 스키마와 맞도록 엔티티 매핑을 되돌렸다.

- `Chat.id`: `chat_id` -> `aichat_session_id`
- `Turn.chat`: `chat_id` -> `aichat_session_id`
- `idx_turn_chat_sequence`: `aichat_session_id, turn_sequence` 기준으로 수정

로컬 DB에는 이전 실패 실행 흔적으로 `turn.chat_id` 컬럼이 남아 있어, 해당 stray 컬럼과 인덱스를 제거하고 `aichat_session_id` 기준 인덱스를 다시 만들었다.

## 검증 결과

다음 명령으로 빌드와 실행을 검증했다.

```bash
cd /Users/hanbin5/Lab/SoftwareEngineering/AI-star-be
./gradlew clean build
java -jar build/libs/ai-star-server.jar
```

확인 결과:

- Gradle build 성공
- README 방식의 JAR 단독 실행 성공
- `GET /api/v1/health` 200 OK
- `GET /api/v1/swagger-ui/index.html` 200 OK
- 로그인 성공
- 채팅 생성 성공
- `POST /api/v1/chats/{chatId}/messages` SSE 응답 성공
- AI 서버에서 `chunk`, `turn_completed`, `done` 이벤트 수신 확인

## 이후 테스트 방법

개발자가 같은 흐름을 다시 확인할 때는 별도 SSH source 없이 아래 순서만 사용한다.

```bash
cd /Users/hanbin5/Lab/SoftwareEngineering/AI-star-be
./gradlew clean build
java -jar build/libs/ai-star-server.jar
```

서버가 뜬 뒤 브라우저 또는 curl로 확인한다.

```bash
curl http://localhost:8080/api/v1/health
open http://localhost:8080/api/v1/swagger-ui
```

Swagger에서는 회원가입 또는 로그인 후 JWT를 넣고, 채팅 생성 API와 메시지 전송 API를 순서대로 호출하면 AI 서버 SSE 응답을 확인할 수 있다.
