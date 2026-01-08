# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Environment Setup

```bash
# Create development environment (uses hatch)
make install

# Install all optional dependencies for full development
make install_all

# Activate environment for specific Python version
hatch shell dev_py_3_9   # Python 3.9
hatch shell dev_py_3_10  # Python 3.10
hatch shell dev_py_3_11  # Python 3.11
hatch shell dev_py_3_12  # Python 3.12
```

### Code Quality

```bash
# Format code with ruff
make format
# or
hatch run format

# Sort imports
make sort

# Lint code
make lint
# or
hatch run lint

# Fix lint issues
make lint-fix
# or
hatch run lint-fix
```

### Testing

```bash
# Run all tests
make test
# or
hatch run test

# Run tests for specific Python version
make test-py-3.9
make test-py-3.10
make test-py-3.11
make test-py-3.12

# Run specific test file
pytest tests/test_memory.py -v

# Run specific test function
pytest tests/test_memory.py::test_add_memory -v
```

### Build and Publish

```bash
# Build package
make build

# Clean build artifacts
make clean

# Publish to PyPI
make publish
```

### Documentation

```bash
# Run local docs server
make docs
```

## Architecture Overview

Mem0 is an intelligent memory layer for AI agents that combines vector storage and graph databases for context-aware memory retrieval.

### Core Architecture Pattern

The system uses a **factory pattern** for component instantiation and a **repository pattern** for data access:

```
Memory (main.py)
    ├── Vector Store (vector_stores/) - Embedding-based similarity search
    ├── Graph Store (graphs/) - Entity-relationship storage (Neo4j, Memgraph, Neptune, Kuzu)
    ├── LLM Provider (llms/) - Text generation and extraction
    ├── Embedder (embeddings/) - Text vectorization
    ├── Reranker (reranker/) - Result re-ranking
    └── Storage (storage.py) - SQLite history tracking
```

### Key Components

**`mem0/memory/main.py`** - Main Memory class (~2500 lines)
- Primary entry point for all memory operations
- Handles both sync and async operations
- Coordinates parallel execution of vector and graph searches using ThreadPoolExecutor
- Implements v1.1 API with consistent `{"results": [...]}` response format

**`mem0/utils/factory.py`** - Component factories
- `LlmFactory` - Creates LLM instances (OpenAI, Anthropic, Groq, etc.)
- `VectorStoreFactory` - Creates vector store instances (Qdrant, Chroma, etc.)
- `EmbedderFactory` - Creates embedding models
- `GraphStoreFactory` - Creates graph store instances (Neo4j, Memgraph, etc.)
- `RerankerFactory` - Creates reranker instances

**`mem0/configs/base.py`** - Configuration models
- `MemoryConfig` - Main configuration with defaults for all components
- `MemoryItem` - Standard memory data structure
- All configs use Pydantic for validation

### Memory Flow

**Add Operation:**
1. Parse input messages (string, dict, or list of messages)
2. Extract facts using LLM with custom prompts
3. Generate embeddings for the facts
4. Search for existing similar memories (to avoid duplicates)
5. Add to vector store
6. Optionally add to graph store (if enabled)
7. Track in SQLite history database

**Search Operation:**
1. Generate query embedding
2. Parallel execution:
   - Vector similarity search in vector store
   - Entity extraction and graph traversal (if enabled)
3. Optional reranking of results
4. Return consistent format: `{"results": [...], "relations": [...]}`

### Graph Memory Architecture

Graph memory extends vector search with entity-relationship context:

**Graph implementations:**
- `mem0/memory/graph_memory.py` - Neo4j implementation
- `mem0/memory/memgraph_memory.py` - Memgraph implementation
- `mem0/memory/kuzu_memory.py` - Kuzu implementation
- `mem0/graphs/neptune/` - AWS Neptune implementations

**Graph workflow:**
1. Extract entities and relationships using LLM tools
2. Match nodes using embedding similarity (configurable threshold)
3. Create/update nodes and relationships in graph
4. Search uses both vector similarity and graph traversal
5. BM25 reranking on graph results

**Threshold configuration (`graph_store.threshold`):**
- 0.95-0.99: Strict matching (UUIDs, IDs)
- 0.85-0.9: Structured data
- 0.7-0.8: Default (general purpose)
- 0.6-0.7: Permissive (natural language, aliases)

### Response Format (v1.1)

All operations return: `{"results": [...], "relations": [...]}`

Each result in `results` array contains:
- `id`: Unique identifier
- `memory`: Extracted memory content
- `score`: Similarity score (for search)
- `metadata`: Associated metadata
- `created_at`/`updated_at`: Timestamps

### Filter System

**Required filters (at least one):**
- `user_id` - User scope
- `agent_id` - Agent scope
- `run_id` - Session scope

**Optional filters:**
- Metadata filters with operators: `eq`, `ne`, `in`, `nin`, `gt`, `gte`, `lt`, `lte`, `contains`, `icontains`
- Logical operators: `AND`, `OR`, `NOT`
- Wildcard: `*`

## Important Implementation Details

### Async vs Sync

The Memory class automatically detects async operations:
- If the caller is in an async context, operations run asynchronously
- ThreadPoolExecutor is used for parallel vector/graph searches in sync mode
- Async operations use `asyncio.to_thread()` for blocking I/O

### Default Behavior

- **LLM**: OpenAI `gpt-4.1-nano-2025-04-14` (requires API key)
- **Vector Store**: Qdrant (local, requires no setup)
- **Embedder**: OpenAI embeddings
- **Graph Store**: Disabled by default (must be explicitly configured)
- **History**: SQLite at `~/.mem0/history.db`

### Testing Notes

- Tests use pytest with asyncio support
- Test fixtures in `tests/conftest.py` provide mock configurations
- Vector store tests are in `tests/vector_stores/` (separate file per provider)
- Integration tests verify end-to-end functionality

### Code Style

- **Line length**: 120 characters (ruff config)
- **Formatter**: ruff format
- **Import sorting**: isort with black profile
- **Excluded from linting**: `embedchain/`, `openmemory/`

## Version Migration

v1.0 removed the `version` parameter - all operations now use v1.1 format by default. See `MIGRATION_GUIDE_v1.0.md` for details if updating older code.

## Environment Variables

- `MEM0_DIR` - Override default `~/.mem0` config directory
- `OPENAI_API_KEY` - Required for default LLM/embedder
- Provider-specific env vars for other integrations
