# Source Tiers — 매체 신뢰도 사전

`musinsa-brief` Step 3.4에서 출처 도메인을 Tier로 매핑할 때 사용. 미분류 도메인은 default `T2`.

## Tier 정의

| Tier | 정의 |
|------|------|
| **T1** | 공식 IR + 메이저 일간/경제지 + Reuters/BoF 등 글로벌 신뢰 매체 |
| **T2** | 패션·리테일·이커머스 전문지 |
| **T3** | 블로그·X·커뮤니티 (cap 1~2개, "신뢰도 낮음" 명시) |

## T1 — 한국

| 도메인 | 매체 | 비고 |
|--------|------|------|
| `mk.co.kr` | 매일경제 | |
| `hankyung.com` | 한국경제 | |
| `mt.co.kr` | 머니투데이 | |
| `biz.chosun.com` | 조선비즈 | |
| `news.chosun.com` | 조선일보 | |
| `donga.com` | 동아일보 | |
| `joongang.co.kr` | 중앙일보 | |
| `news.musinsa.com` | 무신사 PR | 자사 공식 |
| `corp.musinsa.com` | 무신사 PR | 자사 공식 |
| `news.kotra.or.kr` | KOTRA | 글로벌 진출 보도 |

## T1 — 글로벌

| 도메인 | 매체 |
|--------|------|
| `reuters.com` | Reuters |
| `bloomberg.com` | Bloomberg |
| `businessoffashion.com` | Business of Fashion (BoF) |
| `wsj.com` | Wall Street Journal |
| `ft.com` | Financial Times |
| `nikkei.com` | Nikkei |

## T2 — 한국

| 도메인 | 매체 |
|--------|------|
| `fashionbiz.co.kr` | 패션비즈 |
| `apparelnews.co.kr` | 어패럴뉴스 |
| `fpost.co.kr` | 패션포스트 |
| `fpnews.kr` | FPN |
| `ktnews.com` | 한국섬유신문 |
| `ddaily.co.kr` | 디지털데일리 |
| `ndaily.kr` | 뉴데일리 |
| `dailian.co.kr` | 데일리안 |
| `news.naver.com` | 네이버 뉴스 (출처 매체 별도 분류) |
| `n.news.naver.com` | 네이버 뉴스 |

## T2 — 글로벌

| 도메인 | 매체 |
|--------|------|
| `wwd.com` | WWD |
| `voguebusiness.com` | Vogue Business |
| `fashionunited.com` | FashionUnited |
| `retaildive.com` | Retail Dive |
| `modernretail.co` | Modern Retail |
| `glossy.co` | Glossy |

## T3 — 블로그·X·커뮤니티

| 도메인 (또는 prefix) | 종류 | 사용 룰 |
|---------------------|------|---------|
| `x.com`, `twitter.com` | 마이크로블로그 | 카테고리당 cap 1, "신뢰도 낮음" 명시 필수 |
| `brunch.co.kr` | 브런치 | cap 1 |
| `blog.naver.com`, `m.blog.naver.com` | 네이버 블로그 | cap 1 |
| `blind.kr`, `teamblind.com` | 블라인드 | `org_hr` 카테고리 외엔 사용 금지 |
| `jobplanet.co.kr` | 잡플래닛 | `org_hr` 카테고리 외엔 사용 금지 |
| 기타 개인 블로그·뉴스레터 | 개인 | T3로 분류 |

## 미분류 도메인

위 사전에 없는 도메인은 default **T2**. 단:
- 도메인이 `*.gov.kr`, `*.go.kr`, `*.or.kr`이면 T1로 승격 (정부·공공)
- 도메인에 `blog`, `personal`이 들어가면 T3로 강등

## 매체 정규화

같은 매체의 여러 도메인은 하나로 통합 카운트:
- `news.naver.com` + `n.news.naver.com` → "Naver News" (실제 매체는 본문에서 추출)
- `x.com` + `twitter.com` → "X (Twitter)"
- `mk.co.kr` + `m.mk.co.kr` → "매일경제"

`m.<domain>` 모바일 변형은 desktop과 동일 매체로 정규화.

## Tier 카운트 규칙 (Sanity check용)

- T1 비중: T1 출처 수 / 전체 출처 수
- T3 비중: T3 출처 수 / 전체 출처 수
- T3 비중 ≥ 30% → 데이터 품질 섹션에 "신뢰도 낮은 출처 비중 높음" 한 줄
- T1 비중 0% (전 카테고리) → 데이터 품질 섹션에 "T1 출처 부재" 한 줄
