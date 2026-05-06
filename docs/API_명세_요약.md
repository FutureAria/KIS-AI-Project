# API 명세 요약

## 결론

공개 저장소에는 실제 운영 API 코드와 서버 주소를 올리지 않습니다.

대신 프론트엔드가 어떤 데이터를 받아 화면을 구성하는지, 백엔드가 어떤 계약을 지켜야 하는지 이해할 수 있도록 공개 가능한 수준의 API 계약만 정리합니다.

## API 설계 원칙

- 조회 API와 관리자 API를 분리합니다.
- 공개 read-only 화면은 계좌, 토큰, 서버 내부 상태를 노출하지 않습니다.
- 실거래 주문 관련 API는 별도 승인 전까지 잠금 상태를 기본값으로 둡니다.
- 데이터 출처와 최신성을 응답에 포함합니다.
- 장중 값과 마감 확정 값을 구분합니다.
- 휴장일에는 새 손익 기록을 만들지 않고 직전 거래일 기준임을 표시합니다.

## 주요 조회 API

| API 그룹 | 목적 | 프론트 사용 위치 |
|---|---|---|
| Summary | 전체 상태 요약 | 요약 화면 |
| Paper Portfolio | 모의투자 보유/후보/손익 | 모의투자 화면 |
| AI Decision | 매수/대기/회피 판단 근거 | AI 판단 화면 |
| Symbols | 종목별 가격, 뉴스, 차트, 리스크 | 종목 정보 화면 |
| History | 일자별 손익과 누적 성과 | 히스토리 화면 |
| Ops Status | heartbeat, 수집 상태, 알림, 디스크 | 운영/관리 화면 |

## 예시 응답 구조

### Summary

```json
{
  "mode": "paper",
  "market_session": "closed",
  "valuation": {
    "is_final": true,
    "source": "paper_mark",
    "total_equity": 10115498,
    "total_pnl": 115498
  },
  "ai": {
    "decision": "wait",
    "reason": "시장 조건과 데이터 신뢰도 확인 필요",
    "confidence": 0.62
  },
  "risk": {
    "live_order_locked": true,
    "data_stale": false
  }
}
```

### History

```json
{
  "date": "2026-05-04",
  "trading_day": true,
  "is_final": true,
  "source_date": "2026-05-04",
  "fallback_reason": "",
  "cumulative_pnl": 115498,
  "daily_change": 138017
}
```

휴장일 또는 주말이면 `trading_day=false`로 표시하고, `source_date`에 직전 거래일을 넣습니다.

### AI Decision

```json
{
  "decision": "wait",
  "checks": [
    {
      "key": "market_regime",
      "status": "pass",
      "detail": "시장 급락 조건 아님"
    },
    {
      "key": "data_coverage",
      "status": "check",
      "detail": "공매도/대차 원천 확인 필요"
    }
  ],
  "order_policy": {
    "live_order_enabled": false,
    "requires_manual_approval": true
  }
}
```

## 백엔드 관점 체크

- Controller는 응답 조립만 담당하고 판단 로직은 별도 계층으로 분리합니다.
- 실주문 가능 여부는 단일 플래그가 아니라 여러 안전 조건의 결과로 계산합니다.
- API 응답에는 `source`, `collected_at`, `is_final`, `fallback_reason`을 포함합니다.
- 에러 응답에는 토큰, 계좌, 서버 경로를 넣지 않습니다.
- 관리자 API는 별도 인증/인가를 요구합니다.

## 프론트엔드 관점 체크

- 숫자만 보여주지 않고 기준을 함께 표시합니다.
- 장중/마감/휴장 상태를 색상과 문구로 구분합니다.
- AI 판단의 결론보다 “왜”를 먼저 보여줍니다.
- 데이터 부족 상태는 숨기지 않고 `확인 필요`로 노출합니다.
- 실거래 잠금 상태는 사용자가 한눈에 알 수 있게 표시합니다.

