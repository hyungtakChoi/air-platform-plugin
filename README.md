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

### `/air-platform-kit:commit`

Git 커밋 메시지 자동 생성 + Jira 이슈 연동

```bash
/air-platform-kit:commit
```

**흐름:**
```
브랜치 티켓 추출
  ├── Task/Sub-task → 티켓 확인
  ├── Story/Epic   → 서브태스크 목록에서 선택 또는 새로 생성
  └── 없음         → 스프린트 스토리 추천 → 서브태스크 or Task 생성
  ↓
작업 시간 입력 (1h 단위)
  ↓
추가 컨텍스트 입력 (선택)
  ↓
커밋 메시지 자동 생성 → git commit
  ↓
Jira 코멘트 + worklog 등록
  ↓
Jira 상태 변경 여부 확인 (리뷰로 변경)
```

**커밋 메시지 포맷:**
```
<prefix>[(<scope>)]: <요약 (한글, 50자 이내)>

<상세 설명>
- 변경 사항 1

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

### `/air-platform-kit:start-task`

작업 시작 시 Jira 티켓 생성 및 In Progress 전환

```bash
/air-platform-kit:start-task [작업 설명]
```

**흐름:**
```
브랜치 티켓 추출
  ├── Story/Epic → 서브태스크 목록 확인 → 선택 또는 새로 생성
  ├── Task       → 해당 티켓 In Progress 전환
  └── 없음       → 스프린트 스토리 추천 → 서브태스크 or Task 생성
  ↓
생성된 티켓 In Progress 전환
```

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
