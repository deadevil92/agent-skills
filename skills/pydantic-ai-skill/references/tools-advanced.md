# Advanced Tools Reference

## Toolsets

Group related tools together using `FunctionToolset`:

```python
from pydantic_ai.toolsets import FunctionToolset

math_tools = FunctionToolset()

@math_tools.tool
def add(a: float, b: float) -> float:
    """Add two numbers."""
    return a + b

@math_tools.tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

agent = Agent('openai:gpt-4o', toolsets=[math_tools])
```

Use `MCPToolset` to expose MCP server tools, with optional filtering:

```python
from pydantic_ai.toolsets import MCPToolset

mcp_toolset = MCPToolset(server, allowed_tools=['read_file', 'write_file'])
agent = Agent('openai:gpt-4o', toolsets=[mcp_toolset])
```

## Dynamic Tools (per-step customization)

The `prepare` function lets you modify or hide a tool at each agent step:

```python
from pydantic_ai import Agent, RunContext, ToolDefinition

async def only_if_admin(ctx: RunContext[dict], tool_def: ToolDefinition) -> ToolDefinition | None:
    """Return None to hide this tool; return tool_def to show it."""
    if ctx.deps.get('is_admin'):
        return tool_def
    return None  # hides the tool for non-admins

@agent.tool(prepare=only_if_admin)
def delete_user(ctx: RunContext[dict], user_id: int) -> str:
    """Delete a user (admin only)."""
    ...
```

## Human-in-the-Loop Tool Approval

```python
from pydantic_ai import Agent, ApprovalRequired, DeferredToolRequests, DeferredToolResults, RunContext

agent = Agent('openai:gpt-4o', output_type=[str, DeferredToolRequests])

@agent.tool_plain(requires_approval=True)
def delete_file(path: str) -> str:
    """Delete a file. Requires user approval."""
    return f'Deleted {path}'

@agent.tool
def update_file(ctx: RunContext, path: str, content: str) -> str:
    """Update a file. Protected files need approval."""
    if path in {'.env', 'secrets.json'} and not ctx.tool_call_approved:
        raise ApprovalRequired(metadata={'reason': 'protected file'})
    return f'Updated {path}'

# First run — agent requests approval
result = agent.run_sync('Delete config.yaml and clear .env')
messages = result.all_messages()
assert isinstance(result.output, DeferredToolRequests)

# Gather approvals from user
deferred_results = DeferredToolResults()
for call in result.output.approvals:
    user_approved = input(f'Approve {call.tool_name}({call.args})? [y/n]: ') == 'y'
    deferred_results.approvals[call.tool_call_id] = user_approved

# Second run — continue with approvals
result2 = agent.run_sync(
    'Continue',
    message_history=messages,
    deferred_tool_results=deferred_results,
)
print(result2.output)
```

## ToolReturn — Rich Multi-Modal Tool Responses

Use `ToolReturn` when you want to send rich context to the model separately from the tool's return value:

```python
from pydantic_ai import Agent, ToolReturn, BinaryContent

@agent.tool_plain
def capture_screenshot() -> ToolReturn:
    """Take a screenshot of the current state."""
    image_bytes = take_screenshot()
    return ToolReturn(
        return_value='Screenshot captured successfully',
        content=[
            'Here is the current screen state:',
            BinaryContent(data=image_bytes, media_type='image/png'),
        ],
        metadata={'timestamp': time.time()}  # not sent to LLM
    )
```

## Custom Tool Schema

For poorly-named or `**kwargs`-based functions:

```python
from pydantic_ai import Tool

tool = Tool.from_schema(
    function=my_func,
    name='better_name',
    description='Clear description of what this does.',
    json_schema={
        'type': 'object',
        'properties': {
            'x': {'type': 'integer', 'description': 'The input value'},
        },
        'required': ['x'],
        'additionalProperties': False,
    },
    takes_ctx=False,
)

agent = Agent('openai:gpt-4o', tools=[tool])
```
