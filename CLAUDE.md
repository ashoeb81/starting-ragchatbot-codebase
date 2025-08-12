# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Environment Setup
```bash
# Install dependencies
uv sync

# Create .env file with required API key
echo "ANTHROPIC_API_KEY=your_key_here" > .env
```

### Access Points
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

This is a **Retrieval-Augmented Generation (RAG) system** for querying course materials using semantic search and AI responses.

### Core Architecture Pattern
The system follows a **tool-enabled RAG** pattern where Claude AI dynamically decides whether to search course content based on the query type:

1. **Query Classification**: Claude determines if search is needed
2. **Conditional Search**: Semantic search only when required
3. **Context Synthesis**: AI combines search results with conversation history
4. **Response Generation**: Direct answers with source citations

### Component Interaction Flow

```
Frontend (HTML/JS) → FastAPI → RAG System → AI Generator ↔ Claude API
                                     ↓            ↑
                              Search Tools → Vector Store → ChromaDB
```

### Key Components

**`rag_system.py`** - Main orchestrator that coordinates all components. Handles query processing, session management, and tool integration.

**`ai_generator.py`** - Manages Anthropic Claude API interactions with tool support. Implements the tool execution workflow where Claude can call search functions.

**`search_tools.py`** - Provides the `CourseSearchTool` that Claude can use. Supports semantic course name matching and lesson filtering.

**`vector_store.py`** - ChromaDB interface for semantic search. Manages embeddings using sentence-transformers and provides search with metadata filtering.

**`document_processor.py`** - Processes course documents into structured chunks. Parses course metadata and lesson structure from text files.

**`session_manager.py`** - Maintains conversation history for multi-turn context.

### Document Structure Expected

Course documents in `docs/` folder should follow this format:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: Introduction
Lesson Link: [lesson_url]
[lesson content...]

Lesson 1: [title]
[lesson content...]
```

### Configuration

All settings are centralized in `backend/config.py`:
- **Chunk settings**: `CHUNK_SIZE=800`, `CHUNK_OVERLAP=100`
- **AI model**: `claude-sonnet-4-20250514`
- **Embedding model**: `all-MiniLM-L6-v2`
- **Search limits**: `MAX_RESULTS=5`

### Data Models

Three core Pydantic models in `models.py`:
- **`Course`**: Course metadata with lessons list
- **`Lesson`**: Individual lesson with number, title, link
- **`CourseChunk`**: Text chunks for vector storage with metadata

### Tool-Based Search Logic

The system uses Anthropic's tool calling feature:
1. Claude receives query with `search_course_content` tool available
2. Claude decides autonomously whether to search based on query context
3. If searching, Claude can filter by course name and lesson number
4. Search results are fed back to Claude for response synthesis
5. Sources are tracked separately and returned with the response

### Session Management

Sessions maintain conversation context:
- Session ID generated on first query
- Conversation history limited to `MAX_HISTORY=2` exchanges
- Context passed to Claude for coherent multi-turn conversations

### Vector Storage Strategy

ChromaDB collections:
- Course content chunks with embeddings
- Metadata includes course title, lesson number, chunk index
- Semantic search with optional filtering by course/lesson
- Embeddings generated using sentence-transformers

### Frontend Integration

Simple HTML/CSS/JS frontend that:
- Makes REST API calls to `/api/query` and `/api/courses`
- Handles session management client-side
- Renders markdown responses with source citations
- Displays course statistics and suggested questions

## Important Implementation Details

### Document Processing Pipeline
1. **File reading** with UTF-8 encoding and fallback handling
2. **Metadata extraction** from structured headers
3. **Lesson parsing** using regex patterns for lesson markers
4. **Text chunking** with sentence-aware splitting and overlap
5. **Context enhancement** adding course/lesson context to chunks

### AI Response Flow
The AI generator implements a two-phase process:
1. **Initial call** to Claude with tools available
2. **Tool execution** if Claude requests search
3. **Follow-up call** with search results for final response synthesis

### Error Handling Patterns
- Graceful degradation when search fails
- UTF-8 encoding fallbacks for document reading  
- Duplicate course detection during document loading
- Session creation on missing session IDs

### Configuration Flexibility
Environment variables override defaults:
- `ANTHROPIC_API_KEY` (required)
- Model and embedding settings configurable via `config.py`
- ChromaDB path and chunk parameters adjustable