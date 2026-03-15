# Pydantic Evals Reference

Pydantic Evals (`pydantic_evals`) is a separate package for systematic testing and evaluation of AI agents.

## Installation

```bash
pip install pydantic-evals
```

## Core Concepts

A **Dataset** is a collection of **Cases**. Each case has:
- `inputs`: the input(s) to your agent/function
- `expected_output` (optional): the ground truth
- `evaluators`: list of scoring functions

## Quick Start

```python
from pydantic_evals import Dataset, Case
from pydantic_evals.evaluators import IsInstance, LLMJudge

from my_agent import answer_agent  # your Agent

dataset = Dataset(cases=[
    Case(
        name='capital_france',
        inputs='What is the capital of France?',
        expected_output='Paris',
        evaluators=[IsInstance(str), LLMJudge('Is the answer correct?')],
    ),
    Case(
        name='capital_germany',
        inputs='What is the capital of Germany?',
        expected_output='Berlin',
        evaluators=[IsInstance(str), LLMJudge('Is the answer correct?')],
    ),
])

async def run_agent(inputs: str) -> str:
    result = await answer_agent.run(inputs)
    return result.output

report = await dataset.evaluate(run_agent)
report.print()
```

## Built-in Evaluators

| Evaluator | Purpose |
|---|---|
| `IsInstance(type)` | Check output type |
| `Equals(value)` | Exact equality check |
| `Contains(substring)` | String contains check |
| `LLMJudge(prompt)` | Use an LLM to score the output |
| `MaxDuration(seconds)` | Check response time |
| `IsValidJson()` | Check JSON validity |

## LLM Judge

```python
from pydantic_evals.evaluators import LLMJudge

judge = LLMJudge(
    rubric='Is the response helpful, accurate, and concise?',
    model='openai:gpt-4o',  # defaults to a capable model
    include_input=True,
)
```

## Custom Evaluators

```python
from pydantic_evals.evaluators import Evaluator, EvaluatorContext

class WordCountLimit(Evaluator):
    max_words: int = 100

    def evaluate(self, ctx: EvaluatorContext) -> float:
        words = len(str(ctx.output).split())
        return 1.0 if words <= self.max_words else 0.0
```

## Dataset from YAML

```yaml
# cases.yaml
- name: test1
  inputs: "What is 2+2?"
  expected_output: "4"
- name: test2
  inputs: "Capital of Japan?"
  expected_output: "Tokyo"
```

```python
dataset = Dataset.from_yaml('cases.yaml', inputs_type=str, output_type=str)
```

## Integration with Logfire

```python
import logfire
logfire.configure()

# Evals automatically trace to Logfire
report = await dataset.evaluate(run_agent)
```
