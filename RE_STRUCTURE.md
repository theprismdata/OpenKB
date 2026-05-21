# OpenKB Architecture Analysis

## Overview

OpenKB v0.1.3 (Alpha) is an LLM-powered wiki compilation system that transforms documents into persistent, self-organizing knowledge bases. It uses a wiki-style approach rather than traditional RAG, where LLMs generate summaries, concept pages, and cross-references that compound over time.

**License**: Apache 2.0
**Python**: 3.10+
**Total source code**: ~4,800 lines

---

## Architecture Layers

```
CLI (entry point)
  ↓
Configuration Layer (config.py)
  ↓
Ingestion Layer (converter.py → indexer.py → PageIndex)
  ↓
Agent Layer (compiler → query/linter → chat)
  ↓
Wiki File System (.openkb/wiki/)
```

### 1. CLI Layer — `openkb/cli.py` (726 lines)
- Command routing for `init`, `add`, `search`, `compile`, `lint`, `chat`, `watch`
- Workflow orchestration: add → compile → chat pipeline
- Session management with prompt_toolkit

### 2. Configuration Layer — `openkb/config.py` (62 lines)
- YAML-based configuration management
- Defaults: `model: gpt-5.4-mini`, `language: en`, `pageindex_threshold: 20`
- Schema validation for wiki structure

### 3. Ingestion Layer
- **`openkb/converter.py`** (112 lines): Hash-based dedup + format routing (markitdown for text/markdown, pymupdf for PDF)
- **`openkb/indexer.py`** (113 lines): PageIndex integration with 3-retry TOC extraction and cloud/local fallback
- PageIndex threshold: 20 pages (triggers tree indexing for long documents)

### 4. Agent Layer
- **`openkb/agent/compiler.py`** (811 lines) — Most complex module: 4-step wiki compilation pipeline (summary → concept plan → concurrent generation → index update)
- **`openkb/agent/query.py`** (221 lines): OpenAI Agents SDK integration with 3 retrieval tools (`read_file`, `get_page_content`, `get_image`)
- **`openkb/agent/linter.py`** (108 lines): Semantic quality checks
- **`openkb/agent/chat.py`** (605 lines): prompt_toolkit REPL with streaming Rich console output
- **`openkb/agent/tools.py`** (191 lines): Wiki file I/O for agents
- **`openkb/agent/chat_session.py`** (280 lines): Session persistence with atomic writes
- **`openkb/agent/_markdown.py`** (371 lines): Terminal-style Markdown rendering

### 5. Utility Layer
- `openkb/images.py` (247 lines): Image extraction/processing
- `openkb/watcher.py` (98 lines): File system monitoring
- `openkb/state.py` (64 lines): Hash deduplication
- `openkb/schema.py` (60 lines): AGENTS.md template
- `openkb/tree_renderer.py` (45 lines): Tree-to-markdown rendering
- `openkb/log.py` (21 lines): Append-only operational logging

---

## Wiki Structure

```
.openkb/
├── wiki/
│   ├── summaries/       # LLM-generated document summaries
│   ├── concepts/        # LLM-generated concept pages
│   ├── sources/         # Original converted source files
│   ├── images/          # Extracted images
│   └── index.md         # Wiki index/manifest
├── chats/               # Chat sessions (<id>.json, atomic writes via .tmp + os.replace)
└── config.yaml          # Configuration
```

---

## Key Technical Patterns

### LLM Prompt Caching
- Context A cached across calls in `compiler.py` for token cost efficiency
- Leverages LiteLLM provider-agnostic routing

### Concurrent LLM Calls
- `compiler.py` uses `asyncio.gather()` for parallel concept generation
- Reduces compilation time significantly

### Vectorless Retrieval
- PageIndex provides hierarchical document retrieval (tree indexing)
- Activates when documents exceed 20 pages
- No vector embeddings required

### Atomic Session Writes
- `chat_session.py` uses `.tmp` file + `os.replace()` for crash-safe persistence

### Text-Based Frontmatter
- Obsidian `[[wikilinks]]` compatibility
- Trade-off: fragile parsing via `text.find("---")` and `re.sub`

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `pageindex==0.3.0.dev1` | Vectorless hierarchical document retrieval |
| `markitdown[all]` | Document conversion (PDF, DOCX, etc.) |
| `openai-agents` | Agent framework with retrieval tools |
| `litellm` | Provider-agnostic LLM routing |
| `prompt_toolkit>=3.0` | Interactive REPL |
| `rich>=13.0` | Terminal formatting |
| `json-repair` | LLM JSON output repair |

---

## Code Quality Issues

### Critical
1. **`compiler.py:753,804`**: Sync `litellm.completion()` inside `async` functions → event loop blocking
2. **`compiler.py:377-427`**: Text-based frontmatter parsing → fragile edge cases

### High
3. **`chat.py:320`**: Unbounded session history accumulation → context window overflow risk
4. **`lint.py:589`**: `--fix` flag present but "not yet implemented"

### Low
5. **`indexer.py:70-72`**: Debug `logger.info` left in production code
6. **`cli.py:7-8,34-35`**: Duplicate `warnings.filterwarnings("ignore")` calls

---

## Data Flow

```
User adds document (CLI)
    ↓
converter.py: hash check → format routing → text extraction
    ↓
indexer.py: PageIndex integration → TOC extraction
    ↓
compiler.py: 4-step pipeline
    1. Generate summary
    2. Create concept plan
    3. Concurrent LLM calls for concept pages
    4. Update wiki index + cross-reference linking
    ↓
Wiki files written to .openkb/wiki/
```

---

## Architectural Strengths

- Clear layer separation (CLI → infra → agents → wiki)
- Prompt caching for efficiency
- Concurrent LLM calls for speed
- Obsidian `[[wikilinks]]` compatibility
- Atomic writes for crash safety
- Provider-agnostic LLM routing via LiteLLM

## Technical Debt

- Sync LLM calls blocking async event loop
- Fragile frontmatter manipulation (should use `yaml` library)
- No session history truncation policy
- Unimplemented lint features
- Debug logs in production code
