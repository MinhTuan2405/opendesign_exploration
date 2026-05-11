# Repo Audit

> **Date:** 2026-05-11
> **Method:** Direct source inspection of the `open-design` repository (`main` branch, commit `e14b809` and later).
> **Scope:** 12 engineering artifact documents (01–12) plus the underlying source files they describe.

---

## Audit Scope

Files inspected during this documentation pass:

| Area | Files inspected |
|------|----------------|
| Agent adapter layer | `apps/daemon/src/agents.ts` (full, ~900 lines) |
| Prompt composition | `apps/daemon/src/prompts/system.ts`, `discovery.ts`, `official-system.ts`, `deck-framework.ts`, `media-contract.ts`, `panel.ts` |
| Skill system | `apps/daemon/src/skills.ts`, `apps/daemon/src/cwd-aliases.ts`, `apps/daemon/src/craft.ts`, `apps/daemon/src/frontmatter.ts` |
| Stream parsers | `apps/daemon/src/claude-stream.ts`, `apps/daemon/src/acp.ts`, `apps/daemon/src/qoder-stream.ts` |
| Run registry | `apps/daemon/src/runs.ts` |
| Security | `apps/daemon/src/origin-validation.ts`, `apps/daemon/src/server.ts` (partial) |
| Configuration | `apps/daemon/src/app-config.ts`, `apps/daemon/src/media-config.ts` |
| Persistence | `apps/daemon/src/db.ts` (referenced; not fully read in this pass) |
| MCP | `apps/daemon/src/mcp.ts` (referenced; not fully read in this pass) |
| Existing documentation | `docs/architecture.md`, `docs/spec.md`, `docs/agent-adapters.md`, `docs/skills-protocol.md`, `AGENTS.md` |
| Artifact index | `docs/backend_explorations/open-design/open-design-artifacts-index.md` |

---

## Artifact Accuracy Review

| Artifact | Key Claims | Status | Evidence |
|----------|-----------|--------|---------|
| 00 (Index) | 16 agent adapters; 4-layer prompt stack; `.od-skills/<id>/` staging; 31 skills; anti-slop linter; 7 SQLite tables; 8 MCP read-only tools | Partially verified | Agent count confirmed (16 entries in `AGENT_DEFS`); prompt layers confirmed in `system.ts`; staging confirmed in `cwd-aliases.ts`; skill count `[Uncertain]` — glob returns >100 entries, not 31; 7 tables referenced in the index but `db.ts` not fully read in this pass; MCP tool count not verified in this pass |
| 01 (Folder Map) | Directory layout, runtime roles, `.od/` data dir structure | Verified | Cross-checked against `AGENTS.md`, `pnpm-workspace.yaml`, and `db.ts` references |
| 02 (Architecture Overview .mmd) | Component topology: browser ↔ daemon ↔ agent CLIs; BYOK proxy; SQLite; MCP server | Verified | All named components exist and interact as diagrammed |
| 03 (Runtime Sequence .mmd) | End-to-end generation flow: prompt → skill staging → agent spawn → SSE events → artifact parse | Verified | Confirmed against `server.ts`, `cwd-aliases.ts`, `claude-stream.ts`, `runs.ts` |
| 04 (Backend Deep Dive) | 16 agents, 7 stream formats, 4-layer prompt, skill staging, linter P0/P1/P2, SQLite 7 tables, MCP 8 tools | Partially verified | Agents and stream formats confirmed; skill count uncertain; SQLite and MCP details not re-verified in final pass |
| 05 (Data Flow Map) | Request lifecycle, event routing, SSE buffering, artifact persistence | Verified | Confirmed in `runs.ts` and `server.ts` review |
| 06 (Backend Reading Map) | Top 10 files, 8 reading paths, reading order per goal | Verified | File references confirmed against directory listing; reading order validated by tracing actual code paths |
| 07 (Agentic Platform) | Agent CLI details, ACP protocol, stream formats, capability probing | Partially verified | ACP protocol confirmed in `acp.ts`; capability probing confirmed in `agents.ts`; some agent-specific stream format details not individually re-verified |
| 08 (Design Systems & Skills) | Skill frontmatter schema, craft system, design system 9-section schema, skill staging | Verified | Confirmed against `skills.ts`, `cwd-aliases.ts`, `craft.ts`, `prompts/system.ts` |
| 09 (Developer Onboarding) | Setup steps, pnpm commands, workspace structure, common workflows | Partially verified | Commands confirmed against `AGENTS.md`; some onboarding steps verified only against documentation, not live execution |
| 10 (Extension Points) | 9 extension patterns with file lists, conventions, risks | Verified | Each extension point cross-checked against the source files it references |
| 11 (Repo Audit, this file) | Audit scope, risk list, security areas, missing documentation | N/A (this is the audit itself) | — |

---

## Unsupported Claims

The following claims appear in the artifact set but could not be fully verified against live source during this documentation pass:

| Claim | Source artifact | Why unverified |
|-------|----------------|----------------|
| "31 bundled skills" | 04 (Backend Deep Dive), 00 (Index) | `listSkills()` globs all `skills/*/SKILL.md` entries; the glob returned well over 100 entries. The "31" figure appears to be outdated or refers to a subset. Treat as `[Uncertain]`. |
| "72+ design systems" | 04 (Backend Deep Dive), 00 (Index) | The design-systems glob returned more than 72 DESIGN.md files. The exact count was not re-counted. Treat as a lower bound. |
| "8 read-only MCP tools, 3 resources" | 04 (Backend Deep Dive) | `mcp.ts` was not fully read in the final documentation pass. The tool and resource counts were taken from the research summary. |
| "7 SQLite tables" | 00 (Index), 04 | `db.ts` was not fully read to count tables. The 7-table claim (projects, conversations, messages, templates, deployments, tabs, preview_comments) came from prior research. |
| "Critique Theater: 5-dimension self-critique" | 04, 07 | The `critique/` subdirectory was referenced but not fully explored. The 5-dimension claim was not verified against live code. |
| "SSRF blocking covers 10.x, 172.16-31.x, 192.168.x, 169.254.x" | Backend Reading Map, security sections | The `isPrivateIpv4()` function in `origin-validation.ts` was read and confirmed to cover these ranges. This claim is verified. |
| "iframe srcdoc sandbox with CSP" | Architecture Overview | The web renderer's iframe CSP settings were not traced in `apps/web/src/`. The iframe sandbox is confirmed; CSP header configuration is `[Uncertain]`. |
| "PDF stitching depends on deck-framework directive" | `prompts/system.ts` JSDoc | The JSDoc asserts this but the actual PDF export pipeline was not traced. |
| Export format handlers (PDF, PPTX, ZIP) | Architecture docs | `transcript-export.ts` was found; `deploy.ts` was referenced. The PPTX and ZIP export implementations were not confirmed. |

---

## Top Architecture Risks

### Risk 1: Shell Argv Length Limit (Windows / Linux)

**Severity:** High (production correctness)

**Description:** The `deepseek` adapter is the only remaining adapter that passes the full composed prompt as a positional argv argument. On Windows, `CreateProcess` limits `lpCommandLine` to ~32 KB; on Linux, `MAX_ARG_STRLEN` limits a single argument to ~128 KB. A complex prompt (long DESIGN.md + skill body + discovery layer) can exceed these limits.

**Mitigation in place:** `maxPromptArgBytes: 30_000` triggers a user-facing SSE error before spawn. `checkWindowsDirectExeCommandLineBudget()` in `server.ts` provides the pre-flight check.

**Remaining gap:** The Linux `E2BIG` case (full argv size across all arguments, not just one) is handled per the research summary but was not re-verified in code.

---

### Risk 2: Skill Staging Storage Growth

**Severity:** Medium (operational)

**Description:** Every chat turn triggers `stageActiveSkill()`, which copies the active skill directory (1–3 MB on APFS/btrfs/ReFS with copy-on-write; potentially larger on Windows NTFS) to `<projectCwd>/.od-skills/<folderName>/`. A user with many projects and a skill with large assets will accumulate significant staged copies over time.

**Mitigation in place:** Only the *active* skill is staged, not the entire `skills/` tree. Copy-on-write (CoW) reduces the per-copy cost on supported filesystems.

**Remaining gap:** No automatic cleanup of stale staged copies when a skill is deselected or a project is deleted. Over time, `.od/projects/` can grow unboundedly.

---

### Risk 3: API Key Exposure via Shared `OD_DATA_DIR`

**Severity:** High (security)

**Description:** BYOK provider API keys are stored in `.od/media-config.json` (or wherever `OD_DATA_DIR` points). This file is in the filesystem, not in SQLite or an OS keychain. If `OD_DATA_DIR` is set to a shared directory, or if the project root is in a cloud-synced folder, API keys are accessible to other processes or users.

**Mitigation in place:** `media-config.ts` manages the file; it is never stored in SQLite or version control.

**Remaining gap:** No file-permission hardening (e.g. `chmod 600`) on `media-config.json`. No warning when `OD_DATA_DIR` points to a world-readable location.

---

### Risk 4: MCP Server Exposes Full Project File Contents

**Severity:** Medium (security/privacy)

**Description:** The `od mcp` stdio server exposes 8 read-only tools that allow any calling agent to list project files, read artifact HTML, and retrieve conversation history. The tools are read-only and stdio-only (no network exposure), but a compromised or malicious calling agent can exfiltrate all project data.

**Mitigation in place:** Read-only tools; stdio transport only; the daemon must be running locally.

**Remaining gap:** No per-tool access control or per-project permission scoping. Any process that can invoke `od mcp` gets access to all projects.

---

### Risk 5: SSRF via DNS Rebinding in BYOK Proxy

**Severity:** High (security)

**Description:** The BYOK proxy validates the target hostname against a private-IP blocklist at request time. However, DNS rebinding attacks can cause a hostname to resolve to a private IP after the validation check but before the actual TCP connection.

**Mitigation in place:** `isLoopbackOrPrivateLanHost()` + `isPrivateIpv4()` block known private ranges. `requireLocalDaemonRequest()` limits who can trigger proxy requests.

**Remaining gap:** DNS rebinding is not fully mitigated by IP-level checks alone; a TTL-0 DNS attack can bypass it. A full mitigation requires resolving the hostname and validating the resulting IP before every outbound connection. Whether the daemon re-validates after DNS resolution was not confirmed. `[Uncertain]`

---

### Risk 6: SQLite WAL Concurrent Write Contention

**Severity:** Low-Medium (operational)

**Description:** The daemon uses SQLite with WAL mode, which allows concurrent reads but serializes writes. Under high concurrency (multiple simultaneous generation runs writing artifacts and messages), write latency can spike. There is a single daemon process, which reduces but does not eliminate the risk.

**Mitigation in place:** WAL mode. The daemon is architecturally single-process.

**Remaining gap:** No connection pool tuning or explicit write serialization beyond WAL. External tools (e.g. DB browsers, tests) writing to `app.sqlite` concurrently can corrupt state.

---

### Risk 7: Generated Code in iframe — CSP Unverified

**Severity:** Medium (security)

**Description:** Agent-generated HTML artifacts are rendered in a sandboxed `<iframe srcdoc="...">` in the browser. The security model depends on the sandbox attribute and Content Security Policy. If CSP is missing or misconfigured, a malicious generated artifact could exfiltrate user data via XHR or script injection.

**Mitigation in place:** The `srcdoc` sandbox attribute is used. The iframe is local (no external origin).

**Remaining gap:** The exact CSP header for the srcdoc frame was not verified in `apps/web/src/`. `[Uncertain]`

---

## Security-Sensitive Areas

| Area | File | Risk Type | Severity |
|------|------|-----------|----------|
| Origin / host validation | `apps/daemon/src/origin-validation.ts` | DNS rebinding / SSRF | High |
| BYOK proxy | `apps/daemon/src/server.ts` (proxy routes) | SSRF | High |
| API key storage | `apps/daemon/src/media-config.ts`, `.od/media-config.json` | Credential exposure | High |
| Agent spawn command injection | `apps/daemon/src/agents.ts` (`sanitizeCustomModel`) | Command injection via model ID | Medium |
| Skill staging | `apps/daemon/src/cwd-aliases.ts` | Write-amplification via staged symlink | Low (mitigated) |
| MCP server | `apps/daemon/src/mcp.ts` | Project data exfiltration | Medium |
| iframe sandbox | `apps/web/src/` (artifact renderer) | XSS / data exfiltration from generated code | Medium (CSP unverified) |
| SQLite | `apps/daemon/src/db.ts` | Data integrity under concurrent access | Low-Medium |
| ACP permission auto-approval | `apps/daemon/src/acp.ts` (`choosePermissionOutcome`) | Unintended tool execution approval | Low (by design) |

---

## Most Complex Flows

### 1. Agent Spawn and SSE Streaming (Post /api/chat)

**Why complex:** This flow crosses the most subsystems: HTTP request parsing → origin validation → skill staging → prompt composition (4 layers + conditionals) → agent detection + capability probing → child process spawn → stdout stream parsing → SSE event distribution → artifact extraction → lint checking → optional corrective loop. Any of these 10+ steps can fail in a way that affects the others.

**Files involved:** `server.ts`, `cwd-aliases.ts`, `prompts/system.ts`, `agents.ts`, `runs.ts`, `claude-stream.ts` (or other parser), `lint-artifact.ts`, `apps/web/src/artifacts/parser.ts`.

**Hardest part:** The interaction between the run registry's event buffer (for late-joining SSE clients), the linter's P0 corrective loop (which may spawn a second agent turn), and the ACP session lifecycle (which drives a stateful protocol over the same stdio channel).

---

### 2. ACP Session Lifecycle (Devin, Hermes, Kimi, Kiro, Kilo, Vibe)

**Why complex:** ACP is a full JSON-RPC protocol with a multi-step handshake (initialize → session/new → session/prompt → permission callbacks → session/end). The daemon must drive this protocol while simultaneously forwarding events to SSE clients and handling user-initiated cancellation.

**Files involved:** `apps/daemon/src/acp.ts`, `apps/daemon/src/runs.ts`, `apps/daemon/src/agents.ts`.

**Hardest part:** The permission callback handler (`choosePermissionOutcome`) must respond synchronously within the RPC protocol's timeout window, without surfacing a UI prompt. The daemon auto-approves by selecting the least-restrictive option, which is correct for the headless web UI context but must not be applied to destructive tool calls.

---

### 3. Prompt Composition with All Layers Active

**Why complex:** `composeSystemPrompt()` assembles up to 8 distinct text layers with conditional injection logic. The ordering is load-bearing: the discovery layer's hard rules must win over the official prompt's softer wording; the deck framework must override any conflicting slide-handling instructions; craft rules must apply to anything the brand DESIGN.md doesn't override.

**Files involved:** `apps/daemon/src/prompts/system.ts`, `discovery.ts`, `official-system.ts`, `deck-framework.ts`, `media-contract.ts`, `panel.ts`, `craft.ts`.

**Hardest part:** The `isDeckProject && !hasSkillSeed` gate for the deck framework injection — the same semantics must be maintained across both skill-bound and skill-unbound deck projects, and adding a new skill with its own seed must not accidentally suppress the framework for a deck project that still needs it.

---

### 4. Windows CLI Spawn with Correct Argv / stdin Routing

**Why complex:** On Windows, both the `CreateProcess` line-length limit (~32 KB) and the `.cmd` shim encoding must be handled correctly. Each agent adapter has its own stdin vs. argv strategy; the capability probe must happen before the first spawn; and `createCommandInvocation` from `@open-design/platform` must correctly detect `.cmd` shims and set `windowsVerbatimArguments`.

**Files involved:** `apps/daemon/src/agents.ts`, `apps/daemon/src/server.ts`, `@open-design/platform/src/` (not read in this pass).

**Hardest part:** The `deepseek` adapter has no stdin fallback, so the daemon must detect prompt length before spawn and surface a user-facing error rather than a generic `ENAMETOOLONG`. Every other adapter uses `promptViaStdin: true` to avoid this.

---

## Missing Documentation

| Gap | Impact | Suggested action |
|-----|--------|-----------------|
| Critique Theater orchestrator | Engineers adding new post-generation hooks will not know how to integrate with the corrective loop | Read and document `apps/daemon/src/critique/` (subdirectory with `runOrchestrator`, `createRunRegistry`, `handleCritiqueInterrupt`) |
| Full SQLite schema | Engineers adding new features must re-read `db.ts` to avoid table/column conflicts | Extract and document all 7 (or more) table definitions and their migration history |
| Complete route inventory | Engineers navigating `server.ts` (~50 routes) have no quick-reference table | Add a route table to the Backend Deep Dive artifact (04) |
| Orbit service internals | Engineers wanting to add new activity integrations (GitHub, Linear, Notion) have no guide | Document `apps/daemon/src/orbit.ts` and its scheduler |
| MCP tools and resources | Engineers wanting to extend the external agent interface have no reference | Read and document `apps/daemon/src/mcp.ts` in full |
| iframe CSP configuration | Security reviewers cannot assess the generated-code sandbox without this | Trace the `<iframe srcdoc>` rendering path in `apps/web/src/` and document CSP settings |
| Research subsystem | Unknown capability that may affect agent behavior | Document `apps/daemon/src/research/index.ts` and `searchResearch()` |
| Export pipeline (PDF, PPTX, ZIP) | Contributors adding new export formats have no reference | Trace and document `transcript-export.ts` and the deploy/PDF stitching path |
| Windows IPC (named pipes) | Sidecar IPC path on Windows is unclear (`[Uncertain]` in architecture docs) | Read `packages/sidecar-proto/src/` and verify the Windows IPC path format |

---

## Recommended Next Steps

### 1. Verify and correct skill and design system counts

Run `find skills/ -name SKILL.md | wc -l` and `find design-systems/ -name DESIGN.md | wc -l` to get accurate counts. Update all artifact documents that state "31 skills" and "72 design systems" with the verified numbers.

### 2. Read `apps/daemon/src/critique/` in full

The Critique Theater orchestrator is referenced in `server.ts` imports but not documented. It directly affects the post-generation corrective loop. Document its interaction with the main run registry and the lint checker, and add it to the runtime sequence diagram (artifact 03).

### 3. Verify iframe CSP configuration

Trace the `<iframe srcdoc>` rendering path in `apps/web/src/artifacts/` and document the exact `sandbox` attribute values and any CSP headers applied to the frame. This is a security-critical gap.

### 4. Conduct a full route inventory of `server.ts`

Read all ~50 routes and produce a route table (method, path, auth, description, contracts type). Add it to the Backend Deep Dive (artifact 04). This is the single highest-value missing documentation for engineers working on new features.

### 5. Re-verify SSRF mitigation completeness

Determine whether the daemon re-validates the resolved IP of the BYOK proxy target after DNS resolution, or only validates the hostname string. If only the string is validated, document the residual DNS rebinding risk and file a security issue.
