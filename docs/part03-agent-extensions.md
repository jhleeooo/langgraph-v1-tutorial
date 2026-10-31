# PART03. 에이전트 확장

5개 챕터, 노트북 9개입니다. Ch06~08은 에이전트에 들어가고 나가는 내용을 통제하고, Ch09~10은 기억과 외부 연결을 다룹니다.

## Ch06. 미들웨어 (`01-LangGraph-Middleware.ipynb`)

미들웨어는 `create_agent` 루프(모델 호출, 도구 실행, 반복, 종료)의 각 단계 전후에 로직을 끼워 넣습니다.

| 후크 | 시점 | 형태 |
|---|---|---|
| `before_agent` / `after_agent` | 실행 시작 전 / 완료 후 (호출당 1회) | 노드 스타일 |
| `before_model` / `after_model` | 매 모델 호출 전 / 후 | 노드 스타일 |
| `wrap_model_call` / `wrap_tool_call` | 모델·도구 호출을 감쌈 | 랩 스타일 |
| `dynamic_prompt` | 매 호출마다 시스템 프롬프트 생성 | 특수 |

### 내장 미들웨어

| 미들웨어 | 역할 | 핵심 파라미터 |
|---|---|---|
| `SummarizationMiddleware` | 토큰 한도 근접 시 자동 요약 | `trigger=("tokens", 4000)`, `keep=("messages", 20)` |
| `ModelCallLimitMiddleware` | 모델 호출 횟수 제한 | `thread_limit`, `run_limit`, `exit_behavior="end"/"error"` |
| `ToolCallLimitMiddleware` | 도구 호출 횟수 제한 (전역 또는 도구별) | `tool_name`, `thread_limit`, `run_limit` |
| `ModelFallbackMiddleware` | 기본 모델 실패 시 대체 모델로 전환 | 대체 모델 목록 |
| `PIIMiddleware` | 이메일, 카드, API 키 등 감지 | `strategy="redact"/"mask"/"block"`, `detector=정규식` |
| `ToolRetryMiddleware` | 도구 실패 시 지수 백오프 재시도 | `max_retries`, `backoff_factor`, `initial_delay`, `max_delay`, `jitter` |

### 커스텀 미들웨어

- 데코레이터 기반: 단일 후크일 때 간단합니다. `before_*`/`after_*`는 `None`을 반환하면 그대로 진행하고, dict를 반환하면 상태를 업데이트합니다.
- 클래스 기반: `AgentMiddleware`를 상속해 여러 후크를 묶고 인스턴스 상태를 유지합니다.
- 랩 스타일은 `handler`를 0번(단락), 1번(정상), 여러 번(재시도) 호출할 수 있습니다.

```python
class DynamicModelMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        model = "gpt-5.4" if len(request.messages) > 10 else "gpt-5.4-mini"
        return handler(request.override(model=init_chat_model(model)))
```

### 실행 순서 (`[M1, M2]`일 때)

- `before_*`: 1 → 2
- `after_*`: 2 → 1 (역순)
- `wrap_*`: 중첩. 첫 번째 미들웨어가 가장 바깥

## Ch07. 컨텍스트 엔지니어링 (`03-LangGraph-Context-Engineering.ipynb`)

에이전트가 실패하는 이유는 모델 능력 부족이거나 올바른 컨텍스트가 전달되지 않아서이고, 대부분 후자입니다. 올바른 정보와 도구를 올바른 형식으로 주는 일이 컨텍스트 엔지니어링입니다.

| 컨텍스트 | 제어 대상 | 지속성 |
|---|---|---|
| Model Context | 모델 호출에 들어가는 내용 (지시, 메시지, 도구, 모델, 응답 형식) | Transient |
| Tool Context | 도구가 읽고 쓰는 내용 | Persistent |
| Life-cycle Context | 모델·도구 호출 사이의 작업 (요약, 가드레일, 로깅) | Persistent |

| 데이터 소스 | 별칭 | 범위 |
|---|---|---|
| Runtime Context | 정적 구성 | 호출 단위 (user_id, 권한, 환경) |
| State | 단기 메모리 | 대화 |
| Store | 장기 메모리 | 대화 간 |

### Model Context

- System Prompt (`@dynamic_prompt`): State 기반(메시지 10개 초과 시 간결하게), Store 기반(`request.runtime.store.get(("preferences",), user_id)`), Runtime Context 기반(`user_role`, `deployment_env`).
- Messages (`@wrap_model_call`): State의 `uploaded_files` 메타데이터를 파일 설명 메시지로 주입.
- Tools: `@tool(parse_docstring=True)`로 인자 설명 추출, `request.override(tools=[...])`로 필터링. State 기반(미인증이면 `public_search`만), Runtime Context 기반(admin 전부, editor는 `delete_data` 제외, viewer는 `read_data`만).
- Model: 대화 길이에 따라 `request.override(model=...)`.
- Response Format: 대화 상태에 따라 `SimpleResponse` 또는 `DetailedResponse`.

### Tool Context

- 읽기: `runtime.state.get(...)`, `runtime.store.get(...)`
- 쓰기: State는 `Command(update={...})` 반환, Store는 `runtime.store.put(...)`

### Life-cycle Context

`SummarizationMiddleware`가 대표 예입니다. Model Context 변경(일시적)과 달리 요약은 상태를 영구히 바꿉니다.

## Ch08. 가드레일 (`04-LangGraph-Guardrail.ipynb`)

가드레일은 미들웨어의 특수한 용도입니다 (PII 유출, 프롬프트 인젝션, 유해 콘텐츠, 규정 준수, 출력 품질).

| | 결정론적 | 모델 기반 |
|---|---|---|
| 방법 | 정규식, 키워드, 명시적 검사 | LLM·분류기가 의미 평가 |
| 장단점 | 빠르고 싸고 예측 가능, 미묘한 위반은 놓침 | 미묘한 문제 포착, 느리고 비쌈 |

- `PIIMiddleware(타입, strategy=..., apply_to_input/output=...)`. 전략은 `redact`(제거), `mask`(일부 표시), `hash`, `block`(예외로 중단). 내장 타입은 email, credit_card, ip, url 등이고 `detector=` 정규식으로 커스텀 타입을 만듭니다.
- `HumanInTheLoopMiddleware`도 가드레일로 분류됩니다.
- 커스텀: `@before_agent(can_jump_to=["end"])`에서 `{"messages": [...], "jump_to": "end"}`를 반환하면 모델 호출 전에 종료합니다. 클래스형은 `@hook_config(can_jump_to=["end"])`를 씁니다.
- `@after_agent` 가드레일은 별도 안전 모델로 SAFE/UNSAFE를 판정합니다.
- 결합: 빠른 검사를 앞에 둡니다. (1) 결정론적 입력 필터 → (2) PII → (3) 민감 도구 HITL → (4) 모델 기반 출력 검사.
- 모범 사례: 계층적 방어, 순서에 따른 성능, 명확한 에러 메시지, 차단 이벤트 로깅.

## Ch09. 메모리 추가하기

단기 메모리는 Checkpointer와 `thread_id`, 장기 메모리는 Store와 네임스페이스입니다.

### `01-LangGraph-Add-Memory.ipynb` (종합편)

- `InMemorySaver`를 `compile(checkpointer=...)`에 주고 `thread_id`로 세션을 구분합니다.
- `get_state()`(values, next, config, metadata)와 `get_state_history()`로 조회합니다.

| 단기 메모리 기법 | 방법 | 특징 |
|---|---|---|
| 트리밍 | `trim_messages(strategy, max_tokens, token_counter, start_on="human", include_system=True)` 를 모델 호출 직전에 | 단순하고 빠르지만 정보 손실 |
| 삭제 | `RemoveMessage(id=...)` | 상태에서 영구 제거 |
| 요약 | `SummaryState(messages, summary)`, 6개 초과 시 앞 4개를 요약하고 삭제 | 정보 보존, LLM 호출 비용 |

- `token_counter=len`은 메시지 개수를 토큰으로 착각하므로 모델 객체를 넘기라고 명시되어 있습니다.
- 프로덕션 체크포인터: `PostgresSaver`(ACID), `SqliteSaver`(로컬), `RedisSaver`(고속). 새 연결로 다시 열어도 대화가 유지됨을 보여 줍니다.

### 나머지 노트북

- `04-LangGraph-Agent-With-Memory.ipynb`: 검색 에이전트에 `InMemorySaver`, `thread_id`별 격리, 스냅샷의 `config`(`checkpoint_id`), `values`, `next`(완료 시 `()`), `metadata`.
- `09-LangGraph-DeleteMessages.ipynb`: `MessagesState`는 `add_messages`가 있어 `RemoveMessage`를 바로 쓸 수 있습니다. 수동 삭제(`update_state`)와 노드 삭제(항상 최신 3개 유지).
- `12-LangGraph-Add-Conversation-Summary.ipynb`: 메시지가 6개를 넘으면 `summarize_conversation`이 기존 요약을 확장하는 프롬프트로 요약하고 마지막 2개를 빼고 삭제합니다. `ask_llm`은 `summary`가 있으면 `SystemMessage`로 앞에 붙입니다.

### `02-LangGraph-Memory-Postgres.ipynb` — 장기 메모리

- Store API: `put(namespace, key, value)`, `get(namespace, key)`, `search(namespace, query/filter)`. 네임스페이스는 `("users","user_001","profile")`처럼 계층적으로 설계합니다.
- 도구에서 `runtime.store`로 읽기와 쓰기(기존 값과 병합), `@dynamic_prompt`와 결합한 개인화, `("learned", user_id, category)`에 사실을 쌓는 학습 에이전트.
- 프로덕션: `PostgresSaver`(thread_id 기반 단기)와 `PostgresStore`(user_id 기반 장기)를 함께 `compile(checkpointer=..., store=...)`에 넣고 `setup()`으로 테이블을 만듭니다.
- `call_model(state, config, *, store)`는 `store.search(("memories", user_id), query=...)`로 관련 기억을 프롬프트에 넣고, 메시지에 "remember"가 있으면 `create_memory_extractor`로 정보를 뽑아 UUID 키로 저장합니다.

## Ch10. LangGraph MCP 실습 (`01-LangGraph-MCP-Tutorial.ipynb` + `server/`)

MCP는 도구와 컨텍스트를 LLM에 붙이는 표준 오픈 프로토콜입니다. LangChain 1.4부터 `langchain.mcp`에 내장됐고(beta), 기존 `langchain-mcp-adapters`의 `MultiServerMCPClient`를 단일 `MCPAdapter`가 대체합니다.

| 서버 파일 | 도구 | 전송 |
|---|---|---|
| `mcp_server_local.py` | `get_weather` (모의) | stdio |
| `mcp_server_remote.py` | `get_current_time(timezone)` | Streamable HTTP, `127.0.0.1:8002/mcp` |
| `mcp_server_rag.py` | `retrieve(query)` | stdio |

`MCPAdapter(target)`은 target 형태로 전송을 자동 결정합니다.

| target | 전송 |
|---|---|
| `Path("server.py")` | stdio (서브프로세스) |
| `"http://.../mcp"` | Streamable HTTP |
| `FastMCP` 인스턴스 | 인메모리 (테스트용) |
| `{"mcpServers": {...}}` | 다중 서버 |

```python
async with MCPAdapter(target) as adapter:
    tools = await adapter.list_tools()   # 컨텍스트 종료 후에도 호출 가능
agent = create_agent(llm, tools, checkpointer=InMemorySaver())
```

- 다중 서버에서는 도구 이름에 `{서버명}_` 접두사가 자동으로 붙습니다.
- ToolNode 통합: MCP 도구에 Tavily를 더해 `bind_tools`, `agent ⇄ tools` 순환 `StateGraph`를 직접 만듭니다.
- 외부 서버: Context7(`npx -y @upstash/context7-mcp@latest`).
- 디버깅: `npx @modelcontextprotocol/inspector`. 도구 오류(`isError=True`)는 `status="error"`인 `ToolMessage`로 전달되고 전송 계층 실패는 예외가 됩니다.

### 관찰

- `server/mcp_rag_server.py`(vector_search, add_document, web_search 제공)는 저장소 어디서도 참조되지 않습니다. 노트북은 `mcp_server_rag.py`만 씁니다.
- Ch09-02의 Postgres URI(`localhost:5659`)는 Ch09-01(`localhost:5432/langgraph_db`)과 포트가 다릅니다.
- `mcp_server_rag.py`의 `retrieve`는 호출마다 PDF를 다시 로드하고 임베딩합니다. 실사용에서는 캐싱이 필요합니다.
