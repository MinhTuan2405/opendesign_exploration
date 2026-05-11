# Open Design — Agentic Platform Deep Dive

## What Kind of Agentic Platform Is This?

Open Design is an **agent harness**, not a model provider. It does not train models, host inference, or own any AI runtime. Instead it acts as an orchestration layer that routes a user's design intent to whichever locally-installed AI CLI is available, composes a multi-layer system prompt from modular design knowledge (skills + design systems + craft rules), translates each CLI's proprietary stdout format into a unified SSE event stream, and delivers the result to a browser UI.

The core delegation philosophy is: **the AI CLI owns the agentic loop; Open Design owns the context**. When a user sends a message, the daemon composes a prompt that encodes the full design brief, brand identity, and workflow rules, then hands that prompt to the CLI agent. The agent decides how many tool calls to make, what files to read/write, and when to stop. The daemon watches the stream, parses events, and surfaces them to the UI — but never interrupts or steers the agent mid-turn.

This design has a significant architectural implication: Open Design can work with any CLI that speaks a supported stream format, regardless of which underlying model it uses. Adding a new agent adapter requires implementing one `buildArgs()` function and one stream parser — the prompt composition pipeline, the skill system, and the entire UI are inherited automatically.

**Local vs cloud AI design tools:** Unlike Figma AI or Adobe Firefly, Open Design runs entirely on the user's machine. The daemon binds to `127.0.0.1`, the SQLite database is local, and agent CLIs run as local child processes. There is no server-side model inference. Cloud connectivity exists only for BYOK API proxy routes (optional), deployment to Cloudflare Pages/Vercel (optional), and Orbit activity summaries (optional).

---

## The 16 Agent Adapters

| Agent | Binary | Stream Format | Protocol | Prompt Delivery | Skill Injection | Capabilities |
|---|---|---|---|---|---|---|
| Claude Code | `claude` | `claude-stream-json` | stdio JSONL | stdin pipe | `.od-skills/` copy + `--add-dir` | thinking blocks, tool call panel, partial streaming, `--include-partial-messages`, `--add-dir` (capability-gated) |
| Codex CLI | `codex` | `json-event-stream` (eventParser: codex) | stdio JSONL | stdin pipe | `.od-skills/` copy + `--add-dir` | tool calls, reasoning effort (`none`…`xhigh`), plugin disable, image gen override |
| Cursor Agent | `cursor-agent` | `json-event-stream` (eventParser: cursor-agent) | stdio JSONL | stdin pipe | `.od-skills/` copy + `.cursorrules` | tool calls, thinking (reasoning_delta), model picker via `cursor-agent models` |
| Gemini CLI | `gemini` | `json-event-stream` (eventParser: gemini) | stdio JSONL | stdin pipe | prompt injection | `--yolo` non-interactive, workspace trust via env var |
| OpenCode | `opencode` | `json-event-stream` (eventParser: opencode) | stdio JSONL | stdin pipe (`opencode run -`) | prompt injection | dynamic model list (`opencode models`), multi-provider |
| Qwen Code | `qwen` | `plain` | stdio raw | stdin pipe (`qwen -`) | prompt injection | `--yolo` non-interactive; raw stdout only |
| Qoder CLI | `qodercli` | `qoder-stream-json` | stdio JSONL | stdin pipe | prompt injection + `--add-dir` + `--attachment` | image attachments, workspace flag `-w`, model picker, thinking blocks |
| GitHub Copilot CLI | `copilot` | `copilot-stream-json` | stdio JSONL | stdin pipe (no `-p`) | prompt injection + `--add-dir` | tool calls, reasoning_delta (thinking), `--allow-all-tools` |
| Devin for Terminal | `devin` | `acp-json-rpc` | ACP JSON-RPC | `session/prompt` RPC | ACP MCP bridge | ACP model detection, permission auto-approve, graceful abort |
| Hermes | `hermes` | `acp-json-rpc` | ACP JSON-RPC | `session/prompt` RPC | ACP MCP bridge + Live Artifacts MCP server | `mcpDiscovery: 'mature-acp'`, model detection |
| Kimi CLI | `kimi` | `acp-json-rpc` | ACP JSON-RPC | `session/prompt` RPC | ACP MCP bridge + Live Artifacts MCP server | `mcpDiscovery: 'mature-acp'`, model detection |
| Kiro CLI | `kiro-cli` | `acp-json-rpc` | ACP JSON-RPC | `session/prompt` RPC | ACP MCP bridge | ACP model detection |
| Kilo | `kilo` | `acp-json-rpc` | ACP JSON-RPC | `session/prompt` RPC | ACP MCP bridge | ACP model detection |
| Mistral Vibe CLI | `vibe-acp` | `acp-json-rpc` | ACP JSON-RPC | `session/prompt` RPC | ACP MCP bridge | ACP model detection (no args) |
| Pi | `pi` | `pi-rpc` | Custom stdio JSON-RPC | `prompt` RPC command | Prompt injection + `--append-system-prompt` | thinking levels, image forwarding (base64, max 10 / 20MB), multi-provider, `--list-models` via stderr |
| DeepSeek TUI | `deepseek` | `plain` | stdio raw | positional argv (guarded by 30KB budget) | prompt injection | `exec --auto`, plain stdout streaming only |

### Detection Method (per agent)

All agents: `resolveAgentExecutable(def, configuredEnv)` checks `AGENT_BIN_ENV_KEYS` for an explicit binary override env var, then walks `resolveOnPath(bin)` across `process.env.PATH` + `userToolchainDirs()`. For Claude Code, the fallback binary `openclaude` is tried after `claude`. A 3-second `--version` probe confirms the binary is executable; capability flags are probed from `--help` output (Claude Code only, due to `--add-dir` and `--include-partial-messages` being subcommand flags under `claude -p --help`).

### Known Limitations (per agent)

- **Claude Code**: `--add-dir` and `--include-partial-messages` are only passed if the installed CLI advertises them in `--help`; older builds get the safe baseline.
- **Codex CLI**: No `-` stdin sentinel for stdin; uses pipe-only stdin delivery. Plugin disable via `OD_CODEX_DISABLE_PLUGINS=1`. Reasoning effort clamped per model family (`clampCodexReasoning()`).
- **Cursor Agent**: `cursor-agent models` may return "No models available" when unauthenticated; silently falls back to static hint list.
- **Gemini CLI**: `--skip-trust` flag rejected by some builds; workspace trust delivered via env var instead.
- **Qwen Code**: `plain` stream format — no structured tool call panel in the UI; tool output appears as text.
- **DeepSeek TUI**: No stdin sentinel; full composed prompt as positional argv. Hard 30KB budget cap before spawn.
- **Pi**: `--list-models` prints to stderr (not stdout); fetchModels reads `stderr` specifically. Extra-allowed dirs passed as `--append-system-prompt` hints (no sandbox flag).
- **ACP agents (Devin/Hermes/Kimi/Kiro/Kilo/Vibe)**: Stage timeout 180s; slow initializations on first run with large contexts may hit this. Permission auto-approve fires on every `permission_request` — the agent cannot prompt the user for selective approval.

---

## How Skills Are Called

Step-by-step from the UI's skill picker to agent execution:

**Step 1 — User selects a skill in the new-project form or changes skill on an existing project.**
The web client stores `skill_id` in the project record via `PATCH /api/projects/:id`. The daemon calls `updateProject(db, id, { skillId })`.

**Step 2 — User opens a project and starts a new conversation turn.**
The web client sends `POST /api/chat` (or `POST /api/runs`) with a request body that includes `projectId`, `conversationId`, `agentId`, `model`, and the user's message.

**Step 3 — `startChatRun()` in `server.ts` resolves the active skill.**
It calls `listSkills(SKILLS_DIR)` to get all skills, then `findSkillById(skills, project.skillId)` — routing deprecated IDs through `SKILL_ID_ALIASES` first.

**Step 4 — Craft sections are loaded.**
`loadCraftSections(skill.craftRequires, CRAFT_DIR)` reads `craft/<slug>.md` files listed in `skill.od.craft.requires` and concatenates them.

**Step 5 — The active design system is loaded.**
`readDesignSystem(DESIGN_SYSTEMS_DIR, project.designSystemId)` reads `design-systems/<id>/DESIGN.md` and returns its body.

**Step 6 — The system prompt is composed.**
`composeSystemPrompt({ agentId, skillBody, skillName, skillMode, designSystemBody, craftBody, metadata, ... })` assembles all 10 layers into a single string. `derivePreflight(skillBody)` injects the forced-Read-tool preamble if the skill ships a seed template or reference docs.

**Step 7 — The skill is staged into the project cwd.**
`stageActiveSkill(projectCwd, skill.id, skill.dir, log)` copies the skill folder to `<projectCwd>/.od-skills/<skill.id>/`. If staging fails, the absolute path fallback in the preamble is used.

**Step 8 — Extra allowed dirs are computed.**
`resolveChatExtraAllowedDirs({ agentId, skillsDir, designSystemsDir, linkedDirs })` returns the list of directories that need to be added to the agent's sandbox. For Codex, this is the `codexGeneratedImagesDir` instead.

**Step 9 — The agent is spawned.**
`def.buildArgs(prompt, imagePaths, extraAllowedDirs, options, runtimeContext)` produces the argv. `child_process.spawn(resolvedBin, argv, { cwd: projectDir, stdio: ['pipe','pipe','pipe'], env: spawnEnv })` starts the agent. The composed system prompt is written to the agent's stdin (for `promptViaStdin: true` agents) or embedded as an argv positional (DeepSeek).

### Files Involved

| File | Role |
|---|---|
| `apps/daemon/src/skills.ts` | `listSkills()`, `findSkillById()`, `withSkillRootPreamble()` |
| `apps/daemon/src/prompts/system.ts` | `composeSystemPrompt()`, `derivePreflight()` |
| `craft/` | Universal brand-agnostic craft rule files (e.g. `craft/anti-ai-slop.md`) |
| `skills/<name>/SKILL.md` | Skill body + YAML frontmatter |
| `skills/<name>/assets/template.html` | Skill seed template (triggers forced-Read-tool preamble) |
| `skills/<name>/references/*.md` | Skill reference docs (checklists, layout guides) |

---

## How the Prompt Is Assembled (4-Layer Stack + Conditionals)

Source: `apps/daemon/src/prompts/system.ts`

```
Layer 1: DISCOVERY_AND_PHILOSOPHY      (apps/daemon/src/prompts/discovery.ts)
         - Mandatory first; hard rules win precedence over softer wording below
         - Interactive question-form syntax, brand-spec extraction
         - Direction-picker fork (directions.ts library embedded)
         - TodoWrite reinforcement, 5-dimension critique guidance

Layer 2: OFFICIAL_DESIGNER_PROMPT      (apps/daemon/src/prompts/official-system.ts)
         - Full designer identity, workflow charter, and content philosophy
         - Separated by "# Identity and workflow charter (background)" heading

Layer 3: Active DESIGN.md body         (design-systems/<id>/DESIGN.md)
         - Authoritative color, typography, spacing, component rules
         - Only present if project has an active design system
         - Agent is instructed: "Do not invent tokens outside this palette"

Layer 4: Active SKILL.md body          (skills/<id>/SKILL.md)
         - Workflow specific to the artifact kind
         - Prefixed by derivePreflight() preamble if skill ships seed/references
```

### Conditional Layers

| Layer | Condition |
|---|---|
| Craft references block | Skill's `od.craft.requires` contains at least one slug that resolves to a file in `craft/` |
| Metadata block | Project has `metadata` (kind, fidelity, speakerNotes, etc.) or a bound `template` |
| Deck framework directive | `skillMode === 'deck'` OR `metadata.kind === 'deck'`, AND skill body does NOT reference `assets/template.html` |
| Media generation contract | `skillMode` or `metadata.kind` is `'image'`, `'video'`, or `'audio'` |
| Codex imagegen override | `agentId === 'codex'` AND `metadata.kind === 'image'` AND a `gpt-image-*` model is selected |
| Critique Theater panel prompt | `critique.enabled === true` AND `critiqueBrand` AND `critiqueSkill` present AND NOT a media surface |

### Layer Ordering Rationale

Discovery goes first because it contains hard rules ("emit a form on turn 1") that must win over softer wording in the official prompt. The deck framework goes last because it overrides any earlier softer slide-handling wording with the load-bearing nav/counter/print contract. The critique addendum goes last of all so it overrides any earlier softer critique wording.

---

## How Agent Output Is Parsed

All parsers emit events through a common `onEvent({ type, ...payload })` callback, which ultimately calls `design.runs.emit(run, 'agent', event)` to fan out to all SSE clients.

### claude-stream-json

Source: `apps/daemon/src/claude-stream.ts`

Line-delimited JSON from `claude -p --output-format stream-json --verbose [--include-partial-messages]`.

```
stdout line → JSON.parse() → handleObject()
  system+init          → { type:'status', label:'initializing', model, sessionId }
  system+status        → { type:'status', label: obj.status }
  stream_event         → handleStreamEvent()
    content_block_start  (thinking block) → { type:'thinking_start' }
    content_block_delta  (text_delta)     → { type:'text_delta', delta }
    content_block_delta  (thinking_delta) → { type:'thinking_delta', delta }
    content_block_delta  (input_json)     → accumulate in blocks Map
  assistant            → emit tool_use for each tool_use block
                         emit text_delta if NOT already in textStreamed Set
  user (tool_result)   → { type:'tool_result', tool_use_id, content, is_error }
  result               → { type:'usage', usage, cost, stopReason }
```

The `textStreamed` Set prevents duplicate text emission: streaming deltas land with `--include-partial-messages`; older builds deliver text only in the final `assistant` wrapper.

### acp-json-rpc

Source: `apps/daemon/src/acp.ts`

JSON-RPC 2.0 over stdio. The daemon drives the entire lifecycle:

```
1. spawn agent
2. send  initialize { protocolVersion:1, clientCapabilities:{terminal:false} }
3. recv  result (id:1) → send session/new { cwd, mcpServers }
4. recv  result (id:2) → if model: send session/set_model { modelId }
5. recv  result (model id) → send session/prompt { prompt }
6. recv  session/update notifications → map to UI events:
           content.type=text       → text_delta
           content.type=thinking   → thinking_delta
           tool_use                → tool_use
           tool_result             → tool_result
           status updates          → status
7. recv  session/complete → finish stream
```

Permission requests (`permission_request` method) are auto-approved: the daemon selects `approve_for_session` if offered, else `allow_always`, else `allow_once`. The web UI has no surface for interactive approval prompts.

Abort path: `run.acpSession.abort()` → graceful shutdown signal to ACP agent → `PI_ABORT_GRACE_MS` (3s) timeout → `SIGTERM`.

### pi-rpc

Source: `apps/daemon/src/pi-rpc.ts`

Custom JSON-RPC over stdio (not standard ACP). Pi uses its own event taxonomy.

```
1. spawn pi --mode rpc [--model ...] [--thinking ...]
2. send { type:'prompt', id:1, prompt:'...', images:[{data:base64, mimeType},...] }
3. recv events on stdout:
     agent_start      → { type:'status', label:'working' }
     turn_start       → { type:'status', label:'thinking' }
     message_update   → { type:'text_delta', delta } or { type:'thinking_delta', delta }
     tool_use         → { type:'tool_use', id, name, input }
     tool_result      → { type:'tool_result', ... }
     agent_end        → finish stream → usage
4. Extension UI dialogs auto-resolved; fire-and-forget methods silently consumed
```

Image forwarding: Images attached to the run are base64-encoded and embedded in the `prompt` command's `images` array. Max 10 images, 20MB total, `.png/.jpg/.jpeg/.gif/.webp` only.

### plain

Source: stdout forwarding in `server.ts`

No parser. Raw stdout bytes forwarded as `{ type:'text_delta', delta: chunk }` events. Used by Qwen Code and DeepSeek TUI. Tool calls appear inline as text; no structured tool call panel in the UI.

### json-event-stream

Source: `apps/daemon/src/json-event-stream.ts`

Shared JSONL parser dispatched to four sub-handlers by `eventParser` field.

**Codex** events: `thread.started` (status), `turn.started` (status), `item` (text/tool_use/tool_result), `turn.completed` (usage).

**Gemini** events: `gemini_response` with content parts → text_delta; tool_use; usage on `result`.

**OpenCode** events: `step_start` (status), `text` (text_delta), `tool_use` with `callID` (tool_use), `tool_result` (tool_result), `session_complete` with tokens (usage).

**Cursor Agent** events: Similar to Gemini; uses `--output-format stream-json --stream-partial-output`.

---

## Windows ENAMETOOLONG Fallbacks

Windows `CreateProcess` caps `lpCommandLine` at approximately:
- ~32 KB for direct `.exe` invocations
- ~8 KB when the binary resolves to a `.cmd`/`.bat` shim (common with npm-installed CLIs)

Any non-trivial Open Design prompt (skill + design system + craft context + user message) easily exceeds these limits.

### Fallback Chain

For agents with `promptViaStdin: true` (all except DeepSeek): the prompt is not in argv, so the budget check is irrelevant. The daemon pipes the full prompt to stdin with no length cap.

For agents without stdin support (DeepSeek only): `checkPromptArgvBudget(def, composed)` runs before spawn. If `Buffer.byteLength(composed, 'utf8') > def.maxPromptArgBytes` (30,000 for DeepSeek), the daemon emits an actionable SSE error: "Reduce the selected skills/design-system context, shorten the conversation, or pick an adapter with stdin support." The spawn never fires.

For all agents on Windows, before spawn:
1. `checkWindowsCmdShimCommandLineBudget(resolvedBin, argv)` — if the binary ends in `.cmd`/`.bat`, estimates cmd.exe quoting overhead.
2. `checkWindowsDirectExeCommandLineBudget(resolvedBin, argv)` — estimates CreateProcess budget for `.exe`.
3. If over budget and the agent is NOT `promptViaStdin`, the prompt arg is replaced with a temp file redirect:
   - Temp file written to `<projectCwd>/.od-prompt-<timestamp>-<random>.md`
   - Prompt arg replaced with `promptFileBootstrap(fp)`: a short instruction telling the agent to open and read that file first
   - Temp file deleted after child exits

---

## Capability-Driven UI Degradation

The `available`, `streamFormat`, and `capabilityFlags` from agent detection drive UI feature availability:

| Capability | Source | UI Effect When Absent |
|---|---|---|
| `available: false` | Binary not on PATH | Agent greyed out in picker |
| `partialMessages` cap | `--include-partial-messages` in `claude -p --help` | No incremental text streaming for Claude (text arrives only at turn end) |
| `addDir` cap | `--add-dir` in `claude -p --help` | Skill/design-system dirs not added to Claude's sandbox; agent may fail to read skill assets |
| `streamFormat: 'plain'` | Qwen, DeepSeek | No tool call panel, no thinking blocks panel — text only |
| `streamFormat: 'acp-json-rpc'` | ACP agents | ACP session panel shown; MCP server injection available for `mcpDiscovery: 'mature-acp'` agents |
| `streamFormat: 'pi-rpc'` | Pi | Image attachment forwarding enabled |
| `reasoningOptions` | Codex, Pi | Reasoning effort dropdown shown in model picker |
| `models` array length | All | Model picker populated; fallback to static hints if dynamic listing fails |

---

## BYOK Fallback Path (No CLI Installed)

When no AI CLI is installed or the user prefers direct API access, the daemon serves as an API proxy. The BYOK path uses one of four routes:

```
POST /api/proxy/anthropic/stream
POST /api/proxy/openai/stream
POST /api/proxy/azure/stream
POST /api/proxy/google/stream
```

Each route:
1. Validates `baseUrl`, `apiKey`, `model` are present.
2. Calls `validateExternalApiBaseUrl(baseUrl)` to block SSRF (private IPs, loopback).
3. Forwards the request to the upstream API with the user-supplied API key.
4. Streams the upstream SSE response, normalizing all providers to: `start` (model), `delta` (text chunk), `end`, `error` events.
5. Redacts Bearer tokens from any error messages before logging.

**Own tool loop:** In BYOK mode, the daemon implements its own multi-turn tool loop for skills that require tool use (e.g. reading seed template files). The web client handles the tool call/result cycle against the daemon's `/api/tools/*` endpoints using the `OD_TOOL_TOKEN` scoped per-run token.

**Prompt injection replaces native skill loading:** Since there is no CLI agent to inject `.od-skills/` into, the full skill body (including preamble and seed template content) is embedded directly in the system prompt. The `--add-dir` pattern is replaced by copying file content inline.

---

## The MCP Server (Outbound Tool Bridge)

Source: `apps/daemon/src/mcp.ts`

Invoked as `od mcp` (stdio transport). Lets a coding agent in a different repository (Claude Code, Cursor, Zed) pull files from a local Open Design project without the export-zip-import dance.

### Architecture

```
External coding agent
  ↓ stdio JSON-RPC (MCP protocol)
od mcp (stdio MCP server)
  ↓ HTTP fetch()
Local daemon (:17456 or configured port)
  ↓ SQL / filesystem
.od/app.sqlite + .od/projects/<id>/
```

The MCP server is stateless — it holds no state and never touches the filesystem directly. Every tool call resolves to a `fetch()` against `OD_DAEMON_URL`. If the daemon is not running, tool calls return a "daemon not reachable" error; the server itself still launches so the client can list its tool schema.

### 8 Tools

| Tool | Key Behavior |
|---|---|
| `list_projects` | Returns all projects with id, name, skill_id, metadata |
| `get_active_context` | Returns `{ active: true/false, projectId?, fileName? }` — active context has 5-min TTL |
| `get_artifact` | BFS traversal of entry file refs (`<script>`, `<link>`, `<img>`, `@import`, `url()`) up to depth 3; soft cap 1.5MB total, max 200 files; `include='all'` returns every project file |
| `get_project` | Project metadata by UUID or name |
| `get_file` | File content with pagination: up to 2000 lines starting at `offset`; appends `[od:file-window line=N total=M]` marker when truncated |
| `search_files` | Full-text search via `searchProjectFiles()` |
| `list_files` | File listing with names, sizes, mime types |
| `list_comments` | [Uncertain — based on research summary count of 8 tools] |

### 3 Resources

| URI | Content |
|---|---|
| `od://design-systems/<id>/DESIGN.md` | Design system DESIGN.md body |
| `od://skills/<id>/SKILL.md` | Skill SKILL.md body |
| `od://focus/active` | Current active context (project + file, 5-min TTL) |

### Read-Only Contract

All tools: `readOnlyHint: true`, `idempotentHint: true`, `openWorldHint: false`. No write tool is exposed. The MCP server cannot modify the daemon's state. This makes it safe to grant MCP access to any external agent without worrying about it corrupting the project state.

### Project Discovery

UUID → exact match. Name: exact match first → slug match → substring match. If multiple projects match a substring, returns an error asking the caller to use a more specific name or the UUID.

---

## Agentic Loop Lifecycle (Full Turn)

A complete 23-step walk-through from user clicking Send to turn close:

1. **User types message** and clicks Send in the web UI.
2. **Web client sends `POST /api/chat`** with `{ projectId, conversationId, agentId, model, reasoning?, message, imagePaths?, skillId?, designSystemId?, metadata? }`.
3. **`startServer()`'s `/api/chat` handler** calls `design.runs.create()` → new run with status `'queued'`.
4. **`design.runs.stream(run, req, res)`** sets up the SSE response immediately, replaying any existing events from `run.events`.
5. **`design.runs.start(run, () => startChatRun(body, run))`** schedules `startChatRun` asynchronously; run transitions to `'starting'`.
6. **`startChatRun()`** begins: fetches project from SQLite, resolves conversation, creates/updates assistant message record.
7. **Skill resolution**: `listSkills(SKILLS_DIR)` → `findSkillById(skills, skillId)`.
8. **Design system resolution**: `readDesignSystem(DESIGN_SYSTEMS_DIR, designSystemId)`.
9. **Craft resolution**: `loadCraftSections(skill.craftRequires, CRAFT_DIR)`.
10. **Prompt composition**: `composeSystemPrompt(...)` → single composed string.
11. **Skill staging**: `stageActiveSkill(projectCwd, skill.id, skill.dir)` → copies skill to `.od-skills/<id>/`.
12. **Extra allowed dirs**: `resolveChatExtraAllowedDirs(...)` → list of dirs to pass to `--add-dir` (or equivalent).
13. **Tool token**: `toolTokenRegistry.create({ runId, projectId, ... })` → scoped `OD_TOOL_TOKEN`.
14. **Agent binary resolution**: `resolveAgentBin(def, configuredEnv)` → absolute path.
15. **Argv construction**: `def.buildArgs(prompt, imagePaths, extraAllowedDirs, options, runtimeContext)`.
16. **Prompt budget check**: `checkPromptArgvBudget(def, composed)` for argv-delivered agents; Windows budget checks.
17. **Environment construction**: `createAgentRuntimeEnv(baseEnv, daemonUrl, toolTokenGrant, nodeBin)` + `spawnEnvForAgent(agentId, ...)`.
18. **`child_process.spawn()`**: Agent process starts; cwd = project dir; stdin/stdout/stderr all piped.
19. **Prompt delivery**: For `promptViaStdin: true` agents, write composed prompt to `child.stdin` then `child.stdin.end()` (or for pi-rpc: send `{ type:'prompt', prompt, images }` RPC).
20. **Stream parser attaches**: `createClaudeStreamHandler(onEvent)` / `attachAcpSession(...)` / `attachPiRpcSession(...)` / etc., each calling `onEvent` which calls `design.runs.emit(run, 'agent', event)`.
21. **SSE fan-out**: Each `emit()` call writes to all connected `run.clients` SSE connections and appends to `run.events[]` (max 2000).
22. **Child exits**: `child.on('close', (code, signal))` → artifact lint runs if an artifact was saved during the turn; `design.runs.finish(run, status, code, signal)` sends `end` SSE event and closes all client connections.
23. **Message persisted**: The assistant message record is updated in SQLite with `run_status`, `events_json`, `produced_files_json`, `ended_at`.

---

## Security Model

### Shell Spawn Sanitization

- `sanitizeCustomModel(id)`: Alphanumeric + `._/:@-`, max 200 chars. Prevents shell injection via model IDs.
- `checkPromptArgvBudget()`: Hard byte budget prevents argv overflow attacks (or accidental prompt bloat crashing the OS).
- `isSafeAliasSegment(folderName)` in `cwd-aliases.ts`: Rejects path traversal characters (`..`, `/`, `\`, null bytes) before creating `.od-skills/<name>/`. Prevents a malicious skill ID from escaping the project cwd.
- Skill staging uses `{ dereference: true }` in `fs.cp()`: Symlinks in the skill source tree are dereferenced. A compromised skill cannot create a symlink that points outside the skill directory; agents writing to `.od-skills/` cannot modify the source skill.

### SSRF Blocking

`origin-validation.ts` is called by all BYOK proxy routes:
- `isPrivateIpv4(hostname)`: blocks 10/8, 172.16/12, 192.168/16, 169.254/16.
- Loopback hostnames (`localhost`, `127.0.0.1`, `::1`) are also blocked for external API targets.
- HTTP 403 `FORBIDDEN` returned for private targets; not a 400 (which could leak info).
- `configuredAllowedOrigins()` for reverse proxy setups reads `OD_ALLOWED_ORIGINS`; only http/https origins accepted.

### API Key Storage

API keys stored at `.od/media-config.json` (gitignored; never committed). Never stored in SQLite columns. `readMaskedConfig()` redacts key values (returns `'***'`) for `GET /api/media/config`. `spawnEnvForAgent()` strips `ANTHROPIC_API_KEY` from the Claude agent's env unless `ANTHROPIC_BASE_URL` is set (prevents the key leaking to a BYOK path that doesn't need it).

### srcdoc Sandbox

Artifact previews served at `GET /api/projects/:id/files/:name/preview` wrap the generated HTML in `<iframe srcdoc="..." sandbox="allow-scripts allow-same-origin">`. Generated code cannot reach parent frame JavaScript, cookies, localStorage, or IndexedDB. This prevents a malicious artifact from exfiltrating user data via the preview.

### MCP Read-Only

All 8 MCP tools carry `readOnlyHint: true`. The MCP server only makes GET requests to the daemon. No write path exists in the MCP server. External agents granted MCP access cannot modify projects, delete files, or trigger agent runs.

### Loopback Enforcement

`requireLocalDaemonRequest(req, res, next)` is applied to sensitive routes:
1. Checks `req.socket.remoteAddress` is loopback (127.x.x.x, ::1, ::ffff:127.x.x.x).
2. Checks `Host` header is loopback or in `configuredAllowedHosts()`.
3. Checks `Origin` header (if present) passes `isAllowedBrowserOrigin()`.

HTTP 403 returned for any failure. This prevents a remote site loaded in the user's browser from sending cross-origin requests to the daemon via `fetch()`.

---

## Extending the Agentic Platform

### Adding a New Agent Adapter

1. Add a new entry to `AGENT_DEFS` in `apps/daemon/src/agents.ts` with all required fields: `id`, `name`, `bin`, `versionArgs`, `fallbackModels`, `buildArgs`, `streamFormat`.
2. If the agent has a novel stream format, implement a new parser (e.g. `apps/daemon/src/myagent-stream.ts`) following the `createMyAgentStreamHandler(onEvent)` pattern.
3. Wire the new parser into the `startChatRun()` stream dispatch switch in `server.ts`.
4. If the agent supports ACP, set `streamFormat: 'acp-json-rpc'` and optionally `mcpDiscovery: 'mature-acp'` — no new parser needed.
5. If the agent needs a custom binary override env var, add it to `AGENT_BIN_ENV_KEYS`.
6. Run `pnpm --filter @open-design/daemon test` to verify the agent detection tests pass.

### Adding a New Skill

1. Create `skills/<my-skill-name>/SKILL.md` with the required YAML frontmatter:
   ```yaml
   ---
   name: my-skill-name
   description: One-line description
   triggers:
     - keyword phrase for auto-suggest
   od:
     mode: prototype          # prototype|deck|template|design-system|image|video|audio
     surface: web             # web|presentation|document|image|video|audio
     craft:
       requires:
         - anti-ai-slop       # optional craft slugs from craft/
     design_system:
       requires: true
     featured: true
     fidelity: high
   ---
   ```
2. Write the skill body below the frontmatter — this becomes the system prompt injection.
3. Optionally add `assets/template.html` (seed template) and `references/*.md` (checklists/guides). The `withSkillRootPreamble()` function automatically detects these and injects the forced-Read-tool preamble.
4. The daemon rescans `SKILLS_DIR` on every `GET /api/skills` request (no restart needed).

### Adding a New Craft Reference

1. Create `craft/<slug>.md` with the craft rules.
2. Reference it from a skill's `od.craft.requires` list or from the DISCOVERY_AND_PHILOSOPHY prompt for universal application.
3. `loadCraftSections(skillCraftRequires, CRAFT_DIR)` reads and concatenates these files automatically.

### Adding a New BYOK Provider

1. Add a new route `app.post('/api/proxy/<provider>/stream', ...)` following the pattern in the existing four proxy routes.
2. Call `validateExternalApiBaseUrl(baseUrl)` before any fetch — this is mandatory.
3. Normalize the upstream SSE to: `start` (model), `delta` (text chunk), `end`, `error` events.
4. Add the provider to the `protocol` validation in `POST /api/test/connection`.
5. Update `connectionTest.ts` to handle the new provider in `testProviderConnection()`.
