# ALMS LangGraph Agent Patterns

These conventions are anchored in `alms`, which is the optimized target architecture. They also extract reusable lessons from production ALMS implementations: durable jobs, audit trails, deterministic fast paths, approved memory, retrieval-backed reasoning, human review, and rule hardening.

When the target repo already has a local pattern, follow it unless changing it clearly improves correctness. Use `alms` for the base structure and use production evidence for guardrails, not for copying repo-specific names or domain concepts.

Do not turn one source repo's domain concepts into generic rules. Use real implementations as examples of the deeper mechanism, then adapt names, paths, prompts, schemas, endpoints, and tests to the target repo.

## Do Not Overbuild

Before adding workflow state, ledger, memory, or review, check which row the task falls under:

| Task Type | Pattern | Ledger | Memory | Human Review |
|---|---|:---:|:---:|:---:|
| Simple Q&A | Endpoint → Action → Agent | — | — | — |
| One-step extraction | Endpoint → Action → Structured Agent | maybe | — | — |
| Batch or long-running | Endpoint → Action → LangGraph Workflow | yes | maybe | maybe |
| Auditable production decision | Full Production Workflow | yes | yes | yes |

The production feature recipe below applies to the full production workflow. Scale back for simpler tasks by removing what the table row does not require.

## Mental Model

Use this direction:

```text
API endpoint
-> Execution usecase
-> Execution action
-> LangGraph workflow
-> Nodes
-> AgentManager / PromptManager / Tools / Providers
```

For simple work, the chain can be short. For production decisions, add a reliability ladder around the model:

```text
preprocess ledger
-> code rule check
-> approved memory check
-> retrieval-backed LLM reasoning
-> coverage retry / held state
-> conflict check
-> summary and reconciliation
-> human review queue
-> final output write
```

The central lesson is that the LLM is one decision step inside an auditable system, not the system itself.

## Project Shape

Use this structure for agentic FastAPI services:

```text
src/
  api/
    main.py
    endpoints/
      v1/
        dependencies.py
        routers.py
        schemas/
        <feature>.py
  execution/
    actions/
      process_<thing>_action.py
    usecases/
      process_<thing>_usecase.py
  agents/
    agent_manager/
      agent.py or agent_manager.py
    prompts/
      prompt_manager.py
      agents/
        agent_<thing>.md
    schemas/
      <feature>.py
    tools/
      <feature>_tool.py
    workflows/
      <feature>/
        state.py
        nodes.py
        build.py
  providers/
    ai/
      base.py
      factory.py
      langchain_model_loader.py
  database/
    connection.py
    models/
    repositories/
  config/
    settings.py
    logs_config.py
  core/
  utils/
  tests/
```

For small starters, a flat `src/agents/workflows/build.py` and `nodes.py` is acceptable. For real production workflows, use a feature folder like `src/agents/workflows/mapping/` so state, nodes, and graph wiring stay navigable.

Prefer the ALMS provider boundary for new or refactored projects: use `get_ai_provider()` from `src/providers/ai/factory.py`. This returns a configured `AIModelProvider`. Inject models through `provider.get_chat_model(tier)` rather than instantiating `LangchainModelLoader` directly. This keeps nodes and agents vendor-neutral — provider selection stays in settings, not in agent code.

If a starter repo is older than `alms` 0.2.1 and lacks `src/agents/prompts/prompt_manager.py`, `src/agents/prompts/agents/`, `src/agents/schemas/`, or feature-scoped workflow folders, add those skeleton directories before implementing production behavior.

## Production Feature Recipe

To add a production agent workflow called `<thing>`:

1. Inspect existing docs, rules, entrypoints, and layer patterns; do not assume a fixed documentation filename exists.
2. Define endpoint request/response schemas in `src/api/endpoints/v1/schemas/<thing>.py` or locally for tiny endpoints.
3. Add an endpoint in `src/api/endpoints/v1/<thing>.py`.
4. Add or reuse dependencies in `src/api/endpoints/v1/dependencies.py`.
5. Add a usecase that owns job lifecycle, orchestration, status/display payloads, and failure recording.
6. Add an action that lazily builds and invokes the compiled workflow.
7. Add a workflow package: `src/agents/workflows/<thing>/state.py`, `nodes.py`, `build.py`.
8. Add Pydantic structured output schemas under `src/agents/schemas/<thing>.py`.
9. Add prompt markdown files under `src/agents/prompts/agents/`.
10. Register agents in `AgentManager` and prompts in `PromptManager`.
11. Add tools for deterministic lookups, retrieval, memory, review queues, or rule runners.
12. Register routers in `src/api/endpoints/v1/routers.py`.
13. Add tests for endpoint, usecase/action, workflow construction, and critical node behavior.
14. Update docs when public API shape, setup, architecture, or workflow behavior changes.

## Endpoint Pattern

Endpoints should own HTTP shape only: request validation, dependency injection, background task enqueueing, response envelope, and route docs. They should not call LangGraph, tools, repositories, or model loaders directly.

For long-running jobs, prefer:

```python
@router.post("/thing/process", response_model=AppResponse)
async def process_thing(
    payload: ThingProcessRequest,
    background_tasks: BackgroundTasks,
    session: AsyncSession = Depends(get_db_session),
):
    usecase = ProcessThingUseCase()
    input_data = payload.model_dump()
    result = await usecase.create_job(session, input_data)
    background_tasks.add_task(usecase.process_job, result["job_id"], input_data)
    return AppResponse(success=True, data=result)
```

Add status and display endpoints when results are large or frontend-facing:

```text
POST /api/v1/<feature>/process
GET  /api/v1/<feature>/status/{job_id}
GET  /api/v1/<feature>/status/{job_id}/display
```

Use a display endpoint to convert raw workflow output into frontend-friendly rows, summaries, totals, conflicts, held items, audit paths, and review status.

## Usecase Pattern

Usecases orchestrate business flow. In production job workflows they also own:

- job creation
- status transitions
- background execution
- final status mapping from workflow output
- progress fields
- display payloads
- retry/delete/list/stats APIs when needed
- clean failure commits

Pattern:

```python
class ProcessThingUseCase:
    def __init__(self, action: ProcessThingAction | None = None):
        self.action = action or ProcessThingAction()

    async def create_job(self, session: AsyncSession, payload: dict) -> dict:
        job = ThingJob(status="pending", input_data=payload)
        session.add(job)
        await session.commit()
        await session.refresh(job)
        return {"job_id": str(job.id), "status": "pending"}

    async def process_job(self, job_id: str, input_data: dict) -> None:
        async with async_session() as session:
            job = await self._get_job(session, job_id)
            job.status = "processing"
            await session.commit()

            try:
                result = await self.action.execute(job_id, input_data)
                final = result.get("mapping") or result.get("result") or {}
                job.status = final.get("status", "completed")
                job.result = result
                await session.commit()
            except Exception as exc:
                job.status = "failed"
                job.error = str(exc)
                await session.commit()
                # Do not re-raise after the job is recorded as failed.
```

Usecases should read like the business process. Move implementation details into actions, tools, repositories, providers, or utilities.

## Action Pattern

Actions execute one discrete operation. For LangGraph work, an action lazily builds and caches the workflow, then normalizes the return shape.

```python
class ProcessThingAction:
    def __init__(self):
        self._workflow = None

    @property
    def workflow(self):
        if self._workflow is None:
            self._workflow = build_thing_workflow()
        return self._workflow

    async def execute(self, job_id: str, input_data: dict) -> dict:
        state = await self.workflow.ainvoke(
            {
                "job_id": str(job_id),
                "input_data": input_data,
            }
        )
        return {
            "job_id": str(job_id),
            "message": "Thing workflow completed",
            "result": state["final_output"],
        }
```

Actions should not know HTTP response details.

## Workflow State Pattern

Use `TypedDict` for workflow state. Keep it explicit enough that a future developer can see the system's audit contract.

```python
class ThingWorkflowState(TypedDict, total=False):
    job_id: str
    input_data: dict[str, Any]
    preprocess: dict[str, Any]
    ledger: dict[str, Any]
    code_rule_report: dict[str, Any]
    approved_memory_report: dict[str, Any]
    row_mappings: list[dict[str, Any]]
    coverage_report: dict[str, Any]
    conflict_report: dict[str, Any]
    summary: list[dict[str, Any]]
    totals: dict[str, Any]
    human_review_report: dict[str, Any]
    final_output: dict[str, Any]
    errors: list[dict[str, Any]]
```

Prefer state keys that match business reports, not internal helper names.

## Workflow Build Pattern

Keep graph factories in `src/agents/workflows/<feature>/build.py`. Compile and return the workflow; do not invoke it here.

```python
def build_thing_workflow():
    workflow = StateGraph(ThingWorkflowState)
    workflow.add_node("preprocess", preprocess)
    workflow.add_node("code_rule_check", code_rule_check)
    workflow.add_node("approved_memory_check", approved_memory_check)
    workflow.add_node("reasoning", reasoning)
    workflow.add_node("coverage_validator", coverage_validator)
    workflow.add_node("conflict_check", conflict_check)
    workflow.add_node("summary", summary)
    workflow.add_node("human_review_queue", human_review_queue)
    workflow.add_node("ledger_write", ledger_write)

    workflow.add_edge(START, "preprocess")
    workflow.add_edge("preprocess", "code_rule_check")
    workflow.add_edge("code_rule_check", "approved_memory_check")
    workflow.add_edge("approved_memory_check", "reasoning")
    workflow.add_edge("reasoning", "coverage_validator")
    workflow.add_edge("coverage_validator", "conflict_check")
    workflow.add_edge("conflict_check", "summary")
    workflow.add_edge("summary", "human_review_queue")
    workflow.add_edge("human_review_queue", "ledger_write")
    workflow.add_edge("ledger_write", END)
    return workflow.compile()
```

Sequential edges are fine when nodes know how to no-op after a successful fast path. Use conditional edges when routing clarity is worth the extra graph structure.

## Ledger And Coverage Pattern

Use a ledger when each input item must be accounted for. This is the key production pattern for agent workflows where silent loss is unacceptable.

The ledger should track:

- batch or job id
- employee/user/entity grouping when applicable
- chunks or work units
- row/item ids
- raw source row
- normalized row
- status
- path used
- validation state
- missing, duplicate, extra, held, or conflicted items

Before final output:

- input count must equal accounted output count, or missing/held evidence must be explicit
- every mapped item has required category/result fields
- every mapped item has source identifiers and source units/facts preserved
- every fast path goes through the same coverage and conflict validators as the LLM path

Do not let an LLM output silently replace source fields such as source category, source units, source identifiers, dates, or row ids. Anchor those fields from the normalized input and only use the model for classification, reasoning, confidence, and references.

## Node Pattern

Nodes should perform one workflow step and return partial state. They may call tools, structured agents, or pure helpers, but they should not contain API or usecase logic.

LLM node shape:

```python
async def llm_call_thing(state: ThingWorkflowState) -> ThingWorkflowState:
    context = retrieve_context(state)
    messages = [
        SystemMessage(content=prompt_manager.thing),
        HumanMessage(content=build_human_message(state, context)),
    ]
    response = await agent_manager.thing.ainvoke(messages)
    return {"thing_results": [response.model_dump()]}
```

Production node rules:

- Run deterministic checks before LLM calls when a safe answer may already exist.
- Preserve audit paths such as `code_rule`, `approved_memory`, `pageindex`, or `llm`.
- On invalid or incomplete model output, retry with explicit correction context.
- After max attempts, mark the chunk/item held and send it to review; do not trust partial output.
- Keep deterministic conflict checks authoritative. Optional LLM conflict checks can add evidence, not erase deterministic conflicts.

## AgentManager Pattern

Keep model creation and structured agents out of nodes. Use one manager that lazy-loads the model and caches agents.

```python
class AgentManager:
    def __init__(self) -> None:
        self._model = None
        self._agents: dict[str, Any] = {}

    @property
    def model(self) -> Any:
        if self._model is None:
            self._model = get_llm("reasoning")
        return self._model

    def get_agent(self, name: str) -> Any:
        if name not in self._agents:
            schema = self._schema_for(name)
            self._agents[name] = self.model.with_structured_output(schema)
        return self._agents[name]

    @property
    def thing_reasoner(self) -> Any:
        return self.get_agent("thing_reasoner")
```

Use `with_structured_output` for normal schema-bound LLM steps. Use tool-aware agents only when the model must dynamically call deterministic tools; otherwise, call tools in the node and pass retrieved context into the prompt.

### Factory Compatibility

ALMS may use function-based `create_*_agent()` factories alongside `AgentManager`. Do not replace existing factories without being asked. Use `AgentManager` for new production workflows; keep `create_*_agent()` in repos that already have them.

## PromptManager Pattern

Store prompts as markdown files and lazy-load them with properties. This keeps prompts editable without touching node code.

```python
class PromptManager:
    def __init__(self, prompt_base_path: str | Path | None = None) -> None:
        self.prompt_base_path = (
            Path(prompt_base_path)
            if prompt_base_path is not None
            else Path(__file__).resolve().parent
        )
        self._thing_reasoner: str | None = None

    @property
    def thing_reasoner(self) -> str:
        if self._thing_reasoner is None:
            self._thing_reasoner = self._load_prompt("agents/agent_thing_reasoner.md")
        return self._thing_reasoner

    def _load_prompt(self, filename: str) -> str:
        return (self.prompt_base_path / filename).read_text(encoding="utf-8")
```

Prompts should state the output contract and safety boundaries. For source-anchored workflows, tell the model which fields are evidence and which fields it may decide.

## Schema Pattern

Use Pydantic for structured LLM outputs and endpoint request models. Use `TypedDict` for graph state.

For production decision workflows, define separate schemas for:

- row/item mapping result
- chunk/batch mapping result
- conflict item
- conflict check result
- rule candidate specification when rule hardening exists
- human review override requests

Keep LLM output schemas strict enough to validate behavior, but not so clever that normal responses fail because of incidental formatting.

## Tool Patterns

Tools are deterministic adapters around capabilities the workflow needs.

Approved memory tool:

- builds a stable exact signature from normalized source facts
- hashes the signature with sorted JSON
- looks up only approved records
- returns records keyed by signature hash
- blocks conflicting output fingerprints for the same signature

Use memory to reduce repeated LLM calls, not to generalize beyond proof.

Retrieval tool:

- hides PageIndex/vector/search implementation details
- returns scoped context for the node
- supports deterministic fallback for tests and smoke runs
- keeps retrieval config in `settings`

Human review queue tool:

- persists held chunks, conflicts, low-confidence rows, row mappings, summaries, totals, and evidence
- upserts by stable job/entity/review type when reruns happen
- returns a compact report for final output

Rule DSL runner:

- evaluates allowlisted JSON/DSL conditions
- emits mappings only when all conditions match
- can compare shadow output against trusted final mappings
- never executes arbitrary generated Python in production

## Approved Memory And Rule Hardening

Use this promotion path:

```text
LLM/retrieval proposes mapping
-> workflow validates coverage and conflicts
-> human accepts or overrides
-> approved memory record is created
-> repeated clean memory evidence can create a shadow rule candidate
-> shadow rule is compared against trusted outputs
-> manual activation promotes it to code_rule path
```

Guardrails:

- Do not create memory directly from unreviewed LLM output.
- Do not fuzzy-match approved memory unless the product explicitly requires it and review accepts the risk.
- Do not activate generated rules automatically just because an LLM wrote them.
- Keep active rules all-or-nothing for a chunk or make partial fallback evidence explicit.
- Run code-rule and memory outputs through the same coverage, conflict, summary, and ledger checks as the LLM path.

## Conflict And Summary Pattern

Conflict checks should catch issues the row-level model can miss:

- held chunks/items
- missing, duplicate, or extra output rows
- low confidence
- one source category mapping to multiple target categories
- missing references or required reasoning
- total/unit reconciliation mismatch
- domain-specific policy limits

Summary aggregation should group the final row/item mappings into the shape the business needs. It should also reconcile source totals against target totals and expose documented exceptions.

When conflicts or held items exist, final status should become `human_review` unless the repo has a more specific status taxonomy.

## Model Loader Pattern

Centralize model setup through the provider abstraction. In ALMS v0.3.0 the preferred entry point is `get_ai_provider()`:

```python
# src/providers/ai/factory.py
from src.providers.ai.langchain_model_loader import LangchainModelLoader

def get_ai_provider() -> AIModelProvider:
    # ponytail: single provider; add google/anthropic branches here when MODEL_PROVIDER != "openai"
    return LangchainModelLoader()
```

```python
# src/providers/ai/base.py
from abc import ABC, abstractmethod
from typing import Any

class AIModelProvider(ABC):
    @abstractmethod
    def get_chat_model(self, tier: str = "basic", **kwargs: Any) -> Any:
        """Return a configured chat model. tier: 'basic' or 'reasoning'."""
```

```python
# In agent or node code — vendor-neutral
from src.providers.ai.factory import get_ai_provider

model = get_ai_provider().get_chat_model("basic")
reasoning_model = get_ai_provider().get_chat_model("reasoning")
```

`LangchainModelLoader` implements `AIModelProvider` and supports `get_chat_model(tier)` — `"basic"` calls `init_model_openai_basic`, `"reasoning"` calls `init_model_openai_reasoning`. Call `get_ai_provider()` in lazy-loaded properties so non-AI routes do not depend on AI setup.

If the repo exposes a `get_llm("reasoning")` helper that wraps the same loader, prefer it for backward compatibility with existing nodes.

> **Google / Anthropic providers are deferred.** The `MODEL_PROVIDER` setting is the switch point in `factory.py`, but only `openai` is implemented. Do not tell the user that Google or Anthropic providers exist unless the repo already has them.

Common settings for production agent workflows:

- `AI_ENABLED` — must be `True` to activate AI routes; gates production key validation
- `DATABASE_ENABLED` — controls readiness check; set `False` to disable DB dependency
- `REDIS_ENABLED` — controls Redis dependency; set `False` if Redis is not in use
- `MODEL_PROVIDER` — provider selection (currently `openai` only)
- `OPENAI_API_KEY`
- `OPENAI_MODEL_BASIC`
- `OPENAI_MODEL_REASONING`
- `INFERENCE_SERVER_URL`
- `INFERENCE_SERVER_MODEL_BASIC`
- `INFERENCE_SERVER_MODEL_REASONING`
- workflow max attempts
- low-confidence threshold
- unit/result reconciliation tolerance
- optional LLM conflict checker enabled flag
- raw review evidence enabled flag

## Status And Display APIs

Production workflows usually need more than one endpoint:

- `process`: creates the job and starts work
- `status`: raw lifecycle and result for polling
- `display`: frontend-friendly result shape
- `events`: optional SSE stream for live progress
- `reviews`: list/detail/decision for human review
- `rules`: build/list/activate/disable for rule hardening

The display payload should be intentionally redundant and easy for a UI to render:

```text
job_id
status
mapping_status
path_summary
rows
summary
totals
conflicts
held_items
review
error
timestamps
```

## Tests And Verification

Prefer focused tests by layer:

- endpoint test: request validation, response envelope, background enqueue/status
- usecase test: job lifecycle, status mapping, clean failure behavior
- action test: workflow invocation and result normalization
- workflow test: graph compiles and minimal payload reaches final output
- node test: coverage validation, conflict detection, memory hit/miss, rule match/mismatch
- tool test: signature hashing, safe DSL matching, review queue persistence

Run:

```bash
uv run pytest src/tests
uv run pytest src/tests/v1/test_<thing>.py -v
uv run ruff check src
```

For production agent workflows, add a smoke scenario that proves:

- input count equals output count
- all outputs have source ids and required result fields
- totals reconcile
- conflicts trigger review
- failed jobs are recorded cleanly
- second run can hit approved memory or active code rules only when exact coverage exists

## Public Repo Notes

For a public skill or reusable starter:

- Keep generated app code free of private endpoints, API keys, internal hostnames, and domain-only secrets.
- Put domain prompts in markdown files users can swap.
- Keep examples domain-neutral unless the user asks for a specific domain.
- Document run commands using `uv`, because both source repos use `pyproject.toml` and `uv.lock`.
- Include minimal tests for importability and workflow construction, then add endpoint/usecase tests when behavior is stable.
