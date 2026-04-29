# DeepTutor Backend Components

> Architecture reference for the rework. Generated from codebase exploration.

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [API Layer](#2-api-layer)
3. [Core Abstractions](#3-core-abstractions)
4. [Agents](#4-agents)
5. [Capabilities](#5-capabilities)
6. [Runtime System](#6-runtime-system)
7. [Services Layer](#7-services-layer)
8. [Knowledge Base](#8-knowledge-base)
9. [TutorBot System](#9-tutorbot-system)
10. [Tools](#10-tools)
11. [Book Module](#11-book-module)
12. [Co-Writer Module](#12-co-writer-module)
13. [Configuration](#13-configuration)
14. [Event System](#14-event-system)
15. [Logging](#15-logging)
16. [Database & Storage](#16-database--storage)
17. [API Endpoints Reference](#17-api-endpoints-reference)
18. [External Integrations](#18-external-integrations)
19. [Environment Variables](#19-environment-variables)
20. [Background Tasks & Workers](#20-background-tasks--workers)
21. [Middleware & Dependencies](#21-middleware--dependencies)
22. [Architecture & Data Flow](#22-architecture--data-flow)

---

## 1. Directory Structure

```
deeptutor/
├── api/              # FastAPI app, routers, WebSocket handling
├── agents/           # Multi-agent implementations per capability
├── app/              # Application facade
├── book/             # Book generation engine
├── capabilities/     # High-level capability orchestrators
├── config/           # Configuration schema, defaults, accessors
├── core/             # Base protocols, context, stream types
├── events/           # Async event bus (pub/sub)
├── knowledge/        # Knowledge base lifecycle management
├── logging/          # Unified logging (loguru + custom handlers)
├── runtime/          # Orchestrator + capability/tool registries
├── services/         # Shared low-level services (LLM, RAG, session, etc.)
├── tools/            # Tool implementations (Level 1)
├── tutorbot/         # Multi-channel autonomous agent framework
├── utils/            # Shared utilities
└── co_writer/        # Co-writer agent module
```

---

## 2. API Layer

### `api/main.py`
- FastAPI app initialisation, CORS middleware, lifespan handler
- Validates tool/capability consistency on startup
- `SafeOutputStaticFiles` — whitelisted static file handler for `/api/outputs/*`
- Registers all routers under `/api/v1/`

### `api/run_server.py`
- Uvicorn server entry point

### Routers

| File | Mount | Purpose |
|------|-------|---------|
| `routers/unified_ws.py` | `/api/v1/ws` | Primary unified WebSocket — all capability execution |
| `routers/chat.py` | `/api/v1/chat` | Legacy chat REST + WebSocket |
| `routers/solve.py` | `/api/v1/solve` | Legacy Deep Solve WebSocket |
| `routers/question.py` | `/api/v1/question` | Exam question generation |
| `routers/research.py` | `/api/v1/research` | Deep Research |
| `routers/knowledge.py` | `/api/v1/knowledge` | KB CRUD, file upload, indexing progress |
| `routers/notebook.py` | `/api/v1/notebook` | Notebook CRUD + record management |
| `routers/book.py` | `/api/v1/book` | Book generation engine |
| `routers/co_writer.py` | `/api/v1/co_writer` | Co-writing agent |
| `routers/tutorbot.py` | `/api/v1/tutorbot` | TutorBot & soul management |
| `routers/memory.py` | `/api/v1/memory` | User memory files |
| `routers/sessions.py` | `/api/v1/sessions` | Session listing & deletion |
| `routers/settings.py` | `/api/v1/settings` | User interface settings |
| `routers/skills.py` | `/api/v1/skills` | SKILL.md management |
| `routers/dashboard.py` | `/api/v1/dashboard` | Dashboard activity data |
| `routers/vision_solver.py` | `/api/v1/vision-solver` | Image-based problem solving |
| `routers/attachments.py` | `/api/attachments` | Attachment file serving |
| `routers/system.py` | `/api/v1/system` | Health check, runtime topology, provider tests |
| `routers/plugins_api.py` | `/api/v1/plugins` | Plugin management |
| `routers/agent_config.py` | `/api/v1/agent-config` | Agent configuration |

### `api/utils/`
| File | Purpose |
|------|---------|
| `log_interceptor.py` | Captures log output and forwards it over WebSocket |
| `task_id_manager.py` | Generates unique task IDs |
| `progress_broadcaster.py` | Broadcasts KB indexing progress to WebSocket clients |
| `task_log_stream.py` | Per-task log stream management |

---

## 3. Core Abstractions

All in `deeptutor/core/` — imported by virtually every other module.

| File | Key Exports | Purpose |
|------|-------------|---------|
| `context.py` | `UnifiedContext`, `Attachment` | The single context object passed through the orchestrator to every capability |
| `capability_protocol.py` | `BaseCapability`, `CapabilityManifest` | Interface all capabilities implement |
| `tool_protocol.py` | `BaseTool`, `ToolDefinition`, `ToolParameter`, `ToolResult`, `ToolPromptHints` | Interface all tools implement |
| `stream.py` | `StreamEvent`, `StreamEventType` | Typed stream events emitted during capability execution |
| `stream_bus.py` | `StreamBus` | Async fan-out queue; capabilities publish here, WebSocket handlers subscribe |
| `trace.py` | Trace data structures | Metadata attached to stream events |

---

## 4. Agents

`deeptutor/agents/` — one sub-package per capability, plus a shared base.

### `base_agent.py`
`BaseAgent` — provides: LLM client access, token tracking, YAML prompt loading. All agents inherit from this.

### Per-Capability Agent Packages

| Package | Key Classes | Role |
|---------|-------------|------|
| `agents/chat/` | `ChatAgent`, `SessionManager`, `AgenticPipeline` | Lightweight chat with optional RAG |
| `agents/solve/` | `MainSolver`, `PlannerAgent`, `SolverAgent`, `WriterAgent`, `SolverSessionManager` | Multi-step problem solving pipeline |
| `agents/question/` | `AgentCoordinator`, `IdeaAgent`, `QuestionGenerator`, `FollowupAgent` | Exam question generation |
| `agents/research/` | `ResearchPipeline`, `DecomposeAgent`, `ResearchAgent`, `NoteAgent`, `ReportingAgent`, `CitationManager` | Deep research with citation tracking |
| `agents/visualize/` | `VisualizationPipeline`, `AnalysisAgent`, `CodeGeneratorAgent`, `ReviewAgent` | Data visualisation generation |
| `agents/math_animator/` | `Pipeline`, `ConceptAnalysisAgent`, `CodeGeneratorAgent` | Manim-based math animation |
| `agents/notebook/` | `NotebookSummarizeAgent`, `AnalysisAgent` | Notebook analysis & summarisation |
| `agents/vision_solver/` | `VisionSolverAgent` | Image-based problem solving |

---

## 5. Capabilities

`deeptutor/capabilities/` — thin orchestrators that wrap agents and emit to `StreamBus`.

| File | Class | Notes |
|------|-------|-------|
| `deep_solve.py` | `DeepSolveCapability` | Planner → Solver → Writer pipeline |
| `deep_question.py` | `DeepQuestionCapability` | Idea → Generator → Followup pipeline |
| `deep_research.py` | `DeepResearchCapability` | Decompose → Research → Note → Report |
| `chat.py` | `ChatCapability` | Conversational chat with optional tools |
| `visualize.py` | `VisualizeCapability` | Chart/diagram generation |
| `math_animator.py` | `MathAnimatorCapability` | Manim animation generation |
| `request_contracts.py` | — | Pydantic validators for each capability's request schema |

---

## 6. Runtime System

`deeptutor/runtime/`

| File | Class | Purpose |
|------|-------|---------|
| `orchestrator.py` | `ChatOrchestrator` | Core router: reads `active_capability` from context, resolves it via registry, calls `capability.run()` |
| `registry/capability_registry.py` | `CapabilityRegistry` | Singleton registry of all registered capabilities |
| `registry/tool_registry.py` | `ToolRegistry` | Singleton registry of all registered tools |
| `bootstrap/builtin_capabilities.py` | — | Registers all built-in capabilities at startup |
| `mode.py` | — | Execution mode helpers |

### Turn Runtime (`services/session/turn_runtime.py`)
Manages active turn lifecycle: creation, cancellation, resume, subscription fan-out.

---

## 7. Services Layer

`deeptutor/services/` — all shared, reusable infrastructure.

### LLM (`services/llm/`)
| File | Purpose |
|------|---------|
| `client.py` | Unified LLM client (wraps factory) |
| `factory.py` | Provider factory with retry logic |
| `config.py` | LLM configuration (model, API key, base URL) |
| `provider_core/` | Provider implementations: OpenAI-compat, Anthropic, Azure OpenAI |
| `providers/` | Routing/composite providers |
| `types.py` | Shared type definitions (`LLMResponse`, etc.) |
| `capabilities.py` | Model capability detection (vision, tools, reasoning) |
| `multimodal.py` | Image/file encoding for multimodal calls |
| `traffic_control.py` | Rate limiting, backoff, error classification |

**Idle timeout:** all streaming providers wrap each `__anext__()` with `asyncio.wait_for(..., timeout=90)`. Fires `asyncio.TimeoutError` → returns error `LLMResponse` if the stream stalls for >90 s.

### Embedding (`services/embedding/`)
| File | Purpose |
|------|---------|
| `client.py` | Unified embedding client |
| `config.py` | Embedding configuration |
| `adapters/` | OpenAI, Azure, Cohere, Jina, Ollama, SiliconFlow, Aliyun, Qwen, DashScope |
| `validation.py` | Batch validation |

### RAG (`services/rag/`)
| File | Purpose |
|------|---------|
| `service.py` | Main RAG service with event streaming |
| `factory.py` | Pipeline factory |
| `pipelines/llamaindex/` | LlamaIndex pipeline: document loading, chunking, vector storage |
| `smart_retriever.py` | Advanced retrieval strategies |
| `embedding_signature.py` | Embedding model version fingerprinting |
| `index_versioning.py` | Index version management |
| `file_routing.py` | Route documents by type to appropriate loader |

### Session & State (`services/session/`)
| File | Purpose |
|------|---------|
| `base_session_manager.py` | Abstract session manager interface |
| `unified_session_manager.py` | Single session manager for all capability modes |
| `sqlite_store.py` | SQLite backend: sessions, messages, turns tables |
| `turn_runtime.py` | Turn-based execution lifecycle management |
| `context_builder.py` | Build `UnifiedContext` from session history |

### Other Services
| Path | Purpose |
|------|---------|
| `services/memory/service.py` | Two-file memory system (`SUMMARY.md`, `PROFILE.md`) |
| `services/notebook/service.py` | Notebook JSON storage |
| `services/search/` | Web search providers (Brave, Tavily, Jina, DuckDuckGo, Perplexity, Exa, Serper, Baidu, Searxng, OpenRouter) |
| `services/search/consolidation.py` | Merge results from multiple search providers |
| `services/skill/service.py` | SKILL.md YAML frontmatter management |
| `services/storage/attachment_store.py` | Local/remote attachment storage protocol |
| `services/settings/interface_settings.py` | UI preference storage |
| `services/path_service.py` | All data/log directory path resolution |
| `services/prompt/manager.py` | YAML prompt template loading & rendering |
| `services/config/loader.py` | YAML config loading |
| `services/config/env_store.py` | Environment variable accessor with fallbacks |
| `services/config/model_catalog.py` | Model capability catalog |
| `services/config/provider_runtime.py` | Provider runtime specs |
| `services/config/context_window_detection.py` | Model context window size detection |
| `services/tutorbot/manager.py` | TutorBot lifecycle management |

---

## 8. Knowledge Base

`deeptutor/knowledge/`

| File | Class | Purpose |
|------|-------|---------|
| `manager.py` | `KnowledgeBaseManager` | KB lifecycle, embedding reconciliation, reindex |
| `initializer.py` | `KnowledgeBaseInitializer` | First-time KB creation from documents |
| `add_documents.py` | `DocumentAdder` | Incremental document addition |
| `naming.py` | — | `validate_knowledge_base_name()` |
| `progress_tracker.py` | `ProgressTracker` | Track & broadcast indexing progress |
| `kb_health.py` | — | KB health diagnostics |

**Storage:** `data/knowledge_bases/<kb_name>/` — LlamaIndex vector index versioned by embedding signature + `kb_index.json` metadata.

---

## 9. TutorBot System

`deeptutor/tutorbot/` — autonomous multi-channel agent with its own agent loop, separate from the main capability pipeline.

### Agent Core (`tutorbot/agent/`)
| File | Class | Purpose |
|------|-------|---------|
| `loop.py` | `AgentLoop` | Main cycle: message → context → LLM → tool calls → response |
| `context.py` | `ContextBuilder` | Build prompt context from history, memory, skills |
| `memory.py` | `MemoryConsolidator` | Persist/compress memory to `SOUL.md` files |
| `skills.py` | — | Skill injection into agent context |
| `subagent.py` | `SubagentManager` | Spawn and coordinate sub-agents |
| `team/` | `TeamManager`, `Board`, `Mailbox` | Multi-agent team coordination |
| `tools/` | `ShellTool`, `WebTool`, `MCPTool`, `FilesystemTool`, `SpawnTool`, `CronTool` | Tool implementations for TutorBot agents |

### Channels (`tutorbot/channels/`)
Slack, Discord, Telegram, WeChat, DingTalk, Feishu, QQ, WhatsApp, Email, Matrix/Element

### Other TutorBot Modules
| Path | Purpose |
|------|---------|
| `tutorbot/providers/` | LLM provider adapters (OpenAI-compat, Anthropic) |
| `tutorbot/session/` | TutorBot session storage (SQLite per bot) |
| `tutorbot/cron/` | Cron job scheduling (`CronService`) |
| `tutorbot/heartbeat/` | Periodic health checks (`HeartbeatService`) |
| `tutorbot/bus/` | Internal message bus for inter-component events |

**Bot workspace:** `data/user/workspace/tutorbot/<bot_id>/` — `SOUL.md`, `TOOLS.md`, `USER.md`, `session.db`

---

## 10. Tools

`deeptutor/tools/` — Level 1 tools registered in `ToolRegistry`, used by agents via function calling.

| File/Package | Key Function | Purpose |
|------|---------|---------|
| `web_search.py` | `web_search()` | Web search (re-exports from `services/search`) |
| `rag_tool.py` | `rag_search()`, `initialize_rag()` | RAG query wrapper |
| `code_executor.py` | — | Sandboxed Python execution |
| `reason.py` | — | Reasoning/reflection step |
| `brainstorm.py` | — | Idea generation |
| `paper_search_tool.py` | — | Academic paper search (ArXiv) |
| `tex_chunker.py` | — | LaTeX document chunking |
| `tex_downloader.py` | — | LaTeX document downloading |
| `question/` | `QuestionExtractor`, `PDFParser` | Question extraction from PDFs |
| `vision/` | — | Image parsing, GeoGebra integration |

---

## 11. Book Module

`deeptutor/book/`

| File | Class | Purpose |
|------|-------|---------|
| `engine.py` | `BookEngine` | Top-level book generation orchestrator |
| `compiler.py` | `BookCompiler` | Compile individual pages |
| `storage.py` | — | JSON persistence for book projects |
| `streaming.py` | — | Real-time generation event streaming |
| `blocks/` | `BaseBlock` + subclasses | Content block types: text, code, quiz, flash cards, etc. |
| `agents/` | `SpineAgent`, `PagePlanner`, etc. | Ideation, spine planning, page synthesis |

**Storage:** `data/user/workspace/books/<book_id>/`

---

## 12. Co-Writer Module

`deeptutor/co_writer/`

| File | Class | Purpose |
|------|-------|---------|
| `edit_agent.py` | `EditAgent` | Text editing with history tracking & statistics |
| `storage.py` | `CoWriterDocument` | JSON document storage |

**Storage:** `data/user/workspace/co_writer/`

---

## 13. Configuration

`deeptutor/config/`

| File | Purpose |
|------|---------|
| `schema.py` | `AppConfig`, `LLMConfig`, `PathsConfig` — Pydantic config schema |
| `settings.py` | `Settings` dataclass singleton with defaults |
| `defaults.py` | Default constant values |
| `constants.py` | System-wide constants |
| `accessors.py` | Helper functions to read config values |

**Config files (YAML):**
- `config/main.yaml` — system language, paths, logging
- `config/agents.yaml` — per-agent temperature, max_tokens
- `config/llm_providers.yaml` — provider model catalog
- `agents/*/prompts/{en,zh}/*.yaml` — per-agent prompt templates

---

## 14. Event System

`deeptutor/events/`

| File | Class | Purpose |
|------|-------|---------|
| `event_bus.py` | `EventBus`, `Event`, `EventType` | Async pub/sub for cross-module events (`CAPABILITY_COMPLETE`, `SOLVE_COMPLETE`, etc.) |

---

## 15. Logging

`deeptutor/logging/`

| Path | Purpose |
|------|---------|
| `config.py` | Logging initialisation |
| `logger.py` | `get_logger()` factory |
| `handlers/` | Console, file, and WebSocket log handlers |
| `adapters/` | LlamaIndex log adapter |
| `stats/llm_stats.py` | `LLMStats` — per-request token and latency tracking |

---

## 16. Database & Storage

### SQLite (`data/user/workspace/chat_history.db`)

```sql
sessions (id, title, created_at, updated_at, compressed_summary, summary_up_to_msg_id, preferences_json)
messages (id, session_id→sessions, role, content, capability, events_json, attachments_json, created_at)
turns    (id, session_id→sessions, capability, status, error, created_at, updated_at, finished_at)
```

### File-Based Storage

| Path | Contents |
|------|---------|
| `data/user/workspace/memory/SUMMARY.md` | Auto-updated learning journey summary |
| `data/user/workspace/memory/PROFILE.md` | Auto-updated user profile & preferences |
| `data/user/workspace/skills/<name>/` | `SKILL.md` with YAML frontmatter + `.tags.json` |
| `data/user/workspace/notebook/` | One JSON file per notebook + `notebooks_index.json` |
| `data/user/workspace/co_writer/` | Co-writer documents (JSON) |
| `data/user/workspace/books/<book_id>/` | Book project state (JSON) |
| `data/user/workspace/chat/attachments/<session_id>/` | Chat file attachments |
| `data/user/workspace/tutorbot/<bot_id>/` | Per-bot workspace + `session.db` |
| `data/knowledge_bases/<kb_name>/` | LlamaIndex vector index + `kb_index.json` |

### Pydantic Data Models (key ones)

| Model | Location | Purpose |
|-------|---------|---------|
| `UnifiedContext` | `core/context.py` | Full turn context passed through the pipeline |
| `Attachment` | `core/context.py` | File/image attachment metadata |
| `NotebookRecord` / `Notebook` | `services/notebook/` | Notebook data |
| `CoWriterDocument` | `co_writer/storage.py` | Co-writing session state |
| `SkillDetail` / `SkillInfo` | `services/skill/` | Skill metadata |
| `TurnRecord` | `services/session/sqlite_store.py` | Turn execution record |

---

## 17. API Endpoints Reference

### Unified WebSocket (primary runtime)
```
WS  /api/v1/ws                          Unified capability execution
    message types: message, subscribe_turn, subscribe_session,
                   resume_from, cancel_turn, regenerate
```

### Sessions
```
GET    /api/v1/sessions                 List all sessions
GET    /api/v1/sessions/{id}            Get session detail
DELETE /api/v1/sessions/{id}            Delete session
```

### Knowledge Base
```
GET    /api/v1/knowledge                List KBs
POST   /api/v1/knowledge                Create KB
GET    /api/v1/knowledge/{kb}           KB details
DELETE /api/v1/knowledge/{kb}           Delete KB
POST   /api/v1/knowledge/{kb}/documents Upload documents
WS     /api/v1/knowledge/{kb}/upload    Upload with progress stream
POST   /api/v1/knowledge/{kb}/reindex   Reindex
GET    /api/v1/knowledge/supported-types Supported file types
POST   /api/v1/knowledge/{kb}/link-folder Link local folder
POST   /api/v1/knowledge/{kb}/health    Health check
```

### Notebook
```
GET    /api/v1/notebook                 List notebooks
POST   /api/v1/notebook                 Create notebook
GET    /api/v1/notebook/{id}            Get notebook
PUT    /api/v1/notebook/{id}            Update notebook
DELETE /api/v1/notebook/{id}            Delete notebook
POST   /api/v1/notebook/{id}/records    Add record (streaming summary)
PUT    /api/v1/notebook/{id}/records/{rid} Update record
DELETE /api/v1/notebook/{id}/records/{rid} Delete record
```

### Book
```
POST   /api/v1/book                     Create book
GET    /api/v1/book/{id}                Get book
POST   /api/v1/book/{id}/confirm-proposal Confirm structure proposal
POST   /api/v1/book/{id}/confirm-spine  Confirm outline
POST   /api/v1/book/{id}/compile        Compile page
POST   /api/v1/book/{id}/regenerate-block Regenerate block
POST   /api/v1/book/{id}/insert-block   Insert block
DELETE /api/v1/book/{id}/block          Delete block
POST   /api/v1/book/{id}/move-block     Reorder blocks
POST   /api/v1/book/{id}/change-block-type Change block type
WS     /api/v1/book/{id}               Real-time generation stream
```

### Co-Writer
```
POST   /api/v1/co_writer/edit           Edit text
POST   /api/v1/co_writer/react-edit     AI-assisted editing with thinking
POST   /api/v1/co_writer/auto-mark      Auto-markup
GET    /api/v1/co_writer/documents      List documents
POST   /api/v1/co_writer/documents      Create document
GET    /api/v1/co_writer/documents/{id} Get document
PUT    /api/v1/co_writer/documents/{id} Update document
DELETE /api/v1/co_writer/documents/{id} Delete document
```

### TutorBot
```
GET    /api/v1/tutorbot                 List bots
POST   /api/v1/tutorbot                 Create bot
GET    /api/v1/tutorbot/{id}            Get bot
PUT    /api/v1/tutorbot/{id}            Update bot
DELETE /api/v1/tutorbot/{id}            Delete bot
POST   /api/v1/tutorbot/{id}/start      Start bot
POST   /api/v1/tutorbot/{id}/stop       Stop bot
WS     /api/v1/tutorbot/{id}           Bot messaging
GET    /api/v1/tutorbot/souls           List soul templates
POST   /api/v1/tutorbot/souls           Create soul
GET    /api/v1/tutorbot/souls/{id}      Get soul
PUT    /api/v1/tutorbot/souls/{id}      Update soul
DELETE /api/v1/tutorbot/souls/{id}      Delete soul
GET    /api/v1/tutorbot/{id}/files/{type} Get bot file (SOUL.md, etc.)
PUT    /api/v1/tutorbot/{id}/files/{type} Update bot file
```

### Other Endpoints
```
GET/PUT /api/v1/memory                  User memory files
GET/PUT /api/v1/settings                Interface settings
GET/POST/PUT/DELETE /api/v1/skills      Skill management
GET     /api/v1/system/status           System health
GET     /api/v1/system/runtime-topology Execution topology
POST    /api/v1/system/test/llm         Test LLM connection
POST    /api/v1/system/test/embeddings  Test embedding connection
POST    /api/v1/system/test/search      Test search provider
WS      /api/v1/vision-solver           Vision problem solving
GET     /api/attachments/{sid}/{aid}/{name} Serve attachment
GET/POST/DELETE /api/v1/plugins         Plugin management
GET/PUT /api/v1/agent-config            Agent configuration
```

---

## 18. External Integrations

### LLM Providers
| Provider | Notes |
|----------|-------|
| OpenAI | GPT-4o, GPT-4, GPT-3.5 |
| Azure OpenAI | Enterprise, uses Responses API |
| Anthropic | Claude 3.x / Claude 4.x |
| OpenAI-compatible | Ollama, vLLM, LM Studio, DeepSeek, etc. |
| Gemini | Google Gemini API |
| Perplexity | Perplexity API |

### Embedding Providers
OpenAI, Azure OpenAI, Cohere, Jina, Ollama, SiliconFlow, Aliyun DashScope, custom OpenAI-compatible endpoints.

### Web Search Providers
Brave, Tavily, Jina, Searxng, DuckDuckGo, Perplexity, Baidu, Exa, Serper, OpenRouter.

### Messaging Channels (TutorBot)
Slack, Discord, Telegram, WeChat (MoChatAPI), DingTalk, Feishu, QQ, WhatsApp, Email (SMTP), Matrix/Element.

### Other
- **LlamaIndex** — RAG framework (vector store, document loaders, chunking)
- **PyMuPDF / pypdf / python-docx / openpyxl / python-pptx** — document parsing
- **ArXiv** — academic paper search
- **Manim** — math animation (optional)
- **MCP (Model Context Protocol)** — extensible external tool interface
- **GeoGebra** — geometry validation in vision solver

---

## 19. Environment Variables

```bash
# Server
BACKEND_PORT=8001
FRONTEND_PORT=3782

# LLM (required)
LLM_BINDING=openai
LLM_MODEL=gpt-4o-mini
LLM_API_KEY=sk-...
LLM_HOST=https://api.openai.com/v1
LLM_API_VERSION=

# Embedding (required for KB features)
EMBEDDING_BINDING=openai
EMBEDDING_MODEL=text-embedding-3-large
EMBEDDING_API_KEY=sk-...
EMBEDDING_HOST=https://api.openai.com/v1/embeddings
EMBEDDING_DIMENSION=
EMBEDDING_SEND_DIMENSIONS=

# Per-provider keys (optional fallbacks)
SILICONFLOW_API_KEY=
DASHSCOPE_API_KEY=
COHERE_API_KEY=
JINA_API_KEY=

# Web search (optional)
SEARCH_PROVIDER=
SEARCH_API_KEY=
SEARCH_BASE_URL=

# Deployment
NEXT_PUBLIC_API_BASE_EXTERNAL=
NEXT_PUBLIC_API_BASE=

# Misc
DISABLE_SSL_VERIFY=false
CHAT_ATTACHMENT_DIR=
```

---

## 20. Background Tasks & Workers

| Task | Trigger | Location |
|------|---------|---------|
| WebSocket event pusher | Per active WebSocket connection | `api/routers/unified_ws.py` |
| Log capture & forward | Per turn | `api/utils/log_interceptor.py` |
| KB indexing pipeline | `POST /knowledge/{kb}/documents` | `knowledge/initializer.py` |
| Embedding reconciliation | KB access when embedding config changed | `knowledge/manager.py` |
| Progress broadcaster | During KB indexing | `api/utils/progress_broadcaster.py` |
| Memory consolidation | After each turn completes | `services/memory/service.py` |
| SQLite async writer | Continuous | `services/session/sqlite_store.py` |
| TutorBot agent loop | Per running bot | `tutorbot/agent/loop.py` |
| TutorBot cron scheduler | System startup (if bots configured) | `tutorbot/cron/` |
| TutorBot heartbeat | System startup | `tutorbot/heartbeat/` |
| Channel receivers | Per enabled channel per bot | `tutorbot/channels/` |
| Event bus processor | System startup | `events/event_bus.py` |

---

## 21. Middleware & Dependencies

### FastAPI Middleware
- **CORS** — all origins (configurable)
- **Selective access logging** — non-200 responses only
- **WebSocket noise filter** — suppresses connection churn logs
- **SafeOutputStaticFiles** — path-whitelisted static file serving

### Key Python Dependencies
| Package | Role |
|---------|------|
| `fastapi` + `uvicorn` | Web framework & ASGI server |
| `pydantic` | Data validation & config schemas |
| `aiosqlite` | Async SQLite |
| `httpx` / `aiohttp` | Async HTTP clients |
| `openai` | OpenAI SDK |
| `anthropic` | Anthropic SDK |
| `llama-index` | RAG framework |
| `PyMuPDF` | PDF extraction |
| `loguru` | Structured logging |
| `PyYAML` | Config & prompt loading |
| `tenacity` | Retry with backoff |
| `tiktoken` | OpenAI token counting |
| `python-dotenv` | `.env` loading |
| `croniter` | Cron schedule parsing |
| `mcp` | Model Context Protocol |
| `slack-sdk` | Slack channel |
| `python-telegram-bot` | Telegram channel |
| `discord.py` | Discord channel |
| `lark-oapi` | Feishu/DingTalk |
| `manim` | Math animation (optional) |
| `matrix-nio` | Matrix/Element (optional) |

---

## 22. Architecture & Data Flow

### Unified WebSocket Message Flow
```
Client  →  /api/v1/ws
           ↓
        unified_ws.py  (parse message type)
           ↓
        TurnRuntimeManager  (create/resume turn)
           ↓
        ChatOrchestrator  (resolve capability from context)
           ↓
        Capability.run()  (e.g. DeepSolveCapability)
           ├─ emits events → StreamBus
           ├─ calls Tools → Services
           └─ writes session → SQLiteStore
           ↓
        StreamBus fan-out → WebSocket → Client
           ↓
        EventBus.publish(CAPABILITY_COMPLETE)
```

### RAG Query Flow
```
User query
  → RAGService
  → LlamaIndex pipeline
      ├─ embed query (EmbeddingClient)
      ├─ vector search (local index)
      └─ rerank top-k docs
  → SmartRetriever (optional advanced strategies)
  → context injected into LLM prompt
```

### TutorBot Agent Loop
```
Channel event (Slack / Telegram / etc.)
  → Channel receiver
  → AgentLoop
      ├─ ContextBuilder (history + memory + skills)
      ├─ LLM call with tool schemas
      ├─ parse tool calls
      └─ execute tools (web, shell, filesystem, MCP, spawn)
  → MemoryConsolidator (update SOUL.md)
  → response sent to channel
```

### Module Dependency Hierarchy
```
core                    ← no internal deps
  ↑
services                ← depend on core
  ↑
tools                   ← depend on services
  ↑
agents                  ← depend on services + tools
  ↑
capabilities            ← depend on agents + tools
  ↑
runtime                 ← depends on capabilities
  ↑
api routers             ← depend on runtime + services

tutorbot                ← parallel system, its own stack
```
