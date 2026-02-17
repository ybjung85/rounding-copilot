# Agent Card Generation Guide

## 개요
Agent가 API에서 환자 데이터를 수신한 후, Adaptive Card JSON을 직접 생성하여 Teams에 전송합니다.

## 색상 매핑 규칙
API의 `status` 값을 Adaptive Card `color` 속성으로 변환합니다.

| API status | Card color   | 표시 색상 |
|-----------|-------------|----------|
| `high`    | `Attention` | 빨간색    |
| `low`     | `Accent`    | 파란색    |
| `normal`  | (생략)       | 기본색    |

### 적용 예시
```json
// API 응답: { "value": "9.6", "status": "low" }
// → Agent가 생성할 카드 JSON:
{ "type": "TextBlock", "text": "9.6", "weight": "Bolder", "color": "Accent" }

// API 응답: { "value": "23.3", "status": "high" }
// → Agent가 생성할 카드 JSON:
{ "type": "TextBlock", "text": "23.3", "weight": "Bolder", "color": "Attention" }

// API 응답: { "value": "218", "status": "normal" }
// → Agent가 생성할 카드 JSON:
{ "type": "TextBlock", "text": "218", "weight": "Bolder" }
// (color 속성 자체를 생략)
```

## 카드 구조 (Summary Card)

```
1. Header: 환자명 (성별/나이) | HOD/POD
2. Dx / Op
3. 🤖 AI Clinical Insight        [style: accent]
4. 📋 Latest Vitals               [style: emphasis, 색상 적용]
5. 💧 Fluid Balance + 🩹 Drains   [style: emphasis]
6. 🔬 Latest Labs                 [style: emphasis, 색상 적용]
7. 🚨 Last 24h Events             [style: accent]
8. Action Buttons (Vitals Trend / Lab Trends / Events & Orders)
```

## Drains 처리 (동적)
API에서 드레인 목록이 배열로 올 수 있으므로, Agent가 문자열로 조합합니다.

```
API: [{"name": "L-tube", "amount": 800}, {"name": "H/V1", "amount": 200}]
→ "L-tube 800cc, H/V1 200cc"
```

드레인이 없으면: `"None"`

## Action.Submit 버튼
Detail card 요청을 위한 버튼입니다. `mrn`을 포함하여 Agent가 다시 처리할 수 있게 합니다.

```json
{
    "type": "Action.Submit",
    "title": "📊 Vitals & Fluid Trend",
    "data": {
        "action": "showDetail",
        "card": "snapshot",
        "mrn": "7443817"
    }
}
```

Agent는 `action: showDetail`을 수신하면 해당 `card` 타입에 맞는 detail card JSON을 생성하여 응답합니다.

## API 데이터 → 카드 생성 흐름

```
1. API 호출 → 환자 데이터 JSON 수신
2. status → color 매핑
3. drains 배열 → 문자열 조합
4. 카드 JSON 조립 (01_summary_rendered.json 구조 참고)
5. Teams에 카드 전송
```
