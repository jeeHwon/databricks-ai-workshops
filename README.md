> 🇰🇷 **한국어 현지화 버전** — 원본([AnanyaDBJ/databricks-ai-workshops](https://github.com/AnanyaDBJ/databricks-ai-workshops))의 한국 워크숍용 fork입니다.
> 각 문서의 영어 원본은 같은 폴더의 `*.en.md` 파일로 보존돼 있습니다. 코드·CLI·설정은 원본과 동일하게 동작합니다.

# Databricks에서 프로덕션 스케일의 거버넌스가 적용된 AI 에이전트 구축하기

> 장난감 수준의 데모가 아닌, 프로덕션급 AI 에이전트를 만드는 실전 가이드입니다. 처음부터 배포까지, 평가·거버넌스·모니터링을 첫날부터 기본으로 포함합니다.

---

## 시작하기

### 1. 데이터 설정 (모든 레벨에서 필수)

**→ [`data/README.md`](./data/README.md)**

모든 워크숍 레벨이 공통으로 사용하는 데이터셋(테이블, Vector Search 인덱스, Genie Space)을 생성합니다. 약 15분이 소요됩니다.

### 2. 레벨 선택

각 가이드는 독립적으로 완결됩니다 — 하나를 골라 처음부터 끝까지 따라가세요.

| 레벨 | 만들게 되는 것 | 소요 시간 | 가이드 |
|-------|-------------------|----------|-------|
| **Simple (L100)** | Databricks 관리형 서비스(Genie, Vector Search, Playground)를 활용한 AI 에이전트. 커스텀 코드 없음. | 약 2~3시간 | [Lab Guide](./simple/LAB_GUIDE.md) |
| **Medium (L200)** | OpenAI Agents SDK, MCP 툴, Lakebase 메모리를 갖춘 커스텀 에이전트와 Databricks App으로 배포한 채팅 UI. | 약 3~4시간 | [Local Guide](./medium/WORKSHOP_INSTRUCTIONS.md) / [Workspace Guide](./medium/WORKSHOP_INSTRUCTIONS_WORKSPACE.md) |
| **Advanced (L300)** | MCP, 영구 메모리, 스트리밍 UI, 평가 스위트를 갖춘 프로덕션급 LangGraph 에이전트. | 약 5~6시간 | [Local Guide](./advanced/WORKSHOP_INSTRUCTIONS.md) / [Workspace Guide](./advanced/WORKSHOP_INSTRUCTIONS_WORKSPACE.md) |

---

## 배우게 되는 것

모든 레벨은 동일한 에이전트 라이프사이클을 따릅니다:

1. **구축(Build)** — 툴(Genie, Vector Search, UC Functions, MCP)을 사용해 LLM을 데이터에 연결
2. **평가(Evaluate)** — MLflow 트레이싱, 자동화된 LLM 저지(judge), 프롬프트 반복 개선
3. **거버넌스(Govern)** — AI Gateway, Unity Catalog 권한, 가드레일
4. **배포(Deploy)** — 서비스 프린시펄과 옵저버빌리티를 갖춘 Databricks Apps
5. **개선(Improve)** — 휴먼 인 더 루프(human-in-the-loop) 피드백, 라벨링 세션, 지속적 반복 개선

---

## 리포지토리 구조

```
data/            Shared data setup (run this first)
simple/          L100 — Managed services, no custom code
medium/          L200 — Custom agent with memory (OpenAI Agents SDK + Lakebase)
advanced/        L300 — Full custom stack (LangGraph + MCP + React UI)
```

---

## 이 워크숍이 존재하는 이유

"hello world" 수준의 에이전트 데모는 이미 충분히 많습니다. 정작 부족한 것은 어려운 부분 — 평가, 거버넌스, 모니터링, 그리고 프로토타입과 제품을 가르는 운영의 엄밀함입니다.

이 워크숍은 **테스트되고, 거버넌스가 적용되고, 관측 가능하며, 지속적으로 개선되는** 에이전트를 만드는 방법을 가르칩니다.
