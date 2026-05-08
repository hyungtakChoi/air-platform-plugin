# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## 프로젝트 개요

AIR Platform 팀 전용 Claude Code 플러그인입니다.
레포 자체가 하나의 플러그인(`air-platform-kit`)이며, 팀 생산성을 위한 스킬들을 포함합니다.

## 아키텍처

```
air-platform-kit/
├── .claude-plugin/
│   └── plugin.json          # 플러그인 정의 (name: air-platform-kit)
├── skills/
│   ├── commit/SKILL.md      # 커밋 자동화 + Jira 연동
│   └── start-task/SKILL.md  # 작업 시작 시 Jira 티켓 생성
└── README.md
```

## 스킬 목록

### `/air-platform-kit:commit` (v3.1.2)
- git diff 분석 → 커밋 메시지 자동 생성 (한글)
- Jira 티켓 연결 / 생성 / worklog / 상태 변경

### `/air-platform-kit:start-task` (v1.0.0)
- 작업 시작 시 Jira 티켓 생성 및 In Progress 전환
- 브랜치 Story/Epic → 서브태스크 목록 확인 후 선택 또는 생성
- 브랜치 티켓 없음 → 스프린트 스토리 추천 → 생성

## Jira 연동

- 인증: MCP Atlassian 도구 사용 (`mcp__claude_ai_Atlassian__*`)
- 브랜치 패턴: `[A-Z]+-\d+` (예: `feature/BRAIN-32-desc`)
- 하위 이슈 타입: 프로젝트 기존 자식 이슈 타입 자동 감지 (Sub-task 또는 Task)

## 커밋 메시지 포맷

```
<prefix>[(<scope>)]: <요약 메시지 (한글, 50자 이내)>

<상세 설명>
- 변경 사항 1

Refs: BRAIN-100

Generated-By: Claude Code
```

- prefix: `feat`, `fix`, `refactor`, `chore`
- 커밋 메시지는 반드시 한글 (prefix/scope는 영문 유지)

## 작업 규칙

- 작업이 끝난 후에 명령 없이 커밋하지 않는다
- 새 스킬 추가 시 `skills/<skill-name>/SKILL.md` 구조를 따른다
