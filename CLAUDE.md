# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

```bash
# Requires ANTHROPIC_API_KEY in a .env file at the project root
cp .env.example .env   # then fill in the key

./run.sh               # installs deps and starts the server
# or manually:
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access at `http://localhost:8000`. The backend also serves the frontend as static files, so there is no separate frontend dev server.

## Package Management

Uses `uv`. Add/remove dependencies via `pyproject.toml` and run `uv sync`.

## Architecture

### Request Flow (query path)
1. `frontend/script.js` — `POST /api/query` with `{ query, session_id }`
2. `backend/app.py` — FastAPI route creates session if missing, delegates to `RAGSystem.query()`
3. `backend/rag_system.py` — fetches conversation history, calls `AIGenerator.generate_response()` with tool definitions
4. `backend/ai_generator.py` — first Claude API call with `search_course_content` tool; if Claude calls the tool, executes it and makes a second Claude API call with the results to synthesize the final answer
5. `backend/search_tools.py` — `ToolManager` dispatches to `CourseSearchTool`, which calls `VectorStore.search()`
6. `backend/vector_store.py` — optionally resolves a fuzzy course name via semantic search on `course_catalog`, then queries `course_content` with ChromaDB embeddings
7. Sources are collected from the tool, session history is updated, response returned to frontend

### Two ChromaDB Collections
- **`course_catalog`** — one document per course (title, instructor, lesson list as JSON). Used only for fuzzy course-name resolution.
- **`course_content`** — all text chunks across all courses/lessons. Used for the actual semantic search. Metadata fields: `course_title`, `lesson_number`, `chunk_index`.

### Document Ingestion
`DocumentProcessor` parses `.txt` / `.pdf` / `.docx` files expecting this header format:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <title>
Lesson Link: <url>
<content...>

Lesson 2: ...
```
Content is split into sentence-aware chunks (800 chars, 100 char overlap). Chunks are deduplicated on startup by comparing against existing course titles in ChromaDB.

### Tool Use Pattern
`AIGenerator` always sends the `search_course_content` tool definition to Claude with `tool_choice: auto`. Claude decides whether to search. If it does, the code runs a two-turn conversation: assistant tool call → tool result → final answer. If Claude answers directly (general knowledge questions), only one API call is made.

### Session Management
Sessions are stored in-memory in `SessionManager` (lost on restart). History is passed to Claude as plain text appended to the system prompt, not as message history. `MAX_HISTORY = 2` means the last 2 exchanges are retained.

### Key Config (`backend/config.py`)
| Setting | Default | Notes |
|---|---|---|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Change here to switch models |
| `CHUNK_SIZE` | 800 | Characters per chunk |
| `CHUNK_OVERLAP` | 100 | Overlap between chunks |
| `MAX_RESULTS` | 5 | ChromaDB results per search |
| `MAX_HISTORY` | 2 | Conversation exchanges retained |
| `CHROMA_PATH` | `./chroma_db` | Persisted to disk relative to `backend/` |
