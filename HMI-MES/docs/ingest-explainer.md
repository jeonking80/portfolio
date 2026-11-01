# 핵심 코드 해설 — 수집 트랜잭션

`apps/api/ingestion.py`의 `IngestionService.ingest`와 `_receive_event`에서 발췌했습니다. 고정 커밋 `8134cb8`, L976~L985, L779~L810입니다. 보관 기간·세대 검사와 저장 열 일부는 생략했습니다.

```python
# ingest(): 트랜잭션과 잠금 순서
with self._transaction() as connection:
    self._advisory(connection, "gateway", body.gateway_id)
    source = self._lock_stream(connection, body.source_id, body.stream_id)
    if source["gateway_id"] != body.gateway_id:
        raise SourceBindingError("Gateway is not explicitly registered")
    for event_id in sorted(event.event_id for event in body.events):
        self._advisory(connection, "event", event_id)

# _receive_event(): 사건 단위 중복 제거
existing = (
    connection.execute(
        sa.select(event_inbox).where(
            event_inbox.c.event_id == event.event_id,
        )
    )
    .mappings()
    .first()
)
if existing is not None:
    return self._receipt(existing)
payload = event.model_dump(mode="json")
connection.execute(
    event_inbox.insert().values(
        event_id=event.event_id,
        execution_id=event.execution_id,
        applied_at=None,
        reason_code="APPLICATION_PENDING",
    )
)
reason = self._apply_event(connection, event, now)
self._set_application(connection, event.event_id, reason, now)
return EventReceipt(
```

## 읽는 법

1. **잠금 순서를 고정합니다.**
   - 순서는 게이트웨이 → 라인·스트림 행 → 사건 ID입니다.
   - 사건 ID는 정렬해서 잠급니다. 두 묶음이 같은 사건들을 반대 순서로 잡아 서로 기다리는 교착을 막기 위해서입니다.
2. **이미 받은 사건이면 영수증만 돌려줍니다.** `event_inbox`에서 사건 ID로 원본을 찾으면 다시 반영하지 않습니다. 그때 저장한 결과를 그대로 돌려줍니다.
3. **원본을 먼저 저장하고 반영은 그다음에 합니다.**
   - 원본은 "반영 대기"(`APPLICATION_PENDING`)로 먼저 저장합니다.
   - 그다음 실적에 반영합니다.
   - 반영이 거절돼도 원본과 사유는 남습니다.
4. **실패는 격리로 기록합니다.** 반영할 수 없는 사건은 지우지 않습니다. 격리(`QUARANTINED`)로 표시해, 나중에 원래 실행으로 다시 처리할 수 있게 합니다.

## 이 함수 전체의 순서

| 단계 | 하는 일 | 실패하면 |
| --- | --- | --- |
| 1 | 토큰으로 확인한 게이트웨이·수집원·스트림이 묶음 본문과 같은지 확인 | 거절 |
| 2 | 트랜잭션 시작, 게이트웨이·스트림·사건 ID 잠금 | — |
| 3 | 같은 묶음 ID가 있으면 내용 해시 비교 | 같으면 원래 ACK 반환, 다르면 충돌 증거 저장 후 스트림 차단 |
| 4 | 사건별 수신(위 코드) | 격리 기록 |
| 5 | 연속 순번이면 처리 커서 전진, 빈 순번은 누락 구간 기록 | 누락은 정산 잠정 |
| 6 | ACK 생성·자체 검증, 묶음 원문 저장 | 전체 롤백 |
| 7 | 시각 근거·알람·트렌드 기록 | 전체 롤백 |
| 8 | 커밋 후 ACK 반환 | ACK 없음 → 게이트웨이가 같은 묶음 재전송 |

## 왜 이렇게 했나

- **한 트랜잭션으로 묶은 이유:** 중복 제거, 실적, 처리 커서가 따로 저장되면 문제가 생깁니다. 재시작 때 실적이 두 번 잡히거나, 커서만 앞서 나갑니다. 그래서 셋을 같이 커밋하거나 같이 버립니다.
- **재전송이 안전한 이유:** 응답이 끊겨 게이트웨이가 같은 묶음을 다시 보내도 3단계에서 원래 ACK를 돌려받습니다.
- **충돌을 숨기지 않는 이유:** 같은 ID에 다른 내용이 오면 덮어쓰지 않습니다. 증거를 남긴 채 그 스트림을 멈춥니다.
