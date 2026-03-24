---
name: opencode-session-title
description: 사용자가 "이 세션의 제목을 업데이트해줘"처럼 요청했을 때, 현재 대화 문맥을 기반으로 세션 제목을 자동 생성하고 opencode 서버 API로 실제 제목을 갱신/검증하는 스킬.
---

# OpenCode Session Title Updater

## 목표

사용자가 세션 제목 변경을 요청하면, 에이전트가 다음을 자동으로 수행한다.

1. 현재 연결된 opencode 서버 URL을 확보한다.
2. 현재 세션 ID를 확보한다.
3. 최근 대화 문맥에서 제목 후보를 생성한다.
4. `PATCH /session/:id`로 제목을 실제 업데이트한다.
5. `GET /session/:id` 재조회로 변경을 검증한다.

핵심 요구사항:

- 사용자에게 `base_url` 또는 포트를 직접 묻지 않는다.
- 플랫폼(OS)별 명령(`lsof`, `netstat`, `ss`)에 의존하지 않는다.
- 실패 시 원인과 재시도 가능한 다음 액션을 명확히 반환한다.

## 언제 사용할지

다음 요청에서 이 스킬을 사용한다.

- "이 세션 제목 바꿔줘"
- "이 대화 제목 업데이트해줘"
- "현재 세션 이름 좀 정리해줘"
- "문맥에 맞게 세션 타이틀 다시 지어줘"

다음 경우에도 사용 가능:

- 제목이 기본값(`New session - ...`, `Child session - ...`)이라 자동 정리가 필요할 때
- 긴 작업 후 요약형 제목으로 바꾸고 싶을 때

## 입력 계약

스킬 입력은 다음 필드를 허용한다.

```json
{
  "instruction": "optional string",
  "style": "optional string: concise | descriptive",
  "max_length": "optional number, default 60"
}
```

제약:

- `instruction`이 없으면 최근 대화 주제 기반으로 생성한다.
- `max_length` 기본값은 60, 최대 80을 넘기지 않는다.

## Endpoint Resolver (Platform-Agnostic)

세션 수정 API 호출 전에 서버 URL을 아래 순서로 찾는다.

1. **호스트 런타임 제공 URL** 사용 (가장 우선)
   - Desktop/Tauri 등 호스트가 sidecar URL을 전달하는 경우 해당 URL 사용
2. **저장된 기본 서버 URL** 사용
   - 호스트 설정(`default server url`)이 있으면 사용
3. **최근 성공 URL 캐시** 사용
   - 이전 호출에서 health check 통과한 URL
4. **환경변수 URL** 사용 (선택)
   - `OPENCODE_BASE_URL` 등 런타임 표준 변수

모든 URL 후보는 반드시 `GET /global/health` 성공으로 검증 후 채택한다.

주의:

- OS 프로세스 스캔 기반 포트 탐지 금지
- 임의 포트 브루트포스 금지

## 세션 식별 규칙

현재 세션 ID를 다음 순서로 확보한다.

1. 런타임 컨텍스트가 제공한 `session_id`
2. 현재 대화 컨텍스트의 active session id
3. 디렉토리 매칭 fallback (API): `GET /session?limit=20`으로 최근 세션 목록을 조회한 뒤, 현재 작업 디렉토리(`cwd`)와 `directory` 필드가 일치하는 세션 중 가장 최근 것을 사용
4. 워크스페이스 상태 파일 fallback: API 목록에 현재 활성 세션이 누락될 수 있으므로, 워크스페이스 상태 파일에서 세션 ID를 추출하여 `GET /session/:id`로 디렉토리를 검증한 뒤 사용 (상세 절차는 아래 참조)
5. 마지막 fallback: 위 모든 방법이 실패하면 `GET /session?limit=1`로 가장 최근 세션 사용. 단, 이 경우 반드시 사용자에게 확인을 구한다.

세션 확보 경로별 `session_resolution` 값:

| 경로 | `session_resolution` 값 |
|------|------------------------|
| 1번 | `runtime` |
| 2번 | `active_context` |
| 3번 | `fallback_directory_match` |
| 4번 | `fallback_workspace_state` |
| 5번 | `fallback_latest` |

### 워크스페이스 상태 파일 탐색 절차 (4번 fallback)

현재 활성 세션은 `GET /session?limit=N` API 목록에 포함되지 않을 수 있다. 이 경우 워크스페이스 상태 파일에서 세션 ID를 추출한다.

**상태 파일 위치:**

환경변수 `XDG_STATE_HOME`이 가리키는 디렉토리 아래 `opencode.workspace.*.dat` 파일들.

```
${XDG_STATE_HOME}/opencode.workspace.*.dat
```

**탐색 절차:**

1. `${XDG_STATE_HOME}` 디렉토리에서 `opencode.workspace.*.dat` 파일 목록을 수정 시각 역순으로 정렬한다.
2. 각 파일은 JSON 형식이며, 키에 `session:{session_id}:` 패턴이 포함되어 있다. 정규식 `session:(ses_[^:]+):` 으로 세션 ID를 추출한다.
3. 추출한 세션 ID 각각에 대해 `GET /session/:id`를 호출하여 `directory` 필드가 현재 `cwd`와 일치하는지 확인한다.
4. 일치하는 세션이 여러 개면 `time.created`가 가장 최근인 것을 사용한다.
5. 일치하는 세션을 찾으면 `session_resolution: "fallback_workspace_state"`로 보고한다.

**제약:**

- 상태 파일을 최대 5개까지만 확인한다 (수정 시각 최신순).
- `XDG_STATE_HOME`이 설정되지 않았으면 이 단계를 건너뛴다.
- 상태 파일 읽기 실패 시 무시하고 다음 파일로 진행한다.

안전 규칙:

- 3번, 4번 fallback에서 `cwd`와 세션의 `directory`를 **정확히 일치(exact match)** 시킨다.
- 5번 fallback 사용 시 세션의 `directory`가 현재 `cwd`와 다르면, 제목 변경 전에 사용자에게 다음을 안내하고 승인을 받는다:
  > "현재 작업 디렉토리(`{cwd}`)에 해당하는 세션을 찾지 못했습니다. 가장 최근 세션(`{session_id}`, 디렉토리: `{directory}`)의 제목을 변경합니다. 진행할까요?"
- 사용자 승인 없이 다른 프로젝트의 세션 제목을 변경하지 않는다.

## 제목 생성 규칙

제목 생성은 아래 규칙을 따른다.

1. 최근 대화의 핵심 작업/결과를 반영한다.
2. 한 줄 텍스트만 사용한다.
3. 불필요한 접두사 금지:
   - "Session"
   - "Updated"
   - 날짜/시간 자동 접미사
4. 길이 제한 적용:
   - 기본 60자 이하
   - hard cap 80자
5. 언어:
   - 사용자의 최근 대화 언어 우선

예시(한국어):

- `OpenCode 세션 제목 자동화 스킬 설계`
- `Desktop sidecar 동적 포트와 세션 제목 갱신`
- `Platform-agnostic 세션 리네임 규격 정리`

## 실행 절차 (필수)

1. resolver로 서버 URL 결정
2. `GET /global/health` 성공 확인
3. session id 결정 (세션 식별 규칙 1→2→3→4→5 순서 적용)
4. session id가 fallback(3~5번)으로 확보된 경우, 해당 세션의 `directory`와 현재 `cwd` 일치 여부를 확인하고 필요 시 사용자 승인
5. `GET /session/:id`로 기존 제목 조회
6. 새 제목 생성
7. `PATCH /session/:id` with `{ "title": "..." }`
8. `GET /session/:id` 재조회
9. `title` 일치 여부 확인

## API 명세

### Health

```http
GET /global/health
```

### Get session

```http
GET /session/:id
```

### Update session

```http
PATCH /session/:id
Content-Type: application/json

{
  "title": "새 제목"
}
```

## 실패 처리

### URL 탐색 실패

- `error_code`: `SERVER_URL_NOT_FOUND`
- 조치: 호스트 런타임에서 서버 URL 주입 설정 확인

### Health check 실패

- `error_code`: `SERVER_UNHEALTHY`
- 조치: sidecar/server 실행 상태 확인

### Session not found

- `error_code`: `SESSION_NOT_FOUND`
- 조치: 세션 ID source 재확인

### 디렉토리 매칭 세션 없음

- `error_code`: `SESSION_DIRECTORY_MISMATCH`
- 조건: fallback 3번에서 현재 `cwd`와 일치하는 세션이 없고, 4번 fallback에서 사용자가 승인을 거부한 경우
- 조치: 현재 프로젝트에서 새 세션을 시작하거나, 세션 ID를 직접 지정

### Update 실패

- `error_code`: `SESSION_UPDATE_FAILED`
- 조치: 권한/인증 헤더 및 요청 바디 검증

## 출력 계약

성공 시:

```json
{
  "ok": true,
  "used_base_url": "http://127.0.0.1:54659",
  "session_id": "ses_xxx",
  "old_title": "Old title",
  "new_title": "New title",
  "verified": true,
  "session_resolution": "runtime|active_context|fallback_directory_match|fallback_workspace_state|fallback_latest"
}
```

실패 시:

```json
{
  "ok": false,
  "error_code": "SERVER_URL_NOT_FOUND",
  "message": "No healthy opencode endpoint resolved",
  "next_action": "Provide runtime server URL resolver or set OPENCODE_BASE_URL"
}
```

## MUST / MUST NOT

MUST:

- 제목 변경 후 재조회 검증까지 완료
- 어떤 경로로 세션 ID를 찾았는지 출력
- 플랫폼 독립 동작 유지

MUST NOT:

- 사용자에게 포트 직접 입력 강제
- OS별 프로세스 탐지 명령 의존
- 검증 없이 "변경됨"으로 보고
