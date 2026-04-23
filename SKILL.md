---
name: alms-langgraph-agent
description: Build, refactor, or review Python FastAPI AI-agent services using KJ's optimized ALMS LangGraph/LangChain style. Use when creating LangGraph workflows, LangChain agents, structured-output agents, prompt managers, agent managers, action/usecase layers, API endpoints, or service run/test instructions that should follow the ALMS architecture, using asr-transcribe-masking-service only as lived implementation evidence for agent behavior and workflow examples.
---

# ALMS LangGraph Agent

Use this skill to build agentic FastAPI services in the optimized ALMS style: API endpoints stay thin, usecases orchestrate business flow, actions execute compiled workflows, and the agent layer owns prompts, structured outputs, nodes, tools, and LangGraph wiring.

Treat `alms` as the canonical architecture. Treat `asr-transcribe-masking-service` as an early lived implementation to learn from, especially for how agents actually run in production-like code, but normalize conflicting choices back toward ALMS.

For detailed conventions and templates, read [references/alms-patterns.md](references/alms-patterns.md) when adding or changing code.

## Core Workflow

1. Inspect the existing repo shape before writing code:
   - `src/api/endpoints/v1/`
   - `src/execution/usecases/`
   - `src/execution/actions/`
   - `src/agents/agent_manager/`
   - `src/agents/prompts/`
   - `src/agents/schemas/`
   - `src/agents/tools/`
   - `src/agents/workflows/`
   - `src/providers/ai/`
   - `src/config/`

2. Add features layer by layer:
   - Endpoint: validate HTTP input, inject the usecase, return clean dictionaries or the repo response wrapper.
   - Usecase: orchestrate chunking, retries, batching, semaphores, and multiple actions.
   - Action: lazily build and cache the compiled workflow, adapt inputs, call `ainvoke`, normalize failures.
   - Workflow: build `StateGraph`, add nodes, route with edges or `Send`, compile.
   - Nodes: create `SystemMessage` from `PromptManager`, create `HumanMessage` with clear input sections, call an `AgentManager` property, return partial state.
   - Agents and schemas: keep Pydantic structured outputs in `src/agents/schemas/types.py`; register them in `AgentManager`.
   - Prompts: store system prompts as markdown files under `src/agents/prompts/agents/`.

3. Preserve ALMS dependency flow:
   - API can depend on Execution.
   - Execution can depend on Agents and Utils.
   - Agents can depend on Models/Providers, Config, and Utils.
   - Do not make endpoints call LangGraph, model loaders, or prompt files directly.

## Naming Style

Prefer the repo's explicit process names over generic abstractions:

- Endpoint file: `process_<thing>.py`
- Usecase file/class: `process_<thing>_usecase.py`, `Process<Thing>UseCase`
- Action file/class: `process_<thing>_action.py`, `Process<Thing>Action`
- Workflow builder: `build_<thing>_workflow`
- Node function: `llm_call_<agent_or_step>`
- State schema: `<Thing>State`
- Result schema: `<Thing>Result`
- Prompt file: `agent_<thing>.md`
- PromptManager property: `<thing>`
- AgentManager registry key: `<thing>`

## LangGraph Defaults

Use `StateGraph` when the workflow has explicit state, routing, retries, fan-out, or multiple LLM steps. Use simple `create_react_agent` only for small single-agent routes.

Keep state schemas readable with `TypedDict` for graph state and Pydantic `BaseModel` for structured LLM output. When parallel workers aggregate results, use `Annotated[list, operator.add]`.

Use `Send("node_name", custom_state)` for map-style fan-out. Use conditional edges when routing only chooses the next node. Use `Command` only when a node must both update state and route.

Compile workflows in `build.py`, but invoke them from action classes. Cache the compiled graph on the action instance with `self._workflow`.

## LangChain Defaults

Use a `LangchainModelLoader` to centralize model configuration from `settings`. Keep provider details out of node functions.

Use `AgentManager` to lazy-load and cache agents. For normal structured output, prefer:

```python
self.model.with_structured_output(OutputSchema)
```

For tools plus structured output, use:

```python
create_agent(
    model=self.model,
    tools=tools,
    response_format=ToolStrategy(OutputSchema, handle_errors=True),
)
```

## Run And Verify

Prefer the repo's existing commands. Common ALMS commands are:

```bash
uv sync
uv run python -m src.api.main
uv run pytest src/tests
uv run ruff check src
uv run ruff format src
```

For narrow work, run the closest action/usecase/endpoint tests. If no tests exist, add a focused test for the layer touched or at least run import-level checks.
