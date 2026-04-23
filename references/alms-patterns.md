# ALMS LangGraph Agent Patterns

These conventions are anchored in `alms`, which is the optimized target architecture. `asr-transcribe-masking-service` is useful as the first lived implementation of the `/agents` layer: learn from its working agent manager, prompt manager, schemas, workflow builders, nodes, and action/usecase invocation patterns, but do not copy early-version conflicts as rules.

When `alms` and ASR disagree, prefer `alms` unless the current project already follows the ASR version and changing it would create unnecessary churn. Use current LangGraph/LangChain best practices only when they fit the ALMS shape.

## Project Shape

Use this structure for agentic FastAPI services:

```text
src/
  api/
    main.py
    endpoints/
      router/routers.py
      v1/
        routers.py
        process_<thing>.py
  execution/
    actions/
      process_<thing>_action.py
    usecases/
      process_<thing>_usecase.py
  agents/
    agent_manager/
      agent.py
      agent_manager.py
    prompts/
      prompt_manager.py
      sample_agent_prompt.py
      agents/
        agent_<thing>.md
    schemas/
      types.py
    tools/
      tools.py
    workflows/
      build.py
      nodes.py
  models/ or providers/ai/
    langchain_model_loader.py
  config/
    settings.py
    logs_config.py
  utils/
```

Prefer the ALMS provider boundary for new or refactored projects: `src/providers/ai/langchain_model_loader.py`. ASR kept `src/models/langchain_model_loader.py` because it grew from an earlier version with heavy ASR model code in `src/models/`; follow that location only when maintaining a repo that already uses it.

## Feature Recipe

To add a new agent workflow called `<thing>`:

1. Add output and state schemas in `src/agents/schemas/types.py`.
2. Add a prompt markdown file at `src/agents/prompts/agents/agent_<thing>.md`.
3. Add a lazy property to `PromptManager`.
4. Register `<thing>` in `AgentManager._agent_names`.
5. Add tools in `AgentManager._agent_tools` only if the agent needs tools.
6. Add `llm_call_<thing>` in `src/agents/workflows/nodes.py`.
7. Add `build_<thing>_workflow` in `src/agents/workflows/build.py`.
8. Add `Process<Thing>Action` that lazily compiles and invokes the workflow.
9. Add `Process<Thing>UseCase` for batching, chunking, retries, aggregation, and business decisions.
10. Add `process_<thing>.py` endpoint with FastAPI dependency injection.
11. Register the endpoint in `src/api/endpoints/v1/routers.py`.
12. Add tests near the layer touched, usually under `src/tests/`.

## Endpoint Pattern

Endpoints should be thin and boring. They own HTTP shape, request models, dependency injection, logging, and error response shape.

```python
from fastapi import APIRouter, Depends, status
from pydantic import BaseModel

from src.config.logs_config import get_logger
from src.execution.actions.process_thing_action import ProcessThingAction
from src.execution.usecases.process_thing_usecase import ProcessThingUseCase

router = APIRouter()
logger = get_logger(__name__)


class ThingPayload(BaseModel):
    payload: dict


async def get_process_thing_usecase() -> ProcessThingUseCase:
    return ProcessThingUseCase(ProcessThingAction())


@router.post("/process_thing", status_code=status.HTTP_200_OK)
async def process_thing_endpoint(
    payload: ThingPayload,
    usecase: ProcessThingUseCase = Depends(get_process_thing_usecase),
):
    logger.info("Received thing processing request")
    try:
        return await usecase.execute(payload.payload)
    except Exception as e:
        logger.error(f"Thing processing failed: {e}")
        return {"status": "failed", "error": str(e)}
```

## Usecase Pattern

Usecases orchestrate the business process. This is where chunking, deduplication, retry loops, `asyncio.gather`, semaphores, and cross-action decisions belong.

```python
import asyncio
from typing import Any, Dict

from src.config.logs_config import get_logger
from src.execution.actions.process_thing_action import ProcessThingAction

logger = get_logger(__name__)


class ProcessThingUseCase:
    def __init__(self, action: ProcessThingAction):
        self.action = action

    async def execute(self, input_data: Dict[str, Any]) -> Dict[str, Any]:
        logger.info("Starting thing processing")
        chunks = self._make_chunks(input_data)
        semaphore = asyncio.Semaphore(5)

        async def process_single_chunk(chunk: Dict[str, Any]) -> Dict[str, Any]:
            async with semaphore:
                retries = 1
                base_delay = 3.0
                for attempt in range(retries + 1):
                    try:
                        return await self.action.execute(chunk)
                    except Exception as e:
                        if attempt < retries:
                            await asyncio.sleep(base_delay * (2**attempt))
                        else:
                            logger.error(f"Chunk failed: {e}")
                            return {"status": "failed", "error": str(e)}

        results = await asyncio.gather(*(process_single_chunk(c) for c in chunks))
        return {
            "status": "success",
            "total_chunks": len(results),
            "processed_chunks": results,
        }

    def _make_chunks(self, input_data: Dict[str, Any]) -> list[Dict[str, Any]]:
        return input_data.get("chunks", [input_data])
```

## Action Pattern

Actions execute one discrete operation. For LangGraph work, an action lazily builds the workflow and normalizes success/failure.

```python
from typing import Any, Dict

from src.agents.workflows.build import build_thing_workflow
from src.config.logs_config import get_logger

logger = get_logger(__name__)


class ProcessThingAction:
    def __init__(self):
        self._workflow = None

    async def execute(self, chunk_data: Dict[str, Any]) -> Dict[str, Any]:
        chunk_id = chunk_data.get("metadata", {}).get("chunk_index", 0)
        logger.info(f"Processing chunk {chunk_id} through thing workflow")

        try:
            if self._workflow is None:
                logger.debug("Building thing workflow for first time")
                self._workflow = build_thing_workflow()

            result = await self._workflow.ainvoke(
                {
                    "chunk_data": chunk_data,
                    "thing_results": {},
                }
            )

            return {
                "chunk_id": chunk_id,
                "status": "success",
                "thing_results": result.get("thing_results", []),
            }
        except Exception as e:
            logger.error(f"Thing workflow failed for chunk {chunk_id}: {e}")
            return {
                "chunk_id": chunk_id,
                "status": "failed",
                "error": str(e),
                "thing_results": [],
            }
```

## AgentManager Pattern

Keep model creation and structured agents out of nodes. Use one manager that lazy-loads the model and caches agents.

```python
from typing import Any, Dict, Optional

from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy

from src.agents.schemas.types import ThingResult
from src.config.logs_config import get_logger
from src.models.langchain_model_loader import LangchainModelLoader

logger = get_logger(__name__)


class AgentManager:
    def __init__(self, model_loader: Optional[LangchainModelLoader] = None):
        self.model_loader = model_loader or LangchainModelLoader()
        self._model = None
        self._agents: Dict[str, Any] = {}
        self._agent_names = {"thing": ThingResult}
        self._agent_tools = {"thing": []}

    @property
    def model(self):
        if self._model is None:
            self._model = self.model_loader.init_chat_model_inference_server(
                temperature=0.2
            )
            logger.info("Model initialized successfully")
        return self._model

    def get_agent(self, name: str) -> Optional[Any]:
        if name not in self._agents:
            schema = self._agent_names.get(name)
            if schema is None:
                logger.warning(f"Unknown agent name: {name}")
                return None
            self._agents[name] = self.model.with_structured_output(schema)
        return self._agents.get(name)

    def get_agent_with_tools(self, name: str) -> Optional[Any]:
        if name not in self._agents:
            schema = self._agent_names.get(name)
            if schema is None:
                logger.warning(f"Unknown agent name: {name}")
                return None
            self._agents[name] = create_agent(
                model=self.model,
                tools=self._agent_tools[name],
                response_format=ToolStrategy(schema, handle_errors=True),
            )
        return self._agents.get(name)

    @property
    def thing(self):
        return self.get_agent("thing")
```

## PromptManager Pattern

Store prompts as markdown files and lazy-load them with properties. This keeps prompts editable without touching node code.

```python
from pathlib import Path
from typing import Optional

from src.config.logs_config import get_logger
from src.utils.file.markdown_utils import read_markdown_file_with_dedent

logger = get_logger(__name__)


class PromptManager:
    def __init__(self, prompt_base_path: Optional[str] = None):
        self.prompt_base_path = (
            Path(prompt_base_path) if prompt_base_path else Path("src/agents/prompts")
        )
        self._thing: Optional[str] = None

    @property
    def thing(self) -> str:
        if self._thing is None:
            self._thing = self._load_prompt("agents/agent_thing.md")
            logger.debug("Thing prompt loaded")
        return self._thing

    def _load_prompt(self, filename: str) -> str:
        try:
            return read_markdown_file_with_dedent(str(self.prompt_base_path / filename))
        except Exception as e:
            logger.error(f"Failed to load prompt '{filename}': {e}")
            return f"Error: Could not load prompt file '{filename}'"
```

## Schemas Pattern

Use `TypedDict` for graph state and Pydantic `BaseModel` for LLM outputs. Use reducers on list fields that collect parallel worker results.

```python
import operator
from typing import Annotated, Any, Dict, List
from typing_extensions import TypedDict, Literal

from pydantic import BaseModel, Field


class ThingState(TypedDict):
    chunk_data: Dict[str, Any]
    thing_results: Annotated[list, operator.add]


class ThingResult(BaseModel):
    reasoning: str = Field(..., description="Step-by-step analysis")
    status: Literal["PASS", "FAIL"] = Field(..., description="Decision status")
    items: List[Dict[str, Any]] = Field(default_factory=list)
```

## Node Pattern

Nodes should format the LLM call, invoke the manager, and return partial state. Keep heavy orchestration in usecases and pure transformation helpers in `utils/`.

```python
from langchain_core.messages import HumanMessage, SystemMessage

from src.agents.agent_manager.agent_manager import AgentManager
from src.agents.prompts.prompt_manager import PromptManager
from src.agents.schemas.types import ThingState
from src.config.logs_config import get_logger

agent_manager = AgentManager()
prompt_manager = PromptManager()
logger = get_logger(__name__)


async def llm_call_thing(state: ThingState):
    logger.info("=== Processing thing node ===")

    chunk_data = state.get("chunk_data", {})
    messages = [
        SystemMessage(content=prompt_manager.thing),
        HumanMessage(
            content=f"""
            Please analyze the input below.

            ### Input:
            {chunk_data}

            Instruction: Return the result in the requested schema.
            """
        ),
    ]

    response = await agent_manager.thing.ainvoke(messages)
    logger.info("=== Thing Node Success ===")
    return {"thing_results": [response.model_dump()]}
```

## Workflow Pattern

Keep graph factories in `src/agents/workflows/build.py`. Compile and return the workflow; do not invoke it here.

```python
from typing import Any

from langgraph.graph import END, START, StateGraph
from langgraph.types import RetryPolicy

from src.agents.schemas.types import ThingState
from src.agents.workflows.nodes import llm_call_thing
from src.config.logs_config import get_logger

logger = get_logger(__name__)


def build_thing_workflow() -> Any:
    logger.info("Building thing workflow...")

    builder = StateGraph(ThingState)
    builder.add_node(
        "thing",
        llm_call_thing,
        retry_policy=RetryPolicy(max_attempts=1),
    )
    builder.add_edge(START, "thing")
    builder.add_edge("thing", END)

    workflow = builder.compile()
    logger.info("Workflow compiled successfully")
    return workflow
```

## Fan-Out Worker Pattern

Use `Send` when one classification node creates dynamic work for one or more workers. Add `Annotated[list, operator.add]` to the receiving state key.

```python
from langgraph.types import Send


async def assign_workers(state: ThingState):
    sends = []
    for item in state.get("classified_items", []):
        if item.get("needs_worker"):
            sends.append(
                Send(
                    "thing_worker",
                    {
                        "worker_name": item["worker_name"],
                        "item": item,
                    },
                )
            )
    return sends
```

## Model Loader Pattern

Centralize environment setup and model initialization. Keep `settings` as the source of truth.

```python
import os
from typing import Any, Optional

from langchain.chat_models import init_chat_model
from langchain_openai import ChatOpenAI

from src.config.settings import settings


class LangchainModelLoader:
    def __init__(self):
        self.models = {}
        self._setup_api_keys()

    def _setup_api_keys(self):
        if settings.OPENAI_API_KEY:
            os.environ["OPENAI_API_KEY"] = settings.OPENAI_API_KEY

    def init_model_openai_basic(self, temperature: float = 0.0, **kwargs) -> Any:
        model = init_chat_model(
            model=settings.OPENAI_MODEL_BASIC,
            temperature=temperature,
            api_key=settings.OPENAI_API_KEY,
            **kwargs,
        )
        self.models["openai_basic"] = model
        return model

    def init_chat_model_inference_server(
        self, temperature: float = 0.0, **kwargs
    ) -> Any:
        params = {
            "model": settings.INFERENCE_SERVER_MODEL_BASIC,
            "temperature": temperature,
            "top_p": kwargs.get("top_p", 0.90),
            "openai_api_base": settings.INFERENCE_SERVER_URL,
            "max_retries": 0,
            "timeout": 1200,
        }
        if settings.INFERENCE_SERVER_API_KEY:
            params["openai_api_key"] = settings.INFERENCE_SERVER_API_KEY
        model = ChatOpenAI(**params)
        self.models["inference_server"] = model
        return model

    def get_model(self, model_name: str) -> Optional[Any]:
        return self.models.get(model_name)
```

## Scalable LangGraph Additions

Use these additions when they fit the repo:

- Use `input_schema` and `output_schema` on `StateGraph` for workflows with a public graph contract.
- Use `context_schema` for immutable runtime dependencies like tenant id, provider name, or request metadata.
- Use `Command` when a node must update state and route in the same return value.
- Add explicit termination conditions for loops and pass a `recursion_limit` at invocation when needed.
- Keep graph-level retry small for LLM nodes; put business retries in usecases where results can be normalized.
- Prefer tool-backed agents only when the tool has real deterministic value. Otherwise, structured output is simpler and easier to test.

## Public Repo Notes

For a public skill or reusable starter:

- Keep generated app code free of private endpoints, API keys, and internal hostnames.
- Put domain prompts in markdown files that users can swap.
- Keep examples domain-neutral unless the user asks for ASR, PII, or Thai transcript workflows.
- Document run commands using `uv`, because both source repos use `pyproject.toml` and `uv.lock`.
- Include minimal tests for importability and workflow construction, then add endpoint/usecase tests when behavior is stable.
