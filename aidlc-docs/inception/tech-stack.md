# Tech Stack — Quaero

> **Feeds the AI-DLC Inception Phase.** Specifies language, framework, deploy,
> testing, banned libraries, and code examples — so decomposition starts from
> the **real** project context.
>
> **Source of truth:** [`/PRD.md`](../../PRD.md) (§4 Principles, §8 Tech Stack,
> §13 NFR). Locked decisions (`docs/STEERING.md` → "Locked stack") **cannot be
> swapped without approval**.

---

## 1. What is being built, and for whom

**Product:** Quaero — a **desktop** app (macOS / Windows / Linux) for chatting
over PDFs via **RAG**, with **verifiable, persisted citations** (document +
page + snippet). Self-contained, **100% local**, ~zero cost, no deploy.

**For whom:**
- **Alex** — legal/compliance analyst (contracts / tenders / regulations of 30–100 pp; needs to cite the exact page).
- **Pat** — graduate researcher (papers / reports of 50–200 pp; comparative questions).

**Shape:** Electron = **Node.js** main process (backend: parsing, chunking,
embedding, RAG, SQLite, sqlite-vec) + **React 19** renderer (bundled Chromium).
Communication via **typed IPC** (`contextBridge`).

---

## 2. Language, version, framework, and package manager

| Item | Decision | Note |
|---|---|---|
| **Language** | **TypeScript** (`strict`, no `any` unless justified) | Across all code (main + renderer) |
| **Main runtime** | **Node.js** (bundled with Electron) | Version = the target Electron's Node (no separate Node for runtime) |
| **Renderer** | **Chromium** (bundled with Electron) | Consistent cross-platform rendering (no OS WebView) |
| **Desktop shell** | **Electron** | `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true` |
| **UI / renderer** | **React 19** + **Vite** (via `electron-vite`) + **Shadcn/ui** + **Tailwind CSS** | |
| **State** | **Zustand** | Hook-based, TS-first |
| **Routing** | **React Router v7** | Data router, type-safe |
| **PDF parsing** | **`pdfjs-dist`** (Mozilla pdf.js, per-page extraction) | |
| **Embeddings** | **Ollama + `bge-m3`** (1024 dims, multilingual) | HTTP `localhost:11434` |
| **Vector store** | **SQLite + sqlite-vec** (embedded via `better-sqlite3`, cosine) | **Pin exact `sqlite-vec` version** (PRD §17) |
| **LLM** | **`LlmProvider`** (interface) — GLM-5.1 (remote, OpenAI-compatible) **or** Ollama (local) | Configurable via UI |
| **Validation** | **Zod** (renderer response schemas) | |
| **Build/bundle** | **`electron-vite`** + **`electron-builder`** (`.dmg`/`.msi`/`.AppImage`) | |
| **Lint/format** | **ESLint** + **Prettier** | |
| **Package manager** | **npm** | PRD uses `npm run ...`; no `yarn.lock`/`pnpm-lock` yet |

> **Exact versions are not pinned in the PRD.** Decision to make in **Bolt 0**
> (scaffold): pin `react@19`, `electron@<latest>`, **`sqlite-vec@<exact>`**,
> `better-sqlite3`, `pdfjs-dist`, `zustand`, `react-router@7`. Recommended:
> `^` for minor ranges in `package.json` + commit `package-lock.json` for
> reproducibility.

---

## 3. Cloud provider and deployment model

**No cloud provider.** The product is desktop, **100% local**, with no backend
deploy. No server, no managed DB, no queue, no CDN.

**Distribution:** `electron-builder` produces local installers:
- macOS → `.dmg`
- Windows → `.msi`
- Linux → `.AppImage`

**External services (run locally by the user, not "cloud"):**

| Service | Type | Required? | Notes |
|---|---|---|---|
| **Ollama** (`bge-m3`) | Local process (HTTP) | **Yes** (embeddings) | User installs + `ollama pull bge-m3`. Health check on boot. |
| **GLM-5.1** | Remote API (OpenAI-compatible) | **No** (only if remote LLM) | Only potential cost point; key in local config, **never** in the binary. |

**Code signing (OQ-3):** **unsigned** builds initially (Apple notarization /
Windows cert = post-v1).

---

## 4. Testing framework

> ⚠️ **GAP — the PRD does not pin a testing framework.** This is a **pending
> decision** to confirm during Inception / Requirements Analysis.
> Recommendation below.

**Recommended (to be confirmed):**

| Layer | Tool | Why |
|---|---|---|
| **Unit / Integration** | **Vitest** | Same pipeline as Vite/`electron-vite`; fast; native JS API; runs in Node (main) and `jsdom`/`happy-dom` (renderer) |
| **E2E (Electron)** | **`@playwright/test`** (`_electron` API) | Official Electron support (spectron is dead); drives the packaged app |
| **Extra assertions** | `@testing-library/react` (renderer) | De-facto standard for React components |
| **HTTP mock** | `msw` | Mock Ollama (`bge-m3`) and GLM-5.1 with no real network |

**Target coverage (derived from PRD §14 / §13.5):**
- Unit: chunker (limits/overlap/sentence), prompt-builder (assembly order), refusal behavior (cosine > threshold), Result discriminated unions.
- Integration: end-to-end RAG pipeline with Ollama mocked; citation persistence (survives restart); cascade delete.
- Benchmark: 3 PDFs × 20 questions (citation accuracy ≥ 95%, Recall@5 ≥ 90%).

> **Action:** record the choice in `aidlc-docs/inception/requirements/requirements.md`
> when Requirements Analysis runs. Unless instructed otherwise, assume
> **Vitest + Playwright**.

---

## 5. BANNED libraries (with reason and alternative)

> Derived from the "locked" decisions (PRD §4 / `docs/STEERING.md`) and from
> "out of scope" (PRD §6). Swapping requires explicit stakeholder approval.

| Library / Technology | Reason (banned) | Allowed alternative |
|---|---|---|
| External / server vector DB (**Pinecone, Chroma, Weaviate, pgvector, Milvus**) | Must be embedded, local, zero-config, no extra process | **sqlite-vec** (via `better-sqlite3`) |
| Non-Electron shell (**Tauri, Wails, WebView2, CEF**) | Stack locked to Electron for bundled Chromium (consistent cross-OS rendering) | **Electron** |
| Remote embedding API (**OpenAI / Voyage / Cohere embeddings**) | Cost, latency, privacy; embeddings must be local and free | **Ollama + `bge-m3`** |
| **ONNX** in-process embeddings (v1) | Out of MVP scope (PRD §6); future iteration | **Ollama + `bge-m3`** |
| ORMs that hide sqlite-vec / **SQL string concatenation** | `sqlite-vec` is a loadable extension; better to use `better-sqlite3` directly; security (injection) | **`better-sqlite3`** (parameterized prepared statements) |
| UI libs other than Shadcn/ui (**MUI, Ant Design, Chakra UI**) | Stack locked; Shadcn primitives + Tailwind | **Shadcn/ui + Tailwind CSS** |
| State libs other than Zustand (**Redux, MobX, Recoil, Jotai**) | Stack locked; Zustand is minimal and TS-first | **Zustand** |
| Routing libs other than React Router v7 (**TanStack Router, wouter**) | Stack locked | **React Router v7** |
| Auth libs (**jsonwebtoken, Passport, Auth.js, NextAuth**) | No auth, single-user (out of scope) | None (no auth in MVP) |
| **`electron-updater`** / auto-update | Out of MVP scope (PRD §6) | Manual distribution via `electron-builder` |
| **OCR** (**tesseract.js**, etc.) | Out of MVP scope; text PDFs only | `pdfjs-dist` (text PDFs) only |
| **`electron-store`** / opaque config libs | Settings must be typed and auditable (JSON in `app.getPath('userData')`) | Own typed config loader + Zod |

---

## 6. Code examples (endpoint, function, test)

> Conventions (PRD §10–§11, `docs/STEERING.md`): **no exceptions crossing IPC** —
> services return `Result<T, E>` (discriminated union); the renderer validates
> everything with Zod; citation is always by **chunk ID** (page/snippet come
> from the DB).

### 6.1 Endpoint — typed IPC handler (Main process)

```ts
// electron/main/ipc/health.ts
import { ipcMain } from "electron";
import { checkOllamaHealth } from "../services/embedding-client.js";
import { getDb } from "../db/client.js";

type HealthResult =
  | { ok: true; value: { ollama: "up" | "down"; db: "up" | "down"; model: string } }
  | { ok: false; error: { code: string; message: string } };

export function registerHealthHandlers(): void {
  ipcMain.handle("check_health", async (): Promise<HealthResult> => {
    try {
      const db = getDb(); // throws if not initialized
      db.prepare("SELECT 1").get();
      const ollama = await checkOllamaHealth(); // { up: boolean, model: string }
      return {
        ok: true,
        value: {
          ollama: ollama.up ? "up" : "down",
          db: "up",
          model: ollama.model,
        },
      };
    } catch (err) {
      return {
        ok: false,
        error: { code: "HEALTH_CHECK_FAILED", message: String(err) },
      };
    }
  });
}
```

### 6.2 Function — chunker + LlmProvider interface

```ts
// electron/main/services/llm-provider.ts
export type ChatMessage = { role: "system" | "user" | "assistant"; content: string };
export type LlmError = { code: string; message: string };
export type Result<T> = { ok: true; value: T } | { ok: false; error: LlmError };

export interface LlmProvider {
  readonly name: string;
  chat(messages: ChatMessage[]): Promise<Result<string>>;
}

// electron/main/services/chunker.ts
export type Chunk = { page: number; chunkIndex: number; content: string };

const TARGET = 512; // chars
const OVERLAP = 64; // chars
const SENTENCE_END = /[.!?]/;

// Character sliding window with sentence-boundary preference (PRD §5).
export function chunkPageText(page: number, text: string): Chunk[] {
  const chunks: Chunk[] = [];
  if (text.length < 64) return []; // merge with next page (PRD rule)
  for (let start = 0, i = 0; start < text.length; start += TARGET - OVERLAP, i++) {
    let end = Math.min(start + TARGET, text.length);
    if (end < text.length) {
      const window = text.slice(end, end + 128);
      const boundary = window.search(SENTENCE_END);
      if (boundary >= 0) end += boundary + 1; // extend to end of sentence (+128 max)
    }
    chunks.push({ page, chunkIndex: i, content: text.slice(start, end) });
    if (end >= text.length) break;
  }
  return chunks;
}
```

### 6.3 Test — unit (Vitest, candidate framework to confirm)

```ts
// electron/main/services/chunker.test.ts
import { describe, expect, it } from "vitest";
import { chunkPageText } from "./chunker.js";

describe("chunkPageText", () => {
  it("returns empty for pages below the minimum (merge rule)", () => {
    expect(chunkPageText(1, "short")).toEqual([]);
  });

  it("produces overlapping chunks within target size", () => {
    const text = "a".repeat(1200);
    const chunks = chunkPageText(7, text);
    expect(chunks.length).toBeGreaterThan(1);
    for (const c of chunks) {
      expect(c.content.length).toBeLessThanOrEqual(512 + 128);
      expect(c.page).toBe(7);
    }
  });

  it("extends to the next sentence boundary when possible", () => {
    const text = "x".repeat(500) + ". Next sentence here.";
    const [first] = chunkPageText(1, text);
    expect(first.content.endsWith(".")).toBe(true);
  });
});
```

> The examples above are **not** final code — they are **reference contracts**
> to guide AI-DLC Code Generation. Real implementation lives in
> `electron/main/...` and `src/...`, per the layout in `docs/STEERING.md`.
