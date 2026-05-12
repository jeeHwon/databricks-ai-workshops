# Setup Guide: Deploy Entirely Within Databricks Workspace (No CLI)

This guide covers deploying the agent app **entirely from within a Databricks workspace** — no local machine, no Databricks CLI, no `uv` or `npm` needed.

---

## Prerequisites

- A Databricks workspace with:
  - Unity Catalog enabled
  - Databricks Apps enabled
  - Lakebase enabled
- Workspace admin or permissions to create apps, experiments, and Lakebase instances

---

## Step 1: Import the Code into Your Workspace

**Option A — Git Folder (recommended if repo is in GitHub/GitLab):**

1. Go to **Workspace** → **Repos** (or Git Folders)
2. Click **Add** → **Git Folder**
3. Enter the repository URL
4. Click **Create Git Folder**

The code will be available at: `/Workspace/Repos/<your-username>/<repo-name>/agent-openai-agents-sdk`

**Option B — Upload as workspace files:**

1. Download the `agent-openai-agents-sdk` folder as a zip
2. Go to **Workspace** → **Users** → your home folder
3. Click **Import** → upload the zip
4. Unzip into a folder (e.g., `/Workspace/Users/<you>/agent-openai-agents-sdk`)

---

## Step 2: Create a Lakebase Instance

1. Go to the **SQL** section → **Lakebase** in the left nav (or search "Lakebase")
2. Click **Create Project** (for autoscaling) or **Create Instance** (for provisioned)
3. For **autoscaling** (recommended):
   - Set project name (e.g., `my-agent-project`)
   - Create a branch (e.g., `production`)
   - Note the **endpoint path** shown in the connection info:
     ```
     projects/<project-name>/branches/<branch-name>/endpoints/<endpoint-name>
     ```
   - Note the **PGHOST** hostname (e.g., `ep-xxxx-yyyy.database.<region>.cloud.databricks.com`)

---

## Step 3: Create an MLflow Experiment

1. Go to **Machine Learning** → **Experiments**
2. Click **Create Experiment**
3. Name it (e.g., `agent-with-memory`)
4. Set artifact location to your workspace path
5. **Note the Experiment ID** from the URL (e.g., `1234567890123456`)

---

## Step 4: Create the Database Tables via Notebook

Create a new **Python notebook** and connect it to any cluster (or use serverless). Run these cells:

### Cell 1: Install the package

```python
%pip install "databricks-openai[memory]>=0.13.0"
dbutils.library.restartPython()
```

### Cell 2: Create backend agent memory tables

```python
import asyncio
from databricks_openai.agents.session import AsyncDatabricksSession

# Replace with YOUR Lakebase endpoint path from Step 2
LAKEBASE_ENDPOINT = "projects/<your-project>/branches/<your-branch>/endpoints/<your-endpoint>"

async def create_agent_tables():
    session = AsyncDatabricksSession(
        session_id="__setup__",
        autoscaling_endpoint=LAKEBASE_ENDPOINT,
        schema="agent_openai_memory",
    )
    await session._ensure_tables()
    print("✓ Backend agent memory tables created (schema: agent_openai_memory)")

asyncio.run(create_agent_tables())
```

### Cell 3: Create frontend chat history tables

```python
# Connect to Lakebase and create the ai_chatbot schema + tables
import psycopg

# Replace with YOUR Lakebase connection details from Step 2
PGHOST = "<your-lakebase-hostname>"  # e.g., ep-xxxx-yyyy.database.us-east-2.cloud.databricks.com
PGUSER = "<your-email>"  # Your Databricks login email
PGDATABASE = "databricks_postgres"
PGPORT = "5432"

# Get OAuth token for authentication
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
token = w.tokens.create(comment="lakebase-setup", lifetime_seconds=3600).token_value

conn = psycopg.connect(
    host=PGHOST,
    port=PGPORT,
    dbname=PGDATABASE,
    user=PGUSER,
    password=token,
    sslmode="require",
)
conn.autocommit = True
cur = conn.cursor()

# Create schema
cur.execute("CREATE SCHEMA IF NOT EXISTS ai_chatbot;")

# Create User table
cur.execute('''
CREATE TABLE IF NOT EXISTS "ai_chatbot"."User" (
    "id" text PRIMARY KEY NOT NULL,
    "email" varchar(64) NOT NULL
);
''')

# Create Chat table
cur.execute('''
CREATE TABLE IF NOT EXISTS "ai_chatbot"."Chat" (
    "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
    "createdAt" timestamp NOT NULL,
    "title" text NOT NULL,
    "userId" text NOT NULL,
    "visibility" varchar DEFAULT 'private' NOT NULL,
    "lastContext" jsonb
);
''')

# Create Message table
cur.execute('''
CREATE TABLE IF NOT EXISTS "ai_chatbot"."Message" (
    "id" uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
    "chatId" uuid NOT NULL,
    "role" varchar NOT NULL,
    "parts" json NOT NULL,
    "attachments" json NOT NULL,
    "createdAt" timestamp NOT NULL,
    "traceId" text
);
''')

# Create Vote table
cur.execute('''
CREATE TABLE IF NOT EXISTS "ai_chatbot"."Vote" (
    "chatId" uuid NOT NULL,
    "messageId" uuid NOT NULL,
    "isUpvoted" boolean NOT NULL,
    CONSTRAINT "Vote_chatId_messageId_pk" PRIMARY KEY("chatId","messageId")
);
''')

# Add foreign keys (ignore if they already exist)
fks = [
    'ALTER TABLE "ai_chatbot"."Message" ADD CONSTRAINT "Message_chatId_Chat_id_fk" FOREIGN KEY ("chatId") REFERENCES "ai_chatbot"."Chat"("id") ON DELETE no action ON UPDATE no action;',
    'ALTER TABLE "ai_chatbot"."Vote" ADD CONSTRAINT "Vote_chatId_Chat_id_fk" FOREIGN KEY ("chatId") REFERENCES "ai_chatbot"."Chat"("id") ON DELETE no action ON UPDATE no action;',
    'ALTER TABLE "ai_chatbot"."Vote" ADD CONSTRAINT "Vote_messageId_Message_id_fk" FOREIGN KEY ("messageId") REFERENCES "ai_chatbot"."Message"("id") ON DELETE no action ON UPDATE no action;',
]
for fk in fks:
    try:
        cur.execute(fk)
    except Exception as e:
        if "already exists" in str(e):
            pass
        else:
            print(f"Warning: {e}")

print("✓ Frontend chat history tables created (schema: ai_chatbot)")
cur.close()
conn.close()
```

### Cell 4: Grant permissions to PUBLIC (for the app service principal)

```python
import psycopg

# Reuse same connection details
conn = psycopg.connect(
    host=PGHOST,
    port=PGPORT,
    dbname=PGDATABASE,
    user=PGUSER,
    password=token,
    sslmode="require",
)
conn.autocommit = True
cur = conn.cursor()

grants = [
    # Backend agent memory schema
    "GRANT USAGE ON SCHEMA agent_openai_memory TO PUBLIC;",
    "GRANT ALL ON ALL TABLES IN SCHEMA agent_openai_memory TO PUBLIC;",
    "GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA agent_openai_memory TO PUBLIC;",
    "ALTER DEFAULT PRIVILEGES IN SCHEMA agent_openai_memory GRANT ALL ON TABLES TO PUBLIC;",
    "ALTER DEFAULT PRIVILEGES IN SCHEMA agent_openai_memory GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO PUBLIC;",
    # Frontend chat history schema
    "GRANT USAGE ON SCHEMA ai_chatbot TO PUBLIC;",
    "GRANT ALL ON ALL TABLES IN SCHEMA ai_chatbot TO PUBLIC;",
    "GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA ai_chatbot TO PUBLIC;",
    "ALTER DEFAULT PRIVILEGES IN SCHEMA ai_chatbot GRANT ALL ON TABLES TO PUBLIC;",
    "ALTER DEFAULT PRIVILEGES IN SCHEMA ai_chatbot GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO PUBLIC;",
]

for grant in grants:
    try:
        cur.execute(grant)
    except Exception as e:
        print(f"Warning on '{grant[:50]}...': {e}")

print("✓ Permissions granted to PUBLIC for both schemas")
cur.close()
conn.close()
```

---

## Step 5: Edit Configuration Files in Workspace

Navigate to your imported code in the workspace file browser and edit these files:

### Edit `databricks.yml`

Update these values:

| Field | Replace with |
|---|---|
| `name:` under apps | Your app name (e.g., `"agent-my-assistant"`) |
| `experiment_id:` | Your experiment ID from Step 3 |
| `branch:` under postgres | `"projects/<your-project>/branches/<your-branch>"` |
| `database:` under postgres | `"projects/<your-project>/branches/<your-branch>/databases/<your-db>"` |
| `host:` under targets | `https://<your-workspace>.cloud.databricks.com` |

Example:
```yaml
resources:
  apps:
    agent_openai_agents_sdk:
      name: "agent-my-assistant"
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
            experiment_id: "<YOUR_EXPERIMENT_ID>"
            permission: 'CAN_MANAGE'
        - name: 'postgres'
          postgres:
            branch: "projects/<YOUR_PROJECT>/branches/<YOUR_BRANCH>"
            database: "projects/<YOUR_PROJECT>/branches/<YOUR_BRANCH>/databases/<YOUR_DB>"
            permission: 'CAN_CONNECT_AND_CREATE'

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<YOUR_WORKSPACE>.cloud.databricks.com
  prod:
    mode: production
    workspace:
      host: https://<YOUR_WORKSPACE>.cloud.databricks.com
    resources:
      apps:
        agent_openai_agents_sdk:
          name: agent-my-assistant
```

### Edit `agent_server/agent.py` (Optional — Customize Your Agent)

Update the agent configuration:

```python
NAME = 'my-agent'
SYSTEM_PROMPT = 'You are a helpful assistant.'
MODEL = 'databricks-claude-sonnet-4-6'  # or your preferred model
MCP_SERVERS = [
    # Add your own tools here. Find tool URLs in the workspace:
    # Vector Search: /api/2.0/mcp/vector-search/<catalog>/<schema>/<index>
    # Genie Space: /api/2.0/mcp/genie/<space-id>
]
```

---

## Step 6: Create the App via Databricks Apps UI

1. Go to **Compute** → **Apps** (or search "Apps" in the workspace)
2. Click **Create App**
3. Fill in:
   - **Name**: Same as in your `databricks.yml` (e.g., `agent-my-assistant`)
   - **Source code path**: Point to your workspace folder containing the code
4. Configure **Environment Variables** (add each one):

   | Variable | Value |
   |---|---|
   | `MLFLOW_TRACKING_URI` | `databricks` |
   | `MLFLOW_REGISTRY_URI` | `databricks-uc` |
   | `API_PROXY` | `http://localhost:8000/invocations` |
   | `CHAT_APP_PORT` | `3000` |
   | `CHAT_PROXY_TIMEOUT_SECONDS` | `300` |
   | `LAKEBASE_AGENT_MEMORY_SCHEMA` | `agent_openai_memory` |
   | `PGDATABASE` | `databricks_postgres` |
   | `PGPORT` | `5432` |

5. Configure **Resources**:
   - Add an **Experiment** resource:
     - Name: `experiment`
     - Experiment ID: your experiment ID
     - Permission: `CAN_MANAGE`
   - Add a **Postgres** resource:
     - Name: `postgres`
     - Branch: `projects/<your-project>/branches/<your-branch>`
     - Database: `projects/<your-project>/branches/<your-branch>/databases/<your-db>`
     - Permission: `CAN_CONNECT_AND_CREATE`

6. Set the **Command**: `uv run start-app`

7. For env vars that reference resources, set them as `value_from`:
   - `MLFLOW_EXPERIMENT_ID` → value from resource `experiment`
   - `LAKEBASE_AUTOSCALING_ENDPOINT` → value from resource `postgres`

8. Click **Create** / **Deploy**

---

## Step 7: Deploy via Notebook (Alternative to UI)

If you prefer programmatic deployment, run this in a notebook:

```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Create the app
app = w.apps.create(
    name="agent-my-assistant",
    description="OpenAI Agents SDK agent with memory",
)
print(f"App created: {app.name}")
print(f"Service Principal: {app.service_principal_client_id}")

# Deploy from workspace path
deployment = w.apps.deploy(
    app_name="agent-my-assistant",
    source_code_path="/Workspace/Repos/<your-username>/<repo-name>/agent-openai-agents-sdk",
    mode="AUTO_SYNC",  # or "SNAPSHOT"
)
print(f"Deployment started: {deployment.deployment_id}")
```

> **Note:** The Apps UI is easier for configuring resources and env vars. The SDK approach requires additional API calls to configure resources.

---

## Step 8: Verify the Deployment

### Check app status in the UI:
1. Go to **Compute** → **Apps**
2. Find your app
3. Check that status shows **Running**

### Check logs:
1. Click on your app
2. Go to the **Logs** tab
3. Look for: `"Uvicorn running on http://0.0.0.0:8000"` and `"Backend server is running on http://localhost:3000"`

### Test via notebook:

```python
from databricks.sdk import WorkspaceClient
import requests

w = WorkspaceClient()
token = w.config.token

app_url = "https://<your-app-name>-<workspace-id>.aws.databricksapps.com"

response = requests.post(
    f"{app_url}/invocations",
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    },
    json={
        "input": [{"role": "user", "content": "Hello! Remember my name is Alice."}],
        "stream": False,
    },
)
print(response.json())
```

---

## Summary: CLI vs Workspace Comparison

| Step | CLI Approach | Workspace Approach |
|---|---|---|
| Get code | `git clone` locally | Git Folder or upload |
| Authenticate | `databricks auth login` | Already authenticated |
| Create experiment | `uv run quickstart` | UI → Experiments → Create |
| Create Lakebase | CLI or UI | UI → Lakebase → Create |
| Create tables | `uv run start-server` (local) | Notebook (Cell 2-3 above) |
| Grant permissions | `psql` or notebook | Notebook (Cell 4 above) |
| Edit config | Local editor | Workspace file editor |
| Deploy app | `databricks bundle deploy` + `run` | Apps UI → Create App |
| View logs | `databricks apps logs --follow` | Apps UI → Logs tab |
| Test endpoint | `curl` | Notebook with `requests` |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Notebook can't connect to Lakebase | Ensure your cluster has network access. Use `%pip install psycopg[binary]` if `psycopg` fails. |
| `_ensure_tables()` fails | Check that your endpoint path is correct and your user has CREATE permissions on the Lakebase instance. |
| App shows "crashed" | Go to Logs tab. Common causes: missing env var, postgres permission error, or incorrect resource binding. |
| `permission denied for sequence` | Re-run the GRANT statements in Cell 4 (Step 4). Sequences need separate grants from tables. |
| Chat history sidebar is empty | Tables in `ai_chatbot` schema must exist (Cell 3) and have PUBLIC grants (Cell 4). |
| App can't resolve model | Ensure the model name in `agent.py` (e.g., `databricks-claude-sonnet-4-6`) is available in your workspace. Check Foundation Model APIs. |
| Source code not found | Verify the source code path in the App config matches where you imported the code. |

---

## Architecture

```
┌────────────────── Databricks Workspace ──────────────────┐
│                                                           │
│  ┌─────────────┐  ┌───────────┐  ┌──────────────────┐   │
│  │ Git Folder  │  │ Notebook  │  │   Databricks App │   │
│  │ (source)    │  │ (setup)   │  │   (runtime)      │   │
│  └─────────────┘  └───────────┘  └────────┬─────────┘   │
│                                            │              │
│                                            ▼              │
│                              ┌──────────────────────┐    │
│                              │     Lakebase PG      │    │
│                              │  ┌────────────────┐  │    │
│                              │  │agent_openai_   │  │    │
│                              │  │memory (backend)│  │    │
│                              │  ├────────────────┤  │    │
│                              │  │ai_chatbot      │  │    │
│                              │  │(frontend)      │  │    │
│                              │  └────────────────┘  │    │
│                              └──────────────────────┘    │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

Everything stays within the Databricks workspace — no external tools or local development environment required.
