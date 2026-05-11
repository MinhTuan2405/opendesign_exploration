# Open Design Backend Deep Dive

## Executive Summary

Open Design v0.5.0 is a local-first AI design platform built around a privileged Node 24 daemon (`apps/daemon`) that brokers all AI agent interactions, file operations, and streaming through a single Express HTTP server. The daemon orchestrates up to 16 distinct AI CLI agents by spawning them as child processes, composing multi-layer system prompts, and translating each agent's proprietary stdout stream format into a unified SSE event vocabulary consumed by the Next.js 16 frontend (`apps/web`). SQLite (via `better-sqlite3`) provides local persistence for projects, conversations, messages, and deployment metadata, while agent artifacts and project files live under `.od/` on the filesystem. All API traffic is locked to 127.0.0.1 by default; SSRF blocking, origin validation, and read-only MCP server design collectively ensure no agent action can exfiltrate data or reach LAN hosts.

---

## Backend-Relevant Folder Map

| Path | Role |
|---|---|
| `apps/daemon/src/server.ts` | Express app entry, all ~70 API routes, SSE helper, spawn orchestration |
| `apps/daemon/src/agents.ts` | AGENT_DEFS array, detectAgents(), resolveAgentExecutable(), buildArgs(), spawn env |
| `apps/daemon/src/runs.ts` | In-memory run registry, SSE fan-out, cancel/wait/stream primitives |
| `apps/daemon/src/claude-stream.ts` | Parses Claude Code `stream-json` JSONL to typed events |
| `apps/daemon/src/json-event-stream.ts` | Parses Codex/Gemini/OpenCode/Cursor `json-event-stream` JSONL |
| `apps/daemon/src/acp.ts` | ACP JSON-RPC session lifecycle (Devin/Hermes/Kimi/Kiro/Kilo/Vibe) |
| `apps/daemon/src/pi-rpc.ts` | Pi `--mode rpc` custom JSON-RPC session |
| `apps/daemon/src/copilot-stream.ts` | GitHub Copilot CLI JSONL stream parser |
| `apps/daemon/src/qoder-stream.ts` | Qoder CLI `stream-json` JSONL parser |
| `apps/daemon/src/skills.ts` | Skill registry, listSkills(), findSkillById(), withSkillRootPreamble() |
| `apps/daemon/src/cwd-aliases.ts` | stageActiveSkill() — copies skill to `.od-skills/<id>/` per project |
| `apps/daemon/src/prompts/system.ts` | composeSystemPrompt() — 10-layer system prompt composer |
| `apps/daemon/src/prompts/discovery.ts` | DISCOVERY_AND_PHILOSOPHY — mandatory first prompt layer |
| `apps/daemon/src/prompts/official-system.ts` | OFFICIAL_DESIGNER_PROMPT — base identity charter |
| `apps/daemon/src/prompts/deck-framework.ts` | DECK_FRAMEWORK_DIRECTIVE — nav/counter/PDF print contract |
| `apps/daemon/src/prompts/media-contract.ts` | MEDIA_GENERATION_CONTRACT — image/video/audio output rules |
| `apps/daemon/src/db.ts` | SQLite schema, all CRUD helpers |
| `apps/daemon/src/lint-artifact.ts` | Anti-slop HTML linter, P0/P1/P2 findings |
| `apps/daemon/src/critique/orchestrator.ts` | Critique Theater multi-round orchestrator |
| `apps/daemon/src/critique/config.ts` | loadCritiqueConfigFromEnv() |
| `apps/daemon/src/mcp.ts` | `od mcp` — stdio read-only MCP server (8 tools, 3 resources) |
| `apps/daemon/src/origin-validation.ts` | SSRF blocking, isPrivateIpv4(), validateLocalDaemonRequest() |
| `apps/daemon/src/sidecar/server.ts` | Daemon sidecar entry — wraps startServer() for sidecar IPC |
| `apps/daemon/src/deploy.ts` | Cloudflare Pages and Vercel deploy integration |
| `apps/daemon/src/media.ts` | Image/video/audio generation via external provider APIs |
| `apps/daemon/src/projects.ts` | Per-project filesystem helpers (listFiles, readProjectFile, etc.) |
| `apps/daemon/src/craft.ts` | loadCraftSections() — universal craft references |
| `apps/daemon/src/tool-tokens.ts` | toolTokenRegistry — scoped per-run API tokens for agent tool calls |
| `apps/daemon/src/connectors/` | Third-party connector integration (Composio) |
| `apps/daemon/src/live-artifacts/` | Live artifact store, refresh loop, MCP server |
| `skills/` | SKILL.md files (one folder per skill) |
| `design-systems/` | DESIGN.md files (one folder per design system) |
| `craft/` | Universal brand-agnostic craft rules |
| `.od/` | Runtime data root: app.sqlite, projects/, artifacts/ |

---

## API Routes

All routes are registered in `apps/daemon/src/server.ts` inside `startServer()`. Auth is enforced per-route by `requireLocalDaemonRequest` (peer-addr + Host + Origin check) or `authorizeToolRequest` (tool token validation).

### System / Health

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/health` | none | — | `{ ok: true, version }` |
| GET | `/api/version` | none | — | `{ version: { version, ... } }` |

### Active Context (MCP fallback)

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| POST | `/api/active` | local same-origin | `{ projectId, fileName?, active? }` | `{ active, projectId, fileName, ts }` |
| GET | `/api/active` | local same-origin | — | `{ active, projectId?, fileName? }` |

### MCP Install Info

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/mcp/install-info` | none | — | MCP install payload for claude_desktop_config.json |

### Projects

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/projects` | none | — | `{ projects: [...] }` |
| POST | `/api/projects` | none | `{ name, skillId?, designSystemId?, metadata? }` | 201 `{ project }` |
| GET | `/api/projects/:id` | none | — | `{ project }` |
| PATCH | `/api/projects/:id` | none | `{ name?, skillId?, designSystemId?, metadata? }` | `{ project }` |
| DELETE | `/api/projects/:id` | none | — | 204 |
| GET | `/api/projects/:id/events` | none | SSE long-poll | SSE: `file_change`, `live_artifact`, `live_artifact_refresh` |
| POST | `/api/import/folder` | local | multipart `projectZip` (200MB) | `{ project }` |

### Conversations

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/projects/:id/conversations` | none | — | `{ conversations: [...] }` |
| POST | `/api/projects/:id/conversations` | none | `{ title? }` | 201 `{ conversation }` |
| PATCH | `/api/projects/:id/conversations/:cid` | none | `{ title }` | `{ conversation }` |
| DELETE | `/api/projects/:id/conversations/:cid` | none | — | 204 |
| GET | `/api/projects/:id/conversations/:cid/messages` | none | — | `{ messages: [...] }` |
| PUT | `/api/projects/:id/conversations/:cid/messages/:mid` | none | message body | `{ message }` |

### Preview Comments

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/projects/:id/conversations/:cid/comments` | none | — | `{ comments: [...] }` |
| POST | `/api/projects/:id/conversations/:cid/comments` | none | comment body | `{ comment }` |

### Tabs

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/projects/:id/tabs` | none | — | `{ tabs: [...] }` |
| PUT | `/api/projects/:id/tabs` | none | `{ tabs: [...] }` | `{ tabs }` |

### Templates

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/templates` | none | — | `{ templates: [...] }` |
| GET | `/api/templates/:id` | none | — | `{ template }` |
| POST | `/api/templates` | none | `{ name, description?, projectId? }` | 201 `{ template }` |
| DELETE | `/api/templates/:id` | none | — | 204 |

### Agents

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/agents` | none | — | `{ agents: [...] }` (runs detectAgents()) |

### Skills

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/skills` | none | — | `{ skills: [...] }` |
| GET | `/api/skills/:id` | none | — | `{ skill }` |
| GET | `/api/skills/:id/example` | none | — | SSE: generates example artifact |
| GET | `/api/skills/:id/assets/*` | none | static path | skill asset file content |

### Codex Pets

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/codex-pets` | none | — | `{ pets: [...] }` |
| POST | `/api/codex-pets/sync` | none | — | sync result |
| GET | `/api/codex-pets/:id/spritesheet` | none | — | image bytes |

### Design Systems

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/design-systems` | none | — | `{ designSystems: [...] }` |
| GET | `/api/design-systems/:id` | none | — | `{ designSystem }` |
| GET | `/api/design-systems/:id/preview` | none | — | HTML preview |
| GET | `/api/design-systems/:id/showcase` | none | — | HTML showcase |

### Prompt Templates

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/prompt-templates` | none | — | `{ promptTemplates: [...] }` |
| GET | `/api/prompt-templates/:surface/:id` | none | — | `{ promptTemplate }` |

### Uploads / Artifacts

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| POST | `/api/upload` | none | multipart `images[]` (20MB, max 8) | `{ files: [...] }` |
| POST | `/api/artifacts/save` | none | `{ projectId, fileName, content }` | `{ ok, lintFindings? }` |
| POST | `/api/artifacts/lint` | none | `{ content }` | `{ findings: [...] }` |

### Live Artifacts

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/live-artifacts` | none | `?projectId` | `{ artifacts: [...] }` |
| GET | `/api/live-artifacts/:artifactId` | none | — | `{ artifact }` |
| GET | `/api/live-artifacts/:artifactId/preview` | local | — | rendered HTML preview |
| GET | `/api/live-artifacts/:artifactId/refreshes` | none | — | `{ entries: [...] }` |
| POST | `/api/live-artifacts/:artifactId/refresh` | local | — | SSE refresh stream |
| PATCH | `/api/live-artifacts/:artifactId` | none | patch fields | `{ artifact }` |
| DELETE | `/api/live-artifacts/:artifactId` | none | — | 204 |
| POST | `/api/tools/live-artifacts/create` | tool token | `{ projectId, title, code }` | `{ artifact }` |
| GET | `/api/tools/live-artifacts/list` | tool token | `?projectId` | `{ artifacts }` |
| POST | `/api/tools/live-artifacts/update` | tool token | `{ artifactId, ... }` | `{ artifact }` |
| POST | `/api/tools/live-artifacts/refresh` | tool token | `{ artifactId }` | `{ artifact }` |

### Deployment

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/deploy/config` | none | — | `{ config }` |
| PUT | `/api/deploy/config` | none | deploy config | `{ config }` |
| GET | `/api/deploy/cloudflare-pages/zones` | none | — | `{ zones: [...] }` |
| GET | `/api/projects/:id/deployments` | none | — | `{ deployments: [...] }` |
| POST | `/api/projects/:id/deploy` | none | `{ fileName, providerId, target?, ... }` | SSE deploy stream |
| POST | `/api/projects/:id/deploy/preflight` | none | `{ fileName, providerId }` | `{ preflight }` |

### Project Files

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/projects/:id/files` | none | — | `{ files: [...] }` |
| GET | `/api/projects/:id/search` | none | `?q` | `{ results: [...] }` |
| GET | `/api/projects/:id/archive` | none | — | ZIP download |
| POST | `/api/projects/:id/archive/batch` | none | `{ ids: [...] }` | ZIP download |
| GET | `/api/projects/:id/raw/*` | none | path | raw file bytes |
| DELETE | `/api/projects/:id/raw/*` | none | path | 204 |
| GET | `/api/projects/:id/files/:name/preview` | none | — | HTML preview with sandboxing |
| GET | `/api/projects/:id/files/*` | none | path, range | file content (range supported) |
| DELETE | `/api/projects/:id/files/:name` | none | — | 204 |

### Media

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/media/models` | none | — | `{ models: [...] }` |
| GET | `/api/media/config` | none | — | masked config (keys redacted) |
| PUT | `/api/media/config` | none | config object | saved config |
| POST | `/api/projects/:id/media/generate` | none | generation params | SSE generation stream |
| POST | `/api/media/tasks/:id/wait` | none | — | `{ task }` (polls until done) |
| GET | `/api/projects/:id/media/tasks` | none | — | `{ tasks: [...] }` |

### App Config / Orbit

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| GET | `/api/app-config` | none | — | `{ config }` |
| PUT | `/api/app-config` | none | config object | `{ config }` |
| GET | `/api/orbit/status` | none | — | `{ status }` |
| POST | `/api/orbit/run` | none | `{ prompt, skillId? }` | SSE orbit run |

### Utilities

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| POST | `/api/dialog/open-folder` | none | — | `{ path }` (native folder dialog) |
| POST | `/api/research/search` | none | `{ query, maxSources? }` | `{ results }` |
| POST | `/api/test/connection` | none | `{ mode: 'agent'|'provider', ... }` | `{ ok, ... }` |

### Runs (Agent Execution)

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| POST | `/api/runs` | none | run meta | 202 `{ runId }` |
| GET | `/api/runs` | none | `?projectId&conversationId&status` | `{ runs: [...] }` |
| GET | `/api/runs/:id` | none | — | `{ id, status, exitCode, ... }` |
| GET | `/api/runs/:id/events` | none | SSE long-poll (Last-Event-ID) | SSE: `agent`, `error`, `end` |
| POST | `/api/runs/:id/cancel` | none | — | `{ ok }` |
| POST | `/api/chat` | none | chat request body | SSE: immediate stream |

### Critique Theater

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| POST | `/api/projects/:projectId/critique/:runId/interrupt` | none | — | `{ ok }` |

### BYOK Proxy (SSE)

| Method | Path | Auth | Inputs | Outputs |
|---|---|---|---|---|
| POST | `/api/proxy/anthropic/stream` | none | `{ baseUrl, apiKey, model, systemPrompt, messages }` | SSE: `start`, `delta`, `end`, `error` |
| POST | `/api/proxy/openai/stream` | none | `{ baseUrl, apiKey, model, ... }` | SSE: `start`, `delta`, `end`, `error` |
| POST | `/api/proxy/azure/stream` | none | `{ baseUrl, apiKey, apiVersion, model, ... }` | SSE: `start`, `delta`, `end`, `error` |
| POST | `/api/proxy/google/stream` | none | `{ baseUrl, apiKey, model, ... }` | SSE: `start`, `delta`, `end`, `error` |

### Connectors (via `registerConnectorRoutes`)

Connector routes are registered in `apps/daemon/src/connectors/routes.ts` and mounted under `/api/connectors/`.

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/connectors/composio/config` | none | Read Composio config (public/masked) |
| PUT | `/api/connectors/composio/config` | local | Write Composio config |
| Various | `/api/connectors/*` | tool token | Connector tool calls (list, execute, auth) |

---

## Daemon Lifecycle

### Startup

1. `apps/daemon/src/sidecar/server.ts` calls `startServer({ port, returnServer: true })`.
2. `server.ts` runs `resolveDataDir()` — resolves `OD_DATA_DIR` (or falls back to `<projectRoot>/.od`), creates the directory, checks writability.
3. `migrateLegacyDataDirSync()` runs synchronously: if `OD_LEGACY_DATA_DIR` is set and `app.sqlite` is absent at the new path, copies the 0.3.x `.od/` payload.
4. `openDatabase()` opens `RUNTIME_DATA_DIR/app.sqlite` with `PRAGMA journal_mode = WAL; PRAGMA foreign_keys = ON;`, runs the `migrate()` DDL, then runs `migrateCritique()` for the critique tables.
5. Static directory constants are resolved: `SKILLS_DIR`, `DESIGN_SYSTEMS_DIR`, `CRAFT_DIR`, `FRAMES_DIR`, `PROMPT_TEMPLATES_DIR`, `BUNDLED_PETS_DIR` — all derived from `OD_RESOURCE_ROOT` if set, otherwise from `PROJECT_ROOT`.
6. `loadCritiqueConfigFromEnv()` loads and validates all `OD_CRITIQUE_*` env vars at startup; a bad value throws a `RangeError` before the server binds.
7. Express app is configured with `json()`, `urlencoded()`, CORS (allowed origins from `OD_ALLOWED_ORIGINS`), and `express.static(STATIC_DIR)`.
8. All routes are registered.
9. `app.listen(port, host)` binds; `host` defaults to `OD_BIND_HOST ?? '127.0.0.1'`.
10. The sidecar IPC server starts (`createJsonIpcServer`), answering `STATUS`/`SHUTDOWN`/`EVAL`/`SCREENSHOT`/`CONSOLE`/`CLICK` messages over a POSIX socket at `/tmp/open-design/ipc/<namespace>/daemon.sock`.

### Port Binding

`OD_BIND_HOST` controls the listen address (default `127.0.0.1`). The port is provided by the caller (`SIDECAR_ENV.DAEMON_PORT`). A port of `0` lets the OS assign a free port.

### PATH Scan

At `GET /api/agents`, `detectAgents(configuredEnvByAgent)` runs `probe()` concurrently for all 16 AGENT_DEFS. `probe()` calls `resolveAgentExecutable(def, configuredEnv)` which checks `AGENT_BIN_ENV_KEYS` for a per-agent binary override env var, then walks `resolvePathDirs()` — `process.env.PATH` directories plus `userToolchainDirs()` (Node version managers, `~/.local/bin`, Homebrew, etc., via `@open-design/platform`). Results are cached for 5 seconds (`TOOLCHAIN_DIR_CACHE_TTL_MS`).

### Graceful Shutdown

The parent process monitor (via `SIDECAR_ENV.TOOLS_DEV_PARENT_PID`) polls every 1 second. When the parent PID is no longer alive, `stop()` is called: the HTTP server is closed, the IPC server stops, and `process.exit(0)` is called.

---

## Agent Detection

### `detectAgents(configuredEnvByAgent)`

Runs `probe(def, configuredEnvByAgent[def.id])` in parallel for all 16 AGENT_DEFS via `Promise.all`. Returns an array of agent info objects (without function fields) annotated with `available`, `version`, `path`, and `models`. Also updates the in-process `knownModels` validation cache so `/api/chat` can immediately accept any model the user just picked.

### AGENT_DEFS Structure

Each entry is an object with these fields:

| Field | Type | Purpose |
|---|---|---|
| `id` | string | Stable identifier used throughout the codebase |
| `name` | string | Human-readable display name |
| `bin` | string | Primary binary name to search on PATH |
| `fallbackBins` | string[] | Alternate binary names tried if `bin` not found (e.g. `openclaude` for `claude`) |
| `versionArgs` | string[] | Args to pass for version probe (e.g. `['--version']`) |
| `helpArgs` | string[] | Args to pass for capability flag detection |
| `capabilityFlags` | object | Map of `--flag-string` → `capabilityKey`; set true if detected in `--help` output |
| `fallbackModels` | array | Static model hint list used when dynamic listing unavailable |
| `listModels` | object | `{ args, parse, timeoutMs }` for dynamic model listing via CLI |
| `fetchModels` | function | Async model fetcher (used by ACP agents via `detectAcpModels`) |
| `reasoningOptions` | array | Reasoning effort presets (Codex, Pi only) |
| `buildArgs` | function | `(prompt, imagePaths, extraAllowedDirs, options, runtimeContext) => string[]` |
| `promptViaStdin` | boolean | When true, prompt is piped on stdin rather than passed as argv |
| `streamFormat` | string | `'claude-stream-json'`, `'json-event-stream'`, `'acp-json-rpc'`, `'pi-rpc'`, `'copilot-stream-json'`, `'qoder-stream-json'`, `'plain'` |
| `eventParser` | string | Sub-parser hint within `json-event-stream` (e.g. `'codex'`, `'gemini'`, `'opencode'`, `'cursor-agent'`) |
| `maxPromptArgBytes` | number | Byte budget for argv-delivered prompts (DeepSeek: 30,000) |
| `env` | object | Static per-agent process env additions (e.g. Gemini: `GEMINI_CLI_TRUST_WORKSPACE=true`) |
| `supportsImagePaths` | boolean | Agent accepts image paths in RPC protocol (Pi only) |
| `mcpDiscovery` | string | `'mature-acp'` — enables Live Artifacts MCP server injection (Hermes, Kimi) |

### `probe(def, configuredEnv)`

1. Calls `resolveAgentExecutable(def, configuredEnv)` — returns null if not found, marks `available: false`.
2. Builds `probeEnv` via `spawnEnvForAgent()`.
3. Runs `--version` with 3 second timeout to get version string.
4. If `def.helpArgs` and `def.capabilityFlags` defined, runs `helpArgs` with 5 second timeout and checks output for each flag substring; stores result in `agentCapabilities` map.
5. Calls `fetchModels()` — tries `def.fetchModels` first, then `def.listModels`, falls back to `def.fallbackModels`.

### `resolveAgentExecutable(def, configuredEnv)`

1. Checks `AGENT_BIN_ENV_KEYS` for a per-agent env override (e.g. `CODEX_BIN` for Codex).
2. If configured override exists and points to a real file, returns it.
3. Otherwise walks `[def.bin, ...def.fallbackBins]` through `resolveOnPath()`.
4. Returns null if nothing found.

---

## Prompt Composition Pipeline

### `composeSystemPrompt(input)` — 10-Layer Stack

Source: `apps/daemon/src/prompts/system.ts`

| Layer | Content | Condition |
|---|---|---|
| 1 | `DISCOVERY_AND_PHILOSOPHY` (from `./discovery.ts`) | Always present; pinned first so its hard rules win precedence |
| 2 | Separator + `BASE_SYSTEM_PROMPT` (`OFFICIAL_DESIGNER_PROMPT` from `./official-system.ts`) | Always present |
| 3 | Active design system block: `## Active design system — <title>\n\n<designSystemBody>` | Only if `designSystemBody` is non-empty |
| 4 | Active craft references block: `## Active craft references — <sections>\n\n<craftBody>` | Only if `craftBody` is non-empty |
| 5 | Active skill block: `## Active skill — <skillName>\n\n<preflight>\n\n<skillBody>` | Only if `skillBody` is non-empty; includes `derivePreflight()` result |
| 6 | Metadata block (rendered by `renderMetadataBlock()`) | Only if `metadata` or `template` supplied |
| 7 | `DECK_FRAMEWORK_DIRECTIVE` | When `skillMode === 'deck'` OR `metadata.kind === 'deck'`, AND the skill body does not reference `assets/template.html` |
| 8 | `MEDIA_GENERATION_CONTRACT` | When `skillMode` or `metadata.kind` is `'image'`, `'video'`, or `'audio'` |
| 9 | Codex imagegen override (from `renderCodexImagegenOverride()`) | When `agentId === 'codex'`, `metadata.kind === 'image'`, and a `gpt-image-*` model is selected |
| 10 | Critique Theater panel prompt (from `renderPanelPrompt()`) | When `critique.enabled === true` AND `critiqueBrand` and `critiqueSkill` supplied AND not a media surface |

### `derivePreflight(skillBody)`

Scans the skill body for references to `assets/template.html` and/or `references/*.md`. When found, injects a hard pre-flight rule above the skill body that forces the agent to open and read those files before writing any code. This uses `withSkillRootPreamble()` which provides two paths: the cwd-relative `.od-skills/<folder>/` staging path (primary) and the absolute repo path (fallback).

### `craftBody` Injection

Craft sections are loaded by `loadCraftSections(skillCraftRequires, CRAFT_DIR)` from `apps/daemon/src/craft.ts`. The skill's `od.craft.requires` YAML field provides a list of craft slugs (e.g. `['anti-ai-slop', 'typography-system']`). Craft rules sit between the design system and the skill body; brand tokens win on conflict, craft rules fill the gaps.

### Deck Framework Directive

`DECK_FRAMEWORK_DIRECTIVE` pins last to override any softer slide-handling wording. It carries the load-bearing nav counter, scroll JS, and print stylesheet contract that PDF stitching depends on. Skipped when the active skill ships its own `assets/template.html` seed.

---

## Agent Spawn & Transport

### Full Spawn Flow

1. `POST /api/chat` or `POST /api/runs` receives the chat request.
2. `design.runs.create(meta)` creates an in-memory run with status `'queued'`.
3. `design.runs.start(run, () => startChatRun(body, run))` schedules `startChatRun`.
4. `startChatRun()` resolves the agent def, composes the prompt, builds args via `def.buildArgs()`, and calls `child_process.spawn()`.
5. The run transitions to `'starting'`, then `'running'`.

### `buildArgs()` Contract

Each agent's `buildArgs(prompt, imagePaths, extraAllowedDirs, options, runtimeContext)` returns the argv array for `child_process.spawn()`. `options` carries `{ model, reasoning }` from the user's model picker. `runtimeContext` carries `{ cwd }` for CLIs needing an explicit workspace flag.

### `child_process.spawn()`

```
spawn(resolvedBin, argv, {
  cwd: projectDir,            // pinned to .od/projects/<id>/
  stdio: ['pipe', 'pipe', 'pipe'],
  env: spawnEnv,
})
```

CWD is always pinned to the project directory. All three stdio streams are piped so the daemon can write the prompt on stdin and read agent output on stdout/stderr.

### Environment Injection

`createAgentRuntimeEnv(baseEnv, daemonUrl, toolTokenGrant, nodeBin)` builds the agent env:

| Variable | Purpose |
|---|---|
| `OD_DAEMON_URL` | Daemon HTTP URL for agent tool calls |
| `OD_NODE_BIN` | Absolute path to the Node runtime binary |
| `OD_TOOL_TOKEN` | Scoped per-run token for `/api/tools/*` endpoints |
| `OD_BIN` | Absolute path to the daemon CLI script |
| `OD_PROJECT_ID` | Current project UUID |
| `OD_PROJECT_DIR` | Absolute path to project directory |

`spawnEnvForAgent(agentId, baseEnv, configuredEnv)` applies per-agent env stripping: for the `claude` agent, `ANTHROPIC_API_KEY` is stripped unless `ANTHROPIC_BASE_URL` is set (prevents the API key from leaking to a BYOK proxy path).

### Windows ENAMETOOLONG Mitigation

Windows `CreateProcess` caps `lpCommandLine` at ~32 KB (direct) or ~8 KB (via `.cmd` shim). The daemon detects this before spawn:

1. `checkWindowsCmdShimCommandLineBudget()` — checks if the resolved binary ends in `.cmd`/`.bat` and estimates cmd.exe quoting overhead.
2. `checkWindowsDirectExeCommandLineBudget()` — checks CreateProcess budget for direct `.exe` binaries.
3. If over budget and `def.promptViaStdin === false`, the daemon falls back to writing the prompt to a temp file (`.od-prompt-<timestamp>-<random>.md`) in the project cwd and replacing the prompt arg with `promptFileBootstrap(fp)` — a short instruction telling the agent to open and read that file first.
4. DeepSeek is the only agent using argv prompt delivery (`maxPromptArgBytes: 30_000`); `checkPromptArgvBudget()` returns an actionable error before spawn if over budget.

---

## Stream Parsers

### claude-stream-json

Source: `apps/daemon/src/claude-stream.ts`

Parses Claude Code's `--output-format stream-json --verbose` JSONL line by line.

**Events emitted:**

| Event type | Trigger |
|---|---|
| `status` | `system+init` (label: `initializing`, includes `model`, `sessionId`); `system+status` (label: from `obj.status`) |
| `text_delta` | `stream_event` with `content_block_delta` and `delta.type === 'text_delta'`; also `assistant` wrapper text when no streaming deltas preceded it |
| `thinking_delta` | `stream_event` with `delta.type === 'thinking_delta'` |
| `thinking_start` | `stream_event` with `content_block_start` and `block.type === 'thinking'` |
| `tool_use` | `assistant` wrapper with `content[i].type === 'tool_use'` (after input JSON accumulation) |
| `tool_result` | `user` message with `content[i].type === 'tool_result'` |
| `usage` | `result` message with `usage` and optional `cost` |
| `raw` | Lines that fail JSON parse |

**Double-emit prevention:** The `textStreamed` Set tracks message IDs that already emitted text via `stream_event` deltas. When the final `assistant` wrapper arrives, text emission is skipped for those IDs. This handles both old Claude Code (pre-1.0.86, no partial streaming) and new builds.

### plain

Raw stdout bytes forwarded chunk-by-chunk to `text_delta` events. Used by Qwen Code and DeepSeek TUI. No parsing; tool calls appear as text.

### acp-json-rpc

Source: `apps/daemon/src/acp.ts`

**Session lifecycle:**

1. Daemon spawns the agent (`stdio: ['pipe', 'pipe', 'pipe']`).
2. Sends `initialize` RPC: `{ jsonrpc:'2.0', id:1, method:'initialize', params:{ protocolVersion:1, clientCapabilities:{terminal:false}, clientInfo:{...} } }`.
3. On `result` for id 1: sends `session/new` RPC with `{ cwd, mcpServers }`.
4. If a model is specified: sends `session/set_model` RPC.
5. Sends `session/prompt` with the composed prompt.
6. Streams `session/update` notifications — maps to `status`, `text_delta`, `thinking_delta`, `tool_use`, `tool_result`, `usage`.
7. On `session/complete`: finishes the SSE stream.

**Stage timeout:** 180 seconds default (`DEFAULT_STAGE_TIMEOUT_MS`). Each lifecycle stage resets the timer.

**Permission auto-approve:** Any `permission_request` method receives `approve_for_session` or the first `allow_always` option automatically.

**Abort:** `run.acpSession.abort()` sends the graceful shutdown signal; a `PI_ABORT_GRACE_MS` (default 3000ms) timer then sends `SIGTERM` if the child is still alive.

### pi-rpc

Source: `apps/daemon/src/pi-rpc.ts`

**Custom JSON-RPC protocol over stdio.**

1. Daemon spawns `pi --mode rpc [--model ...] [--thinking ...]`.
2. Sends `{ type: 'prompt', id: 1, prompt: '...', images: [...] }` on stdin.
3. Pi streams events: `agent_start`, `turn_start`, `message_update`, `tool_use`, `tool_result`, `agent_end`.
4. Extension UI dialog requests (`confirm`, `select`, `input`, `editor`) are auto-resolved; fire-and-forget methods (`setStatus`, `setWidget`, `notify`, `setTitle`, `set_editor_text`) are silently consumed.

**Image forwarding:** Max 10 images, 20MB total, `.png`/`.jpg`/`.jpeg`/`.gif`/`.webp` only. Images are base64-encoded and embedded in the `prompt` command's `images` array.

### json-event-stream

Source: `apps/daemon/src/json-event-stream.ts`

Handles four sub-parsers selected by `eventParser`:

**Codex** (`eventParser: 'codex'`): `thread.started` → status; `turn.started` → status; `item` (assistant text/tool_use/tool_result) → respective events; `turn.completed` → usage.

**Gemini** (`eventParser: 'gemini'`): Similar streaming structure; uses `--output-format stream-json --yolo`.

**OpenCode** (`eventParser: 'opencode'`): `step_start` → status running; `text` → text_delta; `tool_use` (with `callID`) → tool_use; `tool_result` → tool_result; `session_complete` with tokens → usage.

**Cursor Agent** (`eventParser: 'cursor-agent'`): `--print --output-format stream-json --stream-partial-output`; similar structure to Gemini.

### qoder-stream-json

Source: `apps/daemon/src/qoder-stream.ts`

Parses Qoder CLI's `--output-format stream-json` JSONL:
- `system+init` → status `initializing` (includes model, sessionId, qodercliVersion)
- `assistant+message` → text_delta (text blocks) and thinking_delta (thinking blocks, with `thinking_start` gated by `emittedThinkingStart` flag)
- `result` with `is_error: true` → error event; success → usage event

### copilot-stream-json

Source: `apps/daemon/src/copilot-stream.ts`

Parses GitHub Copilot CLI `--output-format json` JSONL using dotted type names:
- `session.tools_updated` → status `initializing` with model name
- `assistant.turn_start` → status `streaming`
- `assistant.reasoning_delta` → thinking_delta
- `assistant.message_delta` → text_delta
- `tool.execution_start` → tool_use
- `tool.execution_complete` → tool_result
- `result` → usage

---

## BYOK API Proxy

Source: `apps/daemon/src/server.ts` lines ~5423–5800

### Routes

- `POST /api/proxy/anthropic/stream` — forwards to Anthropic Messages API
- `POST /api/proxy/openai/stream` — forwards to OpenAI Chat Completions API
- `POST /api/proxy/azure/stream` — forwards to Azure OpenAI API
- `POST /api/proxy/google/stream` — forwards to Google Gemini API

### SSRF Blocking

`validateExternalApiBaseUrl(baseUrl)` calls `origin-validation.ts`:
- Rejects non-http/https protocols.
- Parses the hostname; if it resolves to a private IPv4 (10/8, 172.16/12, 192.168/16, 169.254/16) or is a loopback/localhost, returns `{ forbidden: true }`.
- The proxy endpoint returns HTTP 403 `FORBIDDEN` for private targets.

### SSE Normalization

All four proxy routes normalize upstream SSE into: `start` (with model), `delta` (text chunk), `end`, `error` events. Auth tokens in error messages are redacted via `redactAuthTokens()` before logging.

---

## Skill Injection Strategies

Strategies differ per agent:

| Agent | Strategy |
|---|---|
| Claude Code | Skill staged to `.od-skills/<id>/` in project cwd + `--add-dir <absoluteSkillDir>` flag passes the dir to Claude's sandbox |
| Codex CLI | Skill staged to `.od-skills/<id>/` + `--add-dir <absoluteSkillDir>` |
| Cursor Agent | Skill staged to `.od-skills/<id>/`; `.cursorrules` file written with preamble |
| Gemini CLI | Prompt injection only (stdin); skill body embedded in composed system prompt |
| OpenCode | Prompt injection only (stdin) |
| Qwen Code | Prompt injection only (stdin) |
| Qoder CLI | Prompt injection (stdin) + `--add-dir` for extraAllowedDirs (which includes skill dir) + `--attachment` for image paths |
| GitHub Copilot CLI | `--add-dir <absoluteSkillDir>` + skill body in composed system prompt (stdin) |
| Devin, Hermes, Kimi, Kiro, Kilo, Vibe | ACP MCP bridge — skill body in composed prompt passed to `session/prompt` |
| Pi | Prompt injection in RPC `prompt` command; `--append-system-prompt` for extraAllowedDirs |
| DeepSeek | Prompt injection as positional argv (guarded by `maxPromptArgBytes`) |

### Staging Safety

`stageActiveSkill(cwd, folderName, sourceDir)` in `cwd-aliases.ts`:
- Validates `folderName`: rejects `.`, `..`, path separators, absolute paths, null bytes.
- Uses `fs.cp(sourceDir, stagedPath, { recursive: true, dereference: true })` — dereferences source symlinks so the staged copy is fully self-contained and cannot write back to the source tree.
- Replaces the previous-turn copy wholesale so skill-source updates mid-session take effect immediately.
- Non-throwing: returns a `SkillStagingResult` with `staged: false` and a `reason` on failure; callers fall back to the absolute path via `--add-dir`.

---

## Side-File Pre-flight

`derivePreflight(skillBody)` in `apps/daemon/src/prompts/system.ts`:

1. Checks if `skillBody` references `assets/template.html` or `references/*.md`.
2. If so, injects the following before the skill body:

```
Before writing any code, open and read:
  .od-skills/<folder>/assets/template.html  (primary)
  <absoluteSkillDir>/assets/template.html   (fallback)
  .od-skills/<folder>/references/*.md       (all reference docs)
Do not begin generating the artifact until you have read the template and all reference docs.
```

3. This is the "forced Read tool" pattern — the agent must read the skill's seed template and checklists before starting. Without it, the agent may generate code that conflicts with the seed or misses required patterns.

The preamble is injected by `withSkillRootPreamble()` in `skills.ts`, which only fires for skills that have a non-empty `assets/` or `references/` directory.

---

## Artifact Linting (Anti-Slop)

Source: `apps/daemon/src/lint-artifact.ts`

`lintArtifact(rawHtml)` runs grep-style checks against raw HTML. Deliberately does NOT parse HTML — false positives are tolerable because each finding includes the matched snippet for agent verification.

### P0 Findings (Must Fix)

| ID | Description |
|---|---|
| `purple-gradient` | Gradient containing a violet/indigo hex or keyword `purple`/`violet` |
| `trust-gradient` | Blue→cyan two-stop "trust gradient" (Tailwind blue + cyan hexes paired) |
| `ai-default-indigo` | Any solid use of canonical LLM accent hexes (indigo/violet) |
| `emoji-icon` | Rocket/sparkle/target/flame/chart-up slop emoji used as feature icon |
| `left-accent-card` | `border-left: 4px solid` card pattern (cardinal AI-slop tell) |
| `sans-display` | h1/h2/h3 using Inter/Roboto/Arial/-apple-system without a display face |
| `invented-metric` | `10x faster`, `99.9% uptime`, `zero-downtime`, etc. |
| `filler-copy` | "feature one/two/three", "lorem ipsum", "placeholder text", "sample content" |
| `scroll-into-view` | `scrollIntoView()` in generated JS (layout anti-pattern) |
| `slide-theme-missing` | Deck missing a `data-theme` attribute |
| `all-caps-no-tracking` | Uppercase text without letter-spacing |

### P1 Findings (Should Fix)

| ID | Description |
|---|---|
| `all-caps-no-tracking` | Same as P0 but detected in inline styles |
| `external-image` | `<img src="https://...">` — CDN images break offline |
| `raw-hex` | Hardcoded hex color outside `:root` (not using CSS vars) |
| `accent-overuse` | More than 3 uses of the accent color (overuse cap) |

### P2 Findings (Nice to Have)

| ID | Description |
|---|---|
| `missing-section-anchor` | Section element without an `id` anchor |
| `slide-rhythm` | Deck slide missing a rhythm class |

### Feedback Loop

`renderFindingsForAgent(findings)` renders a Markdown block prepended to the next-turn system message so the agent can self-correct. Only P0 findings trigger the feedback loop automatically; P1/P2 are advisory badges in the UI.

---

## Critique Theater

Source: `apps/daemon/src/critique/`

### Overview

Multi-round panelist scoring system. After the agent generates an artifact, the orchestrator parses a structured `<CRITIQUE_RUN>` block from the agent's output, runs up to `OD_CRITIQUE_MAX_ROUNDS` scoring rounds, and ships the best-scoring artifact.

### Five Dimensions

Philosophy, Hierarchy, Detail, Function, Innovation — each scored 0–10 (or on a configurable scale via `OD_CRITIQUE_SCORE_SCALE`). Composite is a weighted average.

### `OrchestratorParams`

```typescript
{
  runId, projectId, conversationId, artifactId, artifactDir,
  adapter,              // streamFormat string
  cfg,                  // CritiqueConfig
  db,                   // SQLite handle
  bus,                  // CritiqueSseBus (emits CritiqueSseEvent to SSE stream)
  stdout,               // AsyncIterable<string> from child process
  signal?,              // AbortController signal
  child?,               // ChildProcess handle for kill on non-clean exit
  childExitPromise?,    // resolves when child exits
}
```

### States

| State | Description |
|---|---|
| `running` | Orchestrator active, rounds in progress |
| `shipped` | Score met threshold, artifact shipped |
| `below_threshold` | All rounds exhausted, shipped best-so-far |
| `interrupted` | Abort signal received; flushed best-so-far |
| `timed_out` | `OD_CRITIQUE_TOTAL_TIMEOUT_MS` elapsed; flushed best-so-far |
| `degraded` | Parser error or child exit mid-round; shipped best-so-far |
| `failed` | Catastrophic error, no artifact available |

### Ship Resolution

On any non-clean termination (abort, timeout, degraded), the orchestrator calls `selectFallbackRound()` to select the best-so-far round by composite score and ships that artifact. This ensures the user always receives an output.

### Env Vars

`OD_CRITIQUE_ENABLED` (default false), `OD_CRITIQUE_MAX_ROUNDS`, `OD_CRITIQUE_SCORE_THRESHOLD`, `OD_CRITIQUE_SCORE_SCALE`, `OD_CRITIQUE_PER_ROUND_TIMEOUT_MS`, `OD_CRITIQUE_TOTAL_TIMEOUT_MS`.

---

## File System & Workspace

### `.od/` Layout

```
.od/
  app.sqlite               # SQLite database
  media-config.json        # API keys (gitignored)
  projects/
    <uuid>/                # per-project working directory
      index.html           # generated artifact
      *.html, *.css, ...   # other project files
      .od-skills/          # staged skill copies
        <skill-id>/        # copy of skills/<skill-id>/
  artifacts/               # saved render snapshots
```

`OD_DATA_DIR` relocates all of `.od/` to an arbitrary path. `OD_MEDIA_CONFIG_DIR` narrows the override to just `media-config.json`.

### Per-Project CWD

Each project's files live under `PROJECTS_DIR/<id>/`. The daemon sets this as the `cwd` for all agent spawns. The `OD_PROJECT_DIR` env var is set to the absolute path so agents can write files without knowing the daemon's directory layout.

### `.od-skills/` Staging

`stageActiveSkill()` copies the active skill's folder to `<projectCwd>/.od-skills/<skillId>/` before each spawn. The `.od-skills/` prefix is OD-reserved; if a real file exists at that path, staging refuses to clobber it. A legacy symlink from an earlier daemon version is replaced with a real directory.

### ENAMETOOLONG Fallbacks on Windows

For agents that deliver the prompt as a positional argv arg, the daemon checks the combined command-line budget before spawn. The fallback chain is:
1. Check `checkWindowsCmdShimCommandLineBudget()` if binary ends in `.cmd`/`.bat`.
2. Check `checkWindowsDirectExeCommandLineBudget()` if direct `.exe`.
3. If over budget: write prompt to temp file `<projectCwd>/.od-prompt-<ts>-<rand>.md`; replace the prompt arg with `promptFileBootstrap(fp)`.
4. The temp file is cleaned up after the child exits.

---

## Persistence / SQLite

Source: `apps/daemon/src/db.ts`

Database opened with `PRAGMA journal_mode = WAL; PRAGMA foreign_keys = ON;`. Schema created by `migrate()` on first open; additional columns added forward-compatibly via `PRAGMA table_info` checks.

### Tables

**projects**
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `name` | TEXT NOT NULL | Display name |
| `skill_id` | TEXT | Active skill ID |
| `design_system_id` | TEXT | Active design system ID |
| `pending_prompt` | TEXT | Queued prompt for async send |
| `metadata_json` | TEXT | JSON blob (kind, fidelity, speakerNotes, etc.) |
| `created_at` | INTEGER | Unix ms |
| `updated_at` | INTEGER | Unix ms |

**templates**
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `name` | TEXT NOT NULL | |
| `description` | TEXT | |
| `source_project_id` | TEXT | Source project UUID |
| `files_json` | TEXT NOT NULL | JSON array of `{name, content}` |
| `created_at` | INTEGER | Unix ms |

**conversations**
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `project_id` | TEXT NOT NULL FK → projects | CASCADE DELETE |
| `title` | TEXT | |
| `created_at` | INTEGER | Unix ms |
| `updated_at` | INTEGER | Unix ms |

Index: `idx_conv_project(project_id, updated_at DESC)`

**messages**
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `conversation_id` | TEXT NOT NULL FK → conversations | CASCADE DELETE |
| `role` | TEXT NOT NULL | `'user'` or `'assistant'` |
| `content` | TEXT NOT NULL | Message text |
| `agent_id` | TEXT | Agent that generated this message |
| `agent_name` | TEXT | Display name at time of generation |
| `run_id` | TEXT | Associated run UUID |
| `run_status` | TEXT | Final run status snapshot |
| `last_run_event_id` | TEXT | Last SSE event ID processed |
| `events_json` | TEXT | Stored SSE events array |
| `attachments_json` | TEXT | Image attachment paths |
| `comment_attachments_json` | TEXT | Preview comment attachments |
| `produced_files_json` | TEXT | Files written by agent |
| `started_at` | INTEGER | Unix ms |
| `ended_at` | INTEGER | Unix ms |
| `position` | INTEGER NOT NULL | Message order in conversation |
| `created_at` | INTEGER | Unix ms |

**preview_comments**
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `project_id` | TEXT NOT NULL FK → projects | |
| `conversation_id` | TEXT NOT NULL FK → conversations | |
| `file_path` | TEXT NOT NULL | |
| `element_id` | TEXT NOT NULL | |
| `selector` | TEXT NOT NULL | CSS selector |
| `label` | TEXT NOT NULL | |
| `text` | TEXT NOT NULL | Comment body |
| `position_json` | TEXT NOT NULL | `{x,y,width,height}` JSON |
| `html_hint` | TEXT NOT NULL | Captured HTML snippet |
| `selection_kind` | TEXT | `'element'` or `'pod'` |
| `member_count` | INTEGER | Pod member count |
| `pod_members_json` | TEXT | JSON array of pod members |
| `note` | TEXT NOT NULL | |
| `status` | TEXT NOT NULL | |
| `created_at` | INTEGER | Unix ms |
| `updated_at` | INTEGER | Unix ms |

Unique: `(project_id, conversation_id, file_path, element_id)`

**tabs**
| Column | Type | Notes |
|---|---|---|
| `project_id` | TEXT PK, FK → projects | |
| `name` | TEXT PK | File name |
| `position` | INTEGER NOT NULL | Tab order |
| `is_active` | INTEGER NOT NULL DEFAULT 0 | Boolean (0/1) |

**deployments**
| Column | Type | Notes |
|---|---|---|
| `id` | TEXT PK | UUID |
| `project_id` | TEXT NOT NULL FK → projects | CASCADE DELETE |
| `file_name` | TEXT NOT NULL | |
| `provider_id` | TEXT NOT NULL | `'cloudflare-pages'` or `'vercel'` |
| `url` | TEXT NOT NULL | Deployed URL |
| `deployment_id` | TEXT | Provider-assigned deployment ID |
| `deployment_count` | INTEGER NOT NULL DEFAULT 1 | |
| `target` | TEXT NOT NULL DEFAULT 'preview'` | `'preview'` or `'production'` |
| `status` | TEXT NOT NULL DEFAULT 'ready'` | `'ready'`, `'uploading'`, `'failed'` |
| `status_message` | TEXT | Error message if failed |
| `reachable_at` | INTEGER | Unix ms first reachable |
| `provider_metadata_json` | TEXT | Provider-specific metadata |
| `created_at` | INTEGER | Unix ms |
| `updated_at` | INTEGER | Unix ms |

Unique: `(project_id, file_name, provider_id)`

**Critique tables** (added by `migrateCritique()` in `critique/persistence.ts`): `critique_runs` — stores round scores, composite, status, and transcript path per run.

---

## Export Pipeline

| Format | Handler |
|---|---|
| HTML | `GET /api/projects/:id/files/*` — serves raw file with `Content-Type: text/html` |
| HTML preview (sandboxed) | `GET /api/projects/:id/files/:name/preview` — wraps in `<iframe srcdoc>` sandbox |
| PDF | [Uncertain] — likely handled client-side via print CSS in the deck framework directive |
| PPTX | [Uncertain] — no dedicated server-side handler found; may be client-side |
| ZIP (single project) | `GET /api/projects/:id/archive` — `buildProjectArchive()` in `projects.ts` |
| ZIP (batch) | `POST /api/projects/:id/archive/batch` — `buildBatchArchive()` |
| Markdown | [Uncertain] — possibly via transcript export in `transcript-export.ts` |

---

## MCP Server

Source: `apps/daemon/src/mcp.ts`

Invoked as `od mcp` (stdio transport). Proxies read-only calls to the running daemon's HTTP API. The server holds no state and never touches the filesystem directly.

### 8 Tools

| Tool | Description |
|---|---|
| `list_projects` | List all projects on the daemon |
| `get_active_context` | Returns current open project/file (5-min TTL active context) |
| `get_artifact` | BFS traversal — entry file + all referenced files up to depth 3; soft cap 1.5MB, max 200 files |
| `get_project` | Project metadata by UUID or name |
| `get_file` | File content with pagination (up to 2000 lines, `[od:file-window ...]` marker) |
| `search_files` | Full-text search across project files |
| `list_files` | List files in a project |
| `list_comments` | [Uncertain based on tool count] |

All tools have `readOnlyHint: true`, `idempotentHint: true`, `openWorldHint: false`.

### 3 Resources

| URI | Content |
|---|---|
| `od://design-systems/<id>/DESIGN.md` | Active design system DESIGN.md body |
| `od://skills/<id>/SKILL.md` | Skill SKILL.md body |
| `od://focus/active` | Current active context (project + file) |

### Project Resolution

UUID → exact match. Name: exact match → slug match → substring match. Returns an error if multiple projects match a substring.

---

## Sidecar IPC

Source: `apps/daemon/src/sidecar/server.ts`, `@open-design/sidecar`, `@open-design/sidecar-proto`

### Messages

| Message | Description |
|---|---|
| `STATUS` | Returns `DaemonStatusSnapshot` with pid, state, url, updatedAt |
| `SHUTDOWN` | Initiates graceful shutdown |
| `EVAL` | Executes a JS expression in the daemon process context |
| `SCREENSHOT` | [Uncertain — likely desktop screenshot via Electron] |
| `CONSOLE` | Returns recent console log entries |
| `CLICK` | [Uncertain — likely desktop UI automation] |

### Socket Paths

POSIX: `/tmp/open-design/ipc/<namespace>/daemon.sock`

Windows: Named pipe path [Uncertain].

### Daemon Sidecar Handle

`startDaemonSidecar(runtime)` returns a `DaemonSidecarHandle` with `status()`, `stop()`, `waitUntilStopped()`. The sidecar registers a `STATUS` handler that returns the current `DaemonStatusSnapshot` (pid, state, url, updatedAt).

---

## Environment Variables

| Name | Default | Purpose |
|---|---|---|
| `OD_DATA_DIR` | `<projectRoot>/.od` | Daemon runtime data root (SQLite, artifacts, projects) |
| `OD_LEGACY_DATA_DIR` | — | 0.3.x migration source directory |
| `OD_RESOURCE_ROOT` | — | Override path for bundled skills/design-systems/craft/frames |
| `OD_BIND_HOST` | `127.0.0.1` | HTTP server listen address |
| `OD_MEDIA_CONFIG_DIR` | — | Override path for media-config.json only |
| `OD_CODEX_DISABLE_PLUGINS` | `'0'` | Set to `'1'` to pass `--disable plugins` to Codex |
| `OD_ALLOWED_ORIGINS` | — | Comma-separated allowed origins for reverse proxy setup |
| `OD_DAEMON_URL` | — | Daemon URL injected into agent env and used by MCP server |
| `OD_PROJECT_ID` | — | Current project UUID injected into agent env |
| `OD_PROJECT_DIR` | — | Absolute project directory injected into agent env |
| `OD_NODE_BIN` | `process.execPath` | Node binary path injected into agent env |
| `OD_BIN` | resolved CLI path | Daemon CLI script path injected into agent env |
| `OD_TOOL_TOKEN` | — | Per-run scoped token for `/api/tools/*` endpoints |
| `OD_AGENT_HOME` | — | Override home directory for toolchain binary search (testing) |
| `OD_WEB_PORT` | — | Web listener port (for allowed browser port validation) |
| `CODEX_BIN` | — | Override binary path for Codex CLI |
| `PI_GRACEFUL_SHUTDOWN_MS` | `5000` | Pi RPC graceful shutdown wait time |
| `PI_ABORT_GRACE_MS` | `3000` | Grace period before SIGTERM after pi abort() |
| `OD_CRITIQUE_ENABLED` | `false` | Enable Critique Theater |
| `OD_CRITIQUE_MAX_ROUNDS` | — | Maximum critique rounds |
| `OD_CRITIQUE_SCORE_THRESHOLD` | — | Composite score to stop early |
| `OD_CRITIQUE_SCORE_SCALE` | — | Score scale (default 10) |
| `OD_CRITIQUE_PER_ROUND_TIMEOUT_MS` | — | Per-round timeout |
| `OD_CRITIQUE_TOTAL_TIMEOUT_MS` | — | Total orchestrator timeout |
| `GEMINI_CLI_TRUST_WORKSPACE` | — | Set to `'true'` by Gemini adapter env |
| `CODEX_HOME` | `~/.codex` | Codex home directory (for imagegen output path) |
| `OD_DAEMON_CLI_PATH` | — | Override path to daemon CLI script |

---

## Security-Sensitive Areas

### Spawn Sanitization

- `sanitizeCustomModel(id)`: allows alphanumeric + `._/:@-`, max 200 chars; rejects anything outside this set.
- `checkPromptArgvBudget()`: prevents argv overflow before spawn.
- `checkWindowsCmdShimCommandLineBudget()` / `checkWindowsDirectExeCommandLineBudget()`: prevents Windows CreateProcess overflow.
- `isSafeAliasSegment(folderName)` in `cwd-aliases.ts`: rejects `.`, `..`, path separators, absolute paths, null bytes.
- Skill staging uses `dereference: true` so no symlink can write back to the source tree.

### SSRF Blocking

`origin-validation.ts`:
- `isPrivateIpv4()`: blocks RFC 1918 and link-local (10/8, 172.16/12, 192.168/16, 169.254/16).
- `validateExternalApiBaseUrl()`: called by all BYOK proxy routes before any upstream fetch.
- Returns HTTP 403 `FORBIDDEN` for private/loopback targets.

### Loopback Enforcement

`requireLocalDaemonRequest(req, res, next)` validates:
1. `req.socket.remoteAddress` is loopback (127.x.x.x, ::1).
2. `Host` header is loopback or allowed.
3. `Origin` header (if present) passes `isAllowedBrowserOrigin()`.

`OD_BIND_HOST` defaults to `127.0.0.1` so the HTTP server is not exposed on LAN by default.

### API Key Storage

API keys stored in `.od/media-config.json` (gitignored). Never stored in SQLite. `readMaskedConfig()` redacts key values before serving `GET /api/media/config`. `ANTHROPIC_API_KEY` is stripped from the agent spawn environment by `spawnEnvForAgent()` unless `ANTHROPIC_BASE_URL` is set.

### Generated Code Sandbox

Artifact previews (`GET /api/projects/:id/files/:name/preview`) wrap content in `<iframe srcdoc sandbox="...">` with a restricted sandbox attribute. Generated code cannot access parent frame, cookies, or storage.

### MCP Read-Only Contract

All MCP tools carry `readOnlyHint: true`. The MCP server only calls GET endpoints on the daemon. It never accepts write requests or file system mutations.

---

## Debugging Guide

### Agent Execution

1. Check `GET /api/agents` to verify the agent is detected (`available: true`).
2. Set `OD_AGENT_HOME` to a test directory to isolate binary resolution.
3. Add `console.log` around the `spawn()` call in `server.ts` to log the exact argv and env.
4. Watch daemon stderr — `spawnEnvForAgent()` strips keys; confirm the right env is set.
5. For ACP agents, check the stage timeout (`DEFAULT_STAGE_TIMEOUT_MS = 180s`); increase if model is slow to initialize.

### Artifact Preview

1. Open `GET /api/projects/:id/files/:name/preview` in a browser with DevTools.
2. Check the `srcdoc` content of the iframe for lint issues.
3. Call `POST /api/artifacts/lint` with the artifact HTML to get structured findings.

### Streams

1. For `claude-stream-json`: paste raw JSONL from agent stdout through `createClaudeStreamHandler` in isolation.
2. For ACP: check the JSON-RPC message log in daemon stderr (the `sendRpc` / `sendRpcResult` calls write no logs by default — add temporary `console.log` around them).
3. Use `GET /api/runs/:id/events` with a browser EventSource to watch the live SSE stream.
4. The run's `events` array in memory stores up to `maxEvents` (default 2000) records and replays on reconnect via `Last-Event-ID`.

### SQLite

1. `sqlite3 .od/app.sqlite` for direct inspection.
2. Check `PRAGMA integrity_check;` if the daemon crashes on startup.
3. WAL mode: check for `-shm`/`-wal` files; they are normal and cleaned up on clean close.
4. The `migrateCritique()` migration runs on every open; check `critique_runs` table for stuck orchestrators.

---

## Critical File Analysis

### `apps/daemon/src/agents.ts`

**Role:** Single source of truth for all 16 agent adapters. Exports AGENT_DEFS, detectAgents(), resolveAgentExecutable(), buildArgs() per def, checkPromptArgvBudget(), Windows budget checks, spawnEnvForAgent(), sanitizeCustomModel().

**How invoked:** `detectAgents()` called by `GET /api/agents`. `getAgentDef(id)` and `buildArgs()` called by `startChatRun()` inside `server.ts`. `spawnEnvForAgent()` called just before `child_process.spawn()`.

**Inputs:** `configuredEnvByAgent` map (per-agent env overrides from app config), `process.env` for PATH and toolchain dirs.

**Outputs:** Array of agent info objects (for UI) including `available`, `version`, `path`, `models`, `streamFormat`, `reasoningOptions`.

**Dependencies:** `@open-design/platform` (wellKnownUserToolchainBins, createCommandInvocation), `acp.ts` (detectAcpModels), `pi-rpc.ts` (parsePiModels), `home-expansion.ts`.

**Failure modes:** Binary not on PATH → `available: false`. `--version` times out → still marks available, version null. `--help` fails → caps left empty, buildArgs uses safe baseline. Model listing fails → falls back to `fallbackModels`. Windows budget exceeded → actionable SSE error before spawn.

**Security concerns:** `sanitizeCustomModel()` is the only guard between user-supplied model IDs and the CLI. `spawnEnvForAgent()` must strip sensitive env vars (ANTHROPIC_API_KEY).

**Debugging tips:** Set `OD_AGENT_HOME` to a controlled directory to test binary resolution in isolation. Log `resolveOnPath()` results to see which binary wins.

---

### `apps/daemon/src/server.ts`

**Role:** Express application entry point. Registers all ~70 API routes, initializes all services (SQLite, skill/design-system dirs, critique config, run registry, sidecar-related services), exports `startServer()`.

**How invoked:** Called by `apps/daemon/src/sidecar/server.ts`'s `startDaemonSidecar()`, or directly from `apps/daemon/src/cli.ts` for the standalone `od` binary.

**Inputs:** `{ port, host, returnServer }` from the sidecar or CLI.

**Outputs:** Express `http.Server` bound to `host:port`, or a `DaemonSidecarHandle` wrapper.

**Dependencies:** Nearly every other module in `apps/daemon/src/`. Key: `db.ts`, `agents.ts`, `runs.ts`, `skills.ts`, all stream parsers, `lint-artifact.ts`, `critique/`, `deploy.ts`, `media.ts`, `mcp.ts`.

**Failure modes:** `OD_DATA_DIR` not writable → throws `Error` at startup. Bad `OD_CRITIQUE_*` value → `RangeError` at startup. Port already in use → Node `EADDRINUSE` crash.

**Security concerns:** This is the attack surface. Every route that calls `requireLocalDaemonRequest` is protected; routes without it are accessible to any local process. The BYOK proxy routes validate SSRF; never remove `validateExternalApiBaseUrl()`.

**Debugging tips:** Search for `app.post`/`app.get` etc. to find route handlers. The large `startServer()` function contains most route logic inline; use line number references from the grep output to navigate.

---

### `apps/daemon/src/runs.ts`

**Role:** In-memory run registry and SSE fan-out. Manages the lifecycle of agent runs: creation, queueing, streaming, cancellation, cleanup.

**How invoked:** `createChatRunService(config)` called once in `startServer()`; returned service stored as `design.runs`. Routes call `design.runs.create()`, `design.runs.start()`, `design.runs.stream()`, etc.

**Inputs:** Config: `{ createSseResponse, createSseErrorPayload, maxEvents, ttlMs }`. Run meta: `{ projectId, conversationId, assistantMessageId, clientRequestId, agentId }`.

**Outputs:** Run objects with `id`, `status`, `events[]`, `clients` Set, `child` handle, `acpSession` handle.

**Dependencies:** `runs.ts` is pure logic; it calls back through `createSseResponse` (provided by server.ts) and `createSseErrorPayload`.

**Failure modes:** Runs are in-memory only — daemon restart loses all run state. `maxEvents` (2000) cap prevents unbounded memory growth. `ttlMs` (30 min) schedules cleanup after terminal state.

**Security concerns:** `run.child` is a raw ChildProcess handle. Cancel logic sends SIGTERM; a misbehaving agent that ignores SIGTERM will leak a zombie process until the daemon exits.

**Debugging tips:** Log `runs.size` to watch for leaks. Add a `runs.forEach` dump endpoint during debugging to inspect all active runs.

---

### `apps/daemon/src/claude-stream.ts`

**Role:** Parses Claude Code's `--output-format stream-json --verbose` JSONL stdout into typed events.

**How invoked:** `createClaudeStreamHandler(onEvent)` called in `server.ts`'s `startChatRun()` when `streamFormat === 'claude-stream-json'`. The `feed(chunk)` method is called on each stdout data chunk; `flush()` is called on stream close.

**Inputs:** Raw stdout chunks from Claude Code's child process.

**Outputs:** Events via `onEvent` callback: `status`, `text_delta`, `thinking_delta`, `thinking_start`, `tool_use`, `tool_result`, `usage`, `raw`.

**Dependencies:** None external. Self-contained parser.

**Failure modes:** Malformed JSON lines → emitted as `raw` event (not errors). Missing `--include-partial-messages` → text arrives only in final `assistant` wrapper, no incremental streaming. This is handled by the `textStreamed` Set.

**Security concerns:** None specific; parser never executes code from the stream.

**Debugging tips:** Pipe Claude Code stdout to a file and replay it through `createClaudeStreamHandler` in a test to inspect event sequence.

---

### `apps/daemon/src/skills.ts`

**Role:** Skill registry — scans `<SKILLS_DIR>/*/SKILL.md`, parses YAML frontmatter, returns structured skill objects.

**How invoked:** `listSkills(SKILLS_DIR)` called on `GET /api/skills`, `GET /api/skills/:id`, and inside `startChatRun()` to resolve the active skill. `findSkillById(skills, id)` resolves deprecated IDs via `SKILL_ID_ALIASES`.

**Inputs:** `skillsRoot` — absolute path to the skills directory.

**Outputs:** Array of skill objects: `{ id, name, description, triggers, mode, surface, craftRequires, platform, scenario, previewType, designSystemRequired, defaultFor, upstream, featured, fidelity, speakerNotes, animations, examplePrompt, body, dir }`.

**Dependencies:** `frontmatter.ts` (YAML parser), `cwd-aliases.ts` (SKILLS_CWD_ALIAS constant).

**Failure modes:** SKILL.md missing or unreadable → entry silently skipped. Duplicate `name` values → both returned (no dedup). `SKILL_ID_ALIASES` must be updated on skill renames or stored project `skill_id` values will silently resolve to null.

**Security concerns:** `withSkillRootPreamble()` embeds absolute filesystem paths in the injected preamble; these are sent to the agent. Acceptable because the agent needs them to read skill assets.

**Debugging tips:** Call `listSkills(SKILLS_DIR)` directly in a REPL to inspect parsed output. Check frontmatter parsing with `parseFrontmatter(rawMarkdown)`.

---

### `apps/daemon/src/prompts/system.ts`

**Role:** Assembles the final system prompt from up to 10 layers. The result is what the daemon sends as the `system` field to BYOK providers or embeds in the composed `stdin` prompt for CLI agents.

**How invoked:** `composeSystemPrompt(input)` called inside `startChatRun()` in `server.ts` once per agent turn.

**Inputs:** `ComposeInput` object with optional: `agentId`, `skillBody`, `skillName`, `skillMode`, `designSystemBody`, `designSystemTitle`, `craftBody`, `craftSections`, `metadata`, `template`, `critique`, `critiqueBrand`, `critiqueSkill`.

**Outputs:** Single composed string that is the complete system prompt.

**Dependencies:** `./discovery.ts`, `./official-system.ts`, `./deck-framework.ts`, `./media-contract.ts`, `./panel.ts`, `../media-models.ts`, `@open-design/contracts/critique`.

**Failure modes:** If `skillBody` references `assets/template.html` but staging failed and `--add-dir` is not supported, the agent will receive the correct preamble but the file won't be at the CWD-relative path. The absolute path fallback in the preamble mitigates this.

**Security concerns:** None; pure string composition. The critique panel prompt is gated by `cfg.enabled`; misconfiguring `OD_CRITIQUE_ENABLED` to `true` without valid `critiqueBrand`/`critiqueSkill` produces a no-op (the condition guards all three).

**Debugging tips:** Call `composeSystemPrompt()` with a fixed input in a REPL to inspect the full prompt. Each layer is a distinct string; search for `---` separators to locate layer boundaries.

---

### `apps/daemon/src/prompts/discovery.ts`

**Role:** Exports `DISCOVERY_AND_PHILOSOPHY` — the mandatory first layer of every system prompt. Contains the interactive question-form syntax, brand-spec extraction instructions, direction-picker fork, TodoWrite reinforcement, and the 5-dimension critique guidance.

**How invoked:** Imported by `system.ts`, always prepended first.

**Inputs:** None (static string constant).

**Outputs:** `DISCOVERY_AND_PHILOSOPHY` string.

**Dependencies:** `./directions.ts` (embedded library of direction-picker patterns).

**Failure modes:** None; static content.

**Security concerns:** None.

**Debugging tips:** Read the file directly to understand the agent's mandatory first-turn behavior.

---

### `apps/daemon/src/prompts/directions.ts`

**Role:** Exports the "directions library" embedded in `DISCOVERY_AND_PHILOSOPHY`. Provides the direction-picker fork patterns and brand-spec extraction templates.

**How invoked:** Imported by `discovery.ts`.

**Inputs/Outputs:** Static string constant.

---

### `apps/daemon/src/db.ts`

**Role:** All SQLite persistence — schema creation/migration, CRUD functions for all 7 tables.

**How invoked:** `openDatabase(projectRoot, { dataDir })` called once in `startServer()`. All CRUD functions called throughout `server.ts` route handlers.

**Inputs:** Database file path derived from `OD_DATA_DIR`.

**Outputs:** SQLite `Database` instance (better-sqlite3). CRUD functions return plain JS objects.

**Dependencies:** `better-sqlite3`, `critique/persistence.ts` (for `migrateCritique()`).

**Failure modes:** File permissions error → throws on open. Corrupt WAL → better-sqlite3 may throw on first query. Migration columns already exist → silently skipped via `PRAGMA table_info` check.

**Security concerns:** No parameterized query escape issues (better-sqlite3's `.prepare().get/all/run` always uses bound params). The `metadata_json` column stores arbitrary JSON; server code should validate before trusting.

**Debugging tips:** Use `sqlite3 .od/app.sqlite .schema` to inspect live schema. Use `.mode column` and `.headers on` for readable query output.

---

### `apps/daemon/src/lint-artifact.ts`

**Role:** Anti-slop HTML linter. Runs grep-style checks against artifact HTML and returns structured `LintFinding[]`.

**How invoked:** `lintArtifact(rawHtml)` called in `POST /api/artifacts/save` (inline) and `POST /api/artifacts/lint` (standalone). `renderFindingsForAgent(findings)` renders the P0 findings Markdown block injected as a system message on the next turn.

**Inputs:** Raw HTML string.

**Outputs:** `LintFinding[]` with `{ severity, id, message, fix, snippet? }`.

**Dependencies:** None (pure regex).

**Failure modes:** False positives are intentional (each finding includes a snippet for agent verification). Never throws — regex errors caught internally.

**Security concerns:** None; read-only analysis.

**Debugging tips:** Call `lintArtifact(html)` directly on a saved artifact to get findings. Findings with P0 severity are injected back to the agent automatically; check that the `renderFindingsForAgent` output is well-formed Markdown.
