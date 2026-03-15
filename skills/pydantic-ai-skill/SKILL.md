---
name: pydantic-ai
description: >
  Expert guidance for building production-grade AI agents and workflows using Pydantic AI
  (the `pydantic_ai` Python library). Use this skill whenever the user is: writing, debugging,
  or reviewing any Pydantic AI code; asking how to build AI agents in Python with Pydantic; 
  asking about Agent, RunContext, tools, dependencies, structured outputs, streaming, multi-agent
  patterns, MCP integration, or testing with Pydantic AI; or migrating from LangChain/LlamaIndex
  to Pydantic AI. Trigger even for vague requests like "help me build an AI agent in Python" or
  "how do I add tools to my LLM app" — Pydantic AI is very likely what they need.
---

# Pydantic AI Expert Skill

You are an expert on Pydantic AI — the production-grade Python agent framework by the Pydantic team.
Always write idiomatic Pydantic AI code with full type annotations, dataclass deps, and Pydantic output models.

## Core Mental Model

Pydantic AI = **Agent** container + **Tools** (functions LLM may call) + **Dependencies** (injected context) + **Structured Output** (validated Pydantic models).

```
Agent[DepType, OutputType]
  ├── instructions / system_prompt
  ├── @agent.tool / @agent.tool_plain  →  functions LLM calls
  ├── deps_type  →  injected via RunContext[DepType]
  └── output_type  →  validated Pydantic model or primitive
```

---

## Installation

```bash
pip install pydantic-ai                # base install
pip install 'pydantic-ai[openai]'      # with OpenAI support
pip install 'pydantic-ai[anthropic]'   # with Anthropic support
pip install 'pydantic-ai[logfire]'     # with observability
```

Set your API key:
```bash
export OPENAI_API_KEY=sk-...
export ANTHROPIC_API_KEY=sk-ant-...
```

---

## 1. Agents — The Core Building Block

```python
from pydantic_ai import Agent

# Minimal agent (text output)
agent = Agent('openai:gpt-4o', instructions='Be concise.')
result = agent.run_sync('What is 2+2?')
print(result.output)  # "4"
```

### Agent Constructor Key Parameters
| Parameter | Purpose |
|-----------|---------|
| `model` | Model string: `'openai:gpt-4o'`, `'anthropic:claude-sonnet-4-6'`, etc. |
| `instructions` | Static system prompt string (preferred over `system_prompt`) |
| `deps_type` | Type of dependency object injected at runtime |
| `output_type` | Pydantic model or primitive for structured output |
| `model_settings` | Default `ModelSettings` (temperature, max_tokens, etc.) |
| `retries` | Number of output validation retries (default: 1) |

### Running Agents — 5 Ways

```python
# 1. Sync (simplest, for scripts/notebooks)
result = agent.run_sync('prompt')
print(result.output)

# 2. Async (for production async apps)
result = await agent.run('prompt')

# 3. Streaming text
async with agent.run_stream('prompt') as resp:
    async for chunk in resp.stream_text():
        print(chunk, end='')

# 4. Streaming events (tool calls visible)
async for event in agent.run_stream_events('prompt'):
    ...

# 5. Iter (full graph control)
async with agent.iter('prompt') as run:
    async for node in run:
        ...
```

---

## 2. Tools — Giving the LLM Capabilities

```python
from pydantic_ai import Agent, RunContext

agent = Agent('openai:gpt-4o', deps_type=str)

@agent.tool                          # has RunContext access (for deps)
async def search_db(ctx: RunContext[str], query: str) -> list[str]:
    """Search the database for records matching the query."""
    db_conn = ctx.deps  # access injected dependency
    return await db_conn.search(query)

@agent.tool_plain                    # no RunContext needed
def get_current_time() -> str:
    """Return the current UTC time."""
    from datetime import datetime, UTC
    return datetime.now(UTC).isoformat()
```

### Tool Best Practices
- **Docstring = LLM description**: Write clear docstrings — they become the tool's description sent to the model.
- **Parameter types = schema**: Type annotations generate the JSON schema for the LLM.
- **Return type**: Return strings, dicts, Pydantic models, or any JSON-serializable value.
- **Errors**: Raise `ModelRetry` to tell the LLM to retry with better args.

```python
from pydantic_ai.exceptions import ModelRetry

@agent.tool_plain
def divide(a: float, b: float) -> float:
    """Divide a by b."""
    if b == 0:
        raise ModelRetry('Cannot divide by zero — provide a non-zero b.')
    return a / b
```

---

## 3. Dependencies — Type-Safe Context Injection

Dependencies are the Pydantic AI way to pass DB connections, config, HTTP clients, etc. into tools and instructions without globals.

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext
import httpx

@dataclass
class AppDeps:
    http_client: httpx.AsyncClient
    user_id: int
    api_key: str

agent = Agent('openai:gpt-4o', deps_type=AppDeps)

@agent.tool
async def fetch_user_data(ctx: RunContext[AppDeps]) -> dict:
    """Fetch the current user's profile."""
    resp = await ctx.deps.http_client.get(
        f'/users/{ctx.deps.user_id}',
        headers={'Authorization': f'Bearer {ctx.deps.api_key}'}
    )
    return resp.json()

# Pass deps at runtime:
async with httpx.AsyncClient(base_url='https://api.example.com') as client:
    deps = AppDeps(http_client=client, user_id=42, api_key='secret')
    result = await agent.run('Get my profile', deps=deps)
```

### Dynamic Instructions with Dependencies

```python
@agent.instructions
async def personalized_instructions(ctx: RunContext[AppDeps]) -> str:
    user = await ctx.deps.db.get_user(ctx.deps.user_id)
    return f"The user's name is {user.name!r}. Personalize all responses."
```

---

## 4. Structured Output

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent

class MovieReview(BaseModel):
    title: str
    rating: int = Field(ge=1, le=10, description='Rating from 1-10')
    summary: str
    recommend: bool

agent = Agent('openai:gpt-4o', output_type=MovieReview)
result = agent.run_sync('Review the movie Inception')
review = result.output  # typed as MovieReview
print(review.rating)    # int, validated by Pydantic
```

### Union Output (Multiple Possible Types)

```python
from typing import Union

class SuccessResult(BaseModel):
    data: dict

class ErrorResult(BaseModel):
    error_message: str
    retry_after: int

agent = Agent('openai:gpt-4o', output_type=Union[SuccessResult, ErrorResult])
```

---

## 5. Multi-Agent Patterns

### Simple Delegation

```python
summarizer = Agent('openai:gpt-4o-mini', instructions='Summarize text concisely.')
analyst = Agent('openai:gpt-4o', instructions='Analyze data and provide insights.')

@analyst.tool_plain
async def summarize(text: str) -> str:
    """Summarize a long piece of text before analysis."""
    result = await summarizer.run(text)
    return result.output
```

### Passing Message History (Conversation)

```python
from pydantic_ai.messages import ModelMessagesTypeAdapter

result1 = await agent.run('Hello, my name is Alice.')
result2 = await agent.run('What is my name?', message_history=result1.new_messages())
print(result2.output)  # "Your name is Alice."
```

---

## 6. Streaming Structured Output

```python
from pydantic import BaseModel

class Analysis(BaseModel):
    summary: str
    key_points: list[str]
    confidence: float

agent = Agent('openai:gpt-4o', output_type=Analysis)

async with agent.run_stream('Analyze quantum computing trends') as resp:
    async for partial in resp.stream():
        print(partial)  # partial Analysis objects as they arrive
    final = await resp.get_output()
```

---

## 7. MCP Integration

```python
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPServerStdio

# Connect to an MCP server
server = MCPServerStdio('npx', ['-y', '@modelcontextprotocol/server-filesystem', '/tmp'])

agent = Agent('openai:gpt-4o', mcp_servers=[server])

async with agent.run_mcp_servers():
    result = await agent.run('List the files in /tmp')
    print(result.output)
```

---

## 8. Testing

Use `TestModel` to test agents without real API calls:

```python
from pydantic_ai import Agent
from pydantic_ai.models.test import TestModel

agent = Agent('openai:gpt-4o', output_type=bool)

def test_agent_response():
    with agent.override(model=TestModel(custom_output_text='true')):
        result = agent.run_sync('Is Paris the capital of France?')
    assert result.output is True
```

Use `FunctionModel` for fine-grained control:

```python
from pydantic_ai.models.function import FunctionModel, ModelRequest

def my_model(messages: list[ModelRequest], info) -> str:
    last = messages[-1].parts[-1].content
    return f'Echo: {last}'

with agent.override(model=FunctionModel(my_model)):
    result = agent.run_sync('test')
```

---

## 9. Retries & Error Handling

```python
from pydantic_ai import Agent
from pydantic_ai.exceptions import ModelRetry, UnexpectedModelBehavior

agent = Agent('openai:gpt-4o', retries=3)  # retry output validation up to 3x

@agent.tool_plain
def parse_number(text: str) -> int:
    """Parse a number from user input."""
    try:
        return int(text.strip())
    except ValueError:
        raise ModelRetry(f'Could not parse {text!r} as an integer. Please provide a valid number.')

# Handle run errors:
try:
    result = agent.run_sync('some prompt')
except UnexpectedModelBehavior as e:
    print(f'Model misbehaved: {e}')
```

---

## 10. Model Reference

| Model string | Provider |
|---|---|
| `'openai:gpt-4o'` | OpenAI |
| `'openai:gpt-4o-mini'` | OpenAI (fast/cheap) |
| `'anthropic:claude-sonnet-4-6'` | Anthropic |
| `'anthropic:claude-haiku-4-5'` | Anthropic (fast) |
| `'google-gla:gemini-2.0-flash'` | Google |
| `'groq:llama-3.3-70b-versatile'` | Groq (fast inference) |
| `'ollama:llama3.2'` | Local via Ollama |

---

## Common Patterns & Gotchas

### ✅ DO
- Use `@dataclass` for `deps_type` — clean, type-safe, easy to mock in tests
- Use `agent.run_sync()` in scripts, `await agent.run()` in async apps
- Access deps via `ctx.deps` inside tools/instructions
- Write descriptive docstrings on tools — they're the LLM's only description
- Use `output_type=YourPydanticModel` for validated structured data
- Use `message_history=result.new_messages()` for multi-turn conversations

### ❌ DON'T
- Don't use global variables for context that varies per request — use deps
- Don't forget `async` on tools that do I/O (DB calls, HTTP requests)
- Don't mix up `@agent.tool` (needs `RunContext`) and `@agent.tool_plain` (no context)
- Don't use `system_prompt` kwarg if you want dynamic instructions — use `@agent.instructions` decorator

---

## Reference Files

For deeper dives, load these reference files as needed:
- `references/tools-advanced.md` — Toolsets, deferred tools, human-in-the-loop approval
- `references/graph.md` — Pydantic Graph for complex stateful workflows
- `references/evals.md` — Pydantic Evals for systematic testing
- `references/models.md` — All model providers and custom model implementation
