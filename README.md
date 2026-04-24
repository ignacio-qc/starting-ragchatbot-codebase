# Course Materials RAG System

A Retrieval-Augmented Generation (RAG) system designed to answer questions about course materials using semantic search and AI-powered responses.

## Overview

This application is a full-stack web application that enables users to query course materials and receive intelligent, context-aware responses. It uses ChromaDB for vector storage, Anthropic's Claude for AI generation, and provides a web interface for interaction.

## Query Flow

```mermaid
sequenceDiagram
    actor User
    participant FE as Frontend<br/>(script.js)
    participant API as FastAPI<br/>(app.py)
    participant RAG as RAGSystem<br/>(rag_system.py)
    participant SM as SessionManager<br/>(session_manager.py)
    participant AI as AIGenerator<br/>(ai_generator.py)
    participant TM as ToolManager<br/>(search_tools.py)
    participant VS as VectorStore<br/>(vector_store.py)
    participant DB as ChromaDB
    participant CL as Claude API

    User->>FE: types message, hits send
    FE->>API: POST /api/query<br/>{ query, session_id }

    API->>SM: create_session() if no session_id
    API->>RAG: query(query, session_id)

    RAG->>SM: get_conversation_history(session_id)
    SM-->>RAG: prior exchanges (up to 2)

    RAG->>AI: generate_response(prompt, history, tools)

    AI->>CL: 1st API call<br/>system prompt + user message + tool definition
    CL-->>AI: stop_reason="tool_use"<br/>search_course_content(query, course_name?, lesson?)

    AI->>TM: execute_tool("search_course_content", ...)
    TM->>VS: search(query, course_name, lesson_number)

    opt course_name provided
        VS->>DB: semantic search on course_catalog
        DB-->>VS: best matching course title
    end

    VS->>DB: query course_content collection<br/>with embedding + optional filters
    DB-->>VS: top-N chunks + metadata

    VS-->>TM: SearchResults (documents, metadata)
    TM-->>AI: formatted text chunks + sources stored

    AI->>CL: 2nd API call<br/>original messages + tool results (no tools)
    CL-->>AI: final synthesized answer

    AI-->>RAG: answer text
    RAG->>TM: get_last_sources() + reset_sources()
    RAG->>SM: add_exchange(session_id, query, answer)

    RAG-->>API: (answer, sources)
    API-->>FE: { answer, sources, session_id }

    FE->>User: renders markdown answer<br/>+ collapsible sources
```


## Prerequisites

- Python 3.13 or higher
- uv (Python package manager)
- An Anthropic API key (for Claude AI)
- **For Windows**: Use Git Bash to run the application commands - [Download Git for Windows](https://git-scm.com/downloads/win)

## Installation

1. **Install uv** (if not already installed)
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. **Install Python dependencies**
   ```bash
   uv sync
   ```

3. **Set up environment variables**
   
   Create a `.env` file in the root directory:
   ```bash
   ANTHROPIC_API_KEY=your_anthropic_api_key_here
   ```

## Running the Application

### Quick Start

Use the provided shell script:
```bash
chmod +x run.sh
./run.sh
```

### Manual Start

```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- Web Interface: `http://localhost:8000`
- API Documentation: `http://localhost:8000/docs`

