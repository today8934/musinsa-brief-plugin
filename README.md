# Musinsa Brief Plugin

무신사 자사·경쟁사·입점·규제·인사·트렌드 7개 카테고리를 Tavily 병렬 수집해 **Obsidian vault에 한국어 마크다운 보고서**로 저장하고, 채팅엔 경로+TL;DR+신호 카드만 짧게 보고하는 Claude Code 플러그인.

> "무신사", "오늘 무신사", "무신사 IR", "musinsa-brief" 같은 표현으로 호출하면 자동 트리거됩니다.

## ✨ 주요 특징

- **7 카테고리 매트릭스** — 자사 IR/M&A · 자사 캠페인 · 입점·브랜드 · 경쟁사 · 규제·정책 · 인사·조직 · 업계 트렌드
- **`since_last_report` 모드** — `initial`/`refresh`/`since_last` 자동 분기. 며칠 만에 호출해도 그 사이 변화만 정확히 커버
- **소스 신뢰도 Tier** — T1(공식 IR + 메이저 일간) / T2(패션 전문지) / T3(블로그·X·블라인드, cap 1~2). T1 0건 카테고리는 임팩트 ● 자동 차단
- **카드형 출력 + 신호 카드** — 7행 임팩트 표(●/◐/○) + 카테고리 카드, dominant 라벨
- **Tavily 1개 의존, 무료 한도 30%** — 일간 10 쿼리/일 = 300/월. 무료 1,000 credits/월 안전 여유
- **Trend Memory in 주간 digest** — 일간 frontmatter `categories`+`sources`만 파싱. N≥2일 등장 테마 자동 추출
- **Inline preflight 24h 캐시** — 매번 재확인 없이 빠른 시작
- **저장 경로 configurable** — Obsidian vault default. `~/.claude/data/musinsa-brief/config.json`의 `output_dir`로 변경 가능
- **Sanity check 게이트** — 전 카테고리 0건 / 단일 매체 편중 / 중복 / T3 비중 / Tavily 빈 응답 5+
- **Executive Card 옵션** — "슬랙용으로", "한 줄로" 트리거에만 3줄 카드 추가 생성

## 🚀 설치

### 1. Plugin 설치 (Claude Code 세션에서)
```
/plugin marketplace add today8934/wooksang-marketplace
/plugin install musinsa-brief-plugin@wooksang-marketplace
```

### 2. Tavily MCP 등록 (1회)

Tavily 가입 (무료, 월 1,000 credits): https://tavily.com

```
claude mcp add tavily -e TAVILY_API_KEY=<your-key> -- npx -y tavily-mcp@latest
```

Claude Code 재시작.

### 3. 첫 호출

```
오늘 무신사 어땠어
```

또는

```
무신사 brief
```

자동으로 preflight → 7 카테고리 병렬 수집 → vault에 보고서 저장 → 채팅에 경로 + TL;DR + 신호 카드 보고.

기본 저장 경로: `~/workspace/wooksang-marketplace-documents/musinsa-brief/{YYYY-MM-DD}.md`. 다른 위치(예: Obsidian vault)에 저장하려면 `~/.claude/data/musinsa-brief/config.json`에 `{"output_dir": "<경로>"}` 작성.

## 🎯 사용법

### 일간 리포트 (`musinsa-brief`)

| 트리거 (예시) | 동작 |
|---------------|------|
| "오늘 무신사" / "무신사 brief" | 직전 보고서 이후 변화만 (`since_last`) 또는 14일 누적 (`initial`) |
| "무신사 다시 봐줘" | 같은 날 재호출 → `refresh` 모드로 신규 항목만 |
| "무신사 IR" | 7 카테고리 다 수집하되 IR 카테고리 우선 강조 (트리거 키워드 단서) |
| "무신사 슬랙용으로" | + Executive Card 3줄 추가 생성 |

### 주간/월간 롤업 (`musinsa-weekly-digest`)

| 트리거 | 기간 |
|--------|------|
| "주간 무신사 정리" / "이번 주 무신사" | 7일 |
| "월간 무신사" / "이번 달 무신사" | 30일 또는 해당 월 |
| "지난 N일 무신사" | N일 |

frontmatter만 파싱하므로 외부 MCP 호출 0회. 즉시 합성.

## 📄 산출물

### 일간 보고서 (`{YYYY-MM-DD}.md`)
1. YAML frontmatter (`report_date`, `mode`, `dominant_category`, `categories[]`, `sources[]`, `data_quality`)
2. 제목 + TL;DR (3~5 bullet)
3. 신호 카드 (7 카테고리 ●/◐/○ 표)
4. 카테고리 카드 7개 (빈 카테고리는 "특이사항 없음")
5. mode별 diff 섹션 (`initial`/`refresh`/`since_last` 중 하나)
6. 면책 + Sources + 데이터 품질 푸터

### 주간 디지스트 (`digest-{YYYY}-W{WW}.md`)
1. 기간 요약 (보고서 건수 + 결손 일자)
2. 카테고리 임팩트 분포 표
3. Trend Memory (반복 등장 테마 top 10)
4. 핵심 출처 매체 빈도
5. Dominant Category 변화 추이
6. 다음 기간 관전 포인트 (자동 추출)

## 🧠 데이터 소스 역할

| 소스 | 역할 | 비고 |
|------|------|------|
| `tavily_search` | 일간 7 카테고리 × 한국/글로벌 = 10 쿼리 병렬 | 무료 1,000 credits/월 |
| `WebSearch` (Claude 내장) | Tavily 5+ 카테고리 빈 응답 시 fallback | 무제한 |

## 🛠 문제 해결

### Tavily 미등록 / 401 에러
첫 호출 시 자동으로 Setup Wizard halt 메시지. 명령어 안내 그대로 따라 등록 후 재시작.

### Obsidian vault 경로가 다름
`~/.claude/data/musinsa-brief/config.json` 작성:
```json
{ "output_dir": "~/Documents/obsidian/Musinsa Vault/02-Briefs/musinsa" }
```

### 일간 보고서가 매번 같은 날짜로 덮어쓰기 됨
같은 날짜 파일이 있으면 자동으로 `{YYYY-MM-DD}-{HHMM}.md` 시간 suffix. 덮어쓰기 발생하면 버그 — issue 등록.

### 주간 digest가 "기간 내 보고서 0건"으로 종료
일간 보고서를 먼저 누적해야 합니다. `musinsa-brief`를 며칠 호출 후 다시 시도.

## 📝 라이선스
MIT License. 자세한 내용은 `LICENSE` 참조.

## ⚠️ 면책
이 플러그인의 산출물은 **정보 제공 목적**이며 비즈니스·투자 결정의 권유가 아닙니다. 외부 보도된 사실만 다루며, 사내 데이터는 포함하지 않습니다. Tavily 응답의 정확성·가용성은 해당 서비스 제공자에 의해 결정됩니다.

## 🙋 기여 & 문의
- GitHub: https://github.com/today8934/musinsa-brief-plugin
- Issues/PR 환영

## 📚 References
- Spec: `docs/superpowers/specs/2026-04-25-musinsa-brief-skill-design.md`
- Plan: `docs/superpowers/plans/2026-04-25-musinsa-brief-skill.md`
- Sibling plugins: `arsenal-brief-plugin`, `overnight-market-report-plugin`
