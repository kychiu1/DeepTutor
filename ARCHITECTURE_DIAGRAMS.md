# DeepTutor — Architecture Diagrams

Visual reference for how components connect and how messages flow through each
major feature of the system.

> Code paths reference the `deeptutor/` package unless otherwise noted.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [WebSocket Runtime — Message Exchange](#2-websocket-runtime--message-exchange)
3. [Chat Capability — Agentic Pipeline](#3-chat-capability--agentic-pipeline)
4. [Deep Solve — Plan → ReAct → Write](#4-deep-solve--plan--react--write)
5. [Deep Question — Idea → Generate → Validate](#5-deep-question--idea--generate--validate)
6. [Deep Research — Decompose → Research → Report](#6-deep-research--decompose--research--report)
7. [Knowledge Base — Document Indexing](#7-knowledge-base--document-indexing)
8. [Knowledge Base — RAG Retrieval](#8-knowledge-base--rag-retrieval)
9. [Embedding System — Endpoint Transparency (v1.3.2)](#9-embedding-system--endpoint-transparency-v132)
10. [Memory System — Update & Cleanup (v1.3.2)](#10-memory-system--update--cleanup-v132)
11. [TutorBot — Multi-Channel Autonomous Agent](#11-tutorbot--multi-channel-autonomous-agent)
12. [Session & State Management](#12-session--state-management)
13. [Co-Writer — Document Editing & AI Assistance](#13-co-writer--document-editing--ai-assistance)
14. [Module Dependency Hierarchy](#14-module-dependency-hierarchy)

---

## 1. System Overview

All entry points funnel into `ChatOrchestrator`, which dispatches to a
two-level plugin system backed by shared services.

```mermaid
flowchart TD
    subgraph Entry["Entry Points"]
        CLI["CLI — Typer\ndeeptutor chat / run / serve"]
        WS["WebSocket\n/api/v1/ws"]
        SDK["Python SDK"]
    end

    subgraph Runtime["runtime/"]
        ORCH["ChatOrchestrator\norchestrator.py"]
        CR["CapabilityRegistry\nLevel 2"]
        TR["ToolRegistry\nLevel 1"]
    end

    subgraph Caps["capabilities/"]
        CHAT["ChatCapability"]
        SOLVE["DeepSolveCapability"]
        QUEST["DeepQuestionCapability"]
        RES["DeepResearchCapability"]
        VIZ["VisualizeCapability"]
        ANIM["MathAnimatorCapability"]
    end

    subgraph Svc["services/"]
        LLM["LLM\nllm/"]
        EMB["Embedding\nembedding/"]
        RAG["RAG\nrag/"]
        MEM["Memory\nmemory/"]
        SESS["Session\nsession/"]
        SRCH["Search\nsearch/"]
    end

    subgraph Store["Persistent Storage"]
        SQLITE[("SQLite\nchat_history.db")]
        FILES[("File Store\nknowledge_bases/\nmemory/ workspace/")]
    end

    CLI & WS & SDK --> ORCH
    ORCH --> CR & TR
    CR --> CHAT & SOLVE & QUEST & RES & VIZ & ANIM

    CHAT & SOLVE & QUEST & RES --> LLM
    CHAT & SOLVE & RES --> RAG
    SOLVE & RES --> SRCH
    RAG --> EMB
    ORCH --> MEM & SESS

    SESS --> SQLITE
    MEM & RAG --> FILES
```

---

## 2. WebSocket Runtime — Message Exchange

How a browser or SDK client creates a turn, receives a streaming response, and
performs lifecycle operations over the single `/api/v1/ws` endpoint.

```mermaid
sequenceDiagram
    participant C as Client
    participant WS as unified_ws.py
    participant TRM as TurnRuntimeManager
    participant CTX as ContextBuilder
    participant ORCH as ChatOrchestrator
    participant CAP as Capability
    participant BUS as StreamBus
    participant DB as SQLiteStore

    C->>WS: WebSocket connect
    C->>WS: {type: "message", content, capability, session_id}
    WS->>TRM: create_turn(session_id, capability)
    TRM->>CTX: build_context(session_id, message)
    CTX->>DB: load session history + compressed_summary
    DB-->>CTX: messages[]
    CTX-->>TRM: UnifiedContext

    TRM->>ORCH: run(context, stream_bus)
    ORCH->>CAP: capability.run(context, stream_bus)

    loop Streaming events
        CAP-->>BUS: emit StreamEvent (stage / content / tool_call / result)
        BUS-->>WS: fan-out
        WS-->>C: JSON event
    end

    CAP->>DB: persist turn + messages
    WS-->>C: {type: "done"}

    Note over C,WS: subscribe_turn — watch an existing turn (replay + live)
    C->>WS: {type: "subscribe_turn", turn_id, after_seq}
    TRM-->>WS: AsyncIterator over stored + live events
    WS-->>C: replayed events from after_seq

    Note over C,WS: cancel_turn — stop in-flight execution
    C->>WS: {type: "cancel_turn", turn_id}
    WS->>TRM: cancel(turn_id)
    TRM-->>CAP: CancelledError propagated

    Note over C,WS: regenerate — re-run last message
    C->>WS: {type: "regenerate", session_id, overrides}
    WS->>TRM: create_turn(session_id, regenerate=True)
```

---

## 3. Chat Capability — Agentic Pipeline

The default capability. An agentic loop drives tool selection and response
generation through four internal stages.

```mermaid
sequenceDiagram
    participant C as Client
    participant CAP as ChatCapability
    participant PIPE as AgenticChatPipeline
    participant SM as SessionManager
    participant LLM as LLM Service
    participant TR as ToolRegistry
    participant RAG as RAG Service
    participant WEB as Search Service
    participant CODE as CodeExecutor

    C->>CAP: {capability:"chat", content, session_id, kb_id}
    CAP->>PIPE: run(context, stream)

    PIPE->>SM: compress_if_needed(history)
    SM->>LLM: summarise long history
    LLM-->>SM: compressed_summary

    Note over PIPE,LLM: Stage — thinking
    PIPE->>LLM: system prompt + history + user message
    LLM-->>PIPE: thought + optional tool_calls[]

    loop Tool execution (acting → observing)
        PIPE-->>C: stream {type:"tool_call", name, args}
        alt tool == "rag"
            PIPE->>RAG: search(query, kb_id)
            RAG-->>PIPE: context chunks
        else tool == "web_search"
            PIPE->>WEB: search(query)
            WEB-->>PIPE: results[]
        else tool == "code_execution"
            PIPE->>CODE: execute(code)
            CODE-->>PIPE: stdout / result
        else tool == "reason" / "brainstorm" / "paper_search"
            PIPE->>LLM: dedicated reasoning call
            LLM-->>PIPE: output
        end
        PIPE-->>C: stream {type:"tool_result", name, output}
        PIPE->>LLM: inject observation → next thought
        LLM-->>PIPE: thought + optional further tool_calls[]
    end

    Note over PIPE,LLM: Stage — responding
    PIPE->>LLM: final answer prompt
    LLM-->>PIPE: stream tokens
    PIPE-->>C: stream {type:"content", delta}
    PIPE-->>C: stream {type:"result"}
```

---

## 4. Deep Solve — Plan → ReAct → Write

Three-phase pipeline for structured problem solving. `MainSolver` creates and
orchestrates the three agents; each phase has its own LLM call sequence.

```mermaid
sequenceDiagram
    participant C as Client
    participant CAP as DeepSolveCapability
    participant MS as MainSolver
    participant PL as PlannerAgent
    participant SL as SolverAgent
    participant STR as SolveToolRuntime
    participant WR as WriterAgent
    participant LLM as LLM Service
    participant MEM as Memory Service
    participant EVT as EventBus

    C->>CAP: {capability:"deep_solve", content, kb_id, tools}
    CAP->>MS: ainit() — create agents
    MS->>MEM: load planner memory context
    MEM-->>MS: user profile + preferences

    Note over MS,PL: Phase 1 — PLAN
    MS->>PL: plan(question, attachments)
    PL->>LLM: "Analyse and produce step-by-step plan"
    LLM-->>PL: Plan{steps[], goal}
    PL-->>MS: Plan
    MS-->>C: stream {stage:"planning", plan}

    Note over MS,SL: Phase 2 — SOLVE (ReAct per step)
    loop For each plan step
        MS->>MEM: load solver memory context
        MEM-->>MS: relevant prior solutions
        MS->>SL: solve_step(step, context)

        loop ReAct iterations (max_iterations)
            SL->>LLM: Thought: … Action: …
            LLM-->>SL: tool_call {name, args}
            SL->>STR: execute(tool_call)
            STR-->>SL: Observation
            SL-->>C: stream {stage:"reasoning", thought, observation}
            SL->>LLM: Observation → next Thought
            LLM-->>SL: next action or "Final Answer"
        end

        opt replan needed
            SL->>MS: replan_needed
            MS->>PL: replan(original + observations)
            PL->>LLM: revised plan
            LLM-->>PL: updated Plan
        end
    end
    SL-->>MS: consolidated solution
    MS-->>C: stream {stage:"reasoning", done}

    Note over MS,WR: Phase 3 — WRITE
    MS->>WR: write(plan, solution, detailed_mode)
    WR->>LLM: "Format final answer (LaTeX / markdown)"
    LLM-->>WR: stream formatted answer
    WR-->>MS: final_answer
    MS-->>C: stream {stage:"writing", content}
    MS-->>C: stream {type:"result"}

    MS->>EVT: publish(SOLVE_COMPLETE, {session_id, tokens})
```

---

## 5. Deep Question — Idea → Generate → Validate

Generates an exam question bank through a three-stage ideation and evaluation
pipeline.

```mermaid
sequenceDiagram
    participant C as Client
    participant CAP as DeepQuestionCapability
    participant COORD as AgentCoordinator
    participant IDEA as IdeaAgent
    participant GEN as QuestionGenerator
    participant FOLLOW as FollowupAgent
    participant LLM as LLM Service
    participant RAG as RAG Service

    C->>CAP: {capability:"deep_question", topic, num_questions, kb_id}
    CAP->>COORD: run(topic, config)

    Note over COORD,IDEA: Stage 1 — Ideation
    COORD->>IDEA: generate_ideas(topic)
    opt KB provided
        IDEA->>RAG: search(topic)
        RAG-->>IDEA: relevant passages
    end
    IDEA->>LLM: "Brainstorm angles, subtopics, difficulty levels"
    LLM-->>IDEA: ideas[]
    IDEA-->>COORD: ideas[]
    COORD-->>C: stream {stage:"ideation"}

    Note over COORD,GEN: Stage 2 — Generation
    COORD->>GEN: generate_questions(ideas, num_questions)
    GEN->>LLM: "Draft questions with model answers and distractors"
    LLM-->>GEN: question_drafts[]
    GEN-->>COORD: questions[]
    COORD-->>C: stream {stage:"generation", questions[]}

    Note over COORD,FOLLOW: Stage 3 — Evaluation & Followup
    COORD->>FOLLOW: evaluate(questions[])
    FOLLOW->>LLM: "Quality check — clarity, difficulty, correctness"
    LLM-->>FOLLOW: validated_questions[] + corrections
    FOLLOW->>LLM: "Generate follow-up questions for each"
    LLM-->>FOLLOW: followups[]
    FOLLOW-->>COORD: final_question_set

    COORD-->>C: stream {type:"result", questions: final_question_set}
```

---

## 6. Deep Research — Decompose → Research → Report

Multi-agent research pipeline that decomposes a topic into sub-questions,
researches each in turn, and synthesises a cited report.

```mermaid
sequenceDiagram
    participant C as Client
    participant CAP as DeepResearchCapability
    participant RP as ResearchPipeline
    participant DEC as DecomposeAgent
    participant RA as ResearchAgent
    participant NA as NoteAgent
    participant CM as CitationManager
    participant REP as ReportingAgent
    participant LLM as LLM Service
    participant WEB as Search Service
    participant RAG as RAG Service

    C->>CAP: {capability:"deep_research", topic, depth, kb_id}
    CAP->>RP: run(topic)

    Note over RP,DEC: Stage 1 — Decomposition
    RP->>DEC: decompose(topic)
    DEC->>LLM: "Break into focused sub-questions"
    LLM-->>DEC: sub_questions[]
    DEC-->>RP: research_agenda
    RP-->>C: stream {stage:"decomposing"}

    Note over RP,NA: Stage 2 — Research (per sub-question)
    loop For each sub-question
        RP->>RA: research(sub_question)
        RA->>WEB: web_search(sub_question)
        WEB-->>RA: web_results[]
        opt KB provided
            RA->>RAG: rag_search(sub_question)
            RAG-->>RA: passages[]
        end
        RA->>LLM: "Synthesise findings from sources"
        LLM-->>RA: findings
        RA->>CM: register_sources(urls, passages)
        CM-->>RA: citation_keys
        RA->>NA: take_note(findings, citation_keys)
        NA->>LLM: "Compress into structured note"
        LLM-->>NA: note
        NA-->>RP: note
        RP-->>C: stream {stage:"researching", sub_question}
    end

    Note over RP,REP: Stage 3 — Reporting
    RP->>REP: generate_report(notes[], citations)
    REP->>LLM: "Write full research report with sections"
    LLM-->>REP: draft_report
    REP->>CM: inject_citations(draft_report)
    CM-->>REP: cited_report
    REP-->>RP: final_report
    RP-->>C: stream {stage:"writing", content}
    RP-->>C: stream {type:"result", report}
```

---

## 7. Knowledge Base — Document Indexing

How uploaded documents become a searchable vector index, including the
embedding endpoint normalisation introduced in v1.3.2.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as knowledge.py router
    participant KBM as KnowledgeBaseManager
    participant INIT as KnowledgeBaseInitializer
    participant PB as ProgressBroadcaster
    participant PIPE as LlamaIndexPipeline
    participant EA as EmbeddingAdapter (CustomEmbedding)
    participant EP as embedding_endpoint.py
    participant EC as EmbeddingClient
    participant ADP as Provider Adapter
    participant STOR as VectorStore (disk)
    participant IDX as IndexVersioning

    C->>API: POST /knowledge/{kb}/documents (multipart)
    API->>KBM: create_and_index(kb_name, files)
    KBM->>INIT: initialize(kb_name, docs)
    KBM-->>C: WS progress stream (via ProgressBroadcaster)

    INIT->>PIPE: initialize()
    PIPE->>EP: normalize_embedding_endpoint(provider, base_url)
    EP-->>PIPE: full endpoint URL
    PIPE->>EC: create client (validated endpoint)
    EC->>EC: reject root/base URLs for known providers

    INIT->>PIPE: load_and_index(docs)
    loop For each document
        PIPE->>PIPE: route by file type
        PIPE->>PIPE: load → parse → chunk
        PIPE->>EA: embed_batch(chunks)
        EA->>EP: fingerprint current config
        EA->>EC: encode(texts)
        EC->>ADP: POST endpoint (exact URL)
        ADP-->>EC: vectors[]
        EC-->>EA: vectors[]
        EA-->>PIPE: embedded nodes[]
        PB-->>C: {progress: n/total}
    end

    PIPE->>IDX: compute embedding_signature()
    IDX-->>PIPE: signature
    PIPE->>STOR: persist(index, signature)
    STOR-->>PIPE: saved
    INIT-->>KBM: done
    KBM-->>C: {status:"complete"}
```

---

## 8. Knowledge Base — RAG Retrieval

How a user query travels from the RAG tool through vector search back to the
LLM prompt. Includes the vector validation added in v1.3.2.

```mermaid
flowchart TD
    subgraph In["Input"]
        Q["User query\n(from rag tool / RAGService)"]
    end

    subgraph Pipe["LlamaIndexPipeline — search()"]
        RECONF["reconfigure_embedding()\nrefresh embedding client\nif Settings changed"]
        LOAD["Load persisted index\nfrom disk"]
        VALID{"Validate stored\nvectors"}
        REINDEX["Return\n{needs_reindex: true}\nwith explanation"]
        EMBED["Embed query\nvia EmbeddingAdapter"]
        SIM["Similarity search\nVector cosine / dot"]
        SMART["SmartRetriever\nadvanced strategies"]
        TOPK["Top-k chunks\n(with metadata)"]
    end

    subgraph EmbSvc["Embedding Service"]
        EP["embedding_endpoint.py\nresolve full URL"]
        EC["EmbeddingClient"]
        ADP["Provider Adapter\n(exact URL POST)"]
    end

    subgraph Out["Output"]
        CTX["Context injected\ninto LLM prompt"]
        ERR["Error event\n→ needs_reindex hint\n→ client"]
    end

    Q --> RECONF --> LOAD --> VALID
    VALID -->|"null / shape / non-finite\nvectors detected"| REINDEX --> ERR
    VALID -->|valid| EMBED
    EMBED --> EP --> EC --> ADP
    ADP -->|"query vector"| SIM
    SIM --> SMART --> TOPK --> CTX
```

---

## 9. Embedding System — Endpoint Transparency (v1.3.2)

End-to-end flow showing how an embedding URL set in the UI becomes the exact
URL POSTed to, with no hidden path appending.

```mermaid
flowchart LR
    subgraph UI["Web Settings (Next.js)"]
        FORM["Endpoint URL field\nlabelled 'Endpoint URL'\nnot 'Base URL'"]
    end

    subgraph API["settings.py router\nPUT /api/v1/settings"]
        SAPI["Persist to\nmodel_catalog"]
    end

    subgraph Config["Configuration Layer"]
        CAT["model_catalog.py\nnormalise on load\npersist back if changed"]
        EP["embedding_endpoint.py\nnormalize_embedding_endpoint_for_display()\nlegacy /v1 → /v1/embeddings\nOllama → /api/embed"]
        PR["provider_runtime.py\nresolve active runtime config"]
    end

    subgraph Client["EmbeddingClient — client.py"]
        VAL["Endpoint Validator\nreject root/base paths\nfor known providers"]
        DISP["Adapter Dispatcher\n_resolve_adapter_class()"]
    end

    subgraph Adapters["Provider Adapters"]
        OAI["openai_sdk.py\nOpenAI native"]
        OAIC["openai_compatible.py\nexact-URL HTTP\nOpenRouter + custom gateways"]
        JIN["jina.py"]
        OLL["ollama.py"]
        COH["cohere.py"]
        SFW["siliconflow.py"]
        OTHER["aliyun / qwen /\ndashscope …"]
    end

    FORM -->|save| SAPI --> CAT
    CAT <-->|migrate| EP
    CAT --> PR --> VAL
    VAL -->|"valid endpoint"| DISP
    VAL -->|"invalid → error\nbefore indexing starts"| UI
    DISP --> OAI & OAIC & JIN & OLL & COH & SFW & OTHER
```

---

## 10. Memory System — Update & Cleanup (v1.3.2)

How user memory files are written after each turn, with thinking-tag stripping
applied to prevent `<think>` blocks from reasoning models being persisted.

```mermaid
sequenceDiagram
    participant ORCH as ChatOrchestrator
    participant MEM as MemoryService
    participant LLM as LLM Service
    participant CLN as clean_thinking_tags()
    participant FS as File System (PROFILE.md / SUMMARY.md)

    Note over ORCH,FS: After each capability turn completes
    ORCH->>MEM: consolidate_memory(session_history)

    MEM->>LLM: "Update user learning summary"
    LLM-->>MEM: raw_summary (may contain <think>…</think>)
    MEM->>MEM: _strip_code_fence(raw_summary)
    MEM->>CLN: clean_thinking_tags(text)
    CLN-->>MEM: clean_summary
    MEM->>FS: write SUMMARY.md

    MEM->>LLM: "Update user profile (knowledge, preferences)"
    LLM-->>MEM: raw_profile (may contain <thinking>…</thinking>)
    MEM->>MEM: _strip_code_fence(raw_profile)
    MEM->>CLN: clean_thinking_tags(text)
    CLN-->>MEM: clean_profile
    MEM->>FS: write PROFILE.md

    Note over MEM,FS: Self-repair on read (every access)
    MEM->>FS: read SUMMARY.md or PROFILE.md
    FS-->>MEM: raw_content
    MEM->>CLN: clean_thinking_tags(raw_content)
    alt content changed (tags found)
        CLN-->>MEM: repaired_content
        MEM->>FS: rewrite file with repaired content
    end
    CLN-->>MEM: clean_content → caller
```

---

## 11. TutorBot — Multi-Channel Autonomous Agent

TutorBot is a parallel autonomous agent system, separate from the main
capability pipeline, with its own agent loop and channel receivers.

```mermaid
flowchart TD
    subgraph CH["Inbound Channels (tutorbot/channels/)"]
        SLK["Slack"]
        TG["Telegram"]
        DC["Discord"]
        WX["WeChat"]
        DT["DingTalk / Feishu"]
        MAT["Matrix / Element"]
        EMAIL["Email (SMTP)"]
    end

    subgraph BUS["MessageBus (tutorbot/bus/)"]
        INBUS["Inbound queue"]
        OUTBUS["Outbound queue"]
    end

    subgraph Loop["AgentLoop — agent/loop.py"]
        CTX_B["ContextBuilder\nassemble history + memory + skills"]
        LLM_P["LLM Provider\n(OpenAI-compat / Anthropic)"]
        TOOL_EXEC["Tool Executor"]
        MEM_CONS["MemoryConsolidator\nupdate SOUL.md"]
    end

    subgraph Tools["TutorBot Tools"]
        SHELL["ShellTool\nrun shell commands"]
        WEB["WebTool\nsearch + browse"]
        MCP["MCPTool\nMCP server calls"]
        FS["FilesystemTool\nread / write files"]
        SPAWN["SpawnTool\nlaunch sub-agent"]
        CRON_T["CronTool\nschedule tasks"]
    end

    subgraph Team["Multi-Agent Team (agent/team/)"]
        TM["TeamManager"]
        BOARD["Board\nshared task state"]
        MBOX["Mailbox\ninter-agent messages"]
        SUBA["Sub-Agent loop"]
    end

    subgraph Workspace["Bot Workspace (data/tutorbot/{bot_id}/)"]
        SOUL_F["SOUL.md\nbot identity"]
        TOOLS_F["TOOLS.md\navailable tools"]
        USER_F["USER.md\nuser profile"]
        SESS_DB[("session.db\nper-bot SQLite")]
    end

    subgraph Sched["Background Services"]
        CRONS["CronService\nscheduled tasks"]
        HB["HeartbeatService\nperiodic health pings"]
    end

    CH --> INBUS --> CTX_B
    CTX_B --> Workspace
    CTX_B --> LLM_P
    LLM_P -->|"tool_calls"| TOOL_EXEC
    TOOL_EXEC --> SHELL & WEB & MCP & FS & SPAWN & CRON_T
    SPAWN --> TM --> BOARD & MBOX --> SUBA
    SUBA -.->|"sub-agent result"| MBOX
    TOOL_EXEC -->|observation| LLM_P
    LLM_P -->|final response| MEM_CONS
    MEM_CONS --> SOUL_F
    LLM_P --> OUTBUS --> CH
    CRONS & HB --> INBUS
```

---

## 12. Session & State Management

How conversation state is stored and loaded, including history compression for
long sessions.

```mermaid
flowchart TD
    subgraph APIIn["API Layer"]
        UWS["unified_ws.py\nmessage / resume_from / regenerate"]
        SESS_R["sessions.py router\nGET / DELETE sessions"]
    end

    subgraph SessionSvc["Session Services"]
        USM["UnifiedSessionManager\nservices/session/unified_session_manager.py"]
        TRM["TurnRuntimeManager\nservices/session/turn_runtime.py"]
        CTX_B["ContextBuilder\nservices/session/context_builder.py"]
        STORE["SQLiteStore\nservices/session/sqlite_store.py\n(aiosqlite async)"]
    end

    subgraph Schema["chat_history.db"]
        STBL["sessions\nid · title · preferences\ncompressed_summary\nsummary_up_to_msg_id"]
        MTBL["messages\nid · session_id · role\ncontent · events_json\nattachments_json"]
        TTBL["turns\nid · session_id\ncapability · status · error"]
    end

    subgraph Compress["History Compression (long sessions)"]
        CHK{"token count\n> threshold?"}
        COMP_LLM["LLM: summarise\nolder messages"]
        SAVE_SUM["persist compressed_summary\nto sessions table"]
        TRIM["trim messages from\ncontext window"]
    end

    UWS -->|"new turn"| TRM
    TRM --> USM --> STORE
    STORE --- STBL & MTBL & TTBL

    TRM --> CTX_B
    CTX_B --> STORE
    CTX_B -->|UnifiedContext| UWS

    MTBL --> CHK
    CHK -->|yes| COMP_LLM --> SAVE_SUM --> STBL
    SAVE_SUM --> TRIM
    CHK -->|no| CTX_B

    SESS_R --> STORE
```

---

## 13. Co-Writer — Document Editing & AI Assistance

The Co-Writer module is a standalone writing workspace. It has two independent
subsystems: document storage (CRUD) and AI-assisted editing (two modes).

### 13a. Document Management

```mermaid
flowchart LR
    subgraph Client["Client (Web UI)"]
        REQ["CRUD request"]
    end

    subgraph Router["api/routers/co_writer.py"]
        LIST["GET  /co_writer/documents"]
        CREATE["POST /co_writer/documents"]
        GET["GET  /co_writer/documents/{id}"]
        UPDATE["PUT  /co_writer/documents/{id}"]
        DELETE["DELETE /co_writer/documents/{id}"]
    end

    subgraph Store["CoWriterStorage — co_writer/storage.py"]
        STOR["get_co_writer_storage()\nsingleton"]
        TITLE["_derive_title()\nauto-title from first heading"]
        PREV["_build_preview()\n160-char excerpt"]
        ATOMIC["_atomic_write_json()\nwrite-temp + fsync + rename"]
    end

    subgraph FS["File System"]
        DIR["data/user/workspace/co-writer/\ndocuments/doc_{id}/manifest.json"]
    end

    REQ --> LIST & CREATE & GET & UPDATE & DELETE
    LIST & CREATE & GET & UPDATE & DELETE --> STOR
    STOR --> TITLE & PREV & ATOMIC
    ATOMIC --> FS
```

### 13b. Simple Edit (EditAgent)

Fetches optional context (RAG or web search) then makes a single LLM call.

```mermaid
sequenceDiagram
    participant C as Client
    participant R as co_writer.py router
    participant EA as EditAgent
    participant RAG as rag_search()
    participant WEB as web_search()
    participant LLM as LLM Service
    participant CLN as clean_thinking_tags()
    participant FS as history.json

    C->>R: POST /co_writer/edit\n{text, instruction, action, source, kb_name}
    R->>EA: agent.process(text, instruction, action, source, kb_name)

    alt source == "rag"
        EA->>RAG: rag_search(instruction, kb_name)
        RAG-->>EA: context chunks
        EA->>EA: save_tool_call(op_id, "rag", data)
    else source == "web"
        EA->>WEB: web_search(instruction)
        WEB-->>EA: context + citations
        EA->>EA: save_tool_call(op_id, "web", data)
    end

    EA->>EA: build system_prompt + user_prompt\n(action verb + optional context + text)
    EA->>LLM: stream_llm(system_prompt, user_prompt, stage="edit_{action}")
    LLM-->>EA: raw chunks[]
    EA->>CLN: clean_thinking_tags(joined_chunks)
    CLN-->>EA: edited_text

    EA->>FS: append operation_record to history.json
    EA-->>R: {edited_text, operation_id}
    R-->>C: EditResponse
```

### 13c. ReAct Edit (react_edit) — AI-assisted with tool use

Uses the full agentic pipeline with optional RAG, web search, brainstorm,
paper search, and code execution tools. Supports both blocking and streaming
response modes.

```mermaid
sequenceDiagram
    participant C as Client
    participant R as co_writer.py router
    participant EA as EditAgent (BaseAgent)
    participant BUS as StreamBus
    participant LLM as LLM Service (agentic)
    participant TR as ToolRegistry
    participant RAG as rag_search
    participant WEB as web_search
    participant CLN as clean_thinking_tags()
    participant FS as history.json

    C->>R: POST /co_writer/edit_react\n{selected_text, instruction, mode, tools[], kb_names[]}
    Note over R: _normalize_react_edit_tools()\nallowed: rag, web_search, brainstorm,\nreason, paper_search, code_execution
    R->>EA: _run_react_edit(request, language, stream=bus)

    EA->>EA: build edit prompt\n(mode + instruction + selected_text)
    EA->>EA: refresh_config() — pick up latest Settings

    Note over EA,LLM: Agentic tool loop (stream_llm with tools)
    loop Until final edit or max iterations
        EA->>LLM: prompt + available tool schemas
        LLM-->>EA: thought + tool_call OR final edit
        alt tool called
            EA->>TR: dispatch(tool_name, args)
            TR->>RAG: rag_search() or
            TR->>WEB: web_search() etc.
            TR-->>EA: tool observation
            EA-->>BUS: emit tool_call + result events
            EA->>LLM: inject observation
        end
    end

    LLM-->>EA: final edited text (streamed tokens)
    EA-->>BUS: emit content events
    EA->>CLN: clean_thinking_tags(raw_output)
    CLN-->>EA: edited_text
    EA-->>BUS: emit result

    EA->>FS: append operation_record to history.json
    BUS-->>C: streamed events OR blocking response

    Note over C,R: /edit_react/stream — SSE streaming variant\nuses same _run_react_edit() via AsyncGenerator
```

---

## 14. Module Dependency Hierarchy

The six strict dependency layers guarantee that inner layers never import outer
ones. TutorBot is a parallel system with its own stack.

```mermaid
flowchart BT
    subgraph L0["Layer 0 — Core (no internal imports)"]
        CORE["core/\nstream · context · protocols\nstream_bus · trace"]
    end

    subgraph L1["Layer 1 — Services"]
        SVC["services/\nllm · embedding · rag · memory\nsession · search · storage · skill\nconfig · path_service"]
    end

    subgraph L2["Layer 2 — Tools"]
        TOOLS["tools/\nrag_tool · web_search · code_executor\nreason · brainstorm · paper_search\nvision · question"]
    end

    subgraph L3["Layer 3 — Agents"]
        AGENTS["agents/\nchat · solve · question · research\nvisualize · notebook · vision_solver\nmath_animator"]
    end

    subgraph L4["Layer 4 — Capabilities"]
        CAPS["capabilities/\nchat · deep_solve · deep_question\ndeep_research · visualize · math_animator"]
    end

    subgraph L5["Layer 5 — Runtime"]
        RT["runtime/\norchestrator · registries · bootstrap"]
    end

    subgraph L6["Layer 6 — API Routers (outermost)"]
        API["api/\nrouters · WebSocket · REST endpoints"]
    end

    subgraph TB["Parallel — TutorBot"]
        TBOT["tutorbot/\nown agent loop · channel receivers\ntools · providers · session"]
    end

    CORE --> SVC
    SVC --> TOOLS
    TOOLS --> AGENTS
    AGENTS --> CAPS
    CAPS --> RT
    RT --> API

    CORE --> TBOT
    SVC --> TBOT
```
