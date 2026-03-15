# Pydantic Graph Reference

> **Warning**: Graphs are for advanced, complex workflows. Use a plain `Agent` or multi-agent approach first. Only reach for graphs when control flow becomes spaghetti.

## Installation

```bash
pip install pydantic-graph  # included in pydantic-ai already
```

## Core Components

| Component | Role |
|---|---|
| `BaseNode` | A step in the graph (subclass + implement `run()`) |
| `Graph` | Container for all nodes; executes the state machine |
| `GraphRunContext` | Runtime context passed to each node (holds state + deps) |
| `End` | Signal to terminate the graph run and return a value |

## Simple Example

```python
from __future__ import annotations
from dataclasses import dataclass
from pydantic_graph import BaseNode, End, Graph, GraphRunContext

@dataclass
class DivisibleBy5(BaseNode[None, None, int]):
    foo: int

    async def run(self, ctx: GraphRunContext) -> Increment | End[int]:
        if self.foo % 5 == 0:
            return End(self.foo)
        return Increment(self.foo)

@dataclass
class Increment(BaseNode):
    foo: int

    async def run(self, ctx: GraphRunContext) -> DivisibleBy5:
        return DivisibleBy5(self.foo + 1)

graph = Graph(nodes=[DivisibleBy5, Increment])
result = graph.run_sync(DivisibleBy5(4))
print(result.output)  # 5
```

## Stateful Graphs

Share mutable state across all nodes:

```python
from dataclasses import dataclass, field
from pydantic_graph import BaseNode, End, Graph, GraphRunContext

@dataclass
class PipelineState:
    steps_completed: int = 0
    errors: list[str] = field(default_factory=list)
    result: str = ''

@dataclass
class ProcessStep(BaseNode[PipelineState]):
    data: str

    async def run(self, ctx: GraphRunContext[PipelineState]) -> ValidateStep | End[str]:
        ctx.state.steps_completed += 1
        processed = self.data.upper()
        ctx.state.result = processed
        return ValidateStep(processed)

@dataclass
class ValidateStep(BaseNode[PipelineState, None, str]):
    data: str

    async def run(self, ctx: GraphRunContext[PipelineState]) -> End[str]:
        if len(self.data) < 3:
            ctx.state.errors.append('Data too short')
            return End('FAILED')
        return End(self.data)

graph = Graph(nodes=[ProcessStep, ValidateStep])
result = graph.run_sync(ProcessStep('hello'), state=PipelineState())
print(result.output)  # 'HELLO'
```

## Graph with Dependencies (Injection)

```python
from dataclasses import dataclass
import httpx
from pydantic_graph import BaseNode, End, Graph, GraphRunContext

@dataclass
class MyDeps:
    http_client: httpx.AsyncClient
    api_key: str

@dataclass
class FetchData(BaseNode[None, MyDeps, dict]):
    url: str

    async def run(self, ctx: GraphRunContext[None, MyDeps]) -> ProcessData | End[dict]:
        resp = await ctx.deps.http_client.get(self.url)
        return ProcessData(resp.json())

...

graph = Graph(nodes=[FetchData, ProcessData])
async with httpx.AsyncClient() as client:
    deps = MyDeps(http_client=client, api_key='secret')
    result = await graph.run(FetchData('https://api.example.com/data'), deps=deps)
```

## Combining Graphs with Agents

The most powerful pattern: use graph nodes to orchestrate multiple agents:

```python
from dataclasses import dataclass
from pydantic_ai import Agent
from pydantic_graph import BaseNode, End, Graph, GraphRunContext

planner = Agent('openai:gpt-4o', instructions='Break task into subtasks. Output JSON list.')
executor = Agent('openai:gpt-4o', instructions='Execute the given task step.')

@dataclass
class PlanStep(BaseNode[None, None, str]):
    task: str

    async def run(self, ctx: GraphRunContext) -> ExecuteStep:
        result = await planner.run(self.task)
        subtasks = json.loads(result.output)
        return ExecuteStep(subtasks)

@dataclass
class ExecuteStep(BaseNode[None, None, str]):
    subtasks: list[str]

    async def run(self, ctx: GraphRunContext) -> End[str]:
        results = []
        for subtask in self.subtasks:
            r = await executor.run(subtask)
            results.append(r.output)
        return End('\n'.join(results))
```

## Visualizing Graphs

```python
# Generate a Mermaid diagram of the graph
diagram = graph.mermaid_code()
print(diagram)

# Save as PNG (requires mermaid CLI)
graph.mermaid_save('pipeline.png')
```

## When to Use Graphs

✅ Use graphs when:
- You have complex branching logic that's hard to read as nested if/else
- You need explicit state machine semantics (finite set of states + transitions)
- You want to visualize / audit the workflow structure
- You need to pause and resume execution (with persistence)

❌ Don't use graphs when:
- Simple sequential agent calls suffice
- You have 2-3 agents collaborating (use multi-agent tools pattern instead)
- You're still prototyping — agents are faster to iterate
