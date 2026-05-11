# Developer Onboarding

## Prerequisites

| Tool | Required Version | Notes |
|---|---|---|
| Node.js | `~24` (engines field in `package.json`) | Use `nvm` or `fnm` to manage versions. The `~` pin means any `24.x.y` release. |
| pnpm | `10.33.2` (pinned in `package.json` > `packageManager`) | Do **not** install globally with `npm install -g pnpm`. Use Corepack instead so the pinned version is selected automatically. |
| Corepack | Ships with Node 16.9+; enable it once | Reads `packageManager` from `package.json` and transparently activates the correct pnpm binary. |
| Git | Any recent version | |
| OS | macOS, Linux, or Windows 11 | Desktop build requires Electron prerequisites. |

---

## Install

```bash
# 1. Activate Corepack so pnpm@10.33.2 is used automatically
corepack enable

# 2. Clone the repository
git clone <repo-url>
cd open-design

# 3. Install all workspace dependencies
pnpm install
```

Run `pnpm install` again after any change to `package.json`, `pnpm-workspace.yaml`, workspace layout, or command entrypoints. This keeps workspace symlinks and generated `dist/` entries fresh.

---

## Run Locally — Web Mode

Web mode starts the daemon and the Next.js web frontend without the Electron shell.

```bash
# Quick start with default ports
pnpm tools-dev start web

# Explicit port assignment (preferred when running multiple namespaces)
pnpm tools-dev run web --daemon-port 17456 --web-port 17573
```

Port governance:
- `--daemon-port` sets the Express daemon listen port. The daemon exports `OD_PORT` for the web proxy target.
- `--web-port` sets the Next.js listen port. The daemon exports `OD_WEB_PORT`.
- Do **not** use `NEXT_PORT`; it is not recognized by the current toolchain.

The web UI will be available at `http://localhost:<web-port>` once both processes report healthy.

---

## Run Locally — Full Desktop

```bash
pnpm tools-dev
```

Running `pnpm tools-dev` without a subcommand starts daemon + web + Electron desktop in a coordinated namespace. Desktop discovers the web URL through sidecar IPC — it does not guess ports or read web internals. Use this flow when validating desktop-specific features (window chrome, sidecar lifecycle, packaged-app behavior).

Check desktop status:
```bash
pnpm tools-dev inspect desktop status --json
pnpm tools-dev inspect desktop screenshot --path /tmp/open-design.png
```

---

## Run with Docker

[Uncertain — a `deploy/` directory with a `docker-compose.yml` and `Dockerfile` does exist in the repository, but whether the Docker path is production-ready or maintained as a first-class developer workflow has not been verified. The commands below are inferred from the presence of those files.]

```bash
# From the deploy/ directory
cd deploy
docker compose up -d
```

Consult `deploy/README.md` for environment variables and volume mounts before using this path.

---

## Lifecycle Commands

All lifecycle commands go through `pnpm tools-dev`. There is no root `pnpm dev`, `pnpm start`, or `pnpm daemon` alias — these were intentionally removed to prevent inconsistent env, port, namespace, or log paths.

```bash
# Start all processes (daemon + web + desktop)
pnpm tools-dev

# Start only the web stack (daemon + web, no Electron)
pnpm tools-dev start web

# Run web stack with explicit ports
pnpm tools-dev run web --daemon-port <port> --web-port <port>

# Show running process status as JSON
pnpm tools-dev status --json

# Tail logs for the default namespace
pnpm tools-dev logs --json

# Tail logs for a specific namespace
pnpm tools-dev logs --namespace <name> --json

# Stop all processes in the current namespace
pnpm tools-dev stop

# Validate configuration and connectivity (smoke-check without starting agents)
pnpm tools-dev check

# Inspect desktop process status
pnpm tools-dev inspect desktop status --json

# Capture a desktop screenshot
pnpm tools-dev inspect desktop screenshot --path /tmp/open-design.png

# Evaluate a JS expression in the desktop renderer
pnpm tools-dev inspect desktop eval '<expression>'
```

---

## Run Tests

Tests use [Vitest](https://vitest.dev/). There are 79 test files under `apps/daemon/tests/`.

```bash
# Daemon tests (most of the test surface)
pnpm --filter @open-design/daemon test

# Web tests
pnpm --filter @open-design/web test

# Run a single test file by path pattern
pnpm --filter @open-design/daemon test -- --reporter verbose <pattern>
```

There is **no** root `pnpm test` alias. All test commands must be package-scoped.

End-to-end Playwright tests live in `e2e/`. Read `e2e/AGENTS.md` before editing or running them; they have separate ownership and commands.

---

## Run Lint / Typecheck / Build

```bash
# Repo-level guard: checks for banned .js file additions, workspace invariants
pnpm guard

# Full workspace typecheck (TypeScript)
pnpm typecheck

# Per-package typechecks
pnpm --filter @open-design/web typecheck
pnpm --filter @open-design/daemon typecheck

# Per-package builds
pnpm --filter @open-design/web build
pnpm --filter @open-design/daemon build
pnpm --filter @open-design/desktop build
pnpm --filter @open-design/tools-dev build
pnpm --filter @open-design/tools-pack build
```

There is **no** root `pnpm build` alias. Build commands must be package-scoped.

Before marking work ready: run at least `pnpm guard` and `pnpm typecheck`, plus the package-scoped tests/builds that match the files changed.

---

## Environment Variables

| Name | Default | Purpose |
|---|---|---|
| `OD_DATA_DIR` | `<projectRoot>/.od` | Relocates all daemon runtime data (SQLite, agent CWDs, artifacts, credentials). Used by Playwright for test isolation and by packaged/NixOS installs. Supports `~/` expansion and relative paths. |
| `OD_MEDIA_CONFIG_DIR` | inherits `OD_DATA_DIR` | Narrower override: relocates only `media-config.json`. Use when credentials should live in a different location from the rest of runtime data. |
| `OD_LEGACY_DATA_DIR` | (none) | Source path for 0.3.x → current migration. The daemon runs `migrateLegacyDataDirSync()` on startup if this is set. |
| `OD_RESOURCE_ROOT` | `<projectRoot>` | Override the bundled skills, design-systems, craft, and frames directories. Useful for testing custom content packs. |
| `OD_BIND_HOST` | `127.0.0.1` | Daemon HTTP listen address. Set to `0.0.0.0` only in Docker/reverse-proxy deployments. |
| `OD_ALLOWED_ORIGINS` | (none) | Comma-separated URLs allowed by the CORS policy. Required when routing the daemon through a reverse proxy. |
| `OD_CODEX_DISABLE_PLUGINS` | (none) | Set to `'1'` to disable Codex plugins. Useful for debugging Codex agent behavior in isolation. |
| `OD_DAEMON_URL` | (auto-detected) | Daemon base URL used by the MCP server and by agent subprocesses. |
| `OD_PROJECT_ID` | (injected per-turn) | Current project ID. Injected into the agent's child process environment. |
| `OD_PROJECT_DIR` | (injected per-turn) | Current project directory (absolute path). Injected into the agent's child process environment. |
| `OD_CRITIQUE_ENABLED` | `false` | Enable Critique Theater (multi-round critique loop). |
| `OD_CRITIQUE_MAX_ROUNDS` | (default from config) | Maximum critique rounds before the loop terminates. |
| `OD_CRITIQUE_SCORE_THRESHOLD` | (default from config) | Minimum score (on `OD_CRITIQUE_SCORE_SCALE`) to accept output without another round. |
| `OD_CRITIQUE_SCORE_SCALE` | (default from config) | Scale for the critique score (e.g. 10). |
| `OD_CRITIQUE_PER_ROUND_TIMEOUT_MS` | (default from config) | Timeout per critique round in milliseconds. |
| `OD_CRITIQUE_TOTAL_TIMEOUT_MS` | (default from config) | Total timeout for the entire critique loop in milliseconds. |
| `PI_GRACEFUL_SHUTDOWN_MS` | `5000` | Grace period (ms) for process shutdown before a hard kill. |
| `PI_ABORT_GRACE_MS` | `3000` | Grace period (ms) for abort signal before a hard kill. |
| `CODEX_HOME` | `$HOME/.codex` | Codex user home directory. Used to resolve the `generated_images/` allowlist when the Codex imagegen override is active. |
| `ELECTRON_RUN_AS_NODE` | (sidecar use only) | Set by the sidecar bootstrap to run the Electron binary as a plain Node process. Do not set manually in development. |
| `OD_SIDECAR_IPC_PATH` | `/tmp/open-design/ipc/<namespace>/<app>.sock` | Override the POSIX IPC socket path for sidecar communication. |
| `OD_SIDECAR_NAMESPACE` | (derived from launch context) | Namespace string for sidecar process stamps. |
| `OD_SIDECAR_IPC_BASE` | `/tmp/open-design/ipc` | Base directory for all sidecar IPC sockets. |

Default data precedence: `OD_MEDIA_CONFIG_DIR` > `OD_DATA_DIR` > `<projectRoot>/.od`.

---

## First 60 Minutes Reading Guide

### First 10 minutes: Orientation

- `README.md` — product overview, feature list, quick install.
- `AGENTS.md` (repo root) — single source of truth for workspace layout, boundary rules, and development workflow. Read this before touching any code.

### Next 20 minutes: Daemon Core

- `apps/daemon/src/agents.ts` — agent registry: how each CLI (Claude Code, Codex, Copilot, Kimi, etc.) is registered with its `buildArgs()`, `streamFormat`, `listModels`, and capability flags.
- `apps/daemon/src/server.ts` — Express app: all HTTP routes, how `POST /api/chat` wires `startChatRun()`, how BYOK proxy routes are structured.
- `apps/daemon/src/runs.ts` — run lifecycle service: `create()`, `emit()`, `finish()`, `stream()` (SSE). The in-memory event store and TTL cleanup.
- `apps/daemon/src/claude-stream.ts` — JSONL stream handler for Claude Code's `--output-format stream-json`. Maps raw lines into typed `{ type, ...payload }` events.

### Next 30 minutes: Content Pipeline

- `skills/web-prototype/SKILL.md` — the default skill. Read the full frontmatter and body to understand the SKILL.md contract in practice.
- `apps/daemon/src/prompts/system.ts` — the full prompt composition stack. Trace `composeSystemPrompt()` layer by layer.
- `apps/web/src/artifacts/parser.ts` — the streaming `<artifact>` tag parser. Follow `ArtifactEvent` types and the state machine that drives `srcdoc` assignment.

---

## Debugging Agent Execution

### Tracing the spawn

1. Set `OD_DATA_DIR` to a known location so you can inspect the agent CWD:
   ```bash
   OD_DATA_DIR=/tmp/od-debug pnpm tools-dev run web --daemon-port 17456 --web-port 17573
   ```
2. Find the agent CWD at `/tmp/od-debug/projects/<id>/`.
3. Check `pnpm tools-dev logs --json` for the daemon's startup log. Agent spawn errors appear as `[chat]` prefixed lines.
4. To see the raw argv, add a `console.log` to `buildArgs()` in `apps/daemon/src/agents.ts` for the agent you are debugging, or look at the daemon logs for `[spawn]` lines.

### Tracing the env

The agent child process receives a filtered environment built by `spawnEnvForAgent()`. Key injected vars: `OD_PROJECT_ID`, `OD_PROJECT_DIR`, `OD_DAEMON_URL`, `OD_BIN`, `OD_NODE_BIN`. Check that `OD_BIN` resolves to the Claude Code CLI binary.

### Tracing stdout

Claude Code emits JSONL on stdout when `--output-format stream-json` is passed. Each line is a JSON object with a `type` field. Feed lines through `createClaudeStreamHandler(onEvent)` from `apps/daemon/src/claude-stream.ts` — `onEvent` receives typed events. Add a `console.log` in `handleObject()` to inspect raw lines during debugging.

---

## Debugging Artifact Preview

### Tracing the `<artifact>` tag

1. Open the browser devtools Network tab and find the `GET /api/runs/<id>/events` SSE stream.
2. Look for `event: text_delta` payloads. The artifact tag starts with `<artifact` and ends with `</artifact>`.
3. The streaming parser in `apps/web/src/artifacts/parser.ts` processes deltas incrementally. Set a breakpoint in `feed()` to watch state transitions.

### Tracing the iframe srcdoc

4. When `ArtifactEvent { type: 'artifact:end' }` fires, `fullContent` is the complete HTML.
5. The web component receives `fullContent` and assigns it to the preview iframe's `srcdoc` attribute.
6. If the iframe is blank, check: (a) is `fullContent` non-empty? (b) does the HTML contain a `<!doctype html>` declaration? (c) are there CSP violations in the browser console?

### Common artifact non-render causes

- The agent emitted `<artifact>` but the SSE stream was cut before `</artifact>`. Check the run's `status` — if it is `failed`, the partial artifact was never finalized.
- The `identifier` attribute on `<artifact>` is missing or contains a space. The parser expects `identifier="kebab-slug"`.
- The HTML artifact references external resources blocked by the iframe's sandbox policy.

---

## Debugging Export

[Uncertain — the export routes exist in `server.ts` (`buildProjectArchive`, `buildBatchArchive` from `projects.ts`), but the full export flow for PDF (browser print), PPTX (agent-driven), and ZIP has not been traced end-to-end in this documentation. The routes appear to be at `GET /api/projects/:id/export` and `POST /api/projects/batch/export`.]

For ZIP export: `buildProjectArchive()` in `apps/daemon/src/projects.ts` uses the `archiver` package to bundle all files in the project directory.

For HTML export: files are served inline with assets already embedded (single-file output).

For PDF and PPTX: these are agent-driven — the agent generates the artifact and the UI triggers a browser print dialog or downloads the file the agent produced.

---

## Common Setup Problems

### `OD_BIN` missing / Claude Code not found

**Symptom:** Agent spawn fails immediately with an error like `ENOENT: no such file or directory, claude`.

**Fix:** Ensure the Claude Code CLI is installed and on your `PATH`, or set `OD_BIN` to the absolute path of the `claude` binary:
```bash
export OD_BIN=/usr/local/bin/claude
```

Alternatively, verify the binary is accessible:
```bash
which claude
claude --version
```

### Daemon returns 500

**Symptom:** `POST /api/chat` returns HTTP 500 or the SSE stream immediately emits an `error` event.

**Fix:**
1. Check daemon logs: `pnpm tools-dev logs --json`
2. Look for startup errors (database migration failures, port conflicts, missing resource root).
3. Ensure `pnpm install` has been run after any recent `package.json` changes.
4. Verify the daemon process is running: `pnpm tools-dev status --json`.

### Artifact never renders

**Symptom:** The chat shows agent text but the preview pane stays blank.

**Checklist:**
1. Confirm the agent emitted a complete `<artifact>...</artifact>` block (check the SSE stream in devtools).
2. Confirm the `<artifact>` tag has `identifier`, `type`, and `title` attributes.
3. Confirm the `type` attribute value matches the skill's `od.preview.type` (usually `text/html`).
4. Check the browser console for CSP / sandbox errors in the preview iframe.

### Codex plugin context issues

**Symptom:** Codex agent behaves unexpectedly or requests plugin permissions.

**Fix:** Set `OD_CODEX_DISABLE_PLUGINS=1` to disable Codex plugins and reproduce the behavior in a clean environment:
```bash
OD_CODEX_DISABLE_PLUGINS=1 pnpm tools-dev run web --daemon-port 17456 --web-port 17573
```
