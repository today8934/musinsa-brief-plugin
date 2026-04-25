# Musinsa Brief Plugin — Design Spec

**Date**: 2026-04-25
**Author**: 류욱상 (@today8934)
**Status**: Approved high-level design, pending implementation plan
**Plugin repo**: `today8934/musinsa-brief-plugin` (신규 생성 예정)
**Marketplace**: `today8934/wooksang-marketplace`
**Sibling references**: `arsenal-brief-plugin` v0.1.0 (단일 채팅 출력형) · `overnight-market-report-plugin` v0.4.0 (vault 저장 + 주간 digest 형)

---

## 1. Purpose

무신사 도메인을 둘러싼 **자사·경쟁사·입점 브랜드·규제·인사·업계 트렌드**를 7개 카테고리로 병렬 수집해 **Obsidian vault에 한국어 마크다운 보고서**로 저장하고, 채팅창에는 **경로 + TL;DR + 신호 카드**만 짧게 보고하는 Claude Code 스킬 플러그인.

대상 사용자: 무신사 직원 본인 1인 (외부 노출 정보만 다룸, 사내 데이터 미포함). 매일 5~10분 안에 회사·업계 흐름을 파악하고, 주간 단위로 누적된 흐름과 반복 테마를 트레이스하는 게 목적.

## 2. Scope

### In scope
- **자사**: IR/실적/M&A, 캠페인/마케팅, 입점·브랜드 변동
- **경쟁사**: 29CM(자회사 포함)·SSF샵·에이블리·지그재그·W컨셉·카카오스타일·Shein·Temu·Yoox·Farfetch
- **외부 환경**: 규제·정책(가품·표시광고·플랫폼법·전자상거래법), 인사·조직(외부 보도된 임원·구조조정·채용 트렌드), 업계 트렌드(D2C·라이브커머스·AI 패션·K-fashion 글로벌)
- 한국어 보고서 (마크다운, vault 저장)
- 일간 + 주간/월간 digest 두 skill
- `since_last_report` 모드 (마지막 보고서 이후 변화만 자동 분기)
- 출처 신뢰도 3-Tier 등급 (arsenal-brief 차용)
- 카테고리 카드 + 신호 카드 시각화

### Out of scope
- 사내 데이터 (위키·슬랙·JIRA·내부 대시보드) — 별도 인증 MCP 필요, 차후 v0.2+에서 검토
- 무신사 모회사 주가 분석 (비상장)
- 개별 직원 가십·평판
- 베팅/도박/주식 추천 등 재무 권유성 표현
- 입점 브랜드 단가·재고·매출 (영업기밀)
- 다른 한국 패션 플랫폼의 단독 종합 브리핑 (이번 스킬은 무신사 시점에서만)
- 실시간 모니터링 / push 알림

## 3. Plugin Structure

```
musinsa-brief-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── musinsa-brief/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── categories.md         # 7개 카테고리 정의 + 매체 우선순위
│   │       ├── source-tiers.md       # T1/T2/T3 분류 + 도메인 사전
│   │       ├── query-templates.md    # 카테고리별 Tavily/WebSearch 쿼리 템플릿
│   │       ├── competitors.md        # 경쟁사·자회사·해외 비교 사전
│   │       ├── setup-wizard.md       # Tavily 설치 안내
│   │       └── report-template.md    # 카드형 보고서 템플릿 전문
│   └── musinsa-weekly-digest/
│       ├── SKILL.md
│       └── references/
│           └── trend-memory.md       # 반복 테마 추출 룰
├── docs/
│   └── superpowers/
│       ├── specs/2026-04-25-musinsa-brief-skill-design.md
│       └── plans/2026-04-25-musinsa-brief-skill.md  (writing-plans 단계에서 생성)
├── README.md
├── LICENSE (MIT)
└── .gitignore
```

### 3.1 `plugin.json`

```json
{
  "name": "musinsa-brief-plugin",
  "version": "0.1.0",
  "description": "무신사 자사·경쟁사·입점 브랜드·규제·인사·업계 트렌드를 7개 카테고리로 병렬 수집해 Obsidian vault에 한국어 마크다운 보고서를 저장하고, 채팅엔 경로+TL;DR+신호 카드만 짧게 보고합니다. 일간 + 주간/월간 digest 두 skill 제공, since_last_report 모드로 변화 단위 자동 분기.",
  "author": {
    "name": "류욱상",
    "email": "wooksang.ryu@musinsa.com",
    "url": "https://github.com/today8934"
  },
  "license": "MIT",
  "keywords": [
    "musinsa", "fashion", "retail", "k-fashion",
    "news-brief", "korean", "tavily", "mcp",
    "weekly-digest", "category-matrix", "obsidian"
  ]
}
```

### 3.2 Marketplace 등록

`wooksang-marketplace/.claude-plugin/marketplace.json`에 항목 추가:

```json
{
  "name": "musinsa-brief-plugin",
  "source": { "source": "github", "repo": "today8934/musinsa-brief-plugin" },
  "description": "무신사 자사·경쟁사·입점·규제·인사·트렌드를 7개 카테고리로 병렬 수집해 Obsidian vault에 한국어 보고서로 저장",
  "version": "0.1.0"
}
```

## 4. 7 Category Matrix

각 카테고리는 (1) 키워드 사전, (2) 매체 우선순위, (3) Tavily `include_domains` 후보를 가짐. 자세한 키워드/도메인 리스트는 `references/categories.md` + `references/query-templates.md`로 분리.

| # | id | 라벨 | 핵심 키워드 (요약) | 매체 우선순위 |
|---|---|---|---|---|
| 1 | `musinsa_ir` | 자사 IR/실적/M&A | 거래액·매출·영업이익·투자·인수·IPO·상장 | 매경·한경·머투·조선비즈·BoF·Reuters |
| 2 | `musinsa_campaign` | 자사 캠페인/마케팅 | 무신사 스탠다드 캠페인·광고·기획전·콜라보·모델 | 패션비즈·어패럴뉴스·뉴데일리 |
| 3 | `musinsa_brands` | 입점·브랜드 변동 | 신규 입점·단독·컬래버·PB·브랜드 이탈 | 패션비즈·BoF·자사 PR |
| 4 | `competitors` | 경쟁사 동향 | 29CM·SSF샵·에이블리·지그재그·W컨셉·카카오스타일·Shein·Temu | 매경·패션비즈·BoF·WWD |
| 5 | `regulation` | 규제·정책 | 가품·표시광고·플랫폼법·전자상거래법·공정위 | 매경·디지털데일리·뉴데일리 |
| 6 | `org_hr` | 인사·조직 | 임원·인사발령·채용·감원·구조조정·노조 | 매경·블라인드·잡플래닛 |
| 7 | `industry_trend` | 업계 트렌드 | D2C·라이브커머스·AI 패션·친환경·K-fashion 글로벌 | BoF·WWD·VogueBusiness·트렌드인사이트 |

### 4.1 카테고리별 쿼리 호출 수

- 카테고리 1·4·7: 한국 1쿼리 + 글로벌 1쿼리 (= 2쿼리/카테고리)
- 카테고리 2·3·5·6: 한국 1쿼리만
- **하루 총 호출**: 7 + 3 = 10 쿼리/일 (Tavily)
- **월 누적**: 10 × 30 = 300 credits/월
- **무료 한도**: 1,000 credits/월 → 30% 사용, 안전 여유 확보

## 5. Source Tier 분류

`references/source-tiers.md`로 매체 도메인 사전을 분리해 관리. spec엔 정의만.

| Tier | 정의 | 한국 매체 예시 | 글로벌 매체 예시 |
|------|------|---------------|----------------|
| **T1** | 공식 IR + 메이저 일간/경제지 + 글로벌 신뢰 매체 | mk.co.kr · hankyung.com · mt.co.kr · biz.chosun.com · 자사 PR | reuters.com · bloomberg.com · businessoffashion.com · wsj.com |
| **T2** | 패션·리테일 전문지 | fashionbiz.co.kr · apparelnews.co.kr · fpost.co.kr · ktnews.com · ddaily.co.kr · ndaily.kr | wwd.com · voguebusiness.com · fashionunited.com · retaildive.com |
| **T3** | 블로그·X·커뮤니티 | brunch.co.kr · 네이버 블로그 · blind · jobplanet | x.com · twitter.com · personal blogs |

### 5.1 Tier 운영 룰
- 본문 인용 시 `[T1]`, `[T2]`, `[T3]` 명시
- T3은 카테고리당 cap 1~2개, "신뢰도 낮음" 명시
- 미분류 도메인은 default T2 (보수적)
- T1 출처가 0건인 카테고리는 임팩트 자동 cap = `mid` (●로 올라가지 않음)

## 6. Mode Paradigm — `since_last_report`

overnight v0.4.0의 `initial / intraday_refresh / next_day` 대신 무신사 도메인의 "저강도 일간" 특성에 맞춘 **변화 단위 분기**.

| 직전 보고서 상태 | mode | 검색 기간 (`days_window`) | since_date |
|---|---|---|---|
| 직전 보고서 없음 | `initial` | 14일 (default) | today − 14 |
| 직전 보고서의 `report_date` == today | `refresh` | 1일 | today |
| 직전 보고서의 `report_date` < today | `since_last` | today − last_report_date (max 30 cap) | last_report_date |

### 6.1 결정 절차
1. `{output_dir}` 스캔, `YYYY-MM-DD*.md` 파일 중 가장 최근 Read
2. 그 파일의 YAML `report_date` 추출
3. 위 표 기준으로 mode + since_date + days_window 결정
4. YAML frontmatter `mode`, `since_date`, `days_window`에 기록

### 6.2 mode별 보고서 차이
- `initial`: "## 🔁 Since-last 변화" 섹션 생략, "## 📚 14일 누적 컨텍스트" 섹션 추가
- `refresh`: 신규 항목만 강조, 기존 항목은 "(변동 없음)" 표시. 신호 카드 임팩트는 (직전 보고서 `categories[].item_count` + 이번 신규 항목 수)로 합산해 §7 Step 3 임팩트 룰 재적용 (T1/T2 비중도 합산본 기준)
- `since_last`: "## 🔁 Since-last 변화" 섹션에 신규/진척/종료 항목 분류

## 7. Execution Flow (`musinsa-brief` skill)

### Step 0 — Preflight (24h 캐시 우선)
- 캐시 파일: `~/.claude/data/musinsa-brief/preflight.json`
- 스키마: `{"last_ok_at": "2026-04-25T07:30:00+09:00", "checks": {"tavily": "ok"}}`
- `last_ok_at`이 24h 이내 + `tavily == "ok"` → **skip Step 1로**
- 캐시 miss / TTL 초과 → `references/setup-wizard.md` Read해 inline preflight:
  - `ToolSearch(query="select:mcp__tavily__tavily_search,mcp__tavily__tavily_extract", max_results=2)`
  - 실패 → 안내 메시지 출력 후 **즉시 halt** (메인 워크플로우 진입 금지)
  - 성공 → 캐시 갱신 후 Step 1

### Step 1 — 컨텍스트 결정
1. 오늘 한국 날짜·시각(KST) 확인 (예: `2026-04-25 Sat 08:30`)
2. 출력 경로 결정 (§9 참조)
3. 출력 디렉토리 없으면 `mkdir -p`
4. `{output_dir}` 스캔 → 최신 보고서 Read → mode 결정 (§6)
5. `report_date`, `mode`, `since_date`, `days_window` 산정

### Step 2 — 7개 카테고리 병렬 검색 (한 메시지 안에서 동시)

`references/query-templates.md`에서 카테고리별 쿼리 템플릿을 Read해 사용. 템플릿에 `{days_window}`, `{since_date}`, `{today}` 플레이스홀더 치환.

총 10개 `tavily_search` 호출:
| # | 카테고리 | 쿼리 (예) | days | include_domains |
|---|---|---|---|---|
| 1 | musinsa_ir (KR) | `"무신사 거래액 매출 영업이익"` | `{days_window}` | T1 한국 |
| 2 | musinsa_ir (EN) | `"Musinsa revenue funding IPO"` | `{days_window}` | T1 글로벌 |
| 3 | musinsa_campaign | `"무신사 스탠다드 캠페인 광고"` | `{days_window}` | T2 한국 |
| 4 | musinsa_brands | `"무신사 입점 단독 컬래버"` | `{days_window}` | T2 한국 + 자사 |
| 5 | competitors (KR) | `"29CM 에이블리 지그재그 SSF 무신사"` | `{days_window}` | T1+T2 한국 |
| 6 | competitors (EN) | `"Shein Temu Korea fashion platform"` | `{days_window}` | T1+T2 글로벌 |
| 7 | regulation | `"공정위 패션 플랫폼 표시광고 가품"` | `{days_window}` | T1+T2 한국 |
| 8 | org_hr | `"무신사 임원 인사 채용 노조"` | `{days_window}` | T1 한국 + T3 cap |
| 9 | industry_trend (KR) | `"K-fashion 글로벌 라이브커머스 D2C"` | `{days_window}` | T1+T2 한국 |
| 10 | industry_trend (EN) | `"K-fashion global livestream commerce AI"` | `{days_window}` | T1+T2 글로벌 |

각 호출 공통: `max_results=8`. **반드시 한 메시지 안에서 병렬**, 순차 호출 금지.

### Step 3 — 합성
1. **URL 정규화**: `utm_*`, `ref`, `?fbclid` 등 트래킹 파라미터 제거
2. **중복 제거**: 정규화 URL 일치 또는 제목 유사도 ≥ 0.9
3. **카테고리별 분류**: Step 2 호출 번호로 자동 분류
4. **Tier 등급 부여**: 도메인 사전(`references/source-tiers.md`)으로 매핑, 미분류 default T2
5. **임팩트 자동 산정**:
   - **● high**: 카테고리 내 T1 출처 ≥ 2 또는 매체 다양성 ≥ 4
   - **◐ mid**: T1 출처 1개 또는 T2 출처 ≥ 3
   - **○ low**: T2 ≤ 2 또는 T3만 또는 0건
6. **Dominant category 결정**: 임팩트 high인 카테고리 중 출처 다양성 최고. 동률 시 자사 우선 (`musinsa_ir` > `musinsa_campaign` > `musinsa_brands` > 그 외)
7. **T3 cap**: 카테고리당 T3 항목 최대 2개. 초과분 제거
8. **루머·추측 표현**: 비공식 인용은 본문에 "~로 보임", "~라는 관측" 명시

### Step 4 — Sanity Check 게이트

Step 6 저장 직전 자동 검증. 위반 시 보고서 상단 ⚠️ 배너 또는 데이터 품질 섹션에 한 줄 추가 (배너는 심각 위반에 한정).

| 검증 | 조건 | 액션 |
|------|------|------|
| 전면 0건 | 7 카테고리 모두 item_count = 0 | ⚠️ 배너 "검색 전면 실패 가능성 — Tavily 키/쿼리 점검 필요" |
| 단일 매체 편중 | 한 도메인 점유율 ≥ 70% | "출처 다양성 부족 (X 도메인 N%)" 한 줄 |
| 중복 미정리 | 제목 유사도 ≥ 0.9가 5건 이상 | "중복 정리 부족 가능성" 한 줄 |
| T3 비중 과도 | 전체 출처 중 T3 ≥ 30% | "신뢰도 낮은 출처 비중 높음" 한 줄 |
| Tavily 5+ 카테고리 빈 응답 | 5/7 카테고리 이상 빈 응답 | WebSearch fallback 자동 실행 (해당 카테고리만) + 데이터 품질에 명시 |

### Step 5 — 출처 번호 normalize + YAML 기록

**저장 직전 필수**:
- 본문의 `[[n]](url)` 패턴 등장 순서로 스캔
- 등장 순서대로 `1..N` 재할당, 같은 URL은 같은 번호
- 미사용 URL은 `_Sources_` 블록과 YAML `sources`에서 모두 제외
- 결과: `[[1]]..[[N]]` 연속 번호, 누락 금지
- YAML `sources` 배열에 `{id, url, title, domain, tier, category}` 동시 기록

### Step 6 — 보고서 작성·저장
- 파일명: `{output_dir}/YYYY-MM-DD.md`
- 같은 날짜 파일 존재 시 `YYYY-MM-DD-HHMM.md`로 시간 suffix (덮어쓰기 금지)
- 템플릿: `references/report-template.md` Read해 사용 (§10 참조)

### Step 7 — Preflight 캐시 갱신
- 전체 실행 무결 완료 시 `preflight.json`의 `last_ok_at`을 현재 KST ISO8601로 갱신

### Step 8 — 사용자 보고
채팅창에 **저장 경로 + TL;DR + 신호 카드**만 출력. 본문 재첨부 금지.

```
🛍️ Musinsa Brief 저장 완료
- 경로: ~/Documents/obsidian/musinsa-brief/2026-04-25.md
- mode: since_last (since 2026-04-22, 3일 윈도우)
- TL;DR: (3~5줄)
- 신호 카드: (7개 카테고리 ●◐○ 표 1개)
```

### Step 9 (옵션) — Executive Card
트리거에 `"슬랙용으로"`, `"한 줄로"`, `"executive"`, `"exec card"` 포함 시:
- 3줄 슬랙 카드 추가 생성 (TL;DR을 압축한 보고용)
- 채팅에만 출력, 별도 저장 X

## 8. YAML Frontmatter 스키마

```yaml
---
report_date: 2026-04-25            # KST 보고서 생성 날짜
generated_at: 2026-04-25T08:30:00+09:00
mode: since_last                   # initial | refresh | since_last
since_date: 2026-04-22             # 검색 기준 시작일 (initial일 땐 today - 14)
days_window: 3                     # since_date ~ report_date 일수
dominant_category: competitors     # 7개 카테고리 id 중 1
categories:
  - id: musinsa_ir
    label: 자사 IR/실적/M&A
    impact: high                   # high | mid | low
    item_count: 2
  - id: musinsa_campaign
    label: 자사 캠페인/마케팅
    impact: mid
    item_count: 1
  - id: musinsa_brands
    label: 입점·브랜드 변동
    impact: low
    item_count: 0
  - id: competitors
    label: 경쟁사 동향
    impact: high
    item_count: 3
  - id: regulation
    label: 규제·정책
    impact: mid
    item_count: 2
  - id: org_hr
    label: 인사·조직
    impact: low
    item_count: 0
  - id: industry_trend
    label: 업계 트렌드
    impact: high
    item_count: 2
sources:
  - id: 1
    url: https://hankyung.com/article/...
    title: 무신사 1Q 거래액 ...
    domain: hankyung.com
    tier: T1
    category: musinsa_ir
  - id: 2
    ...
data_quality:
  tier_counts: { T1: 4, T2: 5, T3: 1 }
  warnings: []                     # sanity check 한 줄 모음
---
```

이 frontmatter 구조는 주간 digest skill이 본문 없이 파싱할 수 있도록 설계.

## 9. 저장 경로 + Config

### 9.1 결정 절차
1. `~/.claude/data/musinsa-brief/config.json`이 있고 `output_dir` 필드가 존재 → 우선 사용
2. 없으면 default: `~/Documents/obsidian/musinsa-brief/`
3. 디렉토리 없으면 `mkdir -p`. **default를 처음 사용한 경우(config.json 없음 + 디렉토리 신규 생성)** 채팅 보고에 한 줄 안내 추가:
   `"💡 Obsidian vault 경로가 다르면 ~/.claude/data/musinsa-brief/config.json 생성 후 {\"output_dir\": \"<경로>\"} 작성하세요. 안내는 첫 호출에만 노출."`
   안내 1회 노출 후 `~/.claude/data/musinsa-brief/.welcomed`(빈 파일) touch해 재노출 방지
4. 같은 날짜 파일 존재 시 `YYYY-MM-DD-HHMM.md` 시간 suffix (refresh mode)

### 9.2 config.json 예시
```json
{
  "output_dir": "~/Documents/obsidian/Musinsa Vault/02-Briefs/musinsa"
}
```

사용자가 "출력 경로 바꿔줘" / "저장 위치를 X로" 같이 요청하면 이 파일을 Write해 설정.

### 9.3 Default 결정 근거
`overnight-market-report-plugin`이 `~/workspace/overnight-market-report/`를 default로 쓰는 것과 달리, 이번 brief는 brainstorming에서 사용자가 **Obsidian vault 안 저장**을 명시 선택. 정확한 vault 경로는 첫 호출 시 안내 또는 사용자가 미리 config 작성하면 그걸 우선.

## 10. Output Template (보고서 마크다운)

전체 템플릿은 `references/report-template.md`로 분리 관리. spec엔 골격만.

```markdown
🛍️ **Musinsa Brief** — {report_date} ({DayOfWeek}, KST {HH:MM} 수집) · since {since_date} · mode: {mode}

## TL;DR
- (3~5줄, 가장 임팩트 큰 항목 위주, 인용은 [[n]](url))

## 🚦 신호 카드
| 카테고리 | 임팩트 | 핵심 한 줄 |
|---|:---:|---|
| 자사 IR | ● | ... [[1]] |
| 자사 캠페인 | ◐ | ... [[3]] |
| 입점·브랜드 | ○ | 특이사항 없음 |
| 경쟁사 | ● | ... [[5]] |
| 규제·정책 | ◐ | ... [[7]] |
| 인사·조직 | ○ | 특이사항 없음 |
| 업계 트렌드 | ● | ... [[9]] |

**오늘의 dominant**: {라벨} ({짧은 근거})

## 📂 카테고리 카드

### 1. 자사 IR/실적/M&A
- **[T1] {제목}** — {매체} [[n]](url)
  - 컨텍스트: {1~2줄}
- ...
(또는 "특이사항 없음" — 빈 섹션 금지)

### 2. 자사 캠페인/마케팅
...

### 3. 입점·브랜드 변동
...

### 4. 경쟁사 동향
...

### 5. 규제·정책
...

### 6. 인사·조직
...

### 7. 업계 트렌드
...

{mode별 diff 섹션}
- initial: "## 📚 14일 누적 컨텍스트"
- refresh: "## 🔁 오늘 신규 (vs 직전 보고서)"
- since_last: "## 🔁 Since-last 변화 (since {since_date})"

---

⚠️ 면책: 본 브리핑은 정보 제공 목적이며 특정 비즈니스 결정의 근거가 아닙니다.

_Sources_:
[1] {제목} — {매체} (T1) — {URL}
[2] ...

_수집 도구_: Tavily / WebSearch · KST {HH:MM}
_데이터 품질_: T1 {n}건 · T2 {n}건 · T3 {n}건 · {warnings 요약}
```

### 10.1 템플릿 제약
- 빈 섹션 금지 — "특이사항 없음" 명시
- TL;DR bullet 정확히 3~5개, 한줄평 중복 금지
- 카테고리당 항목 표시 cap 5개 (초과 시 "외 N건" 처리)
- 모든 인용에 `[[n]](url)` inline 출처 필수, 추측은 "~로 보임" 명시
- mode별 diff 섹션은 §6 룰에 따라 정확히 하나만

## 11. `musinsa-weekly-digest` Skill

### 11.1 Frontmatter
```yaml
---
name: musinsa-weekly-digest
description: musinsa-brief 일간 보고서를 주간/월간으로 합성. vault 디렉토리 스캔 → frontmatter만 파싱 → 카테고리 임팩트 분포·반복 테마(trend memory)·출처 매체 빈도·dominant 변화 추이를 정리. "주간 무신사 정리", "이번 주 무신사", "월간 무신사", "musinsa-weekly-digest", "이번 달 무신사", "지난 주 무신사 어땠어" 등으로 트리거.
---
```

### 11.2 실행 흐름
1. 트리거 키워드/인자 파싱 → 기간 결정
   - "주간"·"weekly"·"지난 주" → 7일
   - "월간"·"monthly"·"이번 달"·"지난 달" → 30일
   - default 7일
2. config의 `output_dir` Read → vault 디렉토리 스캔
3. 기간 내 일간 보고서 파일 (`YYYY-MM-DD*.md`) 모두 Read **frontmatter만** (본문 무시 → 가벼움)
4. 합성:
   - **카테고리 누적 임팩트 분포** (high/mid/low 카운트 + item_count 합)
   - **Trend Memory**: `sources` 배열의 `title` 키워드 토큰화 → 빈도 카운팅 → N≥2일 등장 테마 추출
   - **출처 매체 빈도**: domain별 인용 횟수 정렬
   - **Tier 분포**: T1/T2/T3 비중
   - **Dominant category 변화**: 일자별 dominant 시퀀스 + 주간 dominant
5. 출력 저장: `{output_dir}/digest-YYYY-WW.md` (주간) 또는 `digest-YYYY-MM.md` (월간)

### 11.3 출력 템플릿 (digest)
```markdown
🛍️📊 **Musinsa Weekly Digest** — {YYYY} W{WW} ({since} ~ {until}, {N}일)

## 📅 기간 요약
- 일간 보고서 {N}건 합성 ({날짜 리스트} — {결손 일자})
- 총 신호 항목: {sum item_count}건

## 🚦 카테고리 임팩트 분포
| 카테고리 | High | Mid | Low | 누적 항목 |
|---|---|---|---|---|
| ... |

## 🔁 Trend Memory (반복 등장 테마)
- **{테마}**: {N}/{기간일수}일 등장 ({매체 리스트})
- ...

## 📰 핵심 출처 매체 빈도
| 매체 | Tier | 인용 횟수 |
|---|---|---|
| ... |

## 🎯 Dominant Category 변화
- {일자별 dominant 시퀀스}
- 주간 dominant: **{라벨}** ({횟수}회)

## 🧭 다음 기간 관전 포인트
- (Trend Memory + 진행 중 진척 항목 기반 자동 추출 3~5개)

---

_원본 보고서_: {N}건 (vault 경로 참조)
_생성_: {timestamp KST}
```

### 11.4 Trend Memory 룰 (`references/trend-memory.md`)
- title 토큰화: 한국어/영어 명사 추출 (간단한 stopword 필터)
- 카테고리 라벨·매체명·일반어("브랜드", "발표" 등) 제외
- N≥2일 등장한 토큰만 채택, 빈도 정렬 top 10
- 각 테마의 출현 일자 + 인용 매체 리스트 동반

## 12. Trigger 키워드 (description)

`musinsa-brief` SKILL.md frontmatter `description`에 들어갈 트리거 키워드 (광범위 트리거는 의도적으로 제외):

```
무신사 자사·경쟁사·입점·규제·인사·트렌드 7개 카테고리를 Tavily 병렬 수집해 한국어 마크다운 보고서를 Obsidian vault에 저장하고 채팅엔 경로+TL;DR+신호 카드만 보고합니다. 사용자가 "무신사", "musinsa-brief", "무신사 소식", "무신사 뉴스", "무신사 근황", "무신사 어떻게 돼", "무신사 어땠어", "오늘 무신사", "어제 무신사", "무신사 brief", "무신사 IR", "무신사 인사", "무신사 동향", "무신사 정리해줘", "무신사 업데이트", "무신사 업계", "무신사 경쟁사" 같은 직·간접 표현을 쓸 때 반드시 이 skill을 실행하세요. 단순 회사명 조회가 아니라 자사·경쟁사·입점·규제·인사·트렌드를 카테고리별로 묶어 정리한 종합 브리핑이 필요한 모든 상황에서 트리거합니다.
```

**의도적 제외**: "회사 소식" / "사내 동향" / "패션 업계" 같이 무신사를 명시하지 않는 광범위 표현은 다른 컨텍스트와 충돌 위험. v0.2+에서 사용자 패턴 보고 추가 검토.

## 13. Setup Wizard (Tavily 단일)

`references/setup-wizard.md`에 보관, Step 0에서 Read해 사용.

```
Musinsa Brief를 실행하려면 Tavily MCP가 등록돼 있어야 합니다.
본 스킬은 Tavily 외 별도 키가 필요 없습니다.

1. Tavily 가입 (무료, 월 1,000 credits): https://tavily.com
2. API 키 발급 후 아래 명령으로 등록:

   claude mcp add tavily -e TAVILY_API_KEY=<your-key> -- npx -y tavily-mcp@latest

3. 등록 후 Claude Code 재시작 → 다시 호출
```

## 14. Error Handling

| 상황 | 동작 |
|------|------|
| Tavily 미등록 (Step 0 실패) | halt + setup-wizard 안내 |
| 일부 카테고리 쿼리 실패 (1~4건) | 해당 카테고리는 "데이터 없음 (쿼리 실패)" 표시, 나머지 진행. 데이터 품질 섹션에 실패 목록 |
| Tavily 5+ 카테고리 빈 응답 | WebSearch fallback 자동 실행 (해당 카테고리만), 데이터 품질에 명시 |
| 모든 카테고리 0건 (Sanity #1) | ⚠️ 배너 + 데이터 품질에 점검 안내 |
| 출력 디렉토리 생성 실패 (권한 등) | halt + 경로 변경 안내 ("config.json의 output_dir 점검") |
| 같은 날짜 파일 존재 | `YYYY-MM-DD-HHMM.md`로 시간 suffix (refresh 자연스럽게 작동) |
| 직전 보고서 frontmatter 파싱 실패 | mode = `initial`로 fallback, 데이터 품질에 한 줄 |
| Tavily rate limit (429) | 3초 대기 후 1회 재시도, 그래도 실패면 해당 카테고리 단위 skip + 데이터 품질에 명시 |

## 15. Success Criteria

- 한 번 호출로 무신사 도메인의 7개 면을 5~10분 안에 파악 가능
- `since_last_report` 모드 덕분에 며칠 만에 호출해도 그 사이 변화만 정확히 커버
- T1/T2/T3 등급 + dominant 라벨로 신뢰도와 우선순위가 한눈에 보임
- 빈 카테고리도 "특이사항 없음"으로 명시 (빈 섹션 금지)
- 채팅 출력은 경로 + TL;DR + 신호 카드 7행으로 끝 (본문 재첨부 없음)
- 매일 호출해도 Tavily 무료 한도(1,000/월)의 30% 이내
- 주간 digest가 본문을 안 읽고 frontmatter만으로 trend memory 추출 가능
- 일간/주간 두 skill이 같은 vault 디렉토리에서 자연스럽게 협업

## 16. Non-Goals / Explicit Rejections

- **사내 데이터 호출 금지** (위키·슬랙·JIRA — v0.2+에서 별도 검토)
- **주가/시세 호출 금지** (무신사·모회사 비상장)
- **재무 권유성 표현 금지** (투자·매수·매도 추천)
- **T3 출처 카테고리당 3개 이상 금지**
- **본문 채팅 재첨부 금지** (경로 + TL;DR + 신호 카드만)
- **추측을 사실처럼 쓰지 말 것** (`~로 보임` 명시)
- **베팅·도박·가십성 자극 표현 금지**
- **개별 직원 가십·평판 금지** (외부 보도된 임원 인사·조직 변동만)
- **다른 패션 플랫폼의 단독 종합 브리핑 금지** (이번 스킬은 무신사 시점)
- **readability-pass 자동 생성 금지** (본인 전용, 입문자 리라이팅 불필요)
- **`wooksang-marketplace` 레포에 직접 스킬 두지 않음** (인덱스 전용 원칙)
- **Context7·Serena 등 코드 도메인 MCP 사용 금지** (도메인 부적합)

## 17. 비용/한도 분석

| 항목 | 일간 | 월간 (30일) | 무료 한도 | 사용률 |
|------|------|------------|----------|--------|
| Tavily search 호출 | 10 | 300 | 1,000 credits/월 | 30% |
| WebSearch (Claude 내장) | 0~5 (fallback) | 0~150 | 무제한 | - |
| 디스크 저장 (보고서 1건) | ~10KB | ~300KB | - | - |
| 주간 digest 추가 호출 | - | 0 (frontmatter만 Read) | - | - |

**결론**: 매일 + 주 1회 digest 운영해도 Tavily 무료 한도 30% 사용. 안전 여유 70% 확보.

## 18. Open Questions / Future Work

- **v0.2** — 사내 MCP (위키·슬랙) 통합 시 카테고리 추가 (`internal_announcements`)
- **v0.2** — Trend Memory의 키워드 토큰화 정밀도 향상 (한국어 명사 추출 라이브러리 옵션)
- **v0.2** — 카테고리 개인화 (첫 호출 시 관심 카테고리 학습 → config 저장 → 비중 ↑)
- **v0.2** — Executive card 트리거를 `mode` 인자로 명시 처리 (`exec` mode)
- **v0.2** — 광범위 트리거 키워드 ("회사 소식" 등) 도입 검토 (사용자 사용 패턴 분석 후)
- **v0.3** — 동일 구조로 다른 한국 플랫폼 (29CM·에이블리 등) 버전 템플릿화
- **v0.3** — Slack 직접 게시 옵션 (Slack MCP 통합 시)
- **v0.3** — RAG 기반 vault 누적 검색 (`Obsidian` MCP 통합 시)

## 19. References

- `arsenal-brief-plugin` v0.1.0 — 단일 채팅 출력형 패턴 차용 (Tier 신뢰도, 병렬 수집, 빈 섹션 금지)
- `overnight-market-report-plugin` v0.4.0 — vault 저장 + 주간 digest 패턴 차용 (frontmatter 스키마, 24h preflight 캐시, configurable output_dir, 출처 normalize)
- `wooksang-marketplace` — plugin 등록 인덱스
