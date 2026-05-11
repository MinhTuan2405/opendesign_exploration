# Open Design — Sequence Diagrams

> A collection of sequence diagrams tracing the main flows, actor journeys, and protocol interactions in Open Design.
> For the full architecture overview see `open-design-architecture-overview.mmd`.
> For data flow text descriptions see `open-design-data-flow-map.md`.

## Table of Contents

1. [SD-01: Main Generation Turn (Claude Code Path)](#sd-01-main-generation-turn-claude-code-path)
2. [SD-02: Main Generation Turn (ACP Agent Path — Hermes/Kimi/Kiro)](#sd-02-main-generation-turn-acp-agent-path--hermeskimikiro)
3. [SD-03: Main Generation Turn (BYOK Proxy Path — No CLI)](#sd-03-main-generation-turn-byok-proxy-path--no-cli)
4. [SD-04: External Agent Using MCP Bridge](#sd-04-external-agent-using-mcp-bridge)
5. [SD-05: Agent Detection on Daemon Startup](#sd-05-agent-detection-on-daemon-startup)
6. [SD-06: Skill Selection & Prompt Assembly](#sd-06-skill-selection--prompt-assembly)
7. [SD-07: Critique Theater Multi-Round Loop](#sd-07-critique-theater-multi-round-loop)
8. [SD-08: Export Pipeline (HTML/PDF/PPTX/ZIP)](#sd-08-export-pipeline-htmlpdfpptxzip)
9. [SD-09: Pi Agent Session (pi-rpc)](#sd-09-pi-agent-session-pi-rpc)
10. [SD-10: Live Artifact Refresh](#sd-10-live-artifact-refresh)

---

## SD-01: Main Generation Turn (Claude Code Path)

The primary happy-path for a generation request when Claude Code is detected on `PATH`. The daemon assembles a 10-layer system prompt, stages the active skill into the project CWD, spawns the Claude Code CLI, and pipes its JSONL stdout through the stream parser to SSE events consumed by the browser. Lint checks run post-generation; P0 findings trigger an automatic self-correction turn.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (WebUI)
    participant Daemon
    participant SQLite
    participant SkillLoader
    participant CwdAliases
    participant PromptComposer
    participant ClaudeCLI as ClaudeCode CLI
    participant StreamParser
    participant ArtifactParser
    participant LintChecker

    User->>Browser: Submit message in chat
    Browser->>Daemon: POST /api/chat { projectId, conversationId, agentId:'claude', prompt, model }
    Daemon->>SQLite: runs.create() → status:'queued'
    Daemon->>Browser: SSE response opened → event:start

    rect rgb(230, 240, 255)
        Note over Daemon,CwdAliases: Skill & Design System loading
        Daemon->>SQLite: SELECT project (skill_id, design_system_id, metadata)
        Daemon->>SkillLoader: listSkills() + findSkillById(skillId)
        SkillLoader-->>Daemon: skillBody (SKILL.md + references/*.md)
        Daemon->>SkillLoader: readDesignSystem(designSystemId)
        SkillLoader-->>Daemon: designSystemBody (DESIGN.md)
        Daemon->>SkillLoader: loadCraftSections(skill.craftRequires)
        SkillLoader-->>Daemon: craftBody (universal craft rules)
        Daemon->>CwdAliases: stageActiveSkill(skillId, projectDir)
        CwdAliases-->>Daemon: staged to .od/projects/<id>/.od-skills/<skillId>/
    end

    rect rgb(230, 255, 230)
        Note over Daemon,PromptComposer: Prompt composition (10-layer stack)
        Daemon->>PromptComposer: composeSystemPrompt(skill, ds, craft, metadata, opts)
        Note over PromptComposer: Layer 1: DISCOVERY_AND_PHILOSOPHY<br/>Layer 2: OFFICIAL_DESIGNER_PROMPT<br/>Layer 3: Active DESIGN.md body<br/>Layer 4: derivePreflight() + SKILL.md body<br/>Layer 5-9: opt metadata / deck / media / codex / critique
        PromptComposer-->>Daemon: composed system prompt string
    end

    rect rgb(255, 245, 220)
        Note over Daemon,ClaudeCLI: Spawn Claude Code CLI
        Daemon->>Daemon: buildArgs() → argv array (prompt via -p flag)
        Daemon->>Daemon: spawnEnvForAgent() → env { OD_DAEMON_URL, OD_PROJECT_ID, OD_PROJECT_DIR, ... }
        Daemon->>ClaudeCLI: child_process.spawn('claude', args, { cwd:.od/projects/<id>/, env })
        ClaudeCLI->>ClaudeCLI: reads .od-skills/<skillId>/assets/template.html (pre-flight)
        ClaudeCLI->>ClaudeCLI: reads .od-skills/<skillId>/references/checklist.md
        ClaudeCLI->>ClaudeCLI: generates HTML
        ClaudeCLI->>ClaudeCLI: writes to .od/projects/<id>/output.html
    end

    rect rgb(255, 230, 230)
        Note over ClaudeCLI,StreamParser: JSONL stream processing
        ClaudeCLI->>StreamParser: emits JSONL to stdout (system/init, stream_events, result)
        StreamParser->>StreamParser: createClaudeStreamHandler() — parses each JSONL line → AgentEvent
        StreamParser->>Daemon: agent events (text_delta, thinking_delta, tool_use, tool_result)
        Daemon->>Browser: SSE: agent(text_delta) / agent(thinking_delta) / agent(tool_use) / agent(tool_result)
        StreamParser->>ArtifactParser: detects <artifact> tag in text stream
        ArtifactParser->>Browser: SSE: artifact:start / artifact:chunk / artifact:end
    end

    ClaudeCLI->>Daemon: process exit (code 0)
    Daemon->>Daemon: runs.finish('succeeded')

    rect rgb(240, 230, 255)
        Note over Daemon,LintChecker: Post-generation lint
        Daemon->>LintChecker: lintArtifact(artifactHtml)
        LintChecker-->>Daemon: findings[] (P0/P1/P2)
        alt P0 findings exist
            Daemon->>Browser: SSE: lint-feedback event
            Note over Daemon,Browser: Agent self-corrects on next turn
        end
    end

    Daemon->>SQLite: saves message + artifact ref
    Browser->>Browser: srcdoc iframe updated with artifact HTML
    Daemon->>Browser: SSE: event:end
```

---

## SD-02: Main Generation Turn (ACP Agent Path — Hermes/Kimi/Kiro)

The ACP (Agent Control Protocol) path is used by Devin, Hermes, Kimi, Kiro, Kilo, and vibe-acp agents. Unlike the Claude Code path, the prompt is delivered via JSON-RPC over stdin (not argv), the agent lifecycle follows an explicit initialize → session/new → session/set_model → session/prompt sequence, and tool calls go through the MCP bridge rather than directly into the filesystem.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (WebUI)
    participant Daemon
    participant ACPSession as ACPSession (attachAcpSession)
    participant HermesAgent as Hermes Agent Process
    participant MCPBridge as MCP Bridge
    participant ArtifactParser

    User->>Browser: Submit message (agentId:'hermes')
    Browser->>Daemon: POST /api/chat { agentId:'hermes', projectId, prompt, model }
    Daemon->>Browser: SSE response opened → event:start
    Daemon->>HermesAgent: child_process.spawn('hermes', acpArgs, { cwd:.od/projects/<id>/ })

    rect rgb(230, 240, 255)
        Note over Daemon,HermesAgent: ACP handshake
        Daemon->>ACPSession: attachAcpSession({ child, prompt, model, systemPrompt })
        ACPSession->>HermesAgent: stdin: { jsonrpc:'2.0', method:'initialize' }
        HermesAgent-->>ACPSession: stdout: { capabilities: {...} }
        ACPSession->>HermesAgent: stdin: { method:'session/new' }
        HermesAgent-->>ACPSession: stdout: { sessionId: '...' }
        ACPSession->>HermesAgent: stdin: { method:'session/set_model', params:{ model } }
        HermesAgent-->>ACPSession: stdout: { ok: true }
        ACPSession->>HermesAgent: stdin: { method:'session/prompt', params:{ prompt: composedSystemPrompt + userMessage } }
    end

    rect rgb(230, 255, 230)
        Note over HermesAgent,MCPBridge: Agent tool calls via MCP bridge
        HermesAgent->>MCPBridge: tools/call search_files { projectId, query:'...' }
        MCPBridge->>Daemon: GET /api/projects/:id/files?q=...
        Daemon-->>MCPBridge: file matches (literal substring, case-insensitive, max 200)
        MCPBridge-->>HermesAgent: search results { file, line, snippet }

        HermesAgent->>MCPBridge: tools/call get_artifact { projectId, filePath }
        MCPBridge->>Daemon: GET /api/projects/:id/files/:path
        Note over MCPBridge: BFS traversal of referenced files (script/link/img/import), soft cap 1.5MB
        MCPBridge-->>HermesAgent: artifact + supporting files
    end

    rect rgb(255, 245, 220)
        Note over HermesAgent,ArtifactParser: Streaming output
        HermesAgent->>ACPSession: stdout: { type:'agent_message_chunk', text:'...' }
        ACPSession->>Daemon: AgentEvent text_delta
        Daemon->>Browser: SSE: agent(text_delta)
        HermesAgent->>ACPSession: stdout: { type:'agent_thought_chunk', text:'...' }
        ACPSession->>Daemon: AgentEvent thinking_delta
        Daemon->>Browser: SSE: agent(thinking_delta)
        HermesAgent->>ACPSession: stdout: <artifact> tag in message
        ACPSession->>ArtifactParser: artifact content
        ArtifactParser->>Browser: SSE: artifact:start / artifact:chunk / artifact:end
    end

    alt Normal completion
        HermesAgent->>ACPSession: stdout: { method:'session/close' }
    else Process exit
        HermesAgent->>Daemon: process exit
    end

    Note over ACPSession: stageTimeoutMs: 180s per stage enforced throughout
    Daemon->>Daemon: runs.finish('succeeded')
    Daemon->>Browser: SSE: event:end
```

---

## SD-03: Main Generation Turn (BYOK Proxy Path — No CLI)

When `detectAgents()` finds no agent CLIs on `PATH`, Open Design falls back to a direct BYOK (Bring Your Own Key) proxy mode. The daemon itself proxies the request to the Anthropic API using the user-supplied key from `.od/media-config.json`, inlining the skill content directly into the system prompt rather than relying on agent file-reading. SSRF protection prevents the proxy from being used to reach internal network addresses.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (WebUI)
    participant Daemon
    participant SkillLoader
    participant ByokProxy as BYOK Proxy Handler
    participant AnthropicAPI as Anthropic API
    participant ArtifactParser

    Note over Daemon: detectAgents() on startup → no agents found; BYOK mode active

    User->>Browser: Submit message
    Browser->>Daemon: POST /api/proxy/anthropic/stream { messages, model, systemPrompt }

    rect rgb(255, 220, 220)
        Note over Daemon,ByokProxy: Security & auth checks
        Daemon->>ByokProxy: SSRF check on target host
        Note over ByokProxy: Blocks: 10.x, 172.16-31.x, 192.168.x, 169.254.x
        Daemon->>Daemon: reads OD_OPENAI_API_KEY / OPENAI_API_KEY from .od/media-config.json
    end

    rect rgb(230, 240, 255)
        Note over Daemon,SkillLoader: Skill injection into system prompt
        Daemon->>SkillLoader: findSkillById(skillId)
        SkillLoader-->>Daemon: SKILL.md body
        Daemon->>SkillLoader: loadReferences(skillId)
        SkillLoader-->>Daemon: references/*.md contents
        Note over Daemon: Inlines SKILL.md body + references/*.md into systemPrompt<br/>(prompt injection replaces native agent file-reading)
    end

    rect rgb(230, 255, 230)
        Note over Daemon,AnthropicAPI: Proxied streaming request
        Daemon->>AnthropicAPI: POST https://api.anthropic.com/v1/messages { stream:true, system, messages, model }
        AnthropicAPI-->>Daemon: SSE stream: content_block_start
        AnthropicAPI-->>Daemon: SSE stream: content_block_delta (type:'text_delta')
        AnthropicAPI-->>Daemon: SSE stream: message_delta (stop_reason)
        AnthropicAPI-->>Daemon: SSE stream: message_stop
    end

    rect rgb(255, 245, 220)
        Note over Daemon,ArtifactParser: Normalization & forwarding
        Daemon->>ByokProxy: normalize Anthropic SSE → OD format (delta/end/error)
        ByokProxy->>Browser: SSE: normalized delta events
        ByokProxy->>ArtifactParser: text stream
        ArtifactParser->>Browser: SSE: artifact:start / artifact:chunk / artifact:end
    end

    Note over Daemon,Browser: [Uncertain] BYOK own tool loop mechanics — agent cannot call MCP tools directly
    Browser->>Browser: renders artifact from streamed content
```

---

## SD-04: External Agent Using MCP Bridge

A developer running Claude Code (or another MCP-capable agent) in a separate project can attach to Open Design's read-only MCP server (`od mcp`) to access live design artifacts, project files, and context without exporting anything. The MCP server exposes 8 read-only tools and 3 resources backed by daemon HTTP endpoints.

```mermaid
sequenceDiagram
    autonumber
    participant ExtAgent as External Agent (Claude Code)
    participant MCPServer as od mcp server
    participant Daemon as OD Daemon (HTTP)
    participant SQLite
    participant FileSystem

    Note over ExtAgent,MCPServer: Developer runs: claude --mcp-server od:npx od mcp

    ExtAgent->>MCPServer: MCP tools/list
    MCPServer-->>ExtAgent: tool schemas (list_projects, get_active_context, get_artifact,<br/>get_project, get_file, search_files, list_files, + 3 resources)

    rect rgb(230, 240, 255)
        Note over ExtAgent,MCPServer: Project discovery
        ExtAgent->>MCPServer: tools/call list_projects {}
        MCPServer->>Daemon: GET /api/projects
        Daemon->>SQLite: SELECT * FROM projects
        SQLite-->>Daemon: project rows
        Daemon-->>MCPServer: project list JSON
        MCPServer-->>ExtAgent: [{ id, name, skillId, designSystemId, ... }]
    end

    rect rgb(230, 255, 230)
        Note over ExtAgent,MCPServer: Active context lookup (5-min TTL)
        ExtAgent->>MCPServer: tools/call get_active_context {}
        MCPServer->>Daemon: GET /api/active
        Daemon-->>MCPServer: { projectId, filePath } (current user focus in OD UI)
        MCPServer-->>ExtAgent: active context
    end

    rect rgb(255, 245, 220)
        Note over ExtAgent,MCPServer: Artifact retrieval with BFS traversal
        ExtAgent->>MCPServer: tools/call get_artifact { projectId, filePath }
        MCPServer->>Daemon: GET /api/projects/:id/files/:path
        Daemon->>FileSystem: read artifact file
        FileSystem-->>Daemon: HTML content
        Note over MCPServer: BFS traversal: finds script/link/img/@import references<br/>recursively fetches up to soft cap 1.5MB total
        Daemon-->>MCPServer: artifact HTML + referenced assets
        MCPServer-->>ExtAgent: artifact bundle (design tokens, component patterns)
    end

    rect rgb(240, 230, 255)
        Note over ExtAgent,MCPServer: File search
        ExtAgent->>MCPServer: tools/call search_files { projectId, query:'button' }
        MCPServer->>Daemon: GET /api/projects/:id/search?q=button
        Note over Daemon: literal substring, case-insensitive, max 200 matches
        Daemon-->>MCPServer: [{ file, line, snippet }]
        MCPServer-->>ExtAgent: search results
    end

    Note over ExtAgent: Can now reference OD design tokens and component patterns in generation
```

---

## SD-05: Agent Detection on Daemon Startup

At startup (and on `SIGHUP` for live agent swap), the daemon runs `detectAgents()` to probe all 16 registered agent definitions. For each agent it attempts to resolve the executable, capture its version string and help text, parse capability flags, and optionally enumerate available models. Results are cached and served via `/api/agents`.

```mermaid
sequenceDiagram
    autonumber
    participant Daemon
    participant PathScanner as Path Scanner
    participant AgentDef as AgentDef[0..15]
    participant ChildProcess
    participant AgentCache as Agent Cache (liveModelCache)

    Daemon->>Daemon: startServer() → detectAgents(configuredEnvByAgent)

    rect rgb(230, 240, 255)
        Note over Daemon,AgentDef: Parallel probe loop across all 16 AGENT_DEFs
        loop for each AGENT_DEF (parallel)
            Daemon->>PathScanner: resolveAgentExecutable(def, configuredEnv)
            Note over PathScanner: checks AGENT_BIN_ENV_KEYS first, then PATH
            alt executable not found
                PathScanner-->>Daemon: available: false
            else executable found
                PathScanner-->>Daemon: binPath

                Daemon->>ChildProcess: spawn(bin, versionArgs, { timeout:10s })
                ChildProcess-->>Daemon: version string (e.g. 'claude 1.x.x')

                Daemon->>ChildProcess: spawn(bin, helpArgs, { timeout:10s })
                ChildProcess-->>Daemon: help text

                Daemon->>Daemon: parse help text → capabilityFlags
                Note over Daemon: flags: --add-dir, --include-partial-messages,<br/>--no-cache, etc.

                alt agent has listModels or detectAcpModels
                    Daemon->>ChildProcess: spawn(bin, modelListArgs, { timeout:15s })
                    ChildProcess-->>Daemon: model list JSON
                    Daemon->>AgentCache: cache models per agent
                end

                PathScanner-->>Daemon: { id, bin, version, models, available:true, capabilities }
            end
        end
    end

    Daemon->>Daemon: builds available agents array

    rect rgb(230, 255, 230)
        Note over Daemon: Results available to Web UI
        Daemon->>Daemon: registers GET /api/agents handler
        Note over Daemon: Web: GET /api/agents → agent picker populated
    end

    Note over Daemon: On SIGHUP: detectAgents() re-runs (live agent swap without restart)
```

---

## SD-06: Skill Selection & Prompt Assembly

The skill and design system picker flow, from browser enumeration through user selection to the 10-layer prompt composition that feeds agent spawn. The craft loader optionally appends universal brand-agnostic craft rules declared in the skill's `od.craft.requires` frontmatter field.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (WebUI)
    participant Daemon
    participant SkillRegistry
    participant DSRegistry as DesignSystem Registry
    participant CraftLoader
    participant PromptComposer

    rect rgb(230, 240, 255)
        Note over Browser,SkillRegistry: Skill enumeration
        Browser->>Daemon: GET /api/skills
        Daemon->>SkillRegistry: listSkills(SKILLS_DIR)
        SkillRegistry->>SkillRegistry: scans skills/<folder>/SKILL.md, parses frontmatter
        SkillRegistry-->>Daemon: [{ id, name, mode, scenario, platform, featured, previewType, ... }]
        Daemon-->>Browser: skills array
        Browser->>Browser: renders skill picker (grouped by scenario)
    end

    rect rgb(230, 255, 230)
        Note over Browser,DSRegistry: Design system enumeration
        Browser->>Daemon: GET /api/design-systems
        Daemon->>DSRegistry: listDesignSystems(DS_DIR)
        DSRegistry->>DSRegistry: scans design-systems/<folder>/DESIGN.md<br/>extracts title / category / swatches
        DSRegistry-->>Daemon: [{ id, title, category, swatches, ... }]
        Daemon-->>Browser: design systems array
        Browser->>Browser: renders DS picker with color swatches
    end

    User->>Browser: selects skill
    Browser->>Daemon: PATCH /api/projects/:id { skillId }
    User->>Browser: selects design system
    Browser->>Daemon: PATCH /api/projects/:id { designSystemId }

    rect rgb(255, 245, 220)
        Note over Daemon,PromptComposer: Prompt assembly on next generation turn
        Daemon->>DSRegistry: readDesignSystem(dsId)
        DSRegistry-->>Daemon: full DESIGN.md body
        Daemon->>CraftLoader: loadCraftSections(skill.craftRequires)
        CraftLoader->>CraftLoader: resolves craft/<section>.md files from craft/ dir
        CraftLoader-->>Daemon: concatenated craft body

        Daemon->>PromptComposer: composeSystemPrompt(skill, ds, craft, metadata, opts)
        Note over PromptComposer: Layer 1: DISCOVERY_AND_PHILOSOPHY (always)<br/>Layer 2: OFFICIAL_DESIGNER_PROMPT (always)<br/>Layer 3: Active DESIGN.md body (always)<br/>Layer 4: derivePreflight() preamble + SKILL.md body (always)<br/>Layer 5: metadata block (if metadata present)<br/>Layer 6: deck framework directive (if mode:deck)<br/>Layer 7: media generation contract (if media enabled)<br/>Layer 8: Codex imagegen override (if Codex agent)<br/>Layer 9: critique theater addendum (if critique enabled)
        PromptComposer-->>Daemon: composed system prompt string

        Daemon->>Daemon: buildArgs() → argv array
        Note over Daemon: prompt via -p flag (Claude Code, Copilot)<br/>or stdin pipe (Codex/Gemini/OpenCode/Qwen/Qoder/Pi)
    end
```

---

## SD-07: Critique Theater Multi-Round Loop

The Critique Theater is an optional multi-round self-critique loop activated by `OD_CRITIQUE_ENABLED=true`. A critique addendum (Layer 9 of the system prompt) instructs the agent to emit structured panelist XML scoring output on five dimensions (Philosophy, Hierarchy, Detail, Function, Innovation). The daemon-side orchestrator is authoritative for composite score computation and decides whether to iterate or ship.

```mermaid
sequenceDiagram
    autonumber
    participant Browser as Browser (WebUI)
    participant Daemon
    participant CritiqueOrch as CritiqueOrchestrator
    participant AgentCLI as Agent CLI
    participant Parser as Stream Parser
    participant SQLite

    Note over Daemon: critique.enabled=true (OD_CRITIQUE_ENABLED=true)
    Daemon->>Daemon: critique addendum injected into system prompt (Layer 9)
    Daemon->>AgentCLI: child_process.spawn(agentBin, args, { cwd })
    Daemon->>CritiqueOrch: attach to stdout stream

    rect rgb(230, 240, 255)
        Note over CritiqueOrch,Parser: Round 1 generation
        AgentCLI->>Parser: streams text output + artifact HTML
        AgentCLI->>Parser: emits <ROUND n="1">

        Note over AgentCLI,Parser: [Uncertain] Exact panelist XML tag format
        AgentCLI->>Parser: emits <PANELIST role="designer"><br/>  ...<br/>  <SCORE dim="philosophy">8</SCORE><br/>  <SCORE dim="hierarchy">7</SCORE><br/>  <SCORE dim="detail">6</SCORE><br/>  <SCORE dim="function">9</SCORE><br/>  <SCORE dim="innovation">7</SCORE><br/></PANELIST>

        Parser->>CritiqueOrch: panelist_open → panelist_dim events → panelist_close
        CritiqueOrch->>CritiqueOrch: computes composite score (daemon-authoritative weighted avg)
        CritiqueOrch->>Browser: SSE: critique event (live score display, round=1)
    end

    rect rgb(230, 255, 230)
        Note over CritiqueOrch: Ship / iterate decision
        alt composite >= scoreThreshold
            CritiqueOrch->>CritiqueOrch: decideRound() → SHIP
        else composite < scoreThreshold
            CritiqueOrch->>AgentCLI: continue to next round (still in same process turn)
            Note over AgentCLI,CritiqueOrch: Repeat Round N until threshold met or max rounds reached
        end
    end

    AgentCLI->>Parser: emits <SHIP round="1" composite="8.2" status="shipped">
    Parser->>CritiqueOrch: ship event
    CritiqueOrch->>CritiqueOrch: verifies ship round completed
    CritiqueOrch->>CritiqueOrch: checks composite divergence (agent-reported vs daemon-computed)
    CritiqueOrch->>CritiqueOrch: final status: shipped / below_threshold

    rect rgb(255, 230, 230)
        Note over CritiqueOrch: Abort / timeout handling
        alt abort or timeout before any round completes
            CritiqueOrch->>CritiqueOrch: no best-so-far → fail
        else abort/timeout after at least one round
            CritiqueOrch->>CritiqueOrch: ships best-so-far artifact
        end
    end

    CritiqueOrch->>SQLite: persists run record
    CritiqueOrch->>CritiqueOrch: emits transcript → .critique.jsonl
    CritiqueOrch->>Browser: SSE: final critique summary event
    Daemon->>Browser: SSE: event:end
```

---

## SD-08: Export Pipeline (HTML/PDF/PPTX/ZIP)

The export pipeline is triggered from the browser's export menu. The daemon reads the artifact file from the project directory and transforms it according to the requested format. ZIP bundles all visible project files; HTML inlines external assets as base64 data URIs; PDF and PPTX generation details are partially uncertain.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (WebUI)
    participant Daemon
    participant FileSystem
    participant ExportHandler

    User->>Browser: clicks Export (selects format)
    Browser->>Daemon: POST /api/projects/:id/files/:filename/export?format=<fmt>

    Daemon->>FileSystem: read artifact from .od/projects/<id>/<filename>
    FileSystem-->>Daemon: raw HTML content

    Daemon->>ExportHandler: dispatch on format

    rect rgb(230, 240, 255)
        Note over ExportHandler: format = html
        ExportHandler->>ExportHandler: scan for external CSS/JS/image references
        ExportHandler->>ExportHandler: base64-encode external assets → inline data URIs
        ExportHandler-->>Daemon: complete self-contained HTML string
    end

    rect rgb(230, 255, 230)
        Note over ExportHandler: format = pdf
        Note over ExportHandler: [Uncertain] browser triggers window.print()<br/>or headless Puppeteer render
        ExportHandler-->>Daemon: PDF buffer
    end

    rect rgb(255, 245, 220)
        Note over ExportHandler: format = pptx (deck mode)
        Note over ExportHandler: [Uncertain] JSON slide definitions → pptxgenjs → .pptx buffer
        ExportHandler-->>Daemon: PPTX buffer
    end

    rect rgb(255, 230, 230)
        Note over ExportHandler: format = zip
        ExportHandler->>FileSystem: list visible project files (respects MIME allowlist)
        ExportHandler->>ExportHandler: archiver → zip entries
        ExportHandler-->>Daemon: ZIP buffer
    end

    rect rgb(240, 230, 255)
        Note over ExportHandler: format = markdown
        ExportHandler->>ExportHandler: direct copy or skill-defined render
        ExportHandler-->>Daemon: Markdown string
    end

    Daemon->>Browser: HTTP 200 { Content-Disposition: attachment; filename=<name>.<ext>, body: buffer }
    Browser->>Browser: triggers native file download dialog
```

---

## SD-09: Pi Agent Session (pi-rpc)

Pi uses a custom stdio JSON-RPC protocol (`pi-rpc`) distinct from both ACP and the JSONL stream format. Key differences from other agents include explicit image forwarding (max 10 images, 20 MB total with per-file validation), a separate `extension_ui_request` auto-reply mechanism, and a configurable graceful shutdown window.

```mermaid
sequenceDiagram
    autonumber
    participant Browser as Browser (WebUI)
    participant Daemon
    participant PiRpc as PiRpcHandler (attachPiRpcSession)
    participant PiCLI as Pi CLI Process

    Browser->>Daemon: POST /api/chat { agentId:'pi', prompt, model, images:[] }
    Daemon->>PiCLI: child_process.spawn('pi', args, { cwd:.od/projects/<id>/ })
    Daemon->>PiRpc: attachPiRpcSession({ child, prompt, model, imagePaths })

    rect rgb(230, 240, 255)
        Note over PiRpc,PiCLI: Pi RPC handshake
        PiRpc->>PiCLI: stdin: { jsonrpc:'2.0', method:'initialize' }
        PiCLI-->>PiRpc: stdout: { ok: true }
        PiRpc->>PiCLI: stdin: { method:'session/new' }
        PiCLI-->>PiRpc: stdout: { sessionId: '...' }
        PiRpc->>PiCLI: stdin: { method:'session/set_model', params:{ model } }
        PiCLI-->>PiRpc: stdout: { ok: true }
    end

    rect rgb(230, 255, 230)
        Note over PiRpc: Image validation (if imagePaths provided)
        PiRpc->>PiRpc: realpath check (must be inside uploadRoot)
        PiRpc->>PiRpc: count check (≤ 10 files)
        PiRpc->>PiRpc: total size check (≤ 20 MB)
        PiRpc->>PiRpc: extension check (allowed MIME types only)
        PiRpc->>PiRpc: base64-encode each approved image
    end

    PiRpc->>PiCLI: stdin: { method:'session/prompt', params:{ prompt, images:[{type:'image',data:base64,...}] } }

    rect rgb(255, 245, 220)
        Note over PiCLI,PiRpc: Streaming events from Pi
        PiCLI->>PiRpc: stdout: { type:'agent_start' }
        PiRpc->>Daemon: AgentEvent status:streaming
        Daemon->>Browser: SSE: agent(status:streaming)

        PiCLI->>PiRpc: stdout: { type:'message_update', text:'...' }
        PiRpc->>Daemon: AgentEvent text_delta
        Daemon->>Browser: SSE: agent(text_delta)

        PiCLI->>PiRpc: stdout: { type:'tool_execution_start', name:'...' }
        PiRpc->>Daemon: AgentEvent tool_use
        Daemon->>Browser: SSE: agent(tool_use)

        PiCLI->>PiRpc: stdout: { type:'tool_execution_end', result:{...} }
        PiRpc->>Daemon: AgentEvent tool_result
        Daemon->>Browser: SSE: agent(tool_result)

        PiCLI->>PiRpc: stdout: { type:'extension_ui_request', ... }
        Note over PiRpc: auto-replies to dialog/UI requests (not forwarded to browser)
        PiRpc->>PiCLI: stdin: { method:'extension_ui_response', ... }

        PiCLI->>PiRpc: stdout: { type:'agent_end', usage:{...} }
        PiRpc->>Daemon: AgentEvent usage
        Daemon->>Browser: SSE: agent(usage)
    end

    Daemon->>Daemon: runs.finish('succeeded')
    Daemon->>Browser: SSE: event:end

    rect rgb(255, 230, 230)
        Note over PiRpc,PiCLI: Cancellation & shutdown
        alt user cancels
            PiRpc->>PiCLI: stdin: abort signal
            Note over PiRpc: waits PI_ABORT_GRACE_MS (3s) for clean stop
            PiRpc->>PiCLI: SIGTERM
        end
        alt SIGTERM to daemon
            Note over Daemon,PiCLI: PI_GRACEFUL_SHUTDOWN_MS (5s) grace period before SIGKILL
        end
    end
```

---

## SD-10: Live Artifact Refresh

Live artifacts are HTML templates with real-time data bindings that can be refreshed on demand or on a schedule. A refresh run fetches fresh data from a connector source, validates it against bounded JSON constraints, and optionally spawns a mini agent turn to re-render the template with the new data.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Browser as Browser (WebUI)
    participant Daemon
    participant LiveStore as LiveArtifactStore
    participant ConnectorSvc as ConnectorService
    participant FileSystem
    participant AgentCLI as Agent CLI (mini-turn)

    User->>Browser: views live artifact dashboard
    Browser->>Daemon: GET /api/projects/:id/live-artifacts
    Daemon->>LiveStore: list live artifacts for project
    LiveStore-->>Daemon: [{ id, name, templatePath, lastRefreshed, refreshStatus, ... }]
    Daemon-->>Browser: live artifacts array

    User->>Browser: clicks "Refresh" on artifact
    Browser->>Daemon: POST /api/projects/:id/live-artifacts/:artifactId/refresh

    Daemon->>LiveStore: updateRefreshStatus(artifactId, 'running')
    Daemon->>Browser: SSE: { type:'live_artifact_refresh', phase:'started', artifactId }

    rect rgb(230, 240, 255)
        Note over Daemon,FileSystem: Template & data loading
        Daemon->>FileSystem: read template.html from .od/projects/<id>/.live-artifacts/<artifactId>/
        FileSystem-->>Daemon: template HTML
    end

    rect rgb(230, 255, 230)
        Note over Daemon,ConnectorSvc: Fresh data fetch
        Note over ConnectorSvc: [Uncertain] connector tool, Composio, or daemon tool
        Daemon->>ConnectorSvc: fetchData(artifact.connectorConfig)
        ConnectorSvc-->>Daemon: raw data payload

        Daemon->>Daemon: resolves output mapping
        Note over Daemon: identity | compact_table | metric_summary
    end

    rect rgb(255, 245, 220)
        Note over Daemon: Data validation
        Daemon->>Daemon: validate against BoundedJsonConstraints
        Note over Daemon: max 256KB, depth 8, key count limits, no circular refs
        Daemon->>FileSystem: write updated data.json to .live-artifacts/<artifactId>/
    end

    alt refresh requires agent re-render
        rect rgb(240, 230, 255)
            Note over Daemon,AgentCLI: Mini agent turn
            Daemon->>AgentCLI: spawn mini-turn with template + fresh data context
            AgentCLI->>AgentCLI: reads template.html + data.json
            AgentCLI->>AgentCLI: writes updated index.html
            AgentCLI->>Daemon: emits <artifact> in output stream
            AgentCLI->>Daemon: process exit
        end
    else data-only update (no agent needed)
        Daemon->>Daemon: merge data into template bindings directly
        Daemon->>FileSystem: write index.html
    end

    Daemon->>LiveStore: updateRefreshStatus(artifactId, 'succeeded')
    Daemon->>Browser: SSE: { type:'live_artifact_refresh', phase:'completed', artifactId }
    Browser->>Browser: updates preview iframe srcdoc with refreshed artifact

---

## Notes on Uncertainty

The following steps in the diagrams above are marked `[Uncertain]` because they could not be directly confirmed from the source files referenced in the architecture context:

| Diagram | Step | Uncertainty |
|---------|------|-------------|
| SD-03 | BYOK tool loop mechanics | Whether the BYOK proxy supports any form of agentic tool loop or is strictly single-shot completion |
| SD-07 | Panelist XML tag format | The exact XML schema for `<PANELIST>`, `<SCORE>`, `<ROUND>`, `<SHIP>` tags emitted by the agent |
| SD-08 | PDF export implementation | Whether PDF is generated server-side (Puppeteer) or delegated to the browser's print dialog |
| SD-08 | PPTX export implementation | Whether PPTX conversion uses pptxgenjs directly or an intermediate JSON slide format |
| SD-10 | Connector data fetching | The exact connector abstraction used (Composio, daemon tool, or custom connector protocol) |
```
