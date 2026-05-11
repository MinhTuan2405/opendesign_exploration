# Extension Points

> Concrete playbook for each of the 9 supported extension patterns in the Open Design daemon.
> Each section follows the same structure: when to use, files to modify, example to copy, required conventions, tests to add, risks, and a validation checklist.

---

## Extension Point 1: Adding a New Agent Adapter

### When to use

When you want Open Design to drive a new coding-agent CLI that is not already in `AGENT_DEFS` (e.g. a new open-source agent fork, a commercial tool with a compatible CLI, or an internal agent).

### Files to modify

| File | What to do |
|------|-----------|
| `apps/daemon/src/agents.ts` | Add one entry to the `AGENT_DEFS` array |
| `apps/daemon/src/server.ts` | Add a routing branch for the new `streamFormat` in the `POST /api/chat` handler, if the format is new |
| `apps/daemon/src/app-config.ts` | Optionally add to `AGENT_BIN_ENV_KEYS` if the binary path can be overridden via an env var |
| `packages/contracts/src/` | Add the new agent id to any discriminated union or agent-id type that exists there |

### Existing examples to copy

- **`copilot`** — most recently added, best-commented. Shows `promptViaStdin: true`, stdin-pipe rationale JSDoc, `buildArgs` with `--add-dir`, and `fallbackModels`.
- **`kimi`** — minimal ACP adapter. Shows `fetchModels` via `detectAcpModels`, `buildArgs: () => ['acp']`, `streamFormat: 'acp-json-rpc'`.
- **`deepseek`** — the only argv-prompt adapter; shows `maxPromptArgBytes` and `streamFormat: 'plain'`.

### Required conventions

1. `id` must be a stable, unique, lowercase kebab-case string. It is stored in `app-config.json` and in SQLite project rows — changing it after release breaks existing projects.
2. Every adapter must include a non-interactive / auto-approve flag in `buildArgs()`. The daemon runs every CLI without a TTY; interactive permission prompts will hang the run indefinitely.
3. Prefer `promptViaStdin: true`. Only omit it when the CLI has no stdin-pipe support (the `deepseek` case). If the prompt must go via argv, set `maxPromptArgBytes` (recommended: 30,000).
4. `fallbackModels` must always start with `DEFAULT_MODEL_OPTION` (`{ id: 'default', label: 'Default (CLI config)' }`).
5. `streamFormat` must match one of the existing formats or a newly added one. Never silently use a format string that has no corresponding handler in `server.ts`.
6. If the CLI emits output on stderr that is important for debugging, the daemon's spawn `stdio` config must be `['pipe', 'pipe', 'pipe']` to capture it.

### Tests to add

- `apps/daemon/tests/agents.test.ts` — Verify `buildArgs(prompt, [], [], { model: 'default' })` produces the expected baseline argv. Add a second case with a non-default model and any reasoning option, if applicable.
- Verify `resolveOnPath(bin)` returns null when the binary is absent (test env isolation via `OD_AGENT_HOME`).

### Risks

| Risk | Mitigation |
|------|-----------|
| New `streamFormat` string not routed in `server.ts` | Check routing immediately after adding; write a test that sends a mock chat and verifies events arrive |
| CLI flag rejected by installed version | Gate via `capabilityFlags` probe (`helpArgs` + `agentCapabilities` map) |
| Prompt too large for argv on Windows | Use `promptViaStdin: true`; if not possible, set `maxPromptArgBytes: 30_000` |
| `id` collision with existing adapter | The `AGENT_DEFS` map will silently use the first match; add a uniqueness check to the test suite |
| ACP `initialize` timeout | `DEFAULT_TIMEOUT_MS = 15_000` in `acp.ts`; increase with `fetchModels.timeoutMs` if the agent is known to be slow on first boot |

### Validation checklist

- [ ] New entry appears in `GET /api/agents` response from a running daemon
- [ ] `buildArgs()` output is tested for base case and model-selection case
- [ ] A real CLI run produces at least one `text_delta` event in the SSE stream
- [ ] Run completes with `status: 'succeeded'` (not `failed`) and `exitCode: 0`
- [ ] Windows `pnpm tools-dev` smoke: compose a skill + design system prompt, verify no `ENAMETOOLONG`
- [ ] `pnpm guard && pnpm typecheck` pass

---

## Extension Point 2: Adding a New Stream Parser

### When to use

When you add a new agent adapter whose stdout format is not covered by any existing parser. Current stream formats: `claude-stream-json`, `qoder-stream-json`, `acp-json-rpc`, `json-event-stream` (with `eventParser` sub-routing), `copilot-stream-json`, `pi-rpc`, `plain`.

### Files to modify

| File | What to do |
|------|-----------|
| `apps/daemon/src/<agent>-stream.ts` | Create the new parser module |
| `apps/daemon/src/server.ts` | Add routing branch in `POST /api/chat` for the new `streamFormat` |
| `apps/daemon/src/agents.ts` | Set `streamFormat: '<new-format>'` on the adapter |

### Existing examples to copy

- **`apps/daemon/src/claude-stream.ts`** — Gold standard. Shows `{ feed(chunk), flush() }` interface, per-block scratch map, duplicate-text guard, and all 6 event types.
- **`apps/daemon/src/qoder-stream.ts`** — Simpler; shows how to wrap a proprietary format into standard AgentEvents.
- **`apps/daemon/src/json-event-stream.ts`** — Shows `eventParser` sub-routing pattern for agents that share a format family but differ in detail.

### Required conventions

The parser module must export a factory function with this signature:

```typescript
export function createXxxStreamHandler(
  onEvent: (event: AgentEvent) => void
): { feed(chunk: string): void; flush(): void }
```

The `onEvent` callback must emit only events from the standard `AgentEvent` union (defined in `packages/contracts/src/`):

| Event type | Payload fields | Notes |
|-----------|----------------|-------|
| `status` | `label: string` | Lifecycle label: `'initializing'`, `'requesting'`, `'thinking'` |
| `text_delta` | `text: string` | Assistant text chunk; fed to the artifact parser |
| `thinking_delta` | `text: string` | Extended-thinking chunk |
| `tool_use` | `id`, `name`, `input` | Emitted once, when input JSON is complete |
| `tool_result` | `tool_use_id`, `content`, `is_error` | Tool execution result |
| `usage` | `input_tokens`, `output_tokens`, `cache_*` | Token accounting |
| `raw` | `line: string` | Unparseable line; forwarded for debugging |

The parser must:
- Handle incomplete lines (newline-delimited JSON may be split across chunks). Buffer in `feed()`; parse on `'\n'`.
- Call `flush()` after the process exits to handle any buffered remainder.
- Never throw; wrap `JSON.parse` in try/catch and emit `{ type: 'raw', line }` on failure.

### Tests to add

- Unit test the parser with a sequence of pre-recorded stdout chunks. Assert the ordered list of emitted AgentEvents.
- Edge-case: chunk split in the middle of a JSON object. Assert no events are lost or duplicated.
- Edge-case: empty flush (no buffered data). Assert no error.

### Risks

| Risk | Mitigation |
|------|-----------|
| Duplicate text if the format sends both deltas and a final wrapper | Track emitted message ids in a Set; skip final wrapper if deltas were seen (see `textStreamed` in `claude-stream.ts`) |
| Parser throws on malformed input and crashes the daemon | Wrap all parsing in try/catch |
| `tool_use` input emitted before input JSON is complete | Accumulate input fragments in a per-block Map; emit only on block close |

### Validation checklist

- [ ] Unit tests pass with pre-recorded real CLI output (not synthetic)
- [ ] `text_delta` events feed through to `apps/web/src/artifacts/parser.ts` correctly
- [ ] `usage` event fires at end-of-turn with non-zero token counts
- [ ] Incomplete chunk (split mid-JSON) handled without error
- [ ] `flush()` with empty buffer does not throw

---

## Extension Point 3: Adding a New Skill

### When to use

When you want to add a new kind of design artifact to Open Design (a new document type, a new visual style, a specialized deck template, an image-generation workflow, etc.).

### Files to create

| File | Required | Purpose |
|------|----------|---------|
| `skills/<slug>/SKILL.md` | Yes | Workflow instructions + frontmatter metadata |
| `skills/<slug>/assets/template.html` | Recommended | Seed HTML; triggers pre-flight injection |
| `skills/<slug>/references/*.md` | Optional | Side-files the agent reads for detailed rules |

### Existing examples to copy

Browse `skills/` for an existing skill in the same mode. For a new `prototype` skill, copy a simple existing prototype skill's structure. For a new `deck` skill, copy `skills/simple-deck/` or similar.

### Required conventions

**Frontmatter (YAML front matter at the top of SKILL.md):**

```yaml
---
name: my-skill-slug          # required; becomes the skill id; must be unique
description: "One-line description shown in the UI"
triggers:
  - keyword1                 # optional; used for auto-matching user prompts
od:
  mode: prototype            # required: prototype | deck | template | design-system | image | video | audio
  surface: web               # optional: web | mobile | print
  craft:
    requires:
      - typography           # optional: slugs from craft/ folder
  design_system:
    requires: true           # default: true; set false if the skill works without a brand
  preview:
    type: html               # html | image | video | audio (default: html)
  default_for:               # optional: when to auto-select this skill
    - kind: prototype
---
```

**Body (Markdown):**
- Reference `assets/template.html` by that exact path. The daemon's `derivePreflight()` function searches for this string and injects the pre-flight rule.
- Reference side-files as `.od-skills/<slug>/references/<file>.md` (the staged cwd-relative path). Also include the absolute repo path as a fallback for agents without `--add-dir` support.
- Keep the skill body focused on workflow — what the agent must do turn-by-turn. Brand tokens come from DESIGN.md; universal craft rules come from `craft/`. Do not duplicate them.

**Skill id aliasing:** If you rename a skill that is already in production, add the old id → new id mapping to `SKILL_ID_ALIASES` in `skills.ts` and leave it there for at least one release.

### Tests to add

- `GET /api/skills` — verify the new skill appears with the correct `id`, `mode`, `craftRequires`, `designSystemRequired`.
- Manual: bind the skill to a project, submit a prompt, verify the skill body appears in the composed system prompt (visible in daemon debug logs).
- If `assets/template.html` is present, verify the pre-flight instruction appears in the composed prompt.

### Risks

| Risk | Mitigation |
|------|-----------|
| Skill id collision with existing skill | Check `GET /api/skills` listing; ids must be unique |
| Missing `assets/template.html` reference in SKILL.md body | The pre-flight rule won't inject; add the reference or accept the agent will start from scratch |
| Craft slug that doesn't exist under `craft/` | `loadCraftSections()` silently skips missing files; verify craft slug names against the `craft/` directory |
| Skill body exceeds prompt budget | Keep body focused; move verbose rules to references/ side-files |

### Validation checklist

- [ ] `GET /api/skills` lists the new skill with correct metadata
- [ ] Staging: `stageActiveSkill()` completes without error (check daemon logs)
- [ ] Pre-flight rule appears in composed prompt when `assets/template.html` is present
- [ ] Craft references resolve (non-empty `craftBody` in composed prompt)
- [ ] A full generation turn with the new skill produces a valid artifact

---

## Extension Point 4: Adding a New Craft Reference

### When to use

When you want to add a universal, brand-agnostic writing or design rule that multiple skills can opt into — for example, a new typography guideline, a component pattern library, or an expanded anti-slop pattern set.

### Files to create

| File | Required | Purpose |
|------|----------|---------|
| `craft/<slug>.md` | Yes | The craft rule document |

### Existing examples to copy

Read existing files under `craft/` (e.g. `craft/typography.md`, `craft/anti-slop.md`) to understand the expected document structure and tone.

### Required conventions

1. The filename (without `.md`) becomes the slug used in `od.craft.requires` lists in SKILL.md files.
2. The document should be written as authoritative rules the agent follows, not as commentary or reference material.
3. Keep craft rules brand-agnostic. Brand-specific token values belong in DESIGN.md files, not in craft references.
4. On conflicts between a craft rule and a brand DESIGN.md, the brand wins for token values. Craft rules govern anything the brand doesn't override (letter-spacing, accent-overuse caps, anti-slop patterns). This precedence is enforced by the ordering in `composeSystemPrompt()` — DESIGN.md is injected before the craft body.

### Integration with skills

Opt a skill into the new craft reference by adding the slug to `od.craft.requires` in the skill's SKILL.md:

```yaml
od:
  craft:
    requires:
      - my-new-slug
```

`loadCraftSections(craftRequires, craftRoot)` in `craft.ts` will then read `craft/my-new-slug.md` and include it in the composed system prompt between the design system and the skill body.

### Tests to add

- `GET /api/skills` — verify `craftRequires` includes the new slug for skills that opt in.
- Inspect the composed prompt for a project using an opting-in skill; verify craft body is present and non-empty.
- No unit tests exist for `loadCraftSections()` itself [Uncertain: may be covered in daemon tests]; consider adding one.

### Risks

| Risk | Mitigation |
|------|-----------|
| Slug typo in `od.craft.requires` | `loadCraftSections()` silently skips missing files; verify by inspecting the composed prompt |
| Craft rule contradicts brand DESIGN.md | Document the precedence rule clearly in the craft file header |
| Craft body too long for prompt budget | Split into multiple focused files and opt in only the needed ones |

### Validation checklist

- [ ] `craft/<slug>.md` exists and reads correctly
- [ ] At least one skill opts in via `od.craft.requires`
- [ ] Craft body appears in composed prompt for a project using that skill
- [ ] No prompt-length budget exceeded for a typical project

---

## Extension Point 5: Adding a New Design System

### When to use

When you want to add a new brand's visual design tokens (colors, typography, spacing, component rules) so that the agent can generate on-brand artifacts.

### Files to create

| File | Required | Purpose |
|------|----------|---------|
| `design-systems/<id>/DESIGN.md` | Yes | The 9-section brand specification |

### Existing examples to copy

Browse `design-systems/` for an existing design system. The schema is consistent across all entries.

### Required conventions

**DESIGN.md structure (9-section schema):**

```markdown
> Category: <category-label>

# <Brand Name>

## Color System
...

## Typography
...

## Spacing & Layout
...

## Component Patterns
...

## Iconography
...

## Motion & Animation
...

## Voice & Tone
...

## Implementation Notes
...

## Example Usage
...
```

The `> Category: <label>` blockquote at the top drives the category grouping in `listDesignSystems()` in `design-systems.ts`. The `<id>` folder name becomes the design system id stored in project rows.

**Auto-discovery:** `listDesignSystems()` globs `design-systems/*/DESIGN.md`. No registration step is needed beyond creating the file.

### Tests to add

- `GET /api/design-systems` — verify the new design system appears with the correct `id`, `name`, and `category`.
- Bind the design system to a project; inspect the composed prompt to verify the DESIGN.md body appears in the correct layer (after discovery, after official prompt, before craft and skill).

### Risks

| Risk | Mitigation |
|------|-----------|
| Missing `> Category:` line | `listDesignSystems()` may place the system in a default/uncategorized bucket [Uncertain] |
| Design system id collision | Folder name must be unique; check the listing |
| Overly long DESIGN.md | Hits the prompt budget for projects with complex skills; keep token-critical sections concise |
| Injecting conflicting brand CSS variables | The agent is instructed to treat DESIGN.md as authoritative; conflicting craft rules are overridden by the brand for token values |

### Validation checklist

- [ ] `GET /api/design-systems` lists the new system with correct metadata
- [ ] Design system body appears in composed prompt for a project bound to it
- [ ] Agent honors brand color/typography tokens in the generated artifact
- [ ] `> Category:` line present and correct

---

## Extension Point 6: Adding a New BYOK Provider

### When to use

When you want to add a direct browser-to-provider streaming proxy for a new BYOK (Bring Your Own Key) LLM provider — i.e., allowing the web UI to stream completions from a provider's API using the user's own API key, without routing through the agent CLI layer.

### Files to modify

| File | What to do |
|------|-----------|
| `apps/daemon/src/server.ts` | Add `POST /api/proxy/<provider>/stream` route |
| `apps/daemon/src/media-config.ts` | If the provider requires a stored API key, add a key slot here |
| `packages/contracts/src/` | Add the new provider's event types to the SSE event union if they differ |

### Existing examples to copy

Read the existing `POST /api/proxy/<provider>/stream` routes in `server.ts` for Anthropic and OpenAI. They follow the same pattern: validate request origin → read API key from media-config → call `isLoopbackOrPrivateLanHost()` + `isPrivateIpv4()` → proxy the request → normalize response to `delta/end/error` SSE format.

### Required conventions

1. **SSRF validation is mandatory.** Before making any outbound HTTP request, resolve the target hostname and verify it is not a loopback, link-local, or RFC-1918 private address. Use `isLoopbackOrPrivateLanHost()` and `isPrivateIpv4()` from `origin-validation.ts`. Skipping this allows a malicious caller to exfiltrate internal network services.
2. Apply `requireLocalDaemonRequest()` to the route. This middleware ensures only local browser requests (not arbitrary remote callers) can trigger outbound API calls using the user's stored keys.
3. Normalize SSE output to the standard `delta/end/error` event format expected by the web client, regardless of the provider's native event format.
4. Do not store the API key in SQLite. Store it in the provider-keyed slot in `media-config.ts` (which persists to `.od/media-config.json`).
5. [Uncertain] Verify whether the BYOK path fully implements the agent tool loop or only handles single-turn completions; some providers' streaming APIs do not expose a tool-call interface compatible with the daemon's event model.

### Tests to add

- Unit test the SSRF validation: verify that requests targeting `10.0.0.1`, `172.16.x.x`, `192.168.x.x`, `169.254.x.x`, and `localhost` are rejected.
- Integration test: mock the provider's SSE stream and verify the daemon's normalized output matches the expected `delta/end/error` sequence.

### Risks

| Risk | Mitigation |
|------|-----------|
| SSRF bypass via DNS rebinding | `requireLocalDaemonRequest()` + SSRF check at request time, not at config time |
| API key leaked in error responses | Never include the key in error messages or SSE error payloads |
| Provider SSE format diverges after a library update | Pin provider SDK version; add integration test with recorded fixture |
| Proxy endpoint callable from non-local origins | `requireLocalDaemonRequest()` is mandatory; do not omit it |

### Validation checklist

- [ ] `POST /api/proxy/<provider>/stream` returns 403 for requests from non-localhost origins
- [ ] SSRF check rejects private IP targets
- [ ] `delta` events arrive in the browser during a real streaming completion
- [ ] API key is not echoed in any response or log
- [ ] `pnpm guard && pnpm typecheck` pass

---

## Extension Point 7: Adding a New Export Format

> [Uncertain] The exact export pipeline code was not fully traced during this documentation pass. The following is based on what was found in `server.ts` and `transcript-export.ts`.

### When to use

When you want to add a new artifact export format (e.g. a new ZIP layout, a new document format, or a new packaging scheme for generated HTML artifacts).

### Files to modify

| File | What to do |
|------|-----------|
| `apps/daemon/src/server.ts` | Add a new `GET /api/projects/:id/export/<format>` route (or `POST`, depending on the format) |
| `apps/daemon/src/transcript-export.ts` | Add export logic here if it follows the transcript-export pattern |

### Existing examples to copy

- `apps/daemon/src/transcript-export.ts` — Handles conversation/artifact transcript export. Read its structure to understand the expected pattern.
- `apps/daemon/src/deploy.ts` — Handles deployment/publishing. May contain export-adjacent logic. [Uncertain]

### Required conventions

1. Apply `requireLocalDaemonRequest()` to the export route.
2. Stream large exports rather than buffering in memory. Use `res.setHeader('Content-Disposition', 'attachment; filename="..."')` for download.
3. Export must be reproducible: given the same project state, two exports of the same format must produce byte-for-byte identical output (or at minimum semantically equivalent output).
4. [Uncertain] PDF export depends on the deck framework's print stylesheet in `prompts/deck-framework.ts`. Any new deck-format export must respect the `@media print` rules injected into generated HTML.

### Tests to add

- Integration test: export a known fixture project and assert the output structure matches the expected format.
- Edge case: project with no artifacts produces an empty or appropriately-structured export without errors.

### Risks

| Risk | Mitigation |
|------|-----------|
| Large project exports exhausting daemon memory | Stream the response; never buffer the full export in RAM |
| Export including sensitive data (API keys, credentials) | Scan export content before sending; keys live in `media-config.json` not in artifacts, but be cautious |
| Export route accessible to non-local origins | `requireLocalDaemonRequest()` mandatory |

### Validation checklist

- [ ] Export route returns correct `Content-Type` and `Content-Disposition` headers
- [ ] Downloaded file opens correctly in the target application
- [ ] Empty project exports gracefully (no 500 error)
- [ ] `requireLocalDaemonRequest()` applied
- [ ] `pnpm guard && pnpm typecheck` pass

---

## Extension Point 8: Adding a New Backend Route / Service

### When to use

When you need to add a new capability to the daemon — a new API endpoint, a new background service, or a new persistent data type.

### Files to modify

| File | What to do |
|------|-----------|
| `apps/daemon/src/server.ts` | Add the new Express route(s) |
| `apps/daemon/src/db.ts` | Add new table(s), migration(s), and query functions if the feature needs persistence |
| `packages/contracts/src/` | Add new request/response TypeScript types; this is the web/daemon contract |
| `apps/daemon/src/runs.ts` | If the feature involves SSE streaming, create a run via `createChatRunService()` |

### Existing examples to copy

Any existing route in `server.ts` is a valid structural example. For SSE streaming: the `POST /api/chat` route. For simple CRUD: the `GET/POST/PUT/DELETE /api/projects` routes. For background services: `orbit.ts` (Orbit activity summarizer) or `community-pets-sync.ts`.

### Required conventions

1. Apply `requireLocalDaemonRequest()` to all mutating routes (POST, PUT, PATCH, DELETE) and any GET routes that return user-private data. This middleware rejects non-local origins.
2. All new persistent state must go in SQLite via `db.ts`. Do not create ad-hoc JSON files for structured data (the exception is `media-config.json` for API keys and `app-config.json` for preferences — both have dedicated modules).
3. Keep `packages/contracts/src/` up to date before implementing the web client side. The boundary rule: contracts must remain free of Next.js, Express, Node.js process/filesystem APIs, and daemon internals.
4. Emit SSE events via `createSseResponse()` from `runs.ts`. Do not write raw `res.write('data: ...')` calls.
5. New tables need a migration entry in `db.ts`. The migration system applies unapplied migrations in version order on startup.
6. If the service runs on a schedule or as a background loop, use Node's `setInterval` / `setTimeout` with `.unref()` so the daemon process can exit cleanly without the timer keeping it alive.

### Tests to add

- `apps/daemon/tests/` — Integration test calling the new route and asserting the response shape.
- For persistence: test that the new table is created by the migration, data survives a daemon restart, and concurrent reads/writes don't corrupt state.
- For SSE endpoints: test that events are buffered and replayed correctly to late-joining clients.

### Risks

| Risk | Mitigation |
|------|-----------|
| Route accessible to non-local origins | `requireLocalDaemonRequest()` on every mutating route |
| New table without migration | Daemon will 500 on first access if the table doesn't exist |
| SSE event buffer overflow | `maxEvents = 2_000` cap in `createChatRunService()`; consider a lower cap for high-frequency event routes |
| Cross-app import boundary violation | `apps/web/**` must not import `apps/daemon/src/**`; new shared types go in `packages/contracts` |
| SQLite write contention | Keep write transactions short; use WAL mode (already enabled in `db.ts`) |

### Validation checklist

- [ ] New route returns correct HTTP status codes for happy path and error cases
- [ ] `requireLocalDaemonRequest()` applied and tested with a cross-origin request
- [ ] Migration runs cleanly on a fresh daemon start
- [ ] Schema change reflected in `packages/contracts/src/` before web client implementation
- [ ] `pnpm guard && pnpm typecheck` pass
- [ ] `pnpm --filter @open-design/daemon test` passes

---

## Extension Point 9: Adding a New Env Var / Config Option

### When to use

When you need to add a new runtime knob to the daemon — for example, a feature flag, a path override, a timeout adjustment, or a new BYOK key slot.

### Files to modify

| File | What to do |
|------|-----------|
| `apps/daemon/src/app-config.ts` | Add user-facing preferences (persisted to `app-config.json`, accessible via `PUT /api/app-config`) |
| `apps/daemon/src/media-config.ts` | Add API key slots (persisted to `media-config.json`) |
| Relevant module (e.g. `agents.ts`, `server.ts`, `acp.ts`) | Read the env var directly with `process.env.VAR_NAME` at the point of use |
| `AGENTS.md` (root) | Document the new env var in the `## Known Unknowns` or a dedicated config table |

### Existing examples to copy

- `OD_DATA_DIR` — Relocates all daemon runtime data. Documented in `AGENTS.md` under "Where is data written?". Read in `app-config.ts` and `db.ts`.
- `OD_ALLOWED_ORIGINS` — CORS extension for trusted external origins. Read in `origin-validation.ts → configuredAllowedOrigins()`.
- `OD_CODEX_DISABLE_PLUGINS` — Feature flag read inline in `agents.ts` inside `buildArgs`.
- `OD_AGENT_HOME` — Path override for the agent binary search root. Read in `agents.ts → userToolchainDirs()`.
- `QODER_PERSONAL_ACCESS_TOKEN` — User secret passed through inherited environment. Not set in the adapter's `env` field; explicitly excluded by comment in `agents.ts`.

### Required conventions

1. **Prefix daemon-specific env vars with `OD_`.** Exceptions are secrets the underlying CLI owns (e.g. `QODER_PERSONAL_ACCESS_TOKEN`, `GEMINI_CLI_TRUST_WORKSPACE`).
2. **Document every new env var in `AGENTS.md`** (root-level documentation). Include: name, purpose, default value, and any side-effects.
3. **Secrets (API keys) must not be in the adapter's `env` field** in `AGENT_DEFS`. They pass through the inherited process environment. Store user-supplied keys in `media-config.ts`.
4. **Preferences (non-secret runtime knobs)** that the web UI should persist belong in `app-config.ts` with a corresponding entry in `ALLOWED_KEYS`.
5. **Port and path vars** (`OD_PORT`, `OD_WEB_PORT`, `OD_DATA_DIR`, `OD_MEDIA_CONFIG_DIR`) are owned by `tools-dev`. Do not read or set these directly in the daemon; consume them via the sidecar stamp or the config module.
6. Validate env var values at startup (not lazily at use-time) so misconfiguration fails fast with a clear error rather than at an unpredictable moment.

### Tests to add

- Unit test for the consuming module: verify the env var default value, the overridden value, and an invalid value each produce the expected behavior.
- For path-override vars: test that the path is resolved correctly relative to the project root (see `home-expansion.ts` for `~/` expansion logic).

### Risks

| Risk | Mitigation |
|------|-----------|
| Undocumented env var causes confusion for operators | Add to `AGENTS.md` before merging |
| Secret injected into agent `env` field in `AGENT_DEFS` | Review `AGENT_DEFS` diff; secrets must flow via inherited environment, not adapter env |
| Env var read lazily in hot path and causes non-deterministic behavior | Read once at module load or at startup; cache the value |
| `OD_DATA_DIR` override not respected by new module | Use the centralized config/path resolver; do not call `process.env.OD_DATA_DIR` inline in a new module without going through the established resolution chain |

### Validation checklist

- [ ] Env var documented in `AGENTS.md`
- [ ] Default value produces existing behavior unchanged (non-breaking)
- [ ] Overridden value produces the intended new behavior
- [ ] `pnpm guard && pnpm typecheck` pass
- [ ] No secret env vars added to `AGENT_DEFS[*].env`
