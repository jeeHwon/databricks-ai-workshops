# Setup Guide: Deploy This Agent in Your Own Workspace

This guide walks you through cloning this codebase and deploying it as a Databricks App in your own workspace, with Lakebase-powered short-term memory and chat history sidebar.

---

## Prerequisites

- A Databricks workspace with Unity Catalog enabled
- Databricks CLI v0.295.0+ installed (`brew install databricks` or `pip install databricks-cli`)
- `uv` (Python package manager) installed
- Node.js 20+ with npm (for the frontend)
- Access to create Lakebase instances in the workspace

---

## Step 1: Clone the Repository

```bash
git clone <repo-url>
cd agent-openai-agents-sdk
```

---

## Step 2: Authenticate with Databricks CLI

```bash
databricks auth login --host https://<your-workspace>.cloud.databricks.com
```

This creates a profile (default: `DEFAULT`). Verify with:

```bash
databricks auth profiles
```

---

## Step 3: Run Quickstart

```bash
uv run quickstart --profile DEFAULT
```

This will:
- Create a `.env` file with your profile and an MLflow experiment
- Update `databricks.yml` with the experiment ID

---

## Step 4: Create a Lakebase Instance

In the Databricks UI or via CLI, create an **autoscaling** Lakebase project and branch. Note down:

- The **endpoint path**: `projects/<project-name>/branches/<branch-name>/endpoints/<endpoint-name>`
- The **branch path**: `projects/<project-name>/branches/<branch-name>`
- The **database path**: `projects/<project-name>/branches/<branch-name>/databases/<db-name>`
- The **PGHOST** hostname (e.g., `ep-xxxx-yyyy.database.<region>.cloud.databricks.com`)

---

## Step 5: Update `.env` for Local Development

Edit `.env` to add Lakebase configuration:

```env
DATABRICKS_CONFIG_PROFILE=DEFAULT
MLFLOW_EXPERIMENT_ID=<your-experiment-id>  # set by quickstart
CHAT_APP_PORT=3000
CHAT_PROXY_TIMEOUT_SECONDS=300
MLFLOW_TRACKING_URI="databricks"
MLFLOW_REGISTRY_URI="databricks-uc"

# Lakebase short-term memory (backend agent memory)
LAKEBASE_AUTOSCALING_ENDPOINT=projects/<your-project>/branches/<your-branch>/endpoints/<your-endpoint>
LAKEBASE_AGENT_MEMORY_SCHEMA=agent_openai_memory

# Frontend Lakebase connection (for chat history sidebar)
PGHOST=<your-lakebase-hostname>
PGDATABASE=databricks_postgres
PGPORT=5432
PGUSER=<your-email@company.com>
```

---

## Step 6: Update `databricks.yml` with Your Resources

Edit the `resources` section to point to your Lakebase instance and experiment:

```yaml
resources:
  apps:
    agent_openai_agents_sdk:
      name: "agent-<your-app-name>"   # Choose a unique app name
      description: "OpenAI Agents SDK agent application"
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
          - name: LAKEBASE_AUTOSCALING_ENDPOINT
            value_from: "postgres"
          - name: LAKEBASE_AGENT_MEMORY_SCHEMA
            value: "agent_openai_memory"
          - name: PGDATABASE
            value: "databricks_postgres"
          - name: PGPORT
            value: "5432"

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

Also update the workspace `host` in `targets`:

```yaml
targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<your-workspace>.cloud.databricks.com
  prod:
    mode: production
    workspace:
      host: https://<your-workspace>.cloud.databricks.com
    resources:
      apps:
        agent_openai_agents_sdk:
          name: agent-<your-app-name>
```

---

## Step 7: Customize the Agent (Optional)

Edit `agent_server/agent.py` to change the model, system prompt, or MCP servers:

```python
NAME = 'my-agent'
SYSTEM_PROMPT = 'You are a helpful assistant specialized in...'
MODEL = 'databricks-claude-sonnet-4-6'  # or your preferred model
MCP_SERVERS = [
    # Add your own Vector Search indexes, Genie spaces, etc.
    # ('Display Name', '/api/2.0/mcp/vector-search/catalog/schema/index'),
    # ('Genie Space Name', '/api/2.0/mcp/genie/<space-id>'),
]
```

To discover available tools in your workspace:

```bash
uv run discover-tools
```

---

## Step 8: Create Lakebase Tables (First-Time Setup)

Start the backend server locally. It will automatically create the `agent_openai_memory` schema and tables:

```bash
uv run start-server
```

If that fails with a permission error, create tables manually:

```python
import asyncio
from databricks_openai.agents.session import AsyncDatabricksSession

async def setup():
    session = AsyncDatabricksSession(
        session_id="__setup__",
        autoscaling_endpoint="projects/<your-project>/branches/<your-branch>/endpoints/<your-endpoint>",
        schema="agent_openai_memory",
    )
    await session._ensure_tables()
    print("Tables created!")

asyncio.run(setup())
```

Save this as `setup_tables.py` and run: `uv run python setup_tables.py`

---

## Step 9: Run the Frontend Drizzle Migration (For Chat History Sidebar)

```bash
cd e2e-chatbot-app-next
npm install
npx drizzle-kit push
cd ..
```

This creates the `ai_chatbot` schema tables (conversations, messages, etc.) used by the frontend sidebar.

---

## Step 10: Test Locally

```bash
uv run start-app
```

Open `http://localhost:8000`. Send messages and verify the agent remembers context across turns within the same conversation.

---

## Step 11: Deploy to Databricks Apps

```bash
# Validate configuration
databricks bundle validate --profile DEFAULT

# Deploy (uploads code, creates app + resources)
databricks bundle deploy --profile DEFAULT

# Start the app
databricks bundle run agent_openai_agents_sdk --profile DEFAULT
```

> **Note:** `bundle deploy` only uploads files. `bundle run` is required to actually start/restart the app.

---

## Step 12: Grant Permissions to the App's Service Principal

After first deploy, the app's service principal needs access to the Lakebase tables.

Get the SP identity:

```bash
databricks apps get <your-app-name> --output json --profile DEFAULT | jq -r '.service_principal_client_id'
```

Connect to your Lakebase instance (via `psql`, a Databricks notebook, or any PostgreSQL client) and run:

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
```

> **Important:** You must grant on SEQUENCES separately. Table grants alone are not enough. Without this, the service principal will get "permission denied for sequence" errors.

---

## Step 13: Verify the Deployed App

```bash
# Check app status
databricks apps get <your-app-name> --output json --profile DEFAULT | jq '{app_status, compute_status, url}'

# View logs
databricks apps logs <your-app-name> --follow --profile DEFAULT

# Test the endpoint
TOKEN=$(databricks auth token --profile DEFAULT | jq -r '.access_token')
APP_URL=$(databricks apps get <your-app-name> --output json --profile DEFAULT | jq -r '.url')

curl -X POST ${APP_URL}/invocations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input": [{"role": "user", "content": "Hello, remember my name is Alice"}], "stream": false}'
```

---

## Key Config Values to Replace

| Placeholder | Where to find it |
|---|---|
| `<your-workspace>` | Your Databricks workspace URL |
| `<your-experiment-id>` | Created by `uv run quickstart` (saved in `.env`) |
| `<your-project>` | Lakebase project name (from Lakebase UI) |
| `<your-branch>` | Lakebase branch name (usually `production`) |
| `<your-endpoint>` | Lakebase endpoint name (usually `primary`) |
| `<your-db>` | Lakebase database name (from Lakebase UI) |
| `<your-lakebase-hostname>` | PGHOST value (from Lakebase connection info) |
| `<your-email>` | Your Databricks login email (for PGUSER locally) |
| `<your-app-name>` | Name you choose for the Databricks App |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `relation agent_messages does not exist` | Run Step 8 to create tables |
| `permission denied for schema public` | Tables must be created in `agent_openai_memory` schema, not `public`. The code already uses `create_tables=False` to prevent this. Run Step 8 manually. |
| `permission denied for sequence` | Run the GRANT on sequences in Step 12 |
| App crashes after deploy | Check `databricks apps logs` — usually a missing env var or permission issue |
| Frontend shows no chat history after page refresh | Chat history sidebar requires the Lakebase PG vars to be available. Verify `PGDATABASE` and `PGPORT` are set in `app.yaml` and the postgres resource is bound. |
| `databricks bundle deploy` says "unknown field" | Your Databricks CLI version is too old — upgrade to v0.295.0+ |
| `An app with the same name already exists` | Either delete the existing app (`databricks apps delete <name>`) or bind it: `databricks bundle deployment bind agent_openai_agents_sdk <name> --auto-approve` |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Databricks App                         │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────────┐    │
│  │  Frontend (3000) │─────▶│   Backend (8000)     │    │
│  │  Next.js/React   │      │   FastAPI + MLflow   │    │
│  │  Chat UI +       │      │   OpenAI Agents SDK  │    │
│  │  History Sidebar │      │                      │    │
│  └────────┬─────────┘      └──────────┬───────────┘    │
│           │                            │                │
└───────────┼────────────────────────────┼────────────────┘
            │                            │
            ▼                            ▼
   ┌─────────────────┐        ┌─────────────────────┐
   │   Lakebase PG   │        │    Lakebase PG      │
   │  Schema:        │        │  Schema:            │
   │  ai_chatbot     │        │  agent_openai_memory│
   │  (UI history)   │        │  (agent memory)     │
   └─────────────────┘        └─────────────────────┘
```

- **Backend memory** (`agent_openai_memory`): Server-side session memory so the agent recalls conversation context across requests
- **Frontend history** (`ai_chatbot`): Persists chat conversations for the sidebar UI (shows past conversations after page refresh)
- Both schemas live in the same Lakebase instance but serve different purposes
