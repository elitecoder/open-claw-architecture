# Layer 1: Core Runtime — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Layer 1?

Imagine OpenClaw is a robot butler that lives on your computer. Layer 1 is the robot's **brain stem** — the part that handles waking up, remembering its settings, knowing who's talking to it, and keeping notes about past conversations.

Without Layer 1, the robot can't even turn on. Everything else — talking on Discord, using AI, remembering things — sits on top of this foundation.

Layer 1 has **four major parts**:

1. **Boot Sequence** — How the robot wakes up
2. **Config System** — How the robot reads its instruction manual
3. **Session Management** — How the robot keeps a notebook of conversations
4. **Routing** — How the robot figures out which "personality" should answer a message

Let's go through each one.

---

## Part 1: The Boot Sequence (How the Robot Wakes Up)

### The Three Files That Start Everything

When you type `openclaw` in your terminal, three files run in sequence, like dominoes falling:

```
src/entry.ts  →  src/cli/run-main.ts  →  src/index.ts
   (Wake up)       (Get ready)           (Do the work)
```

### File 1: `src/entry.ts` — The Alarm Clock

This is the **very first code** that runs. Think of it as the alarm clock that wakes up the robot. It does a lot of careful checking before letting anything else happen.

#### Step 1: Am I really supposed to be running? (Lines 36-41)

```typescript
if (!isMainModule({ ... })) {
  // Nope, I was imported as a helper. Do nothing.
} else {
  // Yes! I'm the main show. Boot up!
}
```

**Why?** JavaScript bundling tools sometimes accidentally import this file twice. This guard says "only run the boot sequence if I'm the ACTUAL entry point, not a copy."

Think of it like checking your alarm — if your mom already woke you up, you don't need the alarm to go off again.

#### Step 2: Set my name tag (Lines 44-54)

```typescript
process.title = "openclaw"
ensureOpenClawExecMarkerOnProcess()
installProcessWarningFilter()
normalizeEnv()
enableCompileCache()  // best-effort, won't crash if it fails
```

- **`process.title = "openclaw"`**: When you open Activity Monitor (Mac) or Task Manager (Windows), you'll see "openclaw" instead of "node". It's like writing your name on your lunchbox.
- **`ensureOpenClawExecMarkerOnProcess()`**: Stamps `OPENCLAW_EXEC` onto the environment so any child processes know they're part of the OpenClaw family.
- **`installProcessWarningFilter()`**: Tells Node.js "stop showing me annoying warnings about experimental features." Like putting noise-canceling headphones on.
- **`normalizeEnv()`**: Cleans up environment variables — makes sure they're all formatted consistently.
- **`enableCompileCache()`**: Node 22+ can cache compiled code so next time you start, it's faster. Like pre-heating your oven.

#### Step 3: The Respawn Trick (Lines 80-126)

This is the cleverest part. Sometimes Node.js shows warnings like "ExperimentalWarning: this feature is experimental." OpenClaw uses experimental features, so these warnings would spam your terminal.

The fix? **The process restarts itself** with a special flag that silences the warnings:

```
First run:  openclaw send "hello"
  → "Hmm, warnings aren't suppressed."
  → Spawns: node --disable-warning=ExperimentalWarning openclaw send "hello"
  → Sets OPENCLAW_NODE_OPTIONS_READY=1 (so the child doesn't respawn AGAIN)
  → Parent waits for child, then exits with child's exit code.

Second run (the child):
  → "OPENCLAW_NODE_OPTIONS_READY is set? Great, skip respawn."
  → Continues with actual work.
```

**Why not just add the flag to the original command?** Because users run `openclaw` directly — you can't control what flags they pass. This self-healing respawn ensures the environment is always correct, no matter how the user invoked it.

The `OPENCLAW_NODE_OPTIONS_READY=1` guard prevents an infinite loop (robot keeps waking itself up forever).

#### Step 4: Fast Paths for Simple Questions (Lines 128-164)

If you just typed `openclaw --version` or `openclaw --help`, the robot doesn't need to load ALL its plugins, config files, and channels. That would be like hiring a moving company to hand someone a business card.

Instead:
- **`--version`**: Immediately imports just the version number, prints it, exits. Done in milliseconds.
- **`--help`**: Imports just the help text generator, prints it, exits.

These "fast paths" skip 95% of the startup work.

#### Step 5: Profile Handling (Lines 166-179)

You can run `openclaw --profile dev` to load a different set of environment variables (like having a "school outfit" vs "weekend outfit"). This step:
1. Parses the `--profile` flag from your command
2. Loads the profile's environment overrides
3. Strips the `--profile` flag from argv so later code doesn't get confused

#### Step 6: Hand Off to run-main.ts (Lines 183-192)

```typescript
const { runCli } = await import("./cli/run-main.js")
await runCli(process.argv)
```

Dynamic `import()` is used (not static `import`) because we only want to load `run-main.ts` if we actually need it (didn't take a fast path). This keeps startup fast.

### File 2: `src/cli/run-main.ts` — Getting Dressed

Now the robot is awake. Time to get dressed and ready for the day.

#### Step 1: Load Environment (Lines 85-89)

```typescript
loadDotEnv({ quiet: true })   // Read .env file if it exists
normalizeEnv()                  // Clean up env vars again
ensureOpenClawCliOnPath()       // Make sure 'openclaw' command is available in PATH
```

- **`.env` file**: A file where you store secrets like API keys. Loading it puts those values into the environment.
- **Ensure CLI on PATH**: If you installed OpenClaw but it's not in your system PATH, this adds it. Like making sure your pencils are in your pencil case before school.

Some commands skip the PATH step — if you're just running `openclaw status` (read-only), there's no need.

#### Step 2: Check Your Age (Line 92)

```typescript
assertSupportedRuntime()  // Node >= 22 required
```

OpenClaw requires Node.js version 22 or higher. If you're running Node 18, this throws an error immediately rather than crashing mysteriously later. Like a roller coaster height check — "you must be THIS tall to ride."

#### Step 3: Try the Express Lane (Lines 94-100)

```typescript
const routed = await tryRouteCli(normalizedArgv)
if (routed) return  // Handled! Skip everything below.
```

Certain commands (like `gateway`, `config`, `status`) have special "fast routes" that bypass the full program initialization. If your command matches one, it runs immediately and returns — no need to load the entire Commander program, plugins, etc.

#### Step 4: Build the Commander Program (Lines 102-103)

```typescript
const { buildProgram } = await import("./program.js")
const program = buildProgram()
```

[Commander](https://github.com/tj/commander.js) is a library that defines CLI commands. `buildProgram()` creates the main program with all its root options (`--verbose`, `--yes`, `--profile`, etc.).

#### Step 5: Install Safety Nets (Lines 105-112)

```typescript
installUnhandledRejectionHandler()
process.on("uncaughtException", (err) => { ... })
```

These are crash catchers. If any code throws an error that nobody catches, these handlers:
1. Log the error nicely (with stack trace)
2. Exit with code 1 (failure)

Without them, Node.js would just crash silently or show an ugly error.

#### Step 6: Register Commands (Lines 114-145)

This is where the actual commands get wired up:

1. **Rewrite `--update` to `update`**: Users can type `openclaw --update` or `openclaw update` — both work.
2. **Register the primary command**: If you typed `openclaw send`, this loads and registers the `send` command definition.
3. **Register plugin commands**: Plugins can add their own CLI commands (like `openclaw discord setup`). These are loaded from the config and registered dynamically.

The system is smart about skipping plugin loading when unnecessary — if you typed `--help`, why load Discord's plugin just to show help text?

#### Step 7: Go! (Line 147)

```typescript
await program.parseAsync(parseArgv)
```

Commander parses your command and calls the right handler. This is where the actual work happens.

#### Step 8: Clean Up (Lines 148-150)

```typescript
finally {
  await closeCliMemoryManagers()
}
```

Even if something crashes, always clean up memory search managers. Like making sure you turn off the stove even if dinner burned.

### File 3: `src/index.ts` — The Swiss Army Knife

This file serves double duty:
1. **As an alternative entry point**: Can be run directly as `node dist/index.js`
2. **As a library**: Other code can `import` utilities from it

When run directly, it does the same setup as `entry.ts` + `run-main.ts` but in a single file. When imported, it just exports helpful functions.

### Supporting Files

#### `src/globals.ts` — The Shared Whiteboard

A tiny module (about 50 lines) that holds global state accessible everywhere:

- **`verbose` flag**: Are we in verbose/debug mode?
- **`yes` flag**: Auto-confirm all prompts?
- **Theme colors**: `success` (green), `warn` (yellow), `info` (blue), `danger` (red)

Why globals? Because `--verbose` is parsed at the top level but used deep inside config loading, routing, etc. Passing it through every function would be tedious.

#### `src/runtime.ts` — The Stunt Double

Defines a `RuntimeEnv` type with three methods: `log`, `error`, `exit`. The real runtime logs to the console and calls `process.exit()`. But in tests, you can swap in a "stunt double" that throws an error instead of exiting (so your test doesn't die).

```typescript
// Real runtime (production):
exit: (code) => { restoreTerminalState(); process.exit(code); }

// Test runtime:
exit: (code) => { throw new Error(`exit ${code}`); }
```

---

## Part 2: The Config System (The Robot's Instruction Manual)

### Where Does Config Live?

Your config file is at `~/.openclaw/config.json5`. JSON5 is like JSON but friendlier — it allows comments, trailing commas, and unquoted keys. Think of it as JSON's cool older sibling.

### The 14-Step Config Loading Pipeline

When OpenClaw reads your config, it goes through **14 steps**. Here's each one explained simply:

#### Step 1: Load .env File
If you have a `.env` file with secrets (like `OPENAI_API_KEY=sk-...`), load those into the environment first. The config file might reference them.

#### Step 2: Check if Config File Exists
If there's no `config.json5`, return an empty config. The system will use defaults for everything.

#### Step 3: Read and Parse the File
Read the raw text from disk and parse it as JSON5.

#### Step 4: Process `$include` Directives
This is like a "paste from another file" feature:

```json5
{
  "$include": "./agents.json5",      // Single file
  "$include": ["./a.json5", "./b.json5"]  // Multiple files, merged together
}
```

**Security protections:**
- **Max depth of 10**: Can't include a file that includes a file that includes a file... more than 10 levels deep.
- **Circular detection**: If A includes B and B includes A, it catches the loop.
- **Path containment**: Included files must be inside the config directory. You can't `$include: "/etc/passwd"`.
- **Symlink protection**: Even if you create a symlink pointing outside, it's caught.
- **Max file size**: Each included file capped at 2MB.

When multiple files are included, they're **deep-merged**: arrays get concatenated, objects get recursively merged (later values win on conflicts).

#### Step 5: Substitute Environment Variables
Any `${VARIABLE_NAME}` in the config gets replaced with the actual environment variable value:

```json5
{
  models: {
    providers: {
      openai: { apiKey: "${OPENAI_API_KEY}" }  // Becomes "sk-ant-..."
    }
  }
}
```

If a variable is missing, it warns you (but doesn't crash).

#### Step 6: Check for Duplicate Agent Directories
If two agents point to the same directory, that's a conflict. Caught early before anything else happens.

#### Step 7: Validate with Zod Schema
The config is validated against a strict schema using the Zod library. Every field must be the right type, within allowed values, etc. If validation fails, you get a clear error message listing exactly what's wrong.

#### Step 8: Version Compatibility Check
If the config was last edited by a NEWER version of OpenClaw (e.g., you downgraded), it warns you. Like opening a Word document from a newer version of Word — "this might not work perfectly."

#### Step 9: Apply Cascading Defaults (The Layer Cake)
This is where the magic happens. Defaults are applied in a specific order, like building a layer cake:

```
applyMessageDefaults        ← Bottom layer: message/chat settings
  applyLoggingDefaults      ← Log levels, formats
    applySessionDefaults    ← Session TTL, pruning rules
      applyAgentDefaults    ← Agent concurrency, model choices
        applyContextPruningDefaults  ← Context window limits
          applyCompactionDefaults    ← Session compaction thresholds
            applyModelDefaults       ← Model aliases, costs
              applyTalkConfigNormalization  ← Top layer: voice settings
```

Each layer can reference values set by the layer below it. These defaults are ONLY applied at load time — when writing config back to disk, they're stripped out so your file stays clean.

#### Step 10: Normalize Paths
Resolve `~` to your home directory, make relative paths absolute, etc.

#### Step 11: Post-defaults Duplicate Check
Check for duplicate agent dirs again (defaults might have created new conflicts).

#### Step 12: Apply Config-Defined Environment Variables
Your config can set environment variables:
```json5
{
  env: {
    vars: {
      MY_CUSTOM_VAR: "value"
    }
  }
}
```
These get applied to `process.env`, but ONLY if they're not already set (your real environment always wins) and they're not dangerous system vars like `PATH` or `HOME`.

#### Step 13: Shell Environment Fallback
If API keys are still missing after all the above, OpenClaw can optionally spawn a shell and capture its environment. This helps when API keys are set in your `.zshrc` but not in the current process environment.

#### Step 14: Auto-Generate Owner Display Secret
If the config is missing an owner display secret (used for identifying the owner), one is generated automatically and queued for async write-back. Non-blocking — startup isn't delayed.

### Config Caching

Reading and parsing a config file 14 steps deep every time would be slow. So there's a cache:

- **Default TTL: 200ms** (not 45 seconds — that's sessions)
- Configurable via `OPENCLAW_CONFIG_CACHE_MS` environment variable
- Cleared whenever config is written or runtime snapshots change

### Config Writing (The Reverse Journey)

Writing config is even more careful than reading:

1. **Merge-patch**: Only writes values YOU changed, not runtime defaults
2. **Environment variable preservation**: If your config had `"${OPENAI_API_KEY}"` and the resolved value hasn't changed, the `${...}` reference is preserved in the file (so your secrets stay as references, not plaintext)
3. **Atomic writes**: Writes to a temp file first, then renames it. If the rename fails mid-way, your original config is still intact.
4. **Backup rotation**: Maintains a `.bak` copy of your previous config
5. **Audit logging**: Every config write is logged to `~/.openclaw/logs/config-audit.jsonl` with SHA256 hashes, timestamps, and suspicious activity detection (like config shrinking to less than half its previous size)

### Dependency Injection

The entire config system accepts injectable dependencies:
```typescript
type ConfigIoDeps = {
  fs?: typeof fs          // File system (can be mocked in tests)
  json5?: typeof JSON5    // Parser (can be mocked)
  env?: NodeJS.ProcessEnv // Environment (can be faked)
  homedir?: () => string  // Home directory (can be overridden)
  logger?: ...            // Logger (can be captured)
}
```

This makes the config system fully testable without touching the real filesystem.

---

## Part 3: Session Management (The Robot's Notebook)

### What Is a Session?

A session is a conversation. When you chat with OpenClaw on Telegram, that's a session. When you chat on Discord in a different channel, that's another session. Each session has:
- A unique key (like `agent:main:telegram:direct:john`)
- Metadata (last message time, channel, thread info)
- A transcript file (`.jsonl`) with the actual messages

### Where Do Sessions Live?

```
~/.openclaw/
  agents/
    main/                          ← Default agent
      sessions/
        sessions.json              ← The index (list of all sessions)
        main.jsonl                 ← Transcript for the main session
        agent-main-telegram-direct-john.jsonl  ← John's Telegram session
    support/                       ← A custom "support" agent
      sessions/
        sessions.json
        ...
```

### The Session Store (`src/config/sessions/store.ts`)

The `sessions.json` file is the **index** — it maps session keys to metadata entries. Think of it as the table of contents in a notebook.

#### Reading Sessions: The Cache and Retry Dance

Reading `sessions.json` has a sophisticated caching and retry system:

**Cache Layer (45-second TTL):**
- Before reading from disk, check if we have a cached version
- Cache is validated by comparing the file's modification time (mtime) and size
- If the file hasn't changed, use the cache
- 45-second TTL means even if the file hasn't changed, we re-read every 45 seconds (just in case)

**Retry Logic (Windows vs Unix):**

Windows has a problem: file writes aren't truly atomic. When one process writes `sessions.json`, another process reading at the exact same moment might see:
- An empty file (0 bytes — between truncate and write)
- A half-written file (corrupt JSON)

The fix:
- **Windows: Retry 3 times** with 50ms pauses between attempts (using `Atomics.wait()` for precise timing)
- **Unix: Try once** (file renames ARE atomic on Unix, so this problem doesn't exist)

```
Windows retry flow:
  Attempt 1: Read file → Empty! → Wait 50ms
  Attempt 2: Read file → JSON parse error! → Wait 50ms
  Attempt 3: Read file → Success! Parse JSON, cache it.
```

#### Writing Sessions: Locks and Atomic Safety

Multiple parts of OpenClaw might try to update sessions at the same time (incoming Discord message + Telegram message simultaneously). To prevent corruption:

**In-Memory Lock Queue:**
- Each `sessions.json` file has a lock queue
- Tasks queue up and execute one at a time
- Lock timeout: 10 seconds (if a lock is held too long, it's considered stale at 30 seconds)

**Atomic Write Pattern:**
```
1. Write to temp file: sessions.json.12345.abc123.tmp
2. Rename temp file to sessions.json (atomic on Unix)
3. On Windows: If rename fails (EPERM/EEXIST), fall back to copy + chmod
```

**Write Retries:**
- Windows: 5 attempts with exponential backoff (50ms, 100ms, 150ms, 200ms, 250ms)
- Unix: 1 attempt, with a best-effort retry if ENOENT (directory was deleted)

#### Session Maintenance (Automatic Cleanup)

Every time sessions are saved, automatic maintenance runs:
- **Prune stale entries**: Sessions older than `pruneAfterMs` get removed
- **Cap total entries**: If you have more than `maxEntries` sessions, oldest ones are evicted
- **Archive transcripts**: Removed session transcripts are moved to an archive
- **Disk budget enforcement**: Total session storage is capped

#### Legacy Key Normalization

Older versions of OpenClaw stored session keys with mixed casing (e.g., `Agent:Main:Discord:Direct:John`). New versions normalize everything to lowercase. When loading, the system:
1. Looks for the normalized (lowercase) key
2. Also searches for any case-variant legacy keys
3. If multiple variants exist, picks the most recently updated one
4. Migrates to the normalized key going forward

### Session Paths (`src/config/sessions/paths.ts`)

This file handles all the path math — figuring out WHERE session files should live.

**Key Functions:**

- **`resolveAgentSessionsDir(agentId)`**: Returns `~/.openclaw/agents/{agentId}/sessions/`
- **`resolveSessionTranscriptPathInDir(sessionId, dir, topicId?)`**: Builds the transcript filename
  - Without topic: `{sessionId}.jsonl`
  - With topic: `{sessionId}-topic-{encodedTopicId}.jsonl`
- **`resolvePathWithinSessionsDir(path, sessionsDir)`**: Security function that ensures a path stays INSIDE the sessions directory. No `../../etc/passwd` tricks.

**Cross-Agent Session Access:**
Sometimes one agent needs to look up another agent's sessions (e.g., for handoff). The path system supports this by detecting the `agents/{agentId}/sessions/` structure and rewriting the agent ID portion.

---

## Part 4: Routing (The Robot's Switchboard)

### What Is Routing?

Imagine you have three AI personalities set up:
- **"main"**: Your general assistant
- **"work"**: Handles work-related questions
- **"support"**: Handles customer support

When a message arrives on Discord, routing decides which personality should answer. It's like a phone switchboard — "this call goes to the sales department, that call goes to tech support."

### The 7-Tier Routing Hierarchy (`src/routing/resolve-route.ts`)

Routing uses a **priority ladder**. It checks the most specific match first and falls through to more general matches. First match wins.

#### Tier 1: Direct Peer Binding (Most Specific)

"Is there a rule specifically for THIS person?"

```json5
{
  routing: {
    bindings: [{
      match: { peer: { kind: "direct", id: "jane" } },
      agentId: "support"
    }]
  }
}
```

If Jane sends a DM, she always gets the "support" agent. Nobody else is affected.

#### Tier 2: Parent Peer Binding (Thread Inheritance)

"Is this a reply in a thread? What agent handles the thread's parent?"

If someone replies in a Discord thread, the message inherits the agent from the thread's parent channel/message. This prevents agent-switching mid-conversation.

#### Tier 3: Guild + Roles Binding

"Is this person in a specific Discord server WITH a specific role?"

```json5
{
  match: { guild: "acme-guild", roles: ["admin"] },
  agentId: "admin-agent"
}
```

Only people with the "admin" role in the "acme-guild" server get the admin agent.

#### Tier 4: Guild Binding (No Role Requirement)

"Is this person in a specific Discord server?"

Same as Tier 3, but without checking roles. Everyone in the server gets the same agent.

#### Tier 5: Team Binding

"Is this person in a specific team?"

For platforms like Microsoft Teams or Slack workspaces.

#### Tier 6: Account Binding

"Which of my accounts received this message?"

If you have multiple Discord accounts (personal + work), you can route each account to a different agent.

#### Tier 7: Channel Binding (Least Specific)

"What agent handles ALL messages from this channel?"

```json5
{
  match: { channel: "discord", account: "*" },
  agentId: "main"
}
```

The wildcard `*` means "any account on Discord."

#### Tier 8: Default Agent (Fallback)

If nothing matches, use the default agent from config. This is always `"main"` unless you changed it.

### The Two-Tier LRU Cache

Route resolution involves walking through all bindings, checking each one against the message context. This is expensive if done for every single message. So there are TWO caches:

1. **Binding Cache (2,000 entries)**: Caches the evaluated bindings for a given channel+account combination. Key: `"discord\taccount1"`.

2. **Route Cache (4,000 entries)**: Caches the final resolved route for a given full context (channel + account + peer + guild + team + roles). Key: a multi-segment string.

Both caches are LRU (Least Recently Used) — when full, the oldest unused entry gets evicted.

**Cache bypass**: When verbose logging is enabled, caches are skipped so you can see the full resolution debug output.

### Session Key Generation (`src/routing/session-key.ts`)

Once routing determines which agent handles a message, it needs a **session key** — a unique string that identifies the conversation.

**Format:** `agent:{agentId}:{scope}`

**DM Scope Variants:**

For direct messages, you can control how sessions are grouped:

| Scope | Pattern | Meaning |
|-------|---------|---------|
| `main` | `agent:main:main` | ALL DMs collapse into one session |
| `per-peer` | `agent:main:direct:jane` | Each person gets their own session |
| `per-channel-peer` | `agent:main:telegram:direct:jane` | Jane on Telegram ≠ Jane on Discord |
| `per-account-channel-peer` | `agent:main:telegram:acct1:direct:jane` | Most granular — per account + channel + peer |

For group/channel messages, it's always: `agent:{agentId}:{channel}:{peerKind}:{peerId}`

**Identity Links:**

If the same person talks to you on Telegram AND Discord, they normally get two separate sessions. But with identity links, you can merge them:

```json5
{
  identityLinks: {
    "jane@example.com": ["telegram:123456", "discord:jane#1234"]
  }
}
```

Now messages from both platforms resolve to the same canonical identity, and can share a session.

**Agent ID Normalization:**

Agent IDs are cleaned up to be filesystem-safe:
- Lowercased
- Invalid characters replaced with `-`
- Leading/trailing dashes removed
- Max 64 characters
- Empty → defaults to `"main"`

---

## How Layer 1 Evolved: The Git History

### Phase 1: Genesis — "Warelay" (November 24, 2025)

The project started as **warelay** — a simple Twilio WhatsApp relay. A single developer (Peter Steinberger) created everything.

**`src/index.ts`** was the very first file: a **511-line monolith** containing EVERYTHING — Commander CLI setup, Twilio client, Express webhook server, auto-reply logic, config loading (from `~/.warelay/warelay.json`), and the main entry point. All in one file.

**`src/globals.ts`** was extracted the same day — a tiny 29-line module holding `--verbose` and `--yes` flags. This was the first sign of modularization.

**`src/runtime.ts`** appeared the next day (Nov 25) — a minimal 14-line module making auto-reply helpers testable.

### Phase 2: Rapid Growth (Nov-Dec 2025)

The monolith (`index.ts`) grew explosively across ~60 commits:
- WhatsApp Web provider added
- Tailscale integration
- Structured logging via pino
- Discord transport (Dec 15)

The project rebranded from **"warelay"** to **"clawdis"** in December (commit `b3e50cbb3`).

### Phase 3: The Great Modularization (January 4-14, 2026)

This was the **pivotal period** where the monolith was surgically broken apart.

**January 4 — Config system extracted:**
`src/config/io.ts` was born (188 lines), pulling config reading/writing/validation out of `index.ts` into a proper module with dependency injection, JSON5 parsing, and Zod validation.

Same day: project renamed from "clawdis" to **"clawdbot"**.

**January 5 — Boot sequence split:**
`src/entry.ts` (20 lines) and `src/cli/run-main.ts` (48 lines) were created. This was the architectural breakthrough:
- `entry.ts` became the shebang entry point handling profiles and early setup
- `run-main.ts` became the CLI orchestration layer
- `index.ts` was demoted from "everything" to a support module

**January 6 — Routing system created:**
`src/routing/resolve-route.ts` (223 lines) and `src/routing/session-key.ts` (77 lines) were brand new concepts. Before this, there was only ONE agent. Now the system could route different messages to different agents based on peer/guild/team bindings.

**January 11 — `$include` directive:**
~230 lines added to `config/io.ts` implementing modular config includes with circular detection and path security. Contributed by an external developer who needed isolated per-client agent configs for a legal case management use case.

**January 13 — Two key changes:**
1. The experimental warning respawn trick was added to `entry.ts`
2. "Providers" were renamed to "Channels" throughout the routing system — reflecting the evolution from "WhatsApp relay" to "multi-platform message router"

**January 14 — Session system extracted:**
`src/config/sessions/store.ts` (299 lines) and `src/config/sessions/paths.ts` (75 lines) were created, pulling session persistence out of the config module into its own subsystem with caching, retry logic, and per-agent directories.

### Phase 4: The Name Carousel (January 18-30, 2026)

- **Jan 18**: Startup optimization — fast-path routing added to `run-main.ts`
- **Jan 27**: Renamed to **"moltbot"** (all references updated, legacy compat maintained)
- **Jan 30**: Final rename to **"openclaw"** — types became `OpenClawConfig`, env vars became `OPENCLAW_*`

Each rename preserved backward compatibility for config and state directories.

### Phase 5: Polish and Performance (February-March 2026)

**February 8**: `ChatType` unification — `"dm"` became `"direct"` throughout session keys, with backward compatibility (`normalizeChatType('dm')` → `'direct'`).

**February 14**: Startup speed improvements — lazy subcommand registration and smarter plugin loading to avoid expensive imports for `--help`.

**February 16**: `runtime.ts` refactored to extract shared `createRuntimeIo()` factory.

**March 1**: Two performance features added to `entry.ts`:
- Fast-path for `openclaw --version` (skip full CLI load)
- Node compile cache enabled at startup

**March 9**: Latest change — child process environment marking with `OPENCLAW_CLI`.

### Evolution Summary

| Date | Event | Impact |
|------|-------|--------|
| Nov 24, 2025 | Project created as "warelay" | 511-line monolith |
| Dec 2025 | Rename to "clawdis" | Branding change |
| Jan 4, 2026 | Config system extracted | First real module split |
| Jan 4, 2026 | Rename to "clawdbot" | Branding change |
| Jan 5, 2026 | Boot sequence split | `entry.ts` + `run-main.ts` created |
| Jan 6, 2026 | Routing system created | Multi-agent support born |
| Jan 11, 2026 | `$include` directives | Modular config files |
| Jan 13, 2026 | Respawn trick + providers→channels | Key architectural patterns |
| Jan 14, 2026 | Session system extracted | Own subsystem with caching/retry |
| Jan 27, 2026 | Rename to "moltbot" | Branding change |
| Jan 30, 2026 | Final rename to "openclaw" | Final brand identity |
| Feb 8, 2026 | ChatType unification | `dm` → `direct` |
| Mar 1-9, 2026 | Performance optimizations | Compile cache, fast paths |

### Key Takeaways from the History

1. **Monolith → Modules**: The project started as a single file and was methodically decomposed. Each extraction happened when the monolith became painful to work with.

2. **Names tell the story**: warelay → clawdis → clawdbot → moltbot → openclaw. Each rename reflected an expanding scope — from "WhatsApp relay" to "multi-platform AI assistant."

3. **Backward compatibility always preserved**: Every rename maintained legacy config directory support. No user data was ever broken by a rename.

4. **Boot sequence grew from necessity**: The respawn trick, fast paths, and profile support were all added because real users hit real problems. None were "designed upfront."

5. **Config system grew organically**: Started as a 20-line JSON reader, grew to 1,500+ lines with includes, env substitution, atomic writes, auditing, and security protections. Each feature was driven by a specific use case.

6. **Routing was a later addition**: For the first 6 weeks, there was only ONE agent. Multi-agent routing was added January 6, 2026, and has been refined ever since.

---

## File Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/entry.ts` | ~194 | Boot entry: guards, respawn, fast paths, profiles |
| `src/cli/run-main.ts` | ~155 | CLI orchestration: env, runtime check, command registration |
| `src/index.ts` | ~93 | Alternative entry + library exports |
| `src/globals.ts` | ~52 | Global verbose/yes flags + theme colors |
| `src/runtime.ts` | ~53 | Runtime I/O abstraction (log, error, exit) |
| `src/config/io.ts` | ~1559 | Config load/write pipeline (14-step read, atomic write) |
| `src/config/includes.ts` | ~346 | `$include` directive processing with security |
| `src/config/defaults.ts` | ~536 | 8-stage cascading defaults |
| `src/config/validation.ts` | ~611 | Zod schema + domain validation |
| `src/config/env-substitution.ts` | ~150 | `${VAR}` substitution at load time |
| `src/config/env-preserve.ts` | ~135 | `${VAR}` restoration at write time |
| `src/config/sessions/store.ts` | ~883 | Session persistence with cache, retry, locks |
| `src/config/sessions/paths.ts` | ~330 | Session path resolution + security |
| `src/routing/resolve-route.ts` | ~804 | 7-tier binding resolution + 2-tier LRU cache |
| `src/routing/session-key.ts` | ~254 | Session key generation + DM scope + identity links |
