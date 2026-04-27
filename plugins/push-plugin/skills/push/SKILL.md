---
description: Git diff 분석 기반 평가 및 Dashboard API 전송
---

version: 1.0.4

# Post Agent

Git diff를 직접 분석하여 커밋을 평가하고, Dashboard API로 전송하는 에이전트입니다.

> **Note**: 이 에이전트는 `/push` 대신 사용할 수 있습니다.
>
> - `/push`: 커밋 메시지에서 evaluation/time 정보를 **파싱**합니다 (commit-plugin으로 생성된 커밋용)
> - `/post`: Git diff를 **직접 분석**하여 평가합니다 (모든 커밋 대응 가능)

---

## 1. 워크플로우

### Step 1: 설정 로드

```bash
cat .claude/dashboard.config.json
```

파일이 없으면 사용자에게 안내 후 중단합니다.

### Step 2: Unpushed 커밋 확인

```bash
# upstream 브랜치 대비 unpushed 커밋 개수 확인
git rev-list --count @{u}..HEAD

# unpushed 커밋 목록 (오래된 순서로)
git log @{u}..HEAD --reverse --format="%H"

# upstream이 없는 경우 (새 브랜치): 원격 기본 브랜치 자동 감지
git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/@@'
# 결과 예: origin/main, origin/develop, origin/master 등

# 감지된 기본 브랜치로 비교
git log <detected-branch>..HEAD --reverse --format="%H"

# merge commit 확인 (부모가 2개 이상인 커밋)
git rev-list --merges @{u}..HEAD
```

**원격 기본 브랜치 감지 실패 시:**

`git symbolic-ref` 명령이 실패하는 경우 (원격 HEAD가 설정되지 않은 경우):

1. 다음 에러 메시지 출력:

   ```
   ## 오류

   원격 저장소의 기본 브랜치를 자동 감지할 수 없습니다.

   ### 해결 방법
   다음 명령어로 원격 HEAD를 설정하세요:

   git remote set-head origin --auto

   또는 비교할 base 브랜치를 직접 지정해주세요.
   ```

2. 사용자에게 base 브랜치 입력 요청
3. 입력받은 브랜치로 비교 진행

**커밋이 없으면:**

```
Push할 커밋이 없습니다.
```

**Merge commit 필터링:**

- Merge commit은 평가 및 Dashboard 전송에서 **제외**
- Git Push는 모든 커밋 포함 (merge commit 포함)
- 모든 커밋이 merge commit인 경우: API 전송 없이 Push만 실행

### Step 3: Diff 분석 및 평가

각 커밋에 대해 `git show <hash>`로 diff를 분석하고 평가합니다.

```bash
# 커밋별 diff 분석
git show <commit-hash> --stat --format="%H%n%B"
git show <commit-hash> --format=""
```

#### 3.1 분석 항목

각 커밋에서 다음을 분석합니다:

- **변경 파일 목록**: 어떤 파일이 수정되었는지
- **코드 변경량**: 추가/삭제된 라인 수
- **변경 내용**: 실제 diff 내용
- **커밋 메시지**: 작업 의도 파악

#### 3.2 평가 수행

분석 결과를 바탕으로 다음 평가를 수행합니다:

- **complexity**: 코드/설계의 복잡성 (0.0 ~ 10.0)
- **volume**: 변경된 코드의 양 (0.0 ~ 10.0)
- **thinking**: 설계/아키텍처 고려 수준 (0.0 ~ 10.0)
- **others**: 품질, 가독성, 테스트, 문서화 등 (0.0 ~ 10.0)
- **total**: 모든 항목의 합계 (0.0 ~ 40.0)
- **comment**: 작업의 핵심 가치를 한글 1-2문장으로 설명

#### 3.3 시간 추정

- **Human-Time**: 3-5년차 개발자가 혼자 완료하는데 걸리는 예상 시간
- **Ai-Driven-Time**: AI 도구를 사용했을 때 걸리는 예상 시간
- **productivity**: (Human-Time / Ai-Driven-Time) × 100%

### Step 4: 평가 검증

평가 결과가 기준에 맞는지 검증합니다. **검증 실패 시 Step 3으로 돌아가 재평가합니다.**

#### 4.1 점수 범위 검증

| 항목       | 유효 범위  | 현재 값 | 통과 여부 |
| ---------- | ---------- | ------- | --------- |
| complexity | 0.0 ~ 10.0 | ?       | /         |
| volume     | 0.0 ~ 10.0 | ?       | /         |
| thinking   | 0.0 ~ 10.0 | ?       | /         |
| others     | 0.0 ~ 10.0 | ?       | /         |
| **total**  | 0.0 ~ 40.0 | ?       | /         |

#### 4.2 합계 검증

```
total = complexity + volume + thinking + others
```

#### 4.3 점수 분포 논리 검증

| 항목       | 점수 | 기준 범위 매칭                           | 통과 여부 |
| ---------- | ---- | ---------------------------------------- | --------- |
| complexity | ?    | 점수가 작업 복잡도와 일치하는가?         | /         |
| volume     | ?    | 점수가 변경된 라인/파일 수와 일치하는가? | /         |
| thinking   | ?    | 점수가 설계 고려 수준과 일치하는가?      | /         |
| others     | ?    | 점수가 추가 작업 수준과 일치하는가?      | /         |

### Step 5: API Input 검증

모든 커밋에 대해 입력 데이터를 검증합니다. **검증 실패 시 중단하고 오류를 수정합니다.**

#### 5.1 구조 검증

```
CreateCommitLogsInput
├── developer (DeveloperInput)
│   ├── developerEmail: String!
│   └── teamId: ID!
├── product (ProductInput)
│   └── id: ID!
└── commits: [CommitInput!]!
    └── [i] (CommitInput)
        ├── commitId: String!
        ├── message: String!
        ├── type: String!
        ├── comment: String
        ├── evaluation (EvaluationInput)
        ├── lineAdded: Int!
        ├── lineDeleted: Int!
        ├── humanHours: Float!
        ├── aiHours: Float!
        ├── productivity: Float (optional)
        └── version: String!
```

#### 5.2 타입 검증

| 필드               | 예상 타입 | 유효 범위       |
| ------------------ | --------- | --------------- |
| `commitId`         | String    | 40자 해시       |
| `type`             | String    | feat/fix/...    |
| `evaluation.total` | Float     | 0.0 ~ 40.0      |
| `lineAdded`        | Int       | >= 0            |
| `lineDeleted`      | Int       | >= 0            |
| `humanHours`       | Float     | > 0             |
| `aiHours`          | Float     | > 0             |
| `productivity`     | Float     | >= 0 (optional) |

### Step 6: Dashboard API 전송

모든 검증 통과 후 GraphQL API를 호출하여 **모든 커밋 정보를 한 번에** 전송합니다.

### Step 7: Git Push

```bash
git push
```

---

## 2. 평가 기준 (Evaluation Criteria)

### 점수 범위

| 항목       | 범위       | 설명                                   |
| ---------- | ---------- | -------------------------------------- |
| complexity | 0.0 ~ 10.0 | 코드/설계의 복잡성                     |
| volume     | 0.0 ~ 10.0 | 변경된 코드의 양                       |
| thinking   | 0.0 ~ 10.0 | 설계/아키텍처 고려 시간                |
| others     | 0.0 ~ 10.0 | 기타 (품질, 가독성, 테스트, 문서화 등) |
| **total**  | 0.0 ~ 40.0 | 모든 항목의 합계                       |

### Complexity (복잡성) - 0.0 ~ 10.0

| 점수       | 기준                                         |
| ---------- | -------------------------------------------- |
| 0.0 - 1.0  | 단순 오타 수정, 설정 변경                    |
| 1.1 - 3.0  | 단순 버그 수정, 단일 파일 변경               |
| 3.1 - 5.0  | 여러 파일 수정, 기존 패턴 따라 구현          |
| 5.1 - 7.0  | 새로운 기능 구현, API 연동, 복잡한 로직      |
| 7.1 - 10.0 | 아키텍처 설계, 복잡한 상태 관리, 성능 최적화 |

### Volume (코드량) - 0.0 ~ 10.0

| 점수       | 기준                                |
| ---------- | ----------------------------------- |
| 0.0 - 1.0  | 1-10 lines 변경                     |
| 1.1 - 3.0  | 11-50 lines 변경                    |
| 3.1 - 5.0  | 51-100 lines 변경 또는 3-5개 파일   |
| 5.1 - 7.0  | 101-300 lines 변경 또는 6-10개 파일 |
| 7.1 - 10.0 | 300+ lines 변경 또는 10개 이상 파일 |

### Thinking (설계 고려) - 0.0 ~ 10.0

| 점수       | 기준                                |
| ---------- | ----------------------------------- |
| 0.0 - 1.0  | 즉시 구현 가능                      |
| 1.1 - 3.0  | 기존 코드 분석 필요                 |
| 3.1 - 5.0  | 여러 접근법 비교 필요               |
| 5.1 - 7.0  | 아키텍처 결정 필요                  |
| 7.1 - 10.0 | 복잡한 설계 검토, 트레이드오프 분석 |

### Others (기타) - 0.0 ~ 10.0

| 점수       | 기준                                                 |
| ---------- | ---------------------------------------------------- |
| 0.0 - 1.0  | 추가 작업 없음, 기본적인 코드 품질                   |
| 1.1 - 3.0  | 새로운 라이브러리/기술 도입, 가독성 개선             |
| 3.1 - 5.0  | 단위 테스트 작성, 코드 품질 향상                     |
| 5.1 - 7.0  | 통합 테스트 + 문서화 + 유지보수성 개선               |
| 7.1 - 10.0 | 종합적인 테스트 + 문서화 + 리팩토링 + 높은 코드 품질 |

### Comment (정성 평가)

작업의 핵심 가치, 영향, 주목할 만한 점을 한글로 1-2문장으로 작성합니다.

---

## 3. 시간 추정 가이드

### 정의

- **Human-Time**: 3-5년차 주니어 개발자가 혼자 완료하는데 걸리는 예상 시간
- **Ai-Driven-Time**: AI 도구를 사용했을 때 걸리는 예상 시간
- **productivity**: (Human-Time / Ai-Driven-Time) × 100%

### Human-Time 참고 기준

| Total 점수 | 예상 Human-Time |
| ---------- | --------------- |
| 0 - 5      | 30m - 1h        |
| 5 - 10     | 1h - 2h         |
| 10 - 20    | 2h - 4h         |
| 20 - 30    | 4h - 8h (1일)   |
| 30 - 40    | 1-2일           |

### 작업 유형별 AI 효율성 배수

| 작업 유형                    | AI 효율성 배수 | 설명                             |
| ---------------------------- | -------------- | -------------------------------- |
| 보일러플레이트/반복 코드     | 8x - 12x       | AI가 가장 효율적인 영역          |
| CRUD 기능 구현               | 6x - 10x       | 패턴화된 작업, AI 생성 효과 높음 |
| 버그 수정 (명확한 원인)      | 4x - 8x        | 원인 파악 후 수정은 빠름         |
| 새로운 기능 (기존 패턴 따름) | 4x - 6x        | 컨텍스트 이해 필요               |
| API 연동/외부 서비스 통합    | 3x - 5x        | 문서 참조, 테스트 필요           |
| 복잡한 비즈니스 로직         | 2x - 4x        | 도메인 이해 필요, AI 효율 감소   |
| 아키텍처 설계/리팩토링       | 2x - 3x        | 인간 판단 중요, AI는 보조 역할   |
| 디버깅 (복잡한 원인)         | 1.5x - 3x      | 탐색적 작업, AI 효율 제한적      |
| 성능 최적화                  | 1.5x - 2.5x    | 측정/분석 필요, 전문 지식 요구   |

### 계산 방법

```
Ai-Driven-Time = Human-Time ÷ AI 효율성 배수
```

---

## 4. 설정 파일

### 설정 파일 구조

```json
{
  "developer": {
    "email": "developer@example.com"
  },
  "product": {
    "id": "team-uuid/product-uuid"
  },
  "apiKey": "your-api-key"
}
```

### product.id 파싱 규칙

```
product.id = "{team.id}/{product.id}"
```

### API 전송 시 변환

| Config 값           | API 필드                   | 변환 방법                 |
| ------------------- | -------------------------- | ------------------------- |
| `developer.email`   | `developer.developerEmail` | 그대로 사용               |
| `product.id` 앞부분 | `developer.teamId`         | `/`로 split 후 첫 번째 값 |
| `product.id` 뒷부분 | `product.id`               | `/`로 split 후 두 번째 값 |
| `apiKey`            | `x-api-key` 헤더           | 그대로 사용               |

---

## 5. Dashboard API 호출

### GraphQL Mutation

```graphql
mutation CreateCommitLogs($input: CreateCommitLogsInput!) {
  createCommitLogs(input: $input) {
    id
    commitId
    createdAt
  }
}
```

### Input 스키마

```graphql
input CreateCommitLogsInput {
  developer: DeveloperInput!
  product: ProductInput!
  commits: [CommitInput!]!
}

input DeveloperInput {
  developerEmail: String!
  teamId: ID!
}

input ProductInput {
  id: ID!
}

input CommitInput {
  commitId: String!
  message: String!
  type: String!
  comment: String
  evaluation: EvaluationInput!
  lineAdded: Int!
  lineDeleted: Int!
  humanHours: Float!
  aiHours: Float!
  productivity: Float
  version: String!
}

input EvaluationInput {
  total: Float!
  complexity: Float!
  volume: Float!
  thinking: Float!
  others: Float!
}
```

### API 호출 예시

```bash
curl -X POST "https://api.devpulse.platform-eng.megaone.com/graphql" \
  -H "Content-Type: application/json" \
  -H "x-api-key: <api-key>" \
  -d '{
    "query": "mutation CreateCommitLogs($input: CreateCommitLogsInput!) { createCommitLogs(input: $input) { id commitId } }",
    "variables": {
      "input": {
        "developer": {
          "developerEmail": "developer@example.com",
          "teamId": "<team-id>"
        },
        "product": {
          "id": "<product-id>"
        },
        "commits": [
          {
            "commitId": "abc123...",
            "message": "feat: 기능 추가...",
            "type": "feat",
            "comment": "diff 분석 기반 평가",
            "evaluation": {
              "total": 12.0,
              "complexity": 4.0,
              "volume": 3.0,
              "thinking": 3.0,
              "others": 2.0
            },
            "lineAdded": 80,
            "lineDeleted": 10,
            "humanHours": 2.0,
            "aiHours": 0.3,
            "productivity": 667,
            "version": "1.0.0"
          }
        ]
      }
    }
  }'
```

---

## 6. 에러 처리

### 설정 파일 누락

```
## 오류

`.claude/dashboard.config.json` 파일이 없습니다.

### 설정 파일 생성
다음 형식으로 파일을 생성하세요:

{
  "developer": { "email": "your-email@example.com" },
  "product": { "id": "team-uuid/product-uuid" },
  "apiKey": "your-api-key"
}
```

### 분석 실패

Diff 분석이 실패한 경우 해당 커밋에 대해 사용자에게 정보를 요청합니다.

### API 실패

1. **롤백하지 않음** - 커밋은 이미 로컬에 존재하므로 유지
2. 에러 메시지 출력
3. 사용자에게 수동 재시도 안내

### 부분 실패

일부 커밋만 분석에 실패한 경우:

1. 성공한 커밋 목록 표시
2. 실패한 커밋 목록과 이유 표시
3. 사용자에게 선택지 제공:
   - 성공한 커밋만 전송
   - 실패한 커밋 정보 수동 입력
   - 전체 중단

---

## 7. 완료 메시지

### 성공 시 (여러 커밋)

```
## Push 완료

### 요약
| 항목 | 값 |
|------|-----|
| Branch | feature/COM-100 |
| Commits | 3개 (평가: 2개, 스킵: 1개) |
| Total Lines | +125 / -65 |

### 커밋 목록
| # | Commit ID | Type | Total | Human-Time | Ai-Time | Productivity |
|---|-----------|------|-------|------------|---------|--------------|
| 1 | `abc123` | feat | 12.0 | 2h | 18m | 667% |
| 2 | `def456` | fix | 6.0 | 1h | 9m | 667% |
| - | `ghi789` | merge | - | - | - | (스킵) |

### 전체 통계
| 항목 | 값 |
|------|-----|
| Total Evaluation | 18.0 |
| Total Human-Time | 3h |
| Total Ai-Driven-Time | 27m |
| Avg Productivity | 667% |

### 상태
- Diff 분석 완료 (2개 커밋)
- Merge commit 스킵 (1개)
- Dashboard API 전송 성공
- Git Push 완료
```

### 성공 시 (단일 커밋)

```
## Post 완료

| 항목 | 값 |
|------|-----|
| Commit ID | `abc123...` |
| Branch | feature/COM-100 |
| Type | feat |
| Lines | +150 / -30 |

### 평가
| 항목 | 점수 |
|------|------|
| complexity | 5.0 |
| volume | 4.0 |
| thinking | 3.0 |
| others | 3.0 |
| **total** | **15.0** |

### 시간
| 항목 | 값 |
|------|-----|
| Human-Time | 4h |
| Ai-Driven-Time | 30m |
| Productivity | 800% |

### 상태
- Diff 분석 완료
- Dashboard API 전송 성공
- Git Push 완료
```

---

## 8. push-plugin과의 차이점

| 항목      | push-plugin                   | post-plugin           |
| --------- | ----------------------------- | --------------------- |
| 분석 대상 | 커밋 메시지                   | git diff              |
| 평가 방식 | 파싱 (기존 evaluation 읽기)   | 직접 분석 (새로 평가) |
| 대상 커밋 | commit-plugin으로 생성된 커밋 | 모든 커밋             |

---

## Important Guidelines

- **Git diff를 직접 분석**하여 평가합니다 (커밋 메시지 파싱 아님)
- **커밋 메시지는 원본 그대로 API에 전송**합니다 (요약하거나 수정하지 않음)
- 모든 unpushed 커밋을 한 번에 처리합니다
- **Merge commit은 평가 및 Dashboard 전송에서 제외**합니다 (Git Push는 포함)
- **평가 검증 통과 시 사용자 확인 없이 자동으로 API 전송 및 Push를 실행합니다**
- API 실패 시 **커밋을 롤백하지 않습니다**
- Push 전에 반드시 API 전송이 성공해야 합니다 (평가 대상 커밋이 없으면 바로 Push)
- 커밋은 시간순(오래된 것부터)으로 처리합니다
- Dashboard API 호출 시 `commits[].version` 필드는 이 파일 최상단의 `version` 값(현재: `1.0.4`)을 사용합니다
