# 워크숍: 리테일 식료품 AI 에이전트 배포하기 (워크스페이스 전용)

이 가이드는 L300 AI 에이전트 앱을 **Databricks 워크스페이스 안에서만** 배포하는 과정을 안내합니다. 로컬 머신 설정이나 노트북(랩톱)에 터미널 툴을 설치할 필요가 없습니다.

가이드를 마치면 장기 메모리, 문서 검색, 데이터 질의 기능을 갖춘 풀스택 채팅 애플리케이션을 Databricks App으로 배포하게 됩니다.

---

## 사전 준비 사항

시작하기 전에 Databricks 워크스페이스에 다음 기능이 활성화되어 있는지 확인하세요(확실하지 않으면 워크스페이스 관리자에게 문의하세요):

- **Unity Catalog** — 데이터 테이블과 권한 관리
- **Databricks Apps** — 채팅 애플리케이션 호스팅
- **Lakebase** — 관리형 PostgreSQL 데이터베이스 (에이전트 메모리 + 채팅 이력 저장)
- **Vector Search** — 시맨틱 문서 검색(retrieval)
- **Foundation Model API** — Databricks 엔드포인트를 통한 Claude 사용
- **Web Terminal** (웹 터미널) — 브라우저 내 터미널 (Settings > Developer > Web Terminal)
- **실행 중인 SQL warehouse** — 데이터 테이블 생성에 필요 (Compute > SQL Warehouses)

---

## 1단계: 리포지토리를 워크스페이스로 가져오기

코드를 Databricks 워크스페이스로 가져와 직접 수정하고 배포할 수 있게 합니다.

1. 왼쪽 사이드바에서 **Workspace** > **Repos** 클릭 ("Git Folders"로 표시될 수도 있습니다)
2. **Add** > **Git Folder** 클릭
3. 리포지토리 URL 붙여넣기: `https://github.com/AnanyaDBJ/databricks-ai-workshops.git`
4. **Create Git Folder** 클릭

가져오기가 완료되면 코드는 다음과 같은 경로에 있습니다:
`/Workspace/Repos/<your-username>/databricks-ai-workshops/`

---

## 2단계: 데이터 설정

[`data/README.md`](../data/README.md#path-b-워크스페이스-노트북)의 **Path B (워크스페이스 노트북)** 를 완료하세요.

완료되면 노트북 출력에서 다음 4개 값을 저장해 두세요 — 5단계에서 필요합니다:

| 복사할 값 | 형태 예시 | 사용하는 단계 |
|------|---------|---------|
| MLflow Experiment ID | `1234567890123456` | 5단계 (`databricks.yml`) |
| Vector Search Index 이름 | `my_catalog.my_schema.policy_docs_index` | 5단계 (`databricks.yml`) |
| Genie Space ID | `01abcdef12345678` | 5단계 (`databricks.yml`) |
| Catalog.Schema | `my_catalog.my_schema` | 7단계 (권한) |

---

## 3단계: Lakebase 인스턴스 생성

Lakebase는 Databricks 안의 관리형 PostgreSQL 데이터베이스입니다. 앱은 이를 다음 용도로 사용합니다:
- **에이전트 장기 메모리** (세션이 바뀌어도 사용자 선호를 기억)
- **채팅 UI 이력** (대화 사이드바 영구 저장)

### 인스턴스 생성

1. 왼쪽 사이드바에서 **Compute** > **Lakebase**로 이동
2. **Create Project** 클릭 > **Autoscaling** 선택
3. 기억하기 쉬운 이름 지정 (예: `l300-workshop`)
4. Create 클릭 — `production` 브랜치가 자동으로 생성됩니다 (직접 브랜치 이름을 지정할 수도 있습니다)
5. "Ready" 상태가 될 때까지 1~2분 정도 기다리세요

### 연결 정보 확인

앱이 데이터베이스 위치를 알 수 있도록 두 가지 경로가 필요합니다:

**옵션 A — Lakebase UI에서:**
1. 프로젝트 이름 클릭
2. 브랜치(예: `production`) 클릭
3. **프로젝트 이름**과 **브랜치 이름**을 기록

**옵션 B — 노트북에서:**
```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
# List databases in your branch
dbs = w.api_client.do("GET", f"/api/2.0/postgres/projects/<project-name>/branches/<branch-name>/databases")
print(dbs)
```

여기에서 다음 값을 기록하세요:
- **프로젝트 이름**: 예: `l300-workshop`
- **브랜치 이름**: 예: `production` 또는 `l300-workshop-branch`
- **브랜치 경로**: `projects/l300-workshop/branches/production`
- **데이터베이스 경로**: `projects/l300-workshop/branches/production/databases/databricks-postgres`

---

## 3단계 (대안): Lakebase Provisioned 인스턴스 사용

워크스페이스에서 Autoscaling 대신 **Provisioned** Lakebase 인스턴스를 사용한다면 다음 단계를 따르세요:

1. 왼쪽 사이드바에서 **Compute** > **Lakebase**로 이동
2. **Create** 클릭 > **Provisioned** 선택
3. 기억하기 쉬운 이름 지정 (예: `l300-workshop`)
4. 용량 선택 (예: 가장 작은 크기인 `CU_1`)
5. Create를 클릭하고 "Running" 상태가 될 때까지 대기

필요한 값은 **인스턴스 이름** 하나뿐입니다 — 직접 지정한 이름 그대로입니다 (예: `l300-workshop`).

5단계에 도달하면 "Provisioned Lakebase" 안내에 따라 다른 구성을 적용하세요.

---

## 4단계: MLflow Experiment ID 확인

데이터 설정 노트북(2단계)이 이미 MLflow 실험(experiment)을 생성해 두었습니다. **Experiment ID**를 확인하세요:

1. 왼쪽 사이드바에서 **Experiments** (실험) 클릭
2. 노트북이 생성한 실험을 찾으세요 (2단계 출력에 표시된 이름 참고)
3. 클릭하면 — 브라우저 URL에 표시되는 긴 숫자가 **Experiment ID**입니다 (예: `.../ml/experiments/1234567890123456`에서 `1234567890123456` 부분)

**찾을 수 없거나 새로 만들어야 한다면:**
1. 왼쪽 사이드바에서 **Experiments**로 이동
2. 오른쪽 위의 **Create Experiment** 클릭
3. 이름 지정 (예: `l300-retail-agent`)
4. Create 클릭 — URL에서 Experiment ID를 복사

---

## 5단계: `databricks.yml` 편집

워크스페이스 파일 브라우저에서 `advanced/databricks.yml`로 이동해 편집하세요. 이 파일은 Databricks가 앱을 어떻게 배포하고 어떤 리소스가 필요한지를 정의합니다.

### 다음 값을 찾아 바꾸세요:

| 찾을 값 | 바꿀 값 | 출처 |
|---|---|---|
| `"4040511589772447"` (experiment_id) | 본인의 Experiment ID | 4단계 |
| `"01f1595900051132b572b1f717285d6f"` (space_id) | 본인의 Genie Space ID | 2단계 |
| `"ai_days_experiment_catalog.retail_agent.policy_docs_index"` (securable_full_name) | 본인의 `catalog.schema.policy_docs_index` | 2단계 |
| `"l300-workshop-smoke-test"` (LAKEBASE_AUTOSCALING_PROJECT value) | 본인의 Lakebase 프로젝트 이름 | 3단계 |
| `"l300-workshop-smoke-test-branch"` (LAKEBASE_AUTOSCALING_BRANCH value) | 본인의 Lakebase 브랜치 이름 | 3단계 |
| `"projects/l300-workshop-smoke-test/branches/l300-workshop-smoke-test-branch"` (postgres branch) | 본인의 브랜치 경로 | 3단계 |
| `"projects/l300-workshop-smoke-test/branches/l300-workshop-smoke-test-branch/databases/databricks-postgres"` (postgres database) | 본인의 데이터베이스 경로 | 3단계 |

### 편집 후 주요 섹션은 다음과 같은 모습이어야 합니다:

```yaml
bundle:
  name: retail_grocery_ltm_memory

resources:
  apps:
    retail_grocery_ltm_memory:
      name: "retail-grocery-ltm-memory"
      description: "Retail grocery conversational agent with Genie, Vector Search, and long-term memory"
      source_code_path: ./
      config:
        command: ["uv", "run", "start-app"]
        env:
          - name: MLFLOW_TRACKING_URI
            value: "databricks"
          - name: MLFLOW_REGISTRY_URI
            value: "databricks-uc"
          - name: API_PROXY
            value: "http://localhost:8000/invocations"
          - name: CHAT_APP_PORT
            value: "3000"
          - name: CHAT_PROXY_TIMEOUT_SECONDS
            value: "300"
          - name: MLFLOW_EXPERIMENT_ID
            value_from: "experiment"
          - name: LAKEBASE_AUTOSCALING_PROJECT
            value: "l300-workshop"                    # ← your project name
          - name: LAKEBASE_AUTOSCALING_BRANCH
            value: "production"                       # ← your branch name
          - name: GENIE_SPACE_ID
            value_from: "retail_grocery_genie"
          - name: VECTOR_SEARCH_INDEX
            value_from: "policy_docs_index"
      resources:
        - name: "experiment"
          experiment:
            experiment_id: "1234567890123456"         # ← from Step 4
            permission: "CAN_MANAGE"
        - name: "retail_grocery_genie"
          genie_space:
            name: "Retail Data"
            space_id: "01abcdef12345678"              # ← from Step 2
            permission: "CAN_RUN"
        - name: "postgres"
          postgres:
            branch: "projects/l300-workshop/branches/production"                              # ← from Step 3
            database: "projects/l300-workshop/branches/production/databases/databricks-postgres"  # ← from Step 3
            permission: "CAN_CONNECT_AND_CREATE"
        - name: "policy_docs_index"
          uc_securable:
            securable_full_name: "my_catalog.my_schema.policy_docs_index"   # ← from Step 2
            securable_type: "TABLE"
            permission: "SELECT"

targets:
  dev:
    mode: development
    default: true
    workspace:
      root_path: /Workspace/Users/${workspace.current_user.userName}/.bundle/retail_grocery_ltm_memory/${bundle.target}
```

### Provisioned Lakebase — 다른 구성

Provisioned 인스턴스(3단계 대안)를 사용한다면 대신 다음과 같이 변경하세요:

1. `postgres` 리소스를 `database` 리소스로 교체:
```yaml
        - name: "postgres"
          database:
            database_name: databricks_postgres
            instance_name: "l300-workshop"            # ← your instance name
            permission: CAN_CONNECT_AND_CREATE
```

2. Lakebase 환경 변수 교체:
```yaml
          - name: LAKEBASE_INSTANCE_NAME
            value: "l300-workshop"                    # ← your instance name
```
(`LAKEBASE_AUTOSCALING_PROJECT`와 `LAKEBASE_AUTOSCALING_BRANCH`는 제거하세요)

---

## 6단계: 앱 배포

**Web Terminal**을 여세요: 워크스페이스 하단 패널의 `>_` 터미널 아이콘을 클릭하거나, Settings > Developer > Web Terminal로 이동합니다.

### 배포 단계

1. `advanced` 폴더로 이동:
   ```bash
   cd /Workspace/Repos/<your-username>/databricks-ai-workshops/advanced
   ```

2. 구성 검증 (`databricks.yml`의 오류 확인):
   ```bash
   databricks bundle validate -t dev
   ```
   오류가 출력되면 5단계로 돌아가 플레이스홀더를 수정하세요.

3. 번들 배포 (코드 업로드 및 앱 등록):
   ```bash
   databricks bundle deploy -t dev
   ```

4. 앱 시작:
   ```bash
   databricks apps start retail-grocery-ltm-memory
   ```

5. 앱이 RUNNING 상태가 될 때까지 대기 (첫 시작은 3~5분 소요):
   ```bash
   databricks apps get retail-grocery-ltm-memory --output json | jq -r '.status.state'
   ```
   `RUNNING`이 표시될 때까지 반복 실행하세요.

6. 실행 중인 앱에 소스 코드 배포:
   ```bash
   DATABRICKS_USERNAME=$(databricks current-user me | jq -r .userName)
   databricks apps deploy retail-grocery-ltm-memory \
     --source-code-path /Workspace/Users/$DATABRICKS_USERNAME/.bundle/retail_grocery_ltm_memory/dev/files
   ```

---

## 7단계: 권한 부여

앱은 서비스 프린시펄로 실행되며, 이 프린시펄이 리소스에 접근할 수 있어야 합니다.

### 서비스 프린시펄 ID 확인

```bash
SP_CLIENT_ID=$(databricks apps get retail-grocery-ltm-memory --output json | jq -r '.service_principal_client_id')
echo "App SP Client ID: $SP_CLIENT_ID"
```

### Unity Catalog 접근 권한 부여

SQL Editor 또는 노트북에서 실행하세요:

```sql
GRANT USE CATALOG ON CATALOG <your-catalog> TO `<SP_CLIENT_ID>`;
GRANT USE SCHEMA ON SCHEMA <your-catalog>.<your-schema> TO `<SP_CLIENT_ID>`;
GRANT SELECT ON SCHEMA <your-catalog>.<your-schema> TO `<SP_CLIENT_ID>`;
```

### Genie Space 접근 권한 부여

1. 왼쪽 사이드바에서 본인의 **Genie Space**로 이동
2. 오른쪽 위의 **Share** (공유) 클릭
3. 서비스 프린시펄을 추가 (클라이언트 ID로 검색)하고 **Can Run** 권한 부여

### Lakebase 접근 권한 부여 (필요한 경우)

Lakebase 관련 "permission denied" 오류가 보이면 Web Terminal에서 권한 스크립트를 실행하세요:

```bash
cd /Workspace/Repos/<your-username>/databricks-ai-workshops/advanced
pip install databricks-sdk
python scripts/grant_lakebase_permissions.py "$SP_CLIENT_ID" \
  --memory-type langgraph-short-term --project <project-name> --branch <branch-name>
python scripts/grant_lakebase_permissions.py "$SP_CLIENT_ID" \
  --memory-type langgraph-long-term --project <project-name> --branch <branch-name>
```

---

## 8단계: 배포 확인

### 앱 상태 확인

1. 왼쪽 사이드바에서 **Compute** > **Apps**로 이동
2. 본인 앱을 찾으세요 — 상태 표시가 초록색으로 바뀌며 **Running**이 표시되어야 합니다
3. **Starting**이 표시되면 몇 분 더 기다리세요 (의존성을 설치하는 중입니다)
4. **Crashed**가 표시되면 아래 트러블슈팅 섹션을 참고하세요

### 로그 확인 (이상이 있어 보일 때)

1. 앱 이름 클릭 > **Logs** 탭으로 이동
2. 다음 줄이 보이면 정상 동작 중입니다:
   - `"Uvicorn running on http://0.0.0.0:8000"` — 백엔드 준비 완료
   - `"Server is running on http://localhost:3000"` — 프런트엔드 준비 완료

### 앱 테스트

1. 앱 페이지 상단에 표시된 **앱 URL**을 클릭해 채팅 인터페이스를 여세요
2. 다음 테스트 프롬프트를 시험해 보세요:

| 프롬프트 | 테스트 대상 |
|--------|---------------|
| "매출 기준 상위 5개 제품은 무엇인가요?" (원문: "What are the top 5 products by revenue?") | Genie Space (정형 데이터 질의) |
| "신선식품 반품 정책은 무엇인가요?" (원문: "What is the return policy for perishable items?") | Vector Search (정책 문서 검색) |
| "제가 유기농 제품을 선호한다는 걸 기억해 주세요" (원문: "Remember that I prefer organic products") | 장기 메모리 (선호 저장) |
| "제 선호사항이 뭐였죠?" (원문: "What are my preferences?") (새 채팅에서) | 장기 메모리 (선호 회상) |

에이전트가 적절한 답변을 내놓으면 완료입니다!

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `databricks bundle validate`에서 오류 표시 | YAML의 오타 또는 미교체 플레이스홀더 | 오류 메시지를 읽어 보세요 — 대개 정확한 줄을 알려줍니다. 2~4단계 값이 모두 채워졌는지 확인하세요 |
| 앱이 **Crashed** 표시 | 대개 구성 문제 | 앱 클릭 > Logs 탭 > 오류 위치까지 스크롤 |
| `unhandled errors in a TaskGroup` | GENIE_SPACE_ID 또는 VECTOR_SEARCH_INDEX가 잘못됨 | Genie Space ID와 VS 인덱스 이름이 2단계 출력과 정확히 일치하는지 확인 |
| `permission denied for schema` | 서비스 프린시펄이 Lakebase에 접근 불가 | 7단계(Lakebase 권한 섹션)를 다시 실행 |
| `relation "ai_chatbot"."Chat" already exists` | 데이터베이스 테이블을 수동으로 생성함 | 삭제: `DROP SCHEMA IF EXISTS ai_chatbot CASCADE; DROP SCHEMA IF EXISTS drizzle CASCADE;` 실행 후 앱 재시작 |
| 에이전트가 툴을 사용하지 않음 | 구성에 잘못된 리소스 ID | `databricks.yml`의 Genie Space ID와 VS 인덱스가 2단계 출력과 일치하는지 확인 |
| Vector Search 결과 없음 | 인덱스 동기화가 끝나지 않음 | 2단계 이후 5~10분 대기. Catalog Explorer > 인덱스 찾기 > Sync 탭에서 동기화 상태 확인 |
| `Cannot deploy app... not in RUNNING state` | 소스 코드 배포 전에 앱을 먼저 시작해야 함 | `databricks apps start retail-grocery-ltm-memory` 실행 후 RUNNING 상태 대기 |
| 첫 시작 시 `UniqueViolation` | 테이블 생성 시 경쟁 조건(race condition) | 무시해도 안전 — 재시도 시 자동으로 처리됨 |

---

## 아키텍처

```
┌───────────────────── Databricks Workspace ─────────────────────────┐
│                                                                     │
│  ┌────────────────────────────┐                                    │
│  │ 01_quickstart_setup.py     │                                    │
│  │ (Data Setup - Step 2)      │                                    │
│  │                            │                                    │
│  │ Creates:                   │                                    │
│  │  • UC tables (6)           │                                    │
│  │  • policy_docs_chunked     │                                    │
│  │  • Vector Search index ────┼───┐                                │
│  │  • Genie Space ────────────┼───┼───┐                            │
│  │  • MLflow Experiment ──────┼───┼───┼───┐                        │
│  └────────────────────────────┘   │   │   │                        │
│                                   │   │   │                        │
│  ┌────────────────────────────────┼───┼───┼──────────────────────┐ │
│  │  Databricks App               │   │   │                      │ │
│  │  (deployed via bundle)        │   │   │                      │ │
│  │                               ▼   ▼   ▼                      │ │
│  │  ┌──────────────────────────────────────────────────────┐    │ │
│  │  │ agent_server (Python, port 8000)                     │    │ │
│  │  │  • LangGraph agent with MCP tools                    │    │ │
│  │  │  • Connects to VS index and Genie via MCP            │    │ │
│  │  │  • Logs traces to MLflow Experiment                  │    │ │
│  │  │  • Long-term memory (user prefs, tasks, history)     │    │ │
│  │  └──────────────────────────────────────────────────────┘    │ │
│  │  ┌──────────────────────────────────────────────────────┐    │ │
│  │  │ e2e-chatbot-app-next (Node.js, port 3000)           │    │ │
│  │  │  • Chat UI with persistent history                   │    │ │
│  │  │  • Auto-creates ai_chatbot tables (Drizzle)          │    │ │
│  │  └──────────────────────────────────────────────────────┘    │ │
│  └───────────────────────────────┬───────────────────────────────┘ │
│                                  │                                  │
│                                  ▼                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Lakebase (Managed PostgreSQL)                    │  │
│  │                                                              │  │
│  │  langgraph_memory      │  ai_chatbot      │  drizzle         │  │
│  │  (checkpoints, user    │  (Chat, Message,  │  (migration      │  │
│  │   prefs, tasks - auto) │   Vote - auto)    │   tracking)      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 정리: 직접 만드는 것 vs. 앱이 만드는 것

| 리소스 | 생성 주체 | 시점 | 직접 해야 할 일이 있나요? |
|---|---|---|---|
| 데이터 테이블 + Vector Search + Genie | 본인 | 2단계 (노트북 실행) | 예 — 노트북을 실행하세요 |
| MLflow Experiment | 본인 | 2단계 (노트북이 생성) | ID만 기록해 두세요 |
| Lakebase 인스턴스 | 본인 | 3단계 (UI에서 생성) | 예 — UI에서 생성하세요 |
| 배포된 앱 자체 | 본인 | 6단계 (배포 명령어) | 예 — 직접 배포하세요 |
| 에이전트 메모리 테이블 | 앱 | 첫 시작 시 자동 | 아니요 — 손대지 마세요! |
| 채팅 이력 테이블 | 앱 | 첫 시작 시 자동 | 아니요 — 손대지 마세요! |
| 데이터베이스 권한 | 필요 없음 | — | 앱이 자기 테이블을 직접 소유합니다 |
