# AIR Platform Plugin

AIR Platform 팀 전용 Claude Code 플러그인입니다.
`/commit` 하나만 실행하면 Jira 티켓 연동, 커밋 메시지 자동 생성이 처리됩니다.

## 플러그인

| 플러그인 | 명령어 | 버전 | 설명 |
|---|---|---|---|
| commit-plugin | `/commit` | v3.0.0 | 커밋 메시지 자동 생성 + Jira 티켓 연동 |

---

## 설치

### 1. 마켓플레이스 등록 (한 번만)

```bash
/plugin marketplace add https://github.com/hyungtakChoi/air-platform-plugin.git
```

### 2. 플러그인 설치

```bash
/plugin install commit-plugin@air-platform
```

### 3. Claude Code 재시작 후 사용

```bash
/commit-plugin:commit
```

---

## commit-plugin 사용 흐름

```
/commit 실행
  → 브랜치에서 Jira 티켓 ID 추출
  │
  ├── 티켓 있음 (Task/Sub-task) → 티켓 확인
  │     ├── 맞아 → 해당 티켓 연결
  │     └── 아니야 → 새로 만들기
  │
  ├── 티켓 있음 (Story/Epic) → 서브태스크 목록 선택 (복수 선택 가능)
  │     ├── 선택 → 해당 서브태스크 연결
  │     └── 없어 → 새로 만들기
  │
  └── 티켓 없음 → 새로 만들기
        → LLM이 변경사항 분석 + 스프린트 스토리 목록 비교 → 관련 스토리 추천
        ├── 스토리 선택 → LLM이 서브태스크 제목 제안 → 서브태스크 생성
        └── 없어 → LLM이 Task 제목 제안 → 독립 Task 생성

  → 작업 시간 입력 (선택, 스킵 가능)
  → 추가 컨텍스트 입력 (필수)
  → 커밋 메시지 자동 생성 → git commit
  → Jira 코멘트 + worklog 등록
```

---

## 업데이트

```bash
/plugin marketplace update air-platform
/plugin install commit-plugin@air-platform
# Claude Code 재시작
```
