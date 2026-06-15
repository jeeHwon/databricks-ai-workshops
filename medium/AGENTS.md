# 에이전트 개발 가이드

## 필수 첫 번째 작업

**사용자에게 대화형으로 질문하세요:**

1. **앱 배포 대상:**
   > "배포할 기존 Databricks 앱이 있나요, 아니면 새로 만들어야 하나요? 기존 앱이 있다면 앱 이름이 무엇인가요?"

   *참고: 사용자가 따로 지정하지 않는 한, 새 앱은 `agent-*` 접두사를 사용해야 합니다(예: `agent-data-analyst`).*

2. **사용자가 메모리, 대화 기록 또는 지속성을 언급하는 경우:**
   > "메모리 기능을 위해 기존 Lakebase 인스턴스가 있나요? 있다면 인스턴스 이름이 무엇인가요?"

**그다음 quickstart로 환경을 설정하세요:**

1. `.claude/skills/quickstart/SKILL.md`에 있는 **quickstart 스킬을 읽으세요** — 사용 가능한 모든 CLI 플래그, 스크립트가 구성하는 항목, 대체 방법이 담겨 있습니다.
2. **`.env` 파일이 있는지 확인하세요.** 있다면 환경이 이미 구성된 것입니다 — 파일을 읽어 `DATABRICKS_CONFIG_PROFILE`을 확인하고 인증 검증 단계로 건너뛰세요. `.env`가 없다면 quickstart를 실행하세요:
   ```bash
   uv run quickstart --profile <profile-name>
   ```
3. `databricks auth profiles`를 실행해 프로필이 구성되어 있고 유효한지 확인하세요.

**중요: 모든 `databricks` CLI 명령어에는 `.env`의 프로필을 포함해야 합니다.** `--profile`을 사용하거나 환경 변수를 설정하세요:

```bash
databricks <command> --profile <profile>
# or
DATABRICKS_CONFIG_PROFILE=<profile> databricks <command>
```

> **왜 중요한가요:** 프로필이 없으면 CLI가 잘못된 워크스페이스를 대상으로 동작해, 실험·앱·기타 리소스에 대해 "not found" 오류가 발생할 수 있습니다.

## 사용자 목표 파악

**사용자가 무엇을 만들려는지 질문으로 파악하세요:**

1. **에이전트의 목적은 무엇인가요?** (예: 데이터 분석 어시스턴트, 고객 지원, 코드 헬퍼)
2. **어떤 데이터나 툴에 접근해야 하나요?**
   - 데이터베이스/테이블 (Unity Catalog)
   - RAG용 문서 (Vector Search)
   - 자연어 데이터 질의 (Genie Spaces)
   - 외부 API 또는 서비스
3. **연결하고 싶은 특정 Databricks 리소스가 있나요?**

`uv run discover-tools`로 워크스페이스에서 사용 가능한 리소스를 보여 주고, 유스케이스에 맞는 리소스 선택을 도와주세요. **툴 연결과 권한 부여 방법은 `add-tools` 스킬을 참고하세요.**

## 배포 오류 처리

**`databricks bundle deploy`가 "An app with the same name already exists" 오류로 실패하는 경우:**

사용자에게 질문하세요: "같은 이름의 기존 앱이 있습니다. 이 번들에 바인딩해서 관리할까요, 아니면 삭제하고 새로 만들까요?"

- **바인딩을 원하면**: 바인딩 절차는 **deploy** 스킬을 참고하세요
- **삭제를 원하면**: `databricks apps delete <app-name>`을 실행한 뒤 다시 배포하세요

## Supervisor API (에이전트 루프 오프로딩)

**Supervisor API**를 사용하면 Databricks가 툴 선택과 에이전트 루프를 서버 측에서 실행합니다. 호스티드 툴(Genie space, UC 함수, Knowledge Assistant, UC 커넥션 MCP 서버, Databricks App 엔드포인트)을 선언하고 `responses.create()`를 호출하면 — 나머지는 Databricks가 처리합니다.

**사용 시점:** 사용자가 에이전트 루프를 직접 관리하지 않고 Genie space, UC 함수 또는 기타 Databricks 호스티드 툴을 연결하고 싶을 때.

**제약 사항:**
- 툴은 앱의 서비스 프린시펄로 실행됩니다(사용자 토큰 포워딩 없음) — `databricks.yml`에서 권한을 부여하세요
- 같은 요청에서 호스티드 툴과 클라이언트 측 함수 툴을 혼합할 수 없습니다
- 툴을 전달할 때는 추론 파라미터(`temperature`, `top_p` 등)가 지원되지 않습니다
- 같은 요청에서 `stream`과 `background`를 모두 `true`로 설정할 수 없습니다
- 백그라운드 모드의 최대 실행 시간은 30분입니다

**스킬:**
- 호스티드 툴로 Supervisor API를 설정하려면 **supervisor-api**를 사용하세요
- HTTP 타임아웃 제한을 초과할 수 있는 작업(복잡한 멀티 툴 워크플로우, 대규모 데이터 분석)에는 **supervisor-api-background-mode**를 사용하세요

## 에이전트 평가

사용자가 에이전트 평가(품질, 메트릭, 스코어러, 데이터셋 또는 트레이싱)에 대해 물으면, https://github.com/mlflow/skills 의 **MLflow Skills** 설치를 제안하세요. MLflow 네이티브 API를 활용한 평가 워크플로우에 대한 전문적인 가이드를 제공합니다.

**관련 스킬:**
- **agent-evaluation** — 엔드 투 엔드 평가: 데이터셋 생성, 스코어러 선택, 실행, 결과 분석
- **instrumenting-with-mlflow-tracing** — 디버깅과 옵저버빌리티를 위한 자동 트레이싱 설정
- **analyze-mlflow-trace** — 스팬 데이터와 평가 결과(assessment)를 검토해 문제 파악

**설치 명령어:**
```bash
npx skills add mlflow/skills
```

설치 후 스킬은 슬래시 명령어(예: `/agent-evaluation`)로 사용할 수 있습니다. 이 템플릿에는 내장 `evaluate_agent.py` 스크립트도 포함되어 있습니다 — 로컬 서버를 시작한 뒤 `uv run agent-evaluate`로 실행하세요.

---

## 사용 가능한 스킬

**어떤 작업이든 실행하기 전에 `.claude/skills/`의 해당 스킬 파일을 읽으세요** — 검증된 명령어, 패턴, 트러블슈팅 절차가 담겨 있습니다.

| 작업 | 스킬 | 경로 |
|------|-------|------|
| 설정, 인증, 최초 사용 | **quickstart** | `.claude/skills/quickstart/SKILL.md` |
| 툴/리소스 찾기 | **discover-tools** | `.claude/skills/discover-tools/SKILL.md` |
| 툴 리소스 생성 | **create-tools** | `.claude/skills/create-tools/SKILL.md` |
| Databricks에 배포 | **deploy** | `.claude/skills/deploy/SKILL.md` |
| 툴 추가 & 권한 부여 | **add-tools** | `.claude/skills/add-tools/SKILL.md` |
| 로컬 실행/테스트 | **run-locally** | `.claude/skills/run-locally/SKILL.md` |
| 에이전트 코드 수정 | **modify-agent** | `.claude/skills/modify-agent/SKILL.md` |
| 에이전트 루프를 Databricks로 오프로딩 | **supervisor-api** | `.claude/skills/supervisor-api/SKILL.md` |
| 장시간 백그라운드 작업 | **supervisor-api-background-mode** | `.claude/skills/supervisor-api-background-mode/SKILL.md` |

**참고:** 모든 에이전트 스킬은 `.claude/skills/` 디렉터리에 있습니다.

---

## 빠른 명령어

| 작업 | 명령어 |
|------|---------|
| 설정 | `uv run quickstart` |
| 툴 찾기 | `uv run discover-tools` |
| 로컬 실행 | `uv run start-app` |
| 배포 | `databricks bundle deploy && databricks bundle run agent_openai_agents_sdk` |
| 로그 보기 | `databricks apps logs <app-name> --follow` |

---

## 주요 파일

| 파일 | 용도 |
|------|---------|
| `agent_server/agent.py` | 에이전트 로직, 모델, 지시문, MCP 서버 |
| `agent_server/start_server.py` | FastAPI 서버 + MLflow 설정 |
| `agent_server/evaluate_agent.py` | MLflow 스코어러를 활용한 에이전트 평가 |
| `databricks.yml` | 번들 설정 & 리소스 권한 |
| `.github/workflows/deploy.yml` | 이 앱을 배포하는 GitHub Actions 워크플로우(`.scripts/source/.github/workflows/`에서 동기화) |
| `scripts/quickstart.py` | 원커맨드 설정 스크립트 |
| `scripts/discover_tools.py` | 사용 가능한 워크스페이스 리소스 탐색 |

---

## 에이전트 프레임워크 기능

> **⚠️ 중요:** 에이전트에 툴을 추가할 때는 반드시 `databricks.yml`에서도 권한을 부여해야 합니다. 필수 절차와 예시는 **add-tools** 스킬을 참고하세요.

**툴 유형:**
1. **Unity Catalog Function 툴** - 내장 거버넌스와 함께 UC에서 관리되는 SQL UDF
2. **에이전트 코드 툴** - REST API 및 저지연 작업을 위해 에이전트 코드에 직접 정의
3. **MCP 툴** - Model Context Protocol 기반의 상호 운용 가능한 툴(Databricks 관리형, 외부 또는 자체 호스팅)

**내장 툴:**
- **system.ai.python_exec** - 에이전트 질의 중 Python 코드를 동적으로 실행(코드 인터프리터)

**자주 쓰는 패턴:**
- **정형 데이터 검색** - SQL 테이블/데이터베이스 질의
- **비정형 데이터 검색** - Vector Search를 통한 문서 검색과 RAG
- **코드 인터프리터** - system.ai.python_exec를 통한 분석용 Python 실행
- **외부 연결** - HTTP 커넥션으로 Slack 등 서비스 연동

참고 문서: https://docs.databricks.com/aws/en/generative-ai/agent-framework/
