# AIT Phase 4 테스트 보고서

## 1. 테스트 개요

| 항목 | 내용 |
|---|---|
| 프로젝트 | AIT (AI + Git 브랜치 채팅 서비스) |
| 단계 | Phase 4 탐색기 + 맥락 조립 + 사용량 추적 |
| 작성일 | 2026-05-27 |
| 테스트 프레임워크 | JUnit 5, Spring Boot Test, MockMvc |
| 테스트 DB | H2 In-Memory (ddl-auto: create-drop) |
| 테스트 실행 환경 | Java 21, Spring Boot 4.0.4, Gradle 9.3.1 |
| 테스트 클래스 | `Phase4ApiTest.java` |

## 2. 테스트 범위

### 대상 기능 (Phase 4 FR 기준)

| 기능 영역 | 테스트 대상 API / 기능 |
|---|---|
| 탐색기 페이지 (Explorer) | GET /chats/explorer |
| 단일 트리 새로고침 (Explorer) | GET /chats/explorer/{rootChatId} |
| 토큰 사용량 조회 (Usage) | GET /usage/me |
| 사용량 누적 (Usage) | UsageRecord.accumulate() |
| lastActivityAt 전파 | 분기 생성/제목 수정 시 ancestor chain 갱신 |
| turnCount 정확성 | Explorer 노드별 turn 수 집계 |

### 테스트 제외 항목

| 항목 | 사유 |
|---|---|
| 맥락 조립 (ContextAssembler) | AI 서버 스트리밍 과정에서 통합 호출되며, MockMvc 환경에서 SSE 수신 불가 |
| 압축 (Compression) | 맥락 조립에 포함된 내부 로직으로, 대량 토큰 시뮬레이션 필요 |
| SSE turn_completed 확장 필드 | 실시간 스트리밍 환경에서만 검증 가능 |
| accumulateUsage (MessageService 연동) | SSE 스트리밍 완료 시 호출되므로 자동화 테스트 대상 외 |

## 3. 단위 테스트

`UsageRecord.accumulate()` 메서드를 직접 호출하여 토큰 누적, 요청 횟수, 압축 턴 카운트 증가를 단위 수준으로 검증한다.

## 4. 통합 테스트

### 4.1 테스트 실행 결과 요약

| 항목 | 수치 |
|---|---|
| 총 테스트 수 | 25 |
| 성공 | 25 |
| 실패 | 0 |
| 건너뜀 | 0 |
| 총 소요 시간 | 2.197초 |

### 4.2 Explorer Page — 탐색기 트리 조회 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 1 | 기본 조회 | root chat 목록과 트리 구조 반환 | roots[0].nodes 포함, turnCount/depth 확인 | PASS | 0.111s |
| 2 | 분기 포함 트리 | root + branch 구조 | nodes 2개, depth 0/1, parentChatId 일치 | PASS | 0.094s |
| 3 | 빈 결과 | 대화가 없는 사용자 | roots=[], totalRootCount=0, hasNext=false | PASS | 0.069s |
| 4 | 페이지네이션 | size=1, root 2개 | roots 1개, totalRootCount=2, hasNext=true | PASS | 0.077s |
| 5 | name 정렬 | 제목 알파벳순 정렬 | AAA → BBB 순서 | PASS | 0.090s |
| 6 | 잘못된 size — 400 | size=51 (최대 50 초과) | 400 Bad Request | PASS | 0.078s |
| 7 | 잘못된 sort — 400 | sort=invalid | 400 Bad Request | PASS | 0.059s |
| 8 | 삭제된 chat 제외 | includeDeleted=false (기본값) | roots=[] (삭제된 root만 존재) | PASS | 0.085s |
| 9 | 삭제된 chat 포함 | includeDeleted=true | roots[0].nodes[0].deletedAt NOT NULL | PASS | 0.082s |

### 4.3 Explorer Tree — 단일 트리 새로고침 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 10 | 단일 트리 조회 성공 | root + branch 구조 | rootChatId 일치, nodes 2개, depth 0/1 | PASS | 0.070s |
| 11 | 존재하지 않는 rootChatId — 404 | 없는 ID 조회 | 404 Not Found | PASS | 0.065s |
| 12 | 다른 사용자의 chat — 403 | 비소유자 접근 | 403 Forbidden | PASS | 0.154s |
| 13 | root가 아닌 chat 조회 — 400 | branch chatId로 조회 | 400 Bad Request | PASS | 0.064s |

### 4.4 Usage — 토큰 사용량 조회 API

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 14 | 활성 record 조회 성공 | tokensUsed=10,000 / limit=500,000 | planName=Free, remainingTokens=490,000, warningLevel=NONE | PASS | 0.085s |
| 15 | usageRatio 계산 확인 | tokensUsed=250,000 (50%) | usageRatio=0.5, warningLevel=NONE | PASS | 0.069s |
| 16 | WARN 경고 | tokensUsed=400,000 (80%) | warningLevel=WARN | PASS | 0.073s |
| 17 | CRITICAL 경고 | tokensUsed=480,000 (96%) | warningLevel=CRITICAL | PASS | 0.061s |
| 18 | 만료된 record — 새 기간 자동 생성 | 지난달 record만 존재 | tokensUsed=0, 새 기간 생성, plan 계승 | PASS | 0.064s |
| 19 | record 없음 — 404 | UsageRecord 미존재 | 404 Not Found | PASS | 0.077s |

### 4.5 Accumulate — 사용량 누적 (단위)

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 20 | 토큰 누적 | prompt=100, answer=200, compressed=1 | tokensUsed=300, requestCount=1, compressedTurnCount=1 | PASS | 0.082s |
| 21 | 연속 누적 | 2회 호출 합산 | tokensUsed=500, requestCount=2, compressedTurnCount=2 | PASS | 0.109s |

### 4.6 lastActivityAt — 활동 시간 전파

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 22 | 분기 생성 시 ancestor chain 갱신 | branch 생성 후 root 확인 | root.lastActivityAt >= 생성 전 시간 | PASS | 0.140s |
| 23 | 제목 수정 시 ancestor chain 갱신 | branch 제목 수정 후 root 확인 | root.lastActivityAt >= 수정 전 시간 | PASS | 0.098s |

### 4.7 Explorer turnCount — 턴 수 정확성

| # | 테스트명 | 시나리오 | 기대 결과 | 실제 결과 | 소요 시간 |
|---|---|---|---|---|---|
| 24 | turn 3개인 chat | turn 3개 생성 후 explorer 조회 | turnCount=3 | PASS | 0.070s |
| 25 | turn 없는 chat | 빈 chat으로 explorer 조회 | turnCount=0 | PASS | 0.065s |

## 5. 시스템 테스트

프론트엔드-백엔드 연동 후 수동 테스트로 진행하며, 결과는 영상으로 기록하였다.

| # | 시나리오 | 검증 항목 | 결과 |
|---|---|---|---|
| 1 | 탐색기 페이지 렌더링 | 트리 구조 표시, 정렬 전환(recent/created/name) | 영상 참조 |
| 2 | 탐색기 페이지네이션 | 스크롤/페이지 전환 시 추가 root 로딩 | 영상 참조 |
| 3 | 삭제된 대화 토글 | includeDeleted 전환 시 삭제 chat 표시/숨김 | 영상 참조 |
| 4 | 토큰 사용량 표시 | usage/me 호출 → 사용량 바/경고 표시 | 영상 참조 |
| 5 | 긴 대화 맥락 압축 | 다수 turn 후 AI 응답에 맥락 유지 확인 | 영상 참조 |

## 6. 발견된 결함 및 조치 (개발 중 발견, 모두 조치 완료)

| # | 결함 | 심각도 | 원인 | 조치 | 상태 |
|---|---|---|---|---|---|
| 1 | UsageRecordRepository 빌드 실패 | 중 | `import java.util.List` 누락 | import 추가 | 완료 |
| 2 | ctxHolder 스코프 문제 | 중 | try/catch 블록 외부에서 ContextResult 참조 불가 | 배열 홀더(`ctxHolder[0]`) 패턴으로 해결 | 완료 |
| 3 | detached entity 오류 | 중 | TransactionTemplate 내부에서 가상 스레드 외부 엔티티 접근 | ID 기반 재조회로 해결 | 완료 |

## 7. 테스트 결과 종합

| 테스트 단계 | 총 케이스 | 성공 | 실패 | 비고 |
|---|---|---|---|---|
| 단위 테스트 | 2 | 2 | 0 | UsageRecord.accumulate() 직접 검증 |
| 통합 테스트 | 23 | 23 | 0 | MockMvc + H2 기반 |
| 시스템 테스트 | 5 | - | - | 수동 테스트, 영상 기록 |

**결론:** Phase 4의 핵심 API(탐색기 페이지/트리 조회, 토큰 사용량 조회)와 주요 로직(사용량 누적, lastActivityAt 전파, turnCount 집계)에 대한 테스트 25개가 모두 통과하였다. 맥락 조립/압축은 AI 서버 스트리밍과 통합된 내부 로직으로 자동화 테스트 대상에서 제외되었으며, 시스템 테스트(영상)로 보완하였다.
