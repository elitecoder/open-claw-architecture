# OpenClaw — Complete Architecture Analysis

## What It Is

OpenClaw is a **self-hosted, local-first personal AI assistant** that connects to messaging platforms (WhatsApp, Telegram, Discord, Slack, Signal, iMessage, etc.) and runs on your own devices. It's a monorepo built with TypeScript (Node.js 22+), Swift (Apple apps), and Kotlin (Android).

---

## Architecture Layers (Bottom to Top)

### Layer 1: Core Runtime (Required)

**Boot sequence** (`src/entry.ts` → `src/cli/run-main.ts` → `src/index.ts`):

1. Set process title, install warning filters, normalize env
2. Enable Node compile cache, respawn with correct flags if needed
3. Fast-path `--version` / `--help`
4. Load `.env`, assert Node 22+, build Commander CLI program
5. Register all commands, parse args, execute

**Config system** (`src/config/io.ts:725-869`):

- Reads `config.json5` from `~/.openclaw/`
- Supports JSON5 with `include` directives
- Applies cascading defaults: messages → logging → sessions → agents → context pruning → compaction → model → talk normalization
- 45-second in-memory cache with hot-reload watching

**Session management** (`src/config/sessions/store.ts:195-270`):

- Persists to `~/.openclaw/agents/{agentId}/sessions/sessions.json`
- Retry logic on read (Windows: 3x, Unix: 1x)
- Automatic migrations and disk budget enforcement

**Routing** (`src/routing/resolve-route.ts:614-804`):

- Determines which agent handles a message via a resolution hierarchy:
  1. Peer binding (direct match)
  2. Parent peer binding (thread inheritance)
  3. Guild+roles binding (Discord)
  4. Guild → Team → Account → Channel binding
  5. Default agent fallback
- Two-tier LRU cache (2000 bindings + 4000 routes)

---

### Layer 2: Gateway (Required)

The **central nervous system** — `src/gateway/` (247 files, ~4.8MB).

**Entry:** `src/gateway/server.impl.ts:267-1070` — `startGatewayServer()`

**10-phase startup:**

1. Load config snapshot + migrate legacy
2. Activate secrets runtime
3. Bootstrap auth (generate tokens, load TLS)
4. Load plugins + channel plugins
5. Create HTTP/HTTPS + WebSocket servers
6. Start channel manager (all configured channels)
7. Start discovery (mDNS, Tailscale, wide-area)
8. Start cron, health monitor, heartbeat
9. Attach WebSocket handlers + plugin hooks
10. Register config watcher for hot-reload

**Protocol** (`src/gateway/protocol/`): Request/Response/Event frames over WebSocket with AJV-validated schemas.

**Auth** (`src/gateway/auth.ts`): Token, password, Tailscale identity, device pairing, trusted proxy, bootstrap tokens. Role-based access (operator/node/guest) with per-method scopes.

**Key server methods** (`src/gateway/server-methods/`, 69 files):

| Category | Methods |
|----------|---------|
| Chat | `chat.send`, `chat.abort`, `chat.history` |
| Config | `config.get`, `config.set`, `config.patch` |
| Channels | `channels.status`, `channels.logout` |
| Nodes | `node.list`, `node.pair`, `node.invoke` |
| Cron | `cron.list`, `cron.add`, `cron.run` |
| Sessions | `sessions.list`, `sessions.patch`, `sessions.reset` |
| System | `health`, `logs.tail`, `usage.status`, `skills.status` |

**HTTP endpoints** (`src/gateway/server-http.ts`):

- `GET /health`, `/ready`, `/healthz` — health probes
- `POST /v1/chat/completions` — OpenAI-compatible API
- `POST /v1/responses` — OpenResponses protocol
- `POST /hooks/*` — inbound webhooks from channels
- `GET /canvas/*` — Canvas AI UI

**Runtime state** (`src/gateway/server-runtime-state.ts:43-150`):

```typescript
{
  canvasHost: CanvasHostHandler | null   // Embedded browser canvas
  httpServer: HttpServer                 // Main HTTP/HTTPS server
  wss: WebSocketServer                   // WebSocket server
  clients: Set<GatewayWsClient>         // Connected clients
  broadcast: GatewayBroadcastFn          // Send to all clients
  agentRunSeq: Map<string, number>       // Agent run IDs for dedup
  dedupe: Map<string, DedupeEntry>       // Request dedup cache
  chatRunState: {...}                    // Chat session tracking
  chatAbortControllers: Map              // Chat cancellation handles
}
```

**Broadcasting** (`src/gateway/server-broadcast.ts`): Central event system pushing to all connected clients. Event types include `agent`, `chat`, `presence`, `health`, `tick`, `node.pair.requested`, `exec.approval.requested`.

**Channel Manager** (`src/gateway/server-channels.ts:111-300`): Lifecycle management for all messaging channels with exponential backoff restarts (5s → 5m max).

---

### Layer 3: Plugin System (Required framework, individual plugins optional)

**Discovery** (`src/plugins/discovery.ts`): Scans `~/.openclaw/extensions/`, bundled `extensions/`, and workspace paths. Uses `jiti` for dynamic module loading. Results cached with 1-second TTL.

**Plugin API** (`src/plugins/types.ts`):

```typescript
api.registerChannel(registration)   // Add messaging channel
api.registerProvider(provider)       // Add AI provider
api.registerTool(tool)               // Add agent tool
api.registerHook(events, handler)    // Add lifecycle hook
api.registerHttpRoute(params)        // Add HTTP endpoint
```

**Plugin Registry** (`src/plugins/registry.ts:130-143`):

```typescript
{
  plugins: PluginRecord[],
  channels: PluginChannelRegistration[],
  providers: ProviderRegistration[],
  tools: PluginToolRegistration[],
  hooks: PluginHookRegistration[],
  gatewayHandlers: GatewayRequestHandlers,
}
```

**Extension manifest** (`package.json`):

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "discord",
      "label": "Discord",
      "docsPath": "/channels/discord",
      "order": 10
    }
  }
}
```

**Plugin SDK** (`src/plugin-sdk/index.ts`, 150+ exports): Channel plugin types, provider auth helpers, context engine interfaces, tool types, gateway request handler types.

---

### Layer 4: Channel Integrations (Need at least 1)

**Interface** (`src/channels/plugins/types.plugin.ts:49-86`):

`ChannelPlugin` with ~20 optional adapters:

- **Required:** `id`, `meta`, `capabilities`, `config`
- **Key adapters:**
  - `outbound` — send messages to the channel
  - `gateway` — start/stop accounts, QR login
  - `status` — health checks, account validation
  - `messaging` — receive, route, thread messages
  - `security` — allowlist enforcement
  - `directory` — list peers/groups
  - `threading` — reply-to mode, context
  - `actions` — polls, buttons, cards
  - `agentTools` — channel-specific agent tools

**Adapter types** (`src/channels/plugins/types.adapters.ts:1-383`):

- `ChannelConfigAdapter` (Line 52-81): Account resolution, allowlist config
- `ChannelOutboundAdapter` (Line 108-125): deliveryMode (direct/gateway/hybrid), chunker, sendPayload/sendText/sendMedia/sendPoll
- `ChannelGatewayAdapter` (Line 275-289): startAccount, stopAccount, loginWithQr
- `ChannelStatusAdapter` (Line 127-166): probeAccount, buildAccountSnapshot
- `ChannelDirectoryAdapter` (Line 335-344): self(), listPeers, listGroups

**Core types** (`src/channels/plugins/types.core.ts:1-403`):

- `ChannelCapabilities` (Line 181-194): chatTypes array, feature flags
- `ChannelAccountSnapshot` (Line 97-159): enabled, configured, linked, connected, lastMessageAt
- `ChannelSecurityDmPolicy` (Line 196-203): DM allowlist policy

**Channel registry** (`src/channels/registry.ts:1-201`):

- Core channel order: telegram, whatsapp, discord, irc, googlechat, slack, signal, imessage, line
- Aliases: "imsg" → "imessage"
- Functions: `listChatChannels()`, `getChatChannelMeta()`, `normalizeChatChannelId()`

**Built-in channels** (in `extensions/`):

| Channel | Location |
|---------|----------|
| Telegram | `extensions/telegram/` |
| WhatsApp | `extensions/whatsapp/` |
| Discord | `extensions/discord/` |
| Slack | `extensions/slack/` |
| Signal | `extensions/signal/` |
| iMessage | `extensions/imessage/` |
| LINE | `extensions/line/` |
| IRC | `extensions/irc/` |
| Matrix | `extensions/matrix/` |
| Mattermost | `extensions/mattermost/` |
| Zalo | `extensions/zalo/` |
| Google Chat | `extensions/googlechat/` |
| Web UI | `src/channels/web/` |

Each channel is independently removable — just don't configure it.

**Example: Discord registration** (`extensions/discord/index.ts:1-19`):

```typescript
const plugin = {
  id: "discord",
  name: "Discord",
  description: "Discord channel plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setDiscordRuntime(api.runtime)
    api.registerChannel({ plugin: discordPlugin })
    registerDiscordSubagentHooks(api)
  }
}
```

---

### Layer 5: AI Providers (Need at least 1)

**Model config** (`src/config/types.models.ts:3-12`):

Supported API protocols:

- `openai-completions`
- `openai-responses`
- `anthropic-messages`
- `google-generative-ai`
- `github-copilot`
- `bedrock-converse-stream` (AWS)
- `ollama` (local)

**Model definition** (`src/config/types.models.ts:34-61`):

```typescript
ModelDefinitionConfig = {
  id: string,
  name: string,
  reasoning: boolean,
  input: Array<"text" | "image">,
  cost: { input, output, cacheRead, cacheWrite },
  contextWindow: number,
  maxTokens: number,
  compat?: ModelCompatConfig,
}

ModelProviderConfig = {
  baseUrl: string,
  apiKey?: SecretInput,
  auth?: "api-key" | "aws-sdk" | "oauth" | "token",
  models: ModelDefinitionConfig[],
  headers?: Record<string, SecretInput>,
}
```

**Provider builders** (`src/agents/models-config.providers.ts`):

Qwen, BytePlus/Doubao, Minimax, Moonshot/Kimi, Qianfan (Baidu), Xiaomi, OpenRouter, Together, Venice, HuggingFace, Vercel AI Gateway, Kilocode, and more.

**Provider plugin example — Qwen OAuth** (`extensions/qwen-portal-auth/index.ts:1-126`):

```typescript
register(api: OpenClawPluginApi) {
  api.registerProvider({
    id: "qwen-portal",
    label: "Qwen",
    docsPath: "/providers/qwen",
    aliases: ["qwen"],
    auth: [{
      id: "device",
      kind: "device_code",
      label: "Qwen OAuth",
      run: async (ctx) => {
        // OAuth device code flow
        return buildOauthProviderAuthResult({
          providerId: "qwen-portal",
          configPatch: {
            models: { providers: { "qwen-portal": { baseUrl, apiKey, models } } }
          }
        })
      }
    }]
  })
}
```

Providers register via the plugin system with auth flows (API key, OAuth, device code).

---

### Layer 6: Skills (All Optional)

Skills are **documentation files** (`SKILL.md`) — NOT executable code. They teach the agent when and how to use CLI commands/APIs. 53 bundled skills in `skills/`:

| Category | Examples |
|----------|---------|
| Productivity | apple-notes, apple-reminders, notion, obsidian, trello, things-mac |
| Developer | coding-agent, github, gh-issues, tmux |
| Media | camsnap, gifgrep, openai-image-gen, openai-whisper, video-frames |
| Smart Home | openhue, sonoscli, spotify-player |
| Communication | discord, slack, bluebubbles |
| Utility | weather, summarize, oracle, goplaces |
| Voice | sherpa-onnx-tts, voice-call |
| Meta | skill-creator, canvas, session-logs, model-usage, healthcheck |

**Example — Weather skill** (`skills/weather/SKILL.md`): Documents how to use `wttr.in` via curl. No API key needed. Teaches the agent formatting and when to use forecasts vs current conditions.

**Example — GitHub skill** (`skills/github/SKILL.md`): Documents `gh` CLI commands for PRs, issues, CI. Emphasizes when NOT to use (local git ops, code review).

Every skill can be removed independently with zero impact on the system.

---

### Layer 7: Memory System (Optional)

`src/memory/` (105 files, 1.7MB) — RAG-based long-term memory.

**Core manager** (`src/memory/manager.ts`, 28KB):

- `MemoryIndexManager`: Main embedding + search interface
- Handles embeddings, indexing, deduplication
- Supports multiple backend implementations

**Embedding providers:**

| Provider | File |
|----------|------|
| OpenAI | `src/memory/embeddings-openai.ts` |
| Google Gemini | `src/memory/embeddings-gemini.ts` (17KB) |
| Voyage AI | `src/memory/embeddings-voyage.ts` (8KB) |
| Ollama (local) | `src/memory/embeddings-ollama.ts` (4KB) |
| Mistral | also supported |

**Search & retrieval:**

- `search-manager.ts` — query execution and retrieval (7.8KB)
- `query-expansion.ts` — query rewriting for better recall (13KB)
- `mmr.ts` — Maximal Marginal Relevance ranking (6KB)
- `qmd-manager.ts` — Query Markdown processing (70KB)
- `temporal-decay.ts` — recent-message boosting (4KB)

**Batch embedding:** Supports batch APIs for OpenAI, Gemini, and Voyage.

**Storage backends:** Pluggable — e.g., `extensions/memory-lancedb/` for LanceDB vector storage.

**Export** (`src/memory/index.ts:1-12`):

```typescript
export { MemoryIndexManager }
export type { MemorySearchManager, MemorySearchResult }
export { getMemorySearchManager }
```

Without memory, the assistant simply forgets between sessions.

---

### Layer 8: Context Engine (Optional but important)

`src/context-engine/types.ts:1-177` — manages the conversation window:

```typescript
interface ContextEngine {
  bootstrap(sessionId, messages?)
  ingest(message)
  ingestBatch(messages)
  assemble(messages, tokenBudget): AssembleResult
  compact(tokenBudget, force?): CompactResult
  afterTurn(messages, prePromptCount, tokenBudget?)
  prepareSubagentSpawn(parentKey, childKey)
  onSubagentEnded(reason)
  dispose()
}
```

**Key features:**

- **Pluggable contract** — swap implementations (default vs custom) via `src/context-engine/registry.ts`
- **Message compaction** — summarize old turns to fit token budget
- **Subagent isolation** — prepare child sessions with their own context
- **Token budgeting** — proactive compaction to stay within limits

Removing this severely degrades conversation quality but doesn't crash the system.

---

### Layer 9: Voice / TTS (Optional)

`src/tts/` (4 files, 176KB) — text-to-speech with multiple backends:

| Backend | Description |
|---------|-------------|
| ElevenLabs | Cloud, voice cloning, stability/similarity controls |
| OpenAI TTS | tts-1, tts-1-hd models |
| Edge TTS | Offline Microsoft voices via `node-edge-tts` |
| Ollama TTS | Local on-device |
| sherpa-onnx | On-device (via skill) |

**Configuration** (`src/tts/tts-core.ts`):

```typescript
type ResolvedTtsConfig = {
  elevenlabs: {
    baseUrl: string, apiKey: string, voiceId: string,
    voiceSettings: { stability, similarityBoost, style, speed }
  }
  openai: { baseUrl, apiKey, model, voice }
}
```

Only needed for voice interactions.

---

### Layer 10: Canvas & A2UI (Optional)

**Canvas host** (`src/canvas-host/server.ts:1-120`) — HTTP server on port 18793:

- Serves interactive HTML/JS/CSS from `~/.openclaw/canvas/` or custom root
- Live reload via chokidar file watching + WebSocket injection
- URL prefix: `/__openclaw__/canvas/`

**Architecture:**

```
Canvas Host (HTTP:18793)
  |  (TCP:18790 Bridge)
Node Bridge
  |  (WebSocket)
Connected Nodes (Mac/iOS/Android apps)
```

**A2UI** (`vendor/a2ui/`, `src/canvas-host/a2ui.ts`):

- Declarative JSON-based UI rendering (UIKit-like abstraction)
- Can render HTML/CSS/JS or structured UI components
- Bridge injects actions: `openclawSendUserAction()` (general), `openclawCanvasA2UIAction` (iOS/Android)

**Handler interface:**

```typescript
type CanvasHostHandler = {
  handleHttpRequest(req, res): Promise<boolean>
  handleUpgrade(req, socket, head): boolean
  close(): Promise<void>
}
```

---

### Layer 11: Companion Apps (All Optional)

| App | Location | Tech | Description |
|-----|----------|------|-------------|
| macOS menu bar | `apps/macos/` (230 files, 7 modules) | SwiftUI | Menu bar app with IPC and protocol layers |
| iOS | `apps/ios/` (32 dirs) | SwiftUI | Chat, camera, location, Live Activities, widgets |
| Android | `apps/android/` | Kotlin/Gradle | Android NDK for native bridging |
| Shared library | `apps/shared/OpenClawKit/` | Swift package | Shared types/protocols for Apple platforms |

**macOS modules:** OpenClaw (main), OpenClawMacCLI, OpenClawIPC, OpenClawProtocol

**iOS capabilities:** Chat, camera, location, contacts, motion sensors, Live Activities, widgets, media handling

**Key design:** These are **thin clients** — they:

1. Connect to the OpenClaw gateway via WebSocket
2. Receive canvas URLs and bridge notifications
3. Render UI and relay user actions back to gateway
4. **Do NOT** run the agent themselves — the gateway is the backend

The core runs headless without any companion apps.

---

### Layer 12: Infrastructure (All Optional/Swappable)

**Docker** (`docker-compose.yml`, 79 lines):

```yaml
services:
  openclaw-gateway:
    # Main service on port 18789
    command: node dist/index.js gateway --bind lan --port 18789
    healthcheck: GET /healthz
    volumes:
      - ~/.openclaw/   # config
      - workspace dir  # workspace

  openclaw-cli:
    # CLI companion, same network as gateway
    network_mode: "service:openclaw-gateway"
    # Dropped: NET_RAW, NET_ADMIN capabilities
```

**Environment variables:** `OPENCLAW_GATEWAY_TOKEN`, `OPENCLAW_CONFIG_DIR`, `OPENCLAW_WORKSPACE_DIR`

**Other deployment options:**

- `fly.toml` — Fly.io deployment
- `render.yaml` — Render deployment
- `src/daemon/` — systemd/launchd daemonization
- Tailscale integration for secure remote access

---

## Data Flow

```
User sends message on Discord/Telegram/etc.
         |
         v
Channel Plugin (extensions/<channel>/)
  -> receives via platform SDK, normalizes message
         |
         v
Gateway (src/gateway/)
  -> authenticates, resolves route via binding hierarchy
         |
    +---------+-------------------+
    |                             |
    v                             v
Memory Search              Context Engine
(RAG retrieval)            (assemble prompt
                            within token budget)
    +----------+-----------------+
               |
               v
AI Provider (OpenAI/Anthropic/Google/Ollama/etc.)
  -> LLM inference with assembled context
               |
               v
Tool Execution (skills inform, plugin tools execute)
               |
               v
Response routed back: Gateway -> Channel -> User
               |
          (optionally)
               v
Canvas Host renders interactive UI
Companion Apps receive push via bridge
```

---

## What's Optional — Quick Reference

| Layer | Removable? | Impact of Removal |
|-------|-----------|-------------------|
| Core Runtime | **No** | System won't start |
| Gateway | **No** | Nothing works without the control plane |
| Plugin Framework | **No** | Channels/providers can't load |
| Any single channel | **Yes** | Lose that platform only |
| Any single provider | **Yes** | Lose that LLM only (need >= 1) |
| Any skill | **Yes** | Agent loses that capability |
| Memory | **Yes** | No long-term recall between sessions |
| Context Engine | **Yes** | Degraded conversation quality |
| Voice/TTS | **Yes** | Text-only mode |
| Canvas/A2UI | **Yes** | No visual workspace |
| All companion apps | **Yes** | Headless operation only |
| Docker/Fly/Render | **Yes** | Run directly on bare metal instead |
| Cron (`src/cron/`) | **Yes** | No scheduled tasks |
| i18n (`src/i18n/`) | **Yes** | English only |
| Browser control | **Yes** | No browser automation |
| Link understanding | **Yes** | No URL preview/extraction |
| Media understanding | **Yes** | No image/video analysis |

**Minimal viable installation:** Core Runtime + Gateway + 1 Channel + 1 AI Provider.

---

## Key File Reference

| Component | File | Key Lines |
|-----------|------|-----------|
| Entry point | `src/entry.ts` | 36-194 |
| CLI runtime | `src/index.ts` | 36-93 |
| CLI orchestration | `src/cli/run-main.ts` | 74-160+ |
| Config loading | `src/config/io.ts` | 725-869, 1467-1492 |
| Session store | `src/config/sessions/store.ts` | 195-270 |
| Session paths | `src/config/sessions/paths.ts` | 1-330 |
| Message routing | `src/routing/resolve-route.ts` | 614-804 |
| Session keys | `src/routing/session-key.ts` | 1-254 |
| Gateway startup | `src/gateway/server.impl.ts` | 267-1070 |
| Gateway dispatcher | `src/gateway/server-methods.ts` | 100-158 |
| Protocol schemas | `src/gateway/protocol/index.ts` | 1-200 |
| WebSocket handler | `src/gateway/server/ws-connection.ts` | 93-200 |
| Chat handling | `src/gateway/server-methods/chat.ts` | 1-1400+ |
| Channel lifecycle | `src/gateway/server-channels.ts` | 111-300 |
| Gateway client | `src/gateway/client.ts` | 122-150+ |
| Auth system | `src/gateway/auth.ts` | 1-250 |
| HTTP server | `src/gateway/server-http.ts` | 1-1500+ |
| Method scopes | `src/gateway/method-scopes.ts` | role/scope matrix |
| Channel plugin interface | `src/channels/plugins/types.plugin.ts` | 49-86 |
| Channel adapters | `src/channels/plugins/types.adapters.ts` | 1-383 |
| Channel core types | `src/channels/plugins/types.core.ts` | 1-403 |
| Channel registry | `src/channels/registry.ts` | 1-201 |
| Plugin discovery | `src/plugins/discovery.ts` | 1-100 |
| Plugin loader | `src/plugins/loader.ts` | 1-120 |
| Plugin registry | `src/plugins/registry.ts` | 130-143 |
| Plugin SDK | `src/plugin-sdk/index.ts` | 1-150 |
| Plugin SDK core | `src/plugin-sdk/core.ts` | 1-75 |
| Model types | `src/config/types.models.ts` | 1-77 |
| Model builders | `src/agents/models-config.providers.ts` | 1-150+ |
| Tool interface | `src/agents/tools/common.ts` | 1-100 |
| Memory manager | `src/memory/manager.ts` | (28KB) |
| Context engine types | `src/context-engine/types.ts` | 1-177 |
| TTS core | `src/tts/tts-core.ts` | 1-100 |
| Canvas host | `src/canvas-host/server.ts` | 1-120 |
| Globals | `src/globals.ts` | 1-52 |
| Runtime env | `src/runtime.ts` | 37-44 |
| Docker compose | `docker-compose.yml` | 1-79 |
