# Cognee Codebase Memorization Plan

**Goal**: Build a persistent, queryable memory of the Cognee codebase using claude-mem so future sessions can answer "where does X live?", "what pattern does Y use?", and "how do I extend Z?" without re-reading source files.

**Discovery date**: 2026-04-19  
**Repo**: /home/tuanna47/workspace/FSO/cognee  
**Python**: 3.10–3.13 | **Frontend**: Next.js 16 + React 19

---

## Phase 0: Discovery Summary (COMPLETE)

Sources consulted:
- `CLAUDE.md` (594 lines) — authoritative dev guide
- `cognee/__init__.py` — public API exports
- `cognee/` directory tree (1,224 Python files, 252 test files)
- `cognee-frontend/` (216 TS/TSX files, Next.js App Router)
- `pyproject.toml`, `.env.template`, `docker-compose.yml`

### Allowed APIs (claude-mem MCP tools available in session)
- `mcp__plugin_claude-mem_mcp-search__build_corpus` — index files into vector corpus
- `mcp__plugin_claude-mem_mcp-search__prime_corpus` — load corpus for querying
- `mcp__plugin_claude-mem_mcp-search__get_observations` — fetch stored observations
- `mcp__plugin_claude-mem_mcp-search__smart_search` — semantic query
- `mcp__plugin_claude-mem_mcp-search__smart_outline` — outline a topic
- `mcp__plugin_claude-mem_mcp-search__query_corpus` — raw vector search
- `mcp__plugin_claude-mem_mcp-search__search` — keyword search

### Confirmed Anti-Patterns
- Do NOT use `mcp__plugin_claude-mem_mcp-search__search` expecting structured JSON if the corpus isn't primed first
- Do NOT write observations without a corpus handle from `build_corpus`
- Do NOT re-read full files to answer questions once corpus is primed

---

## Phase 1: Build the Searchable Corpus

**Context**: Start a fresh session. The claude-mem plugin is installed and MCP tools are available.

### Tasks

1. **Build corpus for core Python package**
   ```
   Tool: mcp__plugin_claude-mem_mcp-search__build_corpus
   Args:
     path: /home/tuanna47/workspace/FSO/cognee/cognee
     name: cognee-core
     include: ["**/*.py"]
     exclude: [".venv/**", "**/__pycache__/**", "**/migrations/**"]
   ```
   Expected result: corpus handle (save it — needed for all subsequent queries)

2. **Build corpus for tests**
   ```
   Tool: mcp__plugin_claude-mem_mcp-search__build_corpus
   Args:
     path: /home/tuanna47/workspace/FSO/cognee/cognee/tests
     name: cognee-tests
     include: ["**/*.py"]
   ```

3. **Build corpus for frontend**
   ```
   Tool: mcp__plugin_claude-mem_mcp-search__build_corpus
   Args:
     path: /home/tuanna47/workspace/FSO/cognee/cognee-frontend/src
     name: cognee-frontend
     include: ["**/*.ts", "**/*.tsx"]
     exclude: ["**/*.test.*", "**/node_modules/**"]
   ```

### Verification
- `mcp__plugin_claude-mem_mcp-search__list_corpora` should show 3 corpora
- `mcp__plugin_claude-mem_mcp-search__query_corpus` with "DataPoint" should return results from cognee-core

---

## Phase 2: Store Architecture Observations

**Context**: Corpora are built. Prime `cognee-core` first, then add observations.

Prime the corpus:
```
Tool: mcp__plugin_claude-mem_mcp-search__prime_corpus
Args: name: cognee-core
```

### Observations to Store (use `get_observations` + store pattern)

Store each as a named observation in the corpus. Source citations are mandatory.

#### 2a. Core Workflow
```
Topic: core-workflow
Content:
  Cognee's ECL pipeline: add() → cognify() → search()
  - add(): cognee/api/v1/add/add.py — ingest files/text into datasets
  - cognify(): cognee/api/v1/cognify/cognify.py — LLM extracts entities → Kuzu graph
  - search(): cognee/api/v1/search/search.py — routes to 14+ retriever strategies
  All functions are async. Use await or asyncio.run().
  DataPoint base class: cognee/infrastructure/engine/models/DataPoint.py
```

#### 2b. Search Types
```
Topic: search-types
Content:
  Defined in: cognee/modules/search/types/SearchType.py
  14+ types including:
  - GRAPH_COMPLETION (default) — graph traversal + LLM
  - GRAPH_SUMMARY_COMPLETION — pre-computed summaries + graph
  - TRIPLET_COMPLETION — subject-predicate-object triples
  - RAG_COMPLETION — traditional vector RAG
  - CHUNKS — vector similarity on raw chunks
  - CHUNKS_LEXICAL — keyword search on chunks
  - SUMMARIES — search document summaries
  - CYPHER — raw Cypher (requires ALLOW_CYPHER_QUERY=True)
  - NATURAL_LANGUAGE — NL → structured query
  - TEMPORAL — time-aware graph search
  - FEELING_LUCKY — auto-selects best strategy
  - CODING_RULES — code-specific rules
```

#### 2c. Layer Architecture
```
Topic: layer-architecture
Content:
  API Layer: cognee/api/v1/ (FastAPI routes — add, cognify, search, memify, datasets, users, visualize)
  Main Functions: cognee/__init__.py (public SDK: add, cognify, search, memify, delete, config, datasets)
  Pipeline Orchestrator: cognee/modules/pipelines/
  Task Execution: cognee/tasks/ (17 task categories — graph, storage, ingestion, chunking, etc.)
  Domain Modules: cognee/modules/ (25 modules)
  Infrastructure Adapters: cognee/infrastructure/ (LLM, databases, files, loaders)
  External Services: OpenAI/Kuzu/LanceDB/SQLite (defaults)
```

#### 2d. Database Adapter Pattern
```
Topic: database-adapters
Content:
  Graph DB interface: cognee/infrastructure/databases/graph/graph_db_interface.py
  Vector DB interface: cognee/infrastructure/databases/vector/vector_db_interface.py
  Default graph backend: Kuzu (cognee/infrastructure/databases/graph/kuzu/)
  Default vector backend: LanceDB (cognee/infrastructure/databases/vector/lancedb/)
  Default relational: SQLite
  Supported graph backends: kuzu, neo4j, neptune, kuzu-remote, postgres
  Supported vector backends: lancedb, pgvector, chromadb, qdrant, weaviate, milvus
  Factory functions:
    get_graph_engine() → cognee/infrastructure/databases/graph/__init__.py
    get_vector_engine() → cognee/infrastructure/databases/vector/__init__.py
```

#### 2e. LLM Gateway Pattern
```
Topic: llm-gateway
Content:
  Central interface: cognee/infrastructure/llm/LLMGateway.py
  Config: cognee/infrastructure/llm/config.py
  Factory: cognee/infrastructure/llm/get_llm_client.py
  Providers: OpenAI, Azure OpenAI, Anthropic, Gemini, Ollama, Mistral, Bedrock, custom
  Structured output: via Instructor (default) or BAML
  Key method: acreate_structured_output(text_input, system_prompt, response_model)
  Embedding factory: cognee/infrastructure/databases/vector/embeddings/get_embedding_engine.py
```

### Verification
```
Query: mcp__plugin_claude-mem_mcp-search__smart_search
Args: query: "how does cognify build the knowledge graph"
Expected: Returns references to cognify.py, extract_graph_from_data.py, DataPoint
```

---

## Phase 3: Store Module-Level Observations

**Context**: cognee-core corpus is primed. Add fine-grained module observations.

### Observations to Store

#### 3a. Key Task Files
```
Topic: pipeline-tasks
Content:
  All tasks live in cognee/tasks/ (17 categories):
  - cognee/tasks/graph/extract_graph_from_data.py — LLM entity/relationship extraction
  - cognee/tasks/storage/add_data_points.py — write nodes+edges to graph+vector DBs
  - cognee/tasks/ingestion/ingest_data.py — resolve and save input data
  - cognee/tasks/chunks/ — document chunking strategies
  - cognee/tasks/summarization/ — LLM-based text summarization
  Task pattern: async function returning Task object
  Pipeline registration: cognee/modules/pipelines/
```

#### 3b. Extension Points
```
Topic: extension-points
Content:
  New task type: create async fn in cognee/tasks/, return Task, register in pipeline
  New graph DB backend: implement GraphDBInterface in cognee/infrastructure/databases/graph/
  New vector DB backend: implement VectorDBInterface in cognee/infrastructure/databases/vector/
  New LLM provider: add to LLM config (uses litellm)
  New document processor: extend loaders in cognee/modules/data/processing/
  New search type: add to SearchType enum + implement retriever in cognee/modules/retrieval/
  Custom graph model: define Pydantic model extending DataPoint in user code
```

#### 3c. Multi-Tenant Access Control
```
Topic: access-control
Content:
  Hierarchy: User → Dataset → Data
  Enable: ENABLE_BACKEND_ACCESS_CONTROL=True
  Isolated DBs per user+dataset: supported by Kuzu, LanceDB, SQLite, Postgres
  Permission types: read, write, delete, share per dataset
  Auth enabled with: REQUIRE_AUTHENTICATION=True
  User management module: cognee/modules/users/
  API routes: cognee/api/v1/users/
```

#### 3d. Data Models
```
Topic: data-models
Content:
  DataPoint: cognee/infrastructure/engine/models/DataPoint.py — base for all graph nodes (versioned)
  Edge: cognee/infrastructure/engine/models/Edge.py — graph relationships
  Triplet: cognee/infrastructure/engine/models/Triplet.py — subject/predicate/object
  KnowledgeGraph: cognee/shared/data_models.py — container for nodes and edges
  Node: cognee/shared/data_models.py — entity with id, name, type, description
  All custom graph objects must extend DataPoint
```

#### 3e. Frontend Module Map
```
Topic: frontend-modules
Content:
  Framework: Next.js 16 App Router + React 19 + TypeScript + Mantine v8 + Tailwind v4
  Location: cognee-frontend/src/
  App routes: cognee-frontend/src/app/ (auth, app, graph)
  Feature modules: cognee-frontend/src/modules/ (15 modules)
    - datasets/ — dataset management
    - chat/ — chat interface
    - configuration/ — user settings
    - ingestion/ — data ingestion UI
    - graphModels/ — schema management
    - users/ — user management
  UI components: cognee-frontend/src/ui/
  Graph viz: D3 Force Graph, react-force-graph-2d
  Form validation: Mantine Form + Valibot
```

### Verification
```
Query: mcp__plugin_claude-mem_mcp-search__smart_search  
Args: query: "how to add a new search type"
Expected: Returns extension-points observation + SearchType enum location
```

---

## Phase 4: Store Infrastructure & Config Observations

**Context**: Add environment, deployment, and dev workflow observations.

#### 4a. Environment Variables (Critical)
```
Topic: env-config
Content:
  Template: .env.template (386 lines — authoritative reference)
  Minimal setup: LLM_API_KEY + LLM_MODEL (defaults to OpenAI)
  LLM config: LLM_PROVIDER, LLM_MODEL, LLM_ENDPOINT, LLM_API_KEY, LLM_API_VERSION
  DB config: DB_PROVIDER, DB_HOST, DB_PORT, DB_USERNAME, DB_PASSWORD, DB_NAME
  Vector config: VECTOR_DB_PROVIDER, VECTOR_DB_URL
  Graph config: GRAPH_DATABASE_PROVIDER, GRAPH_DATABASE_URL, GRAPH_DATABASE_NAME
  Storage config: STORAGE_BACKEND (local|s3), DATA_ROOT_DIRECTORY, SYSTEM_ROOT_DIRECTORY
  Security: ACCEPT_LOCAL_FILE_PATH, ALLOW_HTTP_REQUESTS, ALLOW_CYPHER_QUERY,
            REQUIRE_AUTHENTICATION, ENABLE_BACKEND_ACCESS_CONTROL
  Debug: LITELLM_LOG="DEBUG", ENV="development", TELEMETRY_DISABLED=1
  Rate limiting: LLM_RATE_LIMIT_ENABLED, LLM_RATE_LIMIT_REQUESTS, LLM_RATE_LIMIT_INTERVAL
```

#### 4b. Dev Workflow
```
Topic: dev-workflow
Content:
  Setup: uv venv && source .venv/bin/activate && uv pip install -e ".[dev]"
  Pre-commit: pre-commit install (ruff lint + format)
  Format: ruff format . && ruff check .
  Tests: pytest cognee/tests/ (unit/, integration/, e2e/)
  Type check: mypy cognee/
  Branch rule: ALWAYS branch from dev, not main
  Code style: Ruff, 100-char lines, double quotes enforced
  Commit prep: pre-commit run --all-files before every commit
```

#### 4c. Docker & Deployment
```
Topic: deployment
Content:
  Local dev: docker-compose.yml (185 lines) — cognee + mcp + postgres services
  Container: Dockerfile + entrypoint.sh
  K8s: deployment/ directory
  Distributed: distributed/ (Modal)
  UI launch: cognee-cli -ui → http://localhost:3000
  MCP server: cognee-mcp/ — Model Context Protocol integration
  Starter kit: cognee-starter-kit/
```

### Verification
```
Query: mcp__plugin_claude-mem_mcp-search__smart_search
Args: query: "how to run tests"
Expected: Returns dev-workflow observation with pytest commands
```

---

## Phase 5: Verification & Smoke Tests

**Context**: All observations stored. Verify retrieval quality.

### Test Queries

Run these `smart_search` queries; each should return the expected topic:

| Query | Expected Topic |
|---|---|
| "what happens when I call cognify?" | core-workflow, pipeline-tasks |
| "how to switch to neo4j" | database-adapters, env-config |
| "where are the data models defined" | data-models |
| "how to extend the search API" | extension-points, search-types |
| "frontend technology stack" | frontend-modules |
| "how to enable authentication" | access-control, env-config |
| "how to debug LLM calls" | env-config |

### Smart Outline Test
```
Tool: mcp__plugin_claude-mem_mcp-search__smart_outline
Args: topic: "cognee architecture"
Expected: Multi-section outline covering core workflow, search types, databases, LLM
```

### Anti-Pattern Checks
- Grep for any invented file paths before citing them
- Confirm `DataPoint` base class exists: `grep -r "class DataPoint" cognee/`
- Confirm `GraphDBInterface`: `grep -r "class GraphDBInterface" cognee/`

---

## Quick Reference (for future sessions)

```bash
# Prime corpus before querying
mcp: prime_corpus(name="cognee-core")

# Find where something lives
mcp: smart_search(query="<your question>")

# Get overview of a topic
mcp: smart_outline(topic="<topic>")

# Targeted file search (when corpus not available)
grep -r "pattern" cognee/ --include="*.py" -l
```

**Key files to know by heart:**
- `cognee/__init__.py` — all public API entry points
- `cognee/infrastructure/engine/models/DataPoint.py` — extend this for custom graph nodes
- `cognee/modules/search/types/SearchType.py` — all search strategies
- `cognee/infrastructure/databases/graph/graph_db_interface.py` — implement for new graph DB
- `cognee/infrastructure/databases/vector/vector_db_interface.py` — implement for new vector DB
- `cognee/infrastructure/llm/get_llm_client.py` — get LLM client in any module
- `.env.template` — authoritative config reference
