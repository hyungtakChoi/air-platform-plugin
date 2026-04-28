---
name: commit
description: Git 커밋 메시지 자동 생성 + Jira 티켓 연동
argument-hint: "[커밋 메시지 힌트 (선택)]"
pipeline: [commit, push]
next-skill: push-plugin:push
handoff: .omc/state/commit-handoff.json
level: 2
---

version: 2.0.0

# Commit Agent

변경사항을 분석하여 git commit 메시지를 생성하고, Jira 이슈에 작업 내용을 자동으로 기록하는 에이전트입니다.

> **Note**: 평가(evaluation)와 Dashboard API 호출은 `/push`를 사용하세요.

<Execution_Policy>
- 사용자에게 질문할 때 **(Y/n) 텍스트나 마크다운 테이블로 묻지 마세요** — 반드시 `AskUserQuestion` 도구를 실제로 호출하여 클릭 가능한 옵션 UI를 표시하세요
- 각 단계에 "Use `AskUserQuestion` with..." 표시가 있으면 해당 파라미터로 도구를 즉시 호출합니다
- staged 파일이 있으면 해당 파일만 커밋하고, 절대로 다른 파일을 추가 stage하지 않습니다
- staged 파일이 없는 경우에만 모든 변경 파일을 auto-stage합니다
- 포맷 검증 통과 시 사용자 확인 없이 자동으로 커밋을 실행합니다
- Step 0 티켓 선택·작업 시간은 스킵 가능하나, 티켓이 선택된 경우 추가 컨텍스트(Step 0.6) 입력은 필수입니다
- 티켓 선택·작업 시간을 모두 스킵하면 기존 v1.1.0과 동일하게 동작합니다
- 커밋 메시지는 반드시 한글로 작성합니다 (prefix와 scope는 영문 유지, 요약 50자 이내)
- 상세 설명은 변경 사항을 bullet point로 구체적으로 나열합니다
- 사용자가 커밋 메시지 힌트를 제공하면 해당 내용을 포맷에 맞게 반영합니다
- Jira API 실패는 커밋을 막지 않습니다 — 경고 출력 후 커밋 진행
- 커밋 완료 후 `.omc/state/commit-handoff.json`에 handoff 정보를 기록합니다 (push-plugin이 읽음)
- Edge cases: unstaged changes만 있으면 auto-stage, detached HEAD면 경고, Jira MCP 미설치면 Step 0 건너뜀
</Execution_Policy>

---

## 1. 워크플로우

### Step 0: Jira 티켓 확인 및 선택

```bash
git branch --show-current
```

브랜치명에서 Jira 티켓 ID를 추출합니다.
지원 패턴: `[A-Z]+-\d+` (예: `feature/AS-99`, `feature/BI-123`, `fix/COM-456-desc`)
추출된 티켓 ID에서 프로젝트 키도 파악합니다 (예: `BI-23` → 프로젝트 키 `BI`)

#### 0.1 시나리오 A: 스토리/이슈 티켓

티켓 ID가 추출되면 `mcp__claude_ai_Atlassian__getJiraIssue`로 이슈 정보를 조회합니다.

이슈 타입이 Story/Epic이거나 subtask가 존재하면:

1. `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 서브태스크 목록 조회:
   ```
   parent = {티켓ID} ORDER BY created DESC
   ```
2. **→ AskUserQuestion 도구를 즉시 호출**합니다:
   - questions[0].question: "연결할 서브태스크를 선택하세요"
   - questions[0].header: "서브태스크 선택"
   - questions[0].multiSelect: true
   - questions[0].options: 각 서브태스크 (label: 티켓ID, description: 요약) + "해당 없음 — 스토리만 연결" + "새 서브태스크 생성"
3. "해당 없음" 선택 시 → 스토리 티켓 자체를 Refs로 사용
4. "새 서브태스크 생성" 선택 시 → Step 0.3으로 이동

#### 0.2 시나리오 B: 단순 작업(Task) 또는 서브태스크 티켓

이슈 타입이 Task/Sub-task이면 **→ AskUserQuestion 도구를 즉시 호출**합니다:

- questions[0].question: "연결된 Jira 티켓을 확인하세요"
- questions[0].header: "Jira 티켓"
- questions[0].multiSelect: false
- questions[0].options:
  - label: "{티켓ID}: {요약}", description: "상태: {status}"
  - label: "다른 티켓 검색", description: "다른 이슈를 검색합니다"

#### 0.3 시나리오 C: 브랜치에 티켓 ID 없음

단계별 폴백으로 처리합니다:

**Step C-1: Jira 이슈 검색**

`mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 관련 이슈 검색:
- JQL: `project = {브랜치에서 추출한 프로젝트 키, 없으면 dashboard.config.json의 jiraProjectKey} AND assignee = currentUser() AND updated >= -14d ORDER BY updated DESC`

검색 결과로 **→ AskUserQuestion 도구를 즉시 호출**합니다:
- questions[0].question: "연결할 Jira 이슈를 선택하세요"
- questions[0].header: "Jira 이슈 선택"
- questions[0].multiSelect: false
- questions[0].options: 검색된 각 이슈 (label: 티켓ID, description: 요약) + "직접 입력 (티켓 ID 직접 입력)" + "새 서브태스크 생성" + "없음 — 티켓 없이 커밋"

**Step C-2: 직접 입력**

"직접 입력" 선택 시 **→ AskUserQuestion 도구를 즉시 호출**합니다:
- questions[0].question: "연결할 Jira 티켓 ID를 입력하세요 (예: BI-123)"
- questions[0].header: "티켓 ID 직접 입력"
- questions[0].multiSelect: false
- questions[0].options:
  - label: "직접 입력", description: "티켓 ID를 입력하세요"
  - label: "새 서브태스크 생성", description: "부모 스토리 아래 서브태스크를 새로 만듭니다"

**Step C-3: 새 서브태스크 생성**

"새 서브태스크 생성" 선택 시 **→ AskUserQuestion 도구를 즉시 호출**합니다:
- questions[0].question: "서브태스크를 생성할 부모 스토리 ID를 입력하세요"
- questions[0].header: "부모 스토리 ID"
- questions[0].multiSelect: false
- questions[0].options: label: "직접 입력"

입력 후 `mcp__claude_ai_Atlassian__createJiraIssue`로 서브태스크 생성.
"없음 — 티켓 없이 커밋" 선택 시: Refs 없이 커밋 진행.

#### 0.4 MCP 연결 실패 처리

`mcp__claude_ai_Atlassian__*` 도구 호출 실패 시:
- 경고 메시지 출력 후 Step 1로 진행 (커밋은 정상 실행)

---

### Step 0.5: 작업 시간 입력

Jira 티켓이 선택된 경우에만 **→ AskUserQuestion 도구를 즉시 호출**합니다:

- questions[0].question: "이 작업에 소요된 시간을 선택하세요"
- questions[0].header: "작업 시간"
- questions[0].multiSelect: false
- questions[0].options:
  - label: "1h", description: "1시간"
  - label: "2h", description: "2시간"
  - label: "3h", description: "3시간"
  - label: "4h", description: "4시간"
  - label: "직접 입력", description: "정수 시간 단위로 입력 (예: 5h, 6h)"

- 지원 형식: `Xh` (정수 시간 단위만, 예: `1h`, `3h`)
- "직접 입력" 선택 시 추가로 시간값 입력 받기

---

### Step 0.6: Jira 추가 컨텍스트 입력

Jira 티켓이 선택된 경우에만 **→ AskUserQuestion 도구를 즉시 호출**합니다:

- questions[0].question: "Jira 티켓에 남길 추가 컨텍스트를 입력하세요 (커밋 메시지에 담지 못한 배경, 시도한 방법, 주의사항 등)"
- questions[0].header: "추가 컨텍스트"
- questions[0].multiSelect: false
- questions[0].options:
  - label: "직접 입력", description: "작업 배경, 이슈 원인, 주의사항 등 자유 입력"

- 입력은 **필수**이며 스킵할 수 없습니다
- 사용자가 입력한 원문을 LLM이 자동으로 정제하여 Jira 코멘트에 등록합니다
  - 문장을 다듬고 구조화하되, 사용자의 의도와 핵심 내용은 그대로 보존
  - 정제 후 Jira에 등록되는 내용을 사용자에게 미리 보여줄 필요 없음 (자동 처리)

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
- 단일 티켓: `Refs: AS-100`
- 복수 티켓: `Refs: AS-100, AS-101, AS-102`
- 티켓 없음: Refs 라인 생략

---

### Step 4: 커밋 메시지 포맷 검증

생성된 커밋 메시지가 포맷 규칙을 준수하는지 검증합니다. **검증 실패 시 Step 3으로 돌아가 메시지를 재생성합니다.**

#### 4.1 필수 요소 존재 검증

| 요소                             | 필수 여부 | 존재 여부 | 통과 여부 |
| -------------------------------- | --------- | --------- | --------- |
| prefix (feat/fix/refactor/chore) | 필수      | ?         | /         |
| 요약 메시지 (한글)               | 필수      | ?         | /         |
| 상세 설명                        | 필수      | ?         | /         |
| Generated-By 라인                | 필수      | ?         | /         |
| Refs (Jira ticket)               | 선택      | ?         | /         |
| scope                            | 선택      | ?         | /         |

#### 4.2 포맷 규칙 검증

| 규칙             | 검증 내용                                       | 통과 여부 |
| ---------------- | ----------------------------------------------- | --------- |
| prefix 유효성    | `feat`, `fix`, `refactor`, `chore` 중 하나인가? | /         |
| 요약 메시지 길이 | 50자 이내인가?                                  | /         |
| 요약 메시지 언어 | 한글로 작성되었는가?                            | /         |
| scope 형식       | `prefix(scope):` 또는 `prefix:` 형식인가?       | /         |

#### 4.3 검증 결과

- 모든 필수 검증 통과 시: **Step 5로 진행**
- 검증 실패 시: **Step 3으로 돌아가 메시지 재생성**

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

Step 0에서 Jira 티켓이 선택된 경우, **커밋 직후** 각 선택 티켓에 대해 실행합니다.

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
{추가 컨텍스트가 있는 경우:
---
**추가 컨텍스트**
{사용자 입력 내용}
}

*Generated-By: Claude Code*
```

#### 6.2 Worklog 등록

작업 시간을 입력한 경우 `mcp__claude_ai_Atlassian__addWorklogToJiraIssue`로 worklog 등록.

#### 6.3 실패 처리

Jira 등록 실패 시:
- 경고 메시지만 출력
- 커밋은 이미 완료되어 있으므로 롤백하지 않음

---

## 2. 커밋 메시지 포맷

```
<prefix>[(<scope>)]: <요약 메시지 (한글, 50자 이내)>

<상세 설명 (한글, 여러 줄 가능)>
- 변경 사항 1
- 변경 사항 2
- 변경 사항 3

Refs: <jira-ticket(s)>

Generated-By: Claude Code
```

### Prefix 규칙

| Prefix     | 용도                       |
| ---------- | -------------------------- |
| `feat`     | 새로운 기능 추가           |
| `fix`      | 버그 수정, 유지보수        |
| `refactor` | 리팩토링                   |
| `chore`    | 설정 변경, 의존성 업데이트 |

### Scope 규칙

- `dashboard.config.json`의 `scopes` 객체에서 참조
- 파일 경로에서 자동 판별
- 판별 불가 시 scope 생략

### Jira Ticket

- 브랜치 이름에서 추출 (예: `feature/AS-985-description`)
- Step 0에서 사용자가 선택/입력한 티켓 사용
- 복수 선택 시: `Refs: AS-100, AS-101`
- 없으면 Refs 라인 생략

---

## 3. 완료 메시지

### 성공 시

```
## Commit 완료

| 항목 | 값 |
|------|-----|
| Commit ID | `abc123...` |
| Branch | feature/AS-99 |
| Files Changed | 5 files |
| Lines | +150 / -30 |

### 커밋 메시지

\`\`\`
feat(auth): 로그인 페이지 구현

소셜 로그인 및 이메일 로그인을 지원하는 페이지를 구현했습니다.
- OAuth2 연동 (Google, GitHub)
- 이메일/비밀번호 폼 구현
- 로그인 실패 에러 메시지 처리

Refs: AS-100, AS-101

Generated-By: Claude Code
\`\`\`

### Jira 연동

| 티켓 | 코멘트 | Worklog |
|------|--------|---------|
| AS-100 | ✅ 등록됨 | 2h |
| AS-101 | ✅ 등록됨 | - |

### 다음 단계
평가 및 Dashboard API 전송은 `/push`를 실행하세요.
```

### Jira 연동 실패 시

```
## Commit 완료 (Jira 연동 실패)

커밋은 성공적으로 완료되었습니다.
Jira 연동 중 오류가 발생했습니다: {오류 내용}

수동으로 Jira 티켓({티켓ID})에 작업 내용을 업데이트해주세요.
```

---

<Tool_Usage>
- Use `AskUserQuestion` for all user-facing choices — provides clickable UI with contextual options. Never use text prompts or (Y/n) questions.
- Use `mcp__claude_ai_Atlassian__getJiraIssue` to fetch ticket info from branch name
- Use `mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql` to list subtasks or search issues
- Use `mcp__claude_ai_Atlassian__createJiraIssue` to create new subtasks when requested
- Use `mcp__claude_ai_Atlassian__addCommentToJiraIssue` to post commit summary to Jira after commit
- Use `mcp__claude_ai_Atlassian__addWorklogToJiraIssue` to log work time after commit
- Use `git branch --show-current`, `git status`, `git diff --staged` for git state
- Use `git commit -m` with HEREDOC for safe multiline commit messages
- Use `git rev-parse HEAD` after commit to capture commit ID

**Handoff**: After a successful commit, write `.omc/state/commit-handoff.json`:
```json
{
  "commitId": "<sha>",
  "branch": "<branch>",
  "jiraTickets": ["<ticket1>", "<ticket2>"],
  "filesChanged": <n>,
  "additions": <n>,
  "deletions": <n>
}
```
This file is consumed by `push-plugin:push` in the next pipeline step.
</Tool_Usage>
