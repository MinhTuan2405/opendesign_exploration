# Backend Reading Map

> How to navigate the Open Design daemon source for common engineering goals.
> All paths are relative to the repository root unless otherwise noted.

---

## Top 10 Files To Read First

| Priority | File | Why | Read After | Notes |
|----------|------|-----|------------|-------|
| 1 | `apps/daemon/src/agents.ts` | Defines all 16 agent adapters — bin names, stream formats, `buildArgs()`, capability flags, model lists | Nothing | The single source of truth for what CLIs Open Design supports and how they are invoked |
| 2 | `apps/daemon/src/server.ts` | Express HTTP surface (~50 routes); the true API contract between web and daemon | `agents.ts` | Every feature in the UI has a route here; reading it gives a complete capability map |
| 3 | `apps/daemon/src/prompts/system.ts` | `composeSystemPrompt()` — the 4-layer prompt assembly entry point | `agents.ts` | Without this, the prompt reaching the agent is a black box |
| 4 | `apps/daemon/src/db.ts` | SQLite schema (7 tables) + migration logic; source of truth for persistence | `server.ts` | Most routes read/write through functions defined here |
| 5 | `apps/daemon/src/runs.ts` | In-memory SSE run registry; explains how streaming events are buffered and replayed to late-joining clients | `server.ts` | Core concurrency primitive; misunderstanding it leads to incorrect assumptions about event ordering |
| 6 | `apps/daemon/src/claude-stream.ts` | Line-delimited JSON parser for Claude Code's `--output-format stream-json --verbose` stream | `runs.ts` | The reference stream parser; all other parsers follow the same `{ feed, flush }` interface |
| 7 | `apps/daemon/src/lint-artifact.ts` | Anti-slop linter with P0/P1/P2 severity system | `runs.ts` | P0 findings trigger a corrective agent turn; understanding it prevents surprise re-runs |
| 8 | `apps/daemon/src/cwd-aliases.ts` | Skill staging logic; explains why `.od-skills/<id>/` copies exist instead of symlinks | `skills.ts` | Security-relevant; why the daemon copies skill assets rather than symlinking them |
| 9 | `apps/daemon/src/mcp.ts` | 8 read-only MCP tools + 3 resources; the external agent interface | `db.ts` | Shows what data is exposed to third-party agents via `od mcp` |
| 10 | `packages/contracts/src/` | Shared TypeScript types (AgentEvent, SSE events, artifact shapes) | `server.ts` | The web/daemon contract layer; changes here have cross-package impact |

---

## Reading Paths By Goal

### I want to understand how an agent is spawned

Read in this order, looking for the indicated things:

1. **`apps/daemon/src/agents.ts`** — Find the `AGENT_DEFS` array. Each entry has: `id`, `bin`, `buildArgs()`, `streamFormat`, `promptViaStdin`, optional `capabilityFlags`. Pay attention to how `buildArgs()` constructs the argv array, and which agents pass the prompt via stdin (`promptViaStdin: true`) vs. as a positional arg.
2. **`apps/daemon/src/server.ts`** — Find the `POST /api/chat` route handler. Trace how it calls `resolveAgentDef()`, invokes `stageActiveSkill()`, calls `composeSystemPrompt()`, then calls `node:child_process` `spawn()` with the resolved bin path and `buildArgs()` output. Note `stdio: ['pipe', 'pipe', 'pipe']`.
3. **`apps/daemon/src/runs.ts`** — Understand `createChatRunService()`. The spawned child process feeds stdout into the run's event stream; late-joining SSE clients replay buffered events up to `maxEvents = 2_000`.
4. **`@open-design/platform` → `createCommandInvocation()`** — On Windows, `.cmd` shim detection and `windowsVerbatimArguments` handling live here. The daemon calls this before `execFile`/`spawn` to handle cross-platform argv quoting.
5. **`apps/daemon/src/cwd-aliases.ts`** — `stageActiveSkill()` is called before spawn. It copies the active skill's directory to `<projectCwd>/.od-skills/<folderName>/` so skill side-files are reachable at a cwd-relative path inside the agent's workspace.

Key things to look for:
- `maxPromptArgBytes` on the `deepseek` adapter — the only adapter that still embeds the prompt as argv.
- `agentCapabilities` map — probed from `--help` output before the first run; gates `--include-partial-messages` and `--add-dir` flags.
- `checkWindowsDirectExeCommandLineBudget()` in `server.ts` — pre-flight that emits an SSE error instead of a cryptic `ENAMETOOLONG` if the composed prompt is too large for argv delivery.

---

### I want to understand how skills reach the agent

Read in this order:

1. **`apps/daemon/src/skills.ts`** — `listSkills(skillsRoot)` scans `skills/*/SKILL.md`, parses frontmatter via `parseFrontmatter()`, and returns a list with `id`, `mode`, `surface`, `craftRequires`, `designSystemRequired`, and `defaultFor` fields.
2. **`apps/daemon/src/frontmatter.ts`** — `parseFrontmatter()` separates YAML front matter from Markdown body. The body is injected into the system prompt; the YAML fields drive routing logic.
3. **`apps/daemon/src/craft.ts`** — `loadCraftSections(craftRequires, craftRoot)` resolves the `od.craft.requires` slug list from SKILL.md frontmatter to file contents under `craft/`. Returns a `craftBody` string and `craftSections` array consumed by `composeSystemPrompt()`.
4. **`apps/daemon/src/cwd-aliases.ts`** — `stageActiveSkill()` copies `skills/<folder>/` to `<projectCwd>/.od-skills/<folder>/` before spawn. Agents that support `--add-dir` also receive the absolute skills path as an extra allowed directory so they can read seed files directly.
5. **`apps/daemon/src/prompts/system.ts`** — `composeSystemPrompt({ skillBody, craftBody, craftSections, designSystemBody })` assembles the final string. The skill body is injected at layer 3; craft references at layer 2.5; a hard "pre-flight" rule is prepended if the skill body references `assets/template.html`.

Key things to look for:
- `SKILL_ID_ALIASES` in `skills.ts` — forwards deprecated skill ids (e.g. `editorial-collage` → `open-design-landing`).
- `derivePreflight(skillBody)` in `system.ts` — dynamically generates the "read assets/template.html FIRST" instruction.
- `designSystemRequired: data.od?.design_system?.requires ?? true` — controls whether a design system is mandatory for the skill.

---

### I want to understand the claude-stream-json protocol

Read in this order:

1. **`apps/daemon/src/claude-stream.ts`** — `createClaudeStreamHandler(onEvent)` returns `{ feed(chunk), flush() }`. It parses line-delimited JSON from Claude Code's `--output-format stream-json --verbose` stdout. Events emitted: `status`, `text_delta`, `thinking_delta`, `tool_use`, `tool_result`, `usage`, `raw`.
2. **`packages/contracts/src/sse/chat.ts`** (or equivalent contracts file) — The typed SSE event union that the daemon sends to the browser. `AgentEvent` types here mirror the handler's output.
3. **`apps/daemon/src/runs.ts`** — `emit(run, event, data)` takes the parsed agent events and distributes them to all connected SSE clients + the in-memory buffer.
4. **`apps/web/src/artifacts/parser.ts`** — Client-side streaming `<artifact>` tag extractor. Receives `text_delta` events and accumulates them; extracts artifact content when `</artifact>` closes.

Key things to look for in `claude-stream.ts`:
- `textStreamed` Set — deduplication guard so the final `assistant` wrapper doesn't re-emit text already streamed via `stream_event` deltas.
- `blocks` Map — per-content-block scratch keyed by `"${messageId}:${blockIndex}"` for accumulating `tool_use` input JSON fragments.
- `obj.type === 'system' && obj.subtype === 'init'` — the first object on the stream; carries `model` and `session_id`.
- Handling of `--include-partial-messages` being absent (older Claude Code <1.0.86): the handler emits text from the final `assistant` wrapper only when no deltas were seen.

---

### I want to understand the ACP protocol

Read in this order:

1. **`apps/daemon/src/acp.ts`** — The ACP JSON-RPC driver. Key functions: `buildAcpSessionNewParams(cwd, { mcpServers })`, `sendRpc(writable, id, method, params)`, `choosePermissionOutcome(options)`. The daemon drives the `initialize` → `session/new` → `session/prompt` lifecycle over stdin/stdout.
2. **`apps/daemon/src/agents.ts`** — Find adapters with `streamFormat: 'acp-json-rpc'`: `devin`, `hermes`, `kimi`, `kiro`, `kilo`, `vibe`. Note that ACP agents do not use `promptViaStdin: true`; the prompt is sent as a JSON-RPC `session/prompt` call after the session is established.
3. **`apps/daemon/src/runs.ts`** — For ACP agents, `run.acpSession` holds the session handle. The run's `child` process is the ACP agent subprocess; the daemon reads newline-delimited JSON from its stdout.
4. **`apps/daemon/src/agents.ts` → `detectAcpModels()`** — How the daemon probes ACP agents for their model list at startup (calls `acp`, waits for `initialize` response, reads `models` from the result, then sends `shutdown`).

Key things to look for in `acp.ts`:
- `ACP_PROTOCOL_VERSION = 1` — the version the daemon sends in `initialize`.
- `DEFAULT_TIMEOUT_MS = 15_000` and `DEFAULT_STAGE_TIMEOUT_MS = 180_000` — short vs. long timeouts for handshake vs. generation.
- `choosePermissionOutcome(options)` — prefers `approve_for_session`, then `allow_always`, then `allow_once`; never presents interactive prompts.
- `formatUsage(usage)` — normalizes camelCase ACP token counts to snake_case for the `usage` event.

---

### I want to understand prompt composition

Read in this order:

1. **`apps/daemon/src/prompts/system.ts`** — `composeSystemPrompt(input: ComposeInput): string`. This is the entry point. Read the JSDoc at the top for the 4-layer stack; then trace the `parts` array assembly.
2. **`apps/daemon/src/prompts/official-system.ts`** — `OFFICIAL_DESIGNER_PROMPT` — the identity + workflow charter (Layer 2). The longest layer; describes the agent's persona, content philosophy, and output rules.
3. **`apps/daemon/src/prompts/discovery.ts`** — `DISCOVERY_AND_PHILOSOPHY` — injected FIRST (Layer 1) so its hard rules take precedence. Contains: interactive question-form syntax, direction-picker fork, brand-spec extraction, TodoWrite reinforcement, 5-dimension critique protocol.
4. **`apps/daemon/src/prompts/deck-framework.ts`** — `DECK_FRAMEWORK_DIRECTIVE` — injected LAST for deck projects (Layer 4+). Contains the load-bearing nav/counter/scroll JS/print stylesheet contract that PDF stitching depends on.
5. **`apps/daemon/src/prompts/media-contract.ts`** — `MEDIA_GENERATION_CONTRACT` — conditional layer for image/video/audio generation projects.
6. **`apps/daemon/src/prompts/panel.ts`** — `renderPanelPrompt()` — renders the project metadata block (kind, fidelity, speaker-notes, animation intent, template snapshot).

The 4-layer stack in assembly order:

```
[1] DISCOVERY_AND_PHILOSOPHY   (discovery.ts)      — hard rules first
[2] OFFICIAL_DESIGNER_PROMPT   (official-system.ts) — identity charter
[3a] Active DESIGN.md body      (design-systems.ts) — brand tokens
[3b] craftBody                  (craft.ts)           — universal craft rules
[3c] Active SKILL.md body       (skills.ts)          — skill workflow
[4]  Metadata block             (panel.ts)           — project context
[5+] DECK_FRAMEWORK_DIRECTIVE  (deck-framework.ts)  — pinned last for decks
```

Conditional layers added by `composeSystemPrompt()`:
- **Design system** — injected only when `designSystemBody` is non-empty.
- **Craft references** — injected only when `craftBody` is non-empty.
- **Skill** — injected only when `skillBody` is non-empty; pre-flight rule prepended if body references `assets/template.html`.
- **Metadata block** — only when project metadata or template is present.
- **Deck framework** — only when `isDeckProject && !hasSkillSeed`.
- **Critique Theater** — only when `critique.enabled === true`; appended at the very end.
- **Media contract** — only for image/video/audio skill modes.

---

### I want to debug a failed generation

Read in this order, looking for specific things in each file:

1. **`apps/daemon/src/server.ts`** — Find the `POST /api/chat` handler. Look for:
   - Route-level validation errors (missing projectId, missing agentId) → these surface as 4xx responses before any SSE.
   - `checkWindowsDirectExeCommandLineBudget()` call → emits an SSE error event if prompt is too large for argv.
   - The `spawn` call and its `stdio: ['pipe', 'pipe', 'pipe']` configuration.
   - `child.on('error', ...)` and `child.on('exit', ...)` handlers — these set run status to `failed`.

2. **`apps/daemon/src/runs.ts`** — Check `run.status`. Terminal states are `succeeded`, `failed`, `canceled`. The `finish()` function emits an `end` SSE event with `{ code, signal, status }`. A non-zero `exitCode` indicates the CLI crashed or rejected an argument.

3. **`apps/daemon/src/claude-stream.ts`** (or the relevant parser for the agent in use) — Look for `onEvent({ type: 'raw', line })` — these fire when a stdout line could not be parsed as JSON. Raw lines from Claude Code often contain error messages when the CLI rejects a flag.

4. **`apps/daemon/src/agents.ts`** — Check `agentCapabilities` for the active agent. If a capability probe failed at startup, a required flag may have been silently omitted, causing the CLI to hang or error.

5. **Daemon logs** — Run `pnpm tools-dev logs --json` to see the daemon's stdout/stderr. Agent process stderr is captured by the daemon and logged. Typical failure signatures:
   - `spawn ENAMETOOLONG` / `ENOENT` — binary not on PATH or prompt too long for argv.
   - `unknown option: --include-partial-messages` — capability probe wasn't run; the flag was passed unconditionally.
   - `permission denied` — skill staging failed; the agent can't read seed files.
   - ACP `json-rpc error` — logged by `rpcErrorMessage()` in `acp.ts`.

6. **`apps/daemon/src/lint-artifact.ts`** — If the run succeeded but the artifact looks broken, check whether P0 findings triggered a corrective loop. The linter runs after each artifact write; P0 findings are fed back to the agent as a new system message.

---

### I want to add a new agent adapter

**Files to modify:**

1. **`apps/daemon/src/agents.ts`** — Add a new entry to the `AGENT_DEFS` array. Required fields: `id`, `name`, `bin`, `versionArgs`, `fallbackModels`, `buildArgs`, `streamFormat`. Optional: `promptViaStdin`, `capabilityFlags`, `listModels`, `fetchModels`, `reasoningOptions`, `maxPromptArgBytes`, `env`, `supportsImagePaths`, `mcpDiscovery`.

2. **Create a new stream parser** if the agent uses a novel output format (see the "Add a new stream parser" goal below). If the agent uses an existing format, set `streamFormat` to one of: `claude-stream-json`, `qoder-stream-json`, `acp-json-rpc`, `json-event-stream`, `copilot-stream-json`, `pi-rpc`, `plain`.

3. **`apps/daemon/src/server.ts`** — If the new `streamFormat` needs a new routing branch in the chat handler, add it there. Existing routing: `claude-stream-json` → `createClaudeStreamHandler`, `acp-json-rpc` → `runAcpSession`, `pi-rpc` → `attachPiRpcSession`, `json-event-stream` → `createJsonEventStreamHandler` (with `eventParser` sub-routing for `codex`, `gemini`, `opencode`, `cursor-agent`).

**Existing example to copy:** The `copilot` adapter is the most recent addition and is well-commented. Copy its shape, especially the `buildArgs` JSDoc explaining the `promptViaStdin` rationale.

**Required conventions:**
- `id` must be a stable kebab-case string — it is persisted in `app-config.json` and project rows.
- Every adapter must have a non-interactive / auto-approve flag so the daemon's no-TTY environment doesn't block on permission prompts.
- Prompt must be delivered via `promptViaStdin: true` unless the CLI has no stdin-pipe support (like `deepseek`). If not, set `maxPromptArgBytes`.
- Add the new `id` to `AGENT_BIN_ENV_KEYS` if the CLI binary path can be overridden via an env var.

**Tests to add:** `apps/daemon/tests/agents.test.ts` — add a test verifying `buildArgs` produces the expected argv for the common case and for model/reasoning options.

**Risks:**
- A new `streamFormat` that isn't routed in `server.ts` will silently produce no events.
- Passing a flag the installed CLI doesn't support causes an immediate non-zero exit.
- On Windows, any adapter that passes the prompt as argv must respect the ~32 KB `CreateProcess` limit.

---

### I want to add a new skill

**Files to create:**

1. **`skills/<slug>/SKILL.md`** — Required. Must have YAML front matter with at minimum: `name`, `description`, `od.mode` (one of `prototype`, `deck`, `template`, `design-system`, `image`, `video`, `audio`). Optional: `od.craft.requires` (list of craft slugs), `od.design_system.requires` (default: true), `od.surface`, `od.platform`, `od.scenario`, `od.preview.type`, `od.default_for`, `triggers`.

2. **`skills/<slug>/assets/template.html`** — Optional but recommended. The seed HTML that the agent uses as its starting point. Its presence causes `composeSystemPrompt()` to inject a "read this BEFORE writing any code" pre-flight rule automatically.

3. **`skills/<slug>/references/*.md`** — Optional. Side-files (layouts.md, checklist.md, etc.) that the agent can read via the staged `.od-skills/<slug>/references/` path.

**Frontmatter fields (full reference):**

```yaml
---
name: my-skill-id           # required; also used as skill id
description: "One-line description"
triggers:
  - keyword1
  - keyword2
od:
  mode: prototype           # prototype | deck | template | design-system | image | video | audio
  surface: web              # web | mobile | print | [Uncertain: full list]
  platform: browser         # browser | desktop | [Uncertain]
  scenario: landing-page    # free-form scenario tag
  craft:
    requires:
      - typography           # slug under craft/ folder
      - anti-slop
  design_system:
    requires: true           # whether a design system binding is mandatory
  preview:
    type: html               # html | image | video | audio
  default_for:               # when to auto-select this skill
    - kind: prototype
  upstream: parent-skill-id  # optional: inherits from another skill
---
```

**Tests to add:** No automated test mechanism was found for skill frontmatter validation. Manual: start the daemon, call `GET /api/skills`, verify the new skill appears with the expected `id`, `mode`, and `craftRequires`.

---

### I want to review security-sensitive code

Read these files for each security concern:

**Origin / host validation (DNS rebinding prevention):**
- `apps/daemon/src/origin-validation.ts` — `isPrivateIpv4()`, `isLoopbackOrPrivateLanHost()`, `configuredAllowedOrigins()`, `allowedBrowserPorts()`.
- `apps/daemon/src/server.ts` — `requireLocalDaemonRequest()` middleware; called on every mutating route. Rejects requests from unexpected origins/hosts.

**SSRF in BYOK proxy:**
- `apps/daemon/src/server.ts` — The `POST /api/proxy/<provider>/stream` route. Look for the DNS + IP validation using `isLoopbackOrPrivateLanHost()` + `isPrivateIpv4()` before the outbound HTTP request.
- `apps/daemon/src/origin-validation.ts` — The blocking primitives; read `isPrivateIpv4()` to verify coverage of 10.x, 172.16-31.x, 192.168.x, 169.254.x.

**API key storage:**
- `apps/daemon/src/media-config.ts` — Reads/writes `.od/media-config.json`. Keys are stored in the filesystem data directory, not in SQLite. Risk: if `OD_DATA_DIR` points to a shared location, all processes can read the keys.

**Skill staging (write-amplification prevention):**
- `apps/daemon/src/cwd-aliases.ts` — `stageActiveSkill()` uses `cp` with `dereference: true`, never `symlink()`. This prevents agents from writing through a staged path back to the live `skills/` tree.

**MCP server (information exposure):**
- `apps/daemon/src/mcp.ts` — 8 read-only tools. Review what each tool can return (project files, conversation history, artifact HTML). The MCP server runs over stdio; there is no network exposure, but the calling agent receives project file contents.

**Agent spawn (command injection):**
- `apps/daemon/src/agents.ts` — `sanitizeCustomModel()` is called on user-supplied model IDs to prevent shell injection via model string. The prompt text itself is passed to the agent (trusted), not interpolated into a shell command string. All spawns use `execFile`/`spawn` (not `exec`), so the shell is never involved.

**SQLite concurrent writes:**
- `apps/daemon/src/db.ts` — WAL mode is enabled, which reduces but does not eliminate contention. A single daemon process serializes writes; risk is mainly from external tools writing to `app.sqlite` concurrently.

**iframe sandbox (generated code):**
- `apps/web/src/` — Generated HTML is rendered in an `<iframe srcdoc="...">` sandbox. Exact CSP header settings for the srcdoc frame were not verified during this documentation pass. `[Uncertain]`
