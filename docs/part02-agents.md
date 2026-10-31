# PART02. 에이전트

4개 챕터, 노트북 8개입니다. `create_agent`를 중심으로 에이전트의 구성요소를 하나씩 쌓습니다.

## Ch02. 에이전트

### `01-LangGraph-Agents.ipynb`

- 모델: `provider:model` 문자열과 `init_chat_model` 두 방식, `temperature`/`max_tokens`/`timeout` 설정, `ChatOpenAI`/`ChatAnthropic` 직접 사용.
- 동적 모델 선택: `wrap_model_call` 미들웨어가 요청(`ModelRequest`)을 검사해 조건부로 모델을 바꿉니다.
- 프롬프트: 정적 시스템 프롬프트와 동적 프롬프트.
- 미들웨어: 클래스 기반 작성, 모델 오류 시 재시도.

### `02-LangGraph-ChatBot.ipynb`

StateGraph로 챗봇을 처음부터 만듭니다. State 정의 → Node 정의 → Graph 정의와 노드 추가 → Edge 추가 → compile → 시각화 → 실행.

### `02-LangGraph-Tools.ipynb`

- `@tool` 데코레이터, 타입 힌트의 중요성, 커스텀 이름과 설명.
- Pydantic 모델로 입력 스키마를 정의하고 검증 오류를 다룹니다.
- `ToolRuntime`으로 상태 업데이트, Context, 메모리(Store), Stream Writer에 접근합니다.

### `03-QuickStart-LangGraph-Ollama.ipynb`

Ollama 로컬 모델로 에이전트 생성, 도구, Context, 응답 형식, 단기 메모리, 미들웨어, Human-in-the-Loop 미들웨어 개요를 압축해서 다룹니다.

## Ch03. Runtime (`04-LangGraph-Runtime.ipynb`)

- 도구에서 Context, Store(장기 메모리), Stream Writer에 접근
- 미들웨어(Dynamic Prompt, Before/After Model)에서 Runtime 사용
- 종합 예제: 사용자 컨텍스트 기반 에이전트
- 실전 패턴: DB 연결 전달, 인증과 권한 검사, 요청별 설정

## Ch04. 구조화된 출력 (`05-LangGraph-Structured-Output.ipynb`)

`response_format`으로 구조화된 출력을 받습니다.

- Provider Strategy: Pydantic, dataclass, TypedDict
- Tool Calling Strategy: Pydantic, Union 타입, 커스텀 도구 메시지 콘텐츠
- 오류 처리: 스키마 검증 오류, 처리 비활성화, 특정 예외만, 여러 예외, 커스텀 핸들러 함수

## Ch05. Human-in-the-Loop

두 노트북이 서로 다른 방식으로 "승인을 기다렸다가 재개"를 보여 줍니다.

### `02-LangGraph-Human-In-The-Loop.ipynb` — `HumanInTheLoopMiddleware`

작동 순서

1. 모델이 도구 호출을 생성
2. 미들웨어가 `after_model` 훅에서 도구 호출을 검사
3. 승인이 필요하면 `interrupt()`로 실행을 일시 중지
4. 상태는 체크포인터에 저장되고 `invoke()` 결과에 `__interrupt__`가 담김
5. 사람이 결정하면 `Command(resume=...)`로 같은 `thread_id`에서 재개

```python
agent = create_agent(
    model=model,
    tools=[write_file, read_file, delete_file],
    middleware=[HumanInTheLoopMiddleware(interrupt_on={
        "write_file": True,    # approve, edit, reject 모두 허용
        "delete_file": True,
        "read_file": False,    # 승인 불필요
    })],
    checkpointer=InMemorySaver(),   # 필수
)
```

| 결정 | 동작 |
|---|---|
| `{"type": "approve"}` | 도구 호출을 그대로 실행 |
| `{"type": "edit", "edited_action": {...}}` | 인수를 바꿔서 실행 |
| `{"type": "reject", "message": "..."}` | 실행 거부와 함께 모델에게 피드백 |

- `interrupt_on`에서 `{"allowed_decisions": ["approve", "reject"]}`로 도구별 허용 결정을 제한합니다. SQL 실행이나 금전 이체는 수정 불가로 둡니다.
- 도구 호출이 여러 개면 `decisions`를 `action_requests` 순서와 맞춰 각각 보냅니다.
- 금액 임계값을 넘으면 코드에서 자동으로 `reject` 결정을 만드는 조건부 승인 정책도 구현할 수 있습니다.
- 모범 사례: 체크포인터 필수(운영은 `PostgresSaver`), `thread_id` 관리, 결정 순서, 편집은 소폭만, 위험도별 승인 정책.

### `06-LangGraph-Human-In-the-Loop.ipynb` — `interrupt_before`

```python
graph = graph_builder.compile(checkpointer=memory, interrupt_before=["tools"])
```

- `graph.get_state(config).next`가 `('tools',)`이면 tools 노드 직전에서 멈춘 상태입니다.
- 재개는 입력으로 `None`을 넘깁니다: `graph.stream(None, config)`.
- `get_state_history()`로 과거 체크포인트를 골라 그 시점부터 다시 실행합니다.

| | HumanInTheLoopMiddleware | interrupt_before |
|---|---|---|
| 세밀도 | 도구별 승인 필요 여부와 허용 결정 지정 | 노드 단위로 무조건 중단 |
| 사람의 개입 | approve/edit/reject | 승인 후 재개 (편집 로직은 직접 구현) |

### 대기는 어떻게 이루어지나

이 노트북들에는 실제로 기다리는 코드가 없습니다(`input()`, `sleep` 등 없음). `interrupt()`는 내부적으로 `GraphInterrupt` 예외를 일으키고, 실행기가 상태를 체크포인터에 저장한 뒤 `invoke()`를 그냥 반환시킵니다. 프로세스가 블로킹되는 것이 아닙니다.

- 대기, 즉 사용자에게 알리고 응답을 받는 일은 호출하는 애플리케이션이 구현합니다.
- 응답이 오면 같은 `thread_id`로 새 요청에서 `Command(resume=...)`를 호출해 이어서 실행합니다.
- 그래서 운영에서는 영속 체크포인터(`PostgresSaver` 등)가 필요합니다. 프로세스가 재시작되어도 `thread_id`만 있으면 재개할 수 있어야 하기 때문입니다.
