---
name: start-task
description: 작업 시작 시 Jira 티켓 생성 및 In Progress 전환
argument-hint: "[작업 설명 (선택)]"
level: 2
---

version: 1.0.0

# Start Task Agent

티켓 없이 코드 작업을 시작한 개발자가 Jira 티켓을 빠르게 생성하고 In Progress로 전환하는 에이전트입니다.

<Execution_Policy>
- 사용자에게 질문할 때 **(Y/n) 텍스트나 마크다운 테이블로 묻지 마세요** — 반드시 `AskUserQuestion` 도구를 실제로 호출하여 클릭 가능한 옵션 UI를 표시하세요
- "Use `AskUserQuestion` with..." 표시가 있으면 해당 파라미터로 도구를 즉시 호출합니다
- commit-plugin과 완전히 독립적으로 동작합니다 — 브랜치 rename 없음, 상태 파일 없음
- Jira API 실패는 경고 출력 후 계속 진행합니다
- 선택지 표현은 존댓말을 사용합니다 ("없음", "취소" 등)
- Edge cases: Jira MCP 미설치 시 에러 안내 후 종료
</Execution_Policy>

---

## 1. 워크플로우

### Step 0.0: 설정 파일 로드

`.claude/commit-plugin.config.json` 파일을 읽습니다. 없으면 기본값을 사용합니다.

| 설정 키 | 설명 | 기본값 |
|---------|------|--------|
| `jiraProjectKey` | 브랜치에 티켓 ID 없을 때 사용할 Jira 프로젝트 키 | 없음 (필수) |

---

### Step 0.1: 작업 설명 확인

인자(`{{ARGUMENTS}}`)가 있으면 그대로 사용합니다.

인자가 없으면 Use `AskUserQuestion` with:
- question: "어떤 작업을 시작하시나요? 작업 내용을 간략히 설명해 주세요."
- header: "작업 설명"
- multiSelect: false
- options:
  - label: "취소", description: "start-task를 취소합니다"

- 사용자가 "Type something" 빈칸에 직접 입력하거나 "취소"를 선택합니다
- "취소" 선택 시 종료

---

### Step 0.2: 브랜치 확인 및 티켓 추출

```bash
git branch --show-current
```

지원 패턴: `[A-Z]+-\d+` (예: `feature/BI-123`, `BRAIN-21-desc`)
추출된 티켓 ID에서 프로젝트 키도 파악합니다 (예: `BRAIN-21` → 프로젝트 키 `BRAIN`)

---

### Step 1: Jira 티켓 분기

#### 분기 A: 브랜치에 Task / Sub-task ID 있음

`mcp__claude_ai_Atlassian__getJiraIssue`로 티켓 정보 조회합니다.

해당 티켓이 이미 존재하므로 새로 생성하지 않고 In Progress로 전환합니다.

→ [Step 3: In Progress 전환]으로

---

#### 분기 B: 브랜치에 Story / Epic ID 있음

`mcp__claude_ai_Atlassian__getJiraIssue`로 스토리/에픽 정보 조회합니다.

`mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 기존 서브태스크 목록 조회:
- JQL: `parent = {티켓ID} ORDER BY created DESC`

Use `AskUserQuestion` with:
- question: "이 스토리의 어떤 서브태스크 작업을 시작하시나요?"
- header: "서브태스크 선택"
- multiSelect: false
- options: 각 서브태스크 (label: 티켓ID, description: 요약 | 상태) + "없음 — 새로 만들기" + "취소"

→ 기존 서브태스크 선택 시: 해당 티켓 → [Step 3: In Progress 전환]으로
→ "없음 — 새로 만들기" 선택 시: [서브태스크 생성 흐름]으로
→ "취소" 선택 시: 종료

---

#### 분기 C: 브랜치에 티켓 ID 없음

프로젝트 키 결정: 브랜치에서 추출한 키 → 없으면 `config.jiraProjectKey` 사용. 둘 다 없으면 사용자에게 입력 요청.

현재 사용자의 accountId 조회:
- JQL: `assignee = currentUser() AND project = {projectKey} ORDER BY updated DESC`
- 첫 번째 결과의 `assignee.accountId`를 저장 (createJiraIssue의 assignee에 사용)

`mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 활성 스프린트 스토리 목록 조회:
- JQL: `project = {projectKey} AND issuetype in (Story, Epic) AND sprint in openSprints() ORDER BY updated DESC`
- 스프린트 없으면 폴백: `project = {projectKey} AND issuetype in (Story, Epic) AND updated >= -30d ORDER BY updated DESC`

LLM이 작업 설명과 스토리 목록을 비교하여 **관련도 순으로 정렬**합니다.

Use `AskUserQuestion` with:
- question: "이 작업은 어떤 스토리에 속하나요? (작업 설명 기반 관련도 순 정렬)"
- header: "스토리 선택"
- multiSelect: false
- options: 관련도 순 스토리 목록 (label: 티켓ID, description: 요약) + "없음 — 독립 Task 생성" + "취소"

→ "취소" 선택 시: 종료
→ "없음 — 독립 Task 생성" 선택 시: [독립 Task 생성 흐름]으로
→ 스토리 선택 시: [서브태스크 생성 흐름]으로

---

### Step 2: 티켓 생성

#### 서브태스크 생성 흐름

LLM이 작업 설명을 기반으로 서브태스크 제목을 자동 제안합니다.

Use `AskUserQuestion` with:
- question: "서브태스크 제목을 확인하세요. AI 제안 제목을 선택하거나 'Type something'에 직접 입력하세요."
- header: "서브태스크 제목"
- multiSelect: false
- options:
  - label: "{LLM이 제안한 제목}", description: "AI 제안"

- 사용자가 AI 제안을 선택하거나 "Type something" 빈칸에 직접 입력합니다

`mcp__claude_ai_Atlassian__createJiraIssue`로 서브태스크 생성:
- issuetype: Sub-task
- parent: {선택한 스토리 ID 또는 브랜치에서 추출한 Story/Epic ID}
- summary: {확인된 제목}
- project: {projectKey}
- assignee: {조회한 currentUser accountId} (없으면 생략)
- description: LLM이 작업 설명 기반으로 자동 생성 — 작업 목적, 주요 변경 예정 내용을 2-3문장으로 요약

→ 생성된 티켓 ID 저장 → [Step 3: In Progress 전환]으로

---

#### 독립 Task 생성 흐름

LLM이 작업 설명을 기반으로 Task 제목을 자동 제안합니다.

Use `AskUserQuestion` with:
- question: "Task 제목을 확인하세요. AI 제안 제목을 선택하거나 'Type something'에 직접 입력하세요."
- header: "Task 제목"
- multiSelect: false
- options:
  - label: "{LLM이 제안한 제목}", description: "AI 제안"

- 사용자가 AI 제안을 선택하거나 "Type something" 빈칸에 직접 입력합니다

`mcp__claude_ai_Atlassian__createJiraIssue`로 Task 생성:
- issuetype: Task
- summary: {확인된 제목}
- project: {projectKey}
- assignee: {조회한 currentUser accountId} (없으면 생략)
- description: LLM이 작업 설명 기반으로 자동 생성 — 작업 목적, 주요 변경 예정 내용을 2-3문장으로 요약

→ 생성된 티켓 ID 저장 → [Step 3: In Progress 전환]으로

---

### Step 3: In Progress 전환

`mcp__claude_ai_Atlassian__transitionJiraIssue`로 티켓 상태를 In Progress로 전환합니다.
- 전환 가능한 transition 목록을 먼저 조회하여 "In Progress" 또는 "진행 중"에 해당하는 transitionId를 찾아 사용합니다
- 해당 transition이 없으면 경고 출력 후 스킵

---

### Step 4: 완료 메시지

```
## Start Task 완료

| 항목 | 값 |
|------|-----|
| 티켓 | BRAIN-75 |
| 제목 | {티켓 제목} |
| 타입 | Sub-task / Task |
| 상태 | 🟡 In Progress |
| 브랜치 | {현재 브랜치명} |

작업을 완료한 후 `/commit-plugin:commit`을 실행하여 커밋하세요.
```

---

<Tool_Usage>
- Use `AskUserQuestion` for all user-facing choices — provides clickable UI. Never use text prompts or (Y/n) questions.
- Use `mcp__claude_ai_Atlassian__getJiraIssue` to fetch ticket info from branch name
- Use `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql` to list sprint stories or find current user
- Use `mcp__claude_ai_Atlassian__createJiraIssue` to create sub-tasks or tasks
- Use `mcp__claude_ai_Atlassian__transitionJiraIssue` to set ticket status to In Progress
- Use `git branch --show-current` to get current branch name
</Tool_Usage>
