---
name: musinsa-weekly-digest
description: musinsa-brief가 vault에 누적한 일간 보고서들을 주간 또는 월간 롤업 보고서로 합성합니다. vault 디렉토리 스캔 → 일간 보고서의 YAML frontmatter(report_date, mode, dominant_category, categories, sources, data_quality)만 파싱 → 카테고리 임팩트 분포·반복 테마(Trend Memory)·출처 매체 빈도·dominant 변화 추이 정리. 사용자가 "주간 무신사 정리", "이번 주 무신사 요약", "musinsa-weekly-digest", "이번 달 무신사", "월간 무신사", "지난 주 무신사 어땠어", "지난 달 무신사 정리" 같은 표현을 쓸 때 트리거하세요. 단일 일간 리포트는 musinsa-brief 사용, 이 skill은 **기간 집계**에만 사용합니다.
---

# Musinsa Weekly Digest

`musinsa-brief`가 vault에 누적한 일간 보고서를 **주간/월간**으로 합성. **본문 무시 frontmatter만 파싱**해서 가볍게 작동합니다.

## 왜 이 skill이 필요한가

매일 호출하다 보면 카테고리·테마·출처 패턴이 누적됩니다. 하나씩 보면 보이지 않던 흐름(예: "3주째 반복되는 글로벌 진출 보도", "공정위 이슈가 4일 연속 dominant")이 디지스트에서 드러납니다. 본문은 안 읽고 frontmatter `categories` + `sources`만 파싱하므로 어떤 기간이든 빠르게 합성 가능합니다.

## 참조 문서

- `references/trend-memory.md` — title 토큰화 + stopword + 빈도 카운팅 + top 10 선택 룰

## 출력 위치

`musinsa-brief`와 같은 vault. 파일명 규칙:
- 주간: `digest-{YYYY}-W{WW}.md` (ISO week number, e.g. `digest-2026-W17.md`)
- 월간: `digest-{YYYY}-{MM}.md` (e.g. `digest-2026-04.md`)

같은 파일 존재 시 `digest-2026-W17-{HHMM}.md` 시간 suffix.

## 실행 순서

### Step 0 — Preflight
이 skill은 외부 MCP를 호출하지 않으므로 별도 preflight 불필요. 단, `musinsa-brief`의 config.json을 공유하므로 출력 경로 결정 절차는 동일.

### Step 1 — 기간 결정

트리거 메시지에서 키워드 + 인자를 파싱:

| 트리거 패턴 | 기간 | 파일명 |
|------------|------|--------|
| "주간"·"weekly"·"지난 주"·"이번 주" + 명시적 기간 없음 | 7일 (today − 7 ~ today) | `digest-{YYYY}-W{ISO_WW}.md` |
| "월간"·"monthly"·"이번 달"·"지난 달" | 30일 또는 해당 월 1~말일 | `digest-{YYYY}-{MM}.md` |
| "지난 N일", "최근 N일" 명시 | N일 | `digest-{YYYY-MM-DD}-{N}d.md` |
| 인자 없음 default | 7일 | `digest-{YYYY}-W{ISO_WW}.md` |

`since_date`, `until_date` 변수 보존.

### Step 2 — 출력 경로 결정 + vault 스캔

1. `~/.claude/data/musinsa-brief/config.json`의 `output_dir` 확인 → 없으면 default `~/workspace/wooksang-marketplace-documents/musinsa-brief/`
2. 디렉토리 스캔으로 `YYYY-MM-DD*.md` 파일 목록 수집 (digest-*.md 제외)
3. 파일명에서 추출한 날짜가 `since_date ~ until_date` 범위 내인 것만 선택

### Step 3 — Frontmatter 파싱

선택된 파일들을 **frontmatter만** Read (Python yaml 또는 첫 `---` 블록만 추출). 본문은 무시. 합집합으로 모음:
- `report_dates` — 기간 내 실제 보고서 일자 set
- `categories_by_date` — `{report_date: [{id, label, impact, item_count}, ...]}`
- `sources_all` — 모든 source flat list (id 충돌은 (date, original_id) 키로 분리)
- `dominant_by_date` — `{report_date: dominant_category}`
- `tier_counts_total` — T1/T2/T3 합계
- `data_quality_warnings` — 모든 warning 합집합

결손 일자 = 기간 일수 − `report_dates.size`. 결손 일자 표기 (digest 본문에 명시).

### Step 4 — 합성

4.1. **카테고리 임팩트 분포**: 각 카테고리 id별로 `(high, mid, low) 카운트` + `item_count 누적합` 산정. 표 형태.

4.2. **Trend Memory**: `references/trend-memory.md` Read해 그 안의 토큰화·stopword·빈도 카운팅·top 10 룰 적용. `sources_all`의 `title` 필드 기반.

4.3. **출처 매체 빈도**: domain별 카운팅. 정규화 룰(source-tiers.md "매체 정규화" 섹션 참고하여 모바일/데스크톱 통합) 적용. Tier 함께 표기. 상위 10개 표.

4.4. **Dominant Category 변화**: `dominant_by_date`를 시간순 정렬 → 시퀀스 표기 + 가장 많이 등장한 dominant = 주간 dominant.

4.5. **다음 기간 관전 포인트**: `references/trend-memory.md`의 "다음 기간 관전 포인트 자동 추출" 룰 적용. 3~5개.

### Step 5 — 보고서 작성·저장

`{output_dir}/digest-*.md` 파일명에 따라 Write. 같은 파일 존재 시 시간 suffix.

### Step 6 — 사용자 보고

채팅에 **저장 경로 + 핵심 통계 + 주간 dominant + Trend Memory top 3**만 출력.

```
🛍️📊 Musinsa Weekly Digest 저장 완료
- 경로: {output_dir}/digest-{YYYY}-W{WW}.md
- 기간: {since_date} ~ {until_date} ({N}일, 보고서 {M}건 합성)
- 주간 dominant: {라벨} ({횟수}회)
- Trend Memory top 3:
  - {테마 1}: {N}/{M}일
  - {테마 2}: {N}/{M}일
  - {테마 3}: {N}/{M}일
```

## 출력 템플릿

```markdown
🛍️📊 **Musinsa Weekly Digest** — {YYYY} W{WW} ({since_date} ~ {until_date}, {N}일)

## 📅 기간 요약
- 일간 보고서 {M}건 합성 ({실제 일자 리스트} — 결손 {결손 일자 또는 "없음"})
- 총 신호 항목: {sum item_count}건

## 🚦 카테고리 임팩트 분포
| 카테고리 | High | Mid | Low | 누적 항목 |
|---|---|---|---|---|
| 자사 IR | {N} | {N} | {N} | {N} |
| 자사 캠페인 | ... |
| 입점·브랜드 | ... |
| 경쟁사 | ... |
| 규제·정책 | ... |
| 인사·조직 | ... |
| 업계 트렌드 | ... |

## 🔁 Trend Memory (반복 등장 테마)
- **{테마}**: {days_appeared}/{기간일수}일 등장 ({domains 매체 리스트})
- ... (top 10)

## 📰 핵심 출처 매체 빈도
| 매체 | Tier | 인용 횟수 |
|---|---|---|
| 매일경제 | T1 | {N} |
| ... |

## 🎯 Dominant Category 변화
- {YYYY-MM-DD}: {라벨} · {YYYY-MM-DD}: {라벨} · ...
- 주간 dominant: **{라벨}** ({횟수}회)

## 🧭 다음 기간 관전 포인트
- {trend-memory.md 룰로 자동 추출 3~5개}

---

_원본 보고서_: {M}건 (vault 경로 참조)
_데이터 품질_: T1 {N}건 · T2 {N}건 · T3 {N}건 · {warnings 요약 또는 "특이사항 없음"}
_생성_: {timestamp KST}
```

## 에러 처리

| 상황 | 동작 |
|------|------|
| vault 디렉토리 없음 | halt + "musinsa-brief를 먼저 호출해 일간 보고서를 누적하세요" 안내 |
| 기간 내 보고서 0건 | halt + "기간({since_date} ~ {until_date}) 내 일간 보고서 없음" 안내 |
| 일부 파일 frontmatter 파싱 실패 | 해당 파일 skip, 데이터 품질 푸터에 "파싱 실패 파일 N건" |
| Trend Memory 0개 | 해당 섹션에 "반복 테마 미감지 (보고서 수 부족 또는 매일 다른 이슈)" 명시 |
| 같은 digest 파일 존재 | `digest-*-{HHMM}.md` 시간 suffix |

## 금지 사항

- 일간 보고서 본문 Read 금지 (frontmatter만)
- 외부 MCP 호출 금지 (Tavily·WebSearch 모두 사용 안 함)
- 일간 보고서 수정·삭제 금지
- 단일 일간 리포트 생성 금지 (그건 `musinsa-brief` 담당)
