# PART05. 핵심 기능 구현하기

9개 챕터, 노트북 11개입니다. 고수준 API(`create_agent`, 미들웨어) 아래에서 실제로 돌아가는 LangGraph 저수준 메커니즘을 다룹니다.

## Ch14. Agent 구축 (`03-LangGraph-Agent.ipynb`)

`create_agent` 없이 ReAct 루프를 StateGraph로 직접 만듭니다.

- `State.messages: Annotated[list, add_messages]`
- `llm.bind_tools(tools)`로 만든 `chatbot` 노드가 도구 호출 여부를 결정합니다.
- `BasicToolNode`를 직접 구현해 `tool_calls`를 순회 실행하고 `ToolMessage`로 반환합니다.
- `route_tools`가 `tool_calls`가 있으면 `"tools"`, 없으면 `END`를 반환하고, `add_conditional_edges(source, path, path_map)`로 연결합니다. `tools → chatbot` 엣지로 순환합니다.
- 메시지 흐름: `HumanMessage → AIMessage(tool_calls) → ToolMessage → AIMessage`.

## Ch15. 상태 수동 업데이트 (`07-LangGraph-Manual-State-Update.ipynb`)

체크포인터와 `interrupt_before=["tools"]`로 멈춘 뒤 `graph.update_state()`로 상태를 고치고 재개합니다.

1. 도구 결과 교체: 사람이 쓴 결과를 `ToolMessage(tool_call_id=...)`로 주입합니다. `as_node="tools"`를 주면 tools 노드가 실행된 것처럼 처리됩니다.
2. 도구 호출 인자 수정: 마지막 `AIMessage`를 복사해 `tool_calls[0]["args"]["query"]`만 바꿉니다. 메시지 `id`를 기존과 같게 두면 `add_messages`가 추가하지 않고 교체합니다.
3. 과거 스냅샷 Replay: `get_state_history()`에서 시점을 골라 수정한 뒤 `graph.stream(None, updated_state)`로 그 지점부터 다시 실행합니다.

재개할 때는 항상 입력으로 `None`을 넘깁니다.

## Ch16. 상태 커스터마이징 (`08-LangGraph-State-Customization.ipynb`)

메시지 밖에 상태 필드를 추가해 LLM이 필요할 때만 사람을 부르게 만듭니다.

- `State`에 `ask_human: bool` 추가.
- `HumanRequest`(Pydantic)는 LLM에 도구처럼 바인딩되지만 실제로 실행되지 않고 플래그 전환용이라 `ToolNode`에 넣지 않습니다.
- `chatbot`이 `HumanRequest`를 호출하면 `ask_human=True`로 반환하고, `select_next_node`가 플래그가 켜져 있으면 `"human"`, 아니면 `tools_condition`으로 보냅니다.
- `compile(interrupt_before=["human"])`로 멈추고, 사람이 `update_state`로 `ToolMessage`를 넣은 뒤 `None`으로 재개합니다. 응답이 없으면 `human_node`가 "No response from human."을 채웁니다.

## Ch17. ToolNode (`10-LangGraph-ToolNode.ipynb`)

- 마지막 메시지가 `tool_calls`를 가진 `AIMessage`이면 그래프 없이도 동작합니다. `tool_calls`가 여러 개면 병렬 실행됩니다.
- 표준 에이전트 그래프: `agent → tools_condition → tools → agent`.
- `handle_tool_errors`:

| 값 | 동작 |
|---|---|
| `True`(기본) | 오류를 `ToolMessage`(status=error)로 LLM에 전달 |
| `False` | 예외를 그대로 발생 |
| 문자열 | 고정 메시지 전달 |
| 함수 | 오류 객체로 메시지 생성 |
| `(ValueError, TypeError)` | 지정한 예외만 처리 |

- `ToolRuntime`을 도구 인자로 선언하면 `runtime.state["messages"]`처럼 그래프 상태에 접근합니다. LLM 스키마에는 노출되지 않습니다.

## Ch18. 병렬 분기 (`11-LangGraph-Branching.ipynb`)

`aggregate: Annotated[list, operator.add]` 상태로 실험합니다.

- fan-out / fan-in: A에서 B, C로 갈라져 병렬 실행되고 D에서 합쳐집니다. 이 병렬 단위가 super-step입니다.
- 예외: super-step은 트랜잭션처럼 처리되어 분기 하나만 실패해도 그 step의 상태 업데이트가 전부 적용되지 않습니다. 대응은 노드 안 try/except 또는 `retry_policy`(실패한 분기만 재시도)입니다.
- 다단계 경로: `add_edge(["b2", "c"], "d")`로 모두 끝나야 d가 실행됩니다.
- `defer=True`: 선행 노드를 나열하지 않아도 대기 중인 작업이 모두 끝난 뒤 실행됩니다.
- 조건부 분기: `which` 값에 따라 `b,c` 또는 `c,d`로 갈라지고, `then="e"`로 공통 종착 노드를 지정할 수 있습니다.
- 신뢰도 순 정렬: 병렬 결과의 적용 순서는 보장되지 않으므로 `fanout_values`에 신뢰도와 함께 기록한 뒤 sink 노드에서 정렬합니다.

## Ch19. 서브그래프 (`13-LangGraph-Subgraph.ipynb`)

- Case 1 (상태 키 공유): 컴파일된 서브그래프를 `builder.add_node("node_2", subgraph)`로 바로 추가합니다. `name`이 공유 키라 상태가 자동 전달되고, `family_name`은 서브그래프 전용입니다.
- Case 2 (키 없음): 서브그래프를 호출하는 노드 함수를 직접 작성해 호출 전후로 상태를 변환합니다.
- 같은 노드 함수 안에서 서브그래프를 둘 이상 호출하면 네임스페이스가 충돌하므로 노드를 나눕니다.
- `stream(..., subgraphs=True)`는 `(namespace, chunk)`를 반환합니다. 빈 튜플은 부모, `('node_2:...',)`는 서브그래프 내부입니다.

## Ch20. 서브그래프 입출력 변환 (`14-LangGraph-Subgraph-Transform-State.ipynb`)

`grandchild → child → parent` 3계층을 서로 다른 키(`my_grandchild_key`, `my_child_key`, `my_parent_key`)로 만듭니다.

```python
def call_grandchild_graph(state: ChildState) -> ChildState:
    out = grandchild_graph.invoke({"my_grandchild_key": state["my_child_key"]})   # 입력 변환
    return {"my_child_key": f'([Child] {out["my_grandchild_key"]})'}               # 출력 변환
child.add_node("child_1", call_grandchild_graph)   # 컴파일된 그래프가 아니라 래퍼 함수
```

키가 없는데 컴파일된 그래프를 바로 `add_node`하면 오류가 납니다. 래퍼는 독립된 서브그래프를 잇는 인터페이스이며 각 서브그래프를 따로 테스트하고 재사용할 수 있게 합니다. PART04 Ch13의 팀 구성 원리입니다.

## Ch21. 스트리밍 모드

### `03-LangGraph-Streaming.ipynb`

| 모드 | 내용 |
|---|---|
| `updates`(기본) | 노드 완료마다 `{노드명: 출력}` |
| `messages` | LLM 토큰 `(AIMessageChunk, metadata)`, 채팅 UI용 |
| `custom` | 도구 안에서 `runtime.stream_writer({...})`로 보내는 데이터 (진행률 등) |

- 모드를 리스트로 주면 `(mode, data)` 튜플로 동시에 받습니다.
- `init_chat_model(..., disable_streaming=True)`로 토큰 스트리밍만 끕니다.
- `messages`는 오버헤드가 가장 크므로 필요한 모드만 씁니다.

### `05-LangGraph-Streaming-Outputs.ipynb`

- `output_keys`로 특정 상태 키만 받습니다.
- `values`(매 단계 전체 상태)와 `updates`(변경분)를 비교합니다.
- `interrupt_before`/`interrupt_after`는 `compile()`에서 설정하고 체크포인터가 필수입니다.

### `15-LangGraph-Streaming-Steps.ipynb`

- 동기 `stream`과 비동기 `astream`으로 values, updates, messages를 실행합니다.
- `metadata["langgraph_node"]`로 특정 노드의 토큰만 필터링합니다.
- `llm.with_config(tags=[...])`와 `astream_events(version="v2")`로 태그 기반 필터링을 합니다.
- `subgraphs=True`로 서브그래프 내부 토큰까지 스트리밍합니다.
- 이벤트 스트리밍 v3 (beta): `stream = await graph.astream_events(inputs, version="v3")`가 스트림 객체를 반환합니다. `stream.messages`, `stream.subgraphs`(`graph_name`, `path`), `await stream.output()` 프로젝션으로 v2의 네임스페이스 파싱이 필요 없어집니다.

## Ch22. Agent Chat UI (`Agent-Chat-UI-OSS.ipynb`)

1. `langgraph.json`의 `graphs`에 `"agent": "./agent.py:graph"`를 지정하고, `agent.py`에서 `create_agent`(Tavily)로 `graph`를 정의합니다. 서버가 체크포인터를 자동 관리하므로 따로 지정하지 않습니다.
2. 노트북에서 로컬 테스트와 멀티턴 확인.
3. `langgraph dev`로 `http://localhost:2024`에 서버 실행.
4. Node.js(fnm), pnpm 설치 후 Chat UI(Next.js 15 / React 19)를 클론해 `pnpm launch` 마법사로 연결.
5. 도구 호출 시각화, 스레드 관리, SSE 스트리밍, LangSmith 연동, 다크/라이트 테마.

## 관찰

- Ch22의 서버 실행 명령이 `cd 13-Deployment`로 되어 있는데 현재 폴더는 `Ch22-Agent-Chat-UI-OSS`입니다. 예전 경로가 남은 것으로 보입니다.
- Ch15는 `graph.stream(..., interrupt_before=["tools"])`처럼 stream 호출에 interrupt를 넘기는데, Ch21은 v1에서는 `compile()`에서 설정한다고 안내합니다. 어느 쪽이 현재 버전에서 동작하는지는 실행해 보지 않아 확인이 필요합니다.
