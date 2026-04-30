# DeepTutor — Agent-Native Architecture

## Overview

DeepTutor is an **agent-native** intelligent learning companion built around
a two-layer plugin model (Tools + Capabilities) with three entry points:
CLI, WebSocket API, and Python SDK.

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

Lightweight single-function tools the LLM calls on demand:

| Tool                | Description                                    |
| ------------------- | ---------------------------------------------- |
| `rag`               | Knowledge base retrieval (RAG)                 |
| `web_search`        | Web search with citations                      |
| `code_execution`    | Sandboxed Python execution                     |
| `reason`            | Dedicated deep-reasoning LLM call              |
| `brainstorm`        | Breadth-first idea exploration with rationale  |
| `paper_search`      | arXiv academic paper search                    |
| `geogebra_analysis` | Image → GeoGebra commands (4-stage vision pipeline) |

### Level 2 — Capabilities

Multi-step agent pipelines that take over the conversation:

| Capability       | Stages                                         |
| ---------------- | ---------------------------------------------- |
| `chat`           | responding (default, tool-augmented)           |
| `deep_solve`     | planning → reasoning → writing                 |
| `deep_question`  | ideation → evaluation → generation → validation |

### Playground Plugins

Extended features in `deeptutor/plugins/`:

| Plugin            | Type       | Description                          |
| ----------------- | ---------- | ------------------------------------ |
| `deep_research`   | playground | Multi-agent research + reporting     |

## CLI Usage

```bash
# Install CLI
pip install -e ".[cli]"

# Run any capability (agent-first entry point)
deeptutor run chat "Explain Fourier transform"
deeptutor run deep_solve "Solve x^2=4" -t rag --kb my-kb
deeptutor run deep_question "Linear algebra" --config num_questions=5

# Interactive REPL
deeptutor chat
# (inside the REPL: /regenerate or /retry re-runs the last user message)

# Knowledge bases
deeptutor kb list
deeptutor kb create my-kb --doc textbook.pdf

# Plugins & memory
deeptutor plugin list
deeptutor memory show

# API server (requires .[server])
deeptutor serve --port 8001
```

## Key Files

| Path                          | Purpose                              |
| ----------------------------- | ------------------------------------ |
| `deeptutor/runtime/orchestrator.py` | ChatOrchestrator — unified entry     |
| `deeptutor/core/stream.py`          | StreamEvent protocol                 |
| `deeptutor/core/stream_bus.py`      | Async event fan-out                  |
| `deeptutor/core/tool_protocol.py`   | BaseTool abstract class              |
| `deeptutor/core/capability_protocol.py` | BaseCapability abstract class    |
| `deeptutor/core/context.py`         | UnifiedContext dataclass             |
| `deeptutor/runtime/registry/tool_registry.py` | Tool discovery & registration |
| `deeptutor/runtime/registry/capability_registry.py` | Capability discovery & registration |
| `deeptutor/runtime/mode.py`         | RunMode (CLI vs SERVER)              |
| `deeptutor/capabilities/`           | Built-in capability wrappers         |
| `deeptutor/tools/builtin/`          | Built-in tool wrappers               |
| `deeptutor/plugins/`                | Playground plugins                   |
| `deeptutor/plugins/loader.py`       | Plugin discovery from manifest.yaml  |
| `deeptutor_cli/main.py`             | Typer CLI entry point                |
| `deeptutor/api/routers/unified_ws.py` | Unified WebSocket endpoint         |
| `deeptutor/services/config/embedding_endpoint.py` | Embedding endpoint URL normalisation & provider defaults |
| `deeptutor/services/config/model_catalog.py` | Embedding/LLM provider catalog with persisted normalisation |
| `deeptutor/services/embedding/client.py` | Unified embedding client; validates endpoint before indexing |
| `deeptutor/services/rag/pipelines/llamaindex/pipeline.py` | LlamaIndex RAG orchestration with per-call state refresh |
| `deeptutor/services/rag/pipelines/llamaindex/storage.py` | Persisted index storage + invalid-vector detection |
| `deeptutor/services/rag/pipelines/llamaindex/errors.py` | RAG error classification → `needs_reindex` hints |
| `deeptutor/services/memory/service.py` | User memory (PROFILE.md / SUMMARY.md) with thinking-tag cleanup |
| `deeptutor/agents/solve/main_solver.py` | Deep Solve orchestrator |
| `deeptutor/agents/solve/agents/solver_agent.py` | Deep Solve ReAct solver |

## Plugin Development

Create a directory under `deeptutor/plugins/<name>/` with:

```
manifest.yaml     # name, version, type, description, stages
capability.py     # class extending BaseCapability
```

Minimal `manifest.yaml`:
```yaml
name: my_plugin
version: 0.1.0
type: playground
description: "My custom plugin"
stages: [step1, step2]
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

## Dependency Layers

Defined in `pyproject.toml` `[project.optional-dependencies]`. Mirrored as flat
lists in `requirements/*.txt` for Docker/CI installs without source code.

```
.[cli]            — CLI full (LLM + RAG + providers + document parsing)
.[server]         — .[cli] + FastAPI/uvicorn (for Web/API)
.[tutorbot]       — .[server] + TutorBot agent engine + channel SDKs
.[matrix]         — Matrix channel for TutorBot (matrix-nio[e2e]; needs libolm)
.[math-animator]  — Manim addon (for `deeptutor animate`)
.[dev]            — .[server] + test/lint tools
.[all]            — Everything above
```

## Notable Behaviour (v1.3.2+)

### Embedding Endpoint Transparency

Embedding adapters POST to the URL stored in Settings exactly — no hidden path
appending at request time. `services/config/embedding_endpoint.py` normalises
legacy `/v1`-style base URLs to their full endpoint form on catalog load and
persists the result. The `EmbeddingClient` rejects known-provider URLs that
point to a root/base path before indexing starts.

### RAG State Refresh

`LlamaIndexPipeline.initialize()`, `search()`, and `add()` all call
`reconfigure_embedding()` before use, and the `CustomEmbedding` adapter
recreates the underlying client whenever the resolved runtime config changes.
This means Settings changes take effect without restarting the server.

Persisted indexes are validated for null or non-finite vectors before retrieval.
An invalid index returns `{"needs_reindex": true}` with a user-facing
explanation rather than a raw exception.

### Memory Thinking-Tag Cleanup

All writes to `PROFILE.md` and `SUMMARY.md` pass through `clean_thinking_tags()`
after code-fence stripping, preventing `<think>` / `<thinking>` scratchpad
blocks from reasoning models from being persisted. Existing files containing
these tags are cleaned and rewritten on the next read.

### Deep Solve Wiring

`MainSolver` no longer passes the stale `attachments` keyword to
`SolverAgent.process()`; attachments are forwarded only on the planner and
replan calls that accept them.
