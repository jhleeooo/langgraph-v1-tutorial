# 학습 가이드

`langgraph-v1-tutorial`의 PART01~05와 Appendix 노트북을 읽고 정리한 설명 문서입니다. 노트북 54개, 챕터 25개를 다룹니다. 내용은 노트북 본문과 코드를 읽고 정리한 것이며, 노트북을 직접 실행해 검증한 것은 아닙니다.

시각화된 진행 현황은 [learning-map.html](learning-map.html)을 브라우저로 열어 확인합니다.

## 프로젝트 요약

LangGraph V1의 핵심 개념과 실전 활용을 다루는 한국어 Jupyter Notebook 튜토리얼입니다.

- 패키지 관리: UV (`pyproject.toml`, `uv.lock`), Python 3.11 이상
- 핵심 라이브러리: `langchain` >= 1.4.0, `langgraph` >= 1.2.11, `deepagents` >= 0.7, `fastmcp` >= 4.0 / `mcp` >= 2.1
- 기본 모델: OpenAI `gpt-5.4` (Anthropic, Ollama로 교체 가능)
- 부가 도구: PostgreSQL/Redis 체크포인터, Neo4j, FAISS/Pinecone, LangSmith

## 문서 목록

| 문서 | 범위 | 챕터 |
|---|---|---|
| [part01-langgraph-basics.md](part01-langgraph-basics.md) | LangGraph 기초 | Ch01 |
| [part02-agents.md](part02-agents.md) | 에이전트 | Ch02~05 |
| [part03-agent-extensions.md](part03-agent-extensions.md) | 에이전트 확장 | Ch06~10 |
| [part04-multi-agent.md](part04-multi-agent.md) | 멀티에이전트 | Ch11~13 |
| [part05-core-features.md](part05-core-features.md) | 핵심 기능 구현하기 | Ch14~22 |
| [appendix.md](appendix.md) | RAG 응용, 유즈케이스, GraphRAG | A, B, C |

## 권장 학습 순서

PART01 → PART02 → PART03 → PART04 → PART05 → Appendix

## 챕터 간 개념 연결

| 처음 등장 | 다시 쓰이는 곳 |
|---|---|
| `add_messages`의 id 교체 (Ch01) | `RemoveMessage` (Ch09), 도구 호출 수정 (Ch15) |
| `context_schema` (Ch01) | Runtime (Ch03), 컨텍스트 엔지니어링 (Ch07) |
| `Send`, `Command`, `Command.PARENT` (Ch01) | 핸드오프·위임 (Ch11), 병렬·서브그래프 (Ch18~19) |
| `interrupt`, `Command(resume)` (Ch01) | Human-in-the-Loop (Ch05), 상태 커스터마이징 (Ch16) |
| 다중 스키마, super-step (Ch01) | 병렬 분기 (Ch18), 서브그래프 변환 (Ch20) |
| 미들웨어 (Ch06) | 컨텍스트 (Ch07), 가드레일 (Ch08) |
