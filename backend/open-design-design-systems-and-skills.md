# Open Design Design Systems & Skills

## Executive Summary

Open Design uses two complementary content registries — **design systems** and **skills** — that are combined at runtime to build the system prompt delivered to an AI agent. A design system (a `DESIGN.md` file inside a folder under `design-systems/`) defines the visual language: palette, typography, component tokens, and layout rules. A skill (a `SKILL.md` file inside a folder under `skills/`) defines the agent workflow: what kind of artifact to build, which reference files to read, and which universal craft rules to enforce. At chat time the daemon merges them through `composeSystemPrompt()` in `apps/daemon/src/prompts/system.ts`, producing a layered context that governs every agent turn.

---

## DESIGN.md Schema

Every design system is a single Markdown file at `design-systems/<id>/DESIGN.md`. The file has no YAML frontmatter. Structure is purely Markdown:

```
# <Title>

> Category: <category name>
> [Optional one-line description blockquote]

## Visual Theme & Atmosphere
## Color Palette & Roles
## Typography Rules
## Component Stylings
## Layout Principles
## Depth & Elevation
## Do's and Don'ts
## Responsive Behavior
## Agent Prompt Guide
```

### The 9 Body Sections

| Section | Purpose |
|---|---|
| **Visual Theme & Atmosphere** | Adjective-level mood description. Tells the agent what feeling the output should evoke ("calm, functional, quietly confident"). Not a spec — a design intent. |
| **Color Palette & Roles** | Named hex values with semantic roles (Background, Foreground, Accent, Muted, Border, Surface, Success, Warn, Danger). Each value is quoted in backticks so the daemon's swatch extractor can parse them. |
| **Typography Rules** | Font stack for display/body/mono, weight scale, size scale in px, line-height, and letter-spacing rules. |
| **Component Stylings** | Token-level specs for Buttons, Cards, Inputs, Links — radius, padding, border, focus states. |
| **Layout Principles** | Grid column count, max-width, gutter size. Hero height. Section spacing at each breakpoint. Use of whitespace vs dividers. |
| **Depth & Elevation** | Shadow levels (typically two: flat and raised). Explicit bans on neumorphism and glassmorphism. |
| **Do's and Don'ts** | Bullet checklist: things the agent must do (sentence-case headings, one accent element per screen) and must never do (gradients on backgrounds, pure black/white). |
| **Responsive Behavior** | Breakpoint table mapping viewport widths to column counts and gutters. |
| **Agent Prompt Guide** | Imperative instructions written directly to the agent — "when in doubt, subtract", "do not invent hex values outside this palette". This section overrides softer wording in the base prompt on the named topic. |

### Example (from `design-systems/default/DESIGN.md`)

```markdown
# Neutral Modern

> Category: Starter

## Color Palette & Roles
- **Background:** `#FAFAFA`
- **Foreground:** `#111111`
- **Accent:** `#2F6FEB` (cobalt)
- **Muted:** `#6B6B6B`
- **Border:** `#E5E5E5`
- **Surface:** `#FFFFFF`
- **Success:** `#17A34A`, **Warn:** `#EAB308`, **Danger:** `#DC2626`
```

---

## SKILL.md Protocol

Every skill is a Markdown file with YAML frontmatter at `skills/<id>/SKILL.md`. The frontmatter lives under the `od:` namespace.

### Top-Level Fields (outside `od:`)

| Field | Type | Purpose |
|---|---|---|
| `name` | string | Canonical skill ID. Used as the primary key throughout the system. Stored in `project.skill_id`. Must be stable — renames require an entry in `SKILL_ID_ALIASES`. |
| `description` | string | One-paragraph description shown in the skill picker. |
| `triggers` | string[] | Keyword phrases that signal this skill should be the default recommendation (e.g. `"landing"`, `"prototype"`). |

### `od:` Namespace Fields

| Field | Type | Purpose |
|---|---|---|
| `od.mode` | `prototype` \| `deck` \| `template` \| `design-system` \| `image` \| `video` \| `audio` | Artifact type. Drives which prompt layers are added (deck framework, media contract, Codex imagegen override). |
| `od.platform` | `web` \| `mobile` \| `desktop` | Runtime target. Used for picker grouping and inferred if omitted. |
| `od.scenario` | `operations` \| `design` \| `engineering` | Use-case category. Used for picker grouping. |
| `od.preview.type` | `html` \| `jsx` \| `pptx` \| `markdown` | How the web UI renders the artifact. |
| `od.preview.entry` | string | Filename to load in the preview iframe (e.g. `index.html`). |
| `od.design_system.requires` | boolean | Whether the skill requires a design system to be active. Defaults to `true`. |
| `od.design_system.sections` | string[] | Which DESIGN.md sections the skill depends on (e.g. `[color, typography, layout, components]`). Future: allows selective injection. |
| `od.craft.requires` | string[] | Slugs of craft reference files to load and inject (e.g. `[typography, anti-ai-slop]`). Each slug maps to `craft/<slug>.md`. |
| `od.default_for` | string \| string[] | Scenario or mode this skill is the default for. Used by the UI to pre-select a skill when a project is created with a matching type. |
| `od.upstream` | string | ID of a skill this skill extends or is a variant of. |
| `od.featured` | boolean \| string[] | Whether the skill appears in the featured row of the picker. |
| `od.fidelity` | string | Hint for the new-project panel: default fidelity level (`wireframe`, `low`, `high`). |
| `od.speaker_notes` | boolean | Hint: default speaker-notes toggle for deck skills. |
| `od.animations` | boolean | Hint: default animation toggle. |
| `od.example_prompt` | string (derived from `description`) | Pre-filled prompt shown in the "Use this prompt" fast-create flow. |

### Example (from `skills/web-prototype/SKILL.md`)

```yaml
---
name: web-prototype
description: |
  General-purpose desktop web prototype. ...
triggers:
  - "prototype"
  - "landing"
od:
  mode: prototype
  platform: desktop
  scenario: design
  preview:
    type: html
    entry: index.html
  design_system:
    requires: true
    sections: [color, typography, layout, components]
---
```

---

## How Design Systems Are Selected

1. The web UI calls `GET /api/design-systems` on page load or when the user opens the design system picker.
2. The daemon's `listDesignSystems(DESIGN_SYSTEMS_DIR)` in `apps/daemon/src/design-systems.ts` scans all subdirectories, reads each `DESIGN.md`, and returns a list of objects: `{ id, title, category, summary, swatches, surface, body }`.
   - `id` is the folder name (e.g. `default`, `linear-app`).
   - `title` is the H1, with the prefix `"Design System (Inspired by|for)"` stripped.
   - `category` comes from the `> Category: <name>` blockquote line.
   - `swatches` is an array of up to 4 hex colors extracted by a greedy regex scan.
3. The web UI renders a picker grouped by category, showing the swatch strip next to each system name.
4. When the user selects a system, its `id` is stored on the project (`project.design_system_id`).
5. When a chat turn starts, the daemon fetches the full `DESIGN.md` body via `readDesignSystem(root, id)` and passes it as `designSystemBody` to `composeSystemPrompt()`.
6. `composeSystemPrompt()` injects the body under the heading `## Active design system — <title>` with an authoritative preamble: "Treat the following DESIGN.md as authoritative for color, typography, spacing, and component rules."

---

## How Skills Are Selected

1. The web UI calls `GET /api/skills`. The daemon calls `listSkills(skillsRoot)` for each discovery root in order:
   - `./.claude/skills/` (project-private, highest priority)
   - `./skills/` (project-committed, shared)
   - `~/.claude/skills/` (user-global, lowest priority)
2. Each `SKILL.md` is read, frontmatter is parsed by `parseFrontmatter()`, and a skill object is assembled with all normalized fields.
3. Any skill whose `name` matches a key in `SKILL_ID_ALIASES` is resolved to its canonical id before being exposed.
4. The web UI renders a picker grouped by `scenario` and `mode`. Featured skills appear in a top row.
5. When the user selects a skill, its `id` is stored on the project (`project.skill_id`).
6. When a chat turn starts, `findSkillById(skills, skillId)` resolves aliases and retrieves the skill object. Its `body` field (the non-frontmatter portion of `SKILL.md`) is passed as `skillBody` to `composeSystemPrompt()`.
7. If the skill folder contains side files (anything other than `SKILL.md`), `withSkillRootPreamble()` prepends a preamble to the body advertising both the cwd-relative `.od-skills/<folder>/` path and the absolute repo path.

---

## How Prompts Are Assembled

`composeSystemPrompt(input: ComposeInput)` in `apps/daemon/src/prompts/system.ts` builds a single string by appending parts in a fixed order. Layer precedence flows top-to-bottom: earlier layers are "background" that later layers can override.

### Layer Order

| # | Layer | Content | Condition |
|---|---|---|---|
| 1 | **DISCOVERY** | `DISCOVERY_AND_PHILOSOPHY` from `./discovery.ts` — turn-1 form syntax, direction-picker, brand-spec extraction, TodoWrite reinforcement, 5-dim critique protocol. | Always |
| 2 | **BASE_SYSTEM** | `OFFICIAL_DESIGNER_PROMPT` from `./official-system.ts` — full identity, workflow, content-philosophy charter. | Always |
| 3 | **Design System** | `## Active design system — <title>` + full `DESIGN.md` body. "Do not invent tokens outside this palette." | When `designSystemBody` is non-empty |
| 4 | **Craft** | `## Active craft references — <slugs>` + concatenated `craft/*.md` sections. Brand wins on token conflicts; craft governs everything the brand doesn't override. | When `craftBody` is non-empty |
| 5 | **Skill + Preflight** | `## Active skill — <name>` + `derivePreflight()` inline directive + full `SKILL.md` body. | When `skillBody` is non-empty |
| 6 | **Metadata Block** | Project intent, fidelity, speaker notes, animation flags, template files. | When metadata / template present |
| 7 | **Deck Framework** | `DECK_FRAMEWORK_DIRECTIVE` — load-bearing nav/counter/scroll JS/print stylesheet contract. | When `skillMode === 'deck'` OR `metadata.kind === 'deck'` AND no skill seed |
| 8 | **Media Contract** | `MEDIA_GENERATION_CONTRACT` — imagegen/video/audio generation contract. | When mode is `image`, `video`, or `audio` |
| 9 | **Codex Imagegen Override** | Codex-specific built-in imagegen path. | When agent is Codex + image mode + GPT-image model |
| 10 | **Critique Addendum** | `renderPanelPrompt()` — Critique Theater protocol (requires enabled config, brand, skill, non-media surface). | When critique is enabled |

### ComposeInput Fields

```typescript
interface ComposeInput {
  agentId?: string;
  includeCodexImagegenOverride?: boolean;
  skillBody?: string;
  skillName?: string;
  skillMode?: 'prototype' | 'deck' | 'template' | 'design-system' | 'image' | 'video' | 'audio';
  designSystemBody?: string;
  designSystemTitle?: string;
  craftBody?: string;
  craftSections?: string[];
  metadata?: ProjectMetadata;
  template?: ProjectTemplate;
  critique?: CritiqueConfig;
  critiqueBrand?: { name: string; design_md: string };
  critiqueSkill?: { id: string };
}
```

---

## Pre-flight Side-File Loading

### `derivePreflight(skillBody)`

Called inside `composeSystemPrompt()` after the skill body is added. Scans the skill body for references to known side-file paths. If any are found, it appends a hard directive to the skill section header:

> **Pre-flight (do this before any other tool):** Read `assets/template.html`, `references/layouts.md`, `references/checklist.md` via the path written in the skill-root preamble. The seed template defines the class system you'll paste into; the layouts file is the only acceptable source of section/screen/slide skeletons; the checklist is your P0/P1/P2 gate before emitting `<artifact>`. Skipping this step is the #1 reason output regresses to generic AI-slop.

Recognized patterns: `assets/template.html`, `references/layouts.md`, `references/themes.md`, `references/components.md`, `references/checklist.md`.

### `withSkillRootPreamble(body, dir)` in `apps/daemon/src/skills.ts`

Prepended to the skill body when `dirHasAttachments()` detects non-SKILL.md files in the skill folder. Provides two path forms:

- **CWD-relative:** `.od-skills/<folder>/` — primary. Agents use this because it is inside the project cwd and not blocked by directory-access policies.
- **Absolute fallback:** the real repo path (e.g. `/path/to/skills/web-prototype/`). Used when staging fails or for `/api/runs` without a project.

Also lists detected side files so the agent knows what is available.

### `.od-skills/<id>/` Staging via `stageActiveSkill()`

Before each agent spawn, `stageActiveSkill(cwd, folderName, sourceDir)` in `apps/daemon/src/cwd-aliases.ts` copies the entire skill folder into `<projectCwd>/.od-skills/<folderName>/`. Key properties:

- **Idempotent:** previous-turn copy is replaced wholesale, so skill edits mid-session are picked up.
- **Not a symlink:** a full copy prevents agents from writing back to the source skill via symlink traversal.
- **`dereference: true`:** follows symlinks in the source so content-addressable mounts work correctly.
- **Non-throwing:** failures are surfaced as a result object, never thrown; the caller falls back to the absolute-path preamble.

The constant `SKILLS_CWD_ALIAS = '.od-skills'` is exported from `cwd-aliases.ts` and consumed by `skills.ts` when building the preamble.

---

## Craft References

### craft/ Folder

Location: `<projectRoot>/craft/` (the root of the Open Design repo). The daemon's resource root can be overridden with `OD_RESOURCE_ROOT`.

Bundled craft files:

| Slug | File | Content |
|---|---|---|
| `typography` | `craft/typography.md` | Letter-spacing caps, font-weight discipline, heading-hierarchy rules. |
| `color` | `craft/color.md` | Accent budget, contrast ratios, forbidden palette patterns. |
| `anti-ai-slop` | `craft/anti-ai-slop.md` | P0 list of AI-slop tropes: purple gradients, emoji feature icons, invented metrics, lorem filler, blue→cyan trust gradients. |
| `state-coverage` | `craft/state-coverage.md` | Interactive state completeness: hover, focus, disabled, loading, error, empty. |
| `animation-discipline` | `craft/animation-discipline.md` | Motion rules: duration budgets, easing curves, `prefers-reduced-motion`. |
| `accessibility-baseline` | `craft/accessibility-baseline.md` | WCAG AA minimum: color contrast, alt text, focus ring, ARIA roles. |
| `rtl-and-bidi` | `craft/rtl-and-bidi.md` | Right-to-left layout: logical properties, text direction, bidirectional text. |
| `form-validation` | `craft/form-validation.md` | Inline validation patterns, error message placement, field recovery flows. |
| `laws-of-ux` | `craft/laws-of-ux.md` | Fitts' Law, Hick's Law, proximity, and other applied UX laws. |

### `loadCraftSections(craftDir, requested)` in `apps/daemon/src/craft.ts`

Takes the craft directory path and the list of slugs from `od.craft.requires`. For each slug:

1. Validates the slug format (`/^[a-z0-9][a-z0-9-]*$/`).
2. Reads `<craftDir>/<slug>.md`.
3. Prepends a `### <slug>` header.
4. Missing files are silently skipped — forward compatibility allows a skill to list a future slug without breaking.

Returns `{ body, sections }` where `body` is the concatenated Markdown (sections separated by `---`) and `sections` is the list of slugs that actually resolved.

### Auto-enforcement of `anti-ai-slop`

`craft/anti-ai-slop.md` is also wired into `apps/daemon/src/lint-artifact.ts` as the P0 check list. When an artifact is saved via `POST /api/artifacts/save`, the linter runs and returns structured findings. P0 findings are fed back to the agent as a system message so it self-corrects on the next turn without user intervention. The lint endpoint is also exposed standalone at `POST /api/artifacts/lint` so the chat UI can surface badges.

---

## DESIGN.md Convention

Refer to the 9-section schema documented above. Key authoring rules:

- The H1 title is the human-readable name. Avoid the boilerplate prefix "Design System Inspired by" — `cleanTitle()` strips it, but cleaner filenames are preferred.
- The `> Category: <name>` line must appear as a blockquote immediately after the H1. It is parsed by `extractCategory()`.
- Color values must be quoted in backticks (`` `#HEXVAL` ``) for the swatch extractor.
- The Agent Prompt Guide section should use imperative voice addressed directly to the agent.
- No YAML frontmatter. No `<script>` or `<style>` blocks. Pure Markdown only.

---

## Adding a New Design System

1. Create a folder under `design-systems/<your-id>/`.
2. Create `design-systems/<your-id>/DESIGN.md` following the 9-section schema above.
3. Add the `> Category: <name>` blockquote immediately after the H1.
4. Include at least the Color Palette and Typography Rules sections with backtick-quoted hex values.
5. No additional registration is needed. `listDesignSystems()` discovers by directory scan on every `GET /api/design-systems`.
6. Restart the daemon (or rely on the hot-scan-on-request behavior) — no build step.

Optional: if the design system targets a non-web surface (image generation, video), add a `> Surface: image` blockquote line.

---

## Adding a New Skill

1. Create a folder under `skills/<your-id>/` (or `.claude/skills/<your-id>/` for project-private, `~/.claude/skills/<your-id>/` for user-global).
2. Create `skills/<your-id>/SKILL.md` with YAML frontmatter and the `od:` namespace fields described above.
3. Set `name:` to the stable identifier — this is what gets stored in `project.skill_id`.
4. Add `triggers:` keywords that describe when this skill should be auto-suggested.
5. Optionally create `assets/template.html` (seed file) and `references/` Markdown files. If these exist, `withSkillRootPreamble()` automatically advertises their paths to the agent.
6. Optionally create `references/checklist.md` for the agent's P0/P1/P2 self-review gate.
7. Set `od.craft.requires` to a list of craft slugs the skill needs.
8. No registration needed. `listSkills()` discovers by directory scan.
9. If you renamed an existing skill, add the old `name` → new `name` mapping to `SKILL_ID_ALIASES` in `apps/daemon/src/skills.ts` and keep it for at least one stable release.

---

## Common Mistakes

1. **Using `od.craft.requires` slugs with uppercase or spaces.** `normalizeCraftRequires()` enforces lowercase alphanumeric-dash format. `"Anti-AI-Slop"` will be silently dropped; use `"anti-ai-slop"`.

2. **Forgetting `> Category: <name>` in DESIGN.md.** Without this line, `extractCategory()` returns `null` and the system defaults to `"Uncategorized"`. The picker will still work but grouping breaks.

3. **Renaming a skill's `name:` without adding the old ID to `SKILL_ID_ALIASES`.** Existing projects saved against the old ID will silently compose without any skill body until the alias is added.

4. **Naming a skill folder differently from its `name:` frontmatter field.** The `id` and `name` in the listing both come from `data.name || entry.name`. If they differ, `findSkillById()` resolves by `name`, not by folder name.

5. **Placing a `<script>` block or YAML frontmatter in a DESIGN.md.** The design-system parser reads raw Markdown; neither HTML nor YAML is parsed or sanitized. Script blocks will be injected verbatim into the system prompt.

6. **Referencing side files in the SKILL.md body without creating the files.** `dirHasAttachments()` detects the skill has side files only if non-SKILL.md content is present in the directory. If the body references `assets/template.html` but the file does not exist, the agent will try to read it and fail silently.

---

## Open Questions

1. **Design system section filtering.** The `od.design_system.sections` field is stored in the skill listing but `composeSystemPrompt()` currently injects the full `DESIGN.md` body regardless. Whether section filtering is implemented server-side or left to the agent to apply is [Uncertain].

2. **User-global skill directory priority.** The discovery order documented in AGENTS.md (`~/.claude/skills/` as lowest priority) is the specification, but whether the daemon actually iterates all three roots in a single merged `listSkills()` call or merges per-request is [Uncertain — the server.ts implementation merges at call time but the exact deduplication behavior for same-name skills across roots has not been verified].

3. **Craft section ordering.** `loadCraftSections()` processes slugs in the order they appear in `od.craft.requires`. Whether the relative order of brand DESIGN.md vs craft sections within the prompt affects agent output meaningfully is not documented. The current implementation always places design system before craft.
