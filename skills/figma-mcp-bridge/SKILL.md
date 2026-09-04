---
name: figma-mcp-bridge
description: Figma MCP 미승인 클라이언트(opencode, omo 등)에서 승인된 CLI(Claude Code, Codex, Gemini CLI)를 헤드리스로 호출해 Figma MCP 도구에 접근하는 스킬. Figma 디자인 파일 조회, 컴포넌트 구조 파악, 디자인 컨텍스트/디자인 토큰 추출이 필요할 때 사용한다. Triggers - figma.com/design URL, Figma 링크, get_design_context, design tokens, Figma variables, 피그마.
---

# Figma MCP Bridge

## 사전 확인 (최우선)

브릿지를 실행하기 **전에**, 현재 환경에서 Figma MCP 도구(`mcp__figma__get_design_context` 등)를 직접 호출할 수 있는지 확인한다.

- **직접 접근 가능** → 브릿지를 쓰지 않고 네이티브 Figma MCP 도구를 호출한다.
- **직접 접근 불가** → 아래 절차를 따른다.

## 언제 이 스킬을 사용할지

- Figma 컴포넌트의 디자인 컨텍스트(레이아웃, 색상, 타이포그래피 등)를 추출해야 할 때
- Figma 디자인을 코드로 변환하기 위한 정보가 필요할 때
- Figma Variables(디자인 토큰)를 조회해야 할 때

다음 상황에서는 **Figma REST API를 직접 사용**한다 (MCP 호출 한도를 소비하지 않음):

- 단순히 파일의 페이지 목록만 필요할 때
- 대량의 노드를 순회해야 할 때

## 동작 원리

Figma 원격 MCP 서버(`https://mcp.figma.com/mcp`)는 [Figma MCP Catalog](https://www.figma.com/mcp-catalog/)에 등록된 클라이언트에서만 OAuth 인증을 허용하며, Personal Access Token(`figd_…`)은 헤더를 어떻게 넣어도 401로 거절된다 (Figma 공식 답변: PAT 인증은 지원하지 않고 활성화할 수도 없음). 이 스킬은 승인된 CLI를 **헤드리스(비대화형) 모드로 한 번 실행**해서 프롬프트를 넘기고, 표준 출력으로 도구 결과를 받아온다.

- tmux나 TUI 스크래핑이 필요 없다. 승인 다이얼로그, idle 프롬프트 감지, 폴링이 모두 사라진다.
- 호출마다 CLI 부팅 + MCP 핸드셰이크가 붙는다 (Claude Code 기준 약 7~12초).
- 대화 컨텍스트는 호출 간에 유지되지 않는다. 필요하면 백엔드의 세션 재개 기능(`--resume` 등)을 쓴다.

## 백엔드 선택

### 자동 감지 (사용자에게 묻지 않음)

"설치됨"이 아니라 **"Figma MCP가 설정되어 있음"**을 기준으로 감지한다. 설정된 백엔드가 하나면 그대로 쓴다.

```bash
# Claude Code: 서버 이름과 연결 상태가 함께 나온다
which claude >/dev/null 2>&1 && claude mcp list 2>/dev/null | grep -i figma
# Codex
grep -q 'mcp_servers.figma' ~/.codex/config.toml 2>/dev/null && echo "codex: figma configured"
# Gemini CLI
grep -q '"figma"' ~/.gemini/settings.json 2>/dev/null && echo "gemini: figma configured"
```

우선순위: **Claude Code → Codex → Gemini CLI**. Claude Code는 이 스킬의 헤드리스 절차가 실제로 검증된 백엔드다.

한 번 결정한 백엔드는 에이전트 메모리나 프로젝트 노트에 기록해 두고, 다음 세션에서는 재감지만 하고 다시 묻지 않는다.

### 설정된 백엔드가 없을 때만 묻는다

> "Figma MCP 브릿지로 사용할 CLI가 설정되어 있지 않습니다. 구독 중인 서비스가 있나요?"
>
> 1. **Anthropic (Claude Pro/Max)** → Claude Code 설치 후 사용
> 2. **OpenAI (ChatGPT Plus/Pro)** → Codex 설치 후 사용
> 3. **Google (Gemini)** → Gemini CLI 설치 후 사용
> 4. **없음** → Gemini CLI 설치 후 사용 (Google API 키 무료 티어로 사용 가능)

| 백엔드 | 패키지 | 필요 구독 | 헤드리스 검증 |
|---|---|---|---|
| Claude Code | `@anthropic-ai/claude-code` | Anthropic 유료 (Pro/Max) 또는 API 크레딧 | 검증됨 |
| Codex | `@openai/codex` | OpenAI 유료 (Plus/Pro) 또는 API 크레딧 | 미검증 |
| Gemini CLI | `@google/gemini-cli` | Google API 키 (유료 구독 또는 무료 티어) | 미검증 |

## 전제 조건

### Claude Code

- Figma MCP OAuth 토큰은 macOS에서는 Keychain의 `Claude Code-credentials`, Linux에서는 `~/.claude/.credentials.json`에 저장된다 (Linux 경로는 미검증). 이 스킬의 절차는 macOS에서 검증됐다.

Figma MCP를 등록하는 방법은 두 가지이며, **어느 쪽이든 서버 이름이 달라진다.** 브릿지 실행 시 이 이름을 그대로 써야 한다.

| 등록 방식 | 명령 | `claude mcp list`에 표시되는 이름 | 도구 접두사 |
|---|---|---|---|
| 공식 플러그인 | `claude` 실행 → `/plugin install figma@claude-plugins-official` | `plugin:figma:figma` | `mcp__plugin_figma_figma__` |
| 수동 등록 | `claude mcp add --transport http --scope user figma https://mcp.figma.com/mcp` | `figma` | `mcp__figma__` |

등록 후 인증: `claude` 실행 → `/mcp` → figma 선택 → `Authenticate` → 브라우저에서 Figma 로그인.

인증 상태 확인:

```bash
claude mcp list 2>/dev/null | grep -i figma
# 예: plugin:figma:figma: https://mcp.figma.com/mcp (HTTP) - ✔ Connected
# 예: figma: https://mcp.figma.com/mcp (HTTP) - ! Needs authentication
```

`Needs authentication`이면 브릿지를 실행하지 말고 사용자에게 위 인증 절차를 안내한다. 헤드리스 세션에서는 OAuth 플로우가 실행되지 않는다.

### Codex

1. `npm i -g @openai/codex` 후 `codex login`
2. `~/.codex/config.toml`에 추가:
   ```toml
   [mcp_servers.figma]
   url = "https://mcp.figma.com/mcp"
   ```
3. `codex` 대화형 실행에서 첫 연결 시 브라우저로 Figma 로그인

### Gemini CLI

1. `npm i -g @google/gemini-cli` 후 `export GEMINI_API_KEY="your-key"` 또는 `gemini` 실행 → "Sign in with Google"
2. `~/.gemini/settings.json`에 추가:
   ```json
   { "mcpServers": { "figma": { "httpUrl": "https://mcp.figma.com/mcp" } } }
   ```
3. `gemini` 대화형 실행에서 첫 연결 시 브라우저로 Figma 로그인

## Figma URL에서 fileKey 추출

일반 URL:
```
https://www.figma.com/design/<FILE_KEY>/<FILE_NAME>?node-id=<NODE_ID>
```

브랜치 URL (이 경우 `BRANCH_KEY`를 fileKey로 사용):
```
https://www.figma.com/design/<FILE_KEY>/branch/<BRANCH_KEY>/<FILE_NAME>
```

- `fileKey`: `/design/` 바로 뒤의 세그먼트. 브랜치 URL이면 `/branch/` 뒤의 세그먼트
- `nodeId`: `node-id` 파라미터의 값. `-`를 `:`로 변환 (예: `1-2` → `1:2`)

## 브릿지 실행 절차 (Claude Code)

### Step 1: 서버 이름 확인 및 MCP 설정 파일 생성

`claude -p`(헤드리스)는 플러그인으로 등록된 MCP 서버를 자동으로 로드하지 않는다. 그래서 `--mcp-config`로 서버를 명시해야 하는데, **OAuth 토큰은 서버 이름에 묶여 있으므로** `claude mcp list`에 표시된 이름을 그대로 키로 써야 기존 인증이 재사용된다.

토큰 조회 키는 서버 이름과 **서버 설정(URL, headers 등)의 해시**로 구성된다. 플러그인 등록의 경우 플러그인 `.mcp.json`에 `X-Figma-Plugin-Bundle` 헤더가 들어 있으므로, URL만 적은 설정으로는 이름이 같아도 토큰을 찾지 못한다. 플러그인의 `.mcp.json`에서 서버 정의를 그대로 복사하고 키 이름만 바꾼다.

```bash
FIGMA_SERVER=$(claude mcp list 2>/dev/null | grep -i 'mcp.figma.com' | head -1 | sed -E 's/: +https?:\/\/.*$//')
# 플러그인이면 "plugin:figma:figma", 수동 등록이면 "figma"
[ -z "$FIGMA_SERVER" ] && { echo "figma MCP not registered in claude"; exit 1; }

FIGMA_TOOL_PREFIX="mcp__$(echo "$FIGMA_SERVER" | tr -c 'A-Za-z0-9\n' '_')"
# "plugin:figma:figma" -> "mcp__plugin_figma_figma", "figma" -> "mcp__figma"

BRIDGE_DIR=$(mktemp -d /tmp/figma-bridge.XXXXXX)   # 고정 경로를 쓰면 병렬 실행 시 충돌한다
python3 - "$FIGMA_SERVER" > "$BRIDGE_DIR/mcp.json" <<'EOF'
import json, os, sys
name = sys.argv[1]
if name.startswith("plugin:"):
    # 활성 플러그인 경로는 installed_plugins.json에서 읽는다 (cache/ 아래에 옛 버전이 남아 있을 수 있음)
    reg = json.load(open(os.path.expanduser("~/.claude/plugins/installed_plugins.json")))["plugins"]
    key = next(k for k in reg if k.startswith("figma@"))
    src = json.load(open(os.path.join(reg[key][0]["installPath"], ".mcp.json")))["mcpServers"]["figma"]
    src.pop("_meta", None)
else:
    src = {"type": "http", "url": "https://mcp.figma.com/mcp"}
print(json.dumps({"mcpServers": {name: src}}))
EOF
```

감지가 불안하면 `claude mcp list` 출력을 직접 보고 `FIGMA_SERVER`를 손으로 지정한다.

**주의 — needs-auth 캐시**: `~/.claude/mcp-needs-auth-cache.json`은 서버 **이름** 기준으로 "인증 필요" 결과를 캐시한다. 잘못된 설정(헤더 누락, 옛 플러그인 버전 등)으로 한 번이라도 실패하면 그 뒤로는 올바른 설정을 넘겨도 인증을 시도하지 않고 바로 needs-auth로 끝난다. `claude mcp list`는 `Connected`인데 `-p`에서만 "not authenticated"가 나오면 이 캐시가 원인이다. 해당 이름의 항목을 지우고 다시 실행한다:

```bash
python3 - "$FIGMA_SERVER" <<'EOF'
import json, os, sys
p = os.path.expanduser("~/.claude/mcp-needs-auth-cache.json")
d = json.load(open(p)) if os.path.exists(p) else {}
d.pop(sys.argv[1], None)
json.dump(d, open(p, "w"))
EOF
```

### Step 2: 헤드리스 호출

```bash
claude -p "<프롬프트>" \
  --mcp-config "$BRIDGE_DIR/mcp.json" --strict-mcp-config \
  --allowedTools "$FIGMA_TOOL_PREFIX" \
  --output-format text > "$BRIDGE_DIR/out.md"
```

- `--strict-mcp-config`: 다른 MCP 서버(claude.ai 커넥터 등)를 로드하지 않아 부팅이 빨라지고 엉뚱한 도구를 부를 여지가 없다.
- `--allowedTools "$FIGMA_TOOL_PREFIX"`: 서버 단위 허용. 개별 도구만 허용하려면 `"${FIGMA_TOOL_PREFIX}__get_metadata"` 형식으로 나열한다. 헤드리스 모드에서 허용 목록 밖의 도구(Write, Edit, Bash 등)는 자동 거부되므로 브릿지가 코드를 건드리지 못한다.
- 출력은 파일로 받고 `read`로 읽는다. 터미널 폭 제한이나 잘림이 없다.
- 응답 구조가 필요하면 `--output-format json`을 쓴다. `result`(본문), `session_id`(재개용), `permission_denials`(거부된 도구 호출), `total_cost_usd`가 들어 있다.
- **결과 본문을 반드시 검사한다.** Figma 쪽 에러(호출 한도, 권한 없음 등)는 브릿지가 정상 종료한 채로 `result` 안에 `<error>…</error>`로 실려 온다. exit code나 `permission_denials`로는 잡히지 않는다:

  ```bash
  grep -q '<error>' "$BRIDGE_DIR/out.md" && { echo "figma tool error:"; grep -o '<error>.*' "$BRIDGE_DIR/out.md"; }
  ```

**대기 방식**: 명령이 끝날 때까지 그냥 기다린다. 폴링하지 않는다.

- 셸 도구의 타임아웃은 300초로 준다. `generate_figma_design`처럼 오래 걸리는 도구는 600초.
- 백그라운드 세션/모니터를 제공하는 하네스(omo 등)에서는 포그라운드로 실행하면 완료 알림이 오므로 별도 처리가 필요 없다.

### Step 3: 연속 질의

같은 파일에 대해 후속 질문을 이어가려면 json 출력의 `session_id`를 `--resume`으로 넘긴다. 부팅 비용은 그대로지만 앞선 도구 결과를 브릿지가 기억한다.

```bash
claude -p "<후속 프롬프트>" --resume <SESSION_ID> \
  --mcp-config "$BRIDGE_DIR/mcp.json" --strict-mcp-config \
  --allowedTools "$FIGMA_TOOL_PREFIX" --output-format json
```

### 프롬프트 작성 규칙

브릿지는 LLM이므로 그대로 두면 도구 결과를 요약·번역하면서 수치와 토큰 이름을 잃는다. 해석은 호출자(이 에이전트)가 하고, 브릿지에는 **원본을 그대로 넘기라고** 지시한다.

- `fileKey`, 있으면 `nodeId`를 반드시 포함한다.
- 호출할 도구 이름을 명시한다 (`get_metadata`, `get_design_context` 등 짧은 이름으로 충분).
- 다음 문장을 붙인다: `Print the raw tool result verbatim, with no summary or commentary. Do not call any other tool.`
- 언어 지시("Respond in Korean")는 넣지 않는다.

## 브릿지 실행 절차 (Codex / Gemini CLI, 미검증)

동일한 원칙으로 각 CLI의 헤드리스 모드를 쓴다. 실측하지 않은 명령이므로 첫 사용 시 `--help`로 플래그를 확인한다.

| 백엔드 | 헤드리스 명령 |
|---|---|
| Codex | `codex exec --full-auto "<프롬프트>" > "$BRIDGE_DIR/out.md"` |
| Gemini CLI | `gemini -p "<프롬프트>" --approval-mode yolo > "$BRIDGE_DIR/out.md"` |

## 대화형 fallback (tmux)

헤드리스 모드가 동작하지 않는 백엔드나, 첫 연결 시 브라우저 OAuth가 꼭 대화형 세션 안에서 떠야 하는 경우에만 쓴다. 하네스가 PTY 세션 도구(백그라운드 셸 + 키 입력 + 화면 캡처)를 제공하면 tmux 대신 그것을 쓴다.

```bash
tmux new-session -d -s figma-bridge -x 200 -y 50
tmux send-keys -t figma-bridge "claude --allowedTools '$FIGMA_TOOL_PREFIX'" Enter   # 또는 codex / gemini
tmux send-keys -t figma-bridge "<프롬프트>" Enter
# 완료 후
tmux capture-pane -t figma-bridge -p -S -2000 > "$BRIDGE_DIR/out.md"
tmux kill-session -t figma-bridge
```

주의:
- 완료 여부를 프롬프트 문자로 판단할 때 `>`를 단독으로 grep하지 않는다. `get_design_context` 결과의 JSX에도 `>`가 들어 있어 응답 도중에 매치된다. 백엔드별로 줄 시작에 앵커한 패턴(`^❯`, `^›`, `^> $`)을 쓴다.
- Claude Code TUI는 작업 중에도 하단 입력창을 항상 그리므로 프롬프트 문자만으로는 idle을 확정할 수 없다. 스피너 줄이 사라졌는지 함께 본다.
- 세션 이름이 고정되어 있어 병렬 실행이 충돌한다.

## MCP 호출 제한

- 호출 한도는 Figma 플랜과 **시트 종류**로 정해진다. 초과 시 도구 결과 본문에 `<error>You've reached the Figma MCP tool call limit for your <seat> seat on the <plan> plan…</error>`가 실려 온다 (Step 2의 결과 검사로 잡는다).
- **View 시트는 한도가 매우 낮아 몇 번 만에 막힌다.** 첫 브릿지 호출은 `whoami`로 하고, 결과의 `plans[].seat`가 `View`뿐이면 파일 구조 조회는 REST API를 우선하고 MCP는 `get_design_context`/`get_variable_defs`처럼 REST로 대체 불가능한 호출에만 쓴다.
- **제한 도달 시**: MCP 사용을 중단하고 아래 REST API로 전환한다. REST API에는 별도의 Figma Personal Access Token(`FIGMA_PAT`)이 필요하다.
- `get_design_context`는 스크린샷을 포함해 비용이 높다. `excludeScreenshot: true`로 줄일 수 있다.
- `whoami`, `add_code_connect_map`, `generate_figma_design`은 호출 제한을 소비하지 않는다. 브릿지 동작 확인은 `whoami`로 한다.

## 인증 만료 시

MCP 호출이 인증 에러(`401`, `Authentication failed`, "not authenticated" 등)를 반환하면 먼저 `claude mcp list`로 상태를 본다.

- `Needs authentication` → OAuth 토큰이 만료된 것. 아래 표대로 재인증한다.
- `Connected`인데 헤드리스에서만 실패 → 토큰은 살아 있다. Step 1의 needs-auth 캐시를 지우고, 설정 파일이 플러그인 `.mcp.json`과 같은 헤더를 갖는지 확인한다.

| 백엔드 | 재인증 방법 |
|---|---|
| Claude Code | `claude` 실행 → `/mcp` → figma → `Authenticate` → 브라우저 로그인 |
| Codex | `codex login` 후 대화형 세션에서 Figma MCP 재연결 |
| Gemini CLI | `gemini` 실행 후 재인증 또는 API 키 갱신 |

## Figma MCP 도구 목록

| 도구 | 용도 | 비고 |
|------|------|------|
| `get_design_context` | React+Tailwind 기반 디자인→코드 변환 | **가장 많이 사용**. 비용 높음 |
| `get_metadata` | 레이어 구조 메타데이터 (ID, 이름, 위치, 크기) | 가벼운 사전 조회용 |
| `get_screenshot` | 선택 영역 스크린샷 캡처 | 시각적 레퍼런스용 |
| `get_variable_defs` | 디자인 토큰 (색상, 간격, 타이포그래피) | 변수/스타일 조회 |
| `get_libraries` | 파일에 연결된 라이브러리 목록 | |
| `search_design_system` | 디자인 시스템 컴포넌트/스타일 검색 | |
| `get_figjam` | FigJam 다이어그램 → XML + 스크린샷 | FigJam 전용 |
| `generate_diagram` | Mermaid 구문 → FigJam 다이어그램 생성 | 파일 컨텍스트 불필요 |
| `get_code_connect_map` | Figma↔코드 컴포넌트 매핑 조회 | Code Connect |
| `add_code_connect_map` | Figma↔코드 매핑 추가 | 호출 제한 없음 |
| `get_context_for_code_connect` | Code Connect 작성용 컨텍스트 | Code Connect |
| `get_code_connect_suggestions` | Code Connect 매핑 제안 | Figma 주도 호출 |
| `send_code_connect_mappings` | 매핑 확인/확정 | Figma 주도 호출 |
| `create_design_system_rules` | 디자인 시스템 rules 파일 생성 | 1회성 설정 |
| `whoami` | 인증된 사용자 정보 조회 | 호출 제한 없음. 브릿지 동작 확인용 |
| `generate_figma_design` | 웹앱 UI → Figma 디자인 캡처 | 호출 제한 없음. 오래 걸림 |
| `create_new_file`, `use_figma`, `upload_assets`, `download_assets` | 파일 생성/조작, 에셋 업다운로드 | 쓰기 계열. 브릿지에서는 필요할 때만 개별 허용 |

## 프롬프트 예시

### 파일 전체 구조 파악

```
Call get_metadata for node <PAGE_NODE_ID> in file <FILE_KEY>.
Print the raw tool result verbatim, with no summary or commentary. Do not call any other tool.
```

페이지 노드 id는 URL의 `node-id` 값(예: `0-1` → `0:1`)이거나 REST API 페이지 목록 조회로 얻는다. nodeId를 생략한 호출은 검증되지 않았다.

### 특정 노드의 디자인 컨텍스트 추출

```
Call get_design_context for node 32:3 in file <FILE_KEY> with excludeScreenshot: true.
Print the raw tool result verbatim, with no summary or commentary. Do not call any other tool.
```

### Variables(디자인 토큰) 조회

```
Call get_variable_defs for node <NODE_ID> in file <FILE_KEY>.
Print the raw tool result verbatim, with no summary or commentary. Do not call any other tool.
```

## Figma REST API (MCP 대안)

MCP 호출 한도를 아끼거나, 단순 조회 시에는 REST API를 직접 사용한다.

`FIGMA_PAT`은 환경 변수를 우선 쓰고, 없을 때만 `.env` 파일에서 읽는다. 값을 감싼 따옴표는 벗긴다 (`FIGMA_PAT="figd_…"` 형태가 흔하고, 따옴표가 남으면 401이 난다):

```bash
ENV_FILE=${ENV_FILE:-.env}   # 다른 프로젝트의 .env를 쓰려면 경로를 지정
[ -z "$FIGMA_PAT" ] && [ -f "$ENV_FILE" ] && export FIGMA_PAT=$(grep '^FIGMA_PAT=' "$ENV_FILE" | cut -d= -f2- | tr -d '"'"'"' ')
: "${FIGMA_PAT:?FIGMA_PAT is not set}"
```

### 페이지 목록 조회

```bash
curl -s -H "X-Figma-Token: $FIGMA_PAT" \
  "https://api.figma.com/v1/files/<FILE_KEY>?depth=1" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for page in data['document']['children']:
    print(f'[{page[\"id\"]}] {page[\"name\"]}')
"
```

### 특정 페이지의 하위 노드 조회

```bash
curl -s -H "X-Figma-Token: $FIGMA_PAT" \
  "https://api.figma.com/v1/files/<FILE_KEY>/nodes?ids=<NODE_ID>&depth=2" | python3 -c "
import sys, json
data = json.load(sys.stdin)
for nid, node_data in data.get('nodes', {}).items():
    doc = node_data.get('document', {})
    print(f'Page: {doc.get(\"name\")}')
    for child in doc.get('children', []):
        ctype = child.get('type', '?')
        cname = child.get('name', '?')
        count = len(child.get('children', []))
        print(f'  - [{ctype}] {cname} ({count} children)')
"
```
