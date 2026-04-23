# alms-langgraph-agent

ALMS-style LangGraph and LangChain agent builder skill for AI coding agents.

This skill captures KJ's optimized ALMS architecture for agentic FastAPI services. It uses `alms` as the canonical project structure and uses `asr-transcribe-masking-service` as lived implementation evidence for how the `/agents` layer works in practice.

## Install

```bash
npx skills add KJ-AIML/alms-langgraph-agent-skill
```

## What It Does

- Builds LangGraph/LangChain agent services in the ALMS style
- Keeps FastAPI endpoints thin and pushes orchestration into usecases
- Uses actions to lazily build and invoke compiled LangGraph workflows
- Places agent code under `src/agents/` with managers, prompts, schemas, tools, and workflows
- Uses markdown prompt files through a `PromptManager`
- Uses structured Pydantic outputs through an `AgentManager`
- Preserves KJ's naming style for `process_*`, `build_*_workflow`, and `llm_call_*`
- Adds scalable LangGraph guidance only when it fits the ALMS shape

## Architecture Preference

Treat `alms` as the optimized source of truth:

```text
API Endpoint -> UseCase -> Action -> LangGraph Workflow -> Agent Manager / Prompt Manager
```

Treat `asr-transcribe-masking-service` as an early but working implementation to learn from:

- `AgentManager`
- `PromptManager`
- `StateGraph` builders
- `nodes.py`
- structured schemas in `types.py`
- action/usecase invocation patterns
- batch, retry, and fan-out workflow examples

When `alms` and ASR disagree, prefer `alms` for new projects.

## Project Structure

The skill expects and reinforces this structure:

```text
src/
  api/
    endpoints/
      v1/
        process_<thing>.py
  execution/
    actions/
      process_<thing>_action.py
    usecases/
      process_<thing>_usecase.py
  agents/
    agent_manager/
      agent_manager.py
    prompts/
      prompt_manager.py
      agents/
        agent_<thing>.md
    schemas/
      types.py
    tools/
      tools.py
    workflows/
      build.py
      nodes.py
  providers/
    ai/
      langchain_model_loader.py
  config/
```

## Usage Examples

```text
User: "Add a transcript masking workflow in this ALMS repo"
-> Creates schemas, prompts, AgentManager registry, nodes, workflow builder, action, usecase, endpoint, and tests.
```

```text
User: "Refactor this LangGraph agent to follow my ALMS style"
-> Moves orchestration into usecases/actions, centralizes prompts and model loading, and keeps endpoints thin.
```

```text
User: "Create a structured-output agent with tools"
-> Uses AgentManager with ToolStrategy, PromptManager markdown prompts, Pydantic result schemas, and ALMS naming.
```

## Files

- `SKILL.md` - main skill instructions and trigger metadata
- `references/alms-patterns.md` - detailed implementation patterns and code templates
- `agents/openai.yaml` - optional UI metadata for compatible agents

## License

MIT
