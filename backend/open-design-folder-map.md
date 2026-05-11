# Open Design Folder Map

> Annotated directory tree for the Open Design monorepo. All paths are relative to
> the repository root. "Runtime data" paths (`.od/`, `.tmp/`) are local-only and
> gitignored; they are documented here for orientation but not present in the repo.

---

## Executive Summary

The repository is a pnpm monorepo with four workspace roots: `apps/`, `packages/`,
`tools/`, and `e2e/`. Three first-class apps (`web`, `daemon`, `desktop`) are
complemented by a thin packaged-runtime entry (`packaged`). Supporting packages
handle contracts (`contracts`), sidecar IPC protocol (`sidecar-proto`), generic
sidecar runtime (`sidecar`), and OS process primitives (`platform`). Content
directories (`skills/`, `design-systems/`, `craft/`) live at the repo root and are
read at runtime by the daemon. All local runtime data flows into `.od/` (daemon
state) and `.tmp/` (tools-dev lifecycle state); both are gitignored.

---

## Annotated Top-Level Tree

```
open-design/                         # Repo root (pnpm workspace)
├── apps/                            # Runnable applications
│   ├── daemon/                      # Node 24 local daemon + `od` CLI binary
│   ├── web/                         # Next.js 16 App Router + React 18 frontend
│   ├── desktop/                     # Electron shell (wraps web UI)
│   └── packaged/                    # Thin packaged-runtime Electron entry
│
├── packages/                        # Shared libraries (no runnable entry points)
│   ├── contracts/                   # Pure TS web/daemon API contract types
│   ├── sidecar-proto/               # Open Design IPC protocol constants + schema
│   ├── sidecar/                     # Generic sidecar bootstrap + IPC transport
│   └── platform/                    # OS process stamp, command parsing, PATH utils
│
├── tools/                           # Developer lifecycle CLIs
│   ├── dev/                         # tools-dev: start/stop/status/logs/inspect
│   └── pack/                        # tools-pack: build/install/cleanup for all platforms
│
├── e2e/                             # Playwright end-to-end and UI automation tests
│
├── skills/                          # 100+ skill directories, each with SKILL.md
├── design-systems/                  # 100+ design system directories, each with DESIGN.md
├── craft/                           # Brand-agnostic craft rules (anti-ai-slop.md, etc.)
│
├── docs/                            # Architecture and protocol documentation
│   ├── architecture.md              # System topology, component diagram, data flow
│   ├── spec.md                      # Product spec and design rationale
│   ├── agent-adapters.md            # Per-agent adapter protocol
│   ├── skills-protocol.md           # Skill YAML/MD schema and injection rules
│   ├── modes.md                     # Skill mode taxonomy
│   ├── roadmap.md                   # Public roadmap
│   └── references.md                # External references
│
├── specs/                           # Internal maintainability specs
│   └── current/
│       └── maintainability-roadmap.md
│
├── scripts/                         # Repo-level scripts (guard.ts, etc.)
│
├── prompt/                          # [Uncertain] Additional prompt content (untracked)
│
├── AGENTS.md                        # Root agent-guidance file (required reading for agents)
├── CLAUDE.md                        # Claude Code project instructions (imports AGENTS.md)
├── CONTRIBUTING.md                  # Contribution guide
├── QUICKSTART.md                    # Five-minute setup guide
├── README.md                        # Product README
├── pnpm-workspace.yaml              # Workspace package roots
├── package.json                     # Root manifest (pnpm version pin, guard/typecheck scripts)
├── tsconfig.json                    # Root TypeScript config
│
├── .od/                             # [RUNTIME, GITIGNORED] Daemon data directory
│   ├── app.sqlite                   # SQLite database (7 tables)
│   ├── projects/                    # Per-project working directories
│   │   └── <uuid>/                  # Project folder (agent cwd)
│   │       ├── .od-skills/          # Staged skill copies (per-turn, per-project)
│   │       ├── *.html               # Generated artifact files
│   │       └── references/          # Skill reference files (staged)
│   ├── artifacts/                   # Saved render outputs
│   └── media-config.json            # API credentials (BYOK keys)
│
└── .tmp/                            # [RUNTIME, GITIGNORED] tools-dev lifecycle data
    └── tools-dev/
        └── <namespace>/             # Per-namespace sidecar runtime files
```

---

## Folder Classification Table

| Path | Category | Purpose | Key files | Runtime role | Notes |
|------|----------|---------|-----------|-------------|-------|
| `apps/daemon/` | Application | Local privileged daemon and `od` CLI | `src/server.ts`, `src/agents.ts`, `src/db.ts` | Long-running process; spawns agent CLIs, serves HTTP/SSE API | Owns `.od/` data dir |
| `apps/web/` | Application | Next.js 16 App Router frontend | `src/app/`, `src/artifacts/parser.ts` | Served by Next.js dev server or Vercel; talks to daemon over HTTP+SSE | Do not import daemon `src/` directly |
| `apps/desktop/` | Application | Electron shell | `src/main.ts` | Wraps web URL discovered via sidecar IPC | Queries runtime status through IPC, not hard-coded ports |
| `apps/packaged/` | Application | Packaged Electron runtime entry | Sidecar start glue, `od://` entry | Starts packaged sidecars; owns `od://` protocol handler | Thin — no business logic |
| `packages/contracts/` | Library | Shared API types (TS-pure) | `src/` (AgentEvent, SSE events, artifact shapes) | Imported by both web and daemon | Must stay free of Node/browser/Next.js APIs |
| `packages/sidecar-proto/` | Library | IPC protocol constants + stamp schema | `src/` (app/mode/namespace constants) | Imported by sidecar, daemon, desktop | Source of truth for IPC message shape |
| `packages/sidecar/` | Library | Generic sidecar bootstrap + IPC transport | `src/` | Runtime: resolves paths, manages Unix sockets | No OD business logic |
| `packages/platform/` | Library | OS process stamp serialization + PATH utils | `src/` (createCommandInvocation, wellKnownUserToolchainBins) | Imported by daemon agents.ts for cross-platform spawn | Handles Windows `.cmd` shim detection |
| `tools/dev/` | CLI Tool | Local dev lifecycle control plane | `src/` | `pnpm tools-dev` — starts daemon + web + desktop, manages namespaces | Only valid dev entry point |
| `tools/pack/` | CLI Tool | Packaged build/release surface | `src/` | `pnpm tools-pack` — builds NSIS/AppImage/DMG, mac beta artifacts | Calls package primitives; no hand-built stamps |
| `e2e/` | Tests | Playwright end-to-end and UI automation | `tests/`, `ui/` | Run separately; isolated from app packages | Cross-app boundary checks live here |
| `skills/` | Content | 100+ skill directories with SKILL.md | `<name>/SKILL.md`, `<name>/assets/template.html`, `<name>/references/` | Scanned by daemon at GET /api/skills; skill bodies injected into prompts | Staged to `.od-skills/<id>/` per agent turn |
| `design-systems/` | Content | 100+ brand DESIGN.md files | `<name>/DESIGN.md` | Read by daemon; body injected as design token layer in system prompt | 9-section schema: palette, typography, spacing, etc. |
| `craft/` | Content | Brand-agnostic craft rules | `anti-ai-slop.md`, etc. | Loaded by `loadCraftSections()`; injected between DESIGN.md and skill body | Skills opt-in via `od.craft.requires` frontmatter |
| `docs/` | Documentation | Architecture and protocol docs | `architecture.md`, `spec.md`, `agent-adapters.md`, `skills-protocol.md` | Human-read reference; not loaded at runtime | Canonical design rationale |
| `.od/` | Runtime data | Daemon data root (gitignored) | `app.sqlite`, `projects/`, `media-config.json` | Written/read by daemon process | Relocatable via `OD_DATA_DIR` env var |
| `.tmp/` | Runtime data | tools-dev lifecycle state (gitignored) | `tools-dev/<ns>/` | Written by tools-dev; logs, PID files | Namespace-scoped |

---

## Detailed Folder Breakdown

### `apps/daemon/`

**Category:** Application — local privileged Node 24 process.

**Purpose:** The brain of the system. Owns all agent spawning, skill/design-system loading, prompt composition, SSE streaming, SQLite persistence, artifact linting, MCP server, and the `od` CLI binary.

**Key files:**

| File | Role |
|------|------|
| `src/server.ts` | Express HTTP server, ~50 routes. Wires all subsystems together. |
| `src/agents.ts` | `AGENT_DEFS` array (16 agents), `detectAgents()`, `probe()`, `buildArgs()`, capability flags. |
| `src/db.ts` | `openDatabase()`, SQLite schema (7 tables), WAL mode, forward-compatible migrations. |
| `src/runs.ts` | In-memory SSE run registry (`createChatRunService`). Buffers up to 2000 events per run; 30-min TTL. |
| `src/prompts/system.ts` | `composeSystemPrompt()` — 4-layer assembly entry point. |
| `src/prompts/official-system.ts` | `OFFICIAL_DESIGNER_PROMPT` — designer identity charter (base layer). |
| `src/prompts/discovery.ts` | `DISCOVERY_AND_PHILOSOPHY` — Turn-1 question form, direction picker, 5-dim critique. |
| `src/prompts/deck-framework.ts` | `DECK_FRAMEWORK_DIRECTIVE` — nav/counter/print JS contract for deck skills. |
| `src/prompts/directions.ts` | 5 design directions with OKLch palette specifications. |
| `src/prompts/media-contract.ts` | `MEDIA_GENERATION_CONTRACT` — image/video/audio generation rules. |
| `src/skills.ts` | `listSkills()`, `findSkillById()`, `resolveSkillId()`, SKILL_ID_ALIASES. |
| `src/design-systems.ts` | `listDesignSystems()`, `readDesignSystem()`. |
| `src/cwd-aliases.ts` | `stageActiveSkill()` — copies skill to `.od-skills/<id>/` before agent spawn. |
| `src/craft.ts` | `loadCraftSections()` — loads craft reference files by slug. |
| `src/lint-artifact.ts` | Anti-slop linter: P0/P1/P2 findings, purple/indigo/emoji/metric/lorem checks. |
| `src/mcp.ts` | Read-only MCP stdio server (`od mcp`); 8 tools proxying daemon HTTP API. |
| `src/acp.ts` | ACP JSON-RPC session management for Devin/Hermes/Kimi/Kiro/Kilo/Mistral Vibe. |
| `src/pi-rpc.ts` | Pi stdio JSON-RPC adapter. |
| `src/claude-stream.ts` | JSONL parser for Claude Code `--output-format stream-json`. |
| `src/qoder-stream.ts` | JSONL parser for Qoder CLI `--output-format stream-json`. |
| `src/copilot-stream.ts` | Stream parser for GitHub Copilot CLI. |
| `src/json-event-stream.ts` | Unified parser for Codex/Gemini/OpenCode/Cursor Agent JSON event streams. |
| `src/deploy.ts` | Deployment provider integration (Vercel, etc.) for GET /api/deploy routes. |
| `src/transcript-export.ts` | HTML/PDF/PPTX/ZIP/Markdown export pipeline. |
| `src/media.ts` | `generateMedia()` — image/video/audio generation via BYOK providers. |
| `src/research/` | Research subsystem (`searchResearch`). [Uncertain — not fully traced] |
| `src/critique/` | Critique orchestrator (`runOrchestrator`, `createRunRegistry`, interrupt handler). [Uncertain — not fully traced] |
| `src/orbit.ts` | `OrbitService` — activity summaries pulling from GitHub/Gmail/Linear/Notion. |

**Runtime role:** Spawned once per session by `pnpm tools-dev`. Listens on a configurable port (default 7456 via `--daemon-port`). Writes all runtime data to `.od/` (or `OD_DATA_DIR`).

**Interactions:** Web app (HTTP+SSE), agent CLIs (child_process.spawn), SQLite, filesystem (`.od/projects/<id>/`), sidecar IPC (via `@open-design/sidecar-proto`), MCP clients (stdio).

---

### `apps/web/`

**Category:** Application — Next.js 16 App Router frontend.

**Purpose:** The browser UI. Provides the chat pane, artifact tree, preview iframe, comment/slider overlay, settings, deployment controls, and design system preview.

**Key files:**

| File | Role |
|------|------|
| `src/artifacts/parser.ts` | Streaming `<artifact>` tag parser. Emits `artifact:start`, `artifact:chunk`, `artifact:end`, `text` events. |
| `src/app/` | Next.js App Router pages and layouts. |

**Runtime role:** Dev: served by Next.js dev server on `--web-port` (default 3000, exported as `OD_WEB_PORT`). Production: deployed to Vercel or served statically.

**Interactions:** Daemon HTTP+SSE (`OD_PORT`), BYOK proxy routes (direct to Anthropic/OpenAI/Azure/Google APIs for no-daemon mode). Must not import daemon `src/` directly.

**Notes:** `apps/nextjs` has been permanently removed; do not restore it.

---

### `apps/desktop/`

**Category:** Application — Electron shell.

**Purpose:** Wraps the web UI in a desktop window. Discovers the running web URL through sidecar IPC (not hard-coded ports).

**Runtime role:** Optional. Launched by `pnpm tools-dev` (full stack) or `pnpm tools-dev inspect desktop status`.

**Interactions:** Sidecar IPC (`/tmp/open-design/ipc/<ns>/desktop.sock`), web UI (embedded WebContentsView).

---

### `apps/packaged/`

**Category:** Application — thin packaged runtime entry.

**Purpose:** Starts packaged sidecar processes and handles the `od://` URL protocol for installed (packaged) builds. Contains no business logic.

**Runtime role:** Replaces `apps/desktop` entry in packaged builds. Reads sidecar process stamps; does not hard-build `--od-stamp-*` arguments.

---

### `packages/contracts/`

**Category:** Library — pure TypeScript.

**Purpose:** Single source of truth for all shared API shapes: `AgentEvent`, SSE event unions, artifact DTO shapes, error shapes, task shapes, critique config, example payloads.

**Key constraint:** Must remain free of Next.js, Express, Node filesystem/process APIs, browser APIs, SQLite, daemon internals, and sidecar control-plane dependencies.

**Runtime role:** Compile-time only (imported by both web and daemon for type safety).

---

### `packages/sidecar-proto/`

**Category:** Library — IPC protocol constants.

**Purpose:** App/mode/source constants, namespace validation, sidecar stamp field definitions (`app`, `mode`, `namespace`, `ipc`, `source`), IPC message schema, status shapes, and error semantics.

**Runtime role:** Imported by daemon, sidecar, desktop, and tools-dev. The five-field stamp contract is enforced here.

---

### `packages/sidecar/`

**Category:** Library — generic sidecar runtime.

**Purpose:** Bootstrap, IPC transport (Unix socket), path/runtime resolution, launch env, and JSON runtime files. Contains no Open Design–specific business logic.

---

### `packages/platform/`

**Category:** Library — OS process primitives.

**Purpose:** `createCommandInvocation()` (cross-platform spawn normalization including Windows `.cmd` shim detection), `wellKnownUserToolchainBins` (toolchain PATH expansion), process stamp serialization, command parsing, and process matching/search.

**Runtime role:** Imported by daemon `agents.ts` for every agent spawn.

---

### `tools/dev/`

**Category:** CLI Tool.

**Purpose:** The only valid local development lifecycle entry point. Commands: `start`, `run`, `stop`, `status`, `logs`, `inspect`, `check`.

**Key constraint:** Do not bypass with `pnpm dev`, `pnpm daemon`, `pnpm start`, or similar root aliases — they are intentionally absent.

---

### `tools/pack/`

**Category:** CLI Tool.

**Purpose:** Packaged build/release surface for all platforms. Commands: `mac build`, `win build`, `linux build`, `install`, `cleanup`. Calls package primitives; does not hand-build stamp arguments.

---

### `e2e/`

**Category:** Tests.

**Purpose:** Playwright end-to-end smoke tests (`tests/`) and UI automation (`ui/`). Owns cross-app boundary checks that require observing more than one app/package. Uses `OD_DATA_DIR` for test isolation.

**Notes:** Do not put Playwright automation in app `tests/` directories. Read `e2e/AGENTS.md` before editing tests.

---

### `skills/`

**Category:** Content — skill definitions.

**Purpose:** Each subdirectory is a named skill with a `SKILL.md` (YAML frontmatter + Markdown body), optional `assets/template.html` seed, and optional `references/` directory (layouts.md, checklist.md, etc.).

**Schema fields (frontmatter):** `name`, `description`, `triggers`, `od.mode`, `od.surface`, `od.craft.requires`, `od.platform`, `od.scenario`, `od.preview.type`, `od.design_system.requires`, `od.default_for`, `od.upstream`.

**Runtime role:** Scanned by daemon `listSkills()` on every `GET /api/skills`. Active skill body injected into system prompt layer 3. Skill folder staged to `.od/projects/<id>/.od-skills/<name>/` before agent spawn.

**Notes:** Skill count exceeds 100 directories in the live repo. The "31 skills" figure in some references appears outdated. `SKILL_ID_ALIASES` in `skills.ts` handles deprecated folder name renames.

---

### `design-systems/`

**Category:** Content — brand design systems.

**Purpose:** Each subdirectory contains a `DESIGN.md` defining brand tokens in a 9-section schema: palette, typography, spacing, component style, motion, illustration, iconography, photography/imagery, voice/tone.

**Runtime role:** Read by daemon `readDesignSystem()`. Active design system body injected into system prompt layer 2 (after OFFICIAL_DESIGNER_PROMPT, before skill body).

**Notes:** Design system count exceeds 100 in the live repo. The "72" figure in some references appears outdated.

---

### `craft/`

**Category:** Content — brand-agnostic craft rules.

**Purpose:** Reusable craft reference sections that any skill can opt into via `od.craft.requires` frontmatter (e.g. `anti-ai-slop`). Loaded by `loadCraftSections()` and injected between DESIGN.md and SKILL.md body.

---

### `docs/`

**Category:** Documentation.

**Purpose:** Canonical architecture and protocol documentation. Not loaded at runtime; intended for human engineers and agents entering the codebase.

**Key files:**

| File | Content |
|------|---------|
| `architecture.md` | Three deployment topologies, component diagram, agent/skill flow |
| `spec.md` | Product and design rationale |
| `agent-adapters.md` | Per-agent stream format details and adapter protocol |
| `skills-protocol.md` | Skill YAML schema, injection strategies, preflight rules |
| `modes.md` | Skill mode taxonomy (prototype, deck, template, design-system, image, video, audio) |
| `roadmap.md` | Feature roadmap |
| `references.md` | External references and inspirations |

---

## Backend Engineer Reading Notes

1. **Start with `apps/daemon/src/server.ts`** to understand the full API surface. It imports every subsystem and wires them into routes — reading its imports gives you a map of all active modules.

2. **Agent definitions are in `apps/daemon/src/agents.ts`.** Each entry in `AGENT_DEFS` is self-contained: bin name, fallback bins, version args, stream format, `buildArgs()`, capability flags, model list. Adding a new agent means adding one entry here plus a stream parser (or reusing an existing format).

3. **Prompt composition is layered.** `composeSystemPrompt()` in `apps/daemon/src/prompts/system.ts` is the single entry point. The four layers are: `OFFICIAL_DESIGNER_PROMPT` (identity), `DISCOVERY_AND_PHILOSOPHY` (planning), DESIGN.md body (tokens), SKILL.md body (workflow). The deck framework directive pins last for deck-mode projects.

4. **Skill injection is agent-specific.** Claude Code gets `--add-dir` pointing at `.od-skills/<id>/`. Cursor Agent gets `.cursorrules` written into the project dir. All other agents get the skill text inlined into the system prompt.

5. **All runtime state is in `.od/`.** SQLite tracks metadata; the filesystem under `.od/projects/<id>/` is the agent's working directory and owns the actual HTML artifacts. `OD_DATA_DIR` relocates everything.

6. **The lint checker (`lint-artifact.ts`) is deterministic and greppy.** It does not parse HTML. P0 findings (purple/indigo gradients, slop emoji, invented metrics) are fed back to the agent as a system message so it self-corrects on the next turn. P1/P2 are advisory badges only.

7. **The MCP server (`mcp.ts`) is read-only and stateless.** It proxies all tool calls to the running daemon HTTP API. Useful for external coding agents in other repos that need to pull OD project files.

8. **`packages/contracts/` is the API contract.** Never add Node.js or browser APIs to it. Any new HTTP endpoint shape belongs here first, before wiring in web and daemon.
