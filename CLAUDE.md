# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync

# Run the server (from project root)
./run.sh

# Or manually (must run from backend/ directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

The server must be launched from `backend/` because static file paths and `docs/` loading are relative to that directory.

- Web UI: `http://localhost:8000`
- API docs: `http://localhost:8000/docs`

## Environment

Create `.env` in the project root:
```
ANTHROPIC_API_KEY=your_key_here
```

## Architecture

Full-stack: FastAPI backend serves both the REST API and the static frontend (`frontend/`). There is no separate frontend build step — plain HTML/CSS/JS.

### Request Flow

```
POST /api/query
  → RAGSystem.query()
  → AIGenerator.generate_response()  # Claude with tool_choice=auto
      → Claude calls search_course_content tool
      → ToolManager.execute_tool()
      → CourseSearchTool.execute()
      → VectorStore.search()          # ChromaDB query
      → formatted results returned to Claude
      → Claude synthesizes final answer
```

Claude is **not** given retrieved context directly in the prompt. Instead, it receives the `search_course_content` tool and decides whether and how to use it. The AI generator handles the two-turn tool-use loop (`_handle_tool_execution`).

### Key Components (`backend/`)

| File | Responsibility |
|------|---------------|
| `app.py` | FastAPI app, endpoints, startup document loading |
| `rag_system.py` | Main orchestrator; wires all components together |
| `ai_generator.py` | Anthropic API calls; manages tool-use turn loop |
| `vector_store.py` | ChromaDB wrapper; two collections: `course_catalog` and `course_content` |
| `search_tools.py` | `Tool` ABC, `CourseSearchTool`, `ToolManager` |
| `document_processor.py` | Parses course `.txt` files; chunks text by sentence |
| `session_manager.py` | In-memory conversation history keyed by session ID |
| `models.py` | Pydantic models: `Course`, `Lesson`, `CourseChunk` |
| `config.py` | Central config dataclass; reads from `.env` |

### ChromaDB Collections

- **`course_catalog`**: one doc per course — used for semantic course-name resolution (`_resolve_course_name`). ID = course title.
- **`course_content`**: chunked lesson text. Metadata fields: `course_title`, `lesson_number`, `chunk_index`. ID format: `{course_title_underscored}_{chunk_index}`.

### Course Document Format (`docs/*.txt`)

```
Course Title: My Course Name
Course Link: https://example.com/course
Course Instructor: John Doe

Lesson 0: Introduction
Lesson Link: https://example.com/lesson/0
...lesson content...

Lesson 1: Next Topic
...lesson content...
```

Documents in `docs/` are loaded at startup. Already-indexed courses (by title) are skipped. Supported extensions: `.txt`, `.pdf`, `.docx`.

### Configuration (`config.py`)

| Key | Default | Meaning |
|-----|---------|---------|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Generation model |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | SentenceTransformer for embeddings |
| `CHUNK_SIZE` | `800` | Characters per chunk |
| `CHUNK_OVERLAP` | `100` | Overlap between chunks |
| `MAX_RESULTS` | `5` | ChromaDB results per query |
| `MAX_HISTORY` | `2` | Conversation exchanges to keep |
| `CHROMA_PATH` | `./chroma_db` | Persisted DB path (relative to `backend/`) |
