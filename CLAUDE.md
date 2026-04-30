# DeepTutor — Developer Reference

## Project Overview

DeepTutor is an agent-native intelligent learning companion. It exposes three
entry points — a Typer CLI, a FastAPI/WebSocket API, and a Python SDK — and
routes all requests through a two-layer plugin model: **Tools** (single-function
LLM-callable tools) and **Capabilities** (multi-step agent pipelines).

Current version: **v1.3.2**

---

## Common Commands

```bash
# Install for local dev (includes server + test tools)
pip install -e ".[dev]"

# Run the API server
deeptutor serve --port 8001

# Interactive chat REPL
deeptutor chat

# Run tests
pytest

# Run a specific test file
pytest tests/services/embedding/test_client_runtime.py -v

# Lint / pre-commit
pre-commit run --all-files
```

---

## Repository Layout

```
deeptutor/              # Main Python package
  agents/               # Agent implementations
    chat/               # Default conversational agent
    solve/              # Deep Solve (plan → reason → write)
    question/           # Deep Question generation
    research/           # Deep Research multi-agent
    vision_solver/      # Vision-based problem solver
    math_animator/      # Manim animation agent
    notebook/           # Notebook agent
    visualize/          # Visualization agent
  capabilities/         # Built-in capability wrappers (thin adapters)
  tools/builtin/        # Built-in tool wrappers
  plugins/              # Playground plugins (deep_research, etc.)
  runtime/              # ChatOrchestrator, RunMode, registries
  api/                  # FastAPI routers + WebSocket endpoint
  core/                 # Protocols & event bus (stream, context, tool/cap protocol)
  services/
    config/             # Configuration helpers
    embedding/          # Embedding client + provider adapters
    llm/                # LLM provider integration
    memory/             # User memory (PROFILE.md / SUMMARY.md)
    rag/                # RAG pipelines (LlamaIndex)
    search/             # Web search providers
    session/            # Session storage (SQLite)
    storage/            # File/blob storage backends
    notebook/           # Notebook storage
    skill/              # Skill loader
    tutorbot/           # TutorBot-specific services
  tutorbot/             # TutorBot agent engine (channels, scheduling)
deeptutor_cli/          # Typer CLI entry point
tests/                  # pytest test suite (mirrors deeptutor/)
web/                    # Next.js frontend (TypeScript / Tailwind)
scripts/                # Utility scripts (setup tour, migration, testing helpers)
assets/
  releases/             # Per-version release notes (ver*.md)
  README/               # Localised README translations
requirements/           # Flat pip requirements mirrors for Docker/CI
```

---

## Architecture

```
Entry Points:  CLI (Typer)  |  WebSocket /api/v1/ws  |  Python SDK
                    ↓                   ↓                   ↓
              ┌─────────────────────────────────────────────────┐
              │              ChatOrchestrator                    │
              │   routes to ChatCapability (default)             │
              │   or a selected deep Capability                  │
              └──────────┬──────────────┬───────────────────────┘
                         │              │
              ┌──────────▼──┐  ┌────────▼──────────┐
              │ ToolRegistry │  │ CapabilityRegistry │
              │  (Level 1)   │  │   (Level 2)        │
              └──────────────┘  └────────────────────┘
```

### Level 1 — Tools

| Tool                | Description                                         |
| ------------------- | --------------------------------------------------- |
| `rag`               | Knowledge base retrieval (RAG)                      |
| `web_search`        | Web search with citations                           |
| `code_execution`    | Sandboxed Python execution                          |
| `reason`            | Dedicated deep-reasoning LLM call                  |
| `brainstorm`        | Breadth-first idea exploration                      |
| `paper_search`      | arXiv academic paper search                         |
| `geogebra_analysis` | Image → GeoGebra commands (4-stage vision pipeline) |

### Level 2 — Capabilities

| Capability      | Stages                                            |
| --------------- | ------------------------------------------------- |
| `chat`          | responding (default, tool-augmented)              |
| `deep_solve`    | planning → reasoning → writing                    |
| `deep_question` | ideation → evaluation → generation → validation  |

---

## Key Files

| Path | Purpose |
| ---- | ------- |
| `deeptutor/runtime/orchestrator.py` | ChatOrchestrator — unified entry point |
| `deeptutor/core/stream.py` | StreamEvent protocol |
| `deeptutor/core/stream_bus.py` | Async event fan-out |
| `deeptutor/core/tool_protocol.py` | BaseTool abstract class |
| `deeptutor/core/capability_protocol.py` | BaseCapability abstract class |
| `deeptutor/core/context.py` | UnifiedContext dataclass |
| `deeptutor/runtime/registry/tool_registry.py` | Tool discovery & registration |
| `deeptutor/runtime/registry/capability_registry.py` | Capability discovery & registration |
| `deeptutor/runtime/mode.py` | RunMode (CLI vs SERVER) |
| `deeptutor/capabilities/` | Built-in capability wrappers |
| `deeptutor/tools/builtin/` | Built-in tool wrappers |
| `deeptutor/plugins/` | Playground plugins |
| `deeptutor/plugins/loader.py` | Plugin discovery from manifest.yaml |
| `deeptutor_cli/main.py` | Typer CLI entry point |
| `deeptutor/api/routers/unified_ws.py` | Unified WebSocket endpoint |
| `deeptutor/services/config/embedding_endpoint.py` | Embedding endpoint URL helpers (v1.3.2) |
| `deeptutor/services/config/model_catalog.py` | Embedding/LLM provider catalog |
| `deeptutor/services/config/provider_runtime.py` | Runtime config resolution |
| `deeptutor/services/embedding/client.py` | Unified embedding client |
| `deeptutor/services/embedding/adapters/` | Per-provider embedding adapters |
| `deeptutor/services/rag/pipelines/llamaindex/pipeline.py` | LlamaIndex RAG pipeline |
| `deeptutor/services/rag/pipelines/llamaindex/storage.py` | Index storage + vector validation |
| `deeptutor/services/rag/pipelines/llamaindex/embedding_adapter.py` | LlamaIndex ↔ embedding client bridge |
| `deeptutor/services/rag/pipelines/llamaindex/errors.py` | RAG error classification |
| `deeptutor/services/memory/service.py` | User memory (PROFILE.md / SUMMARY.md) |
| `deeptutor/agents/solve/main_solver.py` | Deep Solve orchestrator |
| `deeptutor/agents/solve/agents/solver_agent.py` | Deep Solve ReAct solver |

---

## Embedding System (v1.3.2)

Embedding configuration is now **endpoint-transparent**: the URL stored in
Settings and the URL actually POSTed to at runtime are the same string.

### Provider URL Helpers (`services/config/embedding_endpoint.py`)

- `normalize_embedding_endpoint_for_display(provider, base_url)` — converts a
  legacy `/v1`-style base URL to the full endpoint path, or returns the
  provider default if blank.
- `EMBEDDING_PROVIDER_DEFAULT_ENDPOINTS` — maps provider name → full endpoint URL.
- `canonical_embedding_provider_name(name)` — normalises aliases (e.g.
  `"lm_studio"` → `"vllm"`, `"google"` → `"openai"`).

### Adapter Routing (`services/embedding/client.py`)

`EmbeddingClient` resolves the active provider name and dispatches to the
matching adapter:

| Adapter | File |
| ------- | ---- |
| OpenAI SDK | `adapters/openai_sdk.py` |
| OpenAI-compatible HTTP | `adapters/openai_compatible.py` |
| Jina, Cohere, Ollama, etc. | individual adapter files |

OpenRouter now uses the exact-URL HTTP adapter; `custom_openai_sdk` is hidden
from the public provider dropdown but still resolves for saved profiles.

---

## RAG Pipeline (v1.3.2)

Located in `deeptutor/services/rag/pipelines/llamaindex/`.

**State refresh**: `initialize`, `search`, and incremental `add` all call
`reconfigure_embedding()` before use, so a long-lived pipeline picks up
Settings changes without a restart.

**Vector validation** (`storage.py`): persisted indexes are checked for null,
non-numeric, or shape-inconsistent vectors before similarity search. An invalid
index returns `{"needs_reindex": true, ...}` with a user-readable explanation
instead of a raw NumPy traceback.

**Error classification** (`errors.py`): `classify_rag_error()` maps known
failure strings (e.g. `"unsupported operand type(s) for *: 'NoneType' and
'float'"`) to structured responses.

---

## Memory Service (v1.3.2)

`deeptutor/services/memory/service.py` manages two durable files per user:

| File | Content |
| ---- | ------- |
| `PROFILE.md` | User identity, preferences, knowledge levels |
| `SUMMARY.md` | Running summary of the learning journey |

**Thinking-tag cleanup**: all writes pass through `clean_thinking_tags()` (from
`deeptutor.services.llm`) after code-fence stripping. This prevents `<think>` /
`<thinking>` blocks from reasoning models from being persisted in memory files.

**Self-repair on read**: if an existing memory file contains closed or unclosed
thinking tags, the read path cleans and rewrites the file.

---

## Dependency Layers

```
.[cli]        — LLM + RAG + providers + document parsing
.[server]     — .[cli] + FastAPI/uvicorn
.[tutorbot]   — .[server] + TutorBot engine + channel SDKs
.[matrix]     — Matrix protocol support (matrix-nio[e2e])
.[math-animator] — Manim addon
.[dev]        — .[server] + pytest + pre-commit + bandit
.[all]        — Everything
```

Defined in `pyproject.toml [project.optional-dependencies]`. Mirrored as flat
lists in `requirements/*.txt` for Docker/CI installs.

---

## Plugin Development

Create `deeptutor/plugins/<name>/` with:

```
manifest.yaml     # name, version, type, description, stages
capability.py     # class extending BaseCapability
```

Minimal `capability.py`:

```python
from deeptutor.core.capability_protocol import BaseCapability, CapabilityManifest
from deeptutor.core.context import UnifiedContext
from deeptutor.core.stream_bus import StreamBus

class MyPlugin(BaseCapability):
    manifest = CapabilityManifest(
        name="my_plugin",
        description="My custom plugin",
        stages=["step1", "step2"],
    )

    async def run(self, context: UnifiedContext, stream: StreamBus) -> None:
        async with stream.stage("step1", source=self.name):
            await stream.content("Working on step 1...", source=self.name)
        await stream.result({"response": "Done!"}, source=self.name)
```

---

## Release Notes

Per-version release notes live in `assets/releases/`. The most recent releases:

| Version | Date | Summary |
| ------- | ---- | ------- |
| v1.3.2 | 2026.04.29 | Transparent embedding endpoints, RAG re-index resilience, memory thinking-tag cleanup, Deep Solve fix |
| v1.3.1 | 2026.04.28 | Safer RAG routing & embedding validation, Docker persistence, IME-safe input, Windows/GBK robustness |
| v1.3.0 | 2026.04.27 | Versioned KB indexes with re-index workflow, rebuilt Knowledge workspace, embedding auto-discovery |
