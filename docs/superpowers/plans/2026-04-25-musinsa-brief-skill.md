# Musinsa Brief Plugin v0.1.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `musinsa-brief-plugin` v0.1.0 — a Claude Code plugin with two skills (`musinsa-brief` daily + `musinsa-weekly-digest` rollup) that collects Musinsa-domain news/IR/competitor/regulation/HR/industry signals across 7 categories via Tavily MCP and writes a Korean Markdown brief into the user's Obsidian vault, reporting only the path + TL;DR + signal card to chat.

**Architecture:** File-only plugin (no executable code) consisting of one `plugin.json` manifest, two skills each with their own `SKILL.md` + `references/*.md` files for progressive disclosure, plus a marketplace registration entry. The skill is executed by Claude Code itself: Claude reads `SKILL.md`, follows step instructions, calls Tavily/WebSearch tools in parallel, normalizes citations, writes the report file, and reports back. State lives only in two JSON files (`preflight.json`, `config.json`) under `~/.claude/data/musinsa-brief/`.

**Tech Stack:** Markdown (skill content), JSON (manifest + config), YAML frontmatter (skill metadata + report frontmatter), Bash/jq/grep for self-validation, Tavily MCP (`tavily_search`, `tavily_extract`), built-in Claude tools (`WebSearch`, `Read`, `Write`, `Edit`, `Bash`, `ToolSearch`).

**Spec reference:** `docs/superpowers/specs/2026-04-25-musinsa-brief-skill-design.md` (565 lines, committed `79cbfde`).

**Working directory:** `/Users/wooksangryu/workspace/musinsa-brief-plugin/` (already `git init -b main`'d, root commit contains spec).

---

## Pre-flight Environment Check

Before starting tasks, verify the environment:

```bash
# All commands assume cwd is /Users/wooksangryu/workspace/musinsa-brief-plugin
which jq && jq --version       # Expect jq present (used in validation)
which python3 && python3 --version  # Expect Python 3.x (yaml validation)
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin status  # Expect clean working tree on main
ls /Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json  # Expect file present (Task 16 will modify it)
```

If `jq` is missing: `brew install jq`. If marketplace.json missing, halt and notify user (we don't create it).

---

## Phase 1 — Plugin Scaffolding (Tasks 1-3)

### Task 1: `.gitignore` and `LICENSE`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/.gitignore`
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/LICENSE`

- [ ] **Step 1: Write `.gitignore`**

Use the Write tool to create `/Users/wooksangryu/workspace/musinsa-brief-plugin/.gitignore` with this content:

```
.DS_Store
.idea/
.vscode/
.omc/
node_modules/
*.log
.env
.env.local
__pycache__/
*.pyc
```

- [ ] **Step 2: Write `LICENSE` (MIT)**

Use the Write tool to create `/Users/wooksangryu/workspace/musinsa-brief-plugin/LICENSE`:

```
MIT License

Copyright (c) 2026 류욱상 (Wooksang Ryu)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Verify both files exist and are non-empty**

Run:
```bash
[ -s /Users/wooksangryu/workspace/musinsa-brief-plugin/.gitignore ] && echo "✅ gitignore" || echo "❌ gitignore"
[ -s /Users/wooksangryu/workspace/musinsa-brief-plugin/LICENSE ] && echo "✅ license" || echo "❌ license"
```
Expected: both lines start with `✅`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add .gitignore LICENSE
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "chore: add MIT LICENSE and .gitignore"
```

---

### Task 2: `plugin.json`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/.claude-plugin/plugin.json`

- [ ] **Step 1: Create directory and write `plugin.json`**

```bash
mkdir -p /Users/wooksangryu/workspace/musinsa-brief-plugin/.claude-plugin
```

Use the Write tool to create `/Users/wooksangryu/workspace/musinsa-brief-plugin/.claude-plugin/plugin.json`:

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

- [ ] **Step 2: Validate JSON syntactically and check required fields**

Run:
```bash
jq -e '.name == "musinsa-brief-plugin" and .version == "0.1.0" and (.keywords | length) >= 6' \
  /Users/wooksangryu/workspace/musinsa-brief-plugin/.claude-plugin/plugin.json \
  && echo "✅ plugin.json valid" || echo "❌ plugin.json invalid"
```
Expected: `✅ plugin.json valid`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add .claude-plugin/plugin.json
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(manifest): add plugin.json v0.1.0"
```

---

### Task 3: README.md (placeholder, full version in Task 15)

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/README.md`

We write a minimal README first so the repo is presentable; full README replaces it in Task 15.

- [ ] **Step 1: Write minimal README**

Use the Write tool to create `/Users/wooksangryu/workspace/musinsa-brief-plugin/README.md`:

```markdown
# Musinsa Brief Plugin

무신사 자사·경쟁사·입점·규제·인사·업계 트렌드를 7개 카테고리로 병렬 수집해 Obsidian vault에 한국어 마크다운 보고서를 저장하는 Claude Code 플러그인.

> 🚧 v0.1.0 개발 중. 자세한 사용법은 작업 완료 후 갱신.

- Spec: `docs/superpowers/specs/2026-04-25-musinsa-brief-skill-design.md`
- Plan: `docs/superpowers/plans/2026-04-25-musinsa-brief-skill.md`

## License

MIT
```

- [ ] **Step 2: Verify README exists**

```bash
[ -s /Users/wooksangryu/workspace/musinsa-brief-plugin/README.md ] && echo "✅ readme" || echo "❌ readme"
```
Expected: `✅ readme`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add README.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "docs(readme): add placeholder README (full version pending)"
```

---

## Phase 2 — `musinsa-brief` Skill Files (Tasks 4-11)

### Task 4: SKILL.md frontmatter (trigger description)

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md`

This task creates SKILL.md with **only frontmatter + a one-line title** so the skill becomes discoverable. The full body is added in Task 11 after all references exist.

- [ ] **Step 1: Create skill directory**

```bash
mkdir -p /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references
```

- [ ] **Step 2: Write SKILL.md frontmatter**

Use the Write tool to create `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md`:

```markdown
---
name: musinsa-brief
description: 무신사 자사·경쟁사·입점·규제·인사·트렌드 7개 카테고리를 Tavily 병렬 수집해 한국어 마크다운 보고서를 Obsidian vault에 저장하고 채팅엔 경로+TL;DR+신호 카드만 보고합니다. 사용자가 "무신사", "musinsa-brief", "무신사 소식", "무신사 뉴스", "무신사 근황", "무신사 어떻게 돼", "무신사 어땠어", "오늘 무신사", "어제 무신사", "무신사 brief", "무신사 IR", "무신사 인사", "무신사 동향", "무신사 정리해줘", "무신사 업데이트", "무신사 업계", "무신사 경쟁사" 같은 직·간접 표현을 쓸 때 반드시 이 skill을 실행하세요. 단순 회사명 조회가 아니라 자사·경쟁사·입점·규제·인사·트렌드를 카테고리별로 묶어 정리한 종합 브리핑이 필요한 모든 상황에서 트리거합니다.
---

# Musinsa Brief

(본문은 references 파일이 모두 갖춰진 후 Task 11에서 작성됩니다.)
```

- [ ] **Step 3: Validate frontmatter parses as YAML**

Run:
```bash
python3 -c "
import yaml, sys
p = '/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md'
with open(p) as f:
    txt = f.read()
fm = txt.split('---', 2)[1]
data = yaml.safe_load(fm)
assert data['name'] == 'musinsa-brief', f\"name mismatch: {data.get('name')}\"
assert '무신사' in data['description'], 'trigger keyword missing'
assert len(data['description']) > 200, 'description too short'
print('✅ SKILL.md frontmatter ok')
"
```
Expected: `✅ SKILL.md frontmatter ok`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/SKILL.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add musinsa-brief SKILL.md frontmatter (body pending)"
```

---

### Task 5: `references/categories.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/categories.md`

- [ ] **Step 1: Write categories.md**

Use the Write tool to create the file with this exact content:

```markdown
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
```

- [ ] **Step 2: Verify file mentions all 7 category ids**

```bash
for id in musinsa_ir musinsa_campaign musinsa_brands competitors regulation org_hr industry_trend; do
  grep -q "\`$id\`" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/categories.md \
    || { echo "❌ missing $id"; exit 1; }
done
echo "✅ all 7 category ids present"
```
Expected: `✅ all 7 category ids present`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/references/categories.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add categories.md (7-category matrix)"
```

---

### Task 6: `references/source-tiers.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/source-tiers.md`

- [ ] **Step 1: Write source-tiers.md**

Use the Write tool to create the file with this exact content:

```markdown
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
```

- [ ] **Step 2: Verify all required tier sections present**

```bash
for section in "T1 — 한국" "T1 — 글로벌" "T2 — 한국" "T2 — 글로벌" "T3 — 블로그"; do
  grep -q "$section" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/source-tiers.md \
    || { echo "❌ missing section: $section"; exit 1; }
done
echo "✅ all tier sections present"
```
Expected: `✅ all tier sections present`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/references/source-tiers.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add source-tiers.md (T1/T2/T3 domain dictionary)"
```

---

### Task 7: `references/query-templates.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/query-templates.md`

- [ ] **Step 1: Write query-templates.md**

Use the Write tool to create the file with this exact content:

```markdown
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
```

- [ ] **Step 2: Verify all 10 query blocks present**

```bash
count=$(grep -c '^## 쿼리 ' /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/query-templates.md)
[ "$count" = "10" ] && echo "✅ 10 query templates" || echo "❌ found $count, expected 10"
```
Expected: `✅ 10 query templates`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/references/query-templates.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add query-templates.md (10 Tavily query blueprints)"
```

---

### Task 8: `references/competitors.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/competitors.md`

- [ ] **Step 1: Write competitors.md**

Use the Write tool to create the file with this exact content:

```markdown
# Competitors — 경쟁사·자회사·해외 사전

`musinsa-brief` Step 3에서 `competitors` 카테고리 결과를 정렬·분류할 때 참조. 쿼리 결과의 본문 키워드와 매칭해 어느 경쟁사 이슈인지 자동 라벨링.

## 한국 경쟁사

| 사명 | 별칭 | 분류 | 비고 |
|------|------|------|------|
| 29CM | 29센티미터 | **자회사** | 무신사 인수, 별도 운영 |
| SSF샵 | 삼성물산 패션 | 대형 종합 | 명품·SPA 강함 |
| 에이블리 | Ably | 여성 D2C | 시리즈 D 트래킹 |
| 지그재그 | Zigzag | 여성 D2C | 카카오스타일 운영 |
| W컨셉 | W Concept | 여성 디자이너 | SSG 자회사 |
| 카카오스타일 | KakaoStyle | 통합 운영사 | 지그재그·포스티·스타일 모음 |
| 솔드아웃 | SoldOut | 한정판 리셀 | 한정판 스니커즈 위주 |
| KREAM | 크림 | 한정판 리셀 | 네이버 자회사 |
| 트렌비 | Trenbe | 명품 | |
| 발란 | Balaan | 명품 | |
| 머스트잇 | Mustit | 명품 | |

## 글로벌 경쟁사

| 사명 | 분류 | 비고 |
|------|------|------|
| Shein | 글로벌 SPA | 한국 진출 가속 |
| Temu | 글로벌 마켓플레이스 | 저가 |
| Yoox | 글로벌 명품 | YNAP 그룹 |
| Farfetch | 글로벌 명품 | |
| ssense | 글로벌 디자이너 | |
| Vinted | 글로벌 리세일 | 유럽 강함 |
| Zalando | 유럽 종합 | |
| Goat | 글로벌 한정판 | 솔드아웃·KREAM 경쟁 |
| StockX | 글로벌 한정판 | |

## 자동 라벨링 룰

`competitors` 쿼리(5·6) 결과의 title/snippet에서 위 표의 사명·별칭이 등장하면 카테고리 카드 본문에 `[<사명>]` 라벨 prefix 부착:

```
- **[T1] [에이블리]** 시리즈 D 1,500억 마감 — 매일경제 [[5]]
- **[T2] [Shein]** 한국 진출 본격화, ... — Reuters [[6]]
- **[T1] [29CM]** (자회사) 신규 캠페인 ... — 패션비즈 [[7]]
```

29CM가 자회사임을 인지: `[29CM]` 라벨 옆에 `(자회사)` 명시. 카테고리 분류는 `competitors` 유지하되 카테고리 카드 본문에 자회사 표기로 시그널 분리.

## 동시 등장 처리

여러 경쟁사가 한 기사에 동시 등장(예: "에이블리·지그재그·W컨셉 패션 플랫폼 5사 비교") → 라벨은 본문에 등장한 모든 사명 나열: `[에이블리/지그재그/W컨셉]`. 정렬 가중치는 첫 등장 사명 기준.

## 무신사 자체 언급 제거

`competitors` 쿼리 결과에 "무신사"만 단독 언급된 기사가 있으면 → `competitors` 카테고리에서 제외하고 `musinsa_ir` 또는 `musinsa_campaign` 카테고리로 재분류. 키워드 우선순위:
- "거래액·매출·투자·M&A" 포함 → `musinsa_ir`
- "캠페인·광고·기획전·모델" 포함 → `musinsa_campaign`
- "입점·단독·컬래버·PB" 포함 → `musinsa_brands`
- 그 외 → `competitors` 유지하되 라벨 `[무신사 본진]`로 표시
```

- [ ] **Step 2: Verify required competitor names present**

```bash
for name in 29CM 에이블리 지그재그 W컨셉 Shein Temu KREAM; do
  grep -q "$name" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/competitors.md \
    || { echo "❌ missing: $name"; exit 1; }
done
echo "✅ all key competitors listed"
```
Expected: `✅ all key competitors listed`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/references/competitors.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add competitors.md (KR/global competitor dictionary)"
```

---

### Task 9: `references/setup-wizard.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/setup-wizard.md`

- [ ] **Step 1: Write setup-wizard.md**

Use the Write tool to create the file with this exact content:

```markdown
# Setup Wizard — Tavily MCP 단일 의존

`musinsa-brief` Step 0(Preflight)에서 캐시 miss 시 Read해 사용. Tavily 1개만 의존하므로 wizard는 매우 단순.

## Inline Preflight 절차

메인 세션에서 직접 수행 (subagent 아님):

1. `ToolSearch(query="select:mcp__tavily__tavily_search,mcp__tavily__tavily_extract", max_results=2)` 호출
2. 두 도구 스키마 모두 반환 → 캐시 갱신 + Step 1 진행
3. 둘 중 하나라도 실패 → 아래 안내 메시지 출력 후 **즉시 halt** (메인 워크플로우 진입 금지)

## 캐시 파일 스키마

경로: `~/.claude/data/musinsa-brief/preflight.json`

```json
{
  "last_ok_at": "2026-04-25T07:30:00+09:00",
  "checks": {
    "tavily": "ok"
  }
}
```

- `last_ok_at` ISO 8601 KST
- 24h 이내 + `tavily == "ok"` → 캐시 hit, preflight skip
- 그 외 → cache miss, 위 절차 수행

## 갱신 시점

`musinsa-brief` Step 7(Preflight 캐시 갱신)에서 전체 실행 무결 완료 시 갱신.

## halt 시 출력 메시지 (사용자에게)

```
🛑 Musinsa Brief를 실행하려면 Tavily MCP가 등록돼 있어야 합니다.
본 스킬은 Tavily 외 별도 키가 필요 없습니다.

설치 절차:

1. Tavily 가입 (무료, 월 1,000 credits): https://tavily.com
2. API 키 발급 후 아래 명령으로 등록:

   claude mcp add tavily -e TAVILY_API_KEY=<your-key> -- npx -y tavily-mcp@latest

3. 등록 후 Claude Code 재시작 → 다시 호출

이미 설치돼 있는데 이 메시지가 보인다면:
- claude mcp list 로 등록 여부 확인
- 401 에러일 가능성: claude mcp remove tavily 후 재등록
```

## Welcome 안내 (default 경로 첫 사용 시)

§9.1 결정 절차 3단계: config.json 없음 + default 디렉토리 신규 생성 시 채팅 보고에 1회 한정 추가:

```
💡 보고서를 ~/Documents/obsidian/musinsa-brief/에 저장했어요. Obsidian vault 경로가 다르면 ~/.claude/data/musinsa-brief/config.json 생성 후 {"output_dir": "<경로>"} 작성하세요. 이 안내는 첫 호출에만 노출됩니다.
```

표시 후 `~/.claude/data/musinsa-brief/.welcomed` 빈 파일 touch해 재노출 방지.
```

- [ ] **Step 2: Verify file mentions Tavily setup commands**

```bash
grep -q "claude mcp add tavily" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/setup-wizard.md \
  && echo "✅ tavily install instruction present" || echo "❌ missing tavily install"
```
Expected: `✅ tavily install instruction present`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/references/setup-wizard.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add setup-wizard.md (Tavily-only preflight)"
```

---

### Task 10: `references/report-template.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/report-template.md`

- [ ] **Step 1: Write report-template.md**

Use the Write tool to create the file with this exact content:

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

- [ ] **Step 2: Verify all 3 mode sections + executive card present**

```bash
for s in "initial.* 모드" "refresh.* 모드" "since_last.* 모드" "Executive Card 옵션"; do
  grep -qE "$s" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/references/report-template.md \
    || { echo "❌ missing section: $s"; exit 1; }
done
echo "✅ all mode sections + executive card present"
```
Expected: `✅ all mode sections + executive card present`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/references/report-template.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add report-template.md (card-based report template + mode diffs)"
```

---

### Task 11: SKILL.md 본문 (실행 흐름 + 분석 룰)

**Files:**
- Modify: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md`

이 task에서 Task 4의 placeholder 본문을 실제 실행 흐름으로 교체.

- [ ] **Step 1: Replace SKILL.md body**

Use the Write tool (overwrites) to write `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md` with this full content (frontmatter unchanged from Task 4):

```markdown
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
```

- [ ] **Step 2: Validate frontmatter still parses + body has all 9 steps**

```bash
python3 -c "
import yaml
p = '/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md'
with open(p) as f:
    txt = f.read()
fm = txt.split('---', 2)[1]
data = yaml.safe_load(fm)
assert data['name'] == 'musinsa-brief'
print('✅ frontmatter ok')
"
for s in "Step 0" "Step 1" "Step 2" "Step 3" "Step 4" "Step 5" "Step 6" "Step 7" "Step 8" "Step 9"; do
  grep -q "### $s" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md \
    || { echo "❌ missing $s"; exit 1; }
done
echo "✅ all steps 0-9 present"
```
Expected: both `✅` lines.

- [ ] **Step 3: Verify all 6 references files are referenced from SKILL.md**

```bash
for ref in categories source-tiers query-templates competitors setup-wizard report-template; do
  grep -q "references/${ref}.md" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md \
    || { echo "❌ missing reference: $ref"; exit 1; }
done
echo "✅ all 6 references mentioned"
```
Expected: `✅ all 6 references mentioned`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-brief/SKILL.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): write musinsa-brief SKILL.md body (9-step execution flow)"
```

---

## Phase 3 — `musinsa-weekly-digest` Skill (Tasks 12-14)

### Task 12: Weekly digest SKILL.md frontmatter

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md`

- [ ] **Step 1: Create skill directory**

```bash
mkdir -p /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/references
```

- [ ] **Step 2: Write frontmatter (body in Task 14)**

Use the Write tool to create `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md`:

```markdown
---
name: musinsa-weekly-digest
description: musinsa-brief가 vault에 누적한 일간 보고서들을 주간 또는 월간 롤업 보고서로 합성합니다. vault 디렉토리 스캔 → 일간 보고서의 YAML frontmatter(report_date, mode, dominant_category, categories, sources, data_quality)만 파싱 → 카테고리 임팩트 분포·반복 테마(Trend Memory)·출처 매체 빈도·dominant 변화 추이 정리. 사용자가 "주간 무신사 정리", "이번 주 무신사 요약", "musinsa-weekly-digest", "이번 달 무신사", "월간 무신사", "지난 주 무신사 어땠어", "지난 달 무신사 정리" 같은 표현을 쓸 때 트리거하세요. 단일 일간 리포트는 musinsa-brief 사용, 이 skill은 **기간 집계**에만 사용합니다.
---

# Musinsa Weekly Digest

(본문은 Task 14에서 작성됩니다.)
```

- [ ] **Step 3: Validate frontmatter**

```bash
python3 -c "
import yaml
p = '/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md'
with open(p) as f:
    txt = f.read()
fm = txt.split('---', 2)[1]
data = yaml.safe_load(fm)
assert data['name'] == 'musinsa-weekly-digest'
assert '주간' in data['description'] and '월간' in data['description']
print('✅ digest frontmatter ok')
"
```
Expected: `✅ digest frontmatter ok`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-weekly-digest/SKILL.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add musinsa-weekly-digest SKILL.md frontmatter (body pending)"
```

---

### Task 13: `references/trend-memory.md`

**Files:**
- Create: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/references/trend-memory.md`

- [ ] **Step 1: Write trend-memory.md**

Use the Write tool to create the file with this exact content:

```markdown
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
```

- [ ] **Step 2: Verify trend memory rules present**

```bash
for k in "토큰화 절차" "Stopword 필터" "빈도 카운팅" "정렬·top 선택" "출력 스키마"; do
  grep -q "$k" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/references/trend-memory.md \
    || { echo "❌ missing: $k"; exit 1; }
done
echo "✅ trend memory rules complete"
```
Expected: `✅ trend memory rules complete`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-weekly-digest/references/trend-memory.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): add trend-memory.md (theme extraction rules)"
```

---

### Task 14: Weekly digest SKILL.md body

**Files:**
- Modify: `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md`

- [ ] **Step 1: Replace SKILL.md with full body**

Use the Write tool (overwrites) to write `/Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md` with this full content:

```markdown
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

1. `~/.claude/data/musinsa-brief/config.json`의 `output_dir` 확인 → 없으면 default `~/Documents/obsidian/musinsa-brief/`
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
```

- [ ] **Step 2: Validate body has all 6 steps + template + error table**

```bash
for s in "Step 0" "Step 1" "Step 2" "Step 3" "Step 4" "Step 5" "Step 6"; do
  grep -q "### $s" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md \
    || { echo "❌ missing $s"; exit 1; }
done
grep -q "## 출력 템플릿" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md && \
grep -q "## 에러 처리" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md && \
grep -q "trend-memory.md" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md && \
echo "✅ digest body complete"
```
Expected: `✅ digest body complete`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add skills/musinsa-weekly-digest/SKILL.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "feat(skill): write musinsa-weekly-digest SKILL.md body (frontmatter-only digest)"
```

---

## Phase 4 — Integration & Validation (Tasks 15-18)

### Task 15: Full README.md (replaces Task 3 placeholder)

**Files:**
- Modify: `/Users/wooksangryu/workspace/musinsa-brief-plugin/README.md`

- [ ] **Step 1: Overwrite README with full version**

Use the Write tool to overwrite `/Users/wooksangryu/workspace/musinsa-brief-plugin/README.md`:

```markdown
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

기본 저장 경로: `~/Documents/obsidian/musinsa-brief/{YYYY-MM-DD}.md`. vault 경로가 다르면 `~/.claude/data/musinsa-brief/config.json`에 `{"output_dir": "<경로>"}` 작성.

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
```

- [ ] **Step 2: Verify README has install + usage sections**

```bash
for s in "🚀 설치" "🎯 사용법" "📄 산출물" "🛠 문제 해결"; do
  grep -q "$s" /Users/wooksangryu/workspace/musinsa-brief-plugin/README.md \
    || { echo "❌ missing: $s"; exit 1; }
done
echo "✅ README sections complete"
```
Expected: `✅ README sections complete`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin add README.md
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin commit -m "docs(readme): replace placeholder with full v0.1.0 README"
```

---

### Task 16: Marketplace 등록

**Files:**
- Modify: `/Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json`

이 task는 별도 git 저장소(`wooksang-marketplace`)를 수정. 작업 전 해당 repo의 상태 확인.

- [ ] **Step 1: Inspect current marketplace.json**

```bash
cat /Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json
```

확인할 점:
- 최상위 키 구조 (보통 `{ "plugins": [ ... ] }` 형태)
- 기존 등록 plugin 항목 형태 (arsenal-brief-plugin / overnight-market-report-plugin)
- JSON 들여쓰기 스타일 (보통 2 spaces)

- [ ] **Step 2: Add musinsa-brief-plugin entry using jq**

기존 plugins 배열에 항목을 추가. arsenal/overnight 항목과 같은 키 구조 따라:

```bash
jq '.plugins += [{
  "name": "musinsa-brief-plugin",
  "source": { "source": "github", "repo": "today8934/musinsa-brief-plugin" },
  "description": "무신사 자사·경쟁사·입점·규제·인사·트렌드를 7 카테고리로 병렬 수집해 Obsidian vault에 한국어 보고서로 저장",
  "version": "0.1.0"
}]' /Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json > /tmp/marketplace.json.new \
  && mv /tmp/marketplace.json.new /Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json
```

만약 기존 marketplace.json이 `plugins` 배열 키를 안 쓰고 다른 구조라면, `cat` 출력에 따라 `jq` 표현식 조정. 예: 최상위가 곧 배열이면 `jq '. += [{...}]'`, `entries` 키면 `.entries += [...]`.

- [ ] **Step 3: Validate JSON + new entry present**

```bash
jq -e '.plugins | map(select(.name == "musinsa-brief-plugin")) | length == 1' \
  /Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json \
  && echo "✅ marketplace entry registered" || echo "❌ entry missing or schema differs"
```
Expected: `✅ marketplace entry registered`. 다른 키 이름이면 표현식 조정 후 재실행.

- [ ] **Step 4: Commit in marketplace repo**

```bash
git -C /Users/wooksangryu/workspace/wooksang-marketplace add .claude-plugin/marketplace.json
git -C /Users/wooksangryu/workspace/wooksang-marketplace commit -m "feat(marketplace): register musinsa-brief-plugin v0.1.0"
```

(Push는 사용자가 별도로 수행 — 자동 push 금지.)

---

### Task 17: 자체 검증 스크립트 + 실행

**Files:**
- (none new) — 일회성 검증, 결과만 확인

이 task는 plugin 전체 무결성을 한 번에 점검.

- [ ] **Step 1: 디렉토리 트리 검증**

```bash
cd /Users/wooksangryu/workspace/musinsa-brief-plugin

# 필수 파일 모두 존재
for f in \
  .claude-plugin/plugin.json \
  README.md LICENSE .gitignore \
  skills/musinsa-brief/SKILL.md \
  skills/musinsa-brief/references/categories.md \
  skills/musinsa-brief/references/source-tiers.md \
  skills/musinsa-brief/references/query-templates.md \
  skills/musinsa-brief/references/competitors.md \
  skills/musinsa-brief/references/setup-wizard.md \
  skills/musinsa-brief/references/report-template.md \
  skills/musinsa-weekly-digest/SKILL.md \
  skills/musinsa-weekly-digest/references/trend-memory.md \
  docs/superpowers/specs/2026-04-25-musinsa-brief-skill-design.md \
  docs/superpowers/plans/2026-04-25-musinsa-brief-skill.md
do
  [ -s "$f" ] && echo "✅ $f" || echo "❌ MISSING: $f"
done
```
Expected: 모든 라인 `✅`.

- [ ] **Step 2: plugin.json + 두 SKILL.md frontmatter 검증**

```bash
python3 << 'PY'
import json, yaml, sys

base = '/Users/wooksangryu/workspace/musinsa-brief-plugin'

# plugin.json
with open(f'{base}/.claude-plugin/plugin.json') as f:
    p = json.load(f)
assert p['name'] == 'musinsa-brief-plugin', f"name={p['name']}"
assert p['version'] == '0.1.0', f"version={p['version']}"
assert len(p['keywords']) >= 6, f"keywords={p['keywords']}"
print('✅ plugin.json ok')

# musinsa-brief frontmatter
with open(f'{base}/skills/musinsa-brief/SKILL.md') as f:
    txt = f.read()
fm = yaml.safe_load(txt.split('---', 2)[1])
assert fm['name'] == 'musinsa-brief'
assert '무신사' in fm['description'] and len(fm['description']) > 200
print('✅ musinsa-brief frontmatter ok')

# musinsa-weekly-digest frontmatter
with open(f'{base}/skills/musinsa-weekly-digest/SKILL.md') as f:
    txt = f.read()
fm = yaml.safe_load(txt.split('---', 2)[1])
assert fm['name'] == 'musinsa-weekly-digest'
assert '주간' in fm['description'] and '월간' in fm['description']
print('✅ musinsa-weekly-digest frontmatter ok')
PY
```
Expected: 3 lines all `✅`.

- [ ] **Step 3: SKILL.md → references 참조 무결성**

```bash
for ref in categories source-tiers query-templates competitors setup-wizard report-template; do
  grep -q "references/${ref}.md" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-brief/SKILL.md \
    || { echo "❌ musinsa-brief SKILL.md missing reference: $ref"; exit 1; }
done
grep -q "references/trend-memory.md" /Users/wooksangryu/workspace/musinsa-brief-plugin/skills/musinsa-weekly-digest/SKILL.md \
  || { echo "❌ digest SKILL.md missing trend-memory.md ref"; exit 1; }
echo "✅ all references resolved from SKILL.md"
```
Expected: `✅ all references resolved from SKILL.md`.

- [ ] **Step 4: Marketplace 등록 검증**

```bash
jq -e '.plugins | map(select(.name == "musinsa-brief-plugin")) | length == 1' \
  /Users/wooksangryu/workspace/wooksang-marketplace/.claude-plugin/marketplace.json \
  && echo "✅ marketplace registered" || echo "❌ marketplace entry missing"
```
Expected: `✅ marketplace registered`.

- [ ] **Step 5: Git 상태 클린 확인**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin status --short
git -C /Users/wooksangryu/workspace/wooksang-marketplace status --short
```
Expected: 두 출력 모두 빈 줄 (모든 변경 commit됨).

- [ ] **Step 6: Commit log 한 번 출력**

```bash
git -C /Users/wooksangryu/workspace/musinsa-brief-plugin log --oneline
```
Expected: 13~15개 commit (1 spec root commit + Tasks 1~15의 commit들). 모든 commit이 `docs:` / `chore:` / `feat:` prefix.

---

### Task 18: Claude 세션 dry-run 시나리오 (수동 통합 검증)

**Files:**
- (none) — 수동 검증, 결과 기록은 선택

이 task는 자동화할 수 없는 통합 검증. 사용자가 직접 새 Claude Code 세션에서 trigger를 입력해 동작 확인.

- [ ] **Step 1: Plugin 설치 (개발 모드)**

새 Claude Code 세션을 열고:

```
/plugin marketplace list
```

`wooksang-marketplace`가 보이는지 확인 (이미 등록돼 있어야 함). 없으면:

```
/plugin marketplace add /Users/wooksangryu/workspace/wooksang-marketplace
```

그 다음:

```
/plugin install musinsa-brief-plugin@wooksang-marketplace
```

- [ ] **Step 2: Tavily MCP 등록 확인**

```
claude mcp list
```

`tavily`가 보이지 않으면:

```
claude mcp add tavily -e TAVILY_API_KEY=<your-key> -- npx -y tavily-mcp@latest
```

Claude Code 재시작.

- [ ] **Step 3: 트리거 호출 (`initial` mode 시나리오)**

새 세션에서 vault 디렉토리 비어 있는 상태에서:

```
오늘 무신사
```

확인 사항:
- [ ] Step 0 preflight 통과 (Tavily 도구 로드 성공)
- [ ] Step 1 mode = `initial`로 결정 (직전 보고서 없음)
- [ ] Step 2 10개 쿼리 한 메시지 안 병렬 호출 (Claude 응답 흐름 관찰)
- [ ] Step 6 보고서 파일 생성 확인: `ls ~/Documents/obsidian/musinsa-brief/`
- [ ] Step 8 채팅에 경로 + TL;DR + 신호 카드 + (default 처음 사용 시) welcome 안내 출력
- [ ] 보고서 frontmatter에 `mode: initial`, `categories[]` 7개, `sources[]` 다수, `dominant_category` non-null

- [ ] **Step 4: 트리거 호출 (`refresh` mode 시나리오)**

방금 생성된 일간 보고서가 있는 상태에서 같은 세션 또는 새 세션에서:

```
무신사 다시 봐줘
```

확인 사항:
- [ ] Step 1 mode = `refresh` 결정
- [ ] 보고서 파일명이 `YYYY-MM-DD-HHMM.md` 형태로 시간 suffix
- [ ] frontmatter `mode: refresh`, `since_date == report_date == today`
- [ ] mode별 diff 섹션이 "## 🔁 오늘 신규 (vs 직전 보고서)" 헤더

- [ ] **Step 5: 트리거 호출 (`since_last` mode 시나리오)**

vault에 며칠 전 일간 보고서가 있는 상황(예: 4/22 보고서)에서 며칠 후(예: 4/25):

```
무신사 brief
```

확인 사항:
- [ ] Step 1 mode = `since_last`, `since_date == 2026-04-22`, `days_window == 3`
- [ ] mode별 diff 섹션이 "## 🔁 Since-last 변화 (since 2026-04-22)" 헤더
- [ ] diff 섹션에 신규/진척/종료 분류

- [ ] **Step 6: 주간 digest 호출**

일간 보고서가 3건 이상 누적된 상태에서:

```
주간 무신사 정리
```

확인 사항:
- [ ] vault 디렉토리 스캔 → 기간 내 일간 보고서 모두 frontmatter Read (본문 무시 — Claude의 Tool 호출 패턴 관찰)
- [ ] `digest-YYYY-W{WW}.md` 파일 생성
- [ ] Trend Memory 섹션이 N≥2일 등장 테마 추출
- [ ] 카테고리 임팩트 분포 표 + dominant 변화 추이 + 다음 기간 관전 포인트 모두 포함

- [ ] **Step 7: Executive Card 옵션 검증**

```
무신사 슬랙용으로
```

확인 사항:
- [ ] 일반 보고서 + Executive Card 3줄 모두 채팅에 출력
- [ ] vault에는 일반 보고서만 저장 (executive card는 별도 파일 생성 X)

- [ ] **Step 8: Sanity check #1 검증 (전면 0건 시뮬레이션)**

Tavily 키를 일시적으로 제거하거나 잘못된 키로 재등록 후:

```
오늘 무신사
```

확인 사항:
- [ ] Step 0 preflight halt 또는 Step 4 sanity #1 ⚠️ 배너
- [ ] 보고서 상단에 ⚠️ 배너 + 데이터 품질 푸터에 점검 안내

검증 후 원래 키로 복구.

- [ ] **Step 9: 결과 기록 (선택)**

위 시나리오들의 동작 결과를 README의 `## 🛠 문제 해결` 섹션에 추가 케이스로 commit (선택). 만약 이슈를 발견하면 spec/plan에 fix 메모 추가 후 별도 commit.

---

## Self-Review

### Spec Coverage 매핑

각 spec 섹션이 plan의 어느 task로 구현되는지 매핑:

| Spec § | 내용 | Plan Task |
|--------|------|-----------|
| §1 Purpose | 목적·대상 | Task 4 frontmatter (description), Task 11 SKILL.md "왜 이 skill이 필요한가" |
| §2 Scope (in/out) | 범위 | Task 4 description, Task 11 "금지 사항" |
| §3 Plugin Structure | 디렉토리 트리 | Task 1, 2, 4, 5-10, 12-13 (트리 단계별 생성) |
| §3.1 plugin.json | manifest | Task 2 |
| §3.2 marketplace 등록 | 등록 | Task 16 |
| §4 7 Category Matrix | 카테고리 정의 | Task 5 |
| §4.1 호출 수 | 10 쿼리/일 | Task 5 + Task 7 |
| §5 Source Tier | 매체 사전 | Task 6 |
| §6 Mode Paradigm | since_last_report | Task 11 Step 1 + report-template diff (Task 10) |
| §7 Execution Flow | 9 step | Task 11 본문 |
| §7 Step 9 Executive Card | 옵션 | Task 11 Step 9 + Task 10 "Executive Card 옵션" |
| §8 YAML Frontmatter 스키마 | 보고서 frontmatter | Task 10 report-template |
| §9 저장 경로 + Config | 결정 절차 | Task 11 "출력 위치" |
| §9.1 Default 사용 안내 | welcome 1회 | Task 9 setup-wizard.md + Task 11 Step 8 |
| §10 Output Template | 보고서 마크다운 | Task 10 |
| §11 musinsa-weekly-digest | 주간 digest | Task 12, 13, 14 |
| §11.4 Trend Memory 룰 | 키워드 추출 | Task 13 |
| §12 Trigger 키워드 | description 본문 | Task 4 (musinsa-brief), Task 12 (digest) |
| §13 Setup Wizard | Tavily 안내 | Task 9 |
| §14 Error Handling | 에러 표 | Task 11 "에러 처리", Task 14 "에러 처리" |
| §15 Success Criteria | 검증 기준 | Task 18 dry-run 시나리오 |
| §16 Non-Goals | 금지 | Task 11 "금지 사항", Task 14 "금지 사항" |
| §17 비용/한도 분석 | 비용 검증 | Task 5 호출 수 표 + README ✨주요 특징 |
| §18 Open Questions | future work | Task 15 README 미언급 (spec에서 충분), 차후 v0.2 |

**Gaps**: 없음. 모든 spec 섹션이 task로 매핑됨.

### Placeholder Scan

- [ ] "TBD", "TODO", "implement later", "fill in details" — 본 plan에 없음 (사용자 직접 결정해야 할 부분은 Task 16의 "기존 marketplace.json 구조 따라 jq 표현식 조정" 정도, 이건 즉시 결정 가능한 코드 수준 분기로 placeholder 아님)
- [ ] "Add appropriate error handling" / "handle edge cases" — 없음 (모든 에러 케이스가 표로 enumerate됨)
- [ ] "Similar to Task N" — 없음 (모든 코드/콘텐츠 본문이 plan에 그대로 박힘)

### Type Consistency

- 카테고리 id (`musinsa_ir`, `musinsa_campaign`, `musinsa_brands`, `competitors`, `regulation`, `org_hr`, `industry_trend`) — Tasks 5, 7, 10, 11에서 동일 표기 ✅
- mode 값 (`initial`, `refresh`, `since_last`) — Tasks 10, 11에서 동일 표기 ✅
- Tier (`T1`, `T2`, `T3`) — Tasks 6, 10, 11에서 동일 표기 ✅
- 임팩트 (`high`, `mid`, `low` + `●`, `◐`, `○`) — Tasks 10, 11에서 동일 표기 ✅
- 캐시 파일 경로 (`~/.claude/data/musinsa-brief/preflight.json`) — Task 9, Task 11에서 동일 ✅
- Config 경로 (`~/.claude/data/musinsa-brief/config.json`) — Task 11, Task 14에서 동일 ✅
- Welcome 마커 (`~/.claude/data/musinsa-brief/.welcomed`) — Task 9, Task 11에서 동일 ✅

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-04-25-musinsa-brief-skill.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — 각 task마다 fresh subagent 디스패치, task 사이 검토. 빠른 반복.
   - REQUIRED SUB-SKILL: `superpowers:subagent-driven-development`

**2. Inline Execution** — 현재 세션에서 직접 task 순차 실행, checkpoint마다 검토.
   - REQUIRED SUB-SKILL: `superpowers:executing-plans`

**Which approach?**
