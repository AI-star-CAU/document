# AIT Phase 2 테스트 보고서

## 1. 테스트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트 | AIT (AI + Git 브랜치 채팅 서비스) |
| 단계 | Phase 2 Walking Skeleton |
| 작성일 | 2026-05-17 |
| 테스트 프레임워크 | JUnit 5, Spring Boot Test, MockMvc |
| 테스트 DB | H2 In-Memory (ddl-auto: create-drop) |
| 테스트 실행 환경 | Java 21, Spring Boot 4.0.4, Gradle 9.3.1 |
| 테스트 클래스 | `Phase2ApiTest.java` |

## 2. 테스트 범위

### 대상 기능 (Phase 2 FR 기준)

| 기능 영역 | 테스트 대상 API |
|---|---|
| 인증 (Auth) | 회원가입, 로그인 |
| 보안 (Security) | JWT 인증, 소유권 검증 |
| 대화 (Chat) | 생성, 목록 조회, 메타정보 조회 |
| 턴 (Turn) | 커서 기반 목록 조회, 입력값 검증 |
| 메시지 (Message) | Turn/Message 생성, Cancel 상태 전이 |

### 테스트 제외 항목

| 항목 | 사유 |
|---|---|
| SSE 스트리밍 (실시간 chunk) | MockMvc 환경에서 SSE 이벤트 수신 불가, 가상 스레드와 @Transactional 비호환 |
| LLM 실제 호출 | MockLlmClient 사용 (Walking Skeleton 단계) |
| 대화 삭제 | Phase 2 범위 외 (Phase 3 예정) |

## 3. 단위 테스트

Phase 2 Service 계층은 비즈니스 로직이 단순하여(CRUD + 소유권 검증) 별도 단위 테스트 없이 통합 테스트에서 함께 검증한다. Service 메서드는 통합 테스트의 API 호출을 통해 간접적으로 테스트된다.

## 4. 통합 테스트

### 4.1 테스트 실행 결과 요약

| 항목 | 수치 |
|---|---|
| 총 테스트 수 | 19 |
| 성공 | 19 |
| 실패 | 0 |
| 건너뜀 | 0 |
| 총 소요 시간 | 3.726초 |

### 4.2 Auth — 인증 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 1 | 회원가입 성공 | 유효한 이메일/비밀번호/이름으로 가입 | 201 Created, accessToken 반환 | PASS | 0.076s |
| 2 | 이메일 중복 시 409 | 이미 존재하는 이메일로 가입 시도 | 409 Conflict, MEMBER_4091 | PASS | 0.084s |
| 3 | 로그인 성공 | 올바른 이메일/비밀번호 | 200 OK, accessToken + email/name/type 반환 | PASS | 0.155s |
| 4 | 로그인 실패 시 401 | 틀린 비밀번호 | 401 Unauthorized, AUTH_4012 | PASS | 0.130s |

### 4.3 Security — 인증/인가

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 5 | 토큰 없는 보호 API 접근 시 401 | Authorization 헤더 없이 GET /chats | 401 Unauthorized | PASS | 0.010s |
| 6 | 다른 사용자의 chat 접근 시 403 | 다른 회원의 대화 조회 시도 | 403 Forbidden, COMMON_403 | PASS | 0.136s |

### 4.4 Chat — 대화 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 7 | 새 대화 생성 성공 | title + llmProvider + llmModel 전송 | 201 Created, title/llmModel 반환 | PASS | 0.198s |
| 8 | 대화 목록 조회 성공 | 턴이 있는 채팅 목록 조회 | turnCount, lastMessageAt, lastMessagePreview 포함 | PASS | 0.141s |
| 9 | 대화 메타정보 조회 성공 | chatId로 상세 조회 | chatId, title 일치 | PASS | 0.079s |

### 4.5 Turn — 턴 목록 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 10 | 턴 목록 조회 성공 | 2개 턴 생성 후 조회 | ASC 순서 (1→2), senderType USER/ASSISTANT 확인 | PASS | 0.142s |
| 11 | limit=0 → 400 | limit 하한 위반 | 400 Bad Request, COMMON_4001 | PASS | 0.074s |
| 12 | limit=51 → 400 | limit 상한 위반 (최대 50) | 400 Bad Request, COMMON_4001 | PASS | 0.072s |
| 13 | lastTurnSequence=-1 → 400 | 음수 커서 값 | 400 Bad Request, COMMON_4001 | PASS | 0.072s |

### 4.6 Message — 메시지/Cancel API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 14 | 메시지 송신 시 turn/message 생성 | content 전송 | Turn(seq=1) + USER(COMPLETED) + ASSISTANT(STREAMING) 생성 | PASS | 0.187s |
| 15 | STREAMING → CANCELED | 스트리밍 중 cancel | 200 OK, status=CANCELED | PASS | 0.651s |
| 16 | COMPLETED 메시지 cancel 시 409 | 완료된 메시지 cancel 시도 | 409 Conflict, MESSAGE_4091 | PASS | 0.081s |
| 17 | USER 메시지 cancel 시 409 | 사용자 메시지 cancel 시도 | 409 Conflict, MESSAGE_4091 | PASS | 0.085s |
| 18 | 다른 chat의 message cancel 시 404 | chatB URL로 chatA 메시지 cancel | 404 Not Found, MESSAGE_4041 | PASS | 0.100s |
| 19 | cancel 멱등성 (idempotent) | 이미 CANCELED 상태에서 재 cancel | 200 OK, status=CANCELED (변경 없음) | PASS | 0.091s |

## 5. 시스템 테스트

프론트엔드-백엔드 연동 후 수동 테스트로 진행하며, 결과는 영상으로 기록하였다.

| # | 시나리오 | 검증 항목 | 결과 |
|---|---|---|---|
| 1 | 회원가입 → 로그인 | 토큰 발급, 이후 API 접근 가능 | 영상 참조 |
| 2 | 대화 생성 → 메시지 전송 | SSE 스트리밍 chunk 수신, UI 실시간 렌더링 | 영상 참조 |
| 3 | 대화 목록 조회 | 채팅 카드에 lastMessagePreview, turnCount 표시 | 영상 참조 |
| 4 | 턴 목록 스크롤 | BACKWARD 무한 스크롤, ASC 순서 렌더링 | 영상 참조 |
| 5 | 메시지 취소 | cancel 버튼 클릭 → CANCELED 상태 전환 | 영상 참조 |

## 6. 발견된 결함 및 조치 (개발 중 발견, 모두 조치 완료)

| # | 결함 | 심각도 | 원인 | 조치 | 상태 |
|---|---|---|---|---|---|
| 1 | H2 영속성 컨텍스트 캐시로 Turn.messages 미로딩 | 중 | 같은 트랜잭션에서 생성한 엔티티가 캐시되어 lazy loading 미발생 | `em.flush(); em.clear()` 추가 | 완료 |
| 2 | lastMessagePreview null 반환 | 중 | AI 메시지가 STREAMING 상태(content 없음)에서 조회 | 테스트에서 AI 메시지 COMPLETED 처리 후 검증 | 완료 |
| 3 | @Builder.Default 레코드 컴파일 오류 | 하 | Lombok @Builder.Default가 record 필드에서 미지원 | primitive boolean 기본값(false) 활용으로 해결 | 완료 |

## 7. 테스트 결과 종합

| 테스트 단계 | 총 케이스 | 성공 | 실패 | 비고 |
|---|---|---|---|---|
| 단위 테스트 | - | - | - | 통합 테스트에서 간접 검증 |
| 통합 테스트 | 19 | 19 | 0 | MockMvc + H2 기반 |
| 시스템 테스트 | 5 | - | - | 수동 테스트, 영상 기록 |

**결론:** Phase 2 Walking Skeleton의 핵심 API(인증, 대화, 턴, 메시지/취소)에 대한 통합 테스트 19개가 모두 통과하였다. SSE 실시간 스트리밍은 테스트 환경 제약으로 자동화 테스트 대상에서 제외되었으며, 프론트엔드 연동 후 시스템 테스트(영상)로 보완하였다.
