# 데이터 모델

PostgreSQL 테이블 62개를 수명에 따라 네 묶음으로 나눴습니다. 아래는 업무 흐름의 중심이 되는 테이블입니다. 수치는 로컬 PostgreSQL에 마이그레이션 28개를 모두 적용해 센 값입니다.

## 네 묶음

| 묶음 | 관리 방식 | 주요 테이블 |
| --- | --- | --- |
| 기준정보 | 바꾸지 않고 새 버전을 만든다 | `sites`, `lines`, `assets`(설비 3대), `tag_definitions`(태그 48개), `item_master`(품목), `recipe_versions`(공정 설정 버전), `users`, `roles`(역할 4종) |
| 현재 상태 | 행 잠금 아래에서만 바뀐다 | `work_orders`, `work_order_execution_links`(작업지시 ↔ 실행 회차), `production_executions`, `execution_totals`, `command_requests`(명령 원장), `line_queue_entries`(작업 대기열), `stream_progress`(처리 커서) |
| 지울 수 없는 이력 | 트리거가 수정·삭제를 거절한다 | `item_audit`, `registration_receipts`, `work_order_cancellations`, `line_queue_audit`, `alarm_actions`, `downtime_reason_history`, `capture_clock_evidence` 등 21개 |
| 운영 기록 | 보관 기간이 지나면 압축·정리한다 | `event_inbox`(생산 사건 원본), `production_events`, `ingest_batches`(수집 묶음 원문), `stream_gaps`(누락 구간), `analog_history_raw`·`analog_history_minutes`(트렌드), `auth_sessions`, `auth_audit` |

## 주요 관계

```
sites ─< lines ─< assets ─< tag_definitions
item_master ─< recipe_versions ─< work_orders >─ lines
work_orders ─< work_order_execution_links >─ production_executions
production_executions ─< execution_totals
production_executions ─< command_requests
source_streams ─< event_inbox ─< production_events >─ production_executions
source_streams ─ stream_progress / stream_gaps
work_orders ─< line_queue_entries
```

`A ─< B`는 A 하나에 B 여러 개가 외래키로 연결된다는 뜻입니다.

## 불변식

- **생산 사건의 귀속**
  - 양품·불량·폐기 사건은 반드시 원래 `execution_id`를 가집니다.
  - 실적은 이 사건 수량의 합이며, 설비 카운터 값은 대조 근거로만 씁니다.
- **수량 보존:** 투입 = 양품 + 불량 + 폐기 + 현재 재공.
- **한 번만 반영**
  - `event_inbox`의 기본키는 `event_id`입니다.
  - 같은 사건이 다시 오면 원래 영수증만 돌려줍니다.
- **처리 커서**
  - 검증·반영이 끝난 연속 구간만 전진합니다.
  - 빈 순번은 `stream_gaps`에 기록하고, 관련 정산은 잠정으로 남깁니다.
- **대기열**
  - 같은 작업지시는 대기 중(WAITING)일 때 한 번만 들어갈 수 있습니다(부분 고유 인덱스).
  - 끝난 회차는 다시 넣을 수 있습니다.

## DB가 직접 지키는 것

| 장치 | 개수 | 용도 |
| --- | --- | --- |
| 트리거 | 48개 | 이력 테이블의 UPDATE·DELETE 거절, 수집 영수증 생성, 보관 정리 보호 |
| DB 함수 | 25개 | 트리거 함수, 보관 기간 정리·압축, 정규 JSON·해시 계산 |
| 권고 잠금 | — | 게이트웨이·스트림·사건 ID 단위 직렬화, 마이그레이션 동시 실행 방지 |

업무 트랜잭션은 저장 프로시저 대신 Python 코드(SQLAlchemy Core)에서 직접 제어합니다. DB 함수는 불변 이력 보호와 보관 정리에 씁니다.
