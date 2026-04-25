````markdown
# Report Template — 카드형 보고서 템플릿

`musinsa-brief` Step 6에서 Read. mode별 diff 섹션은 Step 1에서 결정한 mode에 따라 정확히 하나만 포함.

## 섹션 순서 체크리스트

1. YAML frontmatter
2. 제목 라인
3. TL;DR (3~5줄)
4. 신호 카드 (7행 표 + dominant 라벨)
5. 카테고리 카드 7개 (빈 카테고리는 "특이사항 없음")
6. **mode별 diff 섹션** (정확히 하나)
   - `initial` → "## 📚 14일 누적 컨텍스트"
   - `refresh` → "## 🔁 오늘 신규 (vs 직전 보고서)"
   - `since_last` → "## 🔁 Since-last 변화 (since {since_date})"
7. 면책 조항
8. _Sources_ 블록 (normalize 후 1..N)
9. 데이터 품질 푸터

## 전체 템플릿

```markdown
---
report_date: {YYYY-MM-DD}
generated_at: {ISO 8601 KST}
mode: {initial|refresh|since_last}
since_date: {YYYY-MM-DD}
days_window: {N}
dominant_category: {category_id}
categories:
  - id: musinsa_ir
    label: 자사 IR/실적/M&A
    impact: {high|mid|low}
    item_count: {N}
  - id: musinsa_campaign
    label: 자사 캠페인/마케팅
    impact: {high|mid|low}
    item_count: {N}
  - id: musinsa_brands
    label: 입점·브랜드 변동
    impact: {high|mid|low}
    item_count: {N}
  - id: competitors
    label: 경쟁사 동향
    impact: {high|mid|low}
    item_count: {N}
  - id: regulation
    label: 규제·정책
    impact: {high|mid|low}
    item_count: {N}
  - id: org_hr
    label: 인사·조직
    impact: {high|mid|low}
    item_count: {N}
  - id: industry_trend
    label: 업계 트렌드
    impact: {high|mid|low}
    item_count: {N}
sources:
  - id: 1
    url: {URL}
    title: {기사 제목}
    domain: {domain}
    tier: {T1|T2|T3}
    category: {category_id}
data_quality:
  tier_counts: { T1: {N}, T2: {N}, T3: {N} }
  warnings: [{string list, [] if none}]
---

🛍️ **Musinsa Brief** — {YYYY-MM-DD} ({DayOfWeek}, KST {HH:MM} 수집) · since {since_date} · mode: {mode}

## TL;DR
- {핵심 요약 3~5줄, 가장 임팩트 큰 항목 위주, 인용은 [[n]](url)}

## 🚦 신호 카드
| 카테고리 | 임팩트 | 핵심 한 줄 |
|---|:---:|---|
| 자사 IR | {●|◐|○} | {요약 + [[n]]} 또는 "특이사항 없음" |
| 자사 캠페인 | {●|◐|○} | ... |
| 입점·브랜드 | {●|◐|○} | ... |
| 경쟁사 | {●|◐|○} | ... |
| 규제·정책 | {●|◐|○} | ... |
| 인사·조직 | {●|◐|○} | ... |
| 업계 트렌드 | {●|◐|○} | ... |

**오늘의 dominant**: {라벨} ({짧은 근거 1줄})

## 📂 카테고리 카드

### 1. 자사 IR/실적/M&A
- **[T1] {제목}** — {매체} [[n]](url)
  - 컨텍스트: {1~2줄, 추측은 "~로 보임" 명시}
- ... (카테고리당 최대 5개, 초과 시 "외 N건")
(또는 "특이사항 없음" — 빈 섹션 금지)

### 2. 자사 캠페인/마케팅
...

### 3. 입점·브랜드 변동
...

### 4. 경쟁사 동향
- **[T1] [에이블리]** {제목} — {매체} [[n]](url)
- **[T2] [Shein]** {제목} — {매체} [[n]](url)
- **[T1] [29CM]** (자회사) {제목} — {매체} [[n]](url)
... (사명 라벨 부착, competitors.md 룰 참조)

### 5. 규제·정책
...

### 6. 인사·조직
...

### 7. 업계 트렌드
...

{여기 mode별 diff 섹션 1개}

---

⚠️ 면책: 본 브리핑은 정보 제공 목적이며 특정 비즈니스·투자 결정의 근거가 아닙니다. 외부 보도된 사실만을 다루며, 사내 데이터는 포함하지 않습니다.

_Sources_:
[1] {제목} — {매체} (T{1|2|3}) — {URL}
[2] ...
(연속 번호 1..N, gap 금지)

_수집 도구_: Tavily / WebSearch · KST {HH:MM}
_데이터 품질_: T1 {N}건 · T2 {N}건 · T3 {N}건 · {warnings 요약 또는 "특이사항 없음"}
```

## mode별 diff 섹션 본문

### `initial` 모드

```markdown
## 📚 14일 누적 컨텍스트

지난 14일간(since {since_date}) 카테고리별 누적 흐름:

- **자사 IR**: {핵심 흐름 1~2줄}
- **자사 캠페인**: ...
- **경쟁사**: ...
- **규제·정책**: ...
- **업계 트렌드**: ...

(인사·조직, 입점은 신규 항목 위주로 본문 카드에서 다룸)
```

### `refresh` 모드 (같은 KST 날짜 재실행)

```markdown
## 🔁 오늘 신규 (vs 직전 보고서)

직전 보고서({YYYY-MM-DD-HHMM}) 이후 추가 캡처된 신규 항목만:

- **{카테고리}**: {신규 항목 한 줄 요약} [[n]]
- ...

(신규 0건이면 "추가 신규 항목 없음" 명시)
```

### `since_last` 모드 (이전 날짜 보고서 존재)

```markdown
## 🔁 Since-last 변화 (since {since_date})

직전 보고서({since_date}) 이후 {days_window}일간 분류:

**🆕 신규 이슈**
- **{카테고리}**: {신규 항목} [[n]]
- ...

**📈 진척**
- {지난 보고서에서 다룬 이슈가 진전된 항목, "지난번 X 발의 → 이번 Y 단계 진입"}
- ... (해당 없으면 "해당 없음")

**🏁 종료/결론**
- {지난 이슈가 해소되거나 결론에 이른 항목}
- ... (해당 없으면 "해당 없음")
```

## Executive Card 옵션 (트리거 키워드 포함 시)

트리거에 `슬랙용으로`, `한 줄로`, `executive`, `exec card` 포함 → 채팅에만 추가 출력 (저장 X):

```markdown
📋 **Musinsa Brief — Executive Card** ({YYYY-MM-DD})
1. {dominant 카테고리 핵심 한 줄}
2. {두 번째 임팩트 카테고리 핵심 한 줄}
3. {세 번째 또는 since-last 핵심 변화}
```

## 템플릿 제약 재확인

- 빈 섹션 금지 — "특이사항 없음" 명시
- TL;DR bullet 정확히 3~5개, 한줄평 중복 금지
- 카테고리당 항목 표시 cap 5개 (초과 시 "외 N건" 처리)
- 모든 인용에 `[[n]](url)` inline 출처 필수
- 추측은 "~로 보임" / "~라는 관측" 명시
- mode별 diff 섹션은 정확히 하나만 (Mode 결정 §6 참조)
- T3 카테고리당 cap 1~2, "신뢰도 낮음" 본문 명시
- `dominant_category`는 `null`/빈값 금지 (high가 0개여도 가장 item_count 많은 카테고리 fallback)
````
