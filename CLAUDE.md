# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## 프로젝트 개요

AIR Platform 팀 전용 Claude Code 플러그인 모음입니다.
개발자가 커밋/푸쉬만 하면 Jira 연동, 생산성 측정, Dashboard 기록이 자동으로 처리됩니다.

## 아키텍처

```
air-platform-plugin/
├── .claude-plugin/
│   └── marketplace.json      # 플러그인 마켓플레이스 정의
└── plugins/
    ├── commit-plugin/        # 커밋 메시지 자동 생성 + Jira 연동
    │   ├── .claude-plugin/plugin.json
    │   └── skills/commit/SKILL.md
    └── push-plugin/          # DevPulse Dashboard 전송 + Git Push
        ├── .claude-plugin/plugin.json
        └── skills/push/SKILL.md
```

## 플러그인 워크플로우

### commit-plugin v2.0.0 (`/commit`)
1. Jira 티켓 확인 (브랜치명 추출 → 서브태스크 선택 or 검색 or 생성)
2. 작업 시간 입력 (선택)
3. Jira 추가 컨텍스트 입력 (선택)
4. Git 상태 확인 (staged 파일 우선)
5. 변경사항 분석
6. 커밋 메시지 생성
7. 포맷 검증
8. 커밋 실행 + Jira 코멘트/worklog 등록 (동시)

### push-plugin v1.0.7 (`/push`)
1. 설정 로드 (`.claude/dashboard.config.json`)
2. Unpushed 커밋 수집
3. Git diff 분석 및 평가 (complexity, volume, thinking, others)
4. 평가 검증
5. API Input 검증
6. Dashboard API 전송 (GraphQL)
7. Git Push 실행

## 설정 파일

사용자 프로젝트의 `.claude/dashboard.config.json` (gitignore 필수):
```json
{
  "developer": { "email": "your-email@mz.co.kr" },
  "product": { "id": "team-uuid/product-uuid" },
  "apiKey": "your-api-key"
}
```

## Jira 연동

- Jira 프로젝트: `AS` (mzdevs.atlassian.net)
- 인증: MCP Atlassian 도구 사용 (별도 토큰 불필요)
- 브랜치 패턴: `AS-\d+` (예: `feature/AS-985-description`)

## 커밋 메시지 포맷

```
<prefix>[(<scope>)]: <요약 메시지 (한글, 50자 이내)>

<상세 설명>
- 변경 사항 1

Refs: AS-100, AS-101

Generated-By: Claude Code
```

- prefix: `feat`, `fix`, `refactor`, `chore`
- 커밋 메시지는 반드시 한글 (prefix/scope는 영문)

## 작업 규칙

- 작업이 끝난 후에 명령 없이 커밋하지 않는다
- staged 파일이 있으면 해당 파일만 커밋, 추가 stage 금지
- 새 플러그인 추가 시 `plugins/<plugin-name>/` 디렉토리 구조를 따른다
