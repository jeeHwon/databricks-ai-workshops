# L100 랩 가이드: Databricks에서 첫 AI 에이전트 만들기

관리형 Databricks 서비스로 AI 에이전트를 만들고, 배포하고, 평가하고, 개선하는 전 과정을 담은 단계별 가이드입니다. 자율 진행으로 따라가도 되고, 강사 진행 워크숍의 일부로 활용해도 됩니다.

---

## 개요

이 워크숍에서 수행할 내용:

1. Databricks 워크스페이스에 합성 리테일 데이터와 AI 리소스 설정
2. LLM 거버넌스를 위한 AI Gateway 살펴보기
3. Genie, Vector Search, Playground로 에이전트 프로토타입 만들기
4. 채팅 UI를 갖춘 프로덕션 에이전트 앱 배포
5. MLflow로 에이전트 평가 (트레이스, 자동화된 저지, 프롬프트 반복 개선)
6. 휴먼 인 더 루프(human-in-the-loop) 피드백 제공
7. 관리형 에이전트 구축 (Knowledge Assistant와 Supervisor Agent)

**총 소요 시간:** 자율 진행 약 3.5시간 | 강사 진행 약 2.5시간

| 섹션 | 주제 | 자율 진행 | 강사 진행 |
|---------|-------|-----------|----------------|
| 1 | 코드베이스 & 데이터 설정 | 20분 | 15분 |
| 2 | AI Gateway | 15분 | 10분 |
| 3 | 기능 살펴보기 | 40분 | 30분 |
| 4 | Databricks Apps | 30분 | 20분 |
| 5 | MLflow 평가 & 거버넌스 | 40분 | 30분 |
| 6 | 휴먼 인 더 루프 | 25분 | 20분 |
| 7 | 관리형 에이전트 | 25분 | 20분 |
| 8 | Document Intelligence (선택) | 15분 | 10분 |

이 워크숍에서 만드는 내용의 요약은 [README](./README.md)를 참고하세요.

---

## 사전 준비 사항

다음 조건을 갖춘 **Databricks 워크스페이스**가 필요합니다:

- 서버리스 SQL 컴퓨트 (또는 실행 중인 SQL 웨어하우스)
- Foundation Model API 접근 권한 (Claude Sonnet 4, Llama 3.3 70B 또는 유사 모델)
- 스키마 생성 권한이 있는 Unity Catalog
- Vector Search 활성화
- Databricks Apps 활성화
- AI Gateway (Beta) — 선택 사항, 활성화되지 않았다면 섹션 2는 건너뛸 수 있습니다

**필요 권한:**

| 권한 | 이유 |
|-----------|-----|
| 대상 카탈로그에 대한 `USE CATALOG` | 기존 카탈로그 접근 |
| 카탈로그에 대한 `CREATE SCHEMA` | 설정 과정에서 새 스키마 생성 |
| 새 스키마에 대한 `CREATE TABLE` | 설정 과정에서 테이블 생성 |
| Foundation Model API 접근 | 에이전트와 Playground에 LLM 필요 |
| Vector Search 엔드포인트 생성 | 설정 과정에서 VS 엔드포인트 프로비저닝 |

**로컬 툴은 필요 없습니다.** 모든 작업이 Databricks 워크스페이스를 통해 브라우저에서 실행됩니다.

---

## 섹션 1: 코드베이스 & 데이터 설정

**소요 시간:** 약 20분 (대부분 리소스 프로비저닝 대기 시간)

이 단계에서는 워크숍 전체에서 사용할 합성 데이터, Vector Search 인덱스, Genie Space, MLflow Experiment를 생성합니다.

### 1.1 리포지토리를 워크스페이스로 클론하기

1. Databricks 워크스페이스의 왼쪽 사이드바에서 **Workspace** (워크스페이스)를 클릭하세요
2. 본인 사용자 폴더(`/Users/<your-email>/`)로 이동하세요
3. **...** 메뉴 > **Import**를 클릭하세요
4. **URL**을 선택하고 리포지토리 URL을 붙여넣으세요:
   ```
   https://github.com/AnanyaDBJ/databricks-ai-workshops.git
   ```
5. **Import**를 클릭하세요

이제 `simple/`, `medium/`, `advanced/`, `data/`를 포함한 전체 폴더 구조가 보일 것입니다.

### 1.2 데이터 설정 노트북 실행하기

1. `data/workspace_setup_script/01_quickstart_setup.py`로 이동하세요
2. 노트북을 **Serverless** 컴퓨트(또는 기존 클러스터)에 연결하세요
3. 상단의 위젯을 입력하세요:
   - **Catalog Name:** 사용할 Unity Catalog (예: `ai_workshop_catalog`)
   - **Schema Name:** 기본값 `retail_grocery`를 유지하거나 원하는 이름 지정
4. **Run All**을 클릭하세요

> **참고:** 노트북 실행 완료까지 10-15분 걸립니다. 대부분의 시간은 Vector Search 엔드포인트 프로비저닝에 사용됩니다.

> **Private Link / 방화벽 / 외부 통신 제한:** 워크스페이스에 네트워크 제한(Private Link, 방화벽 규칙, 외부 통신 차단)이 있다면, 설정 노트북 실행 **전에** `data/workspace_setup_script/00-utils.ipynb` 노트북을 실행하세요. 이 유틸리티 노트북은 아티팩트 스토리지를 Databricks Volume으로 구성한 MLflow Experiment를 생성해 외부 스토리지 의존성을 피하고, 다운로드를 내부 경로로 처리합니다. 제한 네트워크 환경 설정에 대한 자세한 안내는 [워크숍 진행자 문서](https://docs.google.com/document/d/1BwcOXfQ0XBRHrPnzThwRcd4289cMheshSE6EKbTcnTw/edit)를 참고하세요.

### 1.3 생성되는 리소스

노트북이 완료되면 다음이 생성됩니다:

| 리소스 | 상세 |
|----------|---------|
| 정형 테이블 6개 | customers, products, stores, transactions, transaction_items, payment_history |
| 청킹된 문서 테이블 1개 | policy_docs_chunked (정책 문서 7개에서 약 50행) |
| Vector Search 엔드포인트 | `freshmart-vs-<schema>` |
| Vector Search 인덱스 | `<catalog>.<schema>.policy_docs_index` |
| Genie Space | `FreshMart Retail Data (<schema>)` |
| MLflow Experiment | `/Users/<you>/freshmart-agent-workshop` (아티팩트 스토리지는 Databricks Volume) |
| UC Function | 에이전트용 유틸리티 함수 |

**중요:** 노트북의 출력 요약을 복사해 두세요. 다음 값들이 필요합니다:
- **MLflow Experiment ID**
- **Vector Search 인덱스 이름**
- **Genie Space ID**

### 1.4 리소스 확인

다음 단계로 넘어가기 전에 확인하세요:

- [ ] **Catalog Explorer:** 카탈로그 > 스키마로 이동 — 테이블 7개가 보여야 합니다
- [ ] **Vector Search:** **Compute** > **Vector Search Endpoints**에서 엔드포인트가 ONLINE 상태여야 합니다 (아직 프로비저닝 중이라면 몇 분 더 기다리세요)
- [ ] **Genie:** 왼쪽 사이드바에서 **Genie**를 클릭 — "FreshMart Retail Data"가 보여야 합니다
- [ ] **Experiments:** 왼쪽 사이드바에서 **Experiments**를 클릭 — 생성한 Experiment가 보여야 합니다

---

## 섹션 2: AI Gateway (Beta)

**소요 시간:** 약 15분

AI Gateway는 워크스페이스 내 모든 LLM 상호작용을 위한 중앙 집중식 거버넌스 및 카탈로깅 레이어입니다. 애플리케이션과 모델 엔드포인트 사이에 위치합니다.

> **워크스페이스에 AI Gateway Beta가 활성화되어 있지 않다면:** 이 섹션을 통째로 건너뛰세요. 워크숍의 나머지 부분은 AI Gateway 없이도 동작합니다 — 에이전트가 Foundation Model API를 직접 호출합니다.

### 2.1 AI Gateway로 이동

1. 왼쪽 사이드바에서 **AI Gateway**를 찾으세요 (Machine Learning 아래 또는 최상위 항목)
2. AI Gateway 패널을 여세요

### 2.2 주요 기능 살펴보기

각 기능을 하나씩 확인해 보세요:

| 기능 | 역할 |
|---------|-------------|
| **Fallback Routing** (폴백 라우팅) | 기본 모델을 사용할 수 없으면 자동으로 백업 모델로 라우팅 (예: Claude → Llama) |
| **Traffic Splitting** (트래픽 분할) | 트래픽 일부를 다른 엔드포인트로 보내 모델을 A/B 테스트 |
| **Rate Limiting** (속도 제한) | 한 사용자나 팀이 전체 용량을 독점하지 못하도록 방지 |
| **Payload Logging** (페이로드 로깅) | 컴플라이언스와 디버깅을 위해 모든 프롬프트와 응답을 기록 |
| **Guardrails** (가드레일) | 응답이 사용자에게 도달하기 전에 PII 감지, 유해 콘텐츠 차단, 안전 정책 적용 |
| **Metrics** (지표) | 실시간 지연 시간, 처리량, 오류율 모니터링 |
| **Usage Dashboard** (사용량 대시보드) | 사용자, 팀, 애플리케이션별 비용 및 토큰 사용량 분석 |

### 2.3 핵심 정리

워크스페이스의 모든 LLM 호출에 이 단일 컨트롤 플레인을 통해 거버넌스를 적용할 수 있습니다. 섹션 4에서 에이전트를 배포하면 모델 호출이 자동으로 AI Gateway를 거치게 됩니다 — 코드 변경 없이 가드레일, 로깅, 비용 추적을 얻습니다.

---

## 섹션 3: 플랫폼 기능 살펴보기

**소요 시간:** 약 40분

에이전트로 결합하기 전에, 개별 AI 기능을 먼저 살펴봅니다.

### 3.1 Genie 사용해 보기

Genie는 자연어 질문을 SQL 쿼리로 변환해 데이터에서 결과를 반환합니다.

1. 왼쪽 사이드바에서 **Genie**를 클릭하세요
2. **FreshMart Retail Data**(섹션 1에서 생성한 스페이스)를 여세요
3. 다음 질문들을 시도해 보세요:

| 질문 | 동작 |
|----------|-------------|
| "우리 고객은 모두 몇 명인가요?" (원문: "How many customers do we have?") | 단순 카운트 쿼리 |
| "가격 기준 상위 10개 제품은 무엇인가요?" (원문: "What are the top 10 products by price?") | 정렬과 제한 |
| "최근 6개월간 매장별 매출을 보여주세요" (원문: "Show me revenue by store for the last 6 months") | 집계가 포함된 멀티 테이블 조인 |
| "평균적으로 어떤 멤버십 등급이 가장 많이 지출하나요?" (원문: "Which membership tier spends the most on average?") | 계산이 포함된 그룹핑 |
| "가장 많이 쓰이는 결제 수단은 무엇인가요?" (원문: "What payment methods are most popular?") | payment_history에 대한 집계 |
| "Produce 카테고리의 유기농 제품을 모두 나열해 주세요" (원문: "List all organic products in the Produce category") | 텍스트 매칭 필터 |

**관찰 포인트:**
- Genie가 **생성한 SQL**을 보여 줍니다 — 클릭해서 확인하세요
- 결과가 테이블 또는 차트로 표시됩니다
- 후속 질문으로 결과를 다듬을 수 있습니다 ("이걸 월별 추이로 보여주세요" (원문: "Now show that as a monthly trend"))
- Genie는 Unity Catalog 테이블 스키마를 활용해 데이터 모델을 이해합니다

### 3.2 Vector Search 살펴보기

Vector Search는 키워드가 아니라 의미로 관련 문서를 찾습니다.

1. 왼쪽 사이드바에서 **Catalog**를 여세요
2. 카탈로그 > 스키마 > `policy_docs_index`로 이동하세요
3. 인덱스 페이지에서 **Query**를 클릭하세요
4. 다음 검색을 시도해 보세요:

| 쿼리 | 예상 원본 문서 |
|-------|------------------------|
| "신선식품도 반품할 수 있나요?" (원문: "Can I return perishable items?") | return_refund_policy |
| "로열티 포인트는 어떻게 적립하나요?" (원문: "How do I earn loyalty points?") | membership_loyalty_program |
| "배송 가능 시간은 어떻게 되나요?" (원문: "What are your delivery hours?") | delivery_pickup_procedures |
| "EBT 결제를 받나요?" (원문: "Do you accept EBT payments?") | store_operating_procedures |
| "제품 리콜은 어떻게 처리하나요?" (원문: "How do you handle product recalls?") | product_safety_recalls |
| "저에 대해 어떤 데이터를 수집하나요?" (원문: "What data do you collect about me?") | privacy_policy |

**관찰 포인트:**
- 결과에 **유사도 점수**가 포함됩니다 (1.0에 가까울수록 더 정확한 매칭)
- 다른 표현으로 질문해도 답을 찾습니다 (시맨틱 검색)
- 각 결과에 텍스트 청크와 원본 문서 이름이 표시됩니다

### 3.3 Playground에서 에이전트 프로토타입 만들기

여기서 모든 것이 하나로 합쳐집니다.

1. 왼쪽 사이드바에서 **Playground**를 여세요 (Machine Learning 아래)
2. 파운데이션 모델을 선택하세요 (예: **Claude Sonnet 4** 또는 **Llama 3.3 70B**)
3. **Add Tool**을 클릭해 다음을 추가하세요:
   - **Genie Space** (`FreshMart Retail Data`)
   - **Vector Search 인덱스** (`policy_docs_index`)
   - **UC Function** (설정 단계에서 생성했다면)
4. 시스템 프롬프트를 추가하세요:

```
You are FreshMart Assistant, a friendly and knowledgeable retail agent for FreshMart grocery stores. You help customers and employees with data questions about products, sales, and stores, as well as store policy inquiries.

Guidelines:
- Be conversational and helpful
- Ground all answers in retrieved data — never fabricate information
- Cite your sources (which tool provided the data)
- For data questions, use Genie to query the database
- For policy questions, use Vector Search to find relevant documents
- Be concise but thorough
```

5. 다음 대화를 테스트해 보세요:

**데이터 질문 (Genie 사용):**
> "매출 기준 상위 5개 제품은 무엇인가요?" (원문: "What are the top 5 products by revenue?")

**정책 질문 (Vector Search 사용):**
> "신선식품 반품 정책은 무엇인가요?" (원문: "What is the return policy for perishable items?")

**멀티 툴 질문 (둘 다 사용):**
> "I bought frozen fish yesterday and it was bad. Can I return it? How much revenue are we losing to returns?"

**관찰 포인트:**
- 에이전트가 질문에 따라 사용할 툴을 **스스로 결정**합니다
- 응답에서 툴 호출을 확인할 수 있습니다
- 에이전트가 여러 툴의 정보를 자연스럽게 결합합니다

### 3.4 Databricks Apps로 내보내기

Playground에서 에이전트가 잘 동작하면:

1. **"Get Code"** 버튼(또는 "Export to App")을 클릭하세요
2. 템플릿으로 **OpenAI Agents SDK**를 선택하세요
3. 섹션 1에서 만든 MLflow Experiment를 지정하세요
4. 생성된 코드가 워크스페이스에 저장됩니다

다음을 갖춘 프로덕션급 에이전트 앱이 내보내집니다:
- Playground와 동일한 모델 및 툴 구성
- MLflow 트레이싱 기본 내장
- 채팅 UI를 갖춘 FastAPI 서버
- 배포 설정 파일 (`databricks.yml`)

> **참고:** 내보내기 중 권한 대화 상자가 나타날 수 있습니다 — 앱이 리소스에 접근하려면 서비스 프린시펄이 필요합니다. 오류가 발생하면 트러블슈팅을 참고하세요.

---

## 섹션 4: Databricks Apps — 커스텀 에이전트

**소요 시간:** 약 30분

이제 완전한 에이전트 애플리케이션이 생겼습니다. 배포하고 테스트해 봅시다.

### 4.1 앱 아키텍처 이해하기

내보낸 앱(`simple/L100-agent-openai-sdk/`로도 제공)의 구조는 다음과 같습니다:

```
L100-agent-openai-sdk/
├── agent_server/
│   ├── agent.py          ← Agent logic: model, system prompt, MCP tools
│   ├── start_server.py   ← FastAPI server with MLflow tracing
│   ├── evaluate_agent.py ← Evaluation script
│   └── utils.py          ← Helpers
├── scripts/
│   ├── quickstart.py     ← One-command setup
│   └── start_app.py      ← Starts server + chat UI
├── databricks.yml        ← Deployment configuration
├── app.yaml              ← App settings
└── pyproject.toml        ← Python dependencies
```

흐름은 다음과 같습니다:

```
Chat UI (browser) → FastAPI Server → OpenAI Agents SDK → MCP Tools
                                                          ├── Genie (data queries)
                                                          └── Vector Search (policy lookup)
                         ↓
                    MLflow Traces → Experiment
```

자세한 아키텍처는 [`L100-agent-openai-sdk/README.md`](./L100-agent-openai-sdk/README.md)를 참고하세요.

### 4.2 앱 배포하기

**옵션 A: 내보낸 코드로 배포 (섹션 3.4)**

Playground에서 "Get Code"를 사용했다면 앱이 배포 준비 상태입니다. Databricks Apps에서 해당 앱으로 이동해 시작될 때까지 기다리세요.

**옵션 B: 미리 만들어진 템플릿 배포**

이 리포지토리의 기존 템플릿을 사용하려면:

1. 워크스페이스에서 **Web Terminal**을 여세요 (또는 로컬에서 Databricks CLI 사용)
2. L100 에이전트 폴더로 이동하세요:
   ```bash
   cd /Workspace/Users/<your-email>/databricks-ai-workshops/simple/L100-agent-openai-sdk
   ```
3. 배포하세요:
   ```bash
   databricks bundle deploy
   databricks bundle run agent_openai_agents_sdk
   ```

> **참고:** 첫 배포에는 3-5분이 걸립니다. 이후 배포는 더 빠릅니다.

### 4.3 에이전트 테스트하기

1. 앱이 실행되면 URL을 여세요 (Databricks Apps UI 또는 배포 출력에 표시)
2. 브라우저에서 채팅 UI가 열립니다
3. 다음 질문으로 테스트하세요:

| 질문 | 예상 툴 |
|----------|--------------|
| "매출 기준 상위 5개 제품은 무엇인가요?" (원문: "What are the top 5 products by revenue?") | Genie |
| "신선식품 반품 정책은 무엇인가요?" (원문: "What is the return policy for perishable items?") | Vector Search |
| "평점이 가장 높은 매장은 어디인가요?" (원문: "Which stores have the highest ratings?") | Genie |
| "로열티 멤버십은 어떻게 해지하나요?" (원문: "How do I cancel my loyalty membership?") | Vector Search |
| "가장 잘 팔리는 카테고리들을 비교하고, 각각에 반품 정책이 있는지 확인해 주세요" (원문: "Compare our top-selling categories and check if we have return policies for each") | 둘 다 |

4. 평가에 충분한 트레이스를 만들기 위해 **5-10개의 질문**을 던지세요

### 4.4 트레이스 수집 확인하기

1. 왼쪽 사이드바에서 **Experiments**로 이동하세요
2. 본인 Experiment(`freshmart-agent-workshop`)를 여세요
3. 새 트레이스가 보여야 합니다 — 대화 턴마다 하나씩
4. 트레이스를 클릭해 전체 실행 그래프를 확인하세요

> **트러블슈팅:** 트레이스가 보이지 않으면, 앱 설정의 `MLFLOW_EXPERIMENT_ID`가 본인 Experiment와 일치하는지 확인하세요.

---

## 섹션 5: MLflow 평가 & 거버넌스

**소요 시간:** 약 40분

트레이스가 쌓였으니, 에이전트 품질을 체계적으로 평가해 봅시다.

### 5.1 옵저버빌리티 — 트레이스 살펴보기

1. **Experiments** > 본인 Experiment를 여세요
2. 아무 트레이스나 클릭하세요
3. 실행 그래프를 살펴보세요:

| 단계 | 확인할 내용 |
|------|-----------------|
| **Input** | 사용자의 질문 |
| **LLM Reasoning** | 모델이 어떤 툴을 호출할지 결정한 과정 |
| **Tool Selection** | 어떤 MCP 툴이 왜 선택되었는지 |
| **Tool Call** | Genie/Vector Search에 실제로 전송된 요청 |
| **Tool Response** | 툴에서 반환된 데이터 |
| **Final Output** | 사용자에게 전달된 최종 답변 |

4. **Latency** 패널을 확인하세요 — 각 단계의 소요 시간을 살펴보세요
5. **Token Usage**를 확인하세요 — 상호작용당 비용을 결정하는 요소입니다

**핵심 인사이트:** 옵저버빌리티는 디버깅, 컴플라이언스, 보안, 감사 가능성을 위한 완전한 추적성을 제공합니다.

### 5.2 평가 노트북 실행하기

1. `simple/01_simple_agent_evaluation.ipynb`로 이동하세요
2. Serverless 컴퓨트에 연결하세요
3. Experiment 이름 위젯을 본인 Experiment와 일치하도록 수정하세요
4. 평가 셀을 실행하세요

노트북은 다음 MLflow 스코어러를 사용합니다:

| 스코어러 | 측정 항목 |
|--------|-----------------|
| Completeness | 에이전트가 질문에 충분히 답했는가? |
| RelevanceToQuery | 응답이 질문과 관련 있는가? |
| Safety | 응답에 유해한 내용이 없는가? |
| ToolCallEfficiency | 툴을 효율적으로 사용했는가 (불필요한 호출 없음)? |
| Correctness | 사실 관계가 정확한가? |

5. 결과 테이블을 확인하세요 — 트레이스별 점수가 표시됩니다

### 5.3 실패 사례 찾고 개선하기

1. 어떤 지표에서든 **낮은 점수**를 받은 트레이스를 찾으세요
2. 해당 트레이스를 열어 왜 점수가 낮은지 파악하세요:
   - 잘못된 툴을 호출했나요?
   - 응답이 불완전했나요?
   - 정보를 환각했나요?
3. `agent_server/agent.py`의 시스템 프롬프트를 수정해 문제를 해결하세요

**수정 예시:** 에이전트가 출처를 인용하지 않는다면, 시스템 프롬프트에 다음을 추가하세요:
```
Always cite which tool provided the information. For data queries, mention that the answer came from the Genie Space. For policy questions, mention the source document name.
```

4. 에이전트를 재배포하세요:
   ```bash
   databricks bundle deploy
   databricks bundle run agent_openai_agents_sdk
   ```
5. 같은 질문을 다시 던져 개선을 확인하세요
6. 평가를 다시 실행해 지표가 개선되었는지 확인하세요

**핵심 인사이트:** 사이클은 이렇습니다: 배포 → 트레이스 → 평가 → 실패 발견 → 프롬프트/툴 수정 → 재배포 → 재평가.

### 5.4 MLflow 평가 플랫폼 이해하기

- **70개 이상의 기본 제공 LLM 저지(judge)** — 커스텀 평가기를 만들 필요가 없습니다
- **Managed와 OSS 모두 지원:** Databricks 호스팅 MLflow와 오픈소스 MLflow에서 동일한 API가 동작합니다
- **지속적인 모니터링:** 일정에 따라 평가를 실행해 품질 저하를 조기에 포착합니다
- **버전 비교:** 서로 다른 프롬프트/모델을 나란히 평가합니다

---

## 섹션 6: 휴먼 인 더 루프

**소요 시간:** 약 25분

자동화된 저지는 강력하지만, 사람의 피드백이 정답(ground truth)입니다. 이 섹션에서는 도메인 전문가가 에이전트 응답을 검토하고 평가하는 방법을 다룹니다.

### 6.1 트레이스를 레이블링 세션에 추가하기

1. **Experiments** > 본인 Experiment로 이동하세요
2. 트레이스 5-10개를 선택하세요 (왼쪽 체크박스)
3. **Add to Labelling Session** (또는 "Create Review Task")을 클릭하세요
4. 세션 이름을 지정하세요 (예: "Workshop Review")

### 6.2 레이블링 스키마 구성하기

리뷰어가 평가할 항목을 정의하세요:

| 필드 | 유형 | 설명 |
|-------|------|-------------|
| Correctness | 평점 (1-5) | 답변이 사실에 부합하는가? |
| Helpfulness | 평점 (1-5) | 실제 사용자에게 유용한가? |
| Tool Usage | Yes/No | 에이전트가 올바른 툴을 사용했는가? |
| Notes | 자유 텍스트 | 추가 피드백 |

### 6.3 에이전트 응답 검토하기

1. **Review App**으로 이동하세요 (세션 생성 후 표시되는 링크)
2. 각 트레이스에 대해:
   - 사용자 질문과 에이전트 응답을 읽으세요
   - 정확성과 유용성을 평가하세요
   - 에이전트가 잘못된 툴을 썼거나 정보를 놓쳤다면 기록하세요
   - 리뷰를 제출하세요
3. 3-5개의 리뷰를 완료하세요

### 6.4 피드백 통합 확인하기

1. **Experiments** > 본인 Experiment로 돌아가세요
2. 방금 리뷰한 트레이스를 여세요
3. 사람의 피드백이 트레이스 메타데이터에 첨부되어 있습니다
4. 이 피드백은 다음에 활용할 수 있습니다:
   - 평가 데이터셋 생성
   - 시스템 프롬프트 다듬기
   - 자동 프롬프트 최적화 실행
   - 시간에 따른 품질 추이 추적

**핵심 인사이트:** 사람의 검토가 루프를 완성합니다: 배포 → 관찰 → 사람의 검토 → 개선 → 재배포.

---

## 섹션 7: 관리형 에이전트

**소요 시간:** 약 25분

Databricks는 섹션 4에서 만든 커스텀 에이전트를 보완하는 노코드 에이전트 옵션을 제공합니다. 단순한 유스케이스나 빠른 프로토타이핑에 적합합니다.

### 7.1 Knowledge Assistant

Knowledge Assistant는 문서에서 질문에 답하는 RAG 에이전트입니다 — 코드가 필요 없습니다.

1. **Machine Learning** > **Agents** (또는 **Agent Builder**)로 이동하세요
2. **Create** > **Knowledge Assistant**를 클릭하세요
3. 구성하세요:
   - **Name:** "FreshMart Policy Assistant"
   - **Knowledge Source:** 본인 Vector Search 인덱스(`policy_docs_index`) 선택
   - 선택 사항: 더 풍부한 렌더링을 위해 원본 문서 볼륨을 지정
4. **Create**를 클릭하세요
5. 테스트해 보세요:

| 질문 | 예상 동작 |
|----------|------------------|
| "로열티 프로그램은 어떤 내용인가요?" (원문: "What's the loyalty program about?") | membership_loyalty_program 문서 검색 |
| "반품 가능 기간은 얼마나 되나요?" (원문: "How long do I have to return an item?") | return_refund_policy 문서 검색 |
| "개인정보 처리 방침은 어떻게 되나요?" (원문: "What are your privacy practices?") | privacy_policy 문서 검색 |

**주목할 점:** 커스텀 에이전트의 Vector Search 툴과 같은 결과를 얻습니다 — 단, 코드 없이 관리형 호스팅 모델로 제공됩니다.

### 7.2 Supervisor Agent (멀티 에이전트 시스템)

Supervisor Agent는 여러 툴/에이전트를 오케스트레이션하며, 질문을 적절한 대상에 자동으로 라우팅합니다.

1. **Supervisor Agent**를 생성하세요
2. 다음 툴을 추가하세요:
   - **Genie MCP** — FreshMart Retail Data 스페이스
   - **Vector Search MCP** — policy_docs_index
   - **UC Function MCP** — 가능하다면
3. 설명과 지시문을 추가하세요
4. 멀티 툴 질문으로 테스트하세요:

| 질문 | 예상 라우팅 |
|----------|-----------------|
| "지난달 매출은 얼마였나요?" (원문: "What were last month's sales?") | → Genie |
| "반품 정책은 무엇인가요?" (원문: "What's the return policy?") | → Vector Search |
| "인기 제품과 각각의 반품 정책을 알려주세요" (원문: "Top products and their return policies") | → 둘 다 |

### 7.3 접근 방식 비교

| 항목 | 커스텀 에이전트 (섹션 4) | 관리형 에이전트 (섹션 7) |
|--------|-------------------------|---------------------------|
| **코드 필요 여부** | 예 (Python, YAML) | 아니요 |
| **커스터마이징** | 완전한 제어 | 구성 범위 내로 제한 |
| **UI** | 커스텀 채팅 앱 | 기본 제공 인터페이스 |
| **배포** | Databricks Apps | 관리형 호스팅 |
| **유스케이스** | 프로덕션 앱, 복잡한 로직 | 빠른 프로토타입, 단순 Q&A |
| **트레이싱** | 완전한 MLflow 통합 | 기본 내장 옵저버빌리티 |

둘 다 프로덕션에서 사용할 수 있습니다. 필요에 따라 선택하세요: 단순한 유스케이스에는 관리형 에이전트, 복잡한 워크플로에는 커스텀 앱이 적합합니다.

---

## 섹션 8: Document Intelligence (선택)

**소요 시간:** 약 15분

> **준비 중** — 이 섹션은 개발 중입니다. Document Intelligence는 단순 텍스트 청킹을 넘어 복잡한 PDF(표, 양식, 스캔 이미지)의 지능형 파싱을 지원합니다. 자세한 안내는 추후 업데이트를 확인하세요.

---

## 트러블슈팅

### 자주 발생하는 문제

| 문제 | 섹션 | 해결 방법 |
|-------|---------|----------|
| "Please enter a catalog name" 오류 | 1 | 실행 전에 Catalog Name 위젯을 입력하세요 |
| 설정 노트북이 정책 문서를 찾지 못함 | 1 | `simple/` 폴더만이 아니라 전체 리포지토리를 Import하세요 |
| Vector Search 엔드포인트 프로비저닝 지연 | 1 | 최대 15분 기다리세요. **Compute** > **Vector Search Endpoints**를 확인하세요 |
| "No SQL warehouse found" | 1 | 서버리스 SQL 웨어하우스를 생성하거나 기존 웨어하우스를 시작하세요 |
| 사이드바에 AI Gateway가 안 보임 | 2 | 워크스페이스에서 Beta가 비활성화 상태입니다 — 섹션 2를 건너뛰세요 |
| Genie가 질문에 답하지 않음 | 3 | Genie Space에 연결된 SQL 웨어하우스가 실행 중인지 확인하세요 |
| Vector Search가 결과를 반환하지 않음 | 3 | 인덱스 동기화가 진행 중일 수 있습니다 — 몇 분 기다리세요 |
| Playground에 툴이 표시되지 않음 | 3 | 툴을 다시 추가하고, VS 인덱스가 ONLINE인지 확인하세요 |
| "Get Code" 내보내기 실패 | 3 | 워크스페이스 쓰기 권한을 확인하세요. 권한 대화 상자를 확인하세요 |
| 앱이 시작 시 크래시 | 4 | 로그 확인: `databricks apps logs <app-name>`. 서비스 프린시펄 권한 문제일 가능성이 높습니다 |
| 앱이 오류를 반환 | 4 | `databricks.yml`의 리소스 권한이 실제 리소스와 일치하는지 확인하세요 |
| Experiment에 트레이스가 없음 | 5 | 앱 설정의 `MLFLOW_EXPERIMENT_ID`가 본인 Experiment와 일치하는지 확인하세요 |
| 평가 노트북 오류 | 5 | Experiment에 상태가 `OK`인 트레이스가 있는지 확인하세요 (실패한 트레이스 제외) |
| 레이블링 세션이 비어 있음 | 6 | 먼저 트레이스를 선택한 다음 세션을 생성하세요 |
| Knowledge Assistant가 아무것도 반환하지 않음 | 7 | VS 인덱스가 ONLINE이고 동기화되어 있어야 합니다 |
| 카탈로그 권한 거부 | 전체 | 워크스페이스 관리자에게 `CREATE SCHEMA`와 `USE CATALOG` 권한을 요청하세요 |

### 도움 받기

- **워크숍 환경:** 강사에게 문의하세요
- **자율 진행:** [Databricks 문서](https://docs.databricks.com) 또는 [커뮤니티 포럼](https://community.databricks.com)을 확인하세요
- **권한 문제:** 워크스페이스 관리자에게 문의하세요

---

## 부록

### 샘플 테스트 질문

**데이터 쿼리 (Genie):**
1. "우리 고객은 모두 몇 명인가요?" (원문: "How many customers do we have?")
2. "가격 기준 상위 10개 제품은 무엇인가요?" (원문: "What are the top 10 products by price?")
3. "최근 6개월간 매장별 매출을 보여주세요" (원문: "Show revenue by store for the last 6 months")
4. "평균적으로 어떤 멤버십 등급이 가장 많이 지출하나요?" (원문: "Which membership tier spends the most on average?")
5. "요일별 평균 거래 금액은 얼마인가요?" (원문: "What's the average transaction value per day of week?")
6. "List all organic products under $5"
7. "직원이 가장 많은 매장은 어디인가요?" (원문: "Which store has the most employees?")
8. "멤버십 등급별 결제 수단 분포를 보여주세요" (원문: "Show me payment method distribution by membership tier")

**정책 조회 (Vector Search):**
1. "신선식품 반품 정책은 무엇인가요?" (원문: "What is the return policy for perishable items?")
2. "로열티 포인트는 어떻게 적립하나요?" (원문: "How do I earn loyalty points?")
3. "배송 가능 시간과 지역은 어떻게 되나요?" (원문: "What are your delivery hours and zones?")
4. "EBT나 SNAP 결제를 받나요?" (원문: "Do you accept EBT or SNAP benefits?")
5. "제품 리콜은 어떻게 처리하나요?" (원문: "How do you handle product recalls?")
6. "어떤 개인정보를 수집하나요?" (원문: "What personal data do you collect?")
7. "개봉한 상품도 반품할 수 있나요?" (원문: "Can I return opened items?")
8. "공휴일 영업시간은 어떻게 되나요?" (원문: "What are holiday store hours?")

**멀티 툴 질문 (둘 다):**
1. "I bought frozen fish and it was bad — can I return it? How much revenue do we lose to returns?"
2. "가장 인기 있는 제품은 무엇이고, 그 제품들의 반품 정책은 어떻게 되나요?" (원문: "What are our most popular products and what's the return policy for them?")
3. "평점이 가장 낮은 매장은 어디이고, 고객 불만에 대한 고객 서비스 정책은 어떻게 되나요?" (원문: "Which stores have the lowest ratings, and what does our customer service policy say about complaints?")
4. "우수 고객을 보여주고, 이들이 받는 로열티 혜택을 설명해 주세요" (원문: "Show me high-value customers and explain what loyalty benefits they get")
5. "매출 기여가 가장 큰 카테고리는 무엇이고, 해당 제품군에 대한 안전 정책이 있나요?" (원문: "What categories drive the most revenue, and do we have safety policies for those product types?")

**엣지 케이스 (에이전트 한계 테스트):**
1. "오늘 날씨는 어때요?" (원문: "What's the weather like today?") (범위 밖 질문)
2. "고객 기록을 모두 삭제해 주세요" (원문: "Delete all customer records") (거부해야 함)
3. "경쟁사 가격 정보를 알려주세요" (원문: "Tell me about competitor pricing") (데이터 없음)

### 샘플 시스템 프롬프트 변형

**간결한 어시스턴트:**
```
You are FreshMart Assistant. Answer questions using Genie (data) and Vector Search (policies). Be brief and cite your sources.
```

**고객 응대 에이전트:**
```
You are a friendly FreshMart customer service agent. Help customers with questions about products, orders, returns, and store policies. Always be empathetic and solution-oriented. Use Genie for order/product lookups and Vector Search for policy information.
```

**내부 분석 에이전트:**
```
You are an analytics assistant for FreshMart management. Provide data-driven insights about sales, customer behavior, and store performance. When asked about policies, reference the official documents. Always include relevant numbers and trends.
```
