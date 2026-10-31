# Appendix

노트북 16개이며 A(RAG 응용 10), B(유즈케이스 5), C(GraphRAG 1)입니다. A와 B에는 `rag/` 헬퍼 패키지(`pdf.py`, `base.py`, `utils.py`)가, A에는 예제 PDF가, B에는 `Chinook.db`가 들어 있습니다.

## A. RAG 응용

- `01-Building-Graphs`: 문자열만 반환하는 자리표시자 노드로 토폴로지를 연습합니다. `operator.add`(누적)와 일반 필드(덮어쓰기)의 차이를 봅니다.
- `02~05`는 같은 뼈대를 한 단계씩 키웁니다. 모두 `PDFRetrievalChain`의 `pdf_retriever`, `pdf_chain`을 노드에서 나눠 씁니다.

| 노트북 | 추가되는 것 | 그래프 |
|---|---|---|
| 02 Naive RAG | 기본 파이프라인 | `START → retrieve → llm_answer → END` |
| 03 관련성 체크 | `relevance_check`(`GroundednessChecker`) | 관련 없으면 `retrieve`로 되돌아가는 루프 (`recursion_limit`과 `GraphRecursionError` 시연) |
| 04 웹 검색 | `web_search`(Tavily) | 관련 없으면 웹 검색으로 폴백 |
| 05 쿼리 재작성 | `query_rewrite` | `question`을 `Annotated[List[str], add_messages]`로 바꿔 원본과 재작성 질문 누적, 검색에는 마지막 질문 사용 |

- `06 Agentic RAG`: 에이전트가 검색 여부를 결정합니다.
```python
workflow.add_conditional_edges("agent", tools_condition, {"tools": "retrieve", END: END})
workflow.add_conditional_edges("retrieve", grade_documents)   # -> "generate" | "rewrite"
workflow.add_edge("rewrite", "agent")
```
`grade_documents`는 `GradeDocuments(binary_score)` 구조화 출력으로 yes/no를 판정합니다.
- `07 CRAG`: 문서별 관련성을 평가해 걸러 내고, 관련 문서가 하나도 없으면 `query_rewrite → web_search_node → generate`로 보완합니다.
- `08 Self-RAG`: 문서 평가에 더해 생성 결과를 검증합니다. 평가기는 문서 관련성, 환각 여부, 답변 관련성 세 가지입니다.
```python
workflow.add_conditional_edges("generate", grade_generation_v_documents_and_question,
    {"hallucination": "generate", "relevant": END, "not relevant": "transform_query"})
```
- `09 Adaptive RAG`: 07과 08에 입구 라우터를 더했습니다. `RouteQuery(datasource: Literal["vectorstore", "web_search"])`로 `add_conditional_edges(START, route_question, ...)`에서 갈라지고, 이후는 문서 평가, 생성, 환각 체크, 답변 평가, 쿼리 변환 루프입니다.
- `10 Retrieval`: `create_agent`와 LCEL로 RAG 아키텍처 3종을 비교합니다.

| 아키텍처 | 특징 |
|---|---|
| 2-Step RAG | 항상 검색 후 생성. 단순하고 빠름 |
| Agentic RAG | 에이전트가 검색 시점을 결정. 다중 도구 조합 가능 |
| Hybrid RAG | 쿼리 향상, 검색 충분성 검증(YES/NO), 반복 개선 |

모범 사례: 청크 `chunk_size=500, overlap=50`, `k`는 단순 질문 2 / 복잡 질문 5, 컨텍스트에 답이 없으면 "정보가 부족합니다"라고 답하게 지시, 메타데이터(source/page/topic/date/author) 활용.

## B. 유즈케이스

- `01 에이전트 시뮬레이션`: 가상 고객 LLM이 상담사 챗봇과 대화합니다. `_swap_roles`로 고객 LLM에게는 AI와 Human 메시지 역할을 뒤집어 넘기고, 고객은 끝나면 `FINISHED`라고 답합니다. `should_continue`는 메시지 6개 초과 또는 `FINISHED`면 종료합니다.
- `02 메타 프롬프트 생성`: `PromptInstructions`(필수 `objective`, `context`와 선택 필드 다수)를 도구로 바인딩해 정보가 충분하면 LLM이 호출하게 합니다. 노드는 `info`, `add_tool_message`(유효성 검사, 부족하면 재수집), `prompt`(META_PROMPT로 생성)이고 `progress` 필드로 진행 단계를 추적합니다.
- `03 Plan-and-Execute`: `planner → execute → replan → (execute | final_report)`.
  - 상태: `input`, `plan`, `past_steps`(누적), `response`
  - 계획은 `PLANNER_MODEL`, 실행은 가벼운 `EXECUTOR_MODEL`(Tavily 에이전트)
  - `replan`은 `Act = Union[Response, Plan]` 구조화 출력으로 최종 답변 또는 남은 단계 재계획을 결정하며, 프롬프트 규칙으로 단계 확장을 막습니다.
- `04 SQL Agent`: Chinook.db(SQLite). `first_tool_call`(강제로 `sql_db_list_tables`) → `list_tables_tool` → `model_get_schema` → `get_schema_tool` → `query_gen` ⇄ `correct_query`(LLM 점검) → `execute_query`.
  - `create_tool_node_with_fallback`이 도구 오류를 `ToolMessage`로 돌려 쿼리를 고치게 합니다.
  - 종료는 `SubmitFinalAnswer` 도구 호출로 명시합니다.
  - 마지막에 LangSmith Evaluator로 평가합니다 (`GradeAnswer` 0/1 LLM 채점).
- `05 Research Assistant (STORM)`:
  1. `Analyst` 여러 명을 생성하고 `compile(interrupt_before=["human_feedback"])`로 멈춰 `update_state(..., as_node="human_feedback")`로 피드백을 줍니다.
  2. 인터뷰 서브그래프: `ask_question → (search_web ‖ search_arxiv) → answer_question → (route_messages 반복) → save_interview → write_section`
  3. `Send()`로 분석가마다 인터뷰를 병렬 실행하고 `sections`를 `operator.add`로 모읍니다.
  4. `write_report`, `write_introduction`, `write_conclusion`이 병렬로 돌고 `finalize_report`가 합칩니다.

## C. GraphRAG — Text2Cypher (Neo4j)

Neo4j AuraDB 또는 Docker(APOC 포함)를 쓰고 `NEO4J_URI/USERNAME/PASSWORD` 환경변수와 `Neo4jGraph`로 연결한 뒤 Movies 샘플을 로드합니다.

```
START → get_schema → generate_cypher → validate_cypher → execute_cypher
        execute_cypher ─(should_retry)→ generate_answer | generate_cypher(재시도) → END
```

- 스키마 주입: `refresh_schema()` 결과를 프롬프트에 넣고 스키마에 있는 레이블, 관계, 속성만 쓰고 DML은 금지하도록 지시합니다.
- 안전장치: `validate_cypher`가 `CREATE/DELETE/SET/MERGE/DROP/REMOVE/DETACH`와 빈 쿼리를 검사합니다.
- 재시도: 오류가 나면 `retry_count`를 올려 최대 `MAX_RETRIES=3`까지 재생성하고, 초과하면 오류를 포함해 답변을 생성합니다.

## 관찰

- `validate_cypher`는 쿼리를 대문자로 바꿔 부분 문자열로 금지어를 찾습니다. 코드를 읽은 것이며 실행은 해 보지 않았지만, `n.created`나 `dataset` 같은 정상 프로퍼티명에도 `CREATE`/`SET`이 걸려 오탐이 날 수 있어 보입니다.
- SQL Agent와 Text2Cypher는 자유 쿼리 실행을 다룹니다. 실서비스에서는 읽기 전용 계정 같은 권한 제한이 필요합니다.
