---
name: crawl4ai
description: >
  Expert guidance for building web crawlers and scrapers using Crawl4AI
  (the `crawl4ai` Python library). Use this skill whenever the user is:
  writing, debugging, or reviewing any Crawl4AI code; asking how to scrape
  or crawl websites in Python with Crawl4AI; asking about AsyncWebCrawler,
  BrowserConfig, CrawlerRunConfig, CrawlResult, extraction strategies
  (JsonCssExtractionStrategy, LLMExtractionStrategy, RegexExtractionStrategy),
  deep crawling, adaptive crawling, session management, page interaction,
  proxy/anti-bot setup, or content filtering with Crawl4AI; or migrating
  from BeautifulSoup/Scrapy/Playwright to Crawl4AI. Trigger even for vague
  requests like "help me scrape a website in Python" or "how do I extract
  structured data from a web page" — Crawl4AI is very likely what they need.
---

# Crawl4AI Skill

Crawl4AI is an open-source, async-first, LLM-friendly web crawler and scraper.
Its core API is `AsyncWebCrawler`, used as an async context manager.

## Quick Mental Model

```
AsyncWebCrawler(config=BrowserConfig(...))   ← browser/session settings
    └─ .arun(url, config=CrawlerRunConfig(...))  ← per-crawl settings
           └─ returns CrawlResult
```

## Installation

```bash
pip install crawl4ai
crawl4ai-setup   # downloads Playwright browsers
```

## Minimal Example

```python
import asyncio
from crawl4ai import AsyncWebCrawler
from crawl4ai.async_configs import BrowserConfig, CrawlerRunConfig

async def main():
    async with AsyncWebCrawler(config=BrowserConfig()) as crawler:
        result = await crawler.arun(
            url="https://example.com",
            config=CrawlerRunConfig()
        )
        if result.success:
            print(result.markdown)

asyncio.run(main())
```

---

## Key Config Objects

### BrowserConfig  ← browser-level, shared across crawls
| Param | Default | Purpose |
|---|---|---|
| `headless` | `True` | Run browser in headless mode |
| `verbose` | `False` | Enable debug logging |
| `proxy_config` | `None` | Dict with `server`, `username`, `password` |
| `user_agent` | auto | Override browser user agent |
| `use_managed_browser` | `False` | Persistent browser session (for auth/sessions) |

### CrawlerRunConfig  ← per-crawl settings
| Param | Default | Purpose |
|---|---|---|
| `word_count_threshold` | `10` | Min words per content block |
| `excluded_tags` | `[]` | HTML tags to strip (e.g. `['nav','footer']`) |
| `exclude_external_links` | `False` | Strip external links from output |
| `remove_overlay_elements` | `False` | Remove popups/modals |
| `process_iframes` | `False` | Include iframe content |
| `cache_mode` | `ENABLED` | See Cache Modes below |
| `extraction_strategy` | `None` | Structured extraction strategy |
| `js_code` | `None` | JS string(s) to run before extraction |
| `wait_for` | `None` | CSS selector or JS expression to await |
| `screenshot` | `False` | Capture screenshot |
| `pdf` | `False` | Capture PDF |
| `deep_crawl_strategy` | `None` | Enable multi-page crawling |
| `markdown_generator` | `DefaultMarkdownGenerator()` | Controls markdown output |

---

## CrawlResult — Key Properties

```python
result.success          # bool
result.status_code      # int (e.g. 200)
result.error_message    # str if failed

result.html             # raw HTML
result.cleaned_html     # sanitized HTML
result.markdown         # MarkdownGenerationResult object
result.markdown.raw_markdown   # all content as markdown
result.markdown.fit_markdown   # filtered, most relevant content

result.extracted_content  # JSON string from extraction strategy
result.media            # {"images": [...], "videos": [...], "audio": [...]}
result.links            # {"internal": [...], "external": [...]}
result.screenshot       # base64 PNG if screenshot=True
```

---

## Extraction Strategies

### 1. No-LLM: CSS/XPath Schema (preferred for structured repeating data)

```python
from crawl4ai.extraction_strategy import JsonCssExtractionStrategy
import json

schema = {
    "name": "Products",
    "baseSelector": ".product-card",
    "fields": [
        {"name": "title",  "selector": "h2.title",   "type": "text"},
        {"name": "price",  "selector": ".price",      "type": "text"},
        {"name": "url",    "selector": "a",           "type": "attribute", "attribute": "href"},
        {"name": "tags",   "selector": ".tag",        "type": "list",
         "fields": [{"name": "label", "selector": "span", "type": "text"}]}
    ]
}

config = CrawlerRunConfig(
    extraction_strategy=JsonCssExtractionStrategy(schema)
)
result = await crawler.arun(url=url, config=config)
products = json.loads(result.extracted_content)
```

**Field types:** `text`, `html`, `attribute`, `list`, `nested`

### 2. No-LLM: Regex (fast pattern matching)

```python
from crawl4ai.extraction_strategy import RegexExtractionStrategy

config = CrawlerRunConfig(
    extraction_strategy=RegexExtractionStrategy(
        patterns={"emails": r"[\w.-]+@[\w.-]+\.\w+", "phones": r"\+?\d[\d\s\-]{7,}"}
    )
)
result = await crawler.arun(url=url, config=config)
data = json.loads(result.extracted_content)
```

### 3. LLM-Based Extraction (for unstructured/semantic data)

```python
from crawl4ai.extraction_strategy import LLMExtractionStrategy
from pydantic import BaseModel

class Article(BaseModel):
    title: str
    author: str
    summary: str

config = CrawlerRunConfig(
    extraction_strategy=LLMExtractionStrategy(
        provider="openai/gpt-4o-mini",  # or "anthropic/claude-3-haiku-20240307"
        api_token="sk-...",
        schema=Article.model_json_schema(),
        extraction_type="schema",
        instruction="Extract the main article details."
    )
)
```

---

## Deep / Multi-Page Crawling

```python
from crawl4ai.deep_crawling import BFSDeepCrawlStrategy
from crawl4ai.deep_crawling.filters import URLPatternFilter, DomainFilter
from crawl4ai.deep_crawling.scorers import KeywordRelevanceScorer

strategy = BFSDeepCrawlStrategy(
    max_depth=2,
    max_pages=50,
    include_patterns=["**/blog/**"],   # glob patterns to include
    exclude_patterns=["**/tag/**"],    # glob patterns to exclude
    filter_chain=[
        DomainFilter(allowed_domains=["example.com"]),
        URLPatternFilter(patterns=["**/article/**"])
    ],
    scorer=KeywordRelevanceScorer(keywords=["python", "ai"], weight=0.8)
)

config = CrawlerRunConfig(deep_crawl_strategy=strategy, stream=True)

async with AsyncWebCrawler() as crawler:
    async for result in await crawler.arun(url="https://example.com", config=config):
        print(result.url, result.markdown.fit_markdown[:200])
```

**Strategy options:** `BFSDeepCrawlStrategy`, `DFSDeepCrawlStrategy`, `BestFirstCrawlingStrategy`

---

## Page Interaction (JS / Dynamic Content)

```python
config = CrawlerRunConfig(
    js_code="window.scrollTo(0, document.body.scrollHeight);",
    wait_for="css:.content-loaded",   # or "js:() => document.querySelectorAll('.item').length > 10"
    js_only=False  # True = re-run JS on existing page without reloading (for sessions)
)
```

For multi-step interaction, use a list of JS strings:
```python
js_code=["document.querySelector('#load-more').click()", "window.scrollTo(0,9999)"]
```

---

## Session Management (Login / Multi-Step)

```python
browser_config = BrowserConfig(use_managed_browser=True)

async with AsyncWebCrawler(config=browser_config) as crawler:
    # Step 1: Log in
    await crawler.arun(
        url="https://example.com/login",
        config=CrawlerRunConfig(
            session_id="my_session",
            js_code="""
                document.querySelector('#email').value = 'user@example.com';
                document.querySelector('#password').value = 'secret';
                document.querySelector('[type=submit]').click();
            """,
            wait_for="css:.dashboard"
        )
    )
    # Step 2: Crawl authenticated page
    result = await crawler.arun(
        url="https://example.com/protected",
        config=CrawlerRunConfig(session_id="my_session")
    )
```

---

## Content Filtering / Markdown Quality

```python
from crawl4ai.content_filter_strategy import PruningContentFilter, BM25ContentFilter
from crawl4ai.markdown_generation_strategy import DefaultMarkdownGenerator

# Pruning — removes low-information blocks
config = CrawlerRunConfig(
    markdown_generator=DefaultMarkdownGenerator(
        content_filter=PruningContentFilter(threshold=0.5),
        options={"ignore_links": True}
    )
)

# BM25 — keyword-relevance filtering
config = CrawlerRunConfig(
    markdown_generator=DefaultMarkdownGenerator(
        content_filter=BM25ContentFilter(user_query="machine learning research")
    )
)
# result.markdown.fit_markdown has the filtered content
```

---

## Cache Modes

```python
from crawl4ai.async_configs import CacheMode

CacheMode.ENABLED       # Use cache if available, save new results
CacheMode.BYPASS        # Skip cache, always fetch fresh, save result
CacheMode.DISABLED      # Never use or save cache
CacheMode.READ_ONLY     # Use cache only, don't save
CacheMode.WRITE_ONLY    # Don't read cache, always save
```

---

## Multi-URL Crawling

```python
results = await crawler.arun_many(
    urls=["https://a.com", "https://b.com", "https://c.com"],
    config=CrawlerRunConfig(cache_mode=CacheMode.BYPASS)
)
for result in results:
    print(result.url, result.success)
```

---

## Anti-Bot / Stealth

```python
browser_config = BrowserConfig(
    user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64)...",
    headers={"Accept-Language": "en-US,en;q=0.9"},
)

config = CrawlerRunConfig(
    magic=True,           # Enable all anti-detection measures
    simulate_user=True,   # Simulate mouse movements / human timing
    override_navigator=True  # Spoof navigator properties
)
```

---

## Proxy

```python
browser_config = BrowserConfig(
    proxy_config={
        "server": "http://proxy.example.com:8080",
        "username": "user",
        "password": "pass"
    }
)
```

---

## Common Patterns & Tips

- **Always check `result.success`** before using content — failed crawls won't raise exceptions by default.
- **Prefer `fit_markdown`** over `raw_markdown` for LLM pipelines — it's cleaner and shorter.
- **Use `JsonCssExtractionStrategy`** for repeating structured data (product listings, tables, article lists) — faster, cheaper, more reliable than LLM extraction.
- **Use `session_id`** with `use_managed_browser=True` for any multi-step or authenticated flows.
- **`js_only=True`** lets you re-execute JS on an already-loaded page without a fresh request — useful for pagination.
- **`stream=True`** in deep crawls lets you process results as they arrive rather than waiting for the whole crawl.
- For large-scale crawls, combine `arun_many()` with `CacheMode.ENABLED` to avoid re-fetching.

---

## Reference Docs

Full API details are available at: https://docs.crawl4ai.com/
Key sections:
- https://docs.crawl4ai.com/api/async-webcrawler/
- https://docs.crawl4ai.com/api/crawl-result/
- https://docs.crawl4ai.com/api/parameters/
- https://docs.crawl4ai.com/extraction/no-llm-strategies/
- https://docs.crawl4ai.com/extraction/llm-strategies/
- https://docs.crawl4ai.com/core/deep-crawling/
- https://docs.crawl4ai.com/advanced/hooks-auth/
