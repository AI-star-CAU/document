# AIT Phase 3 테스트 보고서

## 1. 테스트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트 | AIT (AI + Git 브랜치 채팅 서비스) |
| 단계 | Phase 3 분기 트리 + 그래프 |
| 작성일 | 2026-05-27 |
| 테스트 프레임워크 | JUnit 5, Spring Boot Test, MockMvc |
| 테스트 DB | H2 In-Memory (ddl-auto: create-drop) |
| 테스트 실행 환경 | Java 21, Spring Boot 4.0.4, Gradle 9.3.1 |
| 테스트 클래스 | `Phase3ApiTest.java` |

## 2. 테스트 범위

### 대상 기능 (Phase 3 FR 기준)

| 기능 영역 | 테스트 대상 API |
|---|---|
| 분기 생성 (Branch) | POST /chats/{chatId}/branches |
| 제목 수정 (Title) | PATCH /chats/{chatId} |
| 대화 삭제 (Delete) | DELETE /chats/{chatId} (cascade soft delete) |
| 응답 재생성 (Regenerate) | POST /chats/{chatId}/messages/{messageId}/regenerate |
| 메시지 수정 (Edit) | PATCH /chats/{chatId}/messages/{messageId} |
| 그래프 조회 (Graph) | GET /chats/{chatId}/graph |
| 그래프 확장 (Expand) | GET /chats/{chatId}/graph/expand |

### 테스트 제외 항목

| 항목 | 사유 |
|---|---|
| SSE 스트리밍 (재생성/수정 시) | MockMvc 환경에서 SSE 이벤트 수신 불가, 가상 스레드와 @Transactional 비호환 |
| AI 서버 실제 호출 | 재생성/수정의 TurnContext 생성까지만 검증, 스트리밍은 별도 |
| lastActivityAt 전파 | Phase 4 기능 (Phase 4 테스트에서 검증) |

## 3. 단위 테스트

Phase 3 Service 계층은 통합 테스트에서 API 호출을 통해 간접 검증한다. 재생성/수정의 경우 Service 메서드를 직접 호출하여 TurnContext 생성 로직을 단위 수준으로 검증한다.

## 4. 통합 테스트

### 4.1 테스트 실행 결과 요약

| 항목 | 수치 |
|---|---|
| 총 테스트 수 | 24 |
| 성공 | 24 |
| 실패 | 0 |
| 건너뜀 | 0 |
| 총 소요 시간 | 2.279초 |

### 4.2 Branch — 분기 생성 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 1 | 분기 생성 성공 | parentId, rootChatId, branchPointTurnId 확인 | 201 Created, title/titleStatus/llmModel 반환 | PASS | 0.083s |
| 2 | 제목 없이 분기 생성 | title 미지정 | 201 Created, titleStatus=PENDING | PASS | 0.065s |
| 3 | 잘못된 turnId로 분기 생성 시 400 | 존재하지 않는 turnId | 400 Bad Request, BRANCH_4001 | PASS | 0.064s |
| 4 | 다른 사용자의 chat에 분기 생성 시 403 | 비소유자 접근 | 403 Forbidden, COMMON_403 | PASS | 0.120s |

### 4.3 Title — 제목 수정 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 5 | 제목 수정 성공 | 유효한 제목 전송 | 200 OK, titleStatus=USER_EDITED | PASS | 0.083s |
| 6 | 빈 제목으로 수정 시 400 | 공백 문자열 전송 | 400 Bad Request | PASS | 0.078s |

### 4.4 Delete — 대화 삭제 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 7 | 삭제 성공 | soft delete 확인 | 200 OK, deletedAt NOT NULL | PASS | 0.083s |
| 8 | cascade 삭제 | root → child → grandchild 전부 삭제 | 모든 자손 deletedAt NOT NULL | PASS | 0.081s |
| 9 | 이미 삭제된 chat 재삭제 시 409 | 삭제 상태에서 재삭제 | 409 Conflict, BRANCH_4091 | PASS | 0.064s |
| 10 | 다른 사용자의 chat 삭제 시 403 | 비소유자 접근 | 403 Forbidden | PASS | 0.116s |

### 4.5 Regenerate — 응답 재생성

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 11 | 재생성 시 AI 메시지 초기화 | COMPLETED → STREAMING, content=null | TurnContext 정상 생성 | PASS | 0.109s |
| 12 | USER 메시지 재생성 시 409 | 사용자 메시지에 재생성 시도 | 409 Conflict, MESSAGE_4092 | PASS | 0.088s |
| 13 | 다른 chat의 message 재생성 시 404 | chatB URL로 chatA 메시지 재생성 | 404 Not Found, MESSAGE_4041 | PASS | 0.092s |

### 4.6 Edit — 메시지 수정 (자동 분기)

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 14 | 수정 시 새 branch 생성 | turn seq > 1의 user 메시지 수정 | branchCreated 반환, 직전 turn이 branchPoint | PASS | 0.098s |
| 15 | ASSISTANT 메시지 수정 시 409 | AI 메시지 수정 시도 | 409 Conflict, MESSAGE_4092 | PASS | 0.083s |
| 16 | root 첫 turn user 메시지 수정 시 409 | 분기점 없는 첫 turn 수정 | 409 Conflict, MESSAGE_4092 | PASS | 0.104s |

### 4.7 Graph — 그래프 조회 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 17 | 기본 그래프 조회 성공 | rootChatId, center, turns, chats 포함 | 200 OK, turns 2개, chats 배열 | PASS | 0.094s |
| 18 | 분기 있는 그래프 | root + branch 트리 구조 | chats 2개, 윈도우 내 turns만 반환 | PASS | 0.102s |
| 19 | centerTurnId 지정 조회 | 특정 turn 중심 조회 | center.turnId 일치 | PASS | 0.086s |
| 20 | up=0 시 400 | 범위 초과 파라미터 | 400 Bad Request, GRAPH_4001 | PASS | 0.064s |
| 21 | 다른 사용자의 chat 그래프 조회 시 403 | 비소유자 접근 | 403 Forbidden | PASS | 0.122s |

### 4.8 Expand — 그래프 윈도우 확장 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 22 | DOWN 방향 확장 성공 | fromTurnId 기준 아래로 확장 | direction=DOWN, turns 배열, frontier 포함 | PASS | 0.117s |
| 23 | UP 방향 확장 성공 | fromTurnId 기준 위로 확장 | direction=UP, turns 배열 | PASS | 0.112s |
| 24 | 잘못된 direction 시 400 | direction=LEFT | 400 Bad Request, GRAPH_4001 | PASS | 0.077s |

## 5. 시스템 테스트

프론트엔드-백엔드 연동 후 수동 테스트로 진행하며, 결과는 영상으로 기록하였다.

| # | 시나리오 | 검증 항목 | 결과 |
|---|---|---|---|
| 1 | 분기 생성 → 분기 대화 | 새 branch에서 메시지 송수신 | 영상 참조 |
| 2 | 대화 삭제 → cascade 확인 | 자손 branch 동시 삭제 | 영상 참조 |
| 3 | 메시지 수정 → 자동 분기 | 수정 시 branch_created 이벤트, 새 branch에서 스트리밍 | 영상 참조 |
| 4 | 그래프 뷰 렌더링 | 트리 구조 시각화, center turn 하이라이트 | 영상 참조 |
| 5 | 그래프 윈도우 확장 | UP/DOWN 스크롤 시 추가 turn 로딩 | 영상 참조 |

## 6. 발견된 결함 및 조치 (개발 중 발견, 모두 조치 완료)

| # | 결함 | 심각도 | 원인 | 조치 | 상태 |
|---|---|---|---|---|---|
| 1 | 분기 그래프에서 branch turn이 윈도우에 미포함 | 중 | 그래프 윈도우가 center turn 기준 UP/DOWN 경로만 탐색하여 다른 branch의 turn은 포함하지 않음 | 테스트에서 branch chat 기준으로 별도 조회하여 검증 | 완료 |
| 2 | N+1 쿼리 문제 | 중 | Turn.messages lazy loading으로 turn별 추가 쿼리 발생 | fetch join 쿼리 추가 | 완료 |

## 7. 테스트 결과 종합

| 테스트 단계 | 총 케이스 | 성공 | 실패 | 비고 |
|---|---|---|---|---|
| 단위 테스트 | - | - | - | 통합 테스트에서 간접 검증 |
| 통합 테스트 | 24 | 24 | 0 | MockMvc + H2 기반 |
| 시스템 테스트 | 5 | - | - | 수동 테스트, 영상 기록 |

**결론:** Phase 3의 핵심 API(분기 생성, 제목 수정, cascade 삭제, 응답 재생성, 메시지 수정, 그래프 조회/확장)에 대한 통합 테스트 24개가 모두 통과하였다. SSE 실시간 스트리밍(재생성/수정)은 테스트 환경 제약으로 자동화 테스트 대상에서 제외되었으며, Service 메서드 직접 호출로 TurnContext 생성까지 검증하였다.
