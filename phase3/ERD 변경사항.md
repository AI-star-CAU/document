## 1. ERD v2.1 변경분
Phase 3에서 ERD v2 대비 다음 변경분이 적용된다. 엔티티 관계는 그대로이며 컬럼·인덱스만 추가된다.

```sql
-- 컬럼명 정리 (이름이 직관적이지 않거나 케이스가 혼용된 컬럼을 통일)
-- chat의 PK와 turn의 FK 이름을 chat_id로 맞추고, turn의 Summary 컬럼은 소문자로.
ALTER TABLE chat
  CHANGE COLUMN aichat_session_id chat_id BIGINT NOT NULL;

ALTER TABLE turn
  CHANGE COLUMN aichat_session_id chat_id BIGINT NOT NULL,
  CHANGE COLUMN Summary summary TEXT NULL;

-- chat 테이블 보강
ALTER TABLE chat
  ADD COLUMN last_turn_id BIGINT NULL,
  ADD COLUMN title_status ENUM('PENDING', 'GENERATED', 'USER_EDITED')
    NOT NULL DEFAULT 'PENDING';

-- turn 요약 상태 정책 정리
-- summary는 요약 생성 전 상태를 표현하기 위해 NULL 허용 (위 ALTER로 이미 nullable).
-- 화면 기본 문구("제목없음", "요약 생성 중")는 클라이언트/DTO(데이터 전송 객체)에서 처리하고 DB에는 저장하지 않는다.

-- usage_record 테이블 보강 (이력 보존)
-- 신규 컬럼은 마이그레이션 호환성을 위해 NULL/DEFAULT 허용, 신규 row 생성 시에는 반드시 채움.
ALTER TABLE usage_record
  ADD COLUMN token_limit BIGINT NOT NULL DEFAULT 0,
  ADD COLUMN plan_id BIGINT NULL,
  ADD CONSTRAINT fk_usage_record_plan
    FOREIGN KEY (plan_id) REFERENCES plan(plan_id);

-- 인덱스 (NFR-P-2 보장)
CREATE INDEX idx_chat_root_deleted  ON chat(root_chat_id, deleted_at);
CREATE INDEX idx_chat_parent        ON chat(parent_id);
CREATE INDEX idx_chat_branch_point  ON chat(branch_point_turn_id);
CREATE INDEX idx_turn_chat_sequence ON turn(chat_id, turn_sequence);
CREATE INDEX idx_message_turn       ON message(turn_id);
```

| 변경 | 근거 |
|---|---|
| `chat.aichat_session_id` → `chat_id` | API·SQL·코드 어디서나 같은 이름으로 부르도록 통일. |
| `turn.aichat_session_id` → `chat_id` | 위와 동일. FK 컬럼명도 도메인 용어로 통일. |
| `turn.Summary` → `summary` | MySQL/ORM 매핑 일관성. 대문자/소문자 혼용 제거. |
| `chat.last_turn_id` | FR-6.2 정렬 최적화. 사이드바 미리보기는 last_turn의 summary 재사용 (별도 preview 컬럼 미도입). |
| `chat.title_status` | FR-2.2 + 비동기 자동 명명 도입에 따른 상태 구분. |
| `turn.summary` nullable 정책 | summary 값이 NULL이면 요약 생성 전(PENDING) 상태. DB 기본 문구 저장 금지. |
| `usage_record.token_limit` | 그 기간 시점의 토큰 한도 스냅샷. 플랜 정책 변경·사용자 플랜 변경 이력 보존. |
| `usage_record.plan_id` | 해당 기간에 적용된 플랜 추적. subscription 이력을 거치지 않고 직접 조회. |
| `idx_turn_chat_sequence` | 특정 chat에서 특정 순번 주변의 turn을 빠르게 찾기 위함. 그래프 조회의 핵심 인덱스. |
| 나머지 인덱스 | NFR-P-2 3초 이내 보장. |

> frontier 및 edge는 DB에 저장하지 않는다. 둘 다 조회 시점 계산값이다.

> `turn.summary`는 nullable로 관리한다. summary 값이 NULL이면 요약 생성 전 상태이며, API 응답의 `summaryStatus`는 `PENDING`이다. "제목없음" 같은 화면용 기본 문구는 클라이언트 또는 DTO 조립 단계에서만 사용하고 DB에는 저장하지 않는다.

> **Phase 2 코드와의 호환성:** Phase 2에서 이미 `aichat_session_id`나 `Summary`를 참조하는 코드가 있으면 이번에 같이 수정해야 한다. ORM 엔티티 클래스, Repository 메서드, 직접 작성한 SQL 모두 새 컬럼명으로 갱신할 것.
