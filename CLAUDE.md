# DeepTutor — Developer Quick-Start

> For comprehensive architecture detail see **[BACKEND_COMPONENTS.md](./BACKEND_COMPONENTS.md)**.

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

# Run the full test suite
pytest

# Lint / pre-commit
pre-commit run --all-files
```

---

## Repository Layout (top-level)

```
deeptutor/          Main Python package (api/, agents/, capabilities/,
                    core/, runtime/, services/, tools/, tutorbot/, …)
deeptutor_cli/      Typer CLI entry point
tests/              pytest suite (mirrors deeptutor/ structure)
web/                Next.js frontend (TypeScript / Tailwind)
scripts/            Utility scripts (setup tour, migration, LLM/embedding tests)
assets/releases/    Per-version release notes (ver*.md)
requirements/       Flat pip requirements files for Docker/CI
```

---

## Architecture in One Diagram

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

See `BACKEND_COMPONENTS.md §6` for the full runtime system and `§22` for
end-to-end data flow diagrams.

---

## Dependency Layers

```
.[cli]           — LLM + RAG + providers + document parsing
.[server]        — .[cli] + FastAPI/uvicorn
.[tutorbot]      — .[server] + TutorBot engine + channel SDKs
.[matrix]        — Matrix protocol (matrix-nio[e2e])
.[math-animator] — Manim addon
.[dev]           — .[server] + pytest + pre-commit + bandit
.[all]           — Everything
```

Defined in `pyproject.toml`. Mirrored in `requirements/*.txt` for Docker/CI.

---

## Key Behaviour Changes in v1.3.2

### Transparent Embedding Endpoint URLs

`deeptutor/services/config/embedding_endpoint.py` (new) provides helpers that
normalise embedding URLs end-to-end:

- Settings shows and saves the **exact URL** that DeepTutor POSTs to.
- Legacy `/v1`-style base URLs are migrated to full endpoint paths on catalog
  load and persisted back.
- `EmbeddingClient` rejects known-provider root/base URLs before indexing
  starts, with an actionable error.
- OpenRouter now routes through the exact-URL HTTP adapter instead of the
  OpenAI SDK's path-appending behaviour.

### RAG Re-index Resilience

All three LlamaIndex pipeline entry points (`initialize`, `search`, `add`) call
`reconfigure_embedding()` before use, so Settings changes are picked up by
long-lived pipelines without a restart.

Persisted vector stores are validated before retrieval. Invalid indexes return
`{"needs_reindex": true, ...}` with a user-facing explanation instead of a raw
NumPy traceback.

### Memory Thinking-Tag Cleanup

All writes to `PROFILE.md` / `SUMMARY.md` pass through `clean_thinking_tags()`
after code-fence stripping. Existing files with `<think>` / `<thinking>` tags
are cleaned and rewritten on the next read.

### Deep Solve Fix

`MainSolver` no longer passes a stale `attachments` keyword to
`SolverAgent.process()`, eliminating a `TypeError` in the ReAct solver loop.

---

## Plugin Development (quick reference)

Create `deeptutor/plugins/<name>/` with `manifest.yaml` + `capability.py`:

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

See `BACKEND_COMPONENTS.md §5` for all built-in capabilities.
