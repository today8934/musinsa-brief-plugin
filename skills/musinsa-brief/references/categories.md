# Categories — 7-Category Matrix

`musinsa-brief` SKILL.md의 Step 2에서 Read해 사용. 각 카테고리는 (1) id, (2) 한국어 라벨, (3) 핵심 키워드(검색·매칭용), (4) 매체 우선순위, (5) Tavily `include_domains` 후보, (6) 호출 수를 가짐.

## 카테고리 1 — `musinsa_ir` (자사 IR/실적/M&A)

- **라벨**: 자사 IR/실적/M&A
- **핵심 키워드 (KR)**: 무신사 거래액, 무신사 매출, 무신사 영업이익, 무신사 적자, 무신사 흑자, 무신사 투자, 무신사 인수, 무신사 상장, 무신사 IPO, 무신사 펀딩, 무신사 valuation, 무신사 시리즈, 무신사 자회사, 무신사 글로벌
- **핵심 키워드 (EN)**: Musinsa revenue, Musinsa funding, Musinsa IPO, Musinsa Series, Musinsa M&A, Musinsa global, Musinsa Japan, Musinsa US
- **매체 우선순위**: T1 한국 일간/경제지(매경·한경·머투·조선비즈·동아·중앙) + T1 글로벌(Reuters·Bloomberg·BoF·WSJ) + 자사 PR
- **호출 수**: 한국 1 + 글로벌 1 = **2 쿼리**

## 카테고리 2 — `musinsa_campaign` (자사 캠페인/마케팅)

- **라벨**: 자사 캠페인/마케팅
- **핵심 키워드 (KR)**: 무신사 스탠다드 캠페인, 무신사 광고, 무신사 기획전, 무신사 콜라보, 무신사 이벤트, 무신사 모델, 무신사 앰버서더, 무신사 스토어, 무신사 오프라인, 무신사 팝업
- **매체 우선순위**: T2 한국 패션 전문지(패션비즈·어패럴뉴스·뉴데일리·패션포스트)
- **호출 수**: 한국 1 = **1 쿼리**

## 카테고리 3 — `musinsa_brands` (입점·브랜드 변동)

- **라벨**: 입점·브랜드 변동
- **핵심 키워드 (KR)**: 무신사 입점, 무신사 단독, 무신사 컬래버, 무신사 신규 브랜드, 무신사 PB, 무신사 셀러, 무신사 베스트, 무신사 카테고리, 무신사 디자이너 브랜드
- **매체 우선순위**: T2 패션 전문지 + 자사 PR
- **호출 수**: 한국 1 = **1 쿼리**

## 카테고리 4 — `competitors` (경쟁사 동향)

- **라벨**: 경쟁사 동향
- **핵심 키워드 (KR)**: 29CM, SSF샵, 에이블리, 지그재그, W컨셉, 카카오스타일, 솔드아웃, KREAM, 크림, 트렌비
- **핵심 키워드 (EN)**: Shein, Temu, Yoox, Farfetch, ssense, Vinted, Zalando, Korea fashion platform
- **매체 우선순위**: T1+T2 한국 + T1+T2 글로벌
- **호출 수**: 한국 1 + 글로벌 1 = **2 쿼리**
- **상세 사전**: `competitors.md` 참조

## 카테고리 5 — `regulation` (규제·정책)

- **라벨**: 규제·정책
- **핵심 키워드 (KR)**: 공정위 패션 플랫폼, 표시광고, 가품, 플랫폼법, 전자상거래법, 온라인 패션 규제, 소비자보호, 전상법 개정, 플랫폼 공정거래
- **매체 우선순위**: T1 매경·한경 + T2 디지털데일리·뉴데일리·패션포스트
- **호출 수**: 한국 1 = **1 쿼리**

## 카테고리 6 — `org_hr` (인사·조직)

- **라벨**: 인사·조직
- **핵심 키워드 (KR)**: 무신사 임원, 무신사 인사, 무신사 채용, 무신사 감원, 무신사 구조조정, 무신사 노조, 무신사 대표, 무신사 CEO, 무신사 CTO, 무신사 임원 영입
- **매체 우선순위**: T1 매경·한경 + T3 cap (블라인드·잡플래닛 합쳐 최대 1)
- **호출 수**: 한국 1 = **1 쿼리**

## 카테고리 7 — `industry_trend` (업계 트렌드)

- **라벨**: 업계 트렌드
- **핵심 키워드 (KR)**: K-fashion 글로벌, K-패션 수출, 라이브커머스 패션, AI 패션 검색, 친환경 패션, Y2K 패션 트렌드, 한국 패션 수출, D2C 브랜드, 패션 D2C
- **핵심 키워드 (EN)**: K-fashion global, livestream commerce fashion, AI fashion search, sustainable fashion Korea, Korean fashion export
- **매체 우선순위**: T1+T2 한국 + T1 글로벌(BoF·WWD·VogueBusiness 위주)
- **호출 수**: 한국 1 + 글로벌 1 = **2 쿼리**

## 호출 수 합계

| 카테고리 | 쿼리 수 |
|---|---|
| 1. musinsa_ir | 2 |
| 2. musinsa_campaign | 1 |
| 3. musinsa_brands | 1 |
| 4. competitors | 2 |
| 5. regulation | 1 |
| 6. org_hr | 1 |
| 7. industry_trend | 2 |
| **합계** | **10 쿼리/일** |

월 누적: 10 × 30 = **300 credits/월** (Tavily 무료 한도 1,000 중 30%, 안전 여유 70%).

## Dominant Category 결정 룰

`musinsa-brief` Step 3.6에서 사용:

1. 임팩트 `high`인 카테고리 후보 모음
2. 후보 중 매체 다양성(unique domain 수) 최고
3. 동률이면 우선순위: `musinsa_ir` > `musinsa_campaign` > `musinsa_brands` > `competitors` > `regulation` > `industry_trend` > `org_hr`
