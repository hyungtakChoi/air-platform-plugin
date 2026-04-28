# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## 프로젝트 개요

AIR Platform 팀 전용 Claude Code 플러그인입니다.
개발자가 `/commit`만 실행하면 Jira 연동과 커밋 메시지 자동 생성이 처리됩니다.

## 아키텍처

```
air-platform-plugin/
├── .claude-plugin/
│   └── marketplace.json      # 플러그인 마켓플레이스 정의
└── plugins/
    └── commit-plugin/        # 커밋 메시지 자동 생성 + Jira 연동
        ├── .claude-plugin/plugin.json
        └── skills/commit/SKILL.md
```

## commit-plugin v3.0.0 (`/commit`) 워크플로우

```
브랜치 티켓 ID 추출
  ├── 있음 (Task/Sub-task) → 티켓 확인 → 맞아/아니야
  ├── 있음 (Story/Epic)   → 서브태스크 multiSelect → 없으면 새로 만들기
  └── 없음                → 새로 만들기

새로 만들기:
  → Jira 스프린트 스토리 목록 조회
  → LLM이 변경사항 기반 관련 스토리 추천
  → 스토리 선택 → 서브태스크 생성 (LLM 제목 제안)
  → 없으면 독립 Task 생성 (LLM 제목 제안)

→ 작업 시간 입력 (선택)
→ 추가 컨텍스트 입력 (필수)
→ 커밋 메시지 자동 생성 → git commit
→ Jira 코멘트 + worklog 등록
```

## Jira 연동

- 인증: MCP Atlassian 도구 사용 (`mcp__claude_ai_Atlassian__*`)
- 브랜치 패턴: `[A-Z]+-\d+` (예: `feature/BI-32-desc`, `fix/AS-985`)
- 지원 프로젝트: BI, AS, COM 등 모든 Jira 프로젝트

## 커밋 메시지 포맷

```
<prefix>[(<scope>)]: <요약 메시지 (한글, 50자 이내)>

<상세 설명>
- 변경 사항 1

Refs: BI-100, BI-101

Generated-By: Claude Code
```

- prefix: `feat`, `fix`, `refactor`, `chore`
- 커밋 메시지는 반드시 한글 (prefix/scope는 영문)

## 작업 규칙

- 작업이 끝난 후에 명령 없이 커밋하지 않는다
- staged 파일이 있으면 해당 파일만 커밋, 추가 stage 금지
- 새 플러그인 추가 시 `plugins/<plugin-name>/` 디렉토리 구조를 따른다
