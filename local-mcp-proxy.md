# Local MCP Proxy: GitHub / Notion 호스팅 MCP 서버 프록시

`local-mcp-proxy.yaml`은 agentgateway를 로컬 모드로 실행해 GitHub와 Notion의 호스팅(원격) MCP 서버를 프록시하는 구성입니다. 2026-07-06에 실제 동작 검증을 마쳤습니다.

## 실행

```bash
cargo build --release --bin agentgateway
./target/release/agentgateway -f local-mcp-proxy.yaml
```

게이트웨이는 설정 파일을 watch 하므로 수정하면 재시작 없이 반영됩니다.

| 엔드포인트 | 업스트림 | 인증 |
|---|---|---|
| `http://localhost:3000/github/mcp` | `api.githubcopilot.com/mcp` | Bearer PAT 패스스루 (`gh auth token`의 OAuth 토큰도 동작) |
| `http://localhost:3000/notion/mcp` | `mcp.notion.com/mcp` | OAuth (게이트웨이가 인증 서버 프록시) |

클라이언트 등록 예 (Claude Code):

```bash
claude mcp add --transport http github http://localhost:3000/github/mcp \
  -H "Authorization: Bearer $(gh auth token)"
claude mcp add --transport http notion http://localhost:3000/notion/mcp
# notion은 세션에서 /mcp → Authenticate 로 브라우저 OAuth 진행
```

## 핵심 패턴 1: 토큰 기반 업스트림 — 헤더 패스스루

agentgateway의 MCP 백엔드는 클라이언트가 보낸 헤더(`Authorization` 포함)를 업스트림에 그대로 전달합니다
(`crates/agentgateway/src/mcp/upstream/mod.rs`의 `apply()` — `content-encoding`, `content-length`,
`mcp-session-id`만 제외). 따라서 GitHub처럼 PAT를 받는 서버는 라우트와 `mcp` 타깃 선언만으로 충분합니다.

## 핵심 패턴 2: OAuth 전용 업스트림 — 인증 서버 프록시 + resource 재작성

Notion(`mcp.notion.com`)처럼 OAuth만 지원하는 업스트림은 단순 패스스루로는 안 됩니다.
클라이언트 주도 OAuth가 프록시 뒤에서 깨지는 지점이 두 곳 있고, 각각 해결이 필요합니다.

### 문제 A: RFC 8707 resource 파라미터 검증

MCP 스펙상 클라이언트는 자신이 접속한 서버 URL(`http://localhost:3000/notion/mcp`)을
`resource` 파라미터로 보냅니다. Notion 인증 서버는 이를 엄격히 검증해서 모르는 resource면
`invalid_target`(400)으로 거부합니다. 실측 결과:

- `resource=http://localhost:3000/notion/mcp` → **400 invalid_target**
- `resource=https://mcp.notion.com/mcp` → 302 (정상)
- resource 생략 → 302 (정상)

**해결**: 게이트웨이가 인증 서버 경로(`/notion-as/*`)를 함께 노출하고, 통과하는 요청의
`resource`를 정식 값으로 재작성합니다.

- RFC 9728 protected-resource 메타데이터를 `directResponse`로 직접 서빙하고
  `authorization_servers`를 게이트웨이의 `/notion-as`로 지정
- RFC 8414 인증 서버 메타데이터도 `directResponse`로 서빙 (well-known 경로 4가지 변형 모두:
  `/.well-known/oauth-authorization-server/notion-as`, `/notion-as/.well-known/oauth-authorization-server`,
  openid-configuration 2종)
- `/notion-as/token`: 업스트림으로 프록시하되 form body의 resource를 CEL로 재작성 —
  `string(request.body).regexReplace("resource=[^&]*", "resource=https%3A%2F%2Fmcp.notion.com%2Fmcp")`
  (refresh token 갱신 요청에도 동일하게 적용됨)
- `/notion-as/register`: 동적 클라이언트 등록(DCR)은 재작성 없이 그대로 프록시
- `/notion/mcp`의 401 응답은 `www-authenticate`를 게이트웨이 메타데이터 URL로 조건부 재작성 —
  CEL 삼항식의 else 분기가 헤더 부재 시 평가 오류를 내면 해당 헤더 변경이 조용히 스킵되는
  동작을 이용해 401에만 적용

### 문제 B: 인증 서버의 상태 쿠키 (⚠️ /authorize는 프록시하면 안 됨)

Notion의 `/authorize`는 응답에 `Domain=.notion.com` 상태 쿠키를 설정하고, OAuth 콜백 단계
(`mcp.notion.com/callback`)에서 이 쿠키를 검증합니다. 브라우저가 `/authorize`를 게이트웨이
(localhost) 경유로 접속하면 cross-domain `Set-Cookie`가 버려져서 콜백에서
**"Invalid MCP state. Please enable browser cookies"** 오류가 납니다.

**해결**: `/notion-as/authorize`는 프록시가 아니라 **302 `directResponse` 리다이렉트**로 처리해서
브라우저가 `mcp.notion.com`에 직접 접속하게 합니다. resource 재작성은 리다이렉트 URL에 반영합니다:

```yaml
directResponse:
  status: 302
  headers:
    location: '"https://mcp.notion.com" + request.pathAndQuery.setQuery("resource", "https://mcp.notion.com/mcp").replace("/notion-as/", "/")'
```

토큰 엔드포인트는 브라우저가 아닌 MCP 클라이언트가 직접 POST 하므로(쿠키 무관) 프록시를 유지해도 됩니다.

### 완성된 플로우

```
클라이언트 → GET /notion/mcp (無인증)
  ← 401 + www-authenticate: …localhost:3000/.well-known/oauth-protected-resource/notion/mcp
클라이언트 → 게이트웨이에서 PRM/AS 메타데이터 디스커버리
클라이언트 → POST /notion-as/register (DCR, 프록시)
브라우저   → GET /notion-as/authorize → 302 → mcp.notion.com/authorize (직접 접속, 쿠키 정상)
           → 로그인/동의 → mcp.notion.com/callback → 클라이언트 로컬 콜백에 code 전달
클라이언트 → POST /notion-as/token (프록시, body의 resource 재작성) → 토큰 발급
클라이언트 → /notion/mcp + Bearer 토큰 → 게이트웨이가 업스트림에 그대로 전달
```

## 다음 단계 후보: mcpAuthorization 인가 정책 (미적용)

라우트에 `mcpAuthorization` 정책을 얹어 도구 수준 접근 제어를 걸 수 있습니다. CEL 규칙 기반이며
평가 순서는 (`crates/agentgateway/src/http/authorization.rs`의 `validate()`):

1. `deny` 규칙이 하나라도 참 → 거부
2. `require` 규칙이 있으면 전부 참이어야 함
3. `allow` 규칙이 하나라도 참 → 허용
4. 매칭 없음: allow 규칙이 하나도 없으면 기본 허용(denylist 모드), 있으면 기본 거부(allowlist 모드)

문자열만 쓰면 allow 규칙입니다. `tools/list` 응답도 자동 필터링되어 차단된 도구는 클라이언트
목록에서 아예 사라집니다.

규칙에서 사용 가능한 컨텍스트: `mcp.tool.name` / `mcp.tool.target` / `mcp.tool.arguments`,
`mcp.methodName`, `mcp.resource.*`, `mcp.prompt.*`, `request.headers`, `source.address`.
`jwt.*` 클레임 규칙은 별도 `jwtAuth` 정책이 필요해서 opaque 토큰을 쓰는 현재 구성에는 해당 없음.

테스트 후보 정책:

```yaml
# 1. 읽기 전용 allowlist (적용 즉시 Notion 도구 목록이 20개 → 4개로 줄어 효과 확인 쉬움)
mcpAuthorization:
  rules:
  - 'mcp.tool.name in ["notion-search", "notion-fetch", "notion-get-users", "notion-get-teams"]'

# 2. 파괴적 도구만 차단 (denylist)
mcpAuthorization:
  rules:
  - deny: 'mcp.tool.name.startsWith("notion-create") || mcp.tool.name.startsWith("notion-update") || mcp.tool.name == "notion-move-pages"'

# 3. 인자 기반 — GitHub 쓰기 도구를 본인 리포로 제한
mcpAuthorization:
  rules:
  - deny: 'mcp.tool.name in ["create_or_update_file", "create_pull_request", "delete_file"] && mcp.tool.arguments.owner != "kyunghoj"'

# 4. 요청 속성 기반 — 로컬 접속만 허용
mcpAuthorization:
  rules:
  - require: 'cidr("127.0.0.1/8").containsIP(source.address) || cidr("::1/128").containsIP(source.address)'
```

참고 예제: `examples/mcp-authorization/` (README는 Cedar 스타일 `permit(...)` 문법을 보여주지만
config.yaml은 순수 CEL 불리언 규칙 — 둘 다 지원됨).

## 기타 참고

- MCP 백엔드는 라우트에 매칭된 어떤 경로든 streamable HTTP로 서빙합니다 (`/sse`만 legacy SSE).
- CEL 변환식이 평가 오류를 내면 해당 헤더 변경은 조용히 스킵됩니다 (조건부 변환에 활용 가능).
- HTTPS 업스트림으로의 일반 HTTP 프록시는 `backends: [host: <host>:443]` + `backendTLS: {}` +
  `urlRewrite.authority.full`(Host 헤더 재작성) 조합으로 구성합니다.
