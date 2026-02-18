# Agent Card Generation Guide

## 개요
Agent가 API에서 환자 데이터를 수신한 후, Adaptive Card JSON을 직접 생성하여 Teams에 전송합니다.

---

## 1. 색상 매핑 규칙

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

---

## 2. AI Clinical Insight 생성 규칙

### 2.1 Agent B 출력 JSON 스키마

Agent B (Insight Generator)는 아래 스키마로 출력합니다.

```json
{
  "active_problems": [
    {
      "problem": "string — 문제명 (한글, 20자 이내)",
      "confidence": "HIGH | MED | LOW",
      "evidence": "string — 근거 요약 (수치 포함, 40자 이내)"
    }
  ],
  "risk_alerts": [
    {
      "risk": "string — 위험 항목명",
      "confidence": "HIGH | MED | LOW",
      "evidence": "string — 근거 요약",
      "priority": "number — 1~5 (1이 가장 긴급)"
    }
  ],
  "today_priorities": [
    {
      "action": "string — 조치 항목",
      "priority": "number — 1~5",
      "reason": "string — 근거 (15자 이내)"
    }
  ]
}
```

### 2.2 Confidence Level 판단 기준

Agent B는 아래 기준으로 confidence를 판단합니다.

#### HIGH (🔴)
- **3개 이상의 데이터**가 같은 임상적 방향을 가리킴
- 또는 **단일 수치가 critical range** 진입
  - Hb < 7.0, K > 6.5, Na < 120, SBP < 80, SpO₂ < 90%, BT ≥ 39.0
- 예: Hb↓↓ + Drain output↑ + Tachycardia = 3개 일치 → **HIGH**

#### MED (🟡)
- **1~2개 데이터**가 이상이나 확정적이지 않음
- 추가 검사/확인이 필요한 상태
- 예: BT 37.8 + WBC 경도 상승, 하지만 혈액배양 미시행 → **MED**

#### LOW (⚪)
- 경미한 이상, 안정적 추세
- 모니터링은 필요하나 즉시 조치 불필요
- 예: Na 133 안정적 유지 → **LOW**

### 2.3 Evidence 작성 규칙

- **수치는 추세 표기**: `Hb 13.2→11.8→9.6` (최근 3회)
- **단위 포함**: `450cc/8hr`, `SBP 88mmHg`
- **미시행 검사 명시**: `UA 미시행`, `혈액배양 미확인`
- **40자 이내**로 핵심만
- **구분자**: 항목 간 ` | ` 사용

```
✅ "Hb 13.2→11.8→9.6 | JP drain 450cc/8hr | PR 112"
✅ "BT 38.5℃ | CRP 23.3↑ | WBC 6.76 (정상)"
✅ "Cr 0.9→1.3 | UO 0.4cc/kg/hr (최근 6hr)"
❌ "환자의 헤모글로빈이 계속 떨어지고 있고 드레인에서 많이 나오고 있음" (너무 길고 수치 없음)
```

### 2.4 Today's Priorities 정렬 기준

Agent B는 아래 기준으로 priority 번호를 부여합니다.

#### Priority 1 — 생명 위협 / 즉각 조치
- 출혈, 패혈증, 호흡부전, 심부정맥, critical lab
- 예: "CBC f/u 및 수혈 준비"

#### Priority 2 — 당일 확인·조치 필요
- 추가 검사 오더, 약물 조정, 원인 감별 검사
- 예: "혈액배양 시행", "항생제 변경 검토"

#### Priority 3 — 일반 관리
- 드레인 관리, 식이 진행, 재활, 퇴원 준비
- 예: "조기 보행 격려", "식이 단계 진행"

#### 정렬 원칙
1. HIGH confidence 항목이 MED/LOW보다 먼저
2. 같은 confidence → 생명 위협 > 검사 필요 > 일반 관리
3. **최대 5개**, 초과 시 낮은 우선순위 제외
4. 각 항목 뒤에 `← 근거` 형태로 짧은 이유 표시

```
1. CBC f/u 및 수혈 준비 ← Hb 지속 하락
2. 혈액배양 시행 ← 발열 원인 감별
3. 수액 속도 재평가 ← 저혈압 반복, I/O 확인
```

---

## 3. Confidence → Card 색상 매핑

| Confidence | 이모지 | Card color  | 텍스트 weight |
|-----------|--------|-------------|-------------|
| HIGH      | 🔴     | `Attention` | `Bolder`    |
| MED       | 🟡     | `Warning`   | `Bolder`    |
| LOW       | ⚪     | (기본색)     | (기본)       |

### Card JSON 구조 (Active Problems / Risk Alerts 각 항목)

```json
{
    "type": "ColumnSet",
    "spacing": "Small",
    "columns": [
        {
            "type": "Column",
            "width": "auto",
            "items": [
                { "type": "TextBlock", "text": "🔴", "spacing": "None" }
            ]
        },
        {
            "type": "Column",
            "width": "stretch",
            "items": [
                {
                    "type": "TextBlock",
                    "text": "수술 후 출혈 의심",
                    "weight": "Bolder",
                    "color": "Attention",
                    "spacing": "None"
                },
                {
                    "type": "TextBlock",
                    "text": "Hb 13.2→11.8→9.6 | JP drain 450cc/8hr | PR 112",
                    "size": "Small",
                    "isSubtle": true,
                    "wrap": true,
                    "spacing": "None"
                }
            ]
        }
    ]
}
```

### Card JSON 구조 (Today's Priorities)

```json
[
    {
        "type": "TextBlock",
        "text": "1. CBC f/u 및 수혈 준비 ← Hb 지속 하락",
        "wrap": true,
        "spacing": "Small",
        "weight": "Bolder"
    },
    {
        "type": "TextBlock",
        "text": "2. 혈액배양 시행 ← 발열 원인 감별",
        "wrap": true,
        "spacing": "None"
    },
    {
        "type": "TextBlock",
        "text": "3. 수액 속도 재평가 ← 저혈압 반복, I/O 확인",
        "wrap": true,
        "spacing": "None"
    }
]
```

- Priority 1 항목만 `weight: "Bolder"` 적용
- Priority 2 이하는 기본 weight
- `← 근거` 부분이 한눈에 보이도록 action과 같은 줄에 표시

---

## 4. 카드 구조 (Summary Card)

```
1. Header: 환자명 (성별/나이) | HOD/POD
2. Dx / Op
3. 🤖 AI Clinical Insight        [style: accent]
   ├─ Active Problems    (🔴/🟡/⚪ + 문제명 + evidence)
   ├─ Risk Alerts        (🔴/🟡/⚪ + 위험명 + evidence)
   └─ Today's Priorities (번호 + 조치 ← 근거)
4. 📋 Latest Vitals               [style: emphasis, 색상 적용]
5. 💧 Fluid Balance + 🩹 Drains   [style: emphasis]
6. 🔬 Latest Labs                 [style: emphasis, 색상 적용]
7. 🚨 Last 24h Events             [style: accent]
8. Action Buttons (Vitals Trend / Lab Trends / Events & Orders)
```

---

## 5. Drains 처리 (동적)

API에서 드레인 목록이 배열로 올 수 있으므로, Agent가 문자열로 조합합니다.

```
API: [{"name": "L-tube", "amount": 800}, {"name": "H/V1", "amount": 200}]
→ "L-tube 800cc, H/V1 200cc"
```

드레인이 없으면: `"None"`

---

## 6. Action.Submit 버튼

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

---

## 7. API 데이터 → 카드 생성 흐름

```
1. API 호출 → 환자 데이터 JSON 수신
2. status → color 매핑
3. drains 배열 → 문자열 조합
4. Agent B에게 데이터 전달 → AI Insight JSON 수신
5. Insight의 confidence → 이모지 + color 매핑
6. Insight의 priority → 정렬 후 Today's Priorities 생성
7. 카드 JSON 조립 (01_summary_rendered.json 구조 참고)
8. Teams에 카드 전송
```
