# alms-langgraph-agent

ALMS-style LangGraph and LangChain agent builder skill for AI coding agents.

This skill captures KJ''s optimized ALMS architecture for agentic FastAPI services. It uses `alms` as the canonical project structure and extracts reusable production patterns from real ALMS implementations without hardcoding one source repo or domain.

## Install

```bash
npx skills add KJ-AIML/alms-langgraph-agent-skill
```

## Compatibility

- Skill version: `0.4.0`
- Compatible with: `alms >=0.3.0`

Use this public skill for LangGraph/LangChain agent workflows. Use the bundled `.agents/skills/alms-dev` skill from the `alms` repo for normal backend changes such as endpoints, usecases, actions, repositories, providers, settings, middleware, tests, and docs.

## Profile Support (new in 0.4.0)

This skill is profile-aware. It reads `[tool.alms]` from the target project''s `pyproject.toml` and constrains code generation to the active capabilities.

Supported profiles:

| Profile | Capabilities |
|---|---|
| `core-api` | FastAPI, Pydantic, usecases, actions, health, tests |
| `llm-agent` | core-api + LangChain, OpenAI, PromptManager, AgentManager, Scalar docs |
| `workflow-agent` | llm-agent + LangGraph workflows (state/nodes/build) |
| `db-agent` | core-api + SQLAlchemy, asyncpg, Alembic, repositories |
| `observable` | core-api + OpenTelemetry, Prometheus, metrics endpoint |
| `full` | all capabilities (current v0.3 behaviour) |

The skill will never add imports, folders, routers, or tests for capabilities that are not enabled.

## What It Does

- Builds LangGraph/LangChain agent services in the ALMS style
- Keeps FastAPI endpoints thin and pushes orchestration into usecases
- Uses actions to lazily build and invoke compiled LangGraph workflows
- Places agent code under `src/agents/` with managers, prompts, schemas, tools, and feature-scoped workflows
- Uses markdown prompt files through a `PromptManager`
- Uses structured Pydantic outputs through an `AgentManager`
- Preserves KJ''s naming style for `process_*`, `build_*_workflow`, workflow nodes, and action/usecase layers
- Adds production guardrails: ledgers, exact approved memory, deterministic safe rules, retrieval-backed reasoning, coverage retry, conflict checks, human review, summary reconciliation, and status/display APIs

## Architecture Preference

Treat `alms` as the optimized source of truth. For simple tasks, a short chain is enough:

```text
API Endpoint -> UseCase -> Action -> Structured Agent
```

For production workflows that require state, auditability, long-running jobs, or multi-step reasoning:

```text
API Endpoint -> UseCase -> Action -> LangGraph Workflow -> Agent Manager / Prompt Manager / Tools
```

See `SKILL.md` "Do Not Overbuild" for the full decision table on when to use each path.

Extract these production implementation patterns when the target problem needs them:

- background mapping jobs
- feature-scoped workflow package: `state.py`, `nodes.py`, `build.py`
- `AgentManager` and `PromptManager`
- structured output schemas
- PageIndex/retrieval-backed reasoning
- approved-memory lookup before LLM reasoning
- safe DSL rule runner before memory and LLM reasoning
- row coverage validation and held state
- deterministic plus optional LLM conflict checks
- summary aggregation and result reconciliation
- human review queue and review decision APIs
- dashboard/status/display/SSE endpoints

When `alms` and a production repo disagree, prefer the target repo''s current shape for low-churn maintenance, and prefer `alms` for new projects.

If the target ALMS repo does not yet contain `src/agents/prompts/prompt_manager.py`, `src/agents/prompts/agents/`, `src/agents/schemas/`, or feature-scoped workflow folders, create that skeleton before adding production agent behavior **only if the `llm` capability is enabled**.

## Project Structure

The skill expects and reinforces the ALMS layer structure. The exact files depend on the active profile (see `SKILL.md` "Profile / Capability Contract").

```text
src/
  api/
    endpoints/
      v1/
        <feature>.py
        schemas/
  execution/
    actions/
      process_<thing>_action.py
    usecases/
      process_<thing>_usecase.py
  agents/              # only when llm is enabled
    agent_manager/
      agent.py
    prompts/
      prompt_manager.py
      agents/
        agent_<thing>.md
    schemas/
      <feature>.py
    tools/              # only when langgraph is enabled
      <feature>_tool.py
    workflows/          # only when langgraph is enabled
      <feature>/
        state.py
        nodes.py
        build.py
  providers/
    ai/                 # only when llm is enabled
      base.py
      factory.py
      langchain_model_loader.py
  database/             # only when database is enabled
  observability/        # only when observability is enabled
  config/
```

## Usage Examples

```text
User: "Add an auditable document classification workflow in this ALMS repo"
-> Checks [tool.alms] for langgraph capability, creates schemas, prompts, AgentManager registry, tools, workflow state/nodes/build, action, usecase, endpoints, status/display APIs, and tests.
```

```text
User: "Refactor this LangGraph agent to follow my ALMS style"
-> Moves orchestration into usecases/actions, centralizes prompts and model loading, and keeps endpoints thin.
```

```text
User: "Add a simple health endpoint in this core-api project"
-> Creates a plain FastAPI endpoint with usecase/action. No agents or AI imports.
```

## Files

- `SKILL.md` - main skill instructions, profile contract, and trigger metadata
- `references/alms-patterns.md` - detailed implementation patterns with capability gates
- `agents/openai.yaml` - optional UI metadata for compatible agents
- `CHANGELOG.md` - compatibility and release notes

## License

MIT