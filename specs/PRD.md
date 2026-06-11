# PRD — Quaero

> **Quaero** (Latin: "I seek / I ask") — ask your PDFs, with verifiable citations.
>
> Self-contained desktop application. Runs 100% locally, ~zero cost, no deploy.

## 1. Overview

**Product:** Quaero

**Goal:** Upload PDFs and conversational chat over their content via an LLM, using
RAG (Retrieval-Augmented Generation) for precise, traceable answers grounded in the
document. Delivered as a cross-platform desktop application (macOS, Windows, Linux).

---

## 2. Problem

Finding information in long documents (contracts, reports, tenders, manuals,
procedures) is slow and error-prone. Traditional chat-with-PDF fails due to poor
context retrieval. RAG + verifiable citations solves this.

---

## 3. Goals

### Business

- Reduce document analysis time.
- Increase answer accuracy.
- Traceability (page/snippet citation).
- Demonstrate Generative AI with RAG.

### Technical

- Complete end-to-end RAG pipeline running locally in a desktop app.
- Verifiable citations (document + page + snippet).
- Multiple documents per conversation.
- Fully local, ~zero cost, no mandatory paid dependency.
- Cross-platform desktop distribution (macOS, Windows, Linux).

---

## 3.5 Users

### Primary Persona

**Alex — Legal/Compliance Analyst**
- Reviews 30–100 page contracts, tenders, and regulatory documents.
- Needs to quickly find specific clauses (termination, liability, confidentiality).
- Currently skims manually or uses Ctrl+F, missing cross-referenced terms.
- Values accuracy and traceability — must cite the exact page.

### Secondary Persona

**Pat — Graduate Researcher**
- Works with academic papers and technical reports (50–200 pages).
- Asks comparative questions across multiple documents.
- Needs verifiable citations for literature review notes.

### User Stories

| # | Story | Acceptance Criteria |
|---|---|---|
| US-1 | As a user, I want to upload one or more PDFs so that I can ask questions about their content. | Drag-drop or file dialog accepts PDFs ≤ 200 MB; documents appear in list with processing status; status transitions to ready/failed within processing timeout. |
| US-2 | As a user, I want to ask natural-language questions and receive answers grounded in my documents. | Answer is generated only from retrieved chunks; if no relevant context, system refuses with clear message; response time ≤ 5 s (excluding LLM latency). |
| US-3 | As a user, I want every answer to cite the source document, page number, and relevant snippet. | Each answer includes ≥ 1 citation; citation data (doc, page, snippet) is persisted in SQLite and survives app restart. |
| US-4 | As a user, I want to manage multiple conversations, each associated with specific documents. | I can create a conversation, attach N documents, and continue the conversation across sessions. |
| US-5 | As a user, I want to configure the LLM provider (remote API vs local Ollama) without editing config files. | Settings UI exposes provider selection, API key/URL fields; changes persist across restarts. |

---

## 4. Principles (locked decisions)

| Topic | Decision | Reason |
|---|---|---|
| Platform | **Tauri 2.x** (Rust backend + system WebView) | Small binary, native performance, cross-platform |
| Frontend | **Vue 3 + TypeScript + Shadcn-vue + Tailwind CSS** | Modern reactive UI, headless component primitives |
| Vector store | **SQLite + sqlite-vec** (embedded) | No external database; single-file, zero config |
| Embeddings | **Ollama + `bge-m3`** (local, 1024 dims) | Free, CPU, multilingual PT-BR |
| Chat LLM | **Configurable `LLMProvider`**: GLM-5.1 (remote API) or Ollama (local) | Flexibility without over-engineering |
| PDF parsing | **`pdf-extract`** (pure Rust, per-page) | No system deps, native Tauri integration |
| Processing | **Async** (background task, non-blocking UI) | Desktop UX demands responsive UI during ingestion |
| Auth | **No auth, single-user** | Focus on the RAG core |
| Distribution | **Tauri bundler** (`.dmg` / `.msi` / `.AppImage`) | Built-in cross-platform packaging |
| Citations | **Persisted** per message | Traceability survives restart |
| State management | **Pinia** | Vue 3 standard, TypeScript-first |

---

## 5. Scope

### PDF Upload

- Native file dialog or drag-and-drop for one or more PDFs.
- Upload cap: 200 MB (configurable via settings).
- Processed asynchronously: up to 1000 pages / 200 MB per file. Above that → clear error.

### Processing (async, on upload)

1. Text extraction **per page** (`pdf-extract` — required for citation).
2. Chunking (see **Chunking Strategy** below).
3. Embedding of each chunk via Ollama (`bge-m3`, 1024 dims).
4. Storage in `document_chunks.embedding` (sqlite-vec, `float32(1024)`).
5. Returns `document ready`.

### Chunking Strategy (locked)

- **Method**: Character-level sliding window with sentence-boundary awareness.
- **Target chunk size**: 512 characters (≈ 100–150 tokens for English/Portuguese).
- **Overlap**: 64 characters (~12%) between consecutive chunks.
- **Boundary preference**: If a chunk boundary falls mid-sentence, extend to the
  next sentence-ending punctuation (`.`, `!`, `?`, line break). Max extension: +128 chars.
- **Minimum chunk**: If a page yields < 64 characters after extraction, merge with
  the next page's text before chunking (preserving page metadata as the first page).
- **Page metadata**: Every chunk records its source page number. If merged across pages,
  record the starting page.
- **Tokenization note**: We use character count (not BPE tokens) for simplicity.
  If evaluation shows poor retrieval quality, this can be tuned to token-count in a
  future iteration (out of current scope).

### Document Chat (RAG)

Natural-language questions ("What is the termination notice period?", "Is there a
confidentiality clause?", "Summarize this document").

Flow:

1. Embed the question (`bge-m3` via Ollama).
2. Vector search (cosine distance) over the chunks **of the conversation's documents**.
3. Top-K chunks become context.
4. Prompt builder assembles the prompt (instruction: answer **only** from the retrieved context).
5. `LLMProvider` generates the answer (GLM-5.1 or Ollama — user-configured).
6. Answer + citations persisted to SQLite.

### Source Citation (persisted)

Every answer carries: document name, page, snippet used. Saved in `message_citations`
→ survives app restart.

```text
Termination notice period: 30 days.

Source:
Contract.pdf — Page 18
"The notice period for termination shall be 30 days..."
```

### Conversation History

List previous messages; continue an existing conversation. A conversation references N
docs.

### Frontend (Vue 3)

#### Documents Screen

- Drag-and-drop area for uploading PDFs + native file dialog fallback.
- Async processing: documents appear as "Processing" immediately, auto-refresh until
  resolved.
- Document list table with status badges (processing / ready / failed).
- Rename and delete documents (cascade: chunks, conversation links, citations).

#### Chat Screen

- Conversational interface (user/assistant messages).
- Answers rendered in Markdown with embedded citations (doc + page + snippet),
  expandable.
- Sidebar showing documents associated with the conversation.
- Question input + send. Suggested prompts on empty state.

#### Conversations Screen

- List of existing conversations with title.
- Create a new conversation + associate documents.
- Continue a previous conversation (history loaded).

#### Theme

- Light / dark / system theme toggle. Persisted to local storage.
- FOUC-free boot (theme applied before Vue hydration).

---

## 6. Future Features

- **MCP server** (expose tools for external clients like Claude Desktop).
- JWT multi-user authentication + isolation.
- OCR for scanned PDFs.
- PDF comparison; table extraction.
- Export (Markdown/PDF/DOCX); structured automatic summarization.
- In-process embeddings (ONNX — no external Ollama dependency).
- Configurable / multi-model embedding (different dimensions).
- Auto-update via Tauri's updater plugin.

---

## 7. Architecture

```text
┌─────────────────────────────────────────────────────┐
│  Vue 3 + Shadcn-vue + Tailwind CSS + Pinia          │
│  (rendered in system WebView via Tauri)              │
└───────────────────────┬─────────────────────────────┘
                        │ Tauri IPC (invoke / events)
┌───────────────────────▼─────────────────────────────┐
│  Rust Backend (Tauri Commands)                       │
│  ├─ Document Service                                 │
│  │   ├─ PDF Parser (pdf-extract, per page)           │
│  │   ├─ Chunker (fixed + overlap)                    │
│  │   └─ Embedding Client ───► Ollama (bge-m3) [HTTP] │
│  ├─ Chat Service (RAG)                               │
│  │   ├─ Embedding Client ───► Ollama (bge-m3)        │
│  │   ├─ Retriever ──────────► sqlite-vec (cosine)     │
│  │   ├─ Prompt Builder                                │
│  │   └─ LLMProvider ────────► GLM-5.1 (remote)       │
│  │                          └─► Ollama (local)        │
│  ├─ Vector Store (sqlite-vec via rusqlite)            │
│  └─ SQLite (rusqlite, single-file DB)                │
└──────────────────────────────────────────────────────┘
```

External dependency (user runs locally):

- **Ollama** (`bge-m3`) — embeddings. Optionally also for chat LLM.
- **GLM-5.1** — remote API (OpenAI-compatible) — the only potential cost point.

---

## 8. Tech Stack

| Layer | Technology |
|---|---|
| Desktop shell | **Tauri 2.x** (Rust) |
| Frontend | **Vue 3** + **TypeScript** + **Vite** |
| UI components | **Shadcn-vue** + **Tailwind CSS** |
| State management | **Pinia** |
| Routing | **Vue Router** |
| PDF parsing | **`pdf-extract`** (Rust, lopdf-based) |
| Embeddings | **Ollama + `bge-m3`** (HTTP, 1024 dims) |
| Vector store | **SQLite + sqlite-vec** (embedded via `rusqlite`) |
| LLM | **`LLMProvider` trait**: GLM-5.1 (reqwest HTTP) or Ollama |
| Validation | **Zod** (frontend response schemas) |
| Build/bundle | **Tauri CLI** (`cargo tauri build`) |

---

## 9. Data Model

SQLite schema. All IDs are text (UUID v4). Timestamps stored as ISO 8601 strings.

```sql
documents (
  id            text primary key,
  filename      text not null,
  page_count    integer,
  size_bytes    integer,
  status        text not null,    -- 'processing' | 'ready' | 'failed'
  error_code    text,             -- null unless failed
  error_message text,             -- null unless failed
  created_at    text not null
)

document_chunks (
  id            text primary key,
  document_id   text not null references documents(id) on delete cascade,
  page          integer not null,
  chunk_index   integer not null,
  content       text not null,
  embedding     blob              -- sqlite-vec float32(1024)
)
-- sqlite-vec virtual table for vector search:
-- create virtual table vec_chunks using vec0(
--   embedding float[1024] distance_metric=cosine
-- )

conversations (
  id            text primary key,
  title         text not null,
  created_at    text not null
)

conversation_documents (
  conversation_id text not null references conversations(id) on delete cascade,
  document_id     text not null references documents(id) on delete cascade,
  primary key (conversation_id, document_id)
)

messages (
  id              text primary key,
  conversation_id text not null references conversations(id) on delete cascade,
  role            text not null,    -- 'user' | 'assistant'
  content         text not null,
  created_at      text not null
)

message_citations (
  id          text primary key,
  message_id  text not null references messages(id) on delete cascade,
  document_id text not null references documents(id) on delete cascade,
  chunk_id    text not null references document_chunks(id) on delete cascade,
  page        integer not null,
  snippet     text not null
)
```

> Foreign keys enabled via `PRAGMA foreign_keys = ON;` on every connection.
> Cascade deletes handle document removal cleanly.

### Migrations

- Schema version tracked in a `_meta` table: `(key TEXT PRIMARY KEY, value TEXT)`.
- On app start, compare current schema version with code version.
- Apply sequential migrations in a transaction.
- No rollback — forward-only migrations (SQLite limitation).
- Migrations are inline Rust functions in `db/migrations.rs`, not SQL files.

---

## 10. Tauri IPC Contracts

All backend operations exposed as **Tauri commands** (invoked from Vue via `invoke()`).

| Command | Direction | Description |
|---|---|---|
| `upload_documents` | Vue → Rust | Accept file paths from native dialog/drop; validate; insert as `processing`; spawn background processing |
| `list_documents` | Vue → Rust | Return all documents with status |
| `get_document` | Vue → Rust | Single document detail |
| `rename_document` | Vue → Rust | Update display filename |
| `delete_document` | Vue → Rust | Cascade delete (chunks, links, citations) in transaction |
| `list_conversations` | Vue → Rust | Return all conversations |
| `get_conversation` | Vue → Rust | Conversation detail with documents, messages, citations |
| `create_conversation` | Vue → Rust | Create conversation, optionally associate documents |
| `associate_documents` | Vue → Rust | Add documents to a conversation |
| `send_message` | Vue → Rust | RAG pipeline: embed → retrieve → prompt → LLM → persist → return with citations |
| `get_config` | Vue → Rust | Read current settings (LLM provider, Ollama URL, etc.) |
| `update_config` | Vue → Rust | Persist settings |
| `check_health` | Vue → Rust | Verify Ollama connectivity + DB status |

### Async events (Rust → Vue)

| Event | Payload | When |
|---|---|---|
| `document:processing` | `{ documentId, status }` | Document status changes during processing |
| `document:ready` | `{ documentId }` | Processing complete |
| `document:failed` | `{ documentId, errorCode, errorMessage }` | Processing failed |

---

## 10.5 Prompt Engineering

### System Prompt (draft, tuneable)

```text
You are Quaero, an assistant that answers questions about PDF documents.
You must answer ONLY using the provided context. If the context does not contain
enough information to answer, respond: "I could not find relevant information
in the provided documents to answer this question."

Rules:
- Never fabricate information.
- Always cite the source when referencing specific content.
- Answer in the same language as the question.
- Be concise and precise.
```

### Context Injection Format

```xml
<documents>
<document name="Contract.pdf" page="18">
The notice period for termination shall be 30 days...
</document>
<document name="Contract.pdf" page="22">
Termination must be communicated in writing...
</document>
</documents>
```

### Message Assembly Order

1. System prompt
2. Context chunks (top-K retrieved, formatted as above)
3. Conversation history (last N messages, configurable, default 10)
4. Current user question

### Refusal Behavior

When no chunks pass the similarity threshold (cosine distance > 0.5 — configurable):

- Do NOT send to LLM.
- Return a pre-defined response: "No relevant passages found in the associated
  documents for your question."
- This saves LLM tokens and avoids hallucination.

---

## 11. LLM Provider Abstraction

```rust
#[async_trait]
trait LlmProvider: Send + Sync {
    async fn chat(&self, messages: Vec<ChatMessage>) -> Result<String, LlmError>;
    fn name(&self) -> &str;
}

struct GlmProvider {
    api_url: String,
    api_key: String,
    model: String,
}

struct OllamaChatProvider {
    url: String,
    model: String,
}
```

Configurable via settings UI (persisted to a local config file, e.g. TOML or JSON).

---

## 12. Folder Layout

```
src-tauri/                    # Rust backend (Tauri)
  src/
    main.rs                   # Tauri entry
    lib.rs                    # Command registration
    commands/
      documents.rs            # Document CRUD + upload commands
      conversations.rs        # Conversation + chat commands
      config.rs               # Settings commands
      health.rs               # Health check
    services/
      document_service.rs     # Upload, parse, chunk, embed orchestration
      embedding_client.rs     # HTTP client to Ollama (bge-m3)
      chat_service.rs         # RAG pipeline orchestrator
      retriever.rs            # Embed question → cosine search → top-K
      prompt_builder.rs       # System prompt + context + history + question
      llm_provider.rs         # LlmProvider trait + GLM + Ollama impls
    db/
      schema.rs               # Table definitions (rusqlite)
      client.rs               # Connection pool, init, pragmas
      migrations.rs           # Inline migrations (or bundled SQL files)
    config/
      settings.rs             # Typed config loader + persistence
  Cargo.toml
  tauri.conf.json

src/                          # Vue frontend
  main.ts                     # Vue entry
  App.vue                     # Root component + router + Pinia
  style.css                   # Tailwind directives + Shadcn CSS vars
  router/
    index.ts                  # Vue Router config
  stores/
    documents.ts              # Pinia store for documents
    conversations.ts          # Pinia store for conversations + chat
    settings.ts               # Pinia store for app settings
    theme.ts                  # Theme state (light/dark/system)
  lib/
    tauri-api.ts              # Typed wrappers around invoke() + Zod validation
  schemas/
    document.schema.ts        # Zod schemas for document shapes
    conversation.schema.ts    # Zod schemas for conversation/message/citation shapes
  components/
    ui/                       # Shadcn-vue primitives
    layout/
      AppLayout.vue           # Sidebar + main content shell
      Sidebar.vue             # Navigation sidebar + theme toggle
    documents/
      DropZone.vue            # Drag-and-drop + native file picker
      DocumentList.vue        # Table with status badges + actions
      DocumentActions.vue     # Rename/delete per row
    conversations/
      ConversationList.vue    # List of conversation cards
      CreateConversationDialog.vue  # Title + document selector
    chat/
      MessageList.vue         # Scrollable messages + auto-scroll + suggested prompts
      MessageBubble.vue       # User or assistant message
      MarkdownMessage.vue     # Markdown renderer (markdown-it or similar)
      CitationCard.vue        # Expandable citation (doc + page + snippet)
      DocumentPanel.vue       # Sidebar showing associated documents
      ChatInput.vue           # Multi-line input + send
    theme/
      ThemeToggle.vue         # Light/dark/system dropdown
  views/
    DocumentsView.vue         # Documents page
    ConversationsView.vue     # Conversations list page
    ChatView.vue              # Chat interface for a conversation
    SettingsView.vue          # LLM provider config + Ollama URL
```

---

## 13. Non-Functional Requirements

### Performance

- Upload accepted up to 200 MB; processed up to 1000 pages / 200 MB.
- Async processing: UI remains responsive during ingestion.
- Chat response: target `< 5 s` (depends on LLM latency).
- Embedding concurrency tunable via config (default 8).

### Security

- No auth (single-user, local desktop).
- Upload validation (type/size/pages).
- API keys stored in local config file (not bundled in binary).

### Desktop-specific

- Native file dialogs for PDF selection.
- Drag-and-drop onto the application window.
- Single-instance (Tauri plugin prevents multiple instances).
- Application data stored in OS-standard directory (`app_data_dir`).
- Database file stored alongside config in app data directory.

### Cross-platform

- macOS: `.dmg` bundle.
- Windows: `.msi` installer.
- Linux: `.AppImage`.

---

## 13.5 Error States & Recovery

| Scenario | Detection | User-Facing Message | Recovery |
|---|---|---|---|
| Ollama unreachable (embed) | HTTP connection refused/timeout on embedding request | "Ollama is not running. Please start Ollama and ensure `bge-m3` is pulled (`ollama pull bge-m3`)." | Document status → `failed` with `error_code = "OLLAMA_UNREACHABLE"`. Retry on next upload or health check. |
| LLM API timeout | HTTP timeout after configurable threshold (default 30 s) | "The LLM service timed out. Please try again or check your connection." | Message still saved with error state. User can retry. |
| LLM API auth failure | HTTP 401/403 | "API key is invalid. Please check your settings." | No retry. Direct user to Settings. |
| Corrupt/encrypted PDF | `pdf-extract` returns error or empty output | "This PDF could not be processed. It may be encrypted or corrupted." | Document status → `failed`. No retry. |
| PDF exceeds limits | Size > 200 MB or pages > 1000 (checked before processing) | "This PDF exceeds the limit (200 MB / 1000 pages)." | Rejected before processing. No DB entry. |
| SQLite write failure | `rusqlite` returns error on insert/update | "A database error occurred. Please restart the application." | Log full error. App continues if possible. |
| Disk space low | Check before large operations (heuristic) | "Not enough disk space to process this document." | Reject gracefully. |

### Retry Policy

- Embedding requests: retry 2x with 1 s backoff per chunk. If still failing, mark document `failed`.
- LLM requests: no automatic retry. User sees error and can resend.
- No exponential backoff in v1 (simple fixed delay).

---

## 14. Success Criteria

### Quantitative

| Metric | Target | Measurement |
|---|---|---|
| Chat response latency | ≤ 5 s (P95) for docs ≤ 100 pages | Timed from `send_message` to response receipt |
| Processing failure rate | ≤ 1% for valid PDFs under limits | (failed documents / total uploaded) × 100 |
| Citation accuracy | ≥ 95% correct doc + page on benchmark set | 20-question benchmark across 3 PDF types |
| RAG retrieval recall | ≥ 90% Recall@5 on benchmark | Ground-truth chunks identified per question |
| App cold start | ≤ 3 s to interactive | Timed from launch to DocumentsView rendered |

### Qualitative

- Answers are grounded in document content (no fabrication detected in benchmark).
- Citations link to verifiable page + snippet in the PDF.
- UI is responsive during document processing (no freezes).

### Benchmark Method

- Prepare 3 PDFs: a contract (30 pp), a technical manual (80 pp), a research paper (15 pp).
- Write 20 questions with known answers + ground-truth page numbers.
- Run all questions through Quaero and score citation accuracy + answer correctness.

---

## 15. Technical Differentiators

- Complete RAG pipeline with vector search in a **self-contained desktop app**.
- Embedded vector store (sqlite-vec) — no external database process.
- Rust backend — memory-safe, high performance, small binary.
- Configurable LLM provider (remote API or local Ollama).
- Verifiable and persisted citations (traceability).
- Cross-platform distribution via Tauri.
- 100% local / zero cost.

---

## 16. External Dependencies

| Dependency | Type | Required | Notes |
|---|---|---|---|
| Ollama | External service | Yes (embeddings) | User must install + pull `bge-m3` |
| GLM-5.1 | Remote API | Optional (chat) | Only if using remote LLM provider |
| System WebView | OS-provided | Yes | Tauri uses the system's WebView |

---

## 17. Risk Analysis

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| `pdf-extract` produces poor output on complex PDFs (tables, columns, images) | High — answers based on garbled text | Medium | Validate with 5 diverse PDFs in Bolt 1. If quality < 90%, switch to `pdfium-render` (bundled pdfium binary). |
| sqlite-vec immaturity (API changes, bugs) | Medium — vector search breaks | Low-Medium | Pin exact sqlite-vec version in `Cargo.toml`. Test cosine search in Bolt 1. |
| Ollama as hard external dependency | Medium — users may not have it installed | High | Health check on app start. Clear error message with install instructions. Future: in-process ONNX embeddings. |
| Large binary size (Rust + SQLite + pdfium fallback) | Low — download size | Medium | Target < 30 MB without pdfium. Monitor with `cargo bloat`. |
| LLM hallucination despite grounding prompt | High — incorrect answers presented as fact | Medium | Strict prompt engineering. Refusal when no relevant context. Citation validation: every citation must resolve to an actual chunk in DB. |
| Single-threaded embedding bottleneck | Low — slow processing for large documents | High | Parallel embedding with configurable concurrency (default 8). Async from day one. |

---

## 18. Open Questions

| # | Question | Owner | Deadline | Default if unresolved |
|---|---|---|---|---|
| OQ-1 | `pdf-extract` extraction quality on complex PDFs (tables, columns). Need validation. | Developer | End of Bolt 1 | Stick with `pdf-extract`; document known limitations. |
| OQ-2 | Should the app auto-detect Ollama + prompt `bge-m3` download, or document as prerequisite? | Stakeholder | Before Bolt 1 spec approval | Document as prerequisite in README + health check error message. |
| OQ-3 | Code signing (Apple notarization, Windows cert) for distribution. | Stakeholder | Post-v1 | Unsigned builds initially. |
| OQ-4 | Chunk size tuning: character-based vs token-based splitting. Evaluate after Bolt 2. | Developer | End of Bolt 2 | Character-based (512 chars) for v1. |
| OQ-5 | Conversation history window: how many past messages to include in prompt? | Developer | During Bolt 2 implementation | Default 10 messages. Configurable via settings. |
