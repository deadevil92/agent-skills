# Models Reference

## Model String Format

Pydantic AI uses the format `provider:model-name`:

```python
agent = Agent('openai:gpt-4o')
agent = Agent('anthropic:claude-sonnet-4-6')
agent = Agent('google-gla:gemini-2.0-flash')
```

## All Supported Providers

### OpenAI
```python
from pydantic_ai.models.openai import OpenAIModel

agent = Agent('openai:gpt-4o')
agent = Agent('openai:gpt-4o-mini')
agent = Agent('openai:o3')
agent = Agent('openai:o4-mini')

# Custom base URL (Azure, OpenAI-compatible)
model = OpenAIModel('gpt-4o', base_url='https://my-azure.openai.azure.com', api_key='...')
```

### Anthropic
```python
agent = Agent('anthropic:claude-opus-4-6')
agent = Agent('anthropic:claude-sonnet-4-6')
agent = Agent('anthropic:claude-haiku-4-5')
```

### Google
```python
agent = Agent('google-gla:gemini-2.0-flash')       # via Google AI (Gemini API)
agent = Agent('google-vertex:gemini-2.0-flash')    # via Vertex AI
```

### Groq
```python
agent = Agent('groq:llama-3.3-70b-versatile')
agent = Agent('groq:mixtral-8x7b-32768')
```

### Mistral
```python
agent = Agent('mistral:mistral-large-latest')
```

### Ollama (local)
```python
from pydantic_ai.models.openai import OpenAIModel

# Ollama uses OpenAI-compatible API
model = OpenAIModel('llama3.2', base_url='http://localhost:11434/v1', api_key='ollama')
agent = Agent(model)
```

### Bedrock (AWS)
```python
from pydantic_ai.models.bedrock import BedrockModel

model = BedrockModel('anthropic.claude-sonnet-4-6-v1:0')
agent = Agent(model)
```

## ModelSettings — Fine-Tuning Requests

```python
from pydantic_ai.settings import ModelSettings

agent = Agent(
    'openai:gpt-4o',
    model_settings=ModelSettings(
        temperature=0.7,
        max_tokens=2000,
        top_p=0.95,
    )
)

# Override per-run:
result = agent.run_sync('prompt', model_settings=ModelSettings(temperature=0.0))
```

## Fallback Model

Automatically fall back to another model on failure:

```python
from pydantic_ai.models.fallback import FallbackModel

model = FallbackModel('openai:gpt-4o', 'anthropic:claude-sonnet-4-6')
agent = Agent(model)
```

## Custom Model

Implement `Model` protocol for custom providers:

```python
from pydantic_ai.models import Model, ModelRequest, ModelResponse

class MyCustomModel(Model):
    async def request(self, messages, model_settings, ...):
        # call your API here
        return ModelResponse(...)
```

## Pydantic AI Gateway

Use the gateway to route to any provider with one API key:

```python
agent = Agent('gateway/openai:gpt-4o')
agent = Agent('gateway/anthropic:claude-sonnet-4-6')
agent = Agent('gateway/google:gemini-2.0-flash')
```

Set `PYDANTIC_AI_API_KEY` for gateway access.
