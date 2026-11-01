# 데이터 모델

PostgreSQL 테이블 41개를 SQL 마이그레이션 22개로 관리합니다. 모든 업무 테이블에 `tenant_id`가 있고, 회사 경계는 RLS(Row Level Security)가 강제합니다.

## 네 묶음

### 1. 기준정보 — 바뀔 수 있지만, 참조 중이면 비활성화할 수 없음

| 테이블 | 내용 |
| --- | --- |
| `tenants` | 회사, 공용 단말 자동 잠금 정책 |
| `warehouses` → `locations` | 창고, 위치(수령·보관·보류·포장·출하 유형) |
| `items` | 품목, 최소 재고 |
| `business_partners` → `delivery_destinations` | 거래처(공급처·고객), 배송지 |
| `master_barcodes` | 품목·위치 바코드. 대상과 포장 수량은 바꾸거나 재사용할 수 없음 |
| `user` · `session` · `memberships` | 계정, 세션, 회사별 역할(관리자·책임자·작업자·조회자) |
| `v2_warehouse_grants` | 사용자별 허가 창고 |
| `v2_machine_keys` · `v2_machine_warehouses` | 외부 연계 키(SHA-256 해시만 저장)와 허가 창고 |

### 2. 현재 상태 — 창고 단위로 잠가서 갱신

| 테이블 | 내용 |
| --- | --- |
| `v2_workflow_state` (헤더·객체 상태) | 창고별 입고예정·주문·포장·출하·실사·승인 상태 |
| `v2_workflow_stock` | 품목×위치별 재고 버킷 S(정상보관)·R(예약)·P(피킹)·Q(검수·적치)·B(보류) |
| `v2_workflow_objects` | 외부 업무번호와 내부 객체 연결 |
| `v2_integration_entities` | 외부 업무의 최신 버전 |

### 3. 지울 수 없는 이력 — UPDATE·DELETE를 트리거로 금지

| 테이블 | 내용 |
| --- | --- |
| `v2_workflow_ledger` | 모든 재고 증감 한 줄: 품목·위치·버킷·증감·명령·작업자·사유·시각 |
| `v2_workflow_requests` | request_id별 요청 해시와 응답. 재시도 시 같은 응답을 돌려주는 근거 |
| `v2_workflow_performances` | 작업 실적과 스캔 증거 |
| `v2_workflow_receivings` | 수령 단위 중복 방지 |
| `api_request_metrics` | 요청별 결과 코드와 처리 시간 |

### 4. 운영 기록

| 테이블 | 내용 |
| --- | --- |
| `v2_integration_events` | 외부 사건 원문, 의존성 대기·업무 거절·격리·재처리 상태 |
| `v2_identity_audit` | 계정·권한 변경 이력 |
| `master_csv_previews` · `master_csv_applies` | CSV 검증 미리보기와 적용 결과 |
| `v2_opening_imports` · `v2_cutovers` | 외부 개시재고 반입, 기존 창고 전환 |
| `rateLimit` | 로그인 시도 제한 |

## 핵심 관계와 불변식

- **원장 합계 = 현재고**: `v2_workflow_ledger`를 품목·위치·버킷별로 더한 값은 `v2_workflow_stock`과 같아야 합니다. 명령을 실행하기 전에 매번 대사하고, 다르면 실행하지 않습니다.
- **요청 1 : 실적 N**: 한 request_id로 만들어진 원장 행·작업 실적·요청 결과는 같은 트랜잭션에서 함께 저장됩니다.
- **재고 7지표**: 총실물 H = S + P + Q + B. 예약 R은 S 안에 포함되고(R ≤ S), 가용 A는 S − R에서 실사·점검 잠금 위치를 뺀 수량입니다.
- **참조 보호**: 재고·진행 중 업무가 참조하는 품목·위치·거래처는 비활성화를 트리거가 거절합니다.

## 보안 설정

- v2 테이블은 `FORCE ROW LEVEL SECURITY`로 테이블 소유자도 정책을 따릅니다.
- 앱 DB 계정(`wms_app`)은 `NOBYPASSRLS`이고 테이블 소유자가 아닙니다. 마이그레이션 계정(`wms_migrator`)과 분리했습니다.
- 트랜잭션마다 `set_config('app.tenant_id', …, true)`로 회사를 설정하고, 정책은 이 값과 같은 행만 보여 줍니다.
