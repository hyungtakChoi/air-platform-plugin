---
name: commit
description: Git 커밋 메시지 자동 생성 + Jira 티켓 연동
argument-hint: "[커밋 메시지 힌트 (선택)]"
level: 2
---

version: 3.0.0

# Commit Agent

변경사항을 분석하여 git commit 메시지를 생성하고, Jira 이슈에 작업 내용을 자동으로 기록하는 에이전트입니다.

<Execution_Policy>
- 사용자에게 질문할 때 **(Y/n) 텍스트나 마크다운 테이블로 묻지 마세요** — 반드시 `AskUserQuestion` 도구를 실제로 호출하여 클릭 가능한 옵션 UI를 표시하세요
- "Use `AskUserQuestion` with..." 표시가 있으면 해당 파라미터로 도구를 즉시 호출합니다
- staged 파일이 있으면 해당 파일만 커밋하고, 절대로 다른 파일을 추가 stage하지 않습니다
- staged 파일이 없는 경우에만 모든 변경 파일을 auto-stage합니다
- 포맷 검증 통과 시 사용자 확인 없이 자동으로 커밋을 실행합니다
- 커밋 메시지는 반드시 한글로 작성합니다 (prefix와 scope는 영문 유지, 요약 50자 이내)
- Jira API 실패는 커밋을 막지 않습니다 — 경고 출력 후 커밋 진행
- Edge cases: unstaged changes만 있으면 auto-stage, detached HEAD면 경고, Jira MCP 미설치면 Step 0 건너뜀
</Execution_Policy>

---

## 1. 워크플로우

### Step 0.0: 설정 파일 로드

`.claude/commit-plugin.config.json` 파일을 읽습니다. 없으면 기본값을 사용합니다.

```json
{
  "jiraProjectKey": "BI"
}
```

| 설정 키 | 설명 | 기본값 |
|---------|------|--------|
| `jiraProjectKey` | 브랜치에 티켓 ID 없을 때 사용할 Jira 프로젝트 키 | 없음 (필수) |

---

### Step 0: Jira 티켓 연결

브랜치명에서 티켓 ID를 추출합니다.

```bash
git branch --show-current
```

지원 패턴: `[A-Z]+-\d+` (예: `feature/BI-123`, `fix/AS-456-desc`)
추출된 티켓 ID에서 프로젝트 키도 파악합니다 (예: `BI-32` → 프로젝트 키 `BI`)

---

#### 분기 1: 브랜치에 티켓 ID 있음

`mcp__claude_ai_Atlassian__getJiraIssue`로 티켓 정보를 조회합니다.

**이슈 타입이 Task / Sub-task인 경우:**

Use `AskUserQuestion` with:
- question: "브랜치에서 감지된 Jira 티켓입니다. 이 티켓으로 연결하시겠습니까?"
- header: "Jira 티켓 확인"
- multiSelect: false
- options:
  - label: "{티켓ID}: {요약}", description: "타입: {type} | 상태: {status}"
  - label: "다른 티켓 선택", description: "새 티켓을 만들거나 다른 티켓을 연결합니다"

→ 티켓 선택 시: 해당 티켓 사용
  - 티켓의 `description` 필드가 비어있으면: LLM이 git diff 기반으로 description 자동 생성 → `mcp__claude_ai_Atlassian__editJiraIssue`로 업데이트
  - description이 이미 있으면: 건드리지 않음
  → Step 0.5로
→ "다른 티켓 선택" 선택 시: [새로 만들기 로직]으로

**이슈 타입이 Story / Epic인 경우:**

`mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 서브태스크 목록 조회:
- JQL: `parent = {티켓ID} ORDER BY created DESC`

Use `AskUserQuestion` with:
- question: "이 스토리의 어떤 서브태스크와 연결하시겠어요?"
- header: "서브태스크 선택"
- multiSelect: true
- options: 각 서브태스크 (label: 티켓ID, description: 요약) + "없어 — 새로 만들기" + "스토리 자체만 연결"

→ 서브태스크 선택 시: 선택된 티켓들 사용
  - 각 티켓의 `description`이 비어있으면: LLM이 git diff 기반으로 자동 생성 → `mcp__claude_ai_Atlassian__editJiraIssue`로 업데이트
  - description이 이미 있으면: 건드리지 않음
  → Step 0.5로
→ "스토리 자체만 연결" 선택 시: 스토리 티켓 사용 → Step 0.5로
→ "없어 — 새로 만들기" 선택 시: [새로 만들기 로직]으로

---

#### 분기 2: 브랜치에 티켓 ID 없음

바로 [새로 만들기 로직]으로 진행합니다.

---

#### 새로 만들기 로직 (공통)

1. `git diff --staged` 또는 `git diff`로 변경사항을 미리 분석합니다
2. 프로젝트 키 결정: 브랜치에서 추출한 키 → 없으면 `config.jiraProjectKey` 사용. 둘 다 없으면 사용자에게 입력 요청
3. `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 현재 프로젝트의 활성 스토리/에픽 목록 조회:
   - JQL: `project = {projectKey} AND issuetype in (Story, Epic) AND sprint in openSprints() ORDER BY updated DESC`
   - 스프린트가 없으면 폴백: `project = {projectKey} AND issuetype in (Story, Epic) AND updated >= -30d ORDER BY updated DESC`
3. LLM이 변경사항과 스토리 목록을 비교하여 **관련도 순으로 정렬**합니다

Use `AskUserQuestion` with:
- question: "어떤 스토리 하위에 서브태스크를 만들까요? (변경사항 기반으로 관련도 순 정렬)"
- header: "스토리 선택"
- multiSelect: false
- options: 관련도 순 스토리 목록 (label: 티켓ID, description: 요약) + "없어 — 독립 Task 생성" + "티켓 없이 커밋"

**스토리 선택 시 → 서브태스크 생성:**

LLM이 변경사항을 기반으로 서브태스크 제목을 자동 제안합니다.

Use `AskUserQuestion` with:
- question: "서브태스크 제목을 확인하세요. AI 제안 제목을 선택하거나 'Type something'에 직접 입력하세요."
- header: "서브태스크 제목"
- multiSelect: false
- options:
  - label: "{LLM이 제안한 제목}", description: "AI 제안"

- 사용자가 AI 제안을 선택하거나 "Type something" 빈칸에 직접 입력합니다
- 직접 입력이 있으면 해당 텍스트를 제목으로 사용합니다

확인 후 `mcp__claude_ai_Atlassian__createJiraIssue`로 서브태스크 생성:
- issuetype: Sub-task
- parent: {선택한 스토리 ID}
- summary: {확인된 제목}
- project: {projectKey}
- description: LLM이 git diff 기반으로 자동 생성 — 변경된 파일, 작업 목적, 주요 변경 내용을 2-3문장으로 요약

**"없어 — 독립 Task 생성" 선택 시:**

LLM이 변경사항을 기반으로 Task 제목을 자동 제안합니다.

Use `AskUserQuestion` with:
- question: "Task 제목을 확인하세요. AI 제안 제목을 선택하거나 'Type something'에 직접 입력하세요."
- header: "Task 제목"
- multiSelect: false
- options:
  - label: "{LLM이 제안한 제목}", description: "AI 제안"

- 사용자가 AI 제안을 선택하거나 "Type something" 빈칸에 직접 입력합니다
- 직접 입력이 있으면 해당 텍스트를 제목으로 사용합니다

확인 후 `mcp__claude_ai_Atlassian__createJiraIssue`로 Task 생성:
- issuetype: Task
- summary: {확인된 제목}
- project: {projectKey}
- description: LLM이 git diff 기반으로 자동 생성 — 변경된 파일, 작업 목적, 주요 변경 내용을 2-3문장으로 요약

**"티켓 없이 커밋" 선택 시:** Refs 없이 커밋 진행

---

#### MCP 연결 실패 처리

`mcp__claude_ai_Atlassian__*` 도구 호출 실패 시:
- 경고 메시지 출력 후 Step 1로 진행 (커밋은 정상 실행)

---

### Step 0.5: 작업 시간 입력

Jira 티켓이 연결된 경우에만 Use `AskUserQuestion` with:
- question: "이 작업에 소요된 시간을 선택하세요"
- header: "작업 시간"
- multiSelect: false
- options:
  - label: "1h", description: "1시간"
  - label: "2h", description: "2시간"
  - label: "3h", description: "3시간"
  - label: "4h", description: "4시간"
  - label: "직접 입력", description: "정수 시간 단위로 입력 (예: 5h, 6h)"
  - label: "스킵", description: "worklog 기록 안 함"

---

### Step 0.6: Jira 추가 컨텍스트 입력

Jira 티켓이 연결된 경우에만 Use `AskUserQuestion` with:
- question: "Jira 티켓에 남길 추가 컨텍스트가 있으면 아래에 직접 입력하세요. (작업 배경, 이슈 원인, 시도한 방법, 주의사항 등 — 커밋 메시지에 담지 못한 내용)"
- header: "추가 컨텍스트"
- multiSelect: false
- options:
  - label: "스킵", description: "추가 컨텍스트 없이 진행합니다"

- 사용자가 "Type something" 빈칸에 직접 입력하거나 "스킵"을 선택합니다
- 입력이 있으면 LLM이 자동으로 정제하여 Jira 코멘트에 등록합니다 (문장 다듬기, 구조화, 의도 보존)
- "스킵" 선택 시 추가 컨텍스트 없이 진행합니다

---

### Step 1: Git 상태 확인

```bash
git status
git diff --staged --shortstat
```

- staged 파일이 있으면 해당 파일만 커밋
- staged 파일이 없고 unstaged 변경사항이 있으면 모두 stage
- 변경사항이 없으면 중단

---

### Step 2: 변경사항 분석

`git diff --staged`로 변경사항을 분석합니다.

- 변경된 파일 목록 확인
- 코드 변경 내용 파악
- 작업 의도 분석

---

### Step 3: 커밋 메시지 생성

분석 결과를 기반으로 커밋 메시지를 생성합니다. (아직 커밋하지 않음)

**Refs 라인 생성 규칙:**
- 단일 티켓: `Refs: BI-100`
- 복수 티켓: `Refs: BI-100, BI-101`
- 티켓 없음: Refs 라인 생략

---

### Step 4: 커밋 메시지 포맷 검증

생성된 커밋 메시지가 포맷 규칙을 준수하는지 검증합니다. **검증 실패 시 Step 3으로 돌아가 메시지를 재생성합니다.**

| 요소 | 필수 여부 |
|------|-----------|
| prefix (feat/fix/refactor/chore) | 필수 |
| 요약 메시지 (한글, 50자 이내) | 필수 |
| 상세 설명 (bullet point) | 필수 |
| Generated-By 라인 | 필수 |
| Refs (Jira ticket) | 선택 |
| scope | 선택 |

---

### Step 5: 커밋 실행

포맷 검증을 통과한 커밋 메시지로 **사용자 확인 없이 자동으로 커밋을 실행**합니다.

```bash
git commit -m "$(cat <<'EOF'
<생성된 커밋 메시지>
EOF
)"
```

커밋 완료 후 커밋 ID를 저장합니다:
```bash
git rev-parse HEAD
```

---

### Step 6: Jira 등록

Step 0에서 Jira 티켓이 연결된 경우, **커밋 직후** 각 티켓에 대해 실행합니다.

#### 6.1 코멘트 등록

`mcp__claude_ai_Atlassian__addCommentToJiraIssue`로 각 티켓에 코멘트 등록:

```markdown
## 커밋 완료

**커밋 메시지**
{커밋 메시지 전문}

**변경 정보**
- Commit: `{commitId}`
- Branch: `{브랜치명}`
- Changes: +{추가라인} / -{삭제라인}

**추가 컨텍스트**
{LLM이 정제한 사용자 입력 내용}

*Generated-By: Claude Code*
```

#### 6.2 Worklog 등록

작업 시간을 입력한 경우 `mcp__claude_ai_Atlassian__addWorklogToJiraIssue`로 worklog 등록.

#### 6.3 Jira 상태 변경

Use `AskUserQuestion` with:
- question: "Jira 티켓 상태를 '리뷰 (In Review)'로 변경하시겠습니까?"
- header: "Jira 상태 변경"
- multiSelect: false
- options:
  - label: "리뷰로 변경", description: "티켓 상태를 In Review로 전환합니다"
  - label: "유지", description: "현재 상태를 그대로 유지합니다"

→ "리뷰로 변경" 선택 시: 연결된 각 티켓에 대해 `mcp__claude_ai_Atlassian__transitionJiraIssue`를 호출합니다
  - 전환 가능한 transition 목록을 먼저 조회하여 "In Review" 또는 "리뷰"에 해당하는 transitionId를 찾아 사용합니다
  - 해당 transition이 없으면 경고 출력 후 스킵
→ "유지" 선택 시: 상태 변경 없이 완료

#### 6.4 실패 처리

Jira 등록 실패 시 경고 메시지만 출력. 커밋은 이미 완료되어 롤백하지 않음.

---

## 2. 커밋 메시지 포맷

```
<prefix>[(<scope>)]: <요약 메시지 (한글, 50자 이내)>

<상세 설명 (한글)>
- 변경 사항 1
- 변경 사항 2

Refs: <jira-ticket(s)>

Generated-By: Claude Code
```

| Prefix | 용도 |
|--------|------|
| `feat` | 새로운 기능 추가 |
| `fix` | 버그 수정, 유지보수 |
| `refactor` | 리팩토링 |
| `chore` | 설정 변경, 의존성 업데이트 |

---

## 3. 완료 메시지

```
## Commit 완료

| 항목 | 값 |
|------|-----|
| Commit ID | `abc123...` |
| Branch | feature/BI-32 |
| Files Changed | 3 files |
| Lines | +80 / -20 |

### 커밋 메시지
feat(api): 사용자 인증 API 구현

JWT 기반 인증 엔드포인트를 구현했습니다.
- 로그인/로그아웃 API 추가
- 토큰 갱신 로직 구현

Refs: BI-33, BI-34

Generated-By: Claude Code

### Jira 연동

| 티켓 | 코멘트 | Worklog | 상태 |
|------|--------|---------|------|
| BI-33 | ✅ 등록됨 | 2h | 리뷰로 변경 |
| BI-34 | ✅ 등록됨 | - | 유지 |
```

---

<Tool_Usage>
- Use `AskUserQuestion` for all user-facing choices — provides clickable UI. Never use text prompts or (Y/n) questions.
- Use `mcp__claude_ai_Atlassian__getJiraIssue` to fetch ticket info from branch name
- Use `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql` to list subtasks or search stories
- Use `mcp__claude_ai_Atlassian__createJiraIssue` to create sub-tasks or tasks
- Use `mcp__claude_ai_Atlassian__editJiraIssue` to update description on existing tickets when empty
- Use `mcp__claude_ai_Atlassian__addCommentToJiraIssue` to post commit summary to Jira after commit
- Use `mcp__claude_ai_Atlassian__addWorklogToJiraIssue` to log work time after commit
- Use `mcp__claude_ai_Atlassian__transitionJiraIssue` to change ticket status to In Review after commit
- Use `git branch --show-current`, `git status`, `git diff --staged` for git state
- Use `git commit -m` with HEREDOC for safe multiline commit messages
- Use `git rev-parse HEAD` after commit to capture commit ID
</Tool_Usage>
