# OpenKB Architecture Documentation

## Overview

**OpenKB** is an open-source LLM Knowledge Base system (Python 3.10+, Apache 2.0) that compiles raw documents into a structured, interlinked wiki-style knowledge base using LLMs, powered by [PageIndex](https://github.com/VectifyAI/PageIndex) for vectorless long document retrieval.

The idea is based on a concept described by Andrej Karpathy: LLMs generate summaries, concept pages, and cross-references, all maintained automatically. Knowledge compounds over time instead of being re-derived on every query.

### Positioning vs Traditional RAG

| | Traditional RAG | OpenKB |
|---|---|---|
| Retrieval | Vector similarity search every query | Wiki with existing cross-references |
| Accumulation | Nothing accumulates between queries | Knowledge compounds over time |
| Contradictions | Not detected | Lint agent flags them |
| Synthesis | Per-query, no memory | Reflects everything consumed |

---

## Directory Structure

```
OpenKB/
├── openkb/                    # Main package
│   ├── __init__.py            # Version info
│   ├── __main__.py            # Entry point (python -m openkb)
│   ├── cli.py                 # CLI commands (Click framework)
│   ├── config.py              # Configuration management (YAML)
│   ├── converter.py           # Document conversion pipeline
│   ├── indexer.py             # PageIndex indexer for long documents
│   ├── images.py              # Image extraction from PDFs/markdown
│   ├── lint.py                # Structural lint checks
│   ├── log.py                 # Append-only operation log
│   ├── schema.py              # Wiki schema (AGENTS.md template)
│   ├── state.py               # Hash registry (deduplication)
│   ├── tree_renderer.py       # PageIndex tree -> Markdown renderer
│   ├── watcher.py             # Filesystem watcher (watch mode)
│   └── agent/                 # Agent submodule
│       ├── __init__.py
│       ├── compiler.py        # Wiki compilation pipeline (core)
│       ├── query.py           # Q&A agent builder
│       ├── chat.py            # Interactive chat REPL
│       ├── chat_session.py    # Session persistence
│       ├── linter.py          # Knowledge lint agent
│       ├── tools.py           # Wiki tool functions
│       └── _markdown.py       # Markdown terminal renderer
├── tests/                     # pytest test suite
├── pyproject.toml             # Build config (hatchling)
└── README.md
```

---

## Data Flow

### Document Ingestion Pipeline

```
raw/ (user drops files)
  |
  v
cli.py:add_single_file()
  |
  +-- HashRegistry.check() -> skip if already known
  |
  v
converter.py:convert_document()
  |
  +-- PDF (>= threshold pages) -> PageIndex indexing
  |                              |
  |                              v
  |                           indexer.py:index_long_document()
  |                              +-- PageIndexClient.collection().add()
  |                              +-- col.get_document() -> tree structure
  |                              +-- convert_pdf_to_pages() -> sources/{doc}.json
  |                              +-- render_summary_md() -> wiki/summaries/{doc}.md
  |                              (doc_id passed to compiler)
  |
  +-- Markdown -> copy relative images -> wiki/sources/{doc}.md
  |
  +-- Other (docx, pptx, html, etc.) -> MarkItDown -> wiki/sources/{doc}.md
                                       extract_base64_images()
  |
  v
compiler.py:compile_long_doc() | compile_short_doc()
  |
  +-- Step 1: Generate summary (LLM call)
  |
  +-- Step 2: Concepts plan (LLM call - create/update/related)
  |
  +-- Step 3: Generate/update concept pages (concurrent LLM calls, cached context)
  |
  +-- Step 3b: Add related links (code-only)
  |
  +-- Step 3c: Backlink summary <-> concepts (code-only)
  |
  +-- Step 4: Update index.md (code-only)
  |
  v
HashRegistry.add() + log.md append
```

### Query Flow

```
cli.py:query() or chat.py:run_chat()
  |
  v
query.py:build_query_agent()
  |
  +-- Agent("wiki-query") with 3 tools:
  |   +-- read_file()      -> Read wiki Markdown files
  |   +-- get_page_content() -> Fetch specific pages from PageIndex docs
  |   +-- get_image()      -> View images from wiki
  |
  v
Runner.run() / run_streamed()
  |
  +-- Agent reads index.md -> identifies relevant documents
  +-- Reads summaries/ -> gets document overviews
  +-- Reads concepts/ -> gets cross-document synthesis
  +-- read_file() -> detailed source content
  +-- get_page_content() -> PageIndex page ranges
  +-- get_image() -> figures/diagrams
  |
  v
Synthesized answer (streamed to stdout)
```

---

## Core Components

### 1. CLI (`cli.py` - 726 lines)

Click-based CLI with these commands:

| Command | Description |
|---|---|
| `openkb init` | Interactive KB initialization (model, API key, directory structure) |
| `openkb add <path>` | Add document(s) - single file or directory |
| `openkb query "question"` | One-shot Q&A over the knowledge base |
| `openkb chat` | Interactive multi-turn chat with slash commands |
| `openkb watch` | Watch `raw/` directory for auto-compilation |
| `openkb lint` | Structural + knowledge health checks |
| `openkb list` | List indexed documents and concepts |
| `openkb status` | Show KB stats |
| `openkb use <path>` | Set default KB |

KB discovery: explicit override -> walk up from cwd -> global config `default_kb`.

### 2. Configuration (`config.py` - 62 lines)

Two-tier YAML config:

- **KB-local**: `.openkb/config.yaml` - model, language, pageindex_threshold
- **Global**: `~/.config/openkb/global.yaml` - known_kbs, default_kb

Merges with `DEFAULT_CONFIG` (model: `gpt-5.4-mini`, language: `en`, pageindex_threshold: 20).

API key loading: system env -> KB-local `.env` -> global `.env` (first wins).

### 3. Document Conversion (`converter.py` - 112 lines)

Hash-check deduplication -> format-specific conversion:

- **Markdown**: Direct read + relative image processing
- **PDF (short)**: pymupdf dict-mode with inline images
- **PDF (long)**: Returns `is_long_doc=True` for PageIndex path
- **Other (docx, pptx, html, xlsx, csv)**: MarkItDown + base64 image extraction

### 4. PageIndex Indexer (`indexer.py` - 113 lines)

For long documents (PDFs >= threshold pages):

1. `PageIndexClient` with `IndexConfig` (TOC, node summaries, doc description)
2. `col.add()` - retry up to 3 times (stochastic TOC accuracy)
3. `col.get_document()` -> tree structure + metadata
4. `convert_pdf_to_pages()` -> per-page JSON with text + images
5. `render_summary_md()` -> Markdown summary with YAML frontmatter

### 5. Wiki Compilation (`agent/compiler.py` - 811 lines)

**The core of OpenKB.** 4-step LLM pipeline with prompt caching:

#### Step 1: Generate Summary
- System prompt: AGENTS.md schema + language
- User prompt: document full text
- Output: JSON `{brief, content}` -> `wiki/summaries/{doc}.md`

#### Step 2: Concepts Plan
- Reads existing concept briefs (frontmatter `brief:` or first 150 chars)
- LLM decides: create / update / related
- Output: JSON `{create: [...], update: [...], related: [...]}`

#### Step 3: Concept Generation (Concurrent)
- `asyncio.Semaphore(max_concurrency=5)` for parallel LLM calls
- **Create**: Generate new concept page from scratch
- **Update**: Read existing content -> rewrite with new info integrated
- Context A (schema + doc) is cached across calls via LiteLLM prompt caching

#### Step 3b: Related Links (Code-only)
- Adds `[[summaries/{doc}]]` to related concept pages
- No LLM call needed

#### Step 3c: Backlinks (Code-only)
- Summary -> concept wikilinks
- Concept -> summary wikilinks
- Bidirectional link closure

#### Step 4: Index Update (Code-only)
- Appends document entry to `wiki/index.md` (with dedup check)
- Appends concept entries to `wiki/index.md`

### 6. Q&A Agent (`agent/query.py` - 221 lines)

Builds an OpenAI Agents SDK `Agent` with 3 tools:

- `read_file(path)` -> Read wiki Markdown (with path traversal protection)
- `get_page_content(doc_name, pages)` -> PageIndex page ranges (e.g. "3-5,7")
- `get_image(image_path)` -> Base64 image URL for figures/diagrams

Search strategy: index.md -> summaries -> concepts -> sources -> images.

### 7. Chat REPL (`agent/chat.py` - 605 lines)

Built on `prompt_toolkit`:
- History, tab completion, bottom toolbar
- Slash commands: `/help`, `/status`, `/list`, `/add`, `/save`, `/clear`, `/lint`, `/exit`
- Streaming output with Rich console
- Markdown rendering in terminal style (markdown-it + Rich primitives)
- Session persistence (see chat_session.py)

### 8. Session Management (`agent/chat_session.py` - 280 lines)

Each session: `<kb>/.openkb/chats/<id>.json`

- ID format: `YYYYMMDD-HHMMSS-xxx` (timestamp + random suffix)
- History sanitization: strips large base64 image payloads, keeps references
- Atomic writes: `.json.tmp` -> `os.replace()`
- Session resolution: full ID, unique prefix, or `__latest__`

### 9. Linting (`lint.py` + `agent/linter.py`)

**Structural lint** (`lint.py` - 264 lines):
- Broken `[[wikilinks]]` detection
- Orphaned pages (no incoming or outgoing links)
- Raw files without wiki entries
- Index.md sync check

**Knowledge lint** (`agent/linter.py` - 108 lines):
- LLM agent that checks: contradictions, gaps, staleness, redundancy, concept coverage
- Uses same tool set as query agent (read_file, list_files)

### 10. Wiki Tools (`agent/tools.py` - 191 lines)

Plain functions (decorated with `@function_tool` only when building agents):
- `list_wiki_files(directory, wiki_root)`
- `read_wiki_file(path, wiki_root)`
- `get_wiki_page_content(doc_name, pages, wiki_root)`
- `read_wiki_image(path, wiki_root)` -> base64 data URL
- `write_wiki_file(path, content, wiki_root)`
- `parse_pages(pages)` -> "3-5,7,10-12" -> `[3,4,5,7,10,11,12]`

### 11. Watcher (`watcher.py` - 98 lines)

Filesystem monitoring via `watchdog`:
- `DebouncedHandler` - collects events, debounces 2 seconds, fires callback
- Ignores directories and dotfiles
- Callback receives sorted list of affected paths

### 12. Image Handling (`images.py` - 247 lines)

- `extract_pdf_images()` -> pymupdf dict-mode block iteration
- `convert_pdf_to_pages()` -> per-page dicts with text + images
- `convert_pdf_with_images()` -> PDF -> Markdown with inline images
- `extract_base64_images()` -> decode base64, save to disk, rewrite links
- `copy_relative_images()` -> copy local images, rewrite links
- Minimum image dimension filter: 32px

### 13. Markdown Renderer (`agent/_markdown.py` - 371 lines)

Terminal-style Markdown rendering (mirrors claude-code's markdown.ts):
- `markdown-it` parser -> syntax tree -> Rich primitives
- Supports: headings, bold, italic, code blocks, blockquotes, tables, lists
- Code: Monokai theme
- Tables: auto-width with alignment

### 14. Tree Renderer (`tree_renderer.py` - 45 lines)

PageIndex tree -> Markdown for wiki summaries:
- YAML frontmatter with `doc_type: pageindex` + `full_text` path
- Recursive heading generation with page ranges

### 15. Schema (`schema.py` - 60 lines)

`AGENTS.md` - the LLM's instruction manual for wiki maintenance:
- Directory structure definitions
- Page type descriptions
- Index page format
- Log format
- Wikilink conventions

Read from disk at runtime (so user edits take effect immediately).

### 16. Hash Registry (`state.py` - 64 lines)

SHA-256 based deduplication:
- `HashRegistry(path)` - loads from JSON
- `is_known(hash)` - check if already processed
- `add(hash, metadata)` - register + persist immediately
- `hash_file(path)` - static SHA-256 computation

### 17. Operation Log (`log.py` - 21 lines)

Append-only `wiki/log.md` entries:
- Format: `## [YYYY-MM-DD HH:MM:SS] operation | description`
- Operations: ingest, query, lint

---

## Dependencies

| Package | Purpose |
|---|---|
| `pageindex==0.3.0.dev1` | Vectorless hierarchical tree indexing |
| `markitdown[all]` | Universal file-to-markdown conversion |
| `click>=8.0` | CLI framework |
| `watchdog>=3.0` | Filesystem monitoring |
| `litellm` | Multi-provider LLM gateway |
| `openai-agents` | Agent framework (supports non-OpenAI via LiteLLM) |
| `pyyaml` | YAML config parsing |
| `python-dotenv` | Environment variable loading |
| `json-repair` | LLM JSON response repair |
| `prompt_toolkit>=3.0` | Interactive REPL |
| `rich>=13.0` | Terminal formatting |
| `pymupdf` | PDF text/image extraction |
| `markdown-it-py` | Markdown parsing for terminal rendering |

---

## Design Decisions

### Vectorless Retrieval (PageIndex)
Traditional RAG uses vector embeddings for similarity search. OpenKB uses PageIndex's hierarchical tree index - LLMs read the document tree structure (TOC with summaries) for reasoning-based retrieval instead of vector similarity. This avoids embedding costs and context rot.

### Wiki-Style Compilation
Instead of per-query retrieval, documents are compiled into a persistent wiki with summaries, concepts, and cross-references. Knowledge accumulates: each document enriches the existing wiki.

### Prompt Caching
The compiler pipeline reuses base context A (schema + document) across LLM calls. LiteLLM's prompt caching reduces token costs for repeated context.

### Atomic Session Writes
Chat sessions use `.json.tmp` + `os.replace()` for crash-safe persistence.

### Image Handling
Images are extracted from PDFs via pymupdf dict-mode (reading order), saved as PNG, and referenced with wiki-root-relative paths. Base64 images in markdown are decoded and saved to disk.

### Wikilink Convention
Cross-references use `[[path/to/page]]` syntax (Obsidian-compatible). Plain Markdown files, openable in Obsidian for graph view and browsing.

---

## Authors

| Author | Email | Contribution |
|---|---|---|
| Kylin | quanqi@pageindex.ai | PageIndex integration, core architecture |
| Ray | ray@vectify.ai | Vectify AI, agent framework |

---

## Version

`0.1.3` (Alpha)

---

## License

Apache 2.0

---

## Roadmap (from README)

- [ ] Extend long document handling to non-PDF formats
- [ ] Scale to large document collections with nested folder support
- [ ] Hierarchical concept (topic) indexing for massive knowledge bases
- [ ] Database-backed storage engine
- [ ] Web UI for browsing and managing wikis
