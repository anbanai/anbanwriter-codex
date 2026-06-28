# CODEX.md

This file provides guidance to OpenAI Codex (codex CLI / IDE) when working with code in this repository.

## Project Overview

**anbanwriter-codex** is the Codex-native port of the Claude Code plugin `claudecode/`. It targets the same workflows:

- **WeChat Official Account articles** (微信公众号图文)
- **SeedNote posts** (种草笔记)
- **Live video slicing** (直播切片)
- **Line art coloring** (线稿上色)
- **Short video cover** (短视频封面复刻 + 人像姿态变体)

It connects to the same `anbanwriter` MCP server as `claudecode/` and `openclaw/`. Content themes, writers, layouts, and the MCP protocol are identical across all three plugins.

## Architecture

The plugin follows Codex's **Skill + Subagent + MCP** model:

- **Skills** (`skills/`) — auto-discovered by Codex (23 leaf skills, identical content to `claudecode/skills/`; plus 1 reference-only directory `writers/` with no `SKILL.md`)
- **Subagents** (`agents/`) — six TOML files installed to `~/.codex/agents/` and registered in `~/.codex/config.toml` (Codex plugins cannot bundle subagents directly — see GitHub issue #18988)
- **MCP server** (`.mcp.json`) — HTTP endpoint at `${ANBAN_API_URL:-https://api.creator.anbanai.com}/mcp`
- **Hooks** (`hooks/hooks.json`) — lifecycle events mirroring `claudecode/hooks/hooks.json`, with `TaskCompleted` replaced by `Stop` (Codex has no `TaskCompleted` event)

### Subagents (`agents/`)

| Subagent | Triggers | Pipeline |
|----------|----------|----------|
| `wechatarticle` | "写文章", "发文章", "公众号文章" | Research → Write → De-AI → SEO → Cover → Illustrations → HTML → Draft |
| `seednote` | "种草笔记", "种草", "复刻", "仿写" | Research → Viral analysis (replicate) → Content → Image plan → Cover + Content images → Compliance → Archive |
| `live-slicer` | "直播切片", "剪直播", "听悟" | ffmpeg prep → TingWu transcription → Invalid sentence filter → Segment/subject planning → Batch cuts/concat → CapCut export → Report |
| `designer` | "上色", "填色", "线稿", "color consistency", "designer" | Init → Progressive coloring → Full audit → Best-effort correction/backtracking → Report with `needs_img2img` where strict line preservation is impossible |
| `short-video-studio` | "短视频封面", "爆款封面", "封面复刻", "人像姿态", "表情封面" | Intent routing → Workspace init → Mode branch: replication (short-video-cover skill) or pose-variants (portrait-pose-variants skill) → Generation + audit → Archive report |

**Codex-specific behavior**:
- Subagents only spawn when the user **explicitly** asks ("use the wechatarticle subagent to ...", "delegate to X"). Codex does not auto-spawn subagents.
- Each subagent declares its own `[mcp_servers.anban]` and `[[skills.config]]` — it does **not** inherit MCP servers or skills from the parent session.
- Multi-agent mode requires `[features] multi_agent = true` in `~/.codex/config.toml` (the install script adds this automatically).
- `[agents] max_threads = 6` bounds concurrent subagent execution.

### Skills (`skills/`)

Identical content to `claudecode/skills/`. Each skill has a `SKILL.md` with YAML frontmatter (`name` + `description`). Codex loads skills implicitly based on `description` matching the user's prompt — no explicit declaration needed.

Key skill groups:
- **Content**: `content-writing`, `topic-research`, `seo-optimization`
- **WeChat article**: `article`, `article-visual-design`, `article-publishing`
- **SeedNote**: `seednote`, `seednote-research`, `seednote-viral-analysis`, `seednote-writing`, `seednote-visual-design`
- **Live slicing**: `live-slice`, `capcut-draft`
- **Design**: `line-art-coloring`
- **Short video**: `short-video-cover`, `portrait-pose-variants`
- **Setup**: `anban-setup` (first-time API Key setup and connectivity verification; Codex-specific — does not auto-write `~/.codex/config.toml`, documents manual setup steps instead)
- **Config**: `config` (project-level runtime configuration: writer, theme, image provider, positioning)

### MCP Server (`.mcp.json`)

Connects to the `anbanwriter` MCP server at `${ANBAN_API_URL:-https://api.creator.anbanai.com}/mcp` with Bearer token auth via `${ANBAN_API_KEY}`. Key MCP tools:

- `list_projects`, `get_project_profile`, `list_drafts`, `list_published_articles`, `list_project_titles`
- `prepare_workspace`, `archive_workspace`
- `write_article`, `convert_markdown`, `optimize_seo`
- `generate_image`, `upload_image`, `download_image`, `compress_image`, `analyze_image`
- `publish_draft` (WeChat draft box)
- `get_feed_detail` (SeedNote source note fetching)
- `upload_live_audio`, `create_live_analysis_task`, `query_live_analysis_task`, `recognize_live_invalid_sentences`, `recognize_live_segments`, `build_live_clip_plan`, `build_live_subject_clip_plan`, `build_live_clip_manifest`, `recognize_live_subjects`, `complete_live_subject`

### Themes (Server-managed)

Themes define visual styling for article排版. Themes are managed server-side via the MCP server's `convert_markdown` tool. Each project has a configured theme applied automatically during Markdown-to-WeChat-HTML conversion.

### Writers (`skills/writers/`)

YAML files defining **writing** styles (the writer dimension only). Each has `name`, `english_name`, `writing_prompt` (required), plus optional `core_beliefs`, `title_formulas`, `quote_templates`. Writers **do not** carry visual identity — image visual style is an orthogonal dimension configured per project/task (resolved at runtime as the `visual_style` field; see `article-visual-design` skill). Built-in styles: `dan-koe`, `cultural-depth`, `casual-science`.

### Hooks (`hooks/hooks.json`)

Lifecycle hooks for quality verification. **Plugin-bundled hooks are not trusted by default in Codex** — the user must explicitly review and approve them on install. As a backup, the same delivery-summary prompts are embedded at the end of each subagent's `developer_instructions`, so quality reports still run even if hooks do not fire.

- **SubagentStop**: Agent-specific delivery summaries checking output files, draft status, and completeness (one matcher per subagent name)
- **Stop**: Generic quality verification (file existence, format compliance) — replaces Claude Code's `TaskCompleted` event

## Key Conventions

- **Zero user interaction**: All subagents run autonomously. Decisions are recorded in `$DIR/*.md` files, never by asking the user.
- **Workspace isolation**: Each creation task calls `prepare_workspace` MCP tool to obtain the canonical workspace path, then creates the directory locally with `mkdir -p`. The MCP tool only computes and returns the path — it does not create directories or move files.
- **File naming**: Subagents use numbered prefixes (`01-research.md`, `02-outline.md`...) or semantic names (`cover.png`, `content.md`, `image-plan.md`).
- **Image reference chain**: First image establishes visual style; subsequent images use the first as reference to maintain consistency. For line-art coloring, current `generate_image` is best-effort reference-image generation, not a guaranteed line-preserving colorize tool.
- **Skill references**: Subagents invoke skills via `using the <skill-name> skill` phrasing, not the Skill tool.
- **Content is Chinese**: All generated content targets Chinese social media platforms. Prohibited words lists (违禁词) are in `references/prohibited-words.md`.
- **Live slicing media dependency**: `live-slicer` and `live-slice` require local `ffmpeg` and `ffprobe`; they support continuous clips and subject-script multi-part concat clips without a local helper runtime.
- **Subagent invocation**: Codex subagents do NOT auto-spawn. To run a full pipeline, the user must explicitly invoke: "use the wechatarticle subagent to write an article about X" or "delegate to designer: colorize line art at /path".

## Modifying This Plugin

- **Adding a new skill**: Create `skills/<name>/SKILL.md` with YAML frontmatter `name` + `description`. Add `references/` for detailed guides. Codex auto-discovers skills via the plugin's `skills` manifest field.
- **Adding a new subagent**: Create `agents/<name>.toml` with required fields (`name`, `description`, `developer_instructions`) and optional `[mcp_servers.*]` / `[[skills.config]]` sections. Update `install/agents-registration.toml` to add the `[agents.<name>]` block. Re-run `install/install-subagents.sh`.
- **Adding a new theme**: Themes are managed server-side. Contact the server admin to add new themes.
- **Adding a new writer style**: Add `skills/writers/<name>.yaml` with required `name`, `english_name`, `writing_prompt`.

## Codex vs Claude Code Differences

| Aspect | Claude Code (`claudecode/`) | Codex (`codex/`) |
|--------|------------------------------|------------------|
| Plugin manifest | `.claude-plugin/plugin.json` | `.codex-plugin/plugin.json` (camelCase fields) |
| Subagent format | `agents/*.md` with YAML frontmatter | `~/.codex/agents/*.toml` (TOML) + registration in `~/.codex/config.toml` |
| Subagent auto-spawn | Not applicable (parent calls subagent) | Never — must be explicit (`use the X subagent`) |
| Skills inheritance | Skills inherit from parent session | Skills MUST be declared per-subagent via `[[skills.config]]` |
| Lifecycle events | `SubagentStop`, `TaskCompleted` | `SubagentStop`, `Stop` (no `TaskCompleted`) |
| Hooks trust | Auto-trusted from plugin | Plugin hooks require explicit user approval |
| MCP server list | `mcpServers` in frontmatter | `[mcp_servers.X]` table in TOML |
| Tools allowlist | `tools:` frontmatter field | `sandbox_mode` field (read-only / workspace-write / danger-full-access) |
| Model override | `model: inherit` | Omit `model` field to inherit parent session |
| Max turns | `maxTurns: 300` | No direct equivalent — subagents run to completion or until the user cancels (optionally bounded by Codex's global `job_max_runtime_seconds`, which this plugin does not set) |

## Installation

See [README.md](./README.md) for end-user installation instructions.
