# Query Templates — 카테고리별 Tavily 쿼리 10건

`musinsa-brief` Step 2에서 Read해 한 메시지 안에서 병렬 호출. 플레이스홀더는 호출 직전 메인 세션에서 치환:

- `{days_window}` — Step 1에서 산정 (initial=14, refresh=1, since_last=today−last_report_date)
- `{since_date}` — KST 날짜 ISO 8601 (예: `2026-04-22`)
- `{today}` — KST 오늘 날짜
- `{YYYY}`, `{Month}` — 영문 쿼리에 현재 연/월 자동 삽입 (Tavily 최신성 보정)

## 쿼리 1 — `musinsa_ir` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "무신사 {YYYY} 거래액 매출 영업이익 투자",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "mk.co.kr", "hankyung.com", "mt.co.kr", "biz.chosun.com",
    "donga.com", "joongang.co.kr", "news.musinsa.com"
  ]
}
```

## 쿼리 2 — `musinsa_ir` (EN)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "Musinsa {YYYY} revenue funding IPO global expansion",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "reuters.com", "bloomberg.com", "businessoffashion.com",
    "wsj.com", "ft.com", "nikkei.com"
  ]
}
```

## 쿼리 3 — `musinsa_campaign` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "무신사 {YYYY} {Month} 캠페인 광고 기획전 콜라보 모델",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "fashionbiz.co.kr", "apparelnews.co.kr", "fpost.co.kr",
    "ndaily.kr", "ktnews.com"
  ]
}
```

## 쿼리 4 — `musinsa_brands` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "무신사 {YYYY} {Month} 입점 단독 컬래버 신규 브랜드 PB",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "fashionbiz.co.kr", "apparelnews.co.kr", "fpost.co.kr",
    "news.musinsa.com", "corp.musinsa.com"
  ]
}
```

## 쿼리 5 — `competitors` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "{YYYY} 29CM 에이블리 지그재그 SSF W컨셉 카카오스타일 패션 플랫폼",
  "days": "{days_window}",
  "max_results": 10,
  "include_domains": [
    "mk.co.kr", "hankyung.com", "mt.co.kr", "biz.chosun.com",
    "fashionbiz.co.kr", "apparelnews.co.kr", "fpost.co.kr"
  ]
}
```

## 쿼리 6 — `competitors` (EN)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "{YYYY} Shein Temu Korea fashion platform competition",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "reuters.com", "bloomberg.com", "businessoffashion.com",
    "wwd.com", "voguebusiness.com", "retaildive.com"
  ]
}
```

## 쿼리 7 — `regulation` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "{YYYY} 공정위 패션 플랫폼 표시광고 가품 전자상거래법 플랫폼법",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "mk.co.kr", "hankyung.com", "mt.co.kr",
    "ddaily.co.kr", "ndaily.kr", "fpost.co.kr"
  ]
}
```

## 쿼리 8 — `org_hr` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "무신사 {YYYY} 임원 인사 채용 노조 구조조정 영입",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "mk.co.kr", "hankyung.com", "mt.co.kr", "biz.chosun.com",
    "fashionbiz.co.kr", "apparelnews.co.kr"
  ]
}
```

## 쿼리 9 — `industry_trend` (KR)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "{YYYY} K-fashion 글로벌 라이브커머스 패션 D2C 한국 패션 수출",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "mk.co.kr", "hankyung.com", "fashionbiz.co.kr",
    "apparelnews.co.kr", "fpost.co.kr", "ndaily.kr"
  ]
}
```

## 쿼리 10 — `industry_trend` (EN)

```json
{
  "tool": "mcp__tavily__tavily_search",
  "query": "{YYYY} K-fashion global expansion livestream commerce sustainable",
  "days": "{days_window}",
  "max_results": 8,
  "include_domains": [
    "businessoffashion.com", "wwd.com", "voguebusiness.com",
    "fashionunited.com", "retaildive.com", "modernretail.co"
  ]
}
```

## WebSearch Fallback (카테고리 빈 응답 시)

Tavily가 5/7+ 카테고리 빈 응답이면 WebSearch fallback. 카테고리당 1쿼리:

| 카테고리 | WebSearch 쿼리 |
|---------|----------------|
| musinsa_ir | `"무신사 {YYYY} {Month} 거래액 매출 site:mk.co.kr OR site:hankyung.com"` |
| musinsa_campaign | `"무신사 {YYYY} {Month} 캠페인 site:fashionbiz.co.kr OR site:apparelnews.co.kr"` |
| musinsa_brands | `"무신사 {YYYY} {Month} 입점 단독 site:fashionbiz.co.kr OR site:fpost.co.kr"` |
| competitors | `"{YYYY} 에이블리 OR 29CM OR 지그재그 OR W컨셉 패션 플랫폼 site:mk.co.kr OR site:fashionbiz.co.kr"` |
| regulation | `"{YYYY} {Month} 공정위 패션 플랫폼 표시광고 site:mk.co.kr OR site:ddaily.co.kr"` |
| org_hr | `"무신사 {YYYY} 임원 인사 site:mk.co.kr OR site:hankyung.com"` |
| industry_trend | `"K-fashion {YYYY} global site:businessoffashion.com OR site:wwd.com"` |
