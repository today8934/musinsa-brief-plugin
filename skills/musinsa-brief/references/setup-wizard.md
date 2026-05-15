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
💡 보고서를 ~/workspace/wooksang-marketplace-documents/musinsa-brief/에 저장했어요. 다른 위치(예: Obsidian vault)에 저장하려면 ~/.claude/data/musinsa-brief/config.json 생성 후 {"output_dir": "<경로>"} 작성하세요. 이 안내는 첫 호출에만 노출됩니다.
```

표시 후 `~/.claude/data/musinsa-brief/.welcomed` 빈 파일 touch해 재노출 방지.
