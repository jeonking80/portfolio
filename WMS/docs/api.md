# API

Next.js Route Handler 51개입니다. 모든 보호 API는 세션(또는 외부 연계 키)을 확인하고, 회사 멤버십·창고 허가·역할을 요청마다 DB에서 다시 읽습니다. 응답은 JSON이며 `Cache-Control: no-store`입니다.

## 현장 운영 (v2)

| 메서드 | 경로 | 용도 |
| --- | --- | --- |
| GET | `/api/v2/workflow?warehouse_id=` | 창고의 업무 상태 스냅샷(입고·주문·포장·재고·승인 등) |
| POST | `/api/v2/workflow` | 업무 명령 실행. 본문 `{warehouse_id, command, request_id, payload}` |
| GET | `/api/v2/workflow/ledger` | 원장 조회(품목·위치·기간 필터, 고정 처리 경계와 커서) |
| GET | `/api/v2/workflow/correction-source` | 재고 정정의 원본 실적 검색 |
| GET | `/api/v2/insights` | 운영 관제 지표(처리량·납기 위험·품절 예측용 집계) |
| POST | `/api/v2/assistant` | AI 운영 도우미(읽기 전용 도구만 사용) |
| GET·POST | `/api/v2/masters/[kind]` | 기준정보 목록(커서 페이지)·등록. kind = items·locations·partners·destinations·barcodes |
| PATCH | `/api/v2/masters/[kind]/[id]` | 기준정보 수정(버전 확인) |
| POST | `/api/v2/masters/[kind]/csv/preview` → `csv/apply` | CSV 검증 미리보기 후 정상 행 적용 |
| GET | `/api/v2/masters/[kind]/export` | 현재 필터 CSV 내보내기 |
| GET·POST | `/api/v2/integration/events` | 외부 사건 접수(기계 키)·조회 |
| POST | `/api/v2/integration/events/[id]/retry` · `/process` | 원래 사건 재처리, 대기 사건 일괄 처리 |
| GET·POST | `/api/v2/identity/users` 외 | 계정 초대·역할·창고 권한·연계 키 발급과 교체 |
| GET | `/api/v2/operations/monitor` | 운영 점검(DB·대사·연계 대기열·백업 등) |
| GET·POST | `/api/v2/operations/cutover` · `opening-import` | 기존 창고 전환, 외부 개시재고 반입 |

## 업무 명령 (40종)

`receipt.create · receive · inspect · putaway · close.request · cancel`, `order.create · reserve · pick · release · shortage · damage · cancel`, `package.create · unpack`, `shipment.create · delivery · delivery.correct`, `return.request · receive · inspect · putaway · close`, `stock.move · hold · inspect · putaway`, `hold.release`, `count.start · record · submit · unlock`, `overage.request`, `approval.decide`, `correction.request`, `task.assign`, `shift.close` 등.

## 멱등과 오류

- `request_id`가 같고 내용이 같으면 저장된 응답을 다시 돌려줍니다(`replayed: true`). 내용이 다르면 `409 IDEMPOTENCY_CONFLICT`입니다.
- 화면이 보던 버전과 다르면 `409 VERSION_CONFLICT`, 승인 대상이 바뀌었으면 `APPROVAL_STALE`입니다.
- 업무 규칙 위반은 `EXCEEDS_PENDING`, `INSUFFICIENT_STOCK`, `WRONG_LOCATION_SCAN`, `SELF_APPROVAL_FORBIDDEN` 같은 코드로 돌려주고, 화면은 한국어 안내로 바꿔 보여 줍니다.
- 권한 오류는 `401 UNAUTHENTICATED`, `403 MEMBERSHIP_REQUIRED · ROLE_FORBIDDEN · WAREHOUSE_FORBIDDEN`입니다. 내부 오류 내용은 응답에 넣지 않고 `503 SERVICE_UNAVAILABLE`과 요청 ID만 돌려줍니다.

## 기존 화면 API

`/api/items`, `/api/locations`, `/api/warehouses`, `/api/documents`(전표 작성·확정·취소·역분개), `/api/stock`, `/api/stock-ledger`, `/api/delivery-destinations`, `/api/dashboard`, `/api/audit-logs`. 상태 확인용 `/api/health`, `/api/ready`는 인증 없이 호출합니다.
