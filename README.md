# alms-langgraph-agent

ALMS-style LangGraph and LangChain agent builder skill for AI coding agents.

This skill captures KJ's optimized ALMS architecture for agentic FastAPI services. It uses `alms` as the canonical project structure and extracts reusable production patterns from real ALMS implementations without hardcoding one source repo or domain.

## Install

```bash
npx skills add KJ-AIML/alms-langgraph-agent-skill
```

## Compatibility

- Skill version: `0.2.1`
- Compatible with: `alms >=0.2.1`

Use this public skill for LangGraph/LangChain agent workflows. Use the bundled `.agents/skills/alms-dev` skill from the `alms` repo for normal backend changes such as endpoints, usecases, actions, repositories, providers, settings, middleware, tests, and docs.

## What It Does

- Builds LangGraph/LangChain agent services in the ALMS style
- Keeps FastAPI endpoints thin and pushes orchestration into usecases
- Uses actions to lazily build and invoke compiled LangGraph workflows
- Places agent code under `src/agents/` with managers, prompts, schemas, tools, and feature-scoped workflows
- Uses markdown prompt files through a `PromptManager`
- Uses structured Pydantic outputs through an `AgentManager`
- Preserves KJ's naming style for `process_*`, `build_*_workflow`, workflow nodes, and action/usecase layers
- Adds production guardrails: ledgers, exact approved memory, deterministic safe rules, retrieval-backed reasoning, coverage retry, conflict checks, human review, summary reconciliation, and status/display APIs

## Architecture Preference

Treat `alms` as the optimized source of truth:

```text
API Endpoint -> UseCase -> Action -> LangGraph Workflow -> Agent Manager / Prompt Manager / Tools
```

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

When `alms` and a production repo disagree, prefer the target repo's current shape for low-churn maintenance, and prefer `alms` for new projects.

If the target ALMS repo does not yet contain `src/agents/prompts/prompt_manager.py`, `src/agents/prompts/agents/`, `src/agents/schemas/`, or feature-scoped workflow folders, create that skeleton before adding production agent behavior.

## Project Structure

The skill expects and reinforces this structure:

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
  agents/
    agent_manager/
      agent.py
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
      langchain_model_loader.py
  config/
```

## Usage Examples

```text
User: "Add an auditable document classification workflow in this ALMS repo"
-> Creates schemas, prompts, AgentManager registry, tools, workflow state/nodes/build, action, usecase, endpoints, status/display APIs, and tests.
```

```text
User: "Refactor this LangGraph agent to follow my ALMS style"
-> Moves orchestration into usecases/actions, centralizes prompts and model loading, and keeps endpoints thin.
```

```text
User: "Make this agent production-safe with memory and review"
-> Adds exact approved memory, coverage validation, conflict reports, human review queue, and safe promotion path for deterministic rules.
```

## Files

- `SKILL.md` - main skill instructions and trigger metadata
- `references/alms-patterns.md` - detailed implementation patterns and code templates
- `agents/openai.yaml` - optional UI metadata for compatible agents
- `CHANGELOG.md` - compatibility and release notes

## License

MIT
