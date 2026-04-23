# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Quick start
./run.sh

# Manual
cd backend && uv run uvicorn app:app --reload --port 8000
```

Requires a `.env` file in the project root:
```
ANTHROPIC_API_KEY=your_key_here
```

App serves at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

## Dependencies

Managed with `uv`. Install with:
```bash
uv sync
```

No test suite exists in this codebase.

## File Structure

```
.
├── backend/
│   ├── app.py               # FastAPI app, API endpoints
│   ├── config.py            # Config dataclass (model, paths, chunk settings)
│   ├── rag_system.py        # Main orchestrator wiring all components
│   ├── document_processor.py# File parsing and text chunking
│   ├── vector_store.py      # ChromaDB wrapper (two collections)
│   ├── ai_generator.py      # Anthropic Claude API calls + tool-use loop
│   ├── search_tools.py      # Tool ABC, CourseSearchTool, ToolManager
│   ├── session_manager.py   # In-memory conversation history
│   └── models.py            # Pydantic models: Course, Lesson, CourseChunk
├── frontend/
│   ├── index.html
│   ├── script.js
│   └── style.css
├── docs/                    # Course transcript .txt files (knowledge base)
├── main.py                  # Unused entry point
├── run.sh                   # Quick-start script
├── pyproject.toml
└── .env                     # ANTHROPIC_API_KEY (not committed)
```

## Architecture

This is a full-stack RAG chatbot. The backend runs from the `backend/` directory (all imports are relative to it). The frontend is served as static files from `frontend/`.

### Query Flow

1. `POST /api/query` → `RAGSystem.query()`
2. `RAGSystem` builds a prompt and calls `AIGenerator.generate_response()` with the `search_course_content` tool available
3. Claude decides whether to call the tool; if it does, `ToolManager.execute_tool()` dispatches to `CourseSearchTool.execute()`
4. `CourseSearchTool` calls `VectorStore.search()`, which optionally resolves a fuzzy course name via semantic search on `course_catalog`, then queries `course_content` with optional `course_title`/`lesson_number` filters
5. Tool results are fed back to Claude for a final response (two API round-trips when a tool is used)
6. Sources and response are returned to the frontend

### Document Ingestion Flow (startup)

`docs/*.txt` files → `DocumentProcessor.process_course_document()` → parses header metadata + `Lesson N:` markers → `chunk_text()` (sentence-aware, 800-char chunks, 100-char overlap) → `CourseChunk` objects with context-prefixed content → stored in ChromaDB `course_content` collection. Course metadata goes into a separate `course_catalog` collection. Already-ingested courses (by title) are skipped on re-start.

### Expected Document Format

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: Introduction
Lesson Link: <url>
<lesson transcript text...>

Lesson 1: Topic Name
<lesson transcript text...>
```

### Key Design Decisions

- **Two ChromaDB collections**: `course_catalog` (one doc per course, used for fuzzy course name resolution) and `course_content` (one doc per chunk, used for semantic search). Both use `all-MiniLM-L6-v2` embeddings.
- **Tool-use pattern**: Claude is given a single `search_course_content` tool and decides when to use it. General knowledge questions are answered without searching. Max one search per query.
- **Session history**: Stored in-memory in `SessionManager` as plain text, injected into the system prompt. Keeps last 2 exchanges (`MAX_HISTORY=2`). Sessions are not persisted across server restarts.
- **ChromaDB persistence**: Stored in `backend/chroma_db/`. Delete this directory to force a full re-index.
- **Adding new tools**: Implement the `Tool` ABC in `search_tools.py` and register via `ToolManager.register_tool()` in `RAGSystem.__init__()`.
