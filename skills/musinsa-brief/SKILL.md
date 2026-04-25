---
name: musinsa-brief
description: 무신사 자사·경쟁사·입점·규제·인사·트렌드 7개 카테고리를 Tavily 병렬 수집해 한국어 마크다운 보고서를 Obsidian vault에 저장하고 채팅엔 경로+TL;DR+신호 카드만 보고합니다. 사용자가 "무신사", "musinsa-brief", "무신사 소식", "무신사 뉴스", "무신사 근황", "무신사 어떻게 돼", "무신사 어땠어", "오늘 무신사", "어제 무신사", "무신사 brief", "무신사 IR", "무신사 인사", "무신사 동향", "무신사 정리해줘", "무신사 업데이트", "무신사 업계", "무신사 경쟁사" 같은 직·간접 표현을 쓸 때 반드시 이 skill을 실행하세요. 단순 회사명 조회가 아니라 자사·경쟁사·입점·규제·인사·트렌드를 카테고리별로 묶어 정리한 종합 브리핑이 필요한 모든 상황에서 트리거합니다.
---

# Musinsa Brief

무신사 도메인을 7개 카테고리로 병렬 수집해 Obsidian vault에 한국어 마크다운 보고서로 저장하고, 채팅엔 경로+TL;DR+신호 카드만 보고하는 1인용 일간 브리핑 스킬.

## 왜 이 skill이 필요한가

무신사 직원 본인은 (1) 자사 IR·캠페인·입점, (2) 경쟁사 동향, (3) 규제·인사·업계 트렌드를 매일 5~10분 안에 한 번에 보고 싶어합니다. 카테고리별로 따로 검색하면 7~10개 쿼리를 매일 반복해야 하고, 결과도 산발적입니다. 이 스킬은 7개 카테고리를 **한 메시지에서 병렬** 수집하고, **소스 신뢰도 Tier**를 매기고, **임팩트 ●/◐/○**로 한눈에 보이게 정리해 vault에 누적합니다.

## 참조 문서 (Progressive disclosure)

자세한 내용은 필요할 때만 `Read`로 로드하세요. 메인 세션 context를 아끼기 위한 분할입니다.

- `references/categories.md` — 7 카테고리 정의 + 키워드 + 매체 우선순위 + dominant 결정 룰
- `references/source-tiers.md` — T1/T2/T3 매체 도메인 사전 + 정규화 규칙
- `references/query-templates.md` — 카테고리별 Tavily/WebSearch 쿼리 템플릿 10건
- `references/competitors.md` — 경쟁사·자회사·해외 사전 + 자동 라벨링 룰
- `references/setup-wizard.md` — Tavily 설치 안내 + 24h preflight 캐시 스키마 + welcome 안내
- `references/report-template.md` — 카드형 보고서 템플릿 + mode별 diff 섹션 + Executive Card 옵션

## 출력 위치

기본 경로: `~/Documents/obsidian/musinsa-brief/{YYYY-MM-DD}.md`.

### 경로 결정 절차
1. `~/.claude/data/musinsa-brief/config.json`이 있고 `output_dir` 필드가 존재 → 우선 사용
2. 없으면 default `~/Documents/obsidian/musinsa-brief/`
3. 디렉토리 없으면 `mkdir -p`. **default를 처음 사용한 경우(config.json 없음 + 디렉토리 신규 생성)** Step 8 채팅 보고에 안내 1줄 추가:
   `"💡 보고서를 ~/Documents/obsidian/musinsa-brief/에 저장했어요. Obsidian vault 경로가 다르면 ~/.claude/data/musinsa-brief/config.json 생성 후 {"output_dir": "<경로>"} 작성하세요."`
   1회 노출 후 `~/.claude/data/musinsa-brief/.welcomed` 빈 파일 touch해 재노출 방지
4. 같은 날짜 파일 존재 시 `{YYYY-MM-DD}-{HHMM}.md` 시간 suffix (refresh mode 자연스럽게 작동)

### config.json 예시
```json
{
  "output_dir": "~/Documents/obsidian/Musinsa Vault/02-Briefs/musinsa"
}
```

사용자가 "출력 경로 바꿔줘" / "저장 위치를 X로" 같이 요청하면 이 파일을 Write해 설정.

## 실행 순서

### Step 0 — Preflight (24h 캐시 우선)
- 캐시 파일 `~/.claude/data/musinsa-brief/preflight.json` Read
- `last_ok_at`이 24h 이내 + `checks.tavily == "ok"` → **skip해서 Step 1로**
- 캐시 miss / TTL 초과 → `references/setup-wizard.md`를 Read해 inline preflight:
  - `ToolSearch(query="select:mcp__tavily__tavily_search,mcp__tavily__tavily_extract", max_results=2)`
  - 실패 → setup-wizard halt 메시지 출력 후 **즉시 halt** (메인 워크플로우 진입 금지)
  - 성공 → 캐시 갱신 후 Step 1

### Step 1 — 컨텍스트 결정
1. 오늘 한국 날짜·시각(KST) 확인 (예: `2026-04-25 Sat 08:30`)
2. 출력 경로 결정 (위 "경로 결정 절차" 참조)
3. 출력 디렉토리 없으면 `mkdir -p`
4. `{output_dir}` 스캔 → `YYYY-MM-DD*.md` 중 가장 최근 파일 Read
5. 직전 파일의 YAML `report_date` 추출 → mode 결정:

| 직전 보고서 상태 | mode | days_window | since_date |
|---|---|---|---|
| 직전 보고서 없음 | `initial` | 14 | today − 14 |
| 직전 `report_date` == today | `refresh` | 1 | today |
| 직전 `report_date` < today | `since_last` | min(today − last, 30) | last `report_date` |

6. mode·since_date·days_window·directory_was_new(boolean) 변수 보존

### Step 2 — 7개 카테고리 병렬 검색 (한 메시지 안에서 동시)

`references/query-templates.md` Read 후 10개 쿼리에서 플레이스홀더 치환:
- `{days_window}` ← Step 1 산정값
- `{since_date}` ← Step 1 산정값 (ISO 8601)
- `{today}` ← KST today
- `{YYYY}` ← 4자리 연도
- `{Month}` ← 영문 월 풀이름 (예: April)

치환된 10개 `tavily_search` 호출을 **반드시 한 메시지 안에서 병렬**로 보냅니다. 순차 호출 금지.

### Step 3 — 합성

3.1. **URL 정규화**: 트래킹 파라미터(`utm_*`, `ref`, `fbclid`, `gclid`, `mc_cid`) 제거. fragment(`#`) 제거. 모바일 prefix(`m.`) 정규화 (source-tiers.md 매체 정규화 룰 참조).

3.2. **중복 제거**: 정규화 URL 일치 또는 제목 유사도 ≥ 0.9 (간이 판정: 제목 토큰 70%+ 일치)

3.3. **카테고리 분류**: Step 2 호출 번호로 1차 분류. `competitors` 카테고리는 `competitors.md` 자동 라벨링 룰로 사명 prefix 부착. "무신사"만 단독 등장한 항목은 같은 룰에 따라 `musinsa_ir/musinsa_campaign/musinsa_brands` 또는 `[무신사 본진]` 라벨로 재분류.

3.4. **Tier 등급 부여**: `source-tiers.md` 사전으로 도메인 → Tier 매핑. 미분류 default `T2`. `*.gov.kr/go.kr/or.kr` → T1 승격. `blog`/`personal` 포함 → T3 강등. T3 카테고리당 cap 1~2개 (초과분 제거).

3.5. **임팩트 자동 산정** (카테고리별):
- **● high**: 카테고리 내 T1 출처 ≥ 2 또는 unique 매체 ≥ 4
- **◐ mid**: T1 출처 1개 또는 T2 출처 ≥ 3
- **○ low**: T2 ≤ 2 또는 T3만 또는 0건
- **T1 출처 0건인 카테고리는 ● 자동 차단** (impact = mid가 cap)

3.6. **Dominant category 결정**:
- 임팩트 high인 카테고리 후보 추출
- 후보 중 unique 매체 수 최대인 것
- 동률이면 우선순위 `musinsa_ir > musinsa_campaign > musinsa_brands > competitors > regulation > industry_trend > org_hr`
- high가 0개면 가장 `item_count` 큰 카테고리로 fallback (절대 null 금지)

### Step 4 — Sanity Check 게이트

Step 6(저장) 직전 자동 검증. 위반 시 보고서 상단 ⚠️ 배너 또는 데이터 품질 푸터에 한 줄 추가 (배너는 #1 한정).

| # | 검증 | 조건 | 액션 |
|---|------|------|------|
| 1 | 전면 0건 | 7 카테고리 모두 `item_count == 0` | ⚠️ 배너 "검색 전면 실패 가능성 — Tavily 키/쿼리 점검 필요" |
| 2 | 단일 매체 편중 | 한 도메인 점유율 ≥ 70% | 푸터에 "출처 다양성 부족 ({domain} {pct}%)" |
| 3 | 중복 미정리 | 제목 유사도 ≥ 0.9 5건+ | 푸터에 "중복 정리 부족 가능성" |
| 4 | T3 비중 과도 | T3 / 전체 ≥ 30% | 푸터에 "신뢰도 낮은 출처 비중 높음" |
| 5 | Tavily 5+ 빈 응답 | 5/7 카테고리 이상 빈 응답 | 해당 카테고리만 WebSearch fallback (`query-templates.md` 하단 표) + 푸터 명시 |

### Step 5 — 출처 번호 normalize + YAML 기록 (저장 직전 필수)

5.1. 본문의 `[[n]](url)` 패턴 등장 순서로 스캔
5.2. 등장 순서대로 `1..N` 재할당. **같은 URL은 같은 번호로 통일**
5.3. 미사용 URL은 `_Sources_` 블록과 YAML `sources` 양쪽에서 모두 제외
5.4. 결과: `[[1]]..[[N]]` 연속 번호, gap 금지
5.5. YAML `sources` 배열에 `{id, url, title, domain, tier, category}` 동시 기록
5.6. YAML `categories` 배열에 카테고리별 `{id, label, impact, item_count}` 기록
5.7. YAML `data_quality.tier_counts` + `warnings` 기록

### Step 6 — 보고서 저장
- 파일명: `{output_dir}/{YYYY-MM-DD}.md`
- 같은 날짜 파일 존재 시 `{YYYY-MM-DD}-{HHMM}.md` (덮어쓰기 금지)
- 템플릿: `references/report-template.md` Read해 사용
- mode별 diff 섹션은 Step 1에서 결정한 mode에 따라 정확히 하나만 포함

### Step 7 — Preflight 캐시 갱신
전체 실행 무결 완료 시 `~/.claude/data/musinsa-brief/preflight.json`의 `last_ok_at`을 현재 KST ISO 8601로 갱신.

### Step 8 — 사용자 보고

채팅창에 **저장 경로 + mode + TL;DR + 신호 카드만** 출력. 본문 재첨부 금지.

```
🛍️ Musinsa Brief 저장 완료
- 경로: {output_dir}/{filename}
- mode: {mode} (since {since_date}, {days_window}일 윈도우)
- TL;DR:
  - {3~5줄}
- 신호 카드:
  | 카테고리 | 임팩트 |
  | ... | ... |
```

`directory_was_new == true`이면 위 보고 끝에 welcome 안내 1줄 추가 (setup-wizard.md "Welcome 안내" 참조). `~/.claude/data/musinsa-brief/.welcomed` 파일 touch.

### Step 9 (옵션) — Executive Card

원래 트리거 메시지에 `슬랙용으로`, `한 줄로`, `executive`, `exec card` 중 하나라도 포함 시:
- `references/report-template.md`의 "Executive Card 옵션" 섹션 사용
- 3줄 카드 추가 생성, 채팅에만 출력 (저장 X)

## 모드별 보고서 차이 (요약)

| mode | 본문 차이 | diff 섹션 |
|------|----------|-----------|
| `initial` | 14일 누적 컨텍스트 부각 | "## 📚 14일 누적 컨텍스트" |
| `refresh` | 신규 항목만 강조, 기존은 "(변동 없음)". 신호 카드 임팩트는 (직전 `categories[].item_count` + 이번 신규 항목 수)로 합산 후 Step 3.5 룰 재적용 | "## 🔁 오늘 신규 (vs 직전 보고서)" |
| `since_last` | 신규/진척/종료 항목 분류 | "## 🔁 Since-last 변화 (since {since_date})" |

## 분석 원칙

1. **사실과 추측 분리**: 인용 가능한 사실은 `[[n]](url)` 출처. 추측은 "~로 보임", "~라는 관측" 명시. T3 인용은 카테고리당 1~2개로 cap.
2. **빈 섹션 금지**: 카테고리 카드든 mode diff든 빈값이면 "특이사항 없음" 명시.
3. **무신사 본진 우선**: 같은 항목이 `competitors`와 `musinsa_*`에 모두 매칭되면 `musinsa_*`로 분류.
4. **자회사 표기**: 29CM 같은 자회사는 `competitors`에 두되 `(자회사)` 라벨 부착.
5. **TL;DR**: 정확히 3~5 bullet, 한줄평 중복 금지. 가장 임팩트 큰 dominant category 한 줄, 그 외 임팩트 high 카테고리에서 1~3줄, since-last 핵심 변화 1줄.

## 에러 처리

| 상황 | 동작 |
|------|------|
| Tavily 미등록 (Step 0 실패) | halt + setup-wizard 안내 |
| 일부 카테고리 쿼리 실패 (1~4건) | 해당 카테고리 "데이터 없음 (쿼리 실패)" 표시, 나머지 진행. 데이터 품질 푸터에 실패 목록 |
| Tavily 5+ 카테고리 빈 응답 (Sanity #5) | WebSearch fallback 자동 실행 (해당 카테고리만), 푸터 명시 |
| 모든 카테고리 0건 (Sanity #1) | ⚠️ 배너 + 푸터에 점검 안내 |
| 출력 디렉토리 생성 실패 (권한 등) | halt + "config.json의 output_dir 점검" 안내 |
| 같은 날짜 파일 존재 | `YYYY-MM-DD-HHMM.md` 시간 suffix |
| 직전 보고서 frontmatter 파싱 실패 | mode = `initial`로 fallback, 푸터에 한 줄 |
| Tavily 429 (rate limit) | 3초 대기 후 1회 재시도, 그래도 실패면 해당 카테고리 단위 skip + 푸터 명시 |

## 금지 사항

- 사내 데이터 호출 금지 (위키·슬랙·JIRA — v0.2+에서 별도 검토)
- 주가/시세 호출 금지 (무신사·모회사 비상장)
- 재무 권유성 표현 금지 (투자·매수·매도 추천)
- T3 출처 카테고리당 3개 이상 금지
- 본문 채팅 재첨부 금지 (경로 + TL;DR + 신호 카드만)
- 추측을 사실처럼 쓰지 말 것 (`~로 보임` 명시)
- 베팅·도박·가십성 자극 표현 금지
- 개별 직원 가십·평판 금지 (외부 보도된 임원 인사·조직 변동만)
- Context7·Serena 등 코드 도메인 MCP 사용 금지 (도메인 부적합)
- 쿼리 순차 호출 금지 (반드시 한 메시지 안에 병렬 10개)
