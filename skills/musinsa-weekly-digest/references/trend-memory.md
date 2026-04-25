# Trend Memory — 반복 등장 테마 추출 룰

`musinsa-weekly-digest` Step 4에서 Read. 일간 보고서 frontmatter `sources[].title`을 토큰화해 N≥2일 등장하는 테마를 자동 추출.

## 입력
- 기간 내 일간 보고서 N건의 frontmatter `sources` 배열 합집합
- 각 source는 `{id, url, title, domain, tier, category}` 구조
- `report_date`별로 그룹화 가능

## 토큰화 절차

### 1. 한국어/영어 명사 후보 추출
- title을 공백·특수문자(`-_.,()[]"'`)로 split
- 토큰 길이 2자 이상만 채택
- 한글 어미·조사 제거 시도 (간이 룰):
  - 어미 `를/을/이/가/은/는/도/만/와/과/의/에/로/으로` 끝나면 잘라냄
  - 공통 1글자 접미 `-사`, `-부`, `-팀`, `-측` 유지

### 2. 영문 토큰 정규화
- 모두 소문자화
- 단순 plural 처리: `s`로 끝나면 `s` 제거 (`brands` → `brand`)
- 약어는 그대로 유지 (`IPO`, `D2C`, `K-fashion`)

### 3. Stopword 필터

다음 토큰은 제외 (반복 등장의 의미가 없음):

```
일반어: 발표, 출시, 진행, 시작, 운영, 강화, 확대, 추진, 검토, 도입,
        전망, 예상, 계획, 목표, 실시, 진출, 추가, 신규, 기존,
        the, of, in, on, for, and, with, to, a, an, is, are, was,
        will, has, have, had, this, that, these, those, new, more,
        업계, 시장, 기업, 회사, 브랜드, 플랫폼, 패션, 한국, 글로벌, fashion, korea, korean, global, market, industry

매체명·카테고리 라벨 (이미 별도 카운팅 됨):
        매일경제, 한국경제, 머니투데이, 패션비즈, 어패럴뉴스,
        Reuters, Bloomberg, BoF, WWD, VogueBusiness,
        무신사, 자사, IR, 캠페인, 입점, 경쟁사, 규제, 인사, 트렌드

호칭·기관 일반어:
        대표, 사장, CEO, CTO, 부사장, 임원, 위원회, 부, 청, 처
```

### 4. 빈도 카운팅
- 토큰별 등장 횟수 = `sum(같은 토큰 등장한 source 수)`
- 토큰별 등장 일자 set = `{report_date for source if 토큰 in source.title}`
- N ≥ 2 (서로 다른 일자 2일 이상 등장)인 토큰만 채택

### 5. 정렬·top 선택
- 1차: 등장 일자 수 내림차순
- 2차: 총 source 등장 수 내림차순
- top 10 선택. 단, top 10 안에 같은 stem 토큰(예: "투자"·"투자한")이 있으면 더 짧은 쪽 1개로 통합

## 출력 스키마

```json
[
  {
    "theme": "에이블리 시리즈D",
    "days_appeared": 5,
    "total_mentions": 8,
    "domains": ["mk.co.kr", "hankyung.com", "fashionbiz.co.kr"],
    "first_seen": "2026-04-21",
    "last_seen": "2026-04-26"
  },
  ...
]
```

이 스키마를 Weekly Digest 보고서의 "## 🔁 Trend Memory" 섹션에 변환:

```markdown
- **에이블리 시리즈D**: 5/7일 등장 (매일경제·한국경제·패션비즈)
- **공정위 표시광고**: 4/7일 등장 (매일경제·디지털데일리·뉴데일리·패션포스트)
- **무신사 일본 진출**: 3/7일 등장 (한국경제·패션비즈·BoF)
```

## 동의어 합치기 (옵션)

다음 패턴은 같은 테마로 통합:
- `시리즈D` ⇔ `Series D` ⇔ `시리즈 D` ⇔ `시리즈디`
- `IPO` ⇔ `상장`
- `M&A` ⇔ `인수합병`
- 같은 인물명의 한국어/영어 표기 (예: `Romano`/`로마노`)

이 동의어 사전은 우선 비워두고, v0.2에서 사용자 보고 후 채움.

## 다음 기간 관전 포인트 자동 추출

Trend Memory의 top 5 테마 중 다음 조건을 만족하는 항목을 "관전 포인트"로 변환:
- `last_seen`이 분석 기간 마지막 3일 이내 (활성 이슈)
- `domains.length` ≥ 2 (멀티 매체 인용 = 추적 가치)

각 항목을 한 줄 미래형 표현으로 변환:
- `"공정위 표시광고"` + 진행 중 → `"공정위 표시광고 가이드라인 입법예고 종료 (관전)"`
- `"에이블리 시리즈D"` + 마감 임박 → `"에이블리 시리즈 D 마감 후 자본 활용 계획 (후속)"`

자동 변환 룰이 모호하면 fallback 표현: `"{테마} 후속 보도 모니터링 필요"`.
