# air-platform-kit

AIR Platform 팀 개발 생산성 스킬 모음입니다.

## 설치

```bash
/plugin marketplace add https://<TOKEN>@github.com/air-platform-aie/air-platform-kit.git
/plugin install air-platform-kit@air-platform-kit
```

> **SAML SSO 토큰 발급**: https://github.com/settings/tokens → "Configure SSO" → "mzcair" → "Authorize"

## 업데이트

```bash
/plugin marketplace update air-platform-kit
/reload-plugins
```

---

## 스킬 목록

### `/air-platform-kit:start-task` — 작업 시작

티켓 없이 코드 작업을 시작할 때 Jira 티켓을 생성하고 In Progress로 전환합니다.

```bash
/air-platform-kit:start-task 로그인 API 인증 로직 수정
```

```mermaid
flowchart TD
    A([/start-task &lt;작업 설명&gt;]) --> B[브랜치명에서 티켓 ID 추출]

    B --> C{브랜치 티켓 유형}

    C -- "Task / Sub-task" --> K[해당 티켓 In Progress 전환]

    C -- "Story / Epic" --> L[서브태스크 목록 조회]
    L --> M{서브태스크 선택}
    M -- 기존 서브태스크 선택 --> K
    M -- "없음 → 새로 만들기" --> N

    C -- "티켓 없음" --> N[스프린트 스토리 조회\nLLM 관련도 순 정렬]
    N --> P{스토리 선택}
    P -- 스토리 선택 --> Q[하위 이슈 제목 제안 → 생성]
    P -- "없음 → 독립 Task" --> R[Task 제목 제안 → 생성]
    P -- 취소 --> Z([종료])

    Q --> K
    R --> K
    K --> O([✅ 완료])
```

---

### `/air-platform-kit:commit` — 커밋

변경사항을 분석해 커밋 메시지를 자동 생성하고 Jira에 작업 내용을 기록합니다.

```bash
/air-platform-kit:commit
```

```mermaid
flowchart TD
    A([/commit 실행]) --> B[브랜치명에서 티켓 ID 추출]

    B --> C{브랜치 티켓 유형}

    C -- "Task / Sub-task" --> D{이 티켓으로 연결?}
    D -- 연결 --> H
    D -- 다른 티켓 선택 --> E

    C -- "Story / Epic" --> F[서브태스크 목록 조회]
    F --> G{서브태스크 선택}
    G -- 선택 --> H[작업 시간 입력\n1h / 2h / 3h / 4h / 직접입력]
    G -- "없음 → 새로 만들기" --> E
    G -- 스토리 자체 연결 --> H

    C -- "티켓 없음" --> E[스프린트 스토리 조회\nLLM 관련도 순 정렬]
    E --> I{스토리 선택}
    I -- 스토리 선택 --> J[하위 이슈 제목 제안 → 생성]
    I -- "없음 → 독립 Task" --> K[Task 제목 제안 → 생성]
    I -- 티켓 없이 커밋 --> H

    J --> H
    K --> H

    H --> L[추가 컨텍스트 입력\nType something / 스킵]
    L --> M[git diff 분석\n커밋 메시지 생성]
    M --> N{포맷 검증}
    N -- 실패 --> M
    N -- 통과 --> O[git commit 자동 실행]
    O --> P[Jira 코멘트 + worklog 등록]
    P --> Q{Jira 상태를 리뷰로 변경?}
    Q -- 리뷰로 변경 --> R[In Review 전환]
    Q -- 유지 --> S
    R --> S([✅ 완료])
```

**커밋 메시지 포맷:**
```
feat(api): 로그인 API 인증 로직 수정

JWT 토큰 만료 처리를 개선했습니다.
- 만료된 토큰 자동 갱신 로직 추가
- 인증 실패 시 에러 메시지 개선

Refs: BRAIN-100

Generated-By: Claude Code
```

| Prefix | 용도 |
|--------|------|
| `feat` | 새로운 기능 |
| `fix` | 버그 수정 |
| `refactor` | 리팩토링 |
| `chore` | 설정/의존성 변경 |

---

## 프로젝트 설정

프로젝트 루트에 `.claude/commit-plugin.config.json` 생성:

```bash
mkdir -p .claude
cat > .claude/commit-plugin.config.json << 'EOF'
{
  "jiraProjectKey": "BRAIN"
}
EOF
```

> **보안**: `.claude/commit-plugin.config.json`을 `.gitignore`에 추가하세요.

## 요구사항

- Claude Code MCP Atlassian 연동 설정 필요
- Jira 프로젝트 접근 권한 필요
- MCP 미설치 시 Jira 연동 없이 커밋만 진행 (commit 스킬)
