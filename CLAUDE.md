# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run the server (from project root)
./run.sh

# Run manually
cd backend && uv run uvicorn app:app --reload --port 8000
```

The app is served at `http://localhost:8000`. The frontend is served as static files from the same FastAPI server — there is no separate frontend dev server.

> **Always use `uv` to run commands** (e.g., `uv run`, `uv sync`). Never use `pip` or invoke `python`/`uvicorn` directly.

## Architecture

This is a single-server RAG chatbot. FastAPI serves both the REST API (`/api/*`) and the static frontend (`../frontend/`). There is no build step for the frontend.

### Query pipeline

Every user query flows through this chain:

```
app.py (route) → RAGSystem.query() → AIGenerator.generate_response()
  → Claude API call #1 (with tool definitions)
    → if stop_reason == "tool_use":
        ToolManager.execute_tool() → CourseSearchTool.execute() → VectorStore.search()
        → Claude API call #2 (tool results appended, no tools)
    → else: return direct answer
```

### Key design decisions

- **Tool-based retrieval**: Claude decides whether to search — it is not forced. The `search_course_content` tool is passed on every query and Claude calls it only for course-specific questions.
- **Two ChromaDB collections**: `course_catalog` stores course-level metadata (title, instructor, links, lessons serialized as JSON); `course_content` stores text chunks with `course_title` and `lesson_number` metadata for filtering.
- **Fuzzy course name resolution**: When the tool is called with a `course_name`, `VectorStore._resolve_course_name()` does a semantic search against `course_catalog` to find the best matching title before filtering `course_content`. This means partial/approximate course names work.
- **Session history**: `SessionManager` keeps conversation history in-memory only (no persistence). History is injected into the system prompt as plain text, not as Claude message turns. Max 2 exchanges (4 messages) are kept per session (`MAX_HISTORY=2`).
- **Source tracking**: `CourseSearchTool` stores sources in `self.last_sources` after each search. `ToolManager.get_last_sources()` retrieves and `reset_sources()` clears them after each query cycle.

### Adding a new tool

1. Create a class extending `Tool` (ABC in `search_tools.py`) implementing `get_tool_definition()` and `execute()`
2. Register it in `RAGSystem.__init__()` via `self.tool_manager.register_tool(your_tool)`
3. If it tracks sources, add a `last_sources` list attribute — `ToolManager` will pick it up automatically

### Course document format

Files in `docs/` must follow this structure for the parser (`document_processor.py`) to extract lessons correctly:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<lesson content...>

Lesson 2: <title>
...
```

Documents are chunked at 800 characters with 100-character sentence-based overlap. Chunks are deduplicated on re-load by checking existing course titles.

## Configuration (`backend/config.py`)

| Setting | Default | Purpose |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model used |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence-transformers model for ChromaDB |
| `CHUNK_SIZE` | 800 | Max chars per text chunk |
| `CHUNK_OVERLAP` | 100 | Overlap between chunks |
| `MAX_RESULTS` | 5 | Max vector search results |
| `MAX_HISTORY` | 2 | Conversation exchanges retained per session |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence path (relative to `backend/`) |
