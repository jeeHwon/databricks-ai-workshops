# 워크숍: Databricks에서 메모리를 갖춘 AI 에이전트 구축하기

OpenAI Agents SDK를 사용해 Genie, Vector Search, 영구 메모리를 갖춘 대화형 AI 에이전트를 구축하고 배포합니다.

## 사전 준비 사항

| 툴 | 설치 방법 |
|------|---------|
| Databricks CLI v0.295+ | `brew tap databricks/tap && brew install databricks` |
| uv | [설치 가이드](https://docs.astral.sh/uv/getting-started/installation/) |
| Node.js 20+ | [nodejs.org](https://nodejs.org) |

워크스페이스에 다음 기능이 필요합니다: Unity Catalog, Databricks Apps, Lakebase, Vector Search, Foundation Model API.

### 데이터 설정 (먼저 필수로 진행)

계속하기 전에 데이터 설정을 완료하세요:
**→ [`data/README.md`](../data/README.md) — Path A (로컬 CLI)**

완료하면 다음 값들이 저장되어 있어야 합니다:
- **Catalog.Schema** (예: `my_catalog.my_schema`)
- **Vector Search 인덱스** (예: `my_catalog.my_schema.policy_docs_index`)
- **Genie Space ID** (예: `01abcdef12345678`)
- **MLflow Experiment ID** (예: `1234567890123456`)

---

## 1단계: 리포지토리 클론

```bash
git clone https://github.com/AnanyaDBJ/databricks-ai-workshops.git
cd databricks-ai-workshops/medium
```

---

## 2단계: Quickstart 실행

```bash
uv run quickstart --profile DEFAULT
```

이 대화형 마법사는 다음을 처리합니다:
- Databricks CLI 인증 (OAuth 로그인)
- MLflow 실험 생성
- `.env` 파일 구성

프롬프트를 따라 진행하세요. 데이터 설정에서 이미 실험을 생성했다면 다음과 같이 전달하세요:

```bash
uv run quickstart --profile DEFAULT --experiment-id <ID>
```

---

## 3단계: Lakebase 인스턴스 생성

Databricks UI에서 **Autoscaling** (오토스케일링) Lakebase 프로젝트를 생성하세요:

1. **Compute** > **Lakebase**로 이동하세요
2. **Create Project**를 클릭하고 > **Autoscaling**을 선택하세요
3. 이름을 지정하고(예: `my-agent-workshop`) Create를 클릭하세요
4. "Ready" 상태가 될 때까지 기다리세요

헬퍼 노트북으로 연결 정보를 확인하세요:

1. 워크스페이스에서 `medium/scripts/lakebase_setup_script.ipynb`를 여세요
2. Cell 3에서 `<project name>`을 프로젝트 이름으로 바꾸세요
3. Cell 3을 실행하세요 — 출력에서 **브랜치 경로**와 **데이터베이스 경로**를 기록해 두세요

> **중요:** Cell 1(선택 사항)과 Cell 3만 실행하세요. Cell 2, 4, 5는 실행하지 마세요 — 앱이 필요한 테이블을 자동으로 생성합니다.

---

## 4단계: 에이전트 구성

### 4a. `.env`에 Lakebase 값 추가

`.env` 파일에 다음을 추가하세요:

```env
LAKEBASE_AUTOSCALING_ENDPOINT=projects/<your-project>/branches/<your-branch>/endpoints/<your-endpoint>
LAKEBASE_AGENT_MEMORY_SCHEMA=agent_openai_memory
PGHOST=<your-lakebase-hostname>
PGDATABASE=databricks_postgres
PGPORT=5432
PGUSER=<your-email@company.com>
```

### 4b. `agent_server/agent.py`에 툴 URL 입력

`MCP_SERVERS` 섹션을 찾아 데이터 설정에서 얻은 값으로 업데이트하세요:

```python
MCP_SERVERS = [
    ('Policy Document Search', '/api/2.0/mcp/vector-search/<catalog>/<schema>/<index-name>'),
    ('Data Query Assistant', '/api/2.0/mcp/genie/<genie-space-id>'),
]
```

> **참고:** 인덱스 이름이 `my_catalog.my_schema.policy_docs_index`(점 포함)라면, URL에서는 점을 슬래시로 바꾸세요: `my_catalog/my_schema/policy_docs_index`

추가 툴을 찾아보려면: `uv run discover-tools`

---

## 5단계: 로컬에서 실행

```bash
uv run start-app
```

백엔드(8000 포트)와 프런트엔드(3000 포트)가 시작됩니다. 최초 실행 시 모든 데이터베이스 테이블을 자동으로 생성합니다.

`http://localhost:3000`을 열고 다음을 시도해 보세요:
- "신선식품 반품 정책은 무엇인가요?" (원문: "What is the return policy for perishable items?") (Vector Search)
- "매출 기준 상위 5개 제품은 무엇인가요?" (원문: "What are the top 5 products by revenue?") (Genie)
- "제 이름은 Alice예요, 기억해 주세요" (원문: "Remember my name is Alice") 입력 후 새로고침하고 "제 이름이 뭐였죠?" (원문: "What's my name?") 질문 (메모리)

---

## 6단계: Databricks Apps에 배포

### 6a. `databricks.yml` 업데이트

`resources` 섹션을 실제 값으로 수정하세요:

```yaml
resources:
  apps:
    agent_openai_agents_sdk:
      name: "agent-<your-app-name>"
      # ...
      resources:
        - name: 'experiment'
          experiment:
            experiment_id: "<your-experiment-id>"
            permission: 'CAN_MANAGE'
        - name: 'postgres'
          postgres:
            branch: "projects/<your-project>/branches/<your-branch>"
            database: "projects/<your-project>/branches/<your-branch>/databases/<your-db>"
            permission: 'CAN_CONNECT_AND_CREATE'
```

`targets`의 워크스페이스 `host`도 업데이트하세요:

```yaml
targets:
  dev:
    workspace:
      host: https://<your-workspace>.cloud.databricks.com
```

### 6b. 검증 및 배포

```bash
# 설정 검증
databricks bundle validate --profile DEFAULT

# 배포 (코드 업로드, 앱 + 리소스 생성)
databricks bundle deploy --profile DEFAULT

# 앱 시작
databricks bundle run agent_openai_agents_sdk --profile DEFAULT
```

> `bundle deploy`는 파일을 업로드하고 리소스를 생성합니다. `bundle run`은 앱을 시작합니다.

---

## 7단계: 권한 부여

최초 배포 후, 앱의 서비스 프린시펄이 로컬 테스트 중에 생성한 테이블에 접근할 수 있도록 권한이 필요합니다.

SP ID를 확인하세요:

```bash
databricks apps get <your-app-name> --output json --profile DEFAULT | jq -r '.service_principal_client_id'
```

Lakebase 인스턴스에 연결한 뒤(psql, 노트북 또는 아무 PostgreSQL 클라이언트) 다음을 실행하세요:

```sql
-- Backend agent memory schema
GRANT USAGE ON SCHEMA agent_openai_memory TO PUBLIC;
GRANT ALL ON ALL TABLES IN SCHEMA agent_openai_memory TO PUBLIC;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA agent_openai_memory TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA agent_openai_memory GRANT ALL ON TABLES TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA agent_openai_memory GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO PUBLIC;

-- Frontend chat history schema
GRANT USAGE ON SCHEMA ai_chatbot TO PUBLIC;
GRANT ALL ON ALL TABLES IN SCHEMA ai_chatbot TO PUBLIC;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA ai_chatbot TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA ai_chatbot GRANT ALL ON TABLES TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA ai_chatbot GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO PUBLIC;

-- Drizzle migration tracking
GRANT USAGE ON SCHEMA drizzle TO PUBLIC;
GRANT ALL ON ALL TABLES IN SCHEMA drizzle TO PUBLIC;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA drizzle TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA drizzle GRANT ALL ON TABLES TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA drizzle GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO PUBLIC;
```

> **대안:** 권한 부여 없이 진행하고 싶다면, 배포 전에 모든 스키마를 삭제하고 앱의 SP가 새로 생성하게 하세요.

---

## 8단계: 배포된 앱 검증

```bash
# 앱 상태 확인
databricks apps get <your-app-name> --output json --profile DEFAULT | jq '{app_status, compute_status, url}'

# 로그 보기
databricks apps logs <your-app-name> --follow --profile DEFAULT

# 엔드포인트 테스트
TOKEN=$(databricks auth token --profile DEFAULT | jq -r '.access_token')
APP_URL=$(databricks apps get <your-app-name> --output json --profile DEFAULT | jq -r '.url')

curl -X POST ${APP_URL}/invocations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input": [{"role": "user", "content": "Hello, what tools do you have?"}]}'
```

브라우저에서 앱 URL을 열면 챗 UI를 사용할 수 있습니다.

---

## 트러블슈팅

| 증상 | 해결 방법 |
|-------|-----|
| `relation "ai_chatbot"."Chat" already exists` | 스키마를 삭제하세요: `DROP SCHEMA IF EXISTS ai_chatbot CASCADE; DROP SCHEMA IF EXISTS drizzle CASCADE;` 실행 후 재시작 |
| `relation agent_messages does not exist` | 앱을 재시작하세요 — `start_server.py`가 자동으로 생성합니다 |
| `permission denied for schema` | 7단계의 GRANT 문을 실행하세요 |
| `permission denied for sequence` | 시퀀스는 별도 권한 부여가 필요합니다 — 7단계 SQL 전체를 실행하세요 |
| 배포 후 앱 크래시 | `databricks apps logs`를 확인하세요 — 보통 환경 변수 누락 또는 권한 문제입니다 |
| `databricks bundle deploy`에서 "unknown field" 표시 | CLI를 v0.295.0+로 업그레이드하세요 |
| `An app with the same name already exists` | 삭제: `databricks apps delete <name>` 또는 바인딩: `databricks bundle deployment bind agent_openai_agents_sdk <name> --auto-approve` |
| MCP 툴이 응답하지 않음 | `agent.py`의 URL이 데이터 설정의 리소스와 일치하는지 확인하세요. 형식: `/api/2.0/mcp/vector-search/catalog/schema/index` |
| Vector Search 결과가 없음 | 인덱스 동기화가 안 끝났을 수 있습니다 — 생성 후 5~10분 기다리세요 |
| 로컬 앱이 시작되지 않음 | `lsof -ti :8000`을 확인하세요 — 잔여 프로세스를 종료하세요 |
