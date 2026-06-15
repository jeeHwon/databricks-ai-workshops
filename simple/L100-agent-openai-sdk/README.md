# FreshMart Agent — 시작하기

OpenAI Agents SDK로 만든 FreshMart 식료품 체인용 프로덕션급 AI 에이전트입니다. 데이터 질문에는 Genie로, 매장 정책 조회에는 Vector Search로 답합니다.

## 사전 준비 사항

시작하기 전에 다음을 준비하세요:

1. **Databricks 워크스페이스** — `01_quickstart_setup.py`로 생성한 리소스 포함 (Genie Space, Vector Search 인덱스, MLflow Experiment)
2. **uv** — Python 패키지 매니저 ([설치](https://docs.astral.sh/uv/getting-started/installation/))
3. **Node.js 20+** — 채팅 UI 실행용 ([nvm으로 설치](https://github.com/nvm-sh/nvm))
4. **Databricks CLI** — 워크스페이스 인증 완료 상태 ([설치](https://docs.databricks.com/dev-tools/cli/install.html))

## 빠른 시작

```bash
# 1. Navigate to this folder
cd simple/L100-agent-openai-sdk

# 2. Run the setup wizard (handles auth, MLflow, .env)
uv run quickstart

# 3. Start the agent + chat UI
uv run start-app
```

**http://localhost:3000** 을 열어 에이전트와 대화하세요.

## 동작 방식

```
User (Chat UI on :3000)
  │
  ▼
Agent Server (FastAPI on :8000)
  │
  ├── Genie Space ──► Natural language queries over retail data
  │                   (products, transactions, customers, stores)
  │
  └── Vector Search ──► Policy document lookup
                        (returns, delivery, loyalty, privacy, etc.)
```

이 에이전트는 **OpenAI Agents SDK**와 MCP(Model Context Protocol) 서버를 사용해 Databricks 툴에 연결합니다. 모든 상호작용은 옵저버빌리티를 위해 **MLflow**에 트레이싱됩니다.

## 주요 파일

| 파일 | 역할 |
|------|-------------|
| `agent_server/agent.py` | 에이전트 로직 — 모델, 시스템 프롬프트, MCP 툴 |
| `agent_server/start_server.py` | MLflow 트레이싱이 포함된 FastAPI 서버 |
| `databricks.yml` | 배포 설정 (앱 이름, 리소스, 권한) |
| `.env` | 로컬 환경 변수 (quickstart가 생성) |

## 에이전트 커스터마이징

### 모델 변경

`agent_server/agent.py`를 수정하세요:
```python
MODEL = 'workshop-ai-endpoint'  # Change to your AI Gateway endpoint
```

### 시스템 프롬프트 변경

`agent_server/agent.py`의 `SYSTEM_PROMPT` 변수를 수정하세요.

### 새 툴 추가

Claude Code의 `/add-tools` 스킬을 사용하거나, 직접 다음을 수행하세요:
1. `agent_server/agent.py`의 `MCP_SERVERS` 리스트에 MCP 서버 항목 추가
2. `databricks.yml`의 `resources` 섹션에서 권한 부여
3. 재배포

## Databricks에 배포하기

```bash
# 1. Verify everything works locally
uv run preflight

# 2. Deploy (uploads code, creates/updates app)
databricks bundle deploy

# 3. Start the app (required after deploy!)
databricks bundle run agent_openai_agents_sdk
```

배포 출력에 표시되는 URL에서 앱을 사용할 수 있습니다.

## 에이전트 평가하기

상위 디렉터리(`simple/`)의 **`02_agent_evaluation.ipynb`** 노트북을 사용하세요. 배포된 에이전트를 MLflow 스코어러로 평가합니다:

- **Completeness** — 에이전트가 질문에 충분히 답했는가?
- **RelevanceToQuery** — 응답이 질문과 관련 있는가?
- **Safety** — 응답이 안전 가이드라인을 따르는가?
- **ToolCallEfficiency** — 툴을 효율적으로 사용했는가?

에이전트 배포 후 Databricks 워크스페이스에서 실행하세요.

## Claude Code 사용하기

Claude Code가 설치되어 있다면, 다음 스킬이 일반적인 작업을 안내해 줍니다:

| 하려는 작업 | 스킬 |
|---------------------|-------|
| 처음 설정하기 | `/quickstart` |
| 에이전트를 로컬에서 실행 | `/run-locally` |
| 사용 가능한 툴 확인 | `/discover-tools` |
| 새 툴 추가 (Genie, Vector Search 등) | `/add-tools` |
| 새 Databricks 리소스 생성 | `/create-tools` |
| 모델 또는 시스템 프롬프트 변경 | `/modify-agent` |
| Databricks에 배포 | `/deploy` |

## 트러블슈팅

| 문제 | 해결 방법 |
|---------|----------|
| `uv run quickstart`가 인증 단계에서 실패 | `databricks auth login`을 직접 실행한 뒤 재시도 |
| 8000 포트가 이미 사용 중 | `lsof -ti :8000 \| xargs kill -9` |
| 3000 포트가 이미 사용 중 | `lsof -ti :3000 \| xargs kill -9` |
| MCP 서버 연결 오류 | Genie Space와 Vector Search 인덱스가 존재하는지 확인 (먼저 `01_quickstart_setup.py` 실행) |
| 배포 시 "App already exists" | `uv run quickstart --app-name <existing-app>`으로 바인딩한 뒤 배포 |
| 배포 후 권한 오류 | `databricks.yml`에 리소스가 올바른 권한과 함께 나열되어 있는지 확인 |
