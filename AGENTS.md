# AGENTS.md — Quaero

> Steering doc for any AI (Claude Code, opencode, etc.) working in this repo.
> `CLAUDE.md` is a symlink to this file. Edit **only** this one.
> Read this **before** touching code.

## What the project is

Quaero — cross-platform desktop app for chatting over PDFs via RAG, with verifiable
citations (doc + page + snippet resolved from the database, never hallucinated).
Built with Tauri 2.x (Rust backend + Vue 3 frontend). Portfolio MVP: runs 100%
locally, ~zero cost, no deploy. Details in [`specs/PRD.md`](specs/PRD.md).

## Methodology: AI-DLC + Spec Driven Development

Current phase: **Inception complete** (PRD locked). Moving to **Construction**,
organized into **Bolts** (short delivery units).

**Hard rule — spec before code:**

```
write spec → stakeholder approves → implement against the spec → validate acceptance criteria → next Bolt
```

- Every Bolt spec lives in `specs/<feature-name>.md` (kebab-case, no `bolt-N` prefix).
- Before implementing a Bolt, read its spec. It is the source of truth.
- Do not implement anything outside the current Bolt's spec scope.
- When done, validate the spec's "Acceptance criteria" checklist — do not declare done without it.

## Locked stack (do not swap without approval)

- **Desktop shell:** Tauri 2.x (Rust backend + system WebView).
- **Frontend:** Vue 3 + TypeScript + Vite + Shadcn-vue + Tailwind CSS + Pinia + Vue Router.
- **PDF parsing:** `pdf-extract` (Rust, lopdf-based, per-page extraction).
- **Embeddings:** Ollama + `bge-m3` (local, 1024 dims) — HTTP to `localhost:11434`.
- **Vector store:** SQLite + sqlite-vec (embedded, `rusqlite`, cosine distance).
- **Chat LLM:** `LLMProvider` trait; MVP impls = GLM-5.1 (remote, OpenAI-compatible) + Ollama (local).
- **Validation:** Zod (frontend response schemas).
- **Build:** Tauri CLI (`cargo tauri dev`, `cargo tauri build`).
- **Lint/format:** Rust: `cargo fmt` + `cargo clippy`. Frontend: ESLint + Prettier.

## Folder layout

```
specs/                  # SDD: 1 spec per feature — source of truth; PRD.md is inception doc
src-tauri/              # Rust backend (Tauri)
  src/
    main.ts             # Tauri entry
    lib.rs              # Command registration
    commands/           # Tauri IPC command handlers
    services/           # Business logic (RAG pipeline, embedding, etc.)
    db/                 # SQLite schema, client, migrations
    config/             # Typed settings loader + persistence
  Cargo.toml
  tauri.conf.json
src/                    # Vue 3 frontend
  main.ts              # Vue entry
  App.vue              # Root component + router + Pinia
  router/              # Vue Router config
  stores/              # Pinia stores
  lib/                 # Tauri IPC wrappers, utilities
  schemas/             # Zod validation schemas
  components/          # Vue components (ui/, layout/, documents/, chat/, etc.)
  views/               # Page-level components
```

## Commands

```bash
# Development
cargo tauri dev              # Start Tauri dev server (Rust + Vue hot reload)
npm run dev                  # Vite dev server only (frontend)

# Build
cargo tauri build            # Production build for current platform
npm run build                # Vite build (frontend only)

# Rust checks
cargo build                  # Compile Rust backend
cargo test                   # Run Rust tests
cargo clippy                 # Lint Rust
cargo fmt --check            # Check Rust formatting

# Frontend checks
npm run lint                 # ESLint
npm run typecheck            # vue-tsc

# Setup
ollama pull bge-m3           # Download embedding model (once)
```

## Conventions

- Rust: idiomatic, `Result<T, E>` error handling, no `unwrap()` in production code.
- TypeScript `strict`. No `any` unless justified.
- Citation = always by chunk ID → page/snippet come from the database, never from LLM generation.
- Strict grounding: chat answers only from the retrieved context; refuses outside it.
- Tauri IPC: all commands return `Result<T, String>` or a typed error struct.
- Frontend: all Tauri `invoke()` responses validated through Zod schemas.
- Commits: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`).

## Out of MVP scope (do not implement)

JWT multi-user auth, MCP server, OCR, PDF comparison, export, auto-update,
in-process embeddings (ONNX). All of this is "future" (PRD §6). Do not add it on
your own.

## Current state

- Inception ✅ (PRD locked).
- No Bolts implemented yet.
- Next: **Bolt 0** (project scaffold: Tauri + Vue + Vite + Shadcn-vue + Pinia + SQLite init + health check).

## Bolt roadmap

| Bolt | Scope | Status |
|---|---|---|
| 0 | Project scaffold: Tauri + Vue + Vite + Shadcn-vue + Pinia + SQLite + health check | 🔲 not started |
| 1 | Upload + Processing: PDF parser → chunker → Ollama embedding → sqlite-vec persist | 🔲 not started |
| 2 | Chat / RAG Pipeline: embed question → cosine search → prompt → LLM → persist citations | 🔲 not started |
| 3 | Conversations CRUD: create, list, detail, associate documents | 🔲 not started |
| 4 | Documents screen (Vue): DropZone, DocumentList, status badges, rename, delete | 🔲 not started |
| 5 | Conversations + Chat screen (Vue): conversation list, chat view, citations, markdown | 🔲 not started |
| 6 | Settings + Theme UI: LLM provider config, Ollama URL, dark mode toggle | 🔲 not started |
| 7 | Polish: typing indicator, suggested prompts, auto-scroll, FOUC-free theme | 🔲 not started |
