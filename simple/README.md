# L100: Databricks에서 첫 AI 에이전트 만들기

가상의 식료품 체인 **FreshMart**를 위한 대화형 AI 어시스턴트를 만듭니다. 관리형 Databricks 서비스만 사용하며, 초기 설정 이후에는 코딩이 필요 없습니다.

---

## 무엇을 만드나요

| 기능 | 사용 서비스 |
|---|---|
| 자연어 데이터 쿼리 | Genie Space |
| 시맨틱 문서 검색 (RAG) | Vector Search |
| LLM 거버넌스 & 가드레일 | AI Gateway |
| 대화형 에이전트 (노코드) | Playground + 관리형 에이전트 |
| 채팅 UI를 갖춘 프로덕션 배포 | Databricks Apps |
| 자동화된 품질 평가 | MLflow Tracing + LLM 저지(judge) |
| 사람의 피드백 & 레이블링 | Review App |

---

## 시작하기

**전체 단계별 안내:** [`LAB_GUIDE.md`](./LAB_GUIDE.md)

랩 가이드는 8개 섹션으로 구성되어 있습니다 (자율 진행 약 3시간, 강사 진행 약 2.5시간):

1. 데이터 & 리소스 설정
2. AI Gateway
3. Genie, Vector Search, Playground
4. 에이전트 앱 배포
5. MLflow 평가
6. 휴먼 인 더 루프(human-in-the-loop)
7. 관리형 에이전트 (Knowledge Assistant + Supervisor)
8. Document Intelligence (준비 중)

---

## 사전 준비 사항

- 서버리스 SQL 컴퓨트가 활성화된 Databricks 워크스페이스
- Foundation Model API 접근 권한 (Claude Sonnet 4 또는 Llama 3.3 70B)
- 스키마 생성 권한이 있는 Unity Catalog
- Vector Search 및 Databricks Apps 활성화

로컬 툴은 필요 없습니다. 모든 작업이 브라우저에서 실행됩니다.

---

## 폴더 구성

| 경로 | 용도 |
|------|---------|
| `LAB_GUIDE.md` | 전체 워크숍 안내 (여기서 시작하세요) |
| `L100-agent-openai-sdk/` | 에이전트 앱 소스 코드 (OpenAI Agents SDK + MCP) |
| `01_simple_agent_evaluation.ipynb` | MLflow 저지를 활용한 평가 노트북 |
