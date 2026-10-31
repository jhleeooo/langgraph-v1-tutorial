# PART04. 멀티에이전트

3개 챕터, 노트북 4개입니다. Supervisor(중앙 집중) → Network(분산 협업) → Hierarchical Team(계층) 순입니다.

## Ch11. 멀티에이전트 Supervisor

멀티에이전트가 필요한 상황은 도구 과부하, 컨텍스트 오버플로우, 전문화 필요입니다.

| 패턴 | 작동 방식 | 제어 흐름 |
|---|---|---|
| Tool Calling (Subagents) | 수퍼바이저가 다른 에이전트를 도구로 호출. 하위 에이전트는 사용자와 직접 대화하지 않음 | 중앙 집중식 |
| Handoffs (Supervisor) | 현재 에이전트가 제어권 자체를 넘김. 활성 에이전트가 바뀜 | 분산형 |

### `01-LangGraph-Supervisor.ipynb`

1. Tool Calling: `@tool`로 감싼 하위 에이전트를 메인 에이전트가 호출합니다.
2. `langgraph-supervisor` 라이브러리:
```python
supervisor = create_supervisor(
    model=..., agents=[research_agent, math_agent],
    add_handoff_back_messages=True, output_mode="full_history",
).compile()
```
3. 커스텀 Supervisor: `Command`로 핸드오프를 직접 구현합니다.
```python
def create_handoff_tool(*, agent_name, description=None):
    @tool(f"transfer_to_{agent_name}", description=description)
    def handoff_tool(state: Annotated[MessagesState, InjectedState],
                     tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
        return Command(goto=agent_name,
                       update={"messages": state["messages"] + [tool_message]},
                       graph=Command.PARENT)
    return handoff_tool
```
그래프는 supervisor와 worker 노드를 `StateGraph(MessagesState)`에 추가하고, worker는 항상 supervisor로 복귀하는 엣지를 갭니다.
4. 고급 위임: Supervisor가 명시적 작업 지시문을 쓰고 `Command(goto=[Send(agent_name, {"messages": [task_description_message]})], graph=Command.PARENT)`로 전달합니다. 정확성, 효율성, 일관성이 좋아집니다.

이어서 3단계 계층 에이전트(Tool Calling 패턴), 고객 지원 시스템(기술/청구/일반), 베스트 프랙티스(전문화 원칙, 오류 처리와 재시도, 병렬 처리·캐싱·배치)를 다룹니다.

### `03-LangGraph-Multi-Agent-Supervisor.ipynb`

같은 Supervisor 패턴을 두 방식으로 비교합니다.

- StateGraph 방식: `AgentState(messages, next)`, `RouteResponse(next: Literal[...])` 구조화 출력 라우터, `add_conditional_edges`. `functools.partial(agent_node, agent=..., name=...)`로 노드를 만듭니다.
- Agent-as-Tool 방식: 워커를 `@tool`로 래핑하고 `create_agent` 한 번으로 Supervisor를 만듭니다.

| | StateGraph | Agent-as-Tool |
|---|---|---|
| 상태 정의 | 직접 | 자동 |
| 라우팅 | `RouteResponse` + 조건부 엣지 | LLM 도구 호출 |
| 제어 수준 | 높음 | 낮음 (LLM에 위임) |
| 적합한 경우 | 복잡한 커스텀 워크플로우 | 빠른 프로토타이핑 |

## Ch12. 협업 네트워크 (`02-LangGraph-Multi-Agent-Collaboration.ipynb`)

중앙 통제자가 없는 네트워크 패턴입니다. Research Agent(웹 검색)와 Chart Generator Agent(Python 실행)가 서로를 호출하며 순환합니다. 에이전트가 답을 마치면 응답에 `FINAL ANSWER`를 포함하고, 라우터가 이를 감지해 종료합니다.

```python
def router(state):
    if "FINAL ANSWER" in last_message_text: return END
    return "continue"

workflow.add_conditional_edges("researcher", router, {"continue": "chart_generator", END: END})
workflow.add_conditional_edges("chart_generator", router, {"continue": "researcher", END: END})
```

시스템 프롬프트로 "다른 assistant가 남은 부분을 이어갈 것"이라는 협업 규칙을 심어 둡니다.

## Ch13. 계층적 에이전트 팀 (`04-LangGraph-Hierarchial-Agent-Team.ipynb`)

팀의 팀을 만듭니다.

1. `AgentFactory`: 같은 LLM 설정으로 워커 노드를 반복 생성합니다 (`create_agent_node(agent, name)`).
2. Research Team(Searcher, WebScraper)과 Doc Writing Team(DocWriter, NoteTaker, ChartGenerator)을 각자의 Sub-Supervisor와 함께 독립 `StateGraph`로 컴파일합니다. Doc Writing Supervisor는 프롬프트에 작업 순서(`NoteTaker → DocWriter → ChartGenerator → DocWriter → FINISH`)를 박아 무한 루프를 막습니다.
3. Super-Graph: 팀 그래프 전체를 하나의 노드로 연결합니다.
```python
super_graph.add_node("ResearchTeam", get_last_message | web_research_app | join_graph)
super_graph.add_node("PaperWritingTeam", get_last_message | authoring_app | join_graph)
super_graph.add_node("Supervisor", supervisor_node)
```
Runnable 파이프(`|`)로 "메시지 추출 → 하위 그래프 실행 → 결과 취합"을 하나의 노드로 합성하는 것이 핵심입니다. 총 감독자는 팀 단위 라우팅만 결정합니다.

## 요약

| 챕터 | 패턴 | 통제 구조 |
|---|---|---|
| Ch11 | Supervisor | 중앙 감독자 1명 (Handoffs 또는 Agent-as-Tool) |
| Ch12 | Network | 중앙 통제 없음, FINAL ANSWER 신호 |
| Ch13 | Hierarchical Team | 팀 내부는 Supervisor, 팀 간에는 Super-Graph |
