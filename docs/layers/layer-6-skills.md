# Layer 6: Skills — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Layer 6?

If the AI provider (Layer 5) is the robot's brain, skills are its **textbooks**. They teach the robot how to do specific things — like controlling your lights, checking the weather, managing your GitHub PRs, or sending an iMessage.

Here's the key insight: **skills are NOT code**. They're documentation files (`SKILL.md`) written in Markdown. The robot reads them, understands the instructions, and then uses the tools available to it (shell commands, APIs, etc.) to carry them out.

Think of it like giving a smart person a recipe card. The person (the AI) already knows how to cook (use tools). The recipe card (the skill) just tells them what to make and how to make it. Take away the recipe card, and they can still cook — they just don't know that particular dish.

This makes skills incredibly easy to create. You don't need to be a programmer. You just write instructions in plain English (with some YAML metadata), and the robot follows them.

Layer 6 has **six major parts**:

1. **Skill Discovery** — Finding all available skill files
2. **Frontmatter & Metadata** — The structured header of each SKILL.md
3. **Eligibility & Filtering** — Deciding which skills are usable right now
4. **Prompt Injection** — How skills get into the AI's brain
5. **Configuration & Environment** — Per-skill settings, API keys, env vars
6. **The 53 Bundled Skills** — Everything that comes pre-installed

---

## Part 1: Skill Discovery (Finding the Textbooks)

### Where Does OpenClaw Look for Skills?

**File: `src/agents/skills/workspace.ts` (~800 lines)**

OpenClaw searches **7 locations** for skill directories, in precedence order (highest wins):

| Priority | Location | Example Path | Who Uses It |
|----------|----------|-------------|-------------|
| 1 (highest) | Workspace skills | `<your-project>/skills/` | Per-project skills |
| 2 | Project agent skills | `<your-project>/.agents/skills/` | Per-project, shared across agents |
| 3 | Personal agent skills | `~/.agents/skills/` | Your personal skills, all projects |
| 4 | Managed skills | `~/.openclaw/skills/` | Curated by you, all agents |
| 5 | Bundled skills | (shipped with OpenClaw) | The 53 built-in skills |
| 6 | Extra directories | `skills.load.extraDirs` in config | Explicitly configured paths |
| 7 (lowest) | Plugin skills | From extension manifests | Plugins that ship skills |

**Precedence matters for duplicates.** If you have a `github` skill in both your workspace (`./skills/github/`) and the bundled skills, yours wins. This lets you override or customize any built-in skill.

### How Discovery Works

```
1. For each location (in precedence order):
       │
2.     List all child directories
       │
3.     For each directory:
       │
       ├── Does it contain a SKILL.md file?
       │     │
       │     Yes → Parse frontmatter, create SkillEntry
       │     No  → Skip (not a skill)
       │
4.     Add to Map<skillName, SkillEntry>
       │     (later entries DON'T overwrite earlier ones — precedence!)
       │
5. Result: Deduplicated list of skills
```

### Safety Limits

The discovery system has guardrails to prevent pathological scans:

```typescript
DEFAULT_MAX_CANDIDATES_PER_ROOT = 300;    // Max subdirectories per location
DEFAULT_MAX_SKILLS_LOADED_PER_SOURCE = 200; // Max skills per source
DEFAULT_MAX_SKILL_FILE_BYTES = 256_000;    // Max SKILL.md file size (256KB)
```

If a directory has more than 300 subdirectories, the scanner stops and logs a warning. This prevents someone from accidentally pointing the skill loader at their home directory.

### Path Security

All skill paths are resolved via `realpath()` to prevent **symlink escape attacks**. If a skill contains a symlink that points outside its root directory, the system catches it and refuses to load that skill. This was added after a security audit in February 2026.

```typescript
function resolveContainedSkillPath(root: string, skillDir: string): string | null {
  const resolved = fs.realpathSync(skillDir);
  if (!resolved.startsWith(root)) {
    // Symlink escape detected! This skill is trying to read files
    // outside its allowed area. Reject it.
    return null;
  }
  return resolved;
}
```

---

## Part 2: Frontmatter & Metadata (The Skill's ID Card)

### What's in a SKILL.md File?

Every SKILL.md has two parts: a **YAML frontmatter header** (the structured data) and the **body** (the actual instructions).

**File: `src/agents/skills/frontmatter.ts` (~200 lines)**

```markdown
---
name: weather
description: "Get weather forecasts via wttr.in"
homepage: https://wttr.in
metadata:
  {
    "openclaw": {
      "emoji": "🌤️",
      "requires": { "bins": ["curl"] },
      "always": false,
      "os": ["darwin", "linux"]
    }
  }
user-invocable: true
disable-model-invocation: false
---

# Weather Skill

## Use when
- User asks about weather, temperature, or forecasts
- User wants to know if they need an umbrella

## Don't use when
- User asks about climate change (that's general knowledge)

## Instructions
Use curl to query wttr.in:
```bash
curl -s "wttr.in/CityName?format=3"
```
...
```

### Frontmatter Fields Explained

| Field | Type | What It Does |
|-------|------|-------------|
| `name` | string | The skill's unique name (e.g., "weather") |
| `description` | string | One-line summary shown to the AI |
| `homepage` | string | Link to documentation |
| `user-invocable` | boolean | Can users trigger this with `/weather`? |
| `disable-model-invocation` | boolean | Hide from AI? (for user-only skills) |
| `command-dispatch` | string | How commands are dispatched ("tool" or default) |
| `command-tool` | string | Which tool to invoke for slash commands |

### OpenClaw Metadata (The Important Part)

The `metadata.openclaw` object is where the real configuration lives:

```typescript
type OpenClawSkillMetadata = {
  // Display
  emoji?: string;           // "🌤️" — shown in the macOS app
  homepage?: string;        // Link to docs

  // Requirements
  requires?: {
    bins?: string[];        // Required CLI tools (ALL must exist)
    anyBins?: string[];     // At least ONE must exist
    env?: string[];         // Required environment variables
    config?: string[];      // Required config keys
  };

  // Installation
  install?: SkillInstallSpec[];  // How to install dependencies

  // Behavior
  always?: boolean;         // Always include, skip eligibility checks
  primaryEnv?: string;      // Main env var (for API key config)
  skillKey?: string;        // Custom config key (for hyphens in names)
  os?: string[];            // Platform restriction: ["darwin", "linux", "win32"]
};
```

### Skill Install Specifications

Skills can declare how to install their dependencies:

```typescript
type SkillInstallSpec = {
  kind: "brew" | "node" | "go" | "uv" | "download";
  label?: string;           // "Install via Homebrew"
  bins?: string[];           // Which binaries this installs
  os?: string[];             // Platform-specific installer
  formula?: string;          // Brew: formula name
  package?: string;          // npm/uv: package name
  module?: string;           // Go: module path
  url?: string;              // Download: URL
  archive?: string;          // Download: archive type
  extract?: boolean;         // Download: auto-extract?
};
```

**Example:** The `goplaces` skill needs the `goplaces` CLI:
```yaml
install:
  - kind: brew
    formula: goplaces
    bins: ["goplaces"]
    os: ["darwin"]
  - kind: go
    module: github.com/user/goplaces@latest
    bins: ["goplaces"]
```

This means: on macOS, install via `brew install goplaces`. On any platform, install via `go install github.com/user/goplaces@latest`.

---

## Part 3: Eligibility & Filtering (Can This Skill Run?)

### The Problem

You might have 53 skills installed, but many won't work on your system. If you're on Linux, macOS-only skills like Apple Notes are useless. If you don't have `ffmpeg` installed, video skills can't work. If you don't have an OpenAI API key, the Whisper API skill is dead.

### The Eligibility Pipeline

**File: `src/agents/skills/config.ts` (~100 lines)**

```typescript
function shouldIncludeSkill(params: {
  entry: SkillEntry;
  config?: OpenClawConfig;
  eligibility?: SkillEligibilityContext;
}): boolean
```

The checks run in order:

```
1. Config disabled?
   skills.entries.weather.enabled === false → EXCLUDE
       │ (not disabled)

2. Bundled allowlist?
   skills.allowBundled: ["github", "notion"]
   Is "weather" in the list? No → EXCLUDE
       │ (no allowlist, or skill is in list)

3. OS check?
   metadata.openclaw.os: ["darwin"]
   Are we on macOS? No → EXCLUDE
       │ (OS matches, or no OS restriction)

4. Binary check?
   requires.bins: ["curl"]
   Is `curl` on PATH? No → EXCLUDE
       │ (all required binaries found)

5. Any-bins check?
   requires.anyBins: ["spogo", "spotify_player"]
   Is at least ONE on PATH? No → EXCLUDE
       │ (at least one found)

6. Env check?
   requires.env: ["OPENAI_API_KEY"]
   Is it set? (check process.env AND config-provided) No → EXCLUDE
       │ (env var available)

7. Config check?
   requires.config: ["browser.enabled"]
   Is it truthy in config? No → EXCLUDE
       │ (config satisfied)

8. All checks passed → INCLUDE ✓
```

### The "Always" Escape Hatch

Skills with `always: true` **skip ALL eligibility checks**. They're always included in the prompt, no matter what. This is used for meta-skills like `healthcheck` and `canvas` that should always be available.

### Remote Eligibility

Here's a clever feature: if your OpenClaw gateway runs on a Linux server but you have a macOS companion app connected, macOS-only skills become eligible! The system considers the capabilities of connected remote nodes, not just the local machine.

```typescript
type SkillEligibilityContext = {
  remote?: {
    hasBin: (bin: string) => boolean;  // Check if binary exists on remote
    platforms: string[];                // Remote platforms available
  };
};
```

So if you have `memo` (Apple Notes CLI) on your Mac but not on your Linux server, the Apple Notes skill is still usable because the gateway can delegate execution to your Mac.

---

## Part 4: Prompt Injection (Getting Skills Into the AI's Brain)

### How Skills Become Part of the AI's Instructions

**File: `src/agents/system-prompt.ts` (~350 lines)**

When the AI starts processing a message, it receives a system prompt — its core instructions. Skills are injected into a mandatory section:

```
## Skills (mandatory)

Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location>,
  then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.

<available_skills>
- weather (Get weather forecasts via wttr.in) | ~/skills/weather/SKILL.md
- github (GitHub operations via gh CLI) | ~/skills/github/SKILL.md
- spotify-player (Terminal Spotify playback) | ~/skills/spotify-player/SKILL.md
... (up to 150 skills)
</available_skills>
```

### How the AI Uses Skills

Notice the AI doesn't receive the full SKILL.md content upfront. It only sees the **name and description**. When it decides a skill applies, it **reads the file** using its file-reading tool. This is important because:

1. **Token efficiency** — 53 full SKILL.md files would be thousands of tokens. Just listing names/descriptions is ~500 tokens.
2. **On-demand loading** — The AI only loads what it needs.
3. **Rate limiting** — The AI is instructed to be selective, not read every skill.

### Path Compaction

Skill file paths are compacted to save tokens:

```typescript
// Before: /Users/alice/.openclaw/skills/weather/SKILL.md
// After:  ~/.openclaw/skills/weather/SKILL.md
```

This saves ~5-6 tokens per skill × 53 skills = ~300 tokens. Not huge, but it adds up.

### Prompt Limits

**File: `src/agents/skills/workspace.ts` (lines 529-565)**

```typescript
maxSkillsInPrompt: 150        // Maximum number of skills listed
maxSkillsPromptChars: 30_000  // Maximum characters for the skills section
```

If you have more than 150 skills, or the skill list exceeds 30,000 characters, the system **truncates** using a binary search algorithm to find the largest prefix that fits. A warning is appended:

```
⚠️ Skills truncated: included 150 of 200. Run `openclaw skills check` to audit.
```

### Skills Excluded From the Prompt

Two types of skills are never shown to the AI:
- Skills with `disable-model-invocation: true` — These are user-only slash commands. The AI can't trigger them.
- Skills that failed eligibility checks — Missing dependencies mean the skill can't execute, so there's no point showing it to the AI.

### Snapshot Caching

Skills aren't re-discovered on every message. A **snapshot** is created at session start:

```typescript
type SkillSnapshot = {
  prompt: string;              // The formatted skills section
  skills: Array<{
    name: string;
    primaryEnv?: string;
    requiredEnv?: string[];
  }>;
  resolvedSkills?: Skill[];    // Full skill objects
  version?: number;            // For cache invalidation
};
```

The snapshot is reused across all turns in the same session. It's refreshed if:
- The skills watcher detects file changes (if `skills.load.watch` is enabled)
- A new remote node connects (adding new binary/platform capabilities)

---

## Part 5: Configuration & Environment (Per-Skill Settings)

### Skill Config in config.json5

**File: `src/config/types.skills.ts` (~48 lines)**

```json5
{
  skills: {
    // Global settings
    allowBundled: ["github", "weather", "notion"],  // Restrict bundled skills

    load: {
      extraDirs: ["~/my-custom-skills"],  // Additional skill directories
      watch: true,                         // Watch for file changes
    },

    install: {
      preferBrew: true,           // Prefer Homebrew when multiple options
      nodeManager: "npm",         // npm, pnpm, yarn, or bun
    },

    limits: {
      maxSkillsInPrompt: 150,
      maxSkillsPromptChars: 30000,
    },

    // Per-skill configuration
    entries: {
      notion: {
        enabled: true,
        apiKey: "env:NOTION_API_KEY",
      },
      trello: {
        enabled: true,
        apiKey: "sk-trello-abc123",
        env: {
          TRELLO_TOKEN: "my-token",
        },
      },
      "apple-notes": {
        enabled: false,  // I don't want this skill
      },
    },
  },
}
```

### Environment Variable Injection

**File: `src/agents/skills/env-overrides.ts` (~300 lines)**

Skills often need API keys or tokens. Instead of requiring you to set global environment variables, OpenClaw can inject them per-skill:

```
1. Read skills.entries.<skillKey>.env and apiKey from config
       │
2. For each env var:
       │
   ├── Is it already set externally? → Skip (don't override user's env)
   │
   ├── Is it a "dangerous" env var? → Block it!
   │   (OPENSSL_CONF, LDFLAGS, LD_PRELOAD, etc.)
   │
   └── Safe to inject → Set process.env[key] = value
       │
3. Track all injected keys
       │
4. After the AI run completes:
       │
5. Release: restore original env values
```

The **dangerous env var blocklist** prevents skills from injecting variables that could compromise system security (like `LD_PRELOAD` which could load malicious shared libraries).

### The primaryEnv Shortcut

Many skills need just one API key. The `primaryEnv` metadata field links a skill to its config `apiKey`:

```yaml
# In SKILL.md metadata:
primaryEnv: NOTION_API_KEY
```

```json5
// In config.json5:
skills: {
  entries: {
    notion: {
      apiKey: "env:NOTION_API_KEY"  // This gets injected as NOTION_API_KEY
    }
  }
}
```

The `apiKey` config value is automatically injected as the `primaryEnv` environment variable. This means you can set API keys in OpenClaw's config without touching your shell profile.

---

## Part 6: The 53 Bundled Skills (Everything Pre-Installed)

### Skill Categories at a Glance

| Category | Count | Skills |
|----------|-------|--------|
| **Developer Tools** | 9 | github, gh-issues, coding-agent, mcporter, tmux, 1password, himalaya, clawhub, node-connect |
| **Media & Imaging** | 8 | camsnap, gifgrep, video-frames, openai-image-gen, nano-banana-pro, nano-pdf, songsee, peekaboo |
| **Productivity** | 7 | apple-notes, bear-notes, notion, obsidian, things-mac, trello, summarize |
| **Utilities** | 6 | weather, goplaces, ordercli, blogwatcher, healthcheck, xurl |
| **Messaging** | 5 | imsg, bluebubbles, discord, slack, wacli |
| **Music & Audio** | 5 | spotify-player, sag, sherpa-onnx-tts, openai-whisper, openai-whisper-api |
| **Smart Home** | 4 | openhue, eightctl, blucli, sonoscli |
| **Workspace/System** | 3 | canvas, skill-creator, model-usage |
| **Content Generation** | 3 | gemini, oracle, gog |
| **Voice** | 1 | voice-call |
| **Session Management** | 2 | session-logs, coding-agent |

### Detailed Skill Guide

#### Developer Tools

**github** (163 lines) — The workhorse developer skill. Teaches the AI to use the `gh` CLI for:
- Creating and reviewing pull requests
- Managing issues
- Checking CI/CD run status
- Querying the GitHub API
- Code searching

Explicitly tells the AI when NOT to use it (local git operations, code review of local files).

**gh-issues** (865 lines — the LARGEST skill!) — An ambitious automated GitHub issue fixer. The AI:
1. Reads a GitHub issue
2. Spawns a background coding agent to fix it
3. Creates a PR with the fix
4. Monitors PR reviews and iterates
5. All in parallel sub-agents

This is essentially an autonomous software engineer in a SKILL.md file.

**coding-agent** (295 lines) — Teaches the AI to delegate coding tasks to other AI coding tools (Codex, Claude Code, Pi agents) running as background processes. Includes patterns for parallel task execution.

**tmux** (190 lines) — Remote-controls terminal multiplexer sessions. The AI can send keystrokes to tmux panes, scrape their output, and interact with interactive CLIs that don't have APIs.

**mcporter** — MCP (Model Context Protocol) server management. Lists, calls, and configures MCP tools.

**1password** — 1Password CLI (`op`) integration for secret management. Critically, it instructs the AI to always use a fresh tmux session for 1Password to avoid credential leakage.

**himalaya** (257 lines) — Full email management via IMAP/SMTP CLI. Supports multiple accounts, MIME composition, attachment handling.

**clawhub** — ClawHub CLI for searching, installing, and publishing skills from the community marketplace.

**node-connect** — Diagnoses connection issues between the OpenClaw gateway and companion apps (iOS, Android, macOS).

#### Media & Imaging

**openai-image-gen** — Generates images via OpenAI's Images API. Includes batch generation with random prompt sampling and HTML gallery output.

**nano-banana-pro** — Image generation/editing via Google's Gemini 3 Pro Image model using `uv` (Python package runner).

**camsnap** — Captures frames or clips from RTSP/ONVIF security cameras using `ffmpeg`.

**gifgrep** — GIF search across Tenor and Giphy with TUI browsing, download, and frame extraction.

**video-frames** — Extracts frames or short clips from video files using `ffmpeg`.

**peekaboo** (190 lines) — macOS UI automation. Can capture screenshots, target UI elements, and simulate input. Essentially gives the AI "eyes and hands" on your Mac.

**nano-pdf** — Edit PDFs with natural-language instructions.

**songsee** — Generate audio spectrograms and feature visualizations.

#### Productivity

**apple-notes** — Manage Apple Notes via the `memo` CLI. Create, view, edit, delete, search, move, and export notes. macOS-only.

**apple-reminders** — Manage Apple Reminders via `remindctl`. List, add, edit, complete, and delete reminders. macOS-only.

**things-mac** — Things 3 todo app integration via the `things` CLI. macOS-only.

**notion** (174 lines) — Full Notion API integration for pages, databases, and blocks. Requires `NOTION_API_KEY`.

**obsidian** — Work with Obsidian vaults (which are just Markdown files) via `obsidian-cli`.

**trello** — Trello REST API for boards, lists, and cards. Requires `TRELLO_API_KEY` and `TRELLO_TOKEN`.

**summarize** — Summarize URLs, local files, and YouTube videos (including transcription).

#### Smart Home

**openhue** — Control Philips Hue lights and scenes. On/off, brightness, color, color temperature, scenes.

**eightctl** — Control Eight Sleep smart mattress pods. Temperature, alarms, schedules.

**sonoscli** — Control Sonos speakers. Discover, play, volume, grouping, queue management.

**blucli** — BluOS CLI for Bluesound/NAD players. Discovery, playback, grouping.

#### Music & Audio

**spotify-player** — Spotify playback and search via `spogo` (preferred) or `spotify_player`.

**sag** — ElevenLabs text-to-speech with macOS `say`-style UX. Requires `ELEVENLABS_API_KEY`.

**sherpa-onnx-tts** — Local on-device text-to-speech via sherpa-onnx. No cloud needed, works offline.

**openai-whisper** — Local speech-to-text using the Whisper CLI. No API key needed — models download to `~/.cache/whisper`.

**openai-whisper-api** — Cloud-based speech-to-text via OpenAI's Audio Transcriptions API. Requires `OPENAI_API_KEY`.

#### Messaging

**discord** (197 lines) — Discord operations via the message tool: reactions, pins, send/edit/delete, threads, search, presence.

**slack** — Slack operations: react, manage pins, send/edit/delete messages, fetch member info.

**imsg** — iMessage/SMS CLI for listing chats, reading history, and sending messages. macOS-only.

**bluebubbles** — iMessage via BlueBubbles REST API. Richer than native iMessage skill (reactions, effects, edit, unsend).

**wacli** — WhatsApp CLI for sending messages and searching/syncing history.

#### Utilities

**weather** — Get weather via `wttr.in` or Open-Meteo. No API key needed. The simplest possible skill.

**goplaces** — Google Places API for location search, details, reviews. Requires `GOOGLE_PLACES_API_KEY`.

**xurl** (461 lines) — Full X (Twitter) API access: post, search, engage, DMs, social graph management.

**blogwatcher** — Monitor blogs and RSS/Atom feeds for updates.

**healthcheck** (245 lines) — Security auditing for OpenClaw deployments. Checks host hardening, risk tolerance, configuration.

**ordercli** — Check food delivery orders (Foodora supported, Deliveroo WIP).

#### Meta/System Skills

**skill-creator** (372 lines) — The meta-skill. Teaches the AI how to create, edit, improve, and audit other skills. Includes Python scripts for scaffolding and packaging.

**canvas** (198 lines) — Display HTML content on connected OpenClaw nodes (Mac, iOS, Android apps). The bridge between AI output and visual UI.

**model-usage** — Get per-model usage costs from local cost logs.

**session-logs** — Search and analyze conversation session logs using `jq` and `ripgrep`.

#### Content & Tools

**gemini** — One-shot Q&A via the Gemini CLI.

**oracle** — Best practices for the `oracle` CLI (prompt bundling, engines, sessions).

**gog** — Google Workspace CLI for Gmail, Calendar, Drive, Contacts, Sheets, and Docs.

**voice-call** — Start voice calls via the OpenClaw voice-call plugin (Twilio, Telnyx, Plivo, or mock).

### Skills by Dependency Type

**No external dependencies (API/gateway-based):**
- bluebubbles, canvas, coding-agent, gh-issues, healthcheck, node-connect, notion, voice-call

**Needs API keys:**
- notion (NOTION_API_KEY), trello (TRELLO_API_KEY + TRELLO_TOKEN), sag (ELEVENLABS_API_KEY), openai-image-gen (OPENAI_API_KEY), openai-whisper-api (OPENAI_API_KEY), goplaces (GOOGLE_PLACES_API_KEY), nano-banana-pro (GEMINI_API_KEY), gh-issues (GH_TOKEN)

**macOS-only:**
- apple-notes, apple-reminders, bear-notes, imsg, things-mac, peekaboo, model-usage

### Skill Size Distribution

**Large skills (200+ lines):** gh-issues (865!), xurl (461), skill-creator (372), coding-agent (295), himalaya (257), healthcheck (245)

**Medium skills (100-199 lines):** canvas (198), discord (197), peekaboo (190), notion (174), github (163), tmux (190+)

**Small skills (<100 lines):** weather, camsnap, voice-call, video-frames, nano-pdf, openai-whisper, gemini — each under 50 lines

**Total SKILL.md content across all 53 skills: ~6,900 lines.**

---

## Part 7: Skill Security (Protecting Against Bad Skills)

### The Security Scanner

**File: `src/security/skill-scanner.ts`**

When a skill is installed from an external source, the system scans it for dangerous patterns:
- Hardcoded credentials
- Suspicious imports
- Unsafe shell commands
- Prompt injection patterns in SKILL.md

Critical findings block installation. Warnings are shown but don't block.

### Capability-Based Security

Added in February 2026 (commit `2c61fb69c`, +1,568 lines), this is a full capability model:

Skills can declare what capabilities they need:
- **shell** — Can run shell commands
- **filesystem** — Can read/write files
- **network** — Can make network requests
- **browser** — Can control a browser
- **sessions** — Can access conversation history

The system enforces these at runtime: if a skill hasn't declared `network` capability, the AI's tool calls involving network access are blocked.

### Zip Slip Prevention

After a security audit found a CVSS 7.7 vulnerability in the skill-creator's packaging script, protections were added:
- Archive extraction validates all paths stay within the target directory
- Symlinks are checked for escape attempts
- Skill download target paths are restricted

---

## Part 8: Skill Slash Commands (User Invocation)

### How /commands Work

**File: `src/auto-reply/skill-commands.ts` (~200 lines)**

Skills with `user-invocable: true` are exposed as slash commands. If the `weather` skill is user-invocable, typing `/weather London` triggers it directly.

Command names are sanitized:
- Max 32 characters
- Lowercase
- Special characters replaced with underscores
- Collisions handled with numeric suffixes (`github_2`, `github_3`)
- Discord descriptions limited to 100 characters
- Reserved names (built-in commands) are excluded

### The Installation System

**File: `src/agents/skills-install.ts` (~400 lines)**

Skills can be installed through the gateway:

```
1. User selects skill + installer option
       │
2. Build install command based on kind:
   ├── brew: brew install <formula>
   ├── node: npm install -g <package> --ignore-scripts
   ├── go: go install <module>@latest
   ├── uv: uv tool install <package>
   └── download: curl <url> + extract
       │
3. Run with timeout
       │
4. Security scan the installed files
       │
5. Return success/failure
```

The `--ignore-scripts` flag for npm installs is a security measure — it prevents arbitrary code execution during package installation.

### Gateway Methods

The gateway exposes four skill-related RPC methods:

| Method | What It Does |
|--------|-------------|
| `skills.status` | Get health report for all skills |
| `skills.bins` | List all required binaries across agents |
| `skills.install` | Install a skill's dependencies |
| `skills.update` | Update a skill's config (apiKey, env, enabled) |

---

## How Layer 6 Evolved (Git History)

### The Big Bang — December 20, 2025

The entire skill system was created in a single day.

**`d1850aaad`** — `feat: add managed skills gating`

This massive commit (+1,233 lines) introduced everything at once:
- `src/agents/skills.ts` (390 lines) — the core skill loader
- Config integration
- Agent integration
- Documentation
- **~25 initial skills** including: blucli, camsnap, eightctl, gog, imsg, mcporter, nano-banana-pro, nano-pdf, openai-image-gen, openai-whisper, openai-whisper-api, openhue, oracle, peekaboo, sag, sonoscli, spotify-player, video-frames, wacli, and more

On the same day, `c4a67b7d0` added the macOS Skills settings UI and standardized metadata across all skills.

The initial skills were heavily focused on smart home, media, and macOS — reflecting the creator's personal use cases.

### The Holiday Sprint — December 20-31, 2025

Skills were added during the holiday season:
- **Dec 29**: obsidian
- **Dec 30**: food-order
- **Dec 31**: gifgrep

### The January Explosion — January 1-7, 2026

**January 2 was the biggest single day** — at least 8 new skills:
- things-mac, trello, songsee, weather, goplaces, coding-agent, apple-notes, apple-reminders, discord

By January 7, the skill count had roughly doubled from the original ~25.

Key additions:
- **Jan 1**: Config unification (`fbcbc60e8` — `feat: unify skills config`)
- **Jan 1**: OS-specific gating (`73d0e2cb8` — `fix: gate skills by OS`)
- **Jan 3**: bear-notes, tmux, github, blogwatcher, notion, clawhub, slack
- **Jan 5**: model-usage
- **Jan 6**: 1password, himalaya
- **Jan 7**: session-logs

### The Skill-Creator & Plugin System — January 11, 2026

Two landmark additions on the same day:

**`7006a4aad`** — `feat: add skill-creator bundled skill` (+1,163 lines)

The self-referential meta-skill. It teaches the AI how to create new skills, complete with Python scripts for scaffolding (`init_skill.py`), packaging (`package_skill.py`), and validation (`quick_validate.py`). This is the second-largest skill at 372 lines.

**`cf0c72a55`** — `feat: add plugin architecture` (+2,408 lines)

The plugin system (Layer 3) was born on the same day, and with it came the ability for plugins to ship their own skills.

### The Second Wave — January 18 to February 19, 2026

| Date | Skill | Notable |
|------|-------|---------|
| Jan 18 | bluebubbles, canvas | Canvas for visual UI |
| Jan 21 | sherpa-onnx-tts | First local TTS skill |
| Jan 26 | bitwarden | Later removed |
| Feb 2 | healthcheck | Security auditing |
| Feb 16 | gh-issues | The 865-line autonomous issue fixer |
| Feb 19 | xurl | X/Twitter API (461 lines) |

### The Routing Blocks — February 17, 2026

**`9cce40d12`** — `feat(skills): Add 'Use when / Don't use when' routing blocks`

This was a game-changer for skill quality. Instead of the AI guessing which skill to use, skills now explicitly declare:

```markdown
## Use when
- User asks about weather, temperature, or forecasts
- User wants to know what to wear

## Don't use when
- User asks about climate change (general knowledge)
- User asks about historical weather data
```

This was inspired by OpenAI's best practices for function calling. It reduced skill mis-selection significantly.

### The Security Hardening — February-March 2026

A sustained security effort:

**Feb 17** — `2c61fb69c` — **Client-side skill security enforcement** (+1,568 lines)
- Capability-based security model
- Static SKILL.md scanner for prompt injection
- Before-tool-call enforcement gate
- Trust tier classification (builtin/community/local)

**Feb 19** — `c275932aa` — **Zip Slip prevention** — Fixed CVSS 7.7 vulnerability in skill packaging

**Feb 27** — `abc08a9f7` — Parse skill capabilities from frontmatter

**Mar 2** — `132794fe7` — Audit workspace skill symlink escapes

**Mar 6** — `bf623a580` — Skill API rate-limit guardrails

### The Naming Cascade — January 27-30, 2026

The project was renamed twice, requiring updates across all 53 skill files:
- **Jan 27**: clawdbot → moltbot (53 skill files updated)
- **Jan 30**: moltbot → openclaw (53 skill files updated again)

### Skill Removals

Not every skill survived:
- **food-order** — Removed Feb 22
- **bird** — Removed (date unclear)
- **local-places** — Duplicate of goplaces, removed
- **QR code skill** — Added and reverted
- **video-quote-finder** — Added and reverted
- **bitwarden** — Added Jan 26, later removed

### Cross-Agent Discovery — February 12, 2026

**`d85150357`** — `feat: support .agents/skills/ directory for cross-agent skill discovery`

Added two new skill loading locations:
- `~/.agents/skills/` — Personal skills shared across all agents
- `<workspace>/.agents/skills/` — Project-level skills shared across agents

This established the full 7-location precedence hierarchy.

### Latest Addition — March 14, 2026

**node-connect** — The most recent skill, helping users diagnose connection issues between the gateway and companion apps.

---

## Architecture Summary

```
┌───────────────────────────────────────────────────────┐
│                     Config                             │
│  skills.entries.github.enabled: true                   │
│  skills.entries.notion.apiKey: "env:NOTION_API_KEY"    │
│  skills.allowBundled: [...] (optional filter)          │
└──────────────────────┬────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         │      Skill Discovery       │
         │                            │
         │  7 locations, precedence:  │
         │  workspace > project >     │
         │  personal > managed >      │
         │  bundled > extra > plugin  │
         │                            │
         │  Dedup by name             │
         │  (highest precedence wins) │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │    Frontmatter Parsing     │
         │                            │
         │  YAML header → metadata    │
         │  requires: { bins, env }   │
         │  install: [brew, go, ...]  │
         │  os: ["darwin", "linux"]   │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │   Eligibility Filtering    │
         │                            │
         │  Config disabled? → skip   │
         │  Bundled allowlist? → skip │
         │  OS mismatch? → skip       │
         │  Missing binary? → skip    │
         │  Missing env var? → skip   │
         │  always: true? → include!  │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │    Prompt Injection        │
         │                            │
         │  Format eligible skills    │
         │  Apply limits (150 / 30K)  │
         │  Compact paths (~ prefix)  │
         │  Cache as SkillSnapshot    │
         │                            │
         │  Inject into system prompt │
         │  as <available_skills>     │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │       AI Runtime           │
         │                            │
         │  AI reads skill names      │
         │  Matches to user request   │
         │  Reads SKILL.md on demand  │
         │  Follows instructions      │
         │  Uses tools to execute     │
         └────────────────────────────┘

Env Injection (parallel):
  Config apiKey/env → sanitize → inject process.env → run → release

Security (parallel):
  Capability enforcement → tool-call gating → symlink audit → rate limiting
```

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/agents/skills/workspace.ts` | ~800 | Core discovery, loading, prompt building, limits |
| `src/agents/skills/frontmatter.ts` | ~200 | SKILL.md YAML parsing |
| `src/agents/skills/config.ts` | ~100 | Eligibility evaluation pipeline |
| `src/agents/skills/types.ts` | ~90 | Core type definitions (SkillEntry, SkillSnapshot, etc.) |
| `src/agents/skills/env-overrides.ts` | ~300 | Environment variable injection & safety |
| `src/agents/skills/plugin-skills.ts` | ~90 | Plugin-shipped skill discovery |
| `src/agents/skills-status.ts` | ~250 | Health reporting (status, missing deps) |
| `src/agents/skills-install.ts` | ~400 | Installation pipeline (brew/node/go/uv/download) |
| `src/agents/system-prompt.ts` | ~350 | System prompt building with skills section |
| `src/auto-reply/skill-commands.ts` | ~200 | Slash command registration from skills |
| `src/config/types.skills.ts` | ~48 | Config schema types |
| `src/gateway/server-methods/skills.ts` | ~250 | Gateway RPC methods (status/install/update) |
| `src/security/skill-scanner.ts` | ~200 | Security scanning for installed skills |
| `src/cli/skills-cli.ts` | ~80 | CLI commands (list, info, check) |
| `skills/` | 53 dirs | All bundled SKILL.md files (~6,900 total lines) |
