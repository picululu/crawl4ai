# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Crawl4AI is an open-source, LLM-friendly web crawler and scraper that converts web pages into clean Markdown for RAG, agents, and data pipelines. It's async-first, uses Playwright for browser automation, and supports structured data extraction.

## Common Commands

### Installation & Setup
```bash
pip install -e .                    # Development install
pip install -e ".[all]"             # With all optional features
crawl4ai-setup                      # Post-installation browser setup
crawl4ai-doctor                     # Verify installation
python -m playwright install chromium  # Manual browser install if needed
```

### Running Tests
```bash
# Run all tests
pytest tests/

# Run specific test file
pytest tests/async/test_basic_crawling.py

# Run single test
pytest tests/async/test_basic_crawling.py::test_successful_crawl -v

# Tests require pytest-asyncio for async tests
```

### CLI Usage
```bash
crwl https://example.com -o markdown           # Basic crawl
crwl https://example.com --deep-crawl bfs --max-pages 10  # Deep crawl
crwl https://example.com -q "Extract prices"   # LLM extraction
```

### Docker
```bash
docker pull unclecode/crawl4ai:latest
docker run -d -p 11235:11235 --name crawl4ai --shm-size=1g unclecode/crawl4ai:latest
# Dashboard: http://localhost:11235/dashboard
# Playground: http://localhost:11235/playground
```

## Architecture

### Core Package Structure (`crawl4ai/`)

**Main Entry Point:**
- `AsyncWebCrawler` in `async_webcrawler.py` - Primary crawler class, use as async context manager or with explicit `start()`/`close()`

**Configuration:**
- `async_configs.py` - Contains `BrowserConfig`, `CrawlerRunConfig`, `LLMConfig`, `ProxyConfig`, and other config classes
- `config.py` - Default constants and settings

**Browser Management:**
- `async_crawler_strategy.py` - `AsyncPlaywrightCrawlerStrategy` handles browser interactions
- `browser_manager.py` - Browser lifecycle management with pooling
- `browser_adapter.py` - Adapters for Playwright and undetected browsers
- `browser_profiler.py` - Persistent browser profile management

**Content Processing Pipeline:**
1. `content_scraping_strategy.py` - HTML parsing (`LXMLWebScrapingStrategy`)
2. `markdown_generation_strategy.py` - `DefaultMarkdownGenerator` converts HTML to Markdown
3. `content_filter_strategy.py` - `PruningContentFilter`, `BM25ContentFilter` for noise removal
4. `extraction_strategy.py` - `LLMExtractionStrategy`, `JsonCssExtractionStrategy`, `CosineStrategy` for structured extraction
5. `table_extraction.py` - Table extraction including `LLMTableExtraction`

**Deep Crawling (`deep_crawling/`):**
- `bfs_strategy.py` - Breadth-first crawling
- `dfs_strategy.py` - Depth-first crawling
- `bff_strategy.py` - Best-first (priority-based) crawling
- `filters.py` - URL filters (`URLPatternFilter`, `DomainFilter`, `ContentRelevanceFilter`)
- `scorers.py` - Link scoring strategies

**Dispatchers (`async_dispatcher.py`):**
- `MemoryAdaptiveDispatcher` - Memory-aware concurrent crawling
- `SemaphoreDispatcher` - Simple concurrent crawling with semaphore

**Models (`models.py`):**
- `CrawlResult` - Main result object with `markdown`, `html`, `extracted_content`, etc.
- `MarkdownGenerationResult` - Contains `raw_markdown`, `fit_markdown`, `markdown_with_citations`

### Docker Server (`deploy/docker/`)

Production-ready FastAPI server with:
- `server.py` - Main FastAPI app with lifecycle management
- `api.py` - REST endpoints (`/crawl`, `/html`, `/md`, `/screenshot`, `/pdf`, `/llm`)
- `crawler_pool.py` - 3-tier browser pool (permanent/hot/cold)
- `monitor.py`, `monitor_routes.py` - Real-time monitoring system
- `webhook.py` - Webhook delivery system

### Key Patterns

**Using the Crawler:**
```python
async with AsyncWebCrawler(config=BrowserConfig()) as crawler:
    result = await crawler.arun(url="https://example.com", config=CrawlerRunConfig())
    print(result.markdown.raw_markdown)
    print(result.markdown.fit_markdown)  # Filtered content
```

**Extraction Strategies:**
```python
# LLM extraction
config = CrawlerRunConfig(
    extraction_strategy=LLMExtractionStrategy(
        llm_config=LLMConfig(provider="openai/gpt-4o"),
        schema=MyPydanticModel.schema()
    )
)

# CSS-based extraction (no LLM)
config = CrawlerRunConfig(
    extraction_strategy=JsonCssExtractionStrategy(schema={...})
)
```

**Content Filtering:**
```python
config = CrawlerRunConfig(
    markdown_generator=DefaultMarkdownGenerator(
        content_filter=PruningContentFilter(threshold=0.48)
        # or: BM25ContentFilter(user_query="your query")
    )
)
```

**Deep Crawling:**
```python
config = CrawlerRunConfig(
    deep_crawl_strategy=BFSDeepCrawlStrategy(max_depth=3, max_pages=50)
)
```

## Git Commit Guidelines

- Do NOT add "Co-Authored-By: Claude" or any AI attribution to commits
- A commit-msg hook is in place to enforce this

## Important Notes

- Python 3.10+ required
- All browser automation uses Playwright (async)
- LLM calls go through LiteLLM for provider flexibility
- The `markdown` property on `CrawlResult` returns a string-like object that also has `.raw_markdown`, `.fit_markdown`, `.markdown_with_citations` attributes
- `CacheMode.BYPASS` skips cache, `CacheMode.ENABLED` uses cache
- Tests are in `tests/` with subdirectories for different test categories (`async/`, `docker/`, `general/`, etc.)
