# AI-DLC Implementation Guide — Quaero

> Handoff guide for **any AI agent** (Claude Code, opencode, Cursor, etc.) tasked with
> implementing Quaero using **AI-DLC + Spec-Driven Development (SDD)**.
> Read this top to bottom **once**, then follow the loop in §4 for every unit of work.

---

## 0. TL;DR (the one rule)

**Never write code without an approved spec.**

```
read spec → confirm approval → implement against spec → validate acceptance criteria → commit → next Bolt
```

If a spec is `draft` / "awaiting approval", stop and get a human "approved" before coding.

---

## 1. What you're building

**Quaero** — cross-platform desktop app for chatting over PDFs via RAG, with
**verifiable citations** (document + page + snippet, resolved from the database,
**never hallucinated**). Portfolio MVP: runs 100% locally, ~zero cost, no deploy.

Source of intent: [`specs/PRD.md`](PRD.md). **Do not rewrite the PRD** without asking
the stakeholder — it's the locked Inception output.

---

## 2. Methodology: AI-DLC + SDD

AI-DLC phases: **Inception** (done — PRD locked) → **Construction** (current) → Operation.

Construction is organized into **Bolts** — short, self-contained delivery units. Each
Bolt has exactly one spec in `specs/<feature-name>.md`, and that spec is the
**single source of truth** for that Bolt.

**Hard rules:**

1. **Spec before code.** A Bolt's spec must exist and be **stakeholder-approved** before you implement.
2. **Stay in scope.** Implement only what the current Bolt's spec covers. Anything in "Out of scope" is forbidden, even if trivial.
3. **Validate before done.** Walk the spec's "Acceptance criteria" checklist. Do not declare a Bolt done until every box can be checked with evidence.
4. **One Bolt at a time.** Finish + validate Bolt N before starting Bolt N+1.

---

## 3. Where everything lives

| Path | What | Rule |
|---|---|---|
| `AGENTS.md` (=`CLAUDE.md` symlink) | Steering doc: stack, conventions, scope | Read first. Edit only `AGENTS.md`. |
| `specs/PRD.md` | Intent / Inception | Don't rewrite without asking. |
| `specs/<feature>.md` | One spec per Bolt — source of truth | Read the current one before coding. |
| `specs/AI-DLC-GUIDE.md` | This guide | — |
| `src-tauri/` | Rust backend implementation | Idiomatic Rust, `Result<T,E>` error handling. |
| `src/` | Vue 3 frontend implementation | TypeScript strict, Zod validation. |

---

## 4. The work loop (do this for every Bolt)

### Step 1 — Load context

Read `AGENTS.md`, then `specs/PRD.md` (skim relevant sections), then the current Bolt
spec in full.

### Step 2 — Confirm the Bolt + approval

Check `AGENTS.md` → "Current state" for which Bolt is active. Check the spec's
`Status:` line. If not `approved`, ask the stakeholder to approve before writing code.

### Step 3 — Plan atomic tasks

List the concrete steps from the spec's Contracts / Pipeline / Folder layout. If >5
steps or non-trivial dependencies, write them down (ordered, with what each depends on)
before touching files.

Example task breakdown for an "Upload + Processing" spec:

```
T1 — Scaffold Rust modules (document_service.rs, pdf_parser.rs, chunker.rs)
T2 — Implement PDF parser (pdf-extract, per-page extraction)
T3 — Implement chunker (fixed-size + overlap, preserve page)
T4 — Implement embedding client (HTTP to Ollama bge-m3)
T5 — Implement document_service orchestration (parse → chunk → embed → persist)
T6 — Create Tauri commands (upload_documents, list_documents)
T7 — Wire Vue frontend (DropZone, DocumentList)
T8 — Write tests (chunker unit test, upload integration test)
```

### Step 4 — Implement against the spec

For each task:

1. **Read the spec** — what contracts does this task fulfill?
2. **Read existing code** — what patterns are established? what can be reused?
3. **Implement** — build exactly what the spec describes.
4. **Verify** — `cargo build` / `cargo test` / `npm run build` / `npm run typecheck`.
5. **Commit** — Conventional Commits, atomic (one logical change per commit).

### Step 5 — Validate acceptance criteria

Go through the spec checklist line by line. For each, state how it's satisfied
(test name, command output, behavior). If any can't be checked, the Bolt is **not done**.

### Step 6 — Update state + next Bolt

Edit `AGENTS.md` → "Current state" to mark the Bolt done and the next one active.

---

## 5. Spec template

Every spec should follow this structure:

```markdown
# Spec — Bolt N: [Feature Name]

> Status: **draft** → awaiting stakeholder approval.

## Objective
(1–2 sentences: what this Bolt delivers)

## Scope
- What's included

### Out of scope
- What's explicitly excluded

## Technical decisions (locked)
| Topic | Decision | Reason |

## Contracts
- Tauri commands (name, params, return type)
- Data shapes (Rust structs / Vue types)
- Folder layout (which files created/modified)
- Pipeline / flow (step-by-step sequence)

## Acceptance criteria (Definition of Done)
- [ ] Each criterion is testable, binary (pass/fail)

## Traceability
- Origin: PRD §X
- Depends on: Bolt N-1
- Next: Bolt N+1
```

---

## 6. Conventions (non-negotiable)

### Rust

- Idiomatic Rust. `Result<T, E>` for fallible operations. No `unwrap()` in production code.
- Tauri commands return `Result<T, String>` or typed error structs.
- Error types implement `std::fmt::Display` for user-facing messages.
- Use `thiserror` for error types, `reqwest` for HTTP, `rusqlite` for SQLite.
- All DB operations use prepared statements (no string interpolation in SQL).

### TypeScript / Vue

- `strict` mode. No `any` without written justification.
- All Tauri `invoke()` responses validated through Zod schemas.
- Pinia stores for state. No scattered `ref()` in components for shared state.
- Shadcn-vue primitives for UI components. No custom modals/dropdowns from scratch.

### General

- Citation = always by chunk ID. Page/snippet come from the DB, never from LLM.
- Strict grounding: chat answers only from retrieved context; refuse outside it.
- Commits: Conventional Commits — `feat:`, `fix:`, `docs:`, `chore:`. Atomic.

---

## 7. Knowledge verification chain (when unsure, in order)

1. **Codebase** — existing code, patterns, conventions already in use.
2. **Project docs** — PRD, this guide, the Bolt spec, inline comments.
3. **Library docs** — official docs for Tauri, Vue, rusqlite, sqlite-vec, etc.
4. **Web** — official docs, reputable sources.
5. **Flag as uncertain** — "I'm not sure about X; here's my reasoning, verify."

**Never fabricate APIs/behaviors.** "I don't know" beats a confident wrong answer.

---

## 8. Commands

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

**First-time setup:** Install Rust toolchain → `npm install` → `ollama pull bge-m3`
→ `cargo tauri dev` → start the Bolt loop.

---

## 9. Definition of Done (per Bolt)

- [ ] Every "Acceptance criteria" box in the Bolt spec is checked with evidence.
- [ ] Required tests written and passing (`cargo test` green).
- [ ] No out-of-scope code added.
- [ ] Conventional commits, atomic.
- [ ] `AGENTS.md` "Current state" updated.
- [ ] Nothing committed that shouldn't be (`.env`, secrets, large binaries).
