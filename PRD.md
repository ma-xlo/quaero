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
| Platform | **Electron** (Node.js main process + bundled Chromium renderer) | Mature desktop ecosystem, consistent rendering across OSes (no WebView fragmentation) |
| Frontend | **React 19 + TypeScript + Shadcn/ui + Tailwind CSS** | Original Shadcn primitives, large component library, large talent pool |
| Vector store | **SQLite + sqlite-vec** (embedded via `better-sqlite3`) | No external database; single-file, zero config |
| Embeddings | **Ollama + `bge-m3`** (local, 1024 dims) | Free, CPU, multilingual PT-BR |
| Chat LLM | **Configurable `LlmProvider`**: GLM-5.1 (remote API) or Ollama (local) | Flexibility without over-engineering |
| PDF parsing | **`pdfjs-dist`** (Mozilla pdf.js, per-page text extraction) | Robust on complex layouts (tables/columns), well-maintained, no system deps |
| Processing | **Async** (main-process worker, non-blocking renderer) | Desktop UX demands responsive UI during ingestion |
| Auth | **No auth, single-user** | Focus on the RAG core |
| Distribution | **electron-builder** (`.dmg` / `.msi` / `.AppImage`) | De-facto cross-platform packaging for Electron |
| Citations | **Persisted** per message | Traceability survives restart |
| State management | **Zustand** | Minimal, hook-based, TypeScript-first |
| Routing | **React Router v7** | Standard, type-safe, mature data router |

---

## 5. Scope

### PDF Upload

- Native file dialog or drag-and-drop for one or more PDFs.
- Upload cap: 200 MB (configurable via settings).
- Processed asynchronously: up to 1000 pages / 200 MB per file. Above that → clear error.

### Processing (async, on upload)

1. Text extraction **per page** (`pdfjs-dist` — required for citation).
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

### Frontend (React)

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
- FOUC-free boot (theme applied before React hydration).

---

## 6. Future Features

- **MCP server** (expose tools for external clients like Claude Desktop).
- JWT multi-user authentication + isolation.
- OCR for scanned PDFs.
- PDF comparison; table extraction.
- Export (Markdown/PDF/DOCX); structured automatic summarization.
- In-process embeddings (ONNX — no external Ollama dependency).
- Configurable / multi-model embedding (different dimensions).
- Auto-update via `electron-updater`.

---

## 7. Architecture

```text
┌─────────────────────────────────────────────────────┐
│  React 19 + Shadcn/ui + Tailwind CSS + Zustand      │
│  (rendered in bundled Chromium via Electron)         │
└───────────────────────┬─────────────────────────────┘
                        │ Electron IPC
                        │ (ipcRenderer.invoke / Main→Renderer events
                        │  via contextBridge-typed API)
┌───────────────────────▼─────────────────────────────┐
│  Node.js Main Process (Electron)                     │
│  ├─ Document Service                                 │
│  │   ├─ PDF Parser (pdfjs-dist, per page)            │
│  │   ├─ Chunker (fixed + overlap)                    │
│  │   └─ Embedding Client ───► Ollama (bge-m3) [HTTP] │
│  ├─ Chat Service (RAG)                               │
│  │   ├─ Embedding Client ───► Ollama (bge-m3)        │
│  │   ├─ Retriever ──────────► sqlite-vec (cosine)     │
│  │   ├─ Prompt Builder                                │
│  │   └─ LlmProvider ────────► GLM-5.1 (remote)       │
│  │                          └─► Ollama (local)        │
│  ├─ Vector Store (sqlite-vec via better-sqlite3)      │
│  └─ SQLite (better-sqlite3, single-file DB)           │
└──────────────────────────────────────────────────────┘
```

External dependency (user runs locally):

- **Ollama** (`bge-m3`) — embeddings. Optionally also for chat LLM.
- **GLM-5.1** — remote API (OpenAI-compatible) — the only potential cost point.

---

## 8. Tech Stack

| Layer | Technology |
|---|---|
| Desktop shell | **Electron** (Node.js main process + Chromium renderer) |
| Frontend | **React 19** + **TypeScript** + **Vite** (via `electron-vite`) |
| UI components | **Shadcn/ui** + **Tailwind CSS** |
| State management | **Zustand** |
| Routing | **React Router v7** |
| PDF parsing | **`pdfjs-dist`** (Mozilla pdf.js, per-page text extraction) |
| Embeddings | **Ollama + `bge-m3`** (HTTP, 1024 dims) |
| Vector store | **SQLite + sqlite-vec** (embedded via `better-sqlite3`) |
| LLM | **`LlmProvider` interface**: GLM-5.1 (fetch HTTP) or Ollama |
| Validation | **Zod** (renderer response schemas) |
| Build/bundle | **electron-vite** + **electron-builder** (`npm run build`) |

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
- Migrations are inline TypeScript functions in `electron/main/db/migrations.ts`, not SQL files.

---

## 10. Electron IPC Contracts

All backend operations exposed as **Electron IPC handlers** registered in the main
process (`ipcMain.handle`) and invoked from the renderer through a typed
`contextBridge` API (`window.api.<command>(...)`).

| Command | Direction | Description |
|---|---|---|
| `upload_documents` | Renderer → Main | Accept file paths from native dialog/drop; validate; insert as `processing`; spawn background processing |
| `list_documents` | Renderer → Main | Return all documents with status |
| `get_document` | Renderer → Main | Single document detail |
| `rename_document` | Renderer → Main | Update display filename |
| `delete_document` | Renderer → Main | Cascade delete (chunks, links, citations) in transaction |
| `list_conversations` | Renderer → Main | Return all conversations |
| `get_conversation` | Renderer → Main | Conversation detail with documents, messages, citations |
| `create_conversation` | Renderer → Main | Create conversation, optionally associate documents |
| `associate_documents` | Renderer → Main | Add documents to a conversation |
| `send_message` | Renderer → Main | RAG pipeline: embed → retrieve → prompt → LLM → persist → return with citations |
| `get_config` | Renderer → Main | Read current settings (LLM provider, Ollama URL, etc.) |
| `update_config` | Renderer → Main | Persist settings |
| `check_health` | Renderer → Main | Verify Ollama connectivity + DB status |

### Async events (Main → Renderer)

| Event | Payload | When |
|---|---|---|
| `document:processing` | `{ documentId, status }` | Document status changes during processing |
| `document:ready` | `{ documentId }` | Processing complete |
| `document:failed` | `{ documentId, errorCode, errorMessage }` | Processing failed |

> Events are pushed from the main process via `webContents.send(...)` and exposed to
> React through a typed `window.api.onDocumentEvent(callback)` subscription API
> (cleaned up via the returned disposer to avoid renderer-side leaks).

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

```typescript
// electron/main/services/llm-provider.ts

interface LlmProvider {
  readonly name: string;
  chat(messages: ChatMessage[]): Promise<Result<string, LlmError>>;
}

class GlmProvider implements LlmProvider {
  readonly name = "glm-5.1";
  constructor(
    private readonly apiUrl: string,
    private readonly apiKey: string,
    private readonly model: string,
  ) {}
  async chat(messages: ChatMessage[]): Promise<Result<string, LlmError>> { /* fetch ... */ }
}

class OllamaChatProvider implements LlmProvider {
  readonly name = "ollama";
  constructor(
    private readonly url: string,
    private readonly model: string,
  ) {}
  async chat(messages: ChatMessage[]): Promise<Result<string, LlmError>> { /* fetch ... */ }
}
```

Configurable via settings UI (persisted to a local config file, e.g. JSON at
`app.getPath('userData')`).

---

## 12. Folder Layout

```
electron/                       # Electron main process (Node.js)
  main/
    index.ts                    # Electron entry: app lifecycle, BrowserWindow, single-instance lock
    ipc/
      register.ts               # Registers all ipcMain.handle handlers
      documents.ts              # Document CRUD + upload IPC handlers
      conversations.ts          # Conversation + chat IPC handlers
      config.ts                 # Settings IPC handlers
      health.ts                 # Health check IPC handler
    services/
      document-service.ts       # Upload, parse, chunk, embed orchestration
      embedding-client.ts       # HTTP client to Ollama (bge-m3)
      chat-service.ts           # RAG pipeline orchestrator
      retriever.ts              # Embed question → cosine search → top-K
      prompt-builder.ts         # System prompt + context + history + question
      llm-provider.ts           # LlmProvider interface + GLM + Ollama impls
    db/
      schema.ts                 # Table definitions (better-sqlite3)
      client.ts                 # Connection, init, pragmas, sqlite-vec load
      migrations.ts             # Inline migrations (or bundled SQL files)
    config/
      settings.ts               # Typed config loader + persistence (JSON)
  preload/
    index.ts                    # contextBridge.exposeInMainWorld('api', ...) typed API
  electron-builder.yml          # Bundle config (.dmg / .msi / .AppImage)
  tsconfig.json

src/                            # React renderer
  main.tsx                      # React entry (createRoot)
  App.tsx                       # Root component + router + providers
  index.css                     # Tailwind directives + Shadcn CSS vars
  router/
    index.tsx                   # React Router v7 config (data router)
  stores/
    documents-store.ts          # Zustand store for documents
    conversations-store.ts      # Zustand store for conversations + chat
    settings-store.ts           # Zustand store for app settings
    theme-store.ts              # Theme state (light/dark/system)
  lib/
    electron-api.ts             # Typed wrappers around window.api + Zod validation
  schemas/
    document.schema.ts          # Zod schemas for document shapes
    conversation.schema.ts      # Zod schemas for conversation/message/citation shapes
  components/
    ui/                         # Shadcn/ui primitives
    layout/
      AppLayout.tsx             # Sidebar + main content shell
      Sidebar.tsx               # Navigation sidebar + theme toggle
    documents/
      DropZone.tsx              # Drag-and-drop + native file picker
      DocumentList.tsx          # Table with status badges + actions
      DocumentActions.tsx       # Rename/delete per row
    conversations/
      ConversationList.tsx      # List of conversation cards
      CreateConversationDialog.tsx  # Title + document selector
    chat/
      MessageList.tsx           # Scrollable messages + auto-scroll + suggested prompts
      MessageBubble.tsx         # User or assistant message
      MarkdownMessage.tsx       # Markdown renderer (react-markdown or similar)
      CitationCard.tsx          # Expandable citation (doc + page + snippet)
      DocumentPanel.tsx         # Sidebar showing associated documents
      ChatInput.tsx             # Multi-line input + send
    theme/
      ThemeToggle.tsx           # Light/dark/system dropdown
  views/
    DocumentsView.tsx           # Documents page
    ConversationsView.tsx       # Conversations list page
    ChatView.tsx                # Chat interface for a conversation
    SettingsView.tsx            # LLM provider config + Ollama URL
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
- Single-instance (Electron `app.requestSingleInstanceLock()`).
- Application data stored in OS-standard directory (`app.getPath('userData')`).
- Database file stored alongside config in app data directory.
- Context isolation **on**, `nodeIntegration` **off**, all main-process access
  mediated through a typed `contextBridge` API (security best practice).

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
| Corrupt/encrypted PDF | `pdfjs-dist` returns error or empty output | "This PDF could not be processed. It may be encrypted or corrupted." | Document status → `failed`. No retry. |
| PDF exceeds limits | Size > 200 MB or pages > 1000 (checked before processing) | "This PDF exceeds the limit (200 MB / 1000 pages)." | Rejected before processing. No DB entry. |
| SQLite write failure | `better-sqlite3` throws on `prepare`/`run` | "A database error occurred. Please restart the application." | Log full error. App continues if possible. |
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
| App cold start | ≤ 5 s to interactive (Electron-inflated baseline) | Timed from launch to DocumentsView rendered |

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
- **Consistent cross-platform rendering** via bundled Chromium (no system-WebView
  fragmentation between macOS/Windows/Linux).
- **Mature Node.js + React ecosystem** — large talent pool, abundant libraries,
  first-class Shadcn/ui component source.
- Configurable LLM provider (remote API or local Ollama).
- Verifiable and persisted citations (traceability).
- Cross-platform distribution via Electron (`.dmg` / `.msi` / `.AppImage`).
- 100% local / zero cost.

---

## 16. External Dependencies

| Dependency | Type | Required | Notes |
|---|---|---|---|
| Ollama | External service | Yes (embeddings) | User must install + pull `bge-m3` |
| GLM-5.1 | Remote API | Optional (chat) | Only if using remote LLM provider |
| Chromium | Bundled with Electron | Yes | Consistent rendering across OSes (no system WebView dependency) |

---

## 17. Risk Analysis

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| `pdfjs-dist` produces poor output on edge-case PDFs (rare — pdf.js is the de-facto standard renderer) | High — answers based on garbled text | Low | pdf.js is robust on tables/columns. Validate with 5 diverse PDFs in Bolt 1. If a problematic PDF appears, try alternative rendering flags (e.g. `disableFontFace`, `useSystemFonts`) before falling back. |
| sqlite-vec Node bindings immaturity (npm `sqlite-vec`, loadable extension) | Medium — vector search breaks | Low-Medium | Pin exact `sqlite-vec` version in `package.json`. Load extension at startup via `db.loadExtension(...)`. Test cosine search in Bolt 1. |
| Ollama as hard external dependency | Medium — users may not have it installed | High | Health check on app start. Clear error message with install instructions. Future: in-process ONNX embeddings. |
| Large binary size (Chromium runtime + SQLite + pdf.js ≈ 100–150 MB) | Low — download size, accepted tradeoff | High (certain) | Accepted as inherent to Electron. No mitigation — communicate size in README. Removed from §15 differentiators. |
| Electron memory footprint (~200–400 MB RAM) | Low — desktop-class, accepted | High (certain) | Accepted. Mitigation: lazy-load heavy services (PDF parser, embedder) in the main process; avoid renderer-side retention of large blobs. |
| LLM hallucination despite grounding prompt | High — incorrect answers presented as fact | Medium | Strict prompt engineering. Refusal when no relevant context. Citation validation: every citation must resolve to an actual chunk in DB. |
| Single-threaded embedding bottleneck | Low — slow processing for large documents | High | Worker thread (`worker_threads`) for embedding with configurable concurrency (default 8). Off-main-thread so renderer stays responsive. |

---

## 18. Open Questions

| # | Question | Owner | Deadline | Default if unresolved |
|---|---|---|---|---|
| OQ-1 | `pdfjs-dist` extraction quality on edge-case PDFs (rare tables/columns). Need validation. | Developer | End of Bolt 1 | Stick with `pdfjs-dist` (standard, robust); document known limitations + tune extraction flags if needed. |
| OQ-2 | Should the app auto-detect Ollama + prompt `bge-m3` download, or document as prerequisite? | Stakeholder | Before Bolt 1 spec approval | Document as prerequisite in README + health check error message. |
| OQ-3 | Code signing (Apple notarization, Windows cert) for distribution. | Stakeholder | Post-v1 | Unsigned builds initially. |
| OQ-4 | Chunk size tuning: character-based vs token-based splitting. Evaluate after Bolt 2. | Developer | End of Bolt 2 | Character-based (512 chars) for v1. |
| OQ-5 | Conversation history window: how many past messages to include in prompt? | Developer | During Bolt 2 implementation | Default 10 messages. Configurable via settings. |
