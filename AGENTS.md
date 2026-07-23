# AGENTS.md — Quaero

> Steering doc for any AI (Claude Code, opencode, etc.) working in this repo.
> `CLAUDE.md` is a symlink to this file. Edit **only** this one.
> Read this **before** touching code.

## What the project is

Quaero — cross-platform desktop app for chatting over PDFs via RAG, with verifiable
citations (doc + page + snippet resolved from the database, never hallucinated).
Built with Electron (Node.js main process + React 19 renderer). Portfolio MVP: runs
100% locally, ~zero cost, no deploy. Details in [`specs/PRD.md`](specs/PRD.md).

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

- **Desktop shell:** Electron (Node.js main process + bundled Chromium renderer).
- **Frontend:** React 19 + TypeScript + Vite (via `electron-vite`) + Shadcn/ui + Tailwind CSS + Zustand + React Router v7.
- **PDF parsing:** `pdfjs-dist` (Mozilla pdf.js, per-page text extraction).
- **Embeddings:** Ollama + `bge-m3` (local, 1024 dims) — HTTP to `localhost:11434`.
- **Vector store:** SQLite + sqlite-vec (embedded, `better-sqlite3`, cosine distance).
- **Chat LLM:** `LlmProvider` TypeScript interface; MVP impls = GLM-5.1 (remote, OpenAI-compatible) + Ollama (local).
- **Validation:** Zod (renderer response schemas).
- **Build:** `electron-vite` (dev + renderer/main build) + `electron-builder` (`.dmg`/`.msi`/`.AppImage`).
- **Lint/format:** ESLint + Prettier; TypeScript `strict`.

## Folder layout

```
specs/                  # SDD: 1 spec per feature — source of truth; PRD.md is inception doc
electron/               # Node.js backend (Electron main + preload)
  main/
    index.ts            # Electron entry: app lifecycle, BrowserWindow, single-instance lock
    ipc/                # ipcMain.handle handlers + typed register()
    services/           # Business logic (RAG pipeline, embedding, etc.)
    db/                 # SQLite schema, client (better-sqlite3), migrations
    config/             # Typed settings loader + persistence (JSON)
  preload/
    index.ts            # contextBridge.exposeInMainWorld('api', ...) typed API
  electron-builder.yml  # Bundle config (.dmg / .msi / .AppImage)
src/                    # React 19 renderer
  main.tsx              # React entry (createRoot)
  App.tsx               # Root component + router + providers
  router/               # React Router v7 config
  stores/               # Zustand stores
  lib/                  # Electron API wrappers (window.api), utilities
  schemas/              # Zod validation schemas
  components/           # React components (ui/, layout/, documents/, chat/, etc.)
  views/                # Page-level components
```

## Commands

```bash
# Development
npm run dev                  # electron-vite dev (Electron main + React HMR)

# Build
npm run build                # electron-vite build (main + preload + renderer)
npm run package              # electron-builder: produce .dmg / .msi / .AppImage

# Type checks
npm run typecheck            # tsc --noEmit (main + renderer)

# Lint / format
npm run lint                 # ESLint
npm run format               # Prettier

# Setup
ollama pull bge-m3           # Download embedding model (once)
```

## Conventions

- TypeScript `strict` everywhere (main + renderer). No `any` unless justified.
- Main process: typed `ipcMain.handle` handlers; renderer never touches Node APIs
  directly — all access mediated via `contextBridge` (`window.api`).
- Electron security defaults: `contextIsolation: true`, `nodeIntegration: false`,
  `sandbox: true` unless a service specifically requires otherwise.
- Error handling: services return `Result<T, LlmError>`-style discriminated unions
  (`{ ok: true, value } | { ok: false, error }`) — no thrown exceptions across IPC.
- Citation = always by chunk ID → page/snippet come from the database, never from LLM generation.
- Strict grounding: chat answers only from the retrieved context; refuses outside it.
- Frontend: all `window.api` responses validated through Zod schemas.
- Commits: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`).

## Out of MVP scope (do not implement)

JWT multi-user auth, MCP server, OCR, PDF comparison, export, auto-update,
in-process embeddings (ONNX). All of this is "future" (PRD §6). Do not add it on
your own.

## Current state

- Inception ✅ (PRD locked).
- No Bolts implemented yet.
- Next: **Bolt 0** (project scaffold: Electron + React + Vite + Shadcn/ui + Zustand + SQLite init + health check).

## Bolt roadmap

| Bolt | Scope | Status |
|---|---|---|
| 0 | Project scaffold: Electron + React + Vite + Shadcn/ui + Zustand + SQLite + health check | 🔲 not started |
| 1 | Upload + Processing: PDF parser → chunker → Ollama embedding → sqlite-vec persist | 🔲 not started |
| 2 | Chat / RAG Pipeline: embed question → cosine search → prompt → LLM → persist citations | 🔲 not started |
| 3 | Conversations CRUD: create, list, detail, associate documents | 🔲 not started |
| 4 | Documents screen (React): DropZone, DocumentList, status badges, rename, delete | 🔲 not started |
| 5 | Conversations + Chat screen (React): conversation list, chat view, citations, markdown | 🔲 not started |
| 6 | Settings + Theme UI: LLM provider config, Ollama URL, dark mode toggle | 🔲 not started |
| 7 | Polish: typing indicator, suggested prompts, auto-scroll, FOUC-free theme | 🔲 not started |
