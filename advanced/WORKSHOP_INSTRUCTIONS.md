# 워크숍: Databricks에서 리테일 식료품 AI 에이전트 구축하기

Genie, Vector Search, 장기 메모리를 활용해 대화형 AI 에이전트를 구축하고 배포합니다.

## 사전 준비 사항

| 툴 | 설치 방법 |
|------|---------|
| Databricks CLI | `brew tap databricks/tap && brew install databricks` |
| uv | [설치 가이드](https://docs.astral.sh/uv/getting-started/installation/) |
| Node.js 20+ | [nodejs.org](https://nodejs.org) |
| jq | `brew install jq` |

워크스페이스에는 다음 기능이 필요합니다: 서버리스 컴퓨트, Foundation Model API (Claude), Unity Catalog, Vector Search, Lakebase.

### 데이터 설정 (가장 먼저 필수로 진행)

계속 진행하기 전에 데이터 설정을 완료하세요:
**→ [`data/README.md`](../data/README.md) — Path A (로컬 CLI)**

완료하면 다음 값들이 준비되어 있어야 합니다:
- **Catalog.Schema** (예: `my_catalog.retail_agent`)
- **Vector Search Index** (예: `my_catalog.retail_agent.policy_docs_index`)
- **Genie Space ID** (예: `01ef...abcd`)

---

## 1단계: 리포지토리 클론

```bash
git clone https://github.com/AnanyaDBJ/databricks-ai-workshops.git
cd databricks-ai-workshops/advanced
```

---

## 2단계: Quickstart 실행

```bash
uv run quickstart
```

이 대화형 마법사는 다음 작업을 처리합니다:
- Databricks CLI 인증 (OAuth 로그인)
- MLflow 실험(experiment) 생성
- Lakebase 오토스케일링 인스턴스 생성 (프로젝트 + 브랜치)
- `.env` 파일 구성 (PGHOST, PGUSER, 실험 ID 등)

화면의 안내를 따라 진행하세요. 완료되면 `.env` 파일에 Lakebase와 MLflow 설정이 채워져 있습니다.

---

## 3단계: `.env`에 리소스 ID 업데이트

데이터 설정에서 얻은 값을 `.env`에 추가하세요:

```bash
GENIE_SPACE_ID=<your-genie-space-id>
VECTOR_SEARCH_INDEX=<your-catalog>.<your-schema>.policy_docs_index
```

이제 `.env`에 필요한 모든 값(quickstart + 데이터 설정)이 들어 있어야 합니다.

---

## 4단계: 로컬 실행

```bash
uv run start-app
```

백엔드가 `http://localhost:8000`에서, 채팅 UI가 `http://localhost:3000`에서 시작됩니다.

`http://localhost:3000`을 열고 다음 프롬프트를 시험해 보세요:

- "매출 기준 상위 5개 제품은 무엇인가요?" (원문: "What are the top 5 products by revenue?") (Genie — 정형 데이터)
- "신선식품 반품 정책은 무엇인가요?" (원문: "What is the return policy for perishable items?") (Vector Search — 정책 문서)
- "제가 유기농 제품을 선호한다는 걸 기억해 주세요" (원문: "Remember that I prefer organic products") 입력 후, 새 채팅에서 "제 선호사항이 뭐였죠?" (원문: "What are my preferences?") (메모리)

Databricks UI의 **Experiments** (실험) > 본인 실험에서 트레이스를 확인하세요.

---

## 5단계: Databricks Apps로 배포

### 5a. `databricks.yml` 업데이트

`advanced/databricks.yml`의 `resources` 섹션을 실제 값으로 수정하세요:

```yaml
      resources:
        - name: "experiment"
          experiment:
            experiment_id: "<EXPERIMENT-ID>"
            permission: "CAN_MANAGE"
        - name: "retail_grocery_genie"
          genie_space:
            name: "Retail Data"
            space_id: "<GENIE-SPACE-ID>"
            permission: "CAN_RUN"
        - name: "postgres"
          postgres:
            branch: "projects/<PROJECT-NAME>/branches/<BRANCH-NAME>"
            database: "projects/<PROJECT-NAME>/branches/<BRANCH-NAME>/databases/databricks-postgres"
            permission: "CAN_CONNECT_AND_CREATE"
        - name: "policy_docs_index"
          uc_securable:
            securable_full_name: "<CATALOG>.<SCHEMA>.policy_docs_index"
            securable_type: "TABLE"
            permission: "SELECT"
```

Lakebase 환경 변수도 함께 업데이트하세요:
```yaml
          - name: LAKEBASE_AUTOSCALING_PROJECT
            value: "<PROJECT-NAME>"
          - name: LAKEBASE_AUTOSCALING_BRANCH
            value: "<BRANCH-NAME>"
```

데이터베이스 리소스 경로를 찾으려면:
```bash
databricks api get /api/2.0/postgres/projects/<PROJECT-NAME>/branches/<BRANCH-NAME>/databases \
  | jq -r '.databases[0].name'
```

### 5b. 검증 및 배포

```bash
# Validate the bundle config
databricks bundle validate -t dev

# Deploy the bundle (creates the app resource)
databricks bundle deploy -t dev

# Start the app (must be RUNNING before deploying source code)
databricks apps start retail-grocery-ltm-memory

# Wait for app to be running
databricks apps get retail-grocery-ltm-memory --output json | jq -r '.status.state'
# Repeat until state is "RUNNING"

# Deploy source code to the running app
DATABRICKS_USERNAME=$(databricks current-user me | jq -r .userName)
databricks apps deploy retail-grocery-ltm-memory \
  --source-code-path /Workspace/Users/$DATABRICKS_USERNAME/.bundle/retail_grocery_ltm_memory/dev/files
```

---

## 6단계: 앱 서비스 프린시펄에 권한 부여

```bash
# Get the app's service principal
SP_CLIENT_ID=$(databricks apps get retail-grocery-ltm-memory --output json | jq -r '.service_principal_client_id')
echo "App SP Client ID: $SP_CLIENT_ID"
```

서비스 프린시펄에 다음 리소스 접근 권한을 부여하세요:

1. **Lakebase:**
```bash
uv run python scripts/grant_lakebase_permissions.py "$SP_CLIENT_ID" \
  --memory-type langgraph-short-term --project <PROJECT-NAME> --branch <BRANCH-NAME>
uv run python scripts/grant_lakebase_permissions.py "$SP_CLIENT_ID" \
  --memory-type langgraph-long-term --project <PROJECT-NAME> --branch <BRANCH-NAME>
```

2. **Genie Space:** UI에서 — **Genie** > 본인 스페이스 > **Share** (공유) > 서비스 프린시펄을 **Can Run** 권한으로 추가

3. **Unity Catalog 테이블:** SQL Editor에서:
```sql
GRANT USE CATALOG ON CATALOG <CATALOG> TO `<SP_CLIENT_ID>`;
GRANT USE SCHEMA ON SCHEMA <CATALOG>.<SCHEMA> TO `<SP_CLIENT_ID>`;
GRANT SELECT ON SCHEMA <CATALOG>.<SCHEMA> TO `<SP_CLIENT_ID>`;
```

---

## 7단계: 배포된 앱 확인

```bash
# Get the URL
APP_URL=$(databricks apps get retail-grocery-ltm-memory --output json | jq -r '.url')
echo "App URL: $APP_URL"

# Test the API
TOKEN=$(databricks auth token | jq -r .access_token)
curl -X POST "${APP_URL}/invocations" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input": [{"role": "user", "content": "What stores do you have?"}]}'
```

브라우저에서 앱 URL을 열어 채팅 UI를 사용해 보세요.

---

## 트러블슈팅

| 증상 | 해결 방법 |
|-------|-----|
| quickstart에서 `JSONDecodeError` | 인증 만료 — `databricks auth login --profile DEFAULT` 실행 |
| `unhandled errors in a TaskGroup` | `.env`의 GENIE_SPACE_ID 또는 VECTOR_SEARCH_INDEX가 비어 있거나 잘못됨 |
| 첫 실행 시 `UniqueViolation` | 무시해도 안전 — 재시도 시 자동으로 처리됨 |
| 앱이 `STOPPED` 상태에서 멈춤 | `databricks apps start retail-grocery-ltm-memory` 실행 |
| Lakebase 권한 거부(permission denied) | 6단계를 다시 실행(배포된 앱) 또는 CLI 인증이 유효한지 확인(로컬) |
| Vector Search 결과가 비어 있음 | 인덱스 동기화가 완료될 때까지 대기 — Catalog Explorer에서 상태 확인 |
| 로컬 앱이 시작되지 않음 | `lsof -ti :8000` 확인 — 남아 있는 프로세스 종료 |
| 배포된 앱 로그 확인 | `databricks apps get-logs retail-grocery-ltm-memory` 또는 UI의 **Apps** > **Logs** |
