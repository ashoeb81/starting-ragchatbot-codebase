# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Frontend<br/>(HTML/JS)
    participant API as FastAPI<br/>(app.py)
    participant RAG as RAG System<br/>(rag_system.py)
    participant AI as AI Generator<br/>(ai_generator.py)
    participant Claude as Anthropic<br/>Claude API
    participant Tools as Search Tools<br/>(search_tools.py)
    participant Vector as Vector Store<br/>(vector_store.py)
    participant ChromaDB as ChromaDB<br/>(Embeddings)
    participant Session as Session Manager<br/>(session_manager.py)

    Note over User, ChromaDB: User Query Processing Flow

    User->>Frontend: 1. Types query in chat input
    Frontend->>Frontend: 2. sendMessage() - disable input, show loading
    Frontend->>API: 3. POST /api/query<br/>{query, session_id}

    API->>API: 4. query_documents() - validate request
    API->>RAG: 5. rag_system.query(query, session_id)

    RAG->>Session: 6. Get conversation history
    Session-->>RAG: Previous context (if exists)

    RAG->>AI: 7. generate_response() with tools & history
    AI->>AI: 8. Build system prompt + conversation context

    AI->>Claude: 9. API call with tools available
    Claude-->>AI: 10. Response (may include tool calls)

    alt Claude decides to use search tool
        AI->>Tools: 11. execute_tool() - search_course_content
        Tools->>Vector: 12. search_course_content(query, filters)
        Vector->>ChromaDB: 13. Semantic search with embeddings
        ChromaDB-->>Vector: 14. Relevant document chunks + metadata
        Vector-->>Tools: 15. SearchResults with sources
        Tools-->>AI: 16. Formatted search results

        AI->>Claude: 17. Send tool results back to Claude
        Claude-->>AI: 18. Final synthesized response
    end

    AI-->>RAG: 19. Generated response text
    RAG->>Tools: 20. get_last_sources() for citation
    Tools-->>RAG: 21. Source list from search
    
    RAG->>Session: 22. add_exchange() - store conversation
    RAG-->>API: 23. (response, sources)

    API->>API: 24. Create QueryResponse object
    API-->>Frontend: 25. JSON {answer, sources, session_id}

    Frontend->>Frontend: 26. Parse response, remove loading
    Frontend->>Frontend: 27. addMessage() - render with markdown
    Frontend->>User: 28. Display answer + sources in chat UI

    Note over User, ChromaDB: Response displayed with sources and session maintained
```

## Component Responsibilities

### Frontend Layer
- **HTML/CSS**: Chat interface, input fields, loading states
- **JavaScript**: Event handling, API calls, UI updates, markdown rendering

### API Layer  
- **FastAPI**: HTTP endpoints, request validation, response formatting
- **CORS**: Cross-origin support, static file serving

### RAG Orchestration
- **RAG System**: Main coordinator, session management integration
- **Session Manager**: Conversation history, context preservation

### AI Processing
- **AI Generator**: Claude API integration, tool orchestration
- **Search Tools**: Semantic search interface, result formatting

### Data Layer
- **Vector Store**: ChromaDB interface, embedding management
- **Document Processor**: Text chunking, metadata extraction (used during indexing)

## Key Data Transformations

1. **User Input** → POST JSON
2. **Query** → System Prompt + Context  
3. **Prompt** → Claude Tool Calls
4. **Tool Parameters** → Vector Search Query
5. **Document Chunks** → Tool Results
6. **Tool Results** → Claude Response
7. **Response** → JSON API Response
8. **JSON** → Rendered HTML/Markdown

## Session Flow
- Session created on first query if not provided
- Conversation history maintained throughout session
- Context passed to Claude for coherent multi-turn conversations
- Sources tracked and reset after each query