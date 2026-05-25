2025-05-25
컬럼 추가 (2건)
chat 테이블
sql`last_activity_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP

용도: 탐색기 트리 정렬용 캐시 컬럼
갱신 시점: 메시지 송신·재생성·수정·branch 생성·branch 복구·chat 제목 수정 시 해당 chat과 ancestor chain 전체를 now()로 갱신
초기값: chat 생성 시 created_at과 동일

usage_record 테이블
sql`compressed_turn_count` INT NOT NULL DEFAULT 0

용도: FR-5.2 압축(summary 치환 + truncation 합산)이 적용된 누적 turn 수
운영 관측용 (API 응답에는 노출 안 함)

인덱스 추가 (2건)
sqlCREATE INDEX idx_chat_member_last_activity ON chat(member_id, last_activity_at DESC);
CREATE INDEX idx_usage_member_period       ON usage_record(member_id, period_end DESC);

idx_chat_member_last_activity: 탐색기 트리 조회 시 root chat을 최근 활동순으로 정렬할 때 사용. NFR-P-2(3초) 충족용.
idx_usage_member_period: GET /usage/me에서 활성 기간 usage_record 조회용.

변경 없음

엔티티(테이블) 추가/삭제 없음
PK/FK 관계 변경 없음
기존 컬럼의 타입·NULL 정책 변경 없음
Phase 3에서 정의된 인덱스 5개는 그대로 유지

의미 명문화 (스키마 변경 없음, 명세서에서만 정의)
Phase 4 명세서에서 기존 컬럼의 의미를 명확히 했지만 ERD 자체는 그대로:

message.prompt_token = 압축 후 LLM에 실제 전달된 입력 토큰 수 (SSE contextTokens와 동일)
usage_record.tokens_used = message.prompt_token + message.answer_token 누적값