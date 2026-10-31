# PART01. LangGraph 기초

## Ch01. 그래프 생성하기

노트북 6개이며 세 덩어리입니다.

- 재료: State(01), 모델(01), 메시지(02)
- 뼈대: Graph API(02, 03)
- 통합: QuickStart로 챗봇, 도구, 메모리, HITL, 타임트래블을 한 번에

README 표의 순서(Introduction, Models, QuickStart, Messages, Graph-API, Building-Graphs)보다 아래 순서가 앞 개념이 뒤에서 그대로 쓰여 읽기 편합니다.
`01-Introduction` → `02-Messages` → `01-Models` → `03-Building-Graphs` → `02-QuickStart-Graph-API` → `01-QuickStart-Tutorial`

### 1. `01-LangGraph-Introduction.ipynb` — State의 문법

- `TypedDict`와 `dict`: dict는 값 타입 변경이나 키 추가에 제약이 없습니다. `TypedDict`는 키와 타입을 명시하고, 정적 타입 검사기가 잘못된 대입과 정의되지 않은 키를 잡습니다. 런타임에는 강제되지 않습니다.
- `Annotated[타입, 메타데이터]`: 타입에 메타데이터를 붙입니다. Pydantic `Field(gt=18, lt=65)`처럼 검증에 쓰이고, LangGraph에서는 리듀서(병합 규칙) 지정에 쓰입니다.
- `add_messages`: 리스트에 메시지를 추가하는 리듀서입니다. 같은 `id`의 메시지는 추가하지 않고 교체합니다.

```python
add_messages([HumanMessage("안녕하세요?", id="1")], [AIMessage("반갑습니다~", id="1")])
# id가 같으므로 AIMessage 하나로 교체됨
```

### 2. `03-LangGraph-Building-Graphs.ipynb` — 기본 부품

| 요소 | 핵심 |
|---|---|
| StateGraph | `StateGraph(State)` → `add_node` → `add_edge` → `compile()` → `invoke()` |
| State / Reducer | 기본은 덮어쓰기, `Annotated[list, add]`는 누적, `MessagesState`는 messages 필드와 리듀서가 미리 정의됨 |
| Node | `state`를 받아 업데이트할 부분만 dict로 반환. `config: RunnableConfig`를 받으면 `thread_id` 접근 가능 |
| Edge | `add_edge`(항상), `add_conditional_edges(source, 라우팅 함수, path_map)`(분기) |
| Send / Command | Send는 map-reduce식 동적 병렬, Command는 상태 업데이트와 이동을 한 번에 |

- 노드가 `{"foo": 2}`만 반환해도 `bar`는 유지됩니다.
- `Send`: 주제 목록에서 주제마다 농담 생성 노드를 병렬 실행하고 `jokes`를 `add`로 병합합니다.
- `Command(update=..., goto=...)`: `decision_node`가 `count`를 올리고 3 미만이면 `continue_node`, 이상이면 `end_node`로 보냅니다.
- Recursion Limit: 한 실행의 최대 super-step 수이며 넘으면 `GraphRecursionError`가 납니다.

### 3. `02-QuickStart-LangGraph-Graph-API.ipynb` — 심화

- 실행 모델: Pregel 방식의 super-step입니다. 같은 단계의 노드는 병렬로 실행됩니다.
- 다중 스키마: `StateGraph(InternalState, input_schema=InputState, output_schema=OutputState)`로 입력, 내부, 출력, private 스키마를 분리합니다.
- Config 스키마: `StateGraph(MyState, context_schema=ConfigSchema)`.
- 노드 캐싱: `add_node(..., cache_policy=CachePolicy(ttl=..., key_func=...))` 와 `compile(cache=InMemoryCache())`.
- 엣지: 일반, 조건부, 매핑 기반, 병렬. 한 노드에서 나가는 엣지가 여러 개면 자동 병렬 실행됩니다.
- `Command(graph=Command.PARENT)`로 상위 그래프의 노드로 이동합니다.

| Command | 조건부 엣지 |
|---|---|
| 상태 업데이트와 라우팅을 동시에 할 때, 멀티 에이전트 핸드오프 | 상태 변경 없이 라우팅만 할 때, 단순 분기 |

### 4. `01-LangGraph-Models.ipynb` — 모델 API

- 모델 지정: `init_chat_model("gpt-5.4")`, `provider:model` 문자열, `ChatOpenAI(temperature, max_tokens, timeout)` 직접 사용.
- 호출: `invoke()`(동기), `stream()`(토큰 스트리밍), `ainvoke()` / `astream()`(비동기), `batch()`(병렬 일괄).
- 응답: `AIMessage`의 `content`, `usage_metadata`(토큰 수), `response_metadata`(finish_reason 등).
- `bind_tools()` 후 `tool_calls` 확인, `with_structured_output(Pydantic)`으로 구조화된 출력.
- `UsageMetadataCallbackHandler`로 여러 모델의 누적 토큰을 추적하고, `config`의 `run_name`, `tags`, `metadata`는 LangSmith에서 확인합니다.
- 기타: `reasoning_effort`, 멀티모달(teddynote `MultiModal`), `logprobs`, OpenAI 호환 API(`base_url`, `model_kwargs`와 `extra_body`의 차이).

### 5. `02-LangGraph-Messages.ipynb` — 메시지

- 입력 형태: 문자열(내부적으로 `HumanMessage`), 메시지 객체 리스트, `{"role": ..., "content": ...}` 딕셔너리.
- 타입: `SystemMessage`, `HumanMessage`, `AIMessage`(`tool_calls` 포함), `ToolMessage`.
- `ToolMessage.tool_call_id`는 호출한 `AIMessage.tool_calls[].id`와 일치해야 합니다. 표준 흐름은 `Human → AI(tool_calls) → Tool → AI`입니다.
- 콘텐츠: 문자열, Provider 네이티브 형식(`image_url`), 표준 `content_blocks`.

### 6. `01-QuickStart-LangGraph-Tutorial.ipynb` — 통합 투어

챗봇에 기능을 한 단계씩 얹습니다. PART02~05의 예고편입니다.

1. 기본 챗봇: State, chatbot 노드, START/END 엣지, compile, 실행
2. 도구: Tavily + `ToolNode` + `tools_condition`
3. 메모리: `call_model(state, config, *, store)`가 `create_memory_extractor`로 정보를 Store에 저장, `InMemorySaver`와 `InMemoryStore` 함께 컴파일 (Ch09-02의 원형)
4. HITL: 도구 안에서 `interrupt({"query": ...})`, `Command(resume={"data": ...})`로 재개. interrupt 중 병렬 도구 호출은 `assert`로 막습니다.
5. 상태 커스터마이징: 도구가 `Command(update=...)`로 상태 갱신 (Ch16의 원형)
6. 타임트래블: `get_state_history()`, 체크포인트 롤백, `update_state`, 재실행 (Ch15의 원형)

### 관찰

- QuickStart의 `RunnableConfig(recursion_limit=20, thread_id="abc123")`는 `thread_id`를 `configurable` 밖에 넣었습니다. 그 지점에는 체크포인터가 없어 문제가 없지만, 메모리를 붙인 뒤에는 `configurable={"thread_id": ...}` 형태로 써야 합니다.
- 노트북 파일명의 번호(`01-`, `02-`, `03-`)가 겹치고 학습 순서와 다릅니다.
