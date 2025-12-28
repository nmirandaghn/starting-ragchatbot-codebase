# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (from root directory)
./run.sh

# Manual start (from root directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

### Package Management
```bash
# Install/sync dependencies
uv sync
```

**IMPORTANT**:
- Always use `uv` to manage all dependencies. Do not use `pip` directly.
- Use `uv run` to execute Python files (e.g., `uv run python script.py` or `uv run uvicorn app:app`)
- All dependencies are managed through `uv` and defined in `pyproject.toml`.

### Environment Setup
Required `.env` file in root directory:
```bash
ANTHROPIC_API_KEY=your_api_key_here
```

## Architecture Overview

This is a **tool-based RAG (Retrieval-Augmented Generation)** chatbot for querying course materials. The key architectural pattern is that Claude decides when to search, rather than always retrieving context upfront.

### Core Flow: Tool-Based RAG Pattern

```
User Query → RAG System → AI Generator (Claude)
                              ↓
                         Tool Decision
                              ↓
                    [Uses search_course_content tool]
                              ↓
                         ToolManager → CourseSearchTool → VectorStore (ChromaDB)
                              ↓
                    [Search results returned to Claude]
                              ↓
                    AI Generator synthesizes final answer
                              ↓
                    Response + Sources
```

**Critical difference from traditional RAG**: Claude receives tool definitions and decides whether to search based on the query. For general questions, it answers directly. For course-specific questions, it uses the search tool.

### Component Responsibilities

**`rag_system.py` (Main Orchestrator)**
- Coordinates all components (document processor, vector store, AI generator, session manager, tool manager)
- Implements `query()` method that handles the complete request flow
- Manages course document ingestion from `docs/` folder
- Extracts sources from tools and manages conversation history

**`ai_generator.py` (Claude Integration)**
- Makes **two Claude API calls per tool-using query**:
  1. First call with `tools` parameter → Claude decides to use `search_course_content`
  2. Second call with `tool_results` → Claude synthesizes answer from search results
- System prompt enforces: "One search per query maximum"
- Temperature set to 0 for consistency
- Max tokens: 800

**`search_tools.py` (Tool Framework)**
- `Tool` abstract base class for extensibility
- `CourseSearchTool` implements search with optional `course_name` and `lesson_number` filters
- `ToolManager` handles tool registration, execution, and source tracking
- Sources tracked in `last_sources` attribute for UI display

**`vector_store.py` (ChromaDB Wrapper)**
- Maintains **two ChromaDB collections**:
  - `course_catalog`: Course metadata (titles, instructors, lessons) for semantic course name resolution
  - `course_content`: Chunked course content with embeddings
- `search()` method implements two-step process:
  1. Resolve fuzzy course names via semantic search in catalog (e.g., "MCP" → "Building Agentic RAG with Claude")
  2. Search content with filters applied
- Uses sentence-transformers embedding model: `all-MiniLM-L6-v2`

**`document_processor.py` (Content Processing)**
- Parses structured course files expecting:
  ```
  Course Title: [title]
  Course Link: [url]
  Course Instructor: [instructor]
  Lesson N: [title]
  Lesson Link: [url]
  [content...]
  ```
- Sentence-based chunking with overlap (800 chars, 100 char overlap)
- Adds context prefixes to chunks: `"Course {title} Lesson {N} content: {chunk}"`
- This context helps AI understand source during retrieval

**`session_manager.py` (Conversation State)**
- Maintains in-memory conversation history per session
- Configured to remember last 2 exchanges (4 messages) via `MAX_HISTORY=2`
- History formatted as: `"User: ...\nAssistant: ...\nUser: ..."`
- Passed to Claude in system prompt for multi-turn context

**`app.py` (FastAPI Backend)**
- **Startup behavior**: Automatically loads all documents from `../docs/` folder into ChromaDB
- API endpoints:
  - `POST /api/query`: Process user queries (creates session if needed)
  - `GET /api/courses`: Return course analytics (count, titles)
- CORS enabled for cross-origin requests
- Serves static frontend from `../frontend/`

### Data Flow Example

For query "What is MCP?":

1. **Frontend** sends: `{"query": "What is MCP?", "session_id": null}`
2. **app.py** creates session_1, calls `rag_system.query()`
3. **rag_system.py** retrieves conversation history (none), gets tool definitions
4. **ai_generator.py** calls Claude API with tools → Claude responds with `tool_use`
5. **Tool execution**: `search_course_content(query="MCP")`
6. **vector_store.py** performs semantic search → returns 5 chunks with metadata
7. **search_tools.py** formats results as: `"[Course - Lesson N]\n{content}"`, tracks sources
8. **ai_generator.py** sends tool_results back to Claude → Claude synthesizes answer
9. **rag_system.py** extracts sources from ToolManager, saves conversation
10. **app.py** returns: `{"answer": "...", "sources": [...], "session_id": "session_1"}`
11. **Frontend** renders markdown answer with collapsible sources

### Configuration (`backend/config.py`)

Key settings:
- `CHUNK_SIZE: 800` - Characters per chunk
- `CHUNK_OVERLAP: 100` - Overlap between chunks for context continuity
- `MAX_RESULTS: 5` - Results returned from vector search
- `MAX_HISTORY: 2` - Conversation exchanges remembered (4 messages total)
- `ANTHROPIC_MODEL: "claude-sonnet-4-20250514"`
- `EMBEDDING_MODEL: "all-MiniLM-L6-v2"`

### Adding New Course Documents

1. Place `.txt`, `.pdf`, or `.docx` files in `docs/` folder
2. Restart server - `startup_event()` in `app.py` loads documents automatically
3. Documents are deduplicated by course title (won't re-add existing courses)

### Adding New Tools

1. Create class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` and `execute()` methods
3. Register tool with ToolManager in `rag_system.py.__init__()`
4. Optionally add `last_sources` attribute for source tracking

### Frontend Architecture

- **Vanilla JavaScript** (no frameworks) in `frontend/script.js`
- Uses `marked.js` for markdown rendering
- Session ID stored in `currentSessionId` global variable
- Sources displayed in collapsible `<details>` element
- Suggested questions hardcoded in `index.html`

## Important Patterns

**Session Management**: Sessions are created on first query and maintained in-memory. Session IDs are returned to frontend and sent with subsequent queries for conversation continuity.

**Source Attribution**: Sources flow from VectorStore metadata → SearchResults → CourseSearchTool.last_sources → ToolManager.get_last_sources() → RAGSystem → API response → Frontend display.

**Error Handling**: Vector store returns `SearchResults.empty(error_msg)` for failures. Tool execution errors returned as strings to Claude, which can explain them to users.

**ChromaDB Persistence**: Database stored at `backend/chroma_db/` - delete this directory to reset the vector store.

**Two-Collection Strategy**: Catalog collection enables fuzzy course name matching (e.g., user says "agents course" → resolves to full course title), then content collection is filtered by exact title for accurate retrieval.
