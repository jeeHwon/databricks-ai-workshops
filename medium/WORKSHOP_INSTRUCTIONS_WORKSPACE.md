# 워크숍: 메모리를 갖춘 AI 에이전트 배포하기 (워크스페이스 전용)

이 가이드는 L200 AI 에이전트 앱을 **Databricks 워크스페이스 안에서만** 배포하는 과정을 안내합니다 — 로컬 머신 설정이나 노트북에 터미널 툴을 설치할 필요가 없습니다.

끝까지 진행하면 문서를 검색하고 데이터를 질의할 수 있는 AI 에이전트 기반 챗 애플리케이션이 완성됩니다.

---

## 사전 준비 사항

시작하기 전에 Databricks 워크스페이스에 다음 기능이 활성화되어 있는지 확인하세요(불확실하면 워크스페이스 관리자에게 문의하세요):

- **Unity Catalog** — 데이터 테이블과 권한을 관리합니다
- **Databricks Apps** — 챗 애플리케이션을 호스팅합니다
- **Lakebase** — 관리형 PostgreSQL 데이터베이스(에이전트 메모리 + 채팅 기록에 사용)
- **Web Terminal** (웹 터미널) — 브라우저 내장 터미널 (Settings > Developer > Web Terminal)
- **실행 중인 SQL warehouse** — 데이터 테이블 생성에 필요 (Compute > SQL Warehouses)

---

## 1단계: 리포지토리를 워크스페이스로 가져오기

1. 왼쪽 사이드바에서 **Workspace** > **Repos**를 클릭하세요("Git Folders"로 표시될 수도 있습니다)
2. **Add** > **Git Folder**를 클릭하세요
3. 리포지토리 URL을 붙여넣으세요: `https://github.com/AnanyaDBJ/databricks-ai-workshops.git`
4. **Create Git Folder**를 클릭하세요

가져오기가 끝나면 코드는 다음과 같은 경로에 있습니다:
`/Workspace/Repos/<your-username>/databricks-ai-workshops/`

---

## 2단계: 데이터 설정

[`data/README.md`](../data/README.md#path-b-workspace-notebook)의 **Path B (워크스페이스 노트북)**를 완료하세요.

1. `data/workspace_setup_script/01_quickstart_setup.py`로 이동하세요
2. 드롭다운 위젯에서 **카탈로그**와 **스키마**를 선택하세요
3. **Run All**을 클릭하세요(약 10~15분 소요)

**노트북 출력에서 다음 4개 값을 복사하세요 — 5단계에서 필요합니다:**

| 복사할 값 | 예시 형태 | 사용 위치 |
|------|---------|---------|
| MLflow Experiment ID | `1234567890123456` | 5a단계 (`databricks.yml`) |
| Vector Search 인덱스 이름 | `my_catalog.my_schema.policy_docs_index` | 5b단계 (`agent.py`) |
| Genie Space ID | `01abcdef12345678` | 5b단계 (`agent.py`) |
| Catalog.Schema | `my_catalog.my_schema` | 5b단계 (`agent.py`) |

---

## 3단계: Lakebase 인스턴스 생성

Lakebase는 Databricks 안의 관리형 PostgreSQL 데이터베이스입니다. 앱은 에이전트 메모리와 채팅 기록에 Lakebase를 사용합니다.

### 인스턴스 생성

1. 왼쪽 사이드바에서 **Compute** > **Lakebase**로 이동하세요
2. **Create Project**를 클릭하고 > **Autoscaling**을 선택하세요
3. 기억하기 쉬운 이름을 지정하세요(예: `my-agent-workshop`)
4. Create를 클릭하세요 — `production` 브랜치가 자동으로 생성됩니다
5. "Ready" 상태가 될 때까지 약 1~2분 기다리세요

### 연결 정보 확인

1. 워크스페이스에서 `medium/scripts/lakebase_setup_script.ipynb` 노트북을 여세요
2. **Cell 1** (pip install)을 실행하세요 — 필요한 의존성을 설치합니다
3. **Cell 4**에서 `<project name>`을 방금 만든 프로젝트 이름으로 바꾸세요
4. **Cell 4**를 실행하고 출력을 기록해 두세요:
   - **브랜치 경로**: `projects/my-agent-workshop/branches/production`
   - **데이터베이스 경로**: `name` 값 전체 (예: `projects/my-agent-workshop/branches/production/databases/databricks-postgres`)

> **중요:** Cell 1(pip install), Cell 2(선택 사항 — 브랜치 목록), Cell 4(데이터베이스 목록)만 실행하세요. Cell 3, 5, 6은 실행하지 마세요 — 앱이 필요한 데이터베이스 테이블을 자동으로 생성합니다.

---

## 3단계 (대안): Lakebase Provisioned 인스턴스 사용

워크스페이스에서 Autoscaling 대신 **Provisioned** Lakebase 인스턴스를 사용한다면:

1. **Compute** > **Lakebase** > **Create** > **Provisioned**로 이동하세요
2. 이름을 지정하고(예: `my-agent-workshop`), 용량 `CU_1`을 선택한 뒤 Create를 클릭하세요
3. "Running" 상태가 될 때까지 기다리세요

필요한 값은 **인스턴스 이름**(예: `my-agent-workshop`) 하나뿐입니다.

5a단계에 도달하면 다음과 같이 다른 설정을 사용하세요:

```yaml
# Replace the postgres resource with:
        - name: 'database'
          database:
            database_name: databricks_postgres
            instance_name: 'my-agent-workshop'
            permission: CAN_CONNECT_AND_CREATE

# Replace LAKEBASE_AUTOSCALING_ENDPOINT env var with:
          - name: LAKEBASE_INSTANCE_NAME
            value: "my-agent-workshop"
```

---

## 4단계: MLflow Experiment ID 확인

데이터 설정 노트북(2단계)이 이미 MLflow 실험을 생성했습니다. **Experiment ID**를 확인하세요:

1. 왼쪽 사이드바에서 **Experiments**를 클릭하세요
2. 노트북이 생성한 실험을 찾으세요
3. 클릭하면 브라우저 URL 표시줄에 긴 숫자가 보입니다 — 이것이 **Experiment ID**입니다

**찾을 수 없거나 새로 만들어야 한다면:**
1. **Experiments** > **Create Experiment** (우측 상단)로 이동하세요
2. 이름을 지정하세요(예: `my-agent-experiment`)
3. Create를 클릭하고 URL에서 Experiment ID를 복사하세요

---

## 5단계: 설정 파일 수정

워크스페이스 파일 브라우저에서 `medium/` 폴더로 이동하세요.

### 5a. `databricks.yml` 수정

다음 값들을 찾아 바꾸세요:

| 찾을 값 | 바꿀 값 | 출처 |
|---|---|---|
| `"my-agent-app"` | 고유한 이름 (예: `"agent-workshop-jsmith"`) | 직접 정하기 |
| `"<your-experiment-id>"` | Experiment ID (예: `"1234567890123456"`) | 4단계 |
| `"projects/<your-project>/branches/<branch-name>"` | 브랜치 경로 | 3단계 |
| `"projects/<your-project>/branches/production/databases/databricks-postgres"` | 데이터베이스 경로 | 3단계 |
| `https://<your-workspace>.cloud.databricks.com` | 워크스페이스 URL | 브라우저 주소창 |

**수정 후 주요 섹션은 다음과 같은 모습이어야 합니다:**

```yaml
resources:
  apps:
    agent_openai_agents_sdk:
      name: "agent-workshop-jsmith"                    # ← 고유한 앱 이름
      # ... (env vars 섹션은 수정하지 마세요) ...
      resources:
        - name: 'experiment'
          experiment:
            experiment_id: "1234567890123456"           # ← 4단계에서 확인한 값
            permission: 'CAN_MANAGE'
        - name: 'postgres'
          postgres:
            branch: "projects/my-agent-workshop/branches/production"         # ← 3단계에서 확인한 값
            database: "projects/my-agent-workshop/branches/production/databases/databricks-postgres"  # ← 3단계에서 확인한 값
            permission: 'CAN_CONNECT_AND_CREATE'

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://my-workspace.cloud.databricks.com   # ← 워크스페이스 URL
```

### 5b. `agent_server/agent.py` 수정

`MCP_SERVERS` 설정을 찾아 플레이스홀더를 바꾸세요:

```python
MCP_SERVERS = [
    ('Policy Document Search', '/api/2.0/mcp/vector-search/<catalog>/<schema>/<index-name>'),
    ('Data Query Assistant', '/api/2.0/mcp/genie/<genie-space-id>'),
]
```

| 플레이스홀더 | 바꿀 값 | 예시 |
|---|---|---|
| `<catalog>/<schema>/<index-name>` | Vector Search 인덱스(점 → 슬래시) | `my_catalog/my_schema/policy_docs_index` |
| `<genie-space-id>` | Genie Space ID | `01abcdef12345678` |

---

## 6단계: 앱 배포

**Web Terminal**을 여세요: 하단 패널의 `>_` 터미널 아이콘을 클릭하거나 Settings > Developer > Web Terminal로 이동하세요.

1. `medium` 폴더로 이동하세요:
   ```bash
   cd /Workspace/Repos/<your-username>/databricks-ai-workshops/medium
   ```

2. 설정을 검증하세요:
   ```bash
   databricks bundle validate
   ```
   오류가 출력되면 5a단계로 돌아가 플레이스홀더를 수정하세요.

3. 앱을 배포하세요:
   ```bash
   databricks bundle deploy
   ```

4. terraform 바이너리를 제거하세요(워크스페이스 배포에 필수):
   ```bash
   rm -rf .databricks/bundle/dev/bin .databricks/bundle/dev/terraform/.terraform
   ```
   > **이유:** 배포 단계에서 대용량 terraform 바이너리(약 50MB+)가 `.databricks/`에 다운로드되는데, 이 디렉터리는 앱 소스 코드로도 사용됩니다. 앱 소스 스냅샷에는 50MB 파일 크기 제한이 있어, 제거하지 않으면 실패합니다.

5. 앱을 시작하세요:
   ```bash
   databricks bundle run agent_openai_agents_sdk
   ```
   최초 시작은 **3~5분** 걸립니다(의존성 설치와 데이터베이스 테이블 생성).

---

## 7단계: 권한 부여

앱은 **서비스 프린시펄**(SP)로 실행되며 데이터 리소스에 접근할 권한이 필요합니다. Lakebase 권한은 번들이 자동으로 처리하지만, Unity Catalog와 Genie Space는 수동으로 권한을 부여해야 합니다.

### 서비스 프린시펄 ID 확인

1. **Compute** > **Apps**로 이동해 앱 이름을 클릭하세요
2. 앱 상세 페이지에 표시된 **Service Principal**을 찾아 클라이언트 ID를 복사하세요

또는 Web Terminal에서:

```bash
SP_CLIENT_ID=$(databricks apps get <your-app-name> --output json | jq -r '.service_principal_client_id')
echo "SP Client ID: $SP_CLIENT_ID"
```

### Unity Catalog 접근 권한 부여

**SQL Editor**(또는 SQL warehouse에 연결된 노트북)에서 실행하세요:

```sql
GRANT USE CATALOG ON CATALOG <your-catalog> TO `<SP_CLIENT_ID>`;
GRANT USE SCHEMA ON SCHEMA <your-catalog>.<your-schema> TO `<SP_CLIENT_ID>`;
GRANT SELECT ON SCHEMA <your-catalog>.<your-schema> TO `<SP_CLIENT_ID>`;
```

`<your-catalog>.<your-schema>`는 2단계의 값으로, `<SP_CLIENT_ID>`는 위에서 확인한 ID로 바꾸세요.

### Genie Space 접근 권한 부여

1. 왼쪽 사이드바에서 **Genie Space**(2단계에서 생성한 것)로 이동하세요
2. 우측 상단의 **Share** (공유)를 클릭하세요
3. 서비스 프린시펄을 검색하세요(클라이언트 ID 또는 이름으로)
4. **Can Run** 권한을 부여하세요

---

## 8단계: 배포 검증

### 앱 상태 확인

1. 왼쪽 사이드바에서 **Compute** > **Apps**로 이동하세요
2. 배포한 앱을 찾으세요 — 상태 표시가 녹색으로 바뀌며 **Running**으로 표시되어야 합니다
3. **Starting**으로 표시되면 몇 분 더 기다리세요
4. **Crashed**로 표시되면 아래 트러블슈팅을 참고하세요

### 로그 확인 (이상이 있어 보이는 경우)

1. 앱 이름을 클릭하고 > **Logs** 탭으로 이동하세요
2. 모든 것이 정상 동작 중임을 뜻하는 다음 줄을 찾으세요:
   - `"Uvicorn running on http://0.0.0.0:8000"` — 백엔드 준비 완료
   - `"Server is running on http://localhost:3000"` — 프런트엔드 준비 완료

### 앱 테스트

1. 앱 페이지 상단에 표시된 **앱 URL**을 클릭해 챗 인터페이스를 여세요
2. 다음 테스트 프롬프트를 시도해 보세요:

| 프롬프트 | 테스트 대상 |
|--------|---------------|
| "환불 정책은 무엇인가요?" (원문: "What is the refund policy?") | Vector Search (문서 검색) |
| "우리 고객은 모두 몇 명인가요?" (원문: "How many customers do we have?") | Genie Space (데이터 질의) |
| "제 이름은 Alice예요, 기억해 주세요" (원문: "Remember my name is Alice") | 에이전트 메모리 (저장) |
| "제 이름이 뭐였죠?" (원문: "What's my name?") (새로고침 후) | 에이전트 메모리 (회상) |

에이전트가 적절한 답변을 돌려준다면 완료입니다!

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| `relation "ai_chatbot"."Chat" already exists` | 배포 전에 데이터베이스 테이블을 수동으로 생성함 | 스키마를 삭제하세요: `DROP SCHEMA IF EXISTS ai_chatbot CASCADE; DROP SCHEMA IF EXISTS drizzle CASCADE;` 실행 후 앱 재시작 |
| `permission denied for schema` | 직접 만든 테이블에 앱이 접근할 수 없음 | 스키마를 삭제하고 앱이 다시 생성하게 하세요(앱이 소유권을 갖게 됩니다) |
| 앱이 **Crashed** 상태로 표시됨 | 대부분 `databricks.yml`의 오타 | 앱 클릭 > Logs 탭 > 오류 위치로 스크롤 |
| `Lakebase unavailable` | 설정의 데이터베이스 경로가 잘못됨 | `databricks.yml`의 `branch:`와 `database:`가 Cell 4 출력과 일치하는지 확인 |
| 에이전트가 툴을 사용하지 않음 | `agent.py`의 URL이 잘못됨 | catalog/schema/index가 2단계와 일치하는지 확인하세요. URL에서 점은 슬래시로 바꿔야 합니다 |
| `Failed to snapshot source code... larger than maximum allowed file size` | terraform 바이너리를 제거하지 않음 | `rm -rf .databricks/bundle/dev/bin .databricks/bundle/dev/terraform/.terraform` 실행 후 `databricks bundle run` 재실행 |
| `An app with the same name already exists` | 이름 충돌 | 다른 이름을 선택하거나 삭제하세요: `databricks apps delete <name>` |
| `databricks bundle validate`에서 오류 표시 | YAML에 바꾸지 않은 플레이스홀더가 있음 | 오류 메시지를 읽으세요 — 정확한 줄을 알려 줍니다 |
| Vector Search 결과가 없음 | 인덱스 동기화가 아직 끝나지 않음 | 2단계 후 5~10분 기다리세요 |
| `PERMISSION_DENIED` 또는 에이전트 툴이 빈 결과 반환 | SP에 UC 또는 Genie 접근 권한이 없음 | 7단계(권한 부여)를 완료하세요 |

---

## 아키텍처

![L200 Architecture](./L200_Architecture.png)

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
│  │  │  • Connects to VS index and Genie via MCP            │    │ │
│  │  │  • Logs traces to MLflow Experiment                  │    │ │
│  │  │  • Auto-creates agent_openai_memory tables           │    │ │
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
│  │  agent_openai_memory    │  ai_chatbot      │  drizzle        │  │
│  │  (agent conversation    │  (Chat, User,    │  (migration     │  │
│  │   memory - auto)        │   Message, Vote  │   journal -     │  │
│  │                         │   - auto)        │   auto)         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 요약: 직접 만드는 것 vs. 앱이 만드는 것

| 리소스 | 생성 주체 | 시점 | 직접 해야 하나요? |
|---|---|---|---|
| 데이터 테이블 + Vector Search + Genie Space | 사용자 | 2단계(노트북 실행) | 예 — 노트북을 실행하세요 |
| MLflow Experiment | 사용자 | 2단계(노트북이 생성) | ID만 기록해 두세요 |
| Lakebase 데이터베이스 인스턴스 | 사용자 | 3단계(UI에서 Create 클릭) | 예 — UI에서 생성하세요 |
| 배포되는 앱 자체 | 사용자 | 6단계(배포 명령) | 예 — 배포하세요 |
| 채팅 기록 테이블 | 앱 | 최초 시작 시 자동 | 아니요 — 손대지 마세요! |
| 에이전트 메모리 테이블 | 앱 | 최초 시작 시 자동 | 아니요 — 손대지 마세요! |
| 데이터베이스 권한 | 불필요 | — | 아니요 — 앱이 자기 테이블을 직접 소유합니다 |
