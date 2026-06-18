---
name: alms-langgraph-agent
description: Build or review ALMS Python FastAPI LangGraph/LangChain agent workflows with action/usecase orchestration, prompt managers, structured outputs, approved memory, human review loops, and production reliability guardrails.
version: 0.4.0
compatible_with: "alms >=0.3.0"
---

# ALMS LangGraph Agent

Use this skill to build agentic FastAPI services in the optimized ALMS style: API endpoints stay thin, usecases orchestrate business flow and job lifecycle, actions execute compiled workflows, and the agent layer owns prompts, structured outputs, tools, state, nodes, and LangGraph wiring.

Treat `alms` as the canonical architecture. Treat production ALMS implementations as evidence for reusable patterns: long-running jobs, item-level ledgers, deterministic fast paths, approved memory, retrieval-backed reasoning, conflict checks, human review, rule hardening, dashboard/status APIs, and clean failure handling.

Do not copy a source repo''s domain, filenames, docs paths, examples, endpoint names, or business nouns unless the target repo already uses them. Extract the mechanism, then adapt it to the target business problem.

For detailed conventions and templates, read [references/alms-patterns.md](references/alms-patterns.md) when adding or changing code.

## Profile / Capability Contract

**Always read `[tool.alms]` from `pyproject.toml` before adding or changing code.**

ALMS projects declare their active profile and capabilities in `pyproject.toml`:

```toml
[tool.alms]
profile = "core-api"
capabilities = ["runtime_auth", "tests"]
```

The skill must honour these constraints. If `[tool.alms]` is missing, infer conservatively by inspecting what already exists:

- If `src/agents/workflows/` has feature folders, treat as `workflow-agent` or `full`.
- If `src/providers/ai/` exists, treat as at least `llm-agent`.
- If `src/database/` exists, treat as at least `db-agent`.
- Otherwise treat as `core-api`.
- **Never assume `full` unless clearly indicated.**

### Profile rules

| Capability | Allowed | Forbidden without this capability |
|---|---|---|
| (always) | FastAPI endpoints, Pydantic schemas, AppResponse, usecases, actions, runtime auth, health endpoints, tests | — |
| `llm` | Simple structured agents, PromptManager, AgentManager, provider/AI model loader, sample agent endpoint | LangChain/OpenAI imports |
| `langgraph` | LangGraph workflows (state/nodes/build), workflow compile tests | `from langgraph` imports; `StateGraph` usage |
| `database` | SQLAlchemy repos, async sessions, Alembic, DB readiness in health | `from sqlalchemy` imports; `from src.database` imports |
| `redis` | Redis cache provider | `import redis` |
| `observability` | Metrics endpoint, tracing setup, observability middleware, Prometheus/OpenTelemetry | `from opentelemetry` or `from prometheus_client` imports |
| `scalar_docs` | Scalar API docs at `/docs` | `from scalar_fastapi` imports |
| `docker` | Dockerfile, docker-compose | — |
| `ci` | GitHub Actions workflows | — |

### Constraint enforcement

- **Do not add imports that require optional extras** unless the capability is present.
- **Do not create folders** under `src/agents/`, `src/providers/ai/`, `src/database/`, or `src/observability/` unless the matching capability is enabled.
- **Do not register routers** for disabled capabilities (e.g. no `/metrics` route without `observability`).
- **Do not make tests import optional systems** in `core-api` profile.
- **Do not add dependencies to `pyproject.toml`** outside the active capability set.

## Do Not Overbuild

Use the minimum architecture the task actually requires — within the project''s capability boundaries.

For a simple task in an `llm-agent` profile, a thin action calling a structured agent is enough:

```text
Endpoint -> UseCase -> Action -> Structured Agent
```

For production tasks in a `workflow-agent` or `full` profile, use the full workflow:

```text
Endpoint -> UseCase -> Action -> LangGraph Workflow -> Nodes / Tools / Agents
```

Only add the production workflow when the task involves AND the `langgraph` capability is enabled:

- batch processing or long-running jobs
- auditability or traceable decisions
- source anchoring the model must not override
- expensive recomputation worth caching in approved memory
- human review
- conflict detection
- approved memory lookup
- deterministic rule promotion
- status or display APIs
- coverage validation that catches missing or duplicate output

| Task Type | Pattern | Needs langgraph? | Ledger | Memory | Human Review | Status API |
|---|---|:---:|:---:|:---:|:---:|:---:|
| Simple Q&A | Endpoint -> UseCase -> Action -> Agent | No | — | — | — | — |
| One-step extraction | Endpoint -> UseCase -> Action -> Structured Agent | No | maybe | — | — | — |
| Batch classification | UseCase -> Action -> Workflow | Yes | yes | maybe | maybe | yes |
| Auditable decision | Production Workflow | Yes | yes | yes | yes | yes |
| Expensive repeated decision | Production Workflow + Memory | Yes | yes | yes | maybe | yes |
| Repeated proof -> safe rule | Production Workflow + Safe Rules | Yes | yes | yes | yes | yes |
| Long-running background job | Process + Status + Display APIs | Yes | yes | maybe | maybe | yes |

Do not add ledger, approved memory, conflict checks, or human review unless the task type in this table says to. Do not add LangGraph at all unless the `langgraph` capability is enabled.

## Core Workflow

1. **Read the project profile before writing any code:**
   - Inspect `pyproject.toml` for `[tool.alms]`.
   - Note the `profile` and `capabilities` list.
   - If `[tool.alms]` is absent, infer from existing files:
     - `src/agents/workflows/` -> `workflow-agent` or `full`
     - `src/providers/ai/` -> `llm-agent`
     - `src/database/` -> `db-agent`
     - else -> `core-api`
   - Then inspect the existing repo shape:
     - architecture, project-structure, guideline, README, rules, or planning docs if they exist
     - `pyproject.toml`, lockfiles, test config, and app entrypoints
     - `src/api/endpoints/v1/`
     - `src/execution/usecases/`
     - `src/execution/actions/`
     - `src/config/`
     - existing tests for the same feature or layer
   - **Only inspect folders that match enabled capabilities:**
     - `src/agents/agent_manager/` — only if `llm` is enabled
     - `src/agents/prompts/` — only if `llm` is enabled
     - `src/agents/schemas/` — only if `llm` is enabled
     - `src/agents/tools/` — only if `llm` or `langgraph` is enabled
     - `src/agents/workflows/` — only if `langgraph` is enabled
     - `src/providers/ai/` — only if `llm` is enabled
     - `src/database/` — only if `database` is enabled
     - `src/observability/` — only if `observability` is enabled

2. Choose the right workflow depth **within capability limits:**
   - For `core-api`: use plain FastAPI + usecase/action. No agents, no AI.
   - For `llm-agent` without `langgraph`: use a thin action around a structured-output agent.
   - For `workflow-agent` or `full`: use stateful workflows with ledger, deterministic fast paths, retrieval or LLM reasoning, coverage validation, conflict checks, summary/reconciliation, and human review.
   - For long-running work in any AI-capable profile: expose `process`, `status`, and display-friendly result endpoints, and use background tasks or a queue instead of blocking the request.

3. Add features layer by layer **only within enabled capabilities:**
   - Endpoint: validate HTTP input, inject the usecase, enqueue background work when needed, return `AppResponse` or the repo response wrapper.
   - Usecase: own job creation, status transitions, orchestration, retries, batching, dashboard/display payloads, and clean failure recording.
   - Action: lazily build and cache the compiled workflow (langgraph only), adapt inputs, call `ainvoke`, and normalize the workflow result.
   - Workflow: (langgraph only) build `StateGraph`, add nodes, wire edges, compile only. Keep feature-sized workflows in `src/agents/workflows/<feature>/`.
   - State: (langgraph only) define durable state in `state.py`; include input, ledger, reports, outputs, final status, and errors.
   - Nodes: (langgraph only) run deterministic tools first, call structured agents only when needed, validate coverage, return partial state, and preserve audit evidence.
   - Agents and schemas: (llm only) keep Pydantic structured outputs in feature schema modules such as `src/agents/schemas/<feature>.py`; register them in `AgentManager`.
   - Prompts: (llm only) store system prompts as markdown files under `src/agents/prompts/agents/` and lazy-load them through `PromptManager`.
   - Tools: (llm or langgraph) keep retrieval, memory lookup, review persistence, and deterministic rule runners behind tool classes.

4. Preserve ALMS dependency flow:
   - API can depend on Execution.
   - Execution can depend on Agents, Providers, Repositories, and Utils.
   - Agents can depend on Providers, Config, and Utils.
   - Tools may use repositories or sessions when they are persistence adapters.
   - Do not make endpoints call LangGraph, model loaders, prompt files, tools, or repositories directly.

## Production Reliability Ladder

(Requires `langgraph` capability.)

Use this production ALMS pattern when the output must be auditable, reviewable, or expensive to recompute:

```text
preprocess ledger
-> deterministic safe rule check
-> exact approved memory lookup
-> retrieval-backed LLM reasoning
-> row/item coverage retry or held state
-> deterministic conflict checks
-> optional LLM conflict evidence
-> summary and unit/result reconciliation
-> human review queue
-> ledger/final output write
-> approved memory after accept/override
-> safe DSL rule candidate after repeated proof
```

Important guardrails:

- No silent loss: every input row/item must be represented in ledger, coverage report, final output, held state, or review evidence.
- Source anchoring: preserve source units, source category, source identifiers, and other user/provider facts. LLMs may classify; they should not silently rewrite the source of truth.
- Exact memory first: approved memory should use stable signatures and hashes. Avoid fuzzy matching in production logic unless the user explicitly accepts that risk.
- Human review is the trust boundary: LLM output can propose decisions; accepted or overridden human decisions create approved memory.
- Deterministic rules come late: promote repeated approved memory into safe DSL or inspectable rules only after enough clean evidence.
- Safe rule execution: prefer JSON/DSL conditions over arbitrary generated Python.
- Status is sacred: job status must reflect workflow result (`completed`, `human_review`, `failed`, etc.), not a hardcoded happy path.
- Clean failure handling: after a background job is marked failed and committed, do not re-raise into the ASGI stack.
- PII minimization: review evidence should store allowlisted context by default, with raw rows behind explicit settings.

## Naming Style

Prefer the repo''s explicit business names over generic abstractions:

- Endpoint file: `<feature>.py` or `process_<thing>.py`
- Endpoint route group: `/api/v1/<feature>/...`
- Usecase file/class: `process_<thing>_usecase.py`, `Process<Thing>UseCase`
- Action file/class: `process_<thing>_action.py`, `Process<Thing>Action`
- Workflow package: `src/agents/workflows/<thing>/`
- Workflow builder: `build_<thing>_workflow`
- Workflow state: `<Thing>WorkflowState` or `<Thing>State`
- Node function: `<step_name>` for workflow steps or `llm_call_<agent_or_step>` for direct LLM nodes
- Result schema: `<Thing>Result`, `<Thing>ChunkResult`, or domain-specific Pydantic models
- Prompt file: `agent_<thing>.md`
- PromptManager property: `<thing>`
- AgentManager property: `<thing>`
- Tool class: `<Thing>Tool`, `<Thing>MemoryTool`, `<Thing>QueueTool`, `<Thing>RuleRunner`

## LangGraph Defaults

(Requires `langgraph` capability.)

Use `StateGraph` when the workflow has explicit state, routing, retries, fan-out, deterministic checks, human review, or multiple LLM steps. Use simple structured-output calls only for small single-agent routes.

Keep state schemas readable with `TypedDict` for graph state and Pydantic `BaseModel` for structured LLM output. When parallel workers aggregate results, use `Annotated[list, operator.add]`.

Use feature folders for real workflows:

```text
src/agents/workflows/<feature>/
  state.py
  nodes.py
  build.py
```

Use `Send("node_name", custom_state)` for map-style fan-out. Use conditional edges when routing chooses the next node. Use `Command` only when a node must both update state and route.

Compile workflows in `build.py`, but invoke them from action classes. Cache the compiled graph on the action instance with `self._workflow` or a `workflow` property.

Keep graph-level retry small for LLM calls. Put business retries in usecases or node helpers where failed outputs can be normalized into held/review state.

## LangChain Defaults

(Requires `llm` capability.)

Use a `LangchainModelLoader` or `get_llm()` provider helper to centralize model configuration from `settings`. Keep provider details out of node functions.

Use `AgentManager` to lazy-load and cache structured-output agents:

```python
self.model.with_structured_output(OutputSchema)
```

For tools plus structured output, use a tool-aware agent only when the tool has deterministic value. Otherwise, retrieve tool context in the node, pass it into the prompt, and keep the LLM output schema simple.

## Factory Compatibility

ALMS may use function-based agent factories alongside or instead of `AgentManager`.

```python
@lru_cache(maxsize=1)
def create_sample_agent() -> Any:
    """Build the sample agent lazily so non-agent routes do not depend on AI setup."""
    ...
```

Do not blindly replace existing factories.

- If the repo uses `create_*_agent()` factories, keep them for backward compatibility.
- Use `AgentManager` for new production structured-output workflows.
- Add `AgentManager` alongside existing factories if both styles are needed.
- Refactor old factories to `AgentManager` only when explicitly requested.

## Future Domain-Module Compatibility

ALMS currently uses a layer-first directory structure:

```text
src/api/endpoints/v1/<feature>.py
src/execution/usecases/<feature>_usecase.py
src/execution/actions/<feature>_action.py
src/agents/workflows/<feature>/
```

A future ALMS version may support a domain-module structure:

```text
src/modules/<feature>/
  api.py
  schemas.py
  usecases.py
  actions.py
  workflows/
    state.py
    nodes.py
    build.py
```

The dependency flow remains the same regardless of directory structure:

```text
API -> UseCase -> Action -> Workflow / Agent / Provider
```

When working in a repo that has not migrated, keep the layer-first structure. Do not reorganize into domain modules unless explicitly requested.

## Run And Verify

Prefer the repo''s existing commands. Common ALMS commands are:

```bash
uv sync
uv run uvicorn src.api.main:app --port 3000 --reload
uv run pytest src/tests
uv run pytest src/tests/v1 -v
uv run ruff check src
uv run ruff format src
```

### Profile-specific verification

For `core-api` — the app must start without any optional dependencies:

```bash
uv sync
uv run pytest src/tests
python -c "import src.api.main; print('core import ok')"
```

Core profile must NOT require: langchain, langgraph, sqlalchemy, asyncpg, redis, opentelemetry, prometheus-client, scalar-fastapi.

For `llm-agent`:

```bash
uv sync --extra ai --extra docs
uv run pytest src/tests
```

For `workflow-agent`:

```bash
uv sync --extra ai --extra workflow --extra docs
uv run pytest src/tests
```

For `db-agent`:

```bash
uv sync --extra db
uv run pytest src/tests
```

For `observable`:

```bash
uv sync --extra observability
uv run pytest src/tests
```

For `full` (current v0.3 behaviour):

```bash
uv sync --extra full
uv run pytest src/tests
```

### Production agent workflow verification

(Requires `langgraph` capability.)

For production agent workflows, also verify:

- Workflow compiles.
- Action can invoke the workflow with a minimal payload.
- Coverage validation catches missing, duplicate, and extra outputs.
- Deterministic fast paths still pass coverage and conflict checks.
- Human review paths persist enough evidence.
- A second run can hit approved memory or code rules only when exact coverage exists.
- Status/display APIs show `completed`, `human_review`, and `failed` clearly.

## Final Response Format

After any changes, report in this format:

```text
## Summary
What changed and why.

## Files Changed
List each file and the purpose of the change.

## Architecture Impact
How the change affects ALMS boundaries:
API, UseCase, Action, Agent/Workflow, Provider, Database, Observability, Config.

## Profile Impact
Which capabilities were used, and whether any capability boundaries were crossed or needed.

## Verification
Commands run, for example:
- uv sync
- uv run pytest src/tests
- uv run ruff check src
- uv run ruff format src

If a command was not run, say so clearly.

## Known Limitations
Incomplete work, assumptions, or follow-up risks.

## Next Recommended Step
One clear next step.
```