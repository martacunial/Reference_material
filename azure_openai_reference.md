# Azure OpenAI API Reference Guide

Personal backbone for calling Azure OpenAI (chat completions) in Python projects.

**What Azure OpenAI is:** Microsoft’s hosted version of OpenAI models (GPT-4o, GPT-4, etc.) accessed via your own Azure subscription. The API is **compatible with the `openai` Python SDK** — the only difference is you use `AzureOpenAI` instead of `OpenAI` and authenticate with Azure-specific credentials.

**Key Azure concepts:**
- **Resource** — the Azure OpenAI service instance you create in the portal. Has a unique endpoint URL.
- **Deployment** — a specific model version you deploy under a custom name (e.g. `"gpt-4o-prod"`). This is what you pass as the `model` argument in API calls, **not** the model family name.
- **API version** — the REST API version (date-based, e.g. `2024-06-01`). Features like JSON mode and function calling require specific minimum versions.

---

## Install

```bash
pip install openai python-dotenv
# or with uv:
uv add openai python-dotenv
```

---

## .env file setup

Never hardcode credentials in source files. The `.env` pattern keeps secrets out of git while making them available as environment variables at runtime. `load_dotenv()` searches for the file starting from the current working directory and walking up to parent directories — so placing `.env` at the project root works for all scripts run from inside the project.

Create a `.env` file in the project root (never commit it — add to `.gitignore`):

```
AZURE_OPENAI_API_KEY=your-key-here
AZURE_OPENAI_API_VERSION=2024-06-01
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT_NAME=gpt-4o
```

Create template for teammates:

```
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_API_VERSION=
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_DEPLOYMENT_NAME=
```

Copy on Windows: `copy .env.example .env`
Copy on Linux/Mac: `cp .env.example .env`

---

## Load credentials

```python
import os
from dotenv import load_dotenv

load_dotenv()  # reads .env from cwd or parent dirs automatically

api_key        = os.getenv("AZURE_OPENAI_API_KEY") or ""
azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT") or ""
api_version    = os.getenv("AZURE_OPENAI_API_VERSION") or ""
deployment     = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME") or ""
```

---

## Initialize client

The client object is reusable — create it once at module level and share it across calls. It manages connection pooling internally. Do **not** re-instantiate it on every API call.

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    api_key=api_key,
    api_version=api_version,
    azure_endpoint=azure_endpoint,
)
```

---

## Basic chat completion call

The `messages` list is the full conversation history. Each message has a `role` (`"system"`, `"user"`, or `"assistant"`) and `content`. The model has no persistent memory — you must resend the entire history on every call if you need multi-turn context.

`temperature` controls randomness: `0` = deterministic (same input always gives same output), `1` = creative/varied. For analysis tasks always use `0`.

`max_completion_tokens` caps the **output** length. It does not affect how much input (prompt) you can send.

```python
response = client.chat.completions.create(
    model=deployment,                  # deployment name, not model family name
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user",   "content": "Explain anomaly detection in 2 sentences."},
    ],
    temperature=0,                     # 0 = deterministic, 1 = creative
    max_completion_tokens=500,
)

answer = response.choices[0].message.content
print(answer)
```

---

## System + User prompt pattern (RCA use case)

The **system message** acts as a permanent instruction layer — it sets role, tone, output format, and constraints. It is processed before any user content and cannot be overridden by the user message. Keep it focused on *how* to respond.

The **user message** provides the *what* — the actual data and question. Separating them makes prompts easier to maintain and test independently.

```python
system_prompt = """
You are a Senior Industrial Reliability Consultant.
Provide executive Root Cause Analysis. Max 150 words.
Bold **Component Names**. No bullet points. No raw numbers.
"""

user_prompt = f"""
Context: anomaly detected from {start_time} to {end_time}.
Top contributing features: {feature_table}
Raw logs sample:
{raw_logs_csv}
"""

response = client.chat.completions.create(
    model=deployment,
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": user_prompt},
    ],
    temperature=0,
    max_completion_tokens=500,
)
rca_answer = response.choices[0].message.content
```

---

## Iterating over multiple items (cluster loop pattern)

```python
results = {}

for cluster_id in cluster_ids:
    system_prompt, user_prompt = build_prompt(cluster_id, ...)
    try:
        response = client.chat.completions.create(
            model=deployment,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user",   "content": user_prompt},
            ],
            temperature=0,
            max_completion_tokens=500,
        )
        results[cluster_id] = response.choices[0].message.content
    except Exception as e:
        results[cluster_id] = f"Analysis failed: {e}"
        # Continue to next cluster instead of crashing
```

---

## Response structure

`finish_reason` tells you **why** the model stopped generating:
- `"stop"` — model finished naturally (normal case).
- `"length"` — hit `max_completion_tokens` limit; output may be truncated. Increase the limit or shorten your prompt.
- `"content_filter"` — Azure content policy blocked the output. Check your prompt for sensitive content.

Always check `finish_reason` in production pipelines to detect silent truncation.

```python
response.choices[0].message.content   # the text answer
response.choices[0].finish_reason     # "stop" | "length" | "content_filter"
response.usage.prompt_tokens
response.usage.completion_tokens
response.usage.total_tokens
```

---

## Streaming response (optional, for long outputs)

Streaming returns tokens as they are generated rather than waiting for the full response. Useful in interactive UIs (shows text appearing progressively) or when `max_completion_tokens` is large and you don’t want to block. Not needed for batch processing pipelines.

```python
stream = client.chat.completions.create(
    model=deployment,
    messages=[{"role": "user", "content": "..."}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

---

## Prompt engineering tips used in this project

- **System role** sets the persona and hard constraints (word count, formatting, no numbers).
- **User prompt** provides structured data: timestamps, feature tables, raw CSV logs.
- Downsampling raw logs with `.iloc[::20]` prevents exceeding the context window.
- `temperature=0` is used for deterministic, reproducible analysis outputs.

---

## Key notes

- `load_dotenv()` searches for `.env` walking up the directory tree from cwd.
- `model` parameter in `create()` = **deployment name** (set in Azure Portal), not the model family.
- Azure OpenAI API version must match what is enabled on the resource (e.g. `2024-06-01`).
- Always use `os.getenv(...) or ""` to avoid passing `None` to the client constructor.
- Wrap each call in `try/except` when looping — partial results are better than a crash.

---

## JSON mode (structured output)

Force the model to return valid JSON — no prose, no markdown fences. This is the most reliable way to get machine-parseable output. Without this flag, the model sometimes wraps JSON in ```json ... ``` fences or adds explanatory sentences, breaking `json.loads()`.

**Requires API version `≥2024-02-15-preview`** and a model that supports it (gpt-4o, gpt-4-turbo).

```python
response = client.chat.completions.create(
    model=deployment,
    messages=[
        {"role": "system", "content": "You are a JSON-only responder. Return valid JSON."},
        {"role": "user",   "content": "List top 3 failure modes as JSON array with keys: id, name, severity."},
    ],
    response_format={"type": "json_object"},  # Azure OpenAI ≥ 2024-02-15-preview
    temperature=0,
)

import json
data = json.loads(response.choices[0].message.content)
```

> **Note:** System prompt must explicitly mention JSON or the API returns an error.

---

## Retry with exponential backoff

Azure OpenAI enforces **rate limits** (tokens-per-minute and requests-per-minute) that vary by deployment tier. When you exceed them you get HTTP 429. `tenacity` retries the call with increasing delays (2 s → 4 s → 8 s → … up to 60 s), giving the quota time to reset. `stop_after_attempt(5)` prevents infinite loops on persistent failures.

`APIStatusError` catches 5xx server errors (transient Azure outages) in addition to rate limits.

Rate limits (429) and transient errors are common in loops. Use `tenacity`:

```bash
uv add tenacity
```

```python
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type
from openai import RateLimitError, APIStatusError

@retry(
    retry=retry_if_exception_type((RateLimitError, APIStatusError)),
    wait=wait_exponential(multiplier=1, min=2, max=60),
    stop=stop_after_attempt(5),
)
def call_llm(messages: list) -> str:
    response = client.chat.completions.create(
        model=deployment,
        messages=messages,
        temperature=0,
        max_completion_tokens=500,
    )
    return response.choices[0].message.content
```

---

## Count tokens before sending (tiktoken)

Avoid hitting context-window limits at runtime. Token counting is fast (pure Python, no API call) and should be done before every call in production. The formula below matches OpenAI’s own counting logic for chat-formatted messages (4 tokens of overhead per message, 2 tokens of overhead for the reply primer).

Avoid hitting context-window limits at runtime:

```bash
uv add tiktoken
```

```python
import tiktoken

def count_tokens(messages: list, model: str = "gpt-4o") -> int:
    enc = tiktoken.encoding_for_model(model)
    total = 0
    for m in messages:
        total += 4  # per-message overhead
        total += len(enc.encode(m["content"]))
    return total + 2  # reply priming

tokens = count_tokens(messages)
if tokens > 120_000:
    raise ValueError(f"Prompt too long: {tokens} tokens")
```

> **gpt-4o context window:** 128 k tokens. Reserve ~2 k for `max_completion_tokens`.

---

## Async client (parallelise multiple calls)

The synchronous client processes calls one at a time. For N clusters with ~2 s latency each, that’s N×2 s total. `AsyncAzureOpenAI` lets you fire all calls concurrently with `asyncio.gather()`, reducing total wall time to roughly the latency of the single slowest call — a major speedup for 10+ clusters.

`return_exceptions=True` in `gather()` means a failure in one cluster doesn’t cancel the others; exceptions are returned as values in the results list. Check with `isinstance(result, Exception)` after.

```python
import asyncio
from openai import AsyncAzureOpenAI

async_client = AsyncAzureOpenAI(
    api_key=api_key,
    api_version=api_version,
    azure_endpoint=azure_endpoint,
)

async def call_async(messages: list) -> str:
    response = await async_client.chat.completions.create(
        model=deployment,
        messages=messages,
        temperature=0,
        max_completion_tokens=500,
    )
    return response.choices[0].message.content

# Run all cluster RCAs in parallel
async def run_all(prompts: dict) -> dict:
    tasks = {cid: call_async(msgs) for cid, msgs in prompts.items()}
    results = await asyncio.gather(*tasks.values(), return_exceptions=True)
    return dict(zip(tasks.keys(), results))

results = asyncio.run(run_all(prompts))  # works in scripts
# In Jupyter: await run_all(prompts)
```

---

## Function calling / tool use

Function calling lets the model output a structured **function invocation** instead of free text. The model reads the function’s `description` and `parameters` schema and decides whether to call it based on the user message — you never execute the function inside the API; you receive the arguments and call the function yourself.

This is more reliable than parsing free text for structured data and works well for: creating tickets, extracting fields, routing to different handlers, or building agent pipelines.

`tool_choice="auto"` lets the model decide; use `{"type": "function", "function": {"name": "..."}}` to force a specific function.

Let the model call structured functions instead of free text:

```python
tools = [{
    "type": "function",
    "function": {
        "name": "create_maintenance_ticket",
        "description": "Create a maintenance ticket for an anomalous component.",
        "parameters": {
            "type": "object",
            "properties": {
                "component": {"type": "string", "description": "Failing component name"},
                "severity":  {"type": "string", "enum": ["low", "medium", "high", "critical"]},
                "summary":   {"type": "string", "description": "One-sentence root cause"},
            },
            "required": ["component", "severity", "summary"],
        },
    },
}]

response = client.chat.completions.create(
    model=deployment,
    messages=[{"role": "user", "content": rca_prompt}],
    tools=tools,
    tool_choice="auto",   # or {"type": "function", "function": {"name": "create_maintenance_ticket"}}
)

import json
tool_call = response.choices[0].message.tool_calls[0]
args = json.loads(tool_call.function.arguments)
print(args)  # {"component": "...", "severity": "high", "summary": "..."}
```
