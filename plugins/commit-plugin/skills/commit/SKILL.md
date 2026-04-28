---
description: Git 커밋 메시지를 자동으로 생성하고 Jira 이슈를 연동합니다
---

version: 2.0.0

# Commit Agent

변경사항을 분석하여 git commit 메시지를 생성하고, Jira 이슈에 작업 내용을 자동으로 기록하는 에이전트입니다.

> **Note**: 평가(evaluation)와 Dashboard API 호출은 `/push`를 사용하세요.

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
2. `AskUserQuestion` 도구로 선택지를 표시합니다 (multiSelect: true):
   - 각 서브태스크를 옵션으로 표시 (label: 티켓ID, description: 요약)
   - "해당 없음 — 스토리만 연결" 옵션 포함
   - "새 서브태스크 생성" 옵션 포함
3. "해당 없음" 선택 시 → 스토리 티켓 자체를 Refs로 사용
4. "새 서브태스크 생성" 선택 시 → Step 0.3으로 이동

#### 0.2 시나리오 B: 서브태스크 티켓

이슈 타입이 Sub-task/Subtask이면:
`AskUserQuestion` 도구로 확인 요청:
- question: "연결된 Jira 티켓: {티켓ID} [{요약}] — 이 티켓으로 진행할까요?"
- options: ["네, 이 티켓으로 진행", "다른 티켓 선택 (검색)"]

#### 0.3 시나리오 C: 브랜치에 티켓 ID 없음

단계별 폴백으로 처리합니다:

**Step C-1: Jira 이슈 검색**

`mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql`로 관련 이슈 검색:
```
project = {브랜치에서 추출한 프로젝트 키, 없으면 dashboard.config.json의 jiraProjectKey} AND assignee = currentUser() AND updated >= -14d ORDER BY updated DESC
```
검색 결과 목록을 표시하고 선택 요청. "없음" 옵션 포함.

**Step C-2: 직접 입력**

`AskUserQuestion` 도구로 입력 요청:
- question: "연결할 Jira 티켓 ID를 입력하세요 (예: BI-123)"
- options: ["직접 입력", "티켓 없이 커밋", "새 서브태스크 생성"]

**Step C-3: 새 서브태스크 생성**

"새 서브태스크 생성" 선택 시:
`AskUserQuestion` 도구로 부모 스토리 ID 입력 요청 후 `mcp__claude_ai_Atlassian__createJiraIssue`로 생성.
"티켓 없이 커밋" 선택 시: Refs 없이 커밋 진행.

#### 0.4 MCP 연결 실패 처리

`mcp__claude_ai_Atlassian__*` 도구 호출 실패 시:
- 경고 메시지 출력 후 Step 1로 진행 (커밋은 정상 실행)

---

### Step 0.5: 작업 시간 입력

Jira 티켓이 선택된 경우에만 `AskUserQuestion` 도구로 표시:
- question: "이 작업에 소요된 시간을 선택하세요"
- options: ["30m", "1h", "2h", "3h 이상 / 직접 입력", "스킵 (기록 안 함)"]
- 지원 형식: `Xh`, `Xm`, `XhYm`

---

### Step 0.6: Jira 추가 컨텍스트 입력

Jira 티켓이 선택된 경우에만 `AskUserQuestion` 도구로 표시:
- question: "Jira 티켓에 남길 추가 컨텍스트가 있나요? (커밋 메시지에 담지 못한 배경, 시도한 방법, 주의사항 등)"
- options: ["직접 입력", "스킵"]

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

## Important Guidelines

- **staged 파일이 있으면 해당 파일만 커밋하고, 절대로 다른 파일을 추가 stage하지 않는다**
- staged 파일이 없는 경우에만 모든 변경 파일을 auto-stage한다
- **포맷 검증 통과 시 사용자 확인 없이 자동으로 커밋을 실행한다**
- **Step 0의 모든 Jira 입력은 Enter로 스킵 가능하며, 모두 스킵하면 기존 v1.1.0과 동일하게 동작한다**
- Jira API 실패는 커밋을 막지 않는다 — 경고 출력 후 커밋 진행
- 커밋 메시지는 반드시 한글로 작성 (prefix와 scope는 영문 유지)
- 요약 메시지는 50자 이내로 간결하게 작성
- 상세 설명은 변경 사항을 bullet point로 구체적으로 나열
- 사용자가 커밋 메시지 내용을 제공하면 해당 내용을 포맷에 맞게 반영
- Handle edge cases gracefully: unstaged changes, detached HEAD, Jira MCP 미설치
