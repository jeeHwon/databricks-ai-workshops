# 데이터 설정

> **모든 워크숍 레벨의 첫 단계입니다.** 어떤 워크숍(Simple, Medium, Advanced)을 시작하든 먼저 이 설정을 완료하세요.

모든 워크숍 레벨이 공통으로 사용하는 데이터셋을 생성합니다: 리테일 데이터 테이블, 청킹된 정책 문서, Vector Search 인덱스, Genie Space, MLflow 실험(experiment).

---

## 경로 선택

| 경로 | 적합한 경우 | 시간 |
|------|----------|------|
| **[Path A: 로컬 CLI](#path-a-로컬-cli)** | 로컬 개발, advanced 워크숍을 로컬에서 실행 | 약 15분 |
| **[Path B: 워크스페이스 노트북](#path-b-워크스페이스-노트북)** | 모든 작업을 Databricks 안에서, 로컬 툴 불필요 | 약 15분 |

두 경로의 결과는 완전히 동일합니다. 하나만 선택하세요.

---

## Path A: 로컬 CLI

로컬 머신에서 이 스크립트들을 실행합니다. 스크립트는 CLI를 통해 Databricks 워크스페이스에 연결합니다.

### 사전 준비 사항

| 툴 | 설치 방법 |
|------|---------|
| Databricks CLI | `brew tap databricks/tap && brew install databricks` |
| Python 3.9+ | [python.org](https://www.python.org/downloads/) |
| jq | `brew install jq` |

추가로 다음이 필요합니다:
- **실행 중인 SQL warehouse** (Databricks의 Compute > SQL Warehouses)
- **Unity Catalog** 접근 권한 (테이블 생성 권한)

### 0단계: 리포지토리 클론

```bash
git clone https://github.com/AnanyaDBJ/databricks-ai-workshops.git
cd databricks-ai-workshops
```

### 1단계: 인증

```bash
databricks auth login --profile DEFAULT
```

브라우저 안내에 따라 진행하세요. 정상 동작 여부를 확인합니다:

```bash
databricks current-user me --profile DEFAULT
```

### 2단계: warehouse ID 확인

```bash
databricks warehouses list --profile DEFAULT --output json | jq -r '.[] | "\(.id)  \(.name)  \(.state)"'
```

`RUNNING` 상태인 warehouse를 골라 ID를 복사하세요.

### 3단계: 카탈로그와 스키마 생성

Databricks SQL Editor에서 실행하거나, CLI로 실행하세요:

```bash
databricks api post /api/2.0/sql/statements \
  --profile DEFAULT \
  --json '{
    "warehouse_id": "<WAREHOUSE-ID>",
    "statement": "CREATE CATALOG IF NOT EXISTS <CATALOG>; CREATE SCHEMA IF NOT EXISTS <CATALOG>.<SCHEMA>;"
  }'
```

`<CATALOG>`과 `<SCHEMA>`를 원하는 이름으로 바꾸세요 (예: `my_catalog`, `retail_agent`).

### 4단계: 리테일 데이터 테이블 생성

리포지토리 루트에서:

```bash
python data/local_cli_setup_script/execute_sql.py \
  --profile DEFAULT \
  --warehouse-id <WAREHOUSE-ID> \
  --catalog <CATALOG> \
  --schema <SCHEMA>
```

6개 테이블이 생성됩니다: `customers`, `products`, `stores`, `transactions`, `transaction_items`, `payment_history`.

### 5단계: 정책 문서 청크 생성

```bash
python data/local_cli_setup_script/execute_chunking.py \
  --profile DEFAULT \
  --warehouse-id <WAREHOUSE-ID> \
  --catalog <CATALOG> \
  --schema <SCHEMA>
```

7개의 정책 문서를 청킹하여 Vector Search용 `policy_docs_chunked` 테이블에 저장합니다.

### 6단계: Vector Search 인덱스 + Genie Space 생성

```bash
python data/local_cli_setup_script/create_resources.py \
  --profile DEFAULT \
  --warehouse-id <WAREHOUSE-ID> \
  --catalog <CATALOG> \
  --schema <SCHEMA>
```

이 작업은 5~10분이 걸립니다 (Vector Search 엔드포인트 프로비저닝). 완료되면 다음을 출력합니다:

```
============================================================
SUMMARY
============================================================
  Vector Search Endpoint: <endpoint-name>
  Vector Search Index:    <CATALOG>.<SCHEMA>.policy_docs_index
  Genie Space ID:         01ef...abcd

Add these to your advanced/.env file:
  VECTOR_SEARCH_INDEX=<CATALOG>.<SCHEMA>.policy_docs_index
  GENIE_SPACE_ID=01ef...abcd
```

**이 값들을 저장해 두세요** — 워크숍 레벨의 구성 단계에서 필요합니다.

### 완료!

이제 준비가 끝났습니다. 본인의 워크숍 레벨로 이동하세요:

| 레벨 | 다음 단계 |
|-------|-----------|
| Simple (L100) | [`simple/LAB_GUIDE.md`](../simple/LAB_GUIDE.md) |
| Medium (L200) — 로컬 | [`medium/WORKSHOP_INSTRUCTIONS.md`](../medium/WORKSHOP_INSTRUCTIONS.md) |
| Medium (L200) — 워크스페이스 | [`medium/WORKSHOP_INSTRUCTIONS_WORKSPACE.md`](../medium/WORKSHOP_INSTRUCTIONS_WORKSPACE.md) |
| Advanced (L300) — 로컬 | [`advanced/WORKSHOP_INSTRUCTIONS.md`](../advanced/WORKSHOP_INSTRUCTIONS.md) |
| Advanced (L300) — 워크스페이스 | [`advanced/WORKSHOP_INSTRUCTIONS_WORKSPACE.md`](../advanced/WORKSHOP_INSTRUCTIONS_WORKSPACE.md) |

---

## Path B: 워크스페이스 노트북

모든 작업을 Databricks 안에서 실행합니다 — 로컬 툴이 필요 없습니다.

### 사전 준비 사항

- **Unity Catalog**, **Vector Search**, **Foundation Model API**가 활성화된 Databricks 워크스페이스
- **실행 중인 SQL warehouse** (Compute > SQL Warehouses)

### 0단계: 리포지토리를 워크스페이스로 가져오기

1. 왼쪽 사이드바에서 **Workspace** > **Repos** 클릭 (또는 "Git Folders")
2. **Add** > **Git Folder** 클릭
3. URL 붙여넣기: `https://github.com/AnanyaDBJ/databricks-ai-workshops.git`
4. **Create Git Folder** 클릭

### 1단계: 노트북 열기

워크스페이스 파일 브라우저에서 `data/workspace_setup_script/01_quickstart_setup.py`로 이동해 여세요.

### 2단계: 설정 후 실행

1. 상단의 드롭다운 위젯에서 **카탈로그**와 **스키마**를 선택하세요
2. **Run All** 클릭
3. 약 10~15분 대기 (대부분 Vector Search 엔드포인트 프로비저닝 시간입니다)

### 3단계: 출력 값 복사

완료되면 노트북이 요약을 출력합니다:

```
======================================================================
  WORKSHOP SETUP COMPLETE
======================================================================
  Catalog/Schema:        my_catalog.retail_agent

  Vector Search Index:   my_catalog.retail_agent.policy_docs_index
  Genie Space ID:        01ef...abcd
  MLflow Experiment ID:  1234567890123456
======================================================================
```

**이 값들을 저장해 두세요** — 워크숍 레벨의 구성 단계에서 필요합니다.

### 완료!

이제 준비가 끝났습니다. 본인의 워크숍 레벨로 이동하세요:

| 레벨 | 다음 단계 |
|-------|-----------|
| Simple (L100) | [`simple/LAB_GUIDE.md`](../simple/LAB_GUIDE.md) |
| Medium (L200) — 워크스페이스 | [`medium/WORKSHOP_INSTRUCTIONS_WORKSPACE.md`](../medium/WORKSHOP_INSTRUCTIONS_WORKSPACE.md) |
| Advanced (L300) — 워크스페이스 | [`advanced/WORKSHOP_INSTRUCTIONS_WORKSPACE.md`](../advanced/WORKSHOP_INSTRUCTIONS_WORKSPACE.md) |

---

## 생성된 리소스

| 리소스 | 설명 |
|----------|-------------|
| `customers` 테이블 | 합성 고객 200명 |
| `products` 테이블 | 리테일 상품 약 500개 |
| `stores` 테이블 | 매장 10곳 |
| `transactions` 테이블 | 거래 2,000건 |
| `transaction_items` 테이블 | 거래 항목 약 8,000건 |
| `payment_history` 테이블 | 결제 기록 400건 |
| `policy_docs_chunked` 테이블 | 검색 가능한 청크로 분할된 정책 문서 |
| Vector Search 인덱스 | 정책 문서에 대한 시맨틱 검색 |
| Genie Space | 리테일 데이터에 대한 자연어 질의 |
| MLflow Experiment | 에이전트 트레이싱과 평가 |

---

## 트러블슈팅

| 증상 | 해결 방법 |
|-------|-----|
| `JSONDecodeError` 또는 인증 오류 | 인증 만료 — `databricks auth login --profile DEFAULT` 실행 |
| `create_resources.py` 타임아웃 | VS 엔드포인트는 10분 이상 걸릴 수 있음 — 다시 실행하세요, 멱등(idempotent)합니다 |
| Vector Search 인덱스가 "Syncing" 표시 | 정상 — 생성 후 초기 동기화까지 5~10분 대기 |
| 노트북 위젯에 카탈로그가 표시되지 않음 | 클러스터에 Unity Catalog 접근 권한이 있는지 확인 |
| `WAREHOUSE_NOT_FOUND` | 먼저 SQL warehouse를 시작하세요 (Compute > SQL Warehouses) |

---

## 기술 참고 자료

### 폴더 구조

```
data/
├── README.md                              ← you are here
├── policy_docs/                           # Source markdown policy documents (7 files)
│   ├── customer_service_guidelines.md
│   ├── delivery_pickup_procedures.md
│   ├── membership_loyalty_program.md
│   ├── privacy_policy.md
│   ├── product_safety_recalls.md
│   ├── return_refund_policy.md
│   └── store_operating_procedures.md
├── local_cli_setup_script/                # Scripts that run from your local machine
│   ├── execute_sql.py                     # Generate structured tables via SQL REST API
│   ├── execute_chunking.py                # Chunk policy docs via SQL REST API
│   └── create_resources.py                # Create Vector Search + Genie Space
└── workspace_setup_script/                # Databricks notebook (does everything on-cluster)
    └── 01_quickstart_setup.py
```

### 스크립트 인자

모든 로컬 CLI 스크립트는 동일한 인자를 받습니다:

| 인자 | 필수 여부 | 기본값 | 설명 |
|----------|----------|---------|-------------|
| `--warehouse-id` | 예 | — | SQL warehouse ID |
| `--catalog` | 예 | — | Unity Catalog 이름 |
| `--schema` | 예 | — | 스키마 이름 |
| `--profile` | 아니요 | `DEFAULT` | Databricks CLI 프로파일 |

`create_resources.py`는 추가로 다음 인자를 받습니다:
| 인자 | 기본값 | 설명 |
|----------|---------|-------------|
| `--vs-endpoint-name` | 자동 생성 | Vector Search 엔드포인트 이름 |
| `--vs-index-name` | `policy_docs_index` | Vector Search 인덱스 이름 |

### 참고

- 모든 스크립트는 재현성을 위해 `random.seed(42)`를 사용합니다
- `create_resources.py`는 멱등합니다 — 중단되어도 다시 실행해도 안전합니다
- 스크립트는 어느 디렉터리에서든 실행할 수 있습니다 (경로는 스크립트 파일 기준으로 해석됩니다)
