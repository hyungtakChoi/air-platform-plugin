# AIR Platform Plugin

AIR Platform 팀 전용 Claude Code 플러그인 모음입니다.
개발자는 `/commit`, `/push`만 실행하면 Jira 연동, 생산성 측정, Dashboard 기록이 자동으로 처리됩니다.

## 플러그인 목록

| 플러그인      | 명령어    | 버전   | 설명                                          |
| ------------- | --------- | ------ | --------------------------------------------- |
| commit-plugin | `/commit` | v2.0.0 | 커밋 메시지 자동 생성 + Jira 티켓 연동        |
| push-plugin   | `/push`   | v1.0.7 | DevPulse Dashboard 전송 + Git Push             |

---

## 설치

### 1. 마켓플레이스 등록 (한 번만)

```bash
/plugin marketplace add git@github.com:air-platform-core/air-platform-plugin.git

/plugin marketplace list
# air-platform 표시되어야 함
```

### 2. 플러그인 설치

```bash
/plugin install commit-plugin@air-platform
/plugin install push-plugin@air-platform
```

### 3. Claude Code 재시작

### 4. 설치 확인

```bash
/plugin
# installed 탭에서 commit-plugin, push-plugin 확인
```

### 5. 사용

```bash
/commit-plugin:commit
/push-plugin:push
```

---

## 프로젝트 설정

각 프로젝트 루트에 `.claude/dashboard.config.json` 생성:

```bash
mkdir -p .claude
cat > .claude/dashboard.config.json << 'EOF'
{
  "developer": {
    "email": "your-email@mz.co.kr"
  },
  "product": {
    "id": "team-uuid/product-uuid"
  },
  "apiKey": "your-api-key"
}
EOF
```

> **보안**: `.claude/dashboard.config.json`을 `.gitignore`에 추가하세요.

```
.claude/dashboard.config.json
```

---

## commit-plugin 사용 흐름

```
/commit 실행
  → Jira 티켓 확인 (브랜치명에서 자동 추출)
  → 서브태스크 선택 or 검색 or 새로 생성
  → 작업 시간 입력 (선택, Enter 스킵)
  → 추가 컨텍스트 입력 (선택, Enter 스킵)
  → 커밋 메시지 자동 생성
  → git commit + Jira 코멘트/worklog 등록 (동시)
```

> Jira 관련 입력을 모두 Enter로 스킵하면 기존과 동일하게 동작합니다.

---

## 업데이트

```bash
/plugin marketplace update air-platform
claude plugin update commit-plugin@air-platform
claude plugin update push-plugin@air-platform
# Claude Code 재시작
```

---

## 플러그인 추가

새 플러그인은 `plugins/<plugin-name>/` 하위에 추가하면 됩니다:

```
plugins/
  new-plugin/
    .claude-plugin/plugin.json
    skills/<skill-name>/SKILL.md
```

`marketplace.json`의 `plugins` 배열에도 등록하세요.
