# Skills

AI 코딩 에이전트를 위한 개인 스킬 컬렉션.

## 설치

```bash
# 전체 설치
npx skills add ndaemy/skills --skill '*'

# 특정 스킬만 설치
npx skills add ndaemy/skills --skill figma-mcp-bridge

# 전역 설치 (모든 프로젝트에서 사용)
npx skills add ndaemy/skills --skill '*' -g
```

## 스킬 목록

| 스킬 | 설명 | 사용 시점 |
|------|------|----------|
| [figma-mcp-bridge](skills/figma-mcp-bridge) | Figma MCP 미승인 클라이언트에서 승인된 CLI(Claude Code, Codex, Gemini CLI)를 tmux 브릿지로 Figma MCP에 접근 | Figma 디자인 파일 조회, 컴포넌트 구조 파악, 디자인 컨텍스트 추출이 필요할 때 |

## 스킬 구조

```
skills/
└── <skill-name>/
    └── SKILL.md          # 에이전트 지시사항
```
