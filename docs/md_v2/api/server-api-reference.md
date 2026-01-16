# Crawl4AI Server API Reference

**Format:** Ultra-Dense Reference (Optimized for AI Assistants & Developers)
**Version:** 0.7.7
**Base URL:** `https://crawl4ai-production-0a7e.up.railway.app`

---

## Navigation

- [Quick Start](#quick-start)
- [Authentication](#authentication)
- [REST API Endpoints](#rest-api-endpoints)
  - [Core Endpoints](#core-endpoints)
  - [Async Job Endpoints](#async-job-endpoints)
  - [Utility Endpoints](#utility-endpoints)
- [MCP Server](#mcp-server)
- [Async Jobs & Webhooks](#async-jobs--webhooks)
- [Configuration Reference](#configuration-reference)
- [Error Handling](#error-handling)
- [Examples](#examples)

---

# Quick Start

## Base URL

```
https://crawl4ai-production-0a7e.up.railway.app
```

All API requests require authentication via API key (see [Authentication](#authentication)).

## First API Call

**curl:**
```bash
# Simple crawl
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/crawl \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"urls": ["https://example.com"]}'

# Get markdown
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/md \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com", "f": "fit"}'
```

**Python:**
```python
import requests

BASE_URL = "https://crawl4ai-production-0a7e.up.railway.app"
HEADERS = {
    "Content-Type": "application/json",
    "X-API-Key": "YOUR_API_KEY"
}

# Simple crawl
response = requests.post(
    f"{BASE_URL}/crawl",
    headers=HEADERS,
    json={"urls": ["https://example.com"]}
)
result = response.json()
print(result["results"][0]["markdown"]["raw_markdown"][:500])

# Using the SDK
from crawl4ai.docker_client import Crawl4aiDockerClient

async with Crawl4aiDockerClient(
    base_url=BASE_URL,
    api_key="YOUR_API_KEY"
) as client:
    result = await client.crawl(["https://example.com"])
    print(result[0].markdown)
```

## Web Interfaces

| Interface | URL | Purpose |
|-----------|-----|---------|
| Playground | `https://crawl4ai-production-0a7e.up.railway.app/playground` | Interactive testing, config generation |
| Dashboard | `https://crawl4ai-production-0a7e.up.railway.app/dashboard` | Real-time monitoring |
| API Docs | `https://crawl4ai-production-0a7e.up.railway.app/docs` | OpenAPI/Swagger UI |

## MCP Quick Setup (Claude Code)

```bash
# Add Crawl4AI as MCP provider
claude mcp add --transport sse c4ai-sse https://crawl4ai-production-0a7e.up.railway.app/mcp/sse

# Verify
claude mcp list
```

---

# Authentication

## API Key Authentication

All API requests (except public endpoints) require an API key.

**Header formats:**
```
X-API-Key: your-api-key
# or
Authorization: Bearer your-api-key
```

**Example:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/crawl \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"urls": ["https://example.com"]}'
```

**Public endpoints (no auth required):**
- `/health`
- `/`
- `/docs`
- `/openapi.json`
- `/redoc`

---

# REST API Endpoints

## Core Endpoints

### POST /crawl

Main batch crawling endpoint. Returns JSON results for one or more URLs.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| urls | List[str] | required | URLs to crawl (1-100) |
| browser_config | Dict | {} | Browser configuration |
| crawler_config | Dict | {} | Crawler run configuration |
| hooks | HookConfig | null | Optional hook functions |

**HookConfig Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| code | Dict[str, str] | {} | Map of hook points to Python code |
| timeout | int | 30 | Hook execution timeout (1-120 seconds) |

**Example Request:**
```json
{
  "urls": ["https://example.com", "https://httpbin.org/html"],
  "browser_config": {
    "type": "BrowserConfig",
    "params": {"headless": true}
  },
  "crawler_config": {
    "type": "CrawlerRunConfig",
    "params": {"cache_mode": "bypass"}
  }
}
```

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/crawl \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "urls": ["https://example.com"],
    "crawler_config": {
      "type": "CrawlerRunConfig",
      "params": {"cache_mode": "bypass"}
    }
  }'
```

**Response:**
```json
{
  "success": true,
  "results": [
    {
      "url": "https://example.com",
      "success": true,
      "html": "<html>...</html>",
      "markdown": {
        "raw_markdown": "# Example Domain...",
        "fit_markdown": "# Example Domain...",
        "markdown_with_citations": "..."
      },
      "links": {"internal": [], "external": []},
      "media": {"images": [], "videos": []},
      "metadata": {...}
    }
  ],
  "server_processing_time_s": 2.35,
  "server_memory_delta_mb": 15.2
}
```

---

### POST /crawl/stream

Streaming crawl endpoint. Returns NDJSON (newline-delimited JSON).

**Request:** Same as `/crawl`

**Response:** Stream of JSON objects, one per line:
```
{"url": "https://example.com", "success": true, "markdown": {...}, ...}
{"url": "https://httpbin.org/html", "success": true, "markdown": {...}, ...}
{"status": "completed"}
```

**Python streaming example:**
```python
import httpx
import json

HEADERS = {"X-API-Key": "YOUR_API_KEY"}

async with httpx.AsyncClient() as client:
    async with client.stream(
        "POST",
        "https://crawl4ai-production-0a7e.up.railway.app/crawl/stream",
        headers=HEADERS,
        json={"urls": ["https://example.com"]}
    ) as response:
        async for line in response.aiter_lines():
            if line:
                data = json.loads(line)
                if data.get("status") == "completed":
                    break
                print(f"Got: {data['url']}")
```

---

### POST /md

Generate markdown from a URL with optional content filtering.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | str | required | URL to fetch |
| f | FilterType | "fit" | Filter: `fit`, `raw`, `bm25`, `llm` |
| q | str | null | Query for BM25/LLM filters |
| c | str | "0" | Cache control ("1" = enabled) |
| provider | str | null | LLM provider override |
| temperature | float | null | LLM temperature (0.0-2.0) |
| base_url | str | null | LLM API base URL override |

**FilterType values:**
- `raw` - No filtering, raw markdown
- `fit` - Pruning filter (removes boilerplate)
- `bm25` - BM25 relevance filtering (requires `q`)
- `llm` - LLM-based filtering (requires `q`)

**curl:**
```bash
# Basic fit markdown
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/md \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com", "f": "fit"}'

# BM25 filtered
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/md \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://news.ycombinator.com", "f": "bm25", "q": "artificial intelligence"}'
```

**Response:**
```json
{
  "url": "https://example.com",
  "filter": "fit",
  "query": null,
  "cache": "0",
  "markdown": "# Example Domain\n\nThis domain is for...",
  "success": true
}
```

---

### POST /map

Discover all URLs on a website. Uses sitemap first, falls back to crawling.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | str | required | Base URL to map |
| max_urls | int | 5000 | Maximum URLs (1-50000) |
| include_subdomains | bool | false | Include subdomain URLs |
| search | str | null | Filter URLs containing string |
| ignore_sitemap | bool | false | Skip sitemap, use crawl |
| max_depth | int | 3 | Max crawl depth (1-10) |

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/map \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com", "max_urls": 100}'
```

**Response:**
```json
{
  "success": true,
  "url": "https://example.com",
  "urls": [
    "https://example.com/",
    "https://example.com/page1",
    "https://example.com/page2"
  ],
  "count": 3,
  "source": "sitemap",
  "time_seconds": 0.854
}
```

---

### POST /html

Get preprocessed HTML optimized for schema extraction.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | str | required | URL to crawl |

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/html \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com"}'
```

**Response:**
```json
{
  "html": "<html><!-- preprocessed HTML -->...</html>",
  "url": "https://example.com",
  "success": true
}
```

---

### POST /screenshot

Capture a full-page PNG screenshot.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | str | required | URL to capture |
| screenshot_wait_for | float | 2 | Delay before capture (seconds) |
| output_path | str | null | Path to save file (recommended) |

**curl:**
```bash
# Return base64-encoded screenshot
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/screenshot \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com"}'

# Save to file (server-side path)
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/screenshot \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com", "output_path": "/tmp/screenshot.png"}'
```

**Response (no output_path):**
```json
{
  "success": true,
  "screenshot": "iVBORw0KGgoAAAANSUhEUgAAA..."
}
```

**Response (with output_path):**
```json
{
  "success": true,
  "path": "/tmp/screenshot.png"
}
```

---

### POST /pdf

Generate a PDF document of a webpage.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | str | required | URL to render |
| output_path | str | null | Path to save file (recommended) |

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/pdf \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com", "output_path": "/tmp/page.pdf"}'
```

**Response:**
```json
{
  "success": true,
  "path": "/tmp/page.pdf"
}
```

---

### POST /execute_js

Execute JavaScript snippets on a webpage and return full crawl result.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | str | required | URL to load |
| scripts | List[str] | required | JS snippets to execute |

**Important:** Each script should be an expression that returns a value (IIFE or async function).

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/execute_js \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "url": "https://example.com",
    "scripts": [
      "return document.title",
      "return Array.from(document.querySelectorAll(\"a\")).map(a => a.href)"
    ]
  }'
```

**Response:** Full CrawlResult with `js_execution_result` containing script outputs.

---

### GET /llm/{url}

Q&A endpoint using LLM with crawled content as context.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| q | str | yes | Question to answer |

**curl:**
```bash
curl "https://crawl4ai-production-0a7e.up.railway.app/llm/https://example.com?q=What+is+this+page+about" \
  -H "X-API-Key: YOUR_API_KEY"
```

**Response:**
```json
{
  "answer": "This page is about example domain usage for documentation..."
}
```

---

### GET /ask

BM25-powered documentation search for Crawl4AI library context.

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| context_type | str | "all" | `code`, `doc`, or `all` |
| query | str | null | Search query (recommended) |
| score_ratio | float | 0.5 | Min score as fraction of max |
| max_results | int | 20 | Maximum chunks returned |

**curl:**
```bash
# Search for extraction strategy info
curl "https://crawl4ai-production-0a7e.up.railway.app/ask?query=extraction+strategy&context_type=doc" \
  -H "X-API-Key: YOUR_API_KEY"

# Get all code context (large response)
curl "https://crawl4ai-production-0a7e.up.railway.app/ask?context_type=code" \
  -H "X-API-Key: YOUR_API_KEY"
```

**Response:**
```json
{
  "doc_results": [
    {"text": "## Extraction Strategies\n...", "score": 12.5},
    {"text": "### LLMExtractionStrategy\n...", "score": 10.2}
  ]
}
```

---

## Async Job Endpoints

### POST /crawl/job

Submit asynchronous crawl job. Returns immediately with task ID.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| urls | List[HttpUrl] | required | URLs to crawl |
| browser_config | Dict | {} | Browser configuration |
| crawler_config | Dict | {} | Crawler configuration |
| webhook_config | WebhookConfig | null | Webhook notification config |

**WebhookConfig Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| webhook_url | HttpUrl | required | URL to POST completion |
| webhook_data_in_payload | bool | false | Include full results in webhook |
| webhook_headers | Dict[str, str] | null | Custom headers for webhook |

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/crawl/job \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "urls": ["https://example.com"],
    "webhook_config": {
      "webhook_url": "https://myapp.com/webhook",
      "webhook_data_in_payload": false
    }
  }'
```

**Response:**
```json
{
  "task_id": "crawl_a1b2c3d4"
}
```

---

### GET /crawl/job/{task_id}

Poll crawl job status and retrieve results.

**Response (processing):**
```json
{
  "task_id": "crawl_a1b2c3d4",
  "status": "processing",
  "created_at": "2025-01-15T10:30:00.000000",
  "url": "[\"https://example.com\"]"
}
```

**Response (completed):**
```json
{
  "task_id": "crawl_a1b2c3d4",
  "status": "completed",
  "created_at": "2025-01-15T10:30:00.000000",
  "url": "[\"https://example.com\"]",
  "result": {
    "success": true,
    "results": [...]
  }
}
```

**Response (failed):**
```json
{
  "task_id": "crawl_a1b2c3d4",
  "status": "failed",
  "error": "Connection timeout"
}
```

---

### POST /llm/job

Submit asynchronous LLM extraction job.

**Request Schema:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| url | HttpUrl | required | URL to extract from |
| q | str | required | Extraction instruction |
| schema | str | null | JSON schema for extraction |
| cache | bool | false | Enable caching |
| provider | str | null | LLM provider override |
| temperature | float | null | LLM temperature |
| base_url | str | null | LLM API base URL |
| webhook_config | WebhookConfig | null | Webhook notification |

**curl:**
```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/llm/job \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "url": "https://example.com/article",
    "q": "Extract the title and main points",
    "provider": "openai/gpt-4o-mini"
  }'
```

**Response:**
```json
{
  "task_id": "llm_1234567890_12345",
  "status": "processing",
  "url": "https://example.com/article"
}
```

---

### GET /llm/job/{task_id}

Poll LLM job status and retrieve extraction results.

---

## Utility Endpoints

### GET /health

Health check endpoint.

**Response:**
```json
{
  "status": "ok",
  "timestamp": 1705312200.123,
  "version": "0.5.1-d1"
}
```

---

### GET /schema

Get default configuration schemas.

**Response:**
```json
{
  "browser": {
    "headless": true,
    "verbose": false,
    "viewport_width": 1080,
    "viewport_height": 600,
    ...
  },
  "crawler": {
    "cache_mode": "bypass",
    "stream": false,
    "screenshot": false,
    "pdf": false,
    ...
  }
}
```

---

### POST /config/dump

Convert Python config expression to JSON (utility for SDK/playground).

**Request:**
```json
{
  "code": "CrawlerRunConfig(cache_mode=CacheMode.BYPASS, screenshot=True)"
}
```

**Response:** Serialized config as JSON dict.

---

### GET /hooks/info

Get available hook points and their signatures.

**Response:**
```json
{
  "available_hooks": {
    "on_browser_created": {
      "parameters": ["browser"],
      "description": "Called after browser instance is created",
      "example": "..."
    },
    "on_page_context_created": {
      "parameters": ["page", "context"],
      "description": "Called after page and context are created - ideal for authentication",
      "example": "async def hook(page, context, **kwargs):\n    await context.add_cookies([...])\n    return page"
    },
    "before_goto": {...},
    "after_goto": {...},
    "on_user_agent_updated": {...},
    "on_execution_started": {...},
    "before_retrieve_html": {...},
    "before_return_html": {...}
  },
  "timeout_limits": {"min": 1, "max": 120, "default": 30}
}
```

---

### POST /token

Generate JWT access token (requires JWT enabled).

**Request:**
```json
{
  "email": "user@example.com"
}
```

**Response:**
```json
{
  "email": "user@example.com",
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
  "token_type": "bearer"
}
```

---

### GET /metrics

Prometheus metrics endpoint (redirects to configured endpoint).

---

# MCP Server

## Overview

The Model Context Protocol (MCP) enables AI models to access Crawl4AI's capabilities through a standardized interface.

## Connection Endpoints

| Transport | URL | Description |
|-----------|-----|-------------|
| SSE | `https://crawl4ai-production-0a7e.up.railway.app/mcp/sse` | Server-Sent Events (recommended) |
| WebSocket | `wss://crawl4ai-production-0a7e.up.railway.app/mcp/ws` | WebSocket transport |
| Schema | `https://crawl4ai-production-0a7e.up.railway.app/mcp/schema` | Tool/resource schemas |

## Claude Code Integration

```bash
# Add MCP provider
claude mcp add --transport sse c4ai-sse https://crawl4ai-production-0a7e.up.railway.app/mcp/sse

# List providers
claude mcp list

# Remove provider
claude mcp remove c4ai-sse
```

## Available MCP Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `crawl` | Crawl URLs and return results | urls, browser_config, crawler_config |
| `md` | Generate markdown with filters | url, f, q, c |
| `map` | Discover site URLs | url, max_urls, include_subdomains, search |
| `html` | Get preprocessed HTML | url |
| `screenshot` | Capture page screenshot | url, screenshot_wait_for, output_path |
| `pdf` | Generate PDF document | url, output_path |
| `execute_js` | Execute JavaScript on page | url, scripts |
| `ask` | Search Crawl4AI documentation | context_type, query, score_ratio |

## MCP Schema Endpoint

Get full tool definitions:

```bash
curl https://crawl4ai-production-0a7e.up.railway.app/mcp/schema \
  -H "X-API-Key: YOUR_API_KEY"
```

**Response:**
```json
{
  "tools": [
    {
      "name": "crawl",
      "description": "Crawl a list of URLs...",
      "inputSchema": {...}
    },
    ...
  ],
  "resources": [],
  "resource_templates": []
}
```

## MCP Client Example (Python)

```python
import asyncio
import json
import websockets

async def mcp_call():
    async with websockets.connect("wss://crawl4ai-production-0a7e.up.railway.app/mcp/ws") as ws:
        # Initialize
        await ws.send(json.dumps({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "initialize",
            "params": {"capabilities": {}}
        }))
        await ws.recv()  # initialization response

        # Call tool
        await ws.send(json.dumps({
            "jsonrpc": "2.0",
            "id": 2,
            "method": "tools/call",
            "params": {
                "name": "md",
                "arguments": {"url": "https://example.com", "f": "fit"}
            }
        }))
        result = await ws.recv()
        print(json.loads(result))

asyncio.run(mcp_call())
```

---

# Async Jobs & Webhooks

## Job Lifecycle

```
1. Submit Job     POST /crawl/job or /llm/job
        |
        v
2. Get Task ID    {"task_id": "crawl_xyz"}
        |
        v
3. Job Runs       (background processing)
        |
        v
4. Completion     Webhook fired OR poll status
        |
        v
5. Get Results    GET /crawl/job/{task_id}
```

## WebhookConfig Schema

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| webhook_url | HttpUrl | required | POST destination |
| webhook_data_in_payload | bool | false | Include full results |
| webhook_headers | Dict[str, str] | null | Custom headers |

## WebhookPayload Format

```json
{
  "task_id": "crawl_a1b2c3d4",
  "task_type": "crawl",
  "status": "completed",
  "timestamp": "2025-01-15T10:30:00.000000+00:00",
  "urls": ["https://example.com"],
  "error": null,
  "data": null
}
```

With `webhook_data_in_payload: true`:
```json
{
  "task_id": "crawl_a1b2c3d4",
  "task_type": "crawl",
  "status": "completed",
  "timestamp": "2025-01-15T10:30:00.000000+00:00",
  "urls": ["https://example.com"],
  "data": {
    "success": true,
    "results": [...]
  }
}
```

## Webhook Retry Logic

- **Max attempts:** 5
- **Backoff:** Exponential (1s -> 2s -> 4s -> 8s -> 16s)
- **Client errors (4xx):** No retry
- **Server errors (5xx):** Retry with backoff
- **Timeout:** 30 seconds per attempt

## Task Status Values

| Status | Description |
|--------|-------------|
| `processing` | Job is running |
| `completed` | Job finished successfully |
| `failed` | Job encountered an error |

---

# Configuration Reference

## Request Config Structure

Non-primitive config values use `{"type": "ClassName", "params": {...}}` format:

```json
{
  "urls": ["https://example.com"],
  "browser_config": {
    "type": "BrowserConfig",
    "params": {
      "headless": true,
      "viewport": {
        "type": "dict",
        "value": {"width": 1200, "height": 800}
      }
    }
  },
  "crawler_config": {
    "type": "CrawlerRunConfig",
    "params": {
      "cache_mode": "bypass",
      "extraction_strategy": {
        "type": "JsonCssExtractionStrategy",
        "params": {
          "schema": {
            "type": "dict",
            "value": {
              "baseSelector": "article",
              "fields": [{"name": "title", "selector": "h1", "type": "text"}]
            }
          }
        }
      }
    }
  }
}
```

---

# Error Handling

## HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 202 | Accepted (async job created) |
| 400 | Bad request (invalid parameters) |
| 401 | Unauthorized (auth required/failed) |
| 404 | Not found (task/resource missing) |
| 429 | Rate limit exceeded |
| 500 | Server error |

## Error Response Format

```json
{
  "detail": "Error message describing the problem"
}
```

For crawl errors with memory info:
```json
{
  "detail": "{\"error\": \"Connection timeout\", \"server_memory_delta_mb\": 15.2}"
}
```

---

# Examples

## Complete Python Client

```python
import asyncio
import requests
import json

BASE_URL = "https://crawl4ai-production-0a7e.up.railway.app"
HEADERS = {
    "Content-Type": "application/json",
    "X-API-Key": "YOUR_API_KEY"
}

# Simple crawl
def simple_crawl():
    response = requests.post(f"{BASE_URL}/crawl", headers=HEADERS, json={
        "urls": ["https://example.com"]
    })
    data = response.json()
    if data["success"]:
        print(data["results"][0]["markdown"]["fit_markdown"][:500])

# Crawl with extraction
def crawl_with_extraction():
    response = requests.post(f"{BASE_URL}/crawl", headers=HEADERS, json={
        "urls": ["https://news.ycombinator.com"],
        "crawler_config": {
            "type": "CrawlerRunConfig",
            "params": {
                "extraction_strategy": {
                    "type": "JsonCssExtractionStrategy",
                    "params": {
                        "schema": {
                            "type": "dict",
                            "value": {
                                "name": "HN Posts",
                                "baseSelector": ".athing",
                                "fields": [
                                    {"name": "title", "selector": ".titleline > a", "type": "text"},
                                    {"name": "link", "selector": ".titleline > a", "type": "attribute", "attribute": "href"}
                                ]
                            }
                        }
                    }
                }
            }
        }
    })
    data = response.json()
    if data["success"]:
        extracted = json.loads(data["results"][0]["extracted_content"])
        for item in extracted[:5]:
            print(f"- {item['title']}")

# Async job with polling
def async_crawl_with_polling():
    # Submit job
    response = requests.post(f"{BASE_URL}/crawl/job", headers=HEADERS, json={
        "urls": ["https://example.com"]
    })
    task_id = response.json()["task_id"]
    print(f"Task: {task_id}")

    # Poll for completion
    import time
    while True:
        status = requests.get(f"{BASE_URL}/crawl/job/{task_id}", headers=HEADERS).json()
        if status["status"] == "completed":
            print("Done:", status["result"]["results"][0]["url"])
            break
        elif status["status"] == "failed":
            print("Failed:", status.get("error"))
            break
        time.sleep(1)

if __name__ == "__main__":
    simple_crawl()
```

## Streaming Crawl (httpx)

```python
import asyncio
import httpx
import json

BASE_URL = "https://crawl4ai-production-0a7e.up.railway.app"
HEADERS = {"X-API-Key": "YOUR_API_KEY"}

async def stream_crawl():
    async with httpx.AsyncClient() as client:
        async with client.stream(
            "POST",
            f"{BASE_URL}/crawl/stream",
            headers=HEADERS,
            json={
                "urls": [
                    "https://example.com",
                    "https://httpbin.org/html"
                ],
                "crawler_config": {
                    "type": "CrawlerRunConfig",
                    "params": {"stream": True}
                }
            },
            timeout=120.0
        ) as response:
            async for line in response.aiter_lines():
                if line:
                    data = json.loads(line)
                    if data.get("status") == "completed":
                        print("Stream complete")
                        break
                    print(f"Got: {data['url']} - {data['success']}")

asyncio.run(stream_crawl())
```

## Webhook Integration (Flask)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/webhook/crawl", methods=["POST"])
def handle_crawl_webhook():
    payload = request.json

    task_id = payload["task_id"]
    status = payload["status"]

    if status == "completed":
        print(f"Task {task_id} completed")
        if "data" in payload:
            # Full results included
            results = payload["data"]["results"]
            for r in results:
                print(f"  - {r['url']}: {r['success']}")
    else:
        print(f"Task {task_id} failed: {payload.get('error')}")

    return jsonify({"received": True})

if __name__ == "__main__":
    app.run(port=5000)
```

## curl Examples

```bash
# Health check (no auth required)
curl https://crawl4ai-production-0a7e.up.railway.app/health

# Simple markdown
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/md \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com", "f": "fit"}'

# Site mapping
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/map \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://docs.python.org", "max_urls": 50}'

# Screenshot
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/screenshot \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{"url": "https://example.com"}'

# LLM Q&A
curl "https://crawl4ai-production-0a7e.up.railway.app/llm/https://example.com?q=What+is+this+page+about" \
  -H "X-API-Key: YOUR_API_KEY"

# Async job with webhook
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/crawl/job \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "urls": ["https://example.com"],
    "webhook_config": {
      "webhook_url": "https://myapp.com/webhook/crawl",
      "webhook_data_in_payload": true
    }
  }'
```

## Using Hooks

```bash
curl -X POST https://crawl4ai-production-0a7e.up.railway.app/crawl \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "urls": ["https://example.com"],
    "hooks": {
      "code": {
        "before_retrieve_html": "async def hook(page, context, **kwargs):\n    await page.evaluate(\"window.scrollTo(0, document.body.scrollHeight)\")\n    await page.wait_for_timeout(2000)\n    return page"
      },
      "timeout": 30
    }
  }'
```

---

## SDK Reference

For the Python SDK (`Crawl4aiDockerClient`), see the main SDK documentation or use:

```python
from crawl4ai.docker_client import Crawl4aiDockerClient

async with Crawl4aiDockerClient(
    base_url="https://crawl4ai-production-0a7e.up.railway.app",
    api_key="YOUR_API_KEY",
    timeout=60.0,
    verbose=True
) as client:
    # Sync-style crawl
    results = await client.crawl(["https://example.com"])

    # With configs
    from crawl4ai import BrowserConfig, CrawlerRunConfig
    results = await client.crawl(
        ["https://example.com"],
        browser_config=BrowserConfig(headless=True),
        crawler_config=CrawlerRunConfig(cache_mode="bypass")
    )

    # Get schema
    schema = await client.get_schema()
```
