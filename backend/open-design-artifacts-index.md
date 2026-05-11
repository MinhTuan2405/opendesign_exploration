# Open Design — Artifact Set

> Engineering documentation for the Open Design repository (v0.5.0, Apache-2.0).
> This index is the entry point. Read it first, then follow the reading order below.

---

## Recommended Reading Order

| # | Artifact | Reason to read |
|---|----------|----------------|
| Index (this file) | open-design-artifacts-index.md | Start here — orientation, reading order, quick architecture summary |
| 01 | open-design-folder-map.md | Understand the directory layout before diving into code |
| 02 | open-design-architecture-overview.mmd | Visual system topology — render in any Mermaid viewer |
| 03 | open-design-runtime-sequence.mmd | End-to-end generation flow from user prompt to rendered preview |
| — | docs/architecture.md | Official upstream architecture doc (three deployment topologies) |
| — | docs/spec.md | Product and design-rationale spec |
| — | docs/agent-adapters.md | Per-agent adapter protocol details |
| — | docs/skills-protocol.md | Skill YAML/Markdown schema and injection rules |
| — | docs/modes.md | Skill mode taxonomy (prototype / deck / template / etc.) |
| — | apps/daemon/src/agents.ts | Ground truth for all 16 agent definitions |
| — | apps/daemon/src/server.ts | Express route surface (~50 routes) |
| — | apps/daemon/src/prompts/system.ts | Prompt composition entry point |

---

## Artifact Descriptions

### open-design-artifacts-index.md (this file)

**Purpose:** Entry-point orientation document for the Open Design engineering artifact set.

**Target reader:** Backend engineers, LLM agents, and contributors approaching the codebase for the first time.

**Questions answered:**
- What does this artifact set cover?
- In what order should I read the documents?
- What are the most important source files?
- What parts of the system are uncertain or unverified?

**Files referenced:** All files in `docs/`, `apps/daemon/src/`, `apps/web/src/`, `packages/contracts/src/`.

---

### open-design-folder-map.md

**Purpose:** Annotated directory tree covering every top-level and second-level folder, with runtime roles, key files, and cross-component interactions.

**Target reader:** New backend engineers doing their first codebase orientation, or anyone who needs to find the right file quickly.

**Questions answered:**
- Where does X live in the repo?
- Which folders are active vs. removed vs. placeholder?
- How do `apps/`, `packages/`, `tools/`, `e2e/`, `skills/`, and `design-systems/` relate to each other?
- What is the runtime data directory layout (`.od/`)?

**Files referenced:** `pnpm-workspace.yaml`, `AGENTS.md`, `apps/daemon/src/db.ts`, `apps/daemon/src/cwd-aliases.ts`, `packages/sidecar-proto/src/`.

---

### open-design-architecture-overview.mmd

**Purpose:** Mermaid flowchart showing the full static architecture — components, data flows, and integration boundaries.

**Target reader:** Engineers wanting a bird's-eye view before reading code; useful as a reference when navigating the codebase.

**Questions answered:**
- How does the browser talk to the daemon?
- Where do agent CLIs fit?
- What is the BYOK proxy path?
- Where does SQLite fit relative to the filesystem?
- How does Electron desktop connect?
- What is the MCP server for?

**Files referenced:** `apps/daemon/src/server.ts`, `apps/daemon/src/agents.ts`, `apps/daemon/src/mcp.ts`, `apps/daemon/src/lint-artifact.ts`, `packages/sidecar-proto/src/`, `packages/contracts/src/`.

---

### open-design-runtime-sequence.mmd

**Purpose:** Mermaid sequence diagram showing the complete end-to-end flow for a single design-generation turn, from user prompt to preview render.

**Target reader:** Backend engineers debugging generation failures, contributors adding new agent adapters, or anyone tracing a request lifecycle.

**Questions answered:**
- What happens step-by-step when a user submits a prompt?
- When is the system prompt composed?
- When does the agent CLI get spawned?
- How do streaming events reach the browser?
- When does the lint checker run?
- Where does the artifact get persisted?

**Files referenced:** `apps/daemon/src/server.ts`, `apps/daemon/src/prompts/system.ts`, `apps/daemon/src/skills.ts`, `apps/daemon/src/cwd-aliases.ts`, `apps/daemon/src/claude-stream.ts`, `apps/daemon/src/lint-artifact.ts`, `apps/daemon/src/runs.ts`, `apps/daemon/src/db.ts`, `apps/web/src/artifacts/parser.ts`.

---

## How To Use This Artifact Set

1. **Orient first.** Read this index completely. The quick architecture summary below gives you a working mental model in under 2 minutes.
2. **Explore the folder map next.** `open-design-folder-map.md` tells you where every important file lives before you open your editor.
3. **Render the diagrams.** Open `open-design-architecture-overview.mmd` and `open-design-runtime-sequence.mmd` in any Mermaid-compatible viewer (VS Code Mermaid Preview, mermaid.live, GitHub, Obsidian). They are plain `.mmd` files containing raw Mermaid syntax.
4. **Cross-reference with source.** Each artifact cites the real source files. When in doubt, the source is authoritative; these documents are summaries.
5. **Check known unknowns.** The final section of this document lists areas that are uncertain or require verification against the live codebase.

---

## Quick Architecture Summary

Open Design is a local-first design tool that uses coding-agent CLIs as its "rendering engine." The user opens a Next.js web UI that communicates with a long-running local daemon (Express, Node 24) over HTTP and Server-Sent Events. When the user submits a prompt, the daemon composes a multi-layer system prompt (identity charter + design system tokens + skill workflow + deck framework), then spawns the selected agent CLI as a child process with the project folder as its working directory. The agent reads skill assets and a template, generates an HTML artifact, and writes it to disk; its stdout is parsed by a per-format stream parser into typed events that are forwarded to the browser via SSE. The browser's artifact parser extracts `<artifact>` tag content and renders it in a sandboxed iframe. An anti-slop linter runs grep-style checks against every saved artifact and feeds P0 findings back to the agent as a corrective system message. Project metadata is stored in SQLite (`app.sqlite`), while actual artifact files live on disk under `.od/projects/<id>/`. An optional Electron shell wraps the web UI for desktop use, discovering the local web URL through sidecar IPC over a Unix socket. A read-only MCP stdio server (`od mcp`) lets external coding agents pull project files without the export-import cycle.

---

## Most Important Files To Read First

| File | One-line reason |
|------|----------------|
| `apps/daemon/src/agents.ts` | Defines all 16 agent adapters: bin names, stream formats, `buildArgs()`, capability flags, model lists |
| `apps/daemon/src/server.ts` | The Express HTTP surface (~50 routes); the true API contract between web and daemon |
| `apps/daemon/src/prompts/system.ts` | `composeSystemPrompt()` — the 4-layer prompt assembly entry point |
| `apps/daemon/src/db.ts` | SQLite schema (7 tables) with migration logic; source of truth for persistence model |
| `apps/daemon/src/lint-artifact.ts` | Anti-slop linter; documents the P0/P1/P2 severity system and what patterns are flagged |
| `apps/daemon/src/runs.ts` | In-memory SSE run registry; explains how streaming events are buffered and replayed |
| `apps/daemon/src/cwd-aliases.ts` | Skill staging logic (why `.od-skills/<id>/` exists instead of symlinks) |
| `apps/daemon/src/mcp.ts` | The 8 read-only MCP tools; shows what external agents can query |
| `apps/web/src/artifacts/parser.ts` | Streaming `<artifact>` tag parser; the client-side counterpart to agent stdout |
| `packages/contracts/src/` | Shared TypeScript types (AgentEvent, SSE events, artifact shapes); the web/daemon contract |

---

## Known Unknowns

The following areas are uncertain or were not fully verified against live source during this documentation pass. They are marked `[Uncertain]` in the diagrams where applicable.

| Area | What is uncertain |
|------|-------------------|
| Export pipeline (PDF/PPTX/ZIP) | The exact route and implementation files for PPTX and ZIP export were not traced in detail; the capability is confirmed to exist from `docs/architecture.md` and referenced in the spec, but the specific modules handling PDF stitching and PPTX conversion were not read. |
| Critique orchestrator | `apps/daemon/src/critique/` is a subdirectory referenced in `server.ts` imports (`runOrchestrator`, `createRunRegistry`, `handleCritiqueInterrupt`) but not fully explored. Its relationship to the main run registry and lint checker is `[Uncertain]`. |
| `od mcp` resource endpoints | `mcp.ts` registers both tools and resources; the `ListResourcesRequestSchema` / `ReadResourceRequestSchema` handlers were not fully read. |
| Research subsystem | `apps/daemon/src/research/index.ts` is imported in `server.ts` as `searchResearch`; the implementation was not explored. |
| Orbit service | `apps/daemon/src/orbit.ts` (`OrbitService`) provides activity summaries (GitHub, Gmail, Linear, Notion integrations per skills); internals not read in detail. |
| Community pets sync | `apps/daemon/src/community-pets-sync.ts` is referenced in server startup; purpose and sync target not verified. |
| Windows-specific spawn paths | `createCommandInvocation` from `@open-design/platform` handles Windows `.cmd` shim detection; the exact branching logic was not traced. |
| BYOK proxy SSRF blocking | The research summary states the proxy blocks SSRF; the actual implementation in `server.ts` was not read to verify the exact middleware. |
| Skill count | The glob returned more than 100 skill directories; the "31 skills" figure in the research summary appears to be outdated or refers to a distinct subset. Treat the exact count as `[Uncertain]`. |
| Design system count | Similarly, the "72 design systems" figure may be a lower bound; the glob returned well over 100 DESIGN.md files. |
| `packages/sidecar-proto` IPC message schema | The proto constants were referenced but the full IPC message schema was not read. |
