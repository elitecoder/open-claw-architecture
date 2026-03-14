# Layer 4: Channel Integrations — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Layer 4?

If Layer 1 is the brain, Layer 2 is the nervous system, and Layer 3 is the organ transplant department, then Layer 4 is **the robot's mouth, ears, and hands** — the parts that actually talk to people on different messaging apps.

Imagine your robot can understand and speak any language, but it needs translators to talk to people on WhatsApp, Discord, Telegram, and so on. Each "channel" is one of those translators. The Telegram translator knows how to receive Telegram messages, convert them into something the robot understands, and then take the robot's response and send it back as a proper Telegram message.

Without at least one channel, the robot is like a brilliant person locked in a soundproof room — it can think, but it can't communicate with anyone.

Layer 4 has **five major parts**:

1. **The ChannelPlugin Contract** — The rules every translator must follow
2. **Adapters** — Specialized sub-abilities each translator can have
3. **The Channel Registry** — The master list of known translators
4. **Channel Lifecycle Management** — Starting, stopping, and restarting translators
5. **The Built-in Channels** — All 22 translators that come pre-installed

---

## Part 1: The ChannelPlugin Contract (The Rules)

### What Every Channel Must Be

Every channel plugin is a TypeScript object that follows a specific shape. Think of it like a job application form — there are required fields and optional fields.

**File: `src/channels/plugins/types.plugin.ts`**

```typescript
type ChannelPlugin<ResolvedAccount = any, Probe = unknown, Audit = unknown> = {
  // === REQUIRED (every channel MUST have these) ===
  id: ChannelId;                    // e.g., "telegram", "discord"
  meta: ChannelMeta;                // Human-readable name, docs link, etc.
  capabilities: ChannelCapabilities; // What can this channel do?
  config: ChannelConfigAdapter<ResolvedAccount>; // How to read/write config

  // === OPTIONAL (extra powers) ===
  defaults?: { queue?: { debounceMs?: number } };
  reload?: { configPrefixes: string[]; noopPrefixes?: string[] };
  onboarding?: ChannelOnboardingAdapter;
  configSchema?: ChannelConfigSchema;
  setup?: ChannelSetupAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  groups?: ChannelGroupAdapter;
  mentions?: ChannelMentionAdapter;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  gatewayMethods?: string[];
  auth?: ChannelAuthAdapter;
  elevated?: ChannelElevatedAdapter;
  commands?: ChannelCommandAdapter;
  streaming?: ChannelStreamingAdapter;
  threading?: ChannelThreadingAdapter;
  messaging?: ChannelMessagingAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  actions?: ChannelMessageActionAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  agentTools?: ChannelAgentToolFactory | ChannelAgentTool[];
};
```

That's about **20 optional adapters**. No channel uses all of them. A simple channel like IRC might use 5-6, while a complex one like Feishu uses 15+.

### The Three Generic Type Parameters

Notice `<ResolvedAccount, Probe, Audit>` at the top. These are like "fill-in-the-blank" types:

- **ResolvedAccount**: What does a "configured account" look like for this channel? For Telegram, it's an object with a `botToken`. For WhatsApp, it's an object with a `sessionId`. Each channel defines its own shape.
- **Probe**: What does a "health check" return? For Telegram, it might be the bot's username and rate limit status.
- **Audit**: What does a "deep audit" return? For Telegram, it's a list of groups the bot is in and its permissions.

This means the TypeScript compiler can check that when Telegram says "here's my account config," it actually matches Telegram's account shape — not Discord's.

### Channel Metadata (`ChannelMeta`)

Every channel describes itself with metadata:

```typescript
type ChannelMeta = {
  id: ChannelId;              // "telegram"
  label: string;              // "Telegram"
  selectionLabel: string;     // What shows in the setup wizard
  docsPath: string;           // "/channels/telegram"
  blurb: string;              // One-line description
  order?: number;             // Display priority (lower = first)
  aliases?: string[];         // ["tg"] so users can type "tg" instead
  systemImage?: string;       // Icon name
  forceAccountBinding?: boolean; // WhatsApp forces this (QR login)
  preferOver?: string[];      // BlueBubbles prefers over native iMessage
  // ... more display/UX hints
};
```

The `order` field controls the display order in the setup wizard. Telegram is first (order ~1), then WhatsApp, Discord, etc.

### Channel Capabilities

This is the channel's "feature checklist" — what can it actually do?

```typescript
type ChannelCapabilities = {
  chatTypes: Array<"direct" | "group" | "channel" | "thread">;
  polls?: boolean;        // Can it send polls?
  reactions?: boolean;    // Can it add emoji reactions?
  edit?: boolean;         // Can it edit sent messages?
  unsend?: boolean;       // Can it delete sent messages?
  reply?: boolean;        // Can it reply to specific messages?
  effects?: boolean;      // Can it send message effects (like iMessage)?
  groupManagement?: boolean; // Can it add/remove group members?
  threads?: boolean;      // Does it support threaded conversations?
  media?: boolean;        // Can it send images/files?
  nativeCommands?: boolean; // Does it have slash commands?
  blockStreaming?: boolean; // Should it wait for full response before sending?
};
```

Here's a comparison of what different channels can do:

| Capability | Telegram | WhatsApp | Discord | Slack | Signal | iMessage | IRC |
|-----------|----------|----------|---------|-------|--------|----------|-----|
| Chat types | DM, group, channel, thread | DM, group | DM, channel, thread | DM, channel, thread | DM, group | DM, group | DM, group |
| Polls | Yes | Yes | Yes | Yes | No | No | No |
| Reactions | Yes | Yes | Yes | Yes | No | No | No |
| Edit | No | No | Yes | Yes | No | No | No |
| Threads | Yes | No | Yes | Yes | No | No | No |
| Media | Yes | Yes | Yes | Yes | Yes | Yes | No |
| Effects | No | No | No | No | No | Yes | No |
| Native commands | Yes | No | Yes | Yes | No | No | No |

The robot checks these capabilities before trying to do something. If you ask it to create a poll on IRC, it knows that's impossible and will tell you.

---

## Part 2: Adapters (The Specialized Sub-Abilities)

Adapters are the real workhorses. Each adapter handles one aspect of channel functionality. Think of them like attachments for a power tool — the drill (channel) is the base, and you snap on different bits (adapters) for different jobs.

### Config Adapter (Required) — "How Do I Read My Settings?"

**File: `src/channels/plugins/types.adapters.ts`**

Every channel MUST have a config adapter. It tells the system how to find and manage accounts for this channel.

```typescript
type ChannelConfigAdapter<ResolvedAccount> = {
  // List all account IDs from config (e.g., ["default", "work-bot"])
  listAccountIds: (cfg: OpenClawConfig) => string[];

  // Turn an account ID into a full account object
  resolveAccount: (cfg: OpenClawConfig, accountId?: string | null) => ResolvedAccount;

  // Which account to use when none is specified
  defaultAccountId?: (cfg: OpenClawConfig) => string;

  // Is this account turned on?
  isEnabled?: (account: ResolvedAccount, cfg: OpenClawConfig) => boolean;

  // Is this account properly set up?
  isConfigured?: (account: ResolvedAccount, cfg: OpenClawConfig) => boolean | Promise<boolean>;

  // Why isn't this account working?
  unconfiguredReason?: (account: ResolvedAccount, cfg: OpenClawConfig) => string;
  disabledReason?: (account: ResolvedAccount, cfg: OpenClawConfig) => string;

  // Enable/disable an account in config
  setAccountEnabled?: (params) => OpenClawConfig;
  deleteAccount?: (params) => OpenClawConfig;

  // Security: who is allowed to DM this bot?
  resolveAllowFrom?: (params) => Array<string | number> | undefined;
  formatAllowFrom?: (params) => string[];
  resolveDefaultTo?: (params) => string | undefined;
};
```

**Why multi-account?** Imagine you run two Telegram bots — one for personal use, one for work. Each is an "account" under the same channel. The config adapter lets OpenClaw manage both.

### Outbound Adapter — "How Do I Send Messages?"

This is how the robot talks *back* to users.

```typescript
type ChannelOutboundAdapter = {
  // How should messages be routed?
  deliveryMode: "direct" | "gateway" | "hybrid";

  // How to split long messages (each platform has character limits)
  chunker?: ((text: string, limit: number) => string[]) | null;
  chunkerMode?: "text" | "markdown";  // Split on words or preserve markdown?
  textChunkLimit?: number;  // Telegram: 4000, Zalo: 2000, etc.

  // Max poll options (Telegram: 10, etc.)
  pollMaxOptions?: number;

  // Figure out WHERE to send
  resolveTarget?: (params) => { ok: true; to: string } | { ok: false; error: Error };

  // The actual "send" functions
  sendPayload?: (ctx: ChannelOutboundPayloadContext) => Promise<OutboundDeliveryResult>;
  sendText?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendMedia?: (ctx: ChannelOutboundContext) => Promise<OutboundDeliveryResult>;
  sendPoll?: (ctx: ChannelPollContext) => Promise<ChannelPollResult>;
};
```

**Delivery modes explained:**
- **`direct`**: The channel plugin sends the message itself, directly to the platform API (most channels use this)
- **`gateway`**: The message goes through the gateway server, which routes it (used when the channel needs the gateway as a middleman)
- **`hybrid`**: Some messages go direct, some go through the gateway

**Why chunking matters:** Telegram has a 4,000-character limit per message. If the AI writes a 10,000-character response, the chunker splits it into 3 messages — but smartly, at paragraph or sentence boundaries, not mid-word. The `chunkerMode: "markdown"` option makes sure it doesn't break markdown formatting (like splitting a code block in half).

### Gateway Adapter — "How Do I Start and Stop?"

This controls the channel's runtime lifecycle.

```typescript
type ChannelGatewayAdapter<ResolvedAccount = unknown> = {
  // Boot up this channel account
  startAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<unknown>;

  // Shut it down gracefully
  stopAccount?: (ctx: ChannelGatewayContext<ResolvedAccount>) => Promise<void>;

  // QR code login (WhatsApp-style)
  loginWithQrStart?: (params) => Promise<ChannelLoginWithQrStartResult>;
  loginWithQrWait?: (params) => Promise<ChannelLoginWithQrWaitResult>;

  // Full logout (clear credentials)
  logoutAccount?: (ctx) => Promise<ChannelLogoutResult>;
};
```

The `ChannelGatewayContext` is the "backpack" given to a channel when it starts. It contains:

```typescript
type ChannelGatewayContext<ResolvedAccount> = {
  cfg: OpenClawConfig;         // Full system config
  accountId: string;           // Which account is starting
  account: ResolvedAccount;    // The resolved account object
  runtime: RuntimeEnv;         // System runtime info
  abortSignal: AbortSignal;    // "Stop!" signal from the system
  log?: ChannelLogSink;        // Where to write logs
  getStatus: () => ChannelAccountSnapshot;  // Read current status
  setStatus: (next: ChannelAccountSnapshot) => void; // Update status
  channelRuntime?: PluginRuntime["channel"]; // SDK helpers (since Feb 2026)
};
```

The `abortSignal` is particularly important. When the system wants to stop a channel, it triggers this signal, and the channel must clean up and exit. It's like a school bell — when it rings, you stop what you're doing.

### Security Adapter — "Who Is Allowed to Talk to Me?"

```typescript
type ChannelSecurityAdapter<ResolvedAccount> = {
  // What's the DM policy for this channel?
  resolveDmPolicy?: (ctx) => ChannelSecurityDmPolicy | null;

  // Any security warnings to show?
  collectWarnings?: (ctx) => Promise<string[]> | string[];
};

type ChannelSecurityDmPolicy = {
  policy: string;                          // "allowlist" or "open"
  allowFrom?: Array<string | number> | null; // Who can DM
  policyPath?: string;                     // Config path for the policy
  allowFromPath: string;                   // Config path for the allowlist
  approveHint: string;                     // "Add their phone number to..."
  normalizeEntry?: (raw: string) => string; // Clean up input
};
```

This is the robot's bouncer. Without it, anyone in the world could message your bot and use your AI credits. The allowlist says "only these people can talk to me." For Telegram, entries are user IDs. For Signal, they're phone numbers. For Discord, they're user IDs or role names.

### Status Adapter — "Am I Healthy?"

```typescript
type ChannelStatusAdapter<ResolvedAccount, Probe, Audit> = {
  // Default runtime state before the channel starts
  defaultRuntime?: ChannelAccountSnapshot;

  // Quick health check (is the token valid? is the bot online?)
  probeAccount?: (params) => Promise<Probe>;

  // Deep audit (what groups am I in? what permissions do I have?)
  auditAccount?: (params) => Promise<Audit>;

  // Build the full status snapshot
  buildAccountSnapshot?: (params) => ChannelAccountSnapshot | Promise<...>;

  // Collect warnings/issues
  collectStatusIssues?: (accounts: ChannelAccountSnapshot[]) => ChannelStatusIssue[];
};
```

The **Account Snapshot** is a detailed health report:

```typescript
type ChannelAccountSnapshot = {
  accountId: string;
  enabled?: boolean;         // Is it turned on?
  configured?: boolean;      // Does it have all required settings?
  linked?: boolean;          // Is it linked to a platform account?
  running?: boolean;         // Is the process running right now?
  connected?: boolean;       // Is it connected to the platform?
  reconnectAttempts?: number; // How many times has it tried to reconnect?
  lastConnectedAt?: number;  // When did it last connect?
  lastMessageAt?: number;    // When did it last send/receive?
  lastError?: string;        // What went wrong?
  busy?: boolean;            // Is it processing something?
  activeRuns?: number;       // How many AI runs are active?
  // ... many more fields
};
```

### Directory Adapter — "Who's Out There?"

```typescript
type ChannelDirectoryAdapter = {
  self?: (params) => Promise<ChannelDirectoryEntry | null>;  // Who am I?
  listPeers?: (params) => Promise<ChannelDirectoryEntry[]>;  // My contacts
  listGroups?: (params) => Promise<ChannelDirectoryEntry[]>; // My groups
  listGroupMembers?: (params) => Promise<ChannelDirectoryEntry[]>; // Group members
  listPeersLive?: (params) => Promise<ChannelDirectoryEntry[]>; // Fresh from API
  listGroupsLive?: (params) => Promise<ChannelDirectoryEntry[]>; // Fresh from API
};

type ChannelDirectoryEntry = {
  kind: "user" | "group" | "channel";
  id: string;
  name?: string;
  handle?: string;
  avatarUrl?: string;
};
```

There are "cached" versions (`listPeers`) and "live" versions (`listPeersLive`). The cached ones are fast but might be stale. The live ones hit the platform's API directly.

### Threading Adapter — "How Do I Handle Conversation Threads?"

```typescript
type ChannelThreadingAdapter = {
  // When should I reply in threads? "off" = never, "first" = only to first msg, "all" = always
  resolveReplyToMode?: (params) => "off" | "first" | "all";

  // Allow explicit @reply tags even when threading is off?
  allowExplicitReplyTagsWhenOff?: boolean;

  // Build context about the current thread
  buildToolContext?: (params) => ChannelThreadingToolContext | undefined;
};
```

The thread context tells the AI *where* it is in a conversation:

```typescript
type ChannelThreadingContext = {
  Channel?: string;           // "telegram"
  From?: string;              // Who sent this message
  To?: string;                // Who received it
  ChatType?: string;          // "direct" | "group" | "channel"
  CurrentMessageId?: string;  // This message's ID
  ReplyToId?: string;         // What message is this replying to
  ThreadLabel?: string;       // Thread name (if applicable)
  MessageThreadId?: string;   // Platform-specific thread ID
};
```

### Message Action Adapter — "What Special Things Can I Do?"

This is the richest adapter. It defines **all the actions** a channel supports:

```typescript
const CHANNEL_MESSAGE_ACTION_NAMES = [
  "send", "broadcast", "poll", "poll-vote", "react", "reactions",
  "read", "edit", "unsend", "reply", "sendWithEffect",
  "renameGroup", "setGroupIcon", "addParticipant", "removeParticipant", "leaveGroup",
  "sendAttachment", "delete", "pin", "unpin", "list-pins",
  "permissions", "thread-create", "thread-list", "thread-reply", "search",
  "sticker", "sticker-search", "member-info", "role-info",
  "emoji-list", "emoji-upload", "sticker-upload",
  "role-add", "role-remove",
  "channel-info", "channel-list", "channel-create", "channel-edit", "channel-delete", "channel-move",
  "category-create", "category-edit", "category-delete",
  "topic-create", "voice-status", "event-list", "event-create",
  "timeout", "kick", "ban", "set-presence", "download-file"
];
```

That's **50+ possible actions**! No single channel supports all of them. Discord supports the most (channels, roles, threads, kicks, bans). iMessage is simpler (send, react, effects). IRC is the simplest (basically just send).

The adapter declares which actions it supports:

```typescript
type ChannelMessageActionAdapter = {
  listActions?: (params) => ChannelMessageActionName[];  // What can I do?
  supportsAction?: (params) => boolean;                  // Can I do this specific thing?
  supportsButtons?: (params) => boolean;                 // Interactive buttons?
  supportsCards?: (params) => boolean;                   // Rich cards?
  handleAction?: (ctx) => Promise<AgentToolResult>;      // Do the thing!
};
```

### Other Adapters (Quick Reference)

| Adapter | Purpose |
|---------|---------|
| `onboarding` | CLI setup wizard steps |
| `setup` | Apply account configuration |
| `pairing` | Device pairing / QR code flows |
| `groups` | Group-specific policies (require @mention, tool restrictions) |
| `mentions` | Strip/format @mentions from messages |
| `auth` | Login flows |
| `elevated` | DM fallback policies |
| `commands` | Command gating (who can run commands) |
| `streaming` | Block streaming behavior (send full response vs. stream) |
| `agentPrompt` | Inject hints into the AI prompt |
| `resolver` | Bulk target resolution |
| `heartbeat` | Health check ping |
| `agentTools` | Channel-specific AI tools |

---

## Part 3: The Channel Registry (The Master List)

**File: `src/channels/registry.ts`**

The registry is the official catalog of known channels.

### The Core Channel Order

```typescript
export const CHAT_CHANNEL_ORDER = [
  "telegram", "whatsapp", "discord", "irc", "googlechat",
  "slack", "signal", "imessage", "line"
] as const;
```

This array defines the **display order** in the setup wizard and status screens. Telegram is first because it's the easiest to set up (just create a bot and paste a token).

### Aliases

```typescript
const CHAT_CHANNEL_ALIASES: Record<string, ChatChannelId> = {
  imsg: "imessage",
  "internet-relay-chat": "irc",
  "google-chat": "googlechat",
  gchat: "googlechat",
};
```

These are shortcuts so users can type `imsg` instead of `imessage`. The `normalizeChatChannelId()` function handles the translation.

### Plugin Loading and Caching

**File: `src/channels/plugins/index.ts`**

```typescript
type CachedChannelPlugins = {
  registryVersion: number;   // Invalidates when plugins reload
  sorted: ChannelPlugin[];   // All plugins, sorted by order
  byId: Map<string, ChannelPlugin>; // Quick lookup by ID
};

function listChannelPlugins(): ChannelPlugin[]
function getChannelPlugin(id: ChannelId): ChannelPlugin | undefined
function normalizeChannelId(raw?: string | null): ChannelId | null
```

The cache is invalidated whenever the plugin registry version changes (which happens when plugins are loaded or hot-reloaded). This means calling `listChannelPlugins()` a hundred times in quick succession only actually loads plugins once.

---

## Part 4: Channel Lifecycle Management (Starting, Stopping, Restarting)

**File: `src/gateway/server-channels.ts`**

The Channel Manager is the "operations center" that controls every channel's lifecycle.

### The Channel Manager

```typescript
type ChannelManager = {
  getRuntimeSnapshot: () => ChannelRuntimeSnapshot;
  startChannels: () => Promise<void>;                              // Start ALL channels
  startChannel: (channel: ChannelId, accountId?: string) => Promise<void>;  // Start one
  stopChannel: (channel: ChannelId, accountId?: string) => Promise<void>;   // Stop one
  markChannelLoggedOut: (channelId, cleared, accountId?) => void;
  isManuallyStopped: (channelId, accountId) => boolean;
  resetRestartAttempts: (channelId, accountId) => void;
};
```

### The Startup Flow

When the gateway boots, here's what happens for each channel:

```
1. Load config → get list of channel accounts
         |
2. For each account:
         |
    a. Is it enabled? (config.isEnabled) ─── No ──→ Skip
         |
        Yes
         |
    b. Is it configured? (config.isConfigured) ─── No ──→ Log warning, skip
         |
        Yes
         |
    c. Create AbortController (the "stop button")
         |
    d. Create runtime store (status tracking)
         |
    e. Call gateway.startAccount(context) ──→ Channel connects to platform
         |
    f. Update status: { running: true, connected: true }
         |
    g. Channel is live! Listening for messages.
```

### The Auto-Restart System

Channels can crash. Network drops, API rate limits, expired tokens — lots can go wrong. The Channel Manager handles this with **exponential backoff**:

```
Attempt 1: Wait 5 seconds, then restart
Attempt 2: Wait 10 seconds
Attempt 3: Wait 20 seconds
Attempt 4: Wait 40 seconds
...
Attempt N: Wait min(5s × 2^N, 5 minutes)  ← caps at 5 minutes
```

After **10 failed attempts**, the system gives up and marks the channel as permanently failed. A human needs to investigate.

Key rules:
- If a user **manually stops** a channel, it does NOT auto-restart
- Each account is tracked independently (your personal Telegram bot crashing doesn't affect your work bot)
- Successful reconnection resets the attempt counter to zero

### Internal Runtime Storage

```typescript
type ChannelRuntimeStore = {
  aborts: Map<string, AbortController>;    // One "stop button" per account
  tasks: Map<string, Promise<unknown>>;    // Running account tasks
  runtimes: Map<string, ChannelAccountSnapshot>; // Per-account status
};
```

### Runtime Snapshot

At any time, you can ask "what's the status of all my channels?" The snapshot gives you:

```typescript
type ChannelRuntimeSnapshot = {
  channels: Partial<Record<ChannelId, ChannelAccountSnapshot>>;          // Default accounts
  channelAccounts: Partial<Record<ChannelId, Record<string, ChannelAccountSnapshot>>>; // All accounts
};
```

This powers the health dashboard that shows green/yellow/red status per channel.

---

## Part 5: The Inbound Message Flow (How Messages Get Into the System)

When someone sends a message on Telegram, here's the full journey:

### Step 1: Platform SDK Receives the Message

Each channel uses its platform's SDK or API. Telegram uses webhook callbacks. Discord uses WebSocket. WhatsApp uses a web session. The channel's `startAccount` function sets up the listener.

### Step 2: Channel Normalizes Into MsgContext

The channel converts the platform-specific message into a universal `MsgContext` — the lingua franca of OpenClaw.

**File: `src/auto-reply/templating.ts`**

```typescript
type MsgContext = {
  // --- Content ---
  Body?: string;              // "Hey, what's the weather?"
  BodyForAgent?: string;      // Cleaned version for the AI
  RawBody?: string;           // Original untouched text

  // --- Sender ---
  From?: string;              // "telegram:123456789"
  SenderId?: string;          // "123456789"
  SenderName?: string;        // "Alice"
  SenderUsername?: string;    // "alice_wonderland"

  // --- Destination ---
  To?: string;                // "telegram:bot-default"
  Channel?: string;           // "telegram"
  AccountId?: string;         // "default"
  ChatType?: string;          // "direct" | "group"

  // --- Message Identity ---
  MessageSid?: string;        // Unique message ID
  MessageThreadId?: string;   // Thread ID (if in a thread)

  // --- Reply Context ---
  ReplyToId?: string;         // ID of the message being replied to
  ReplyToBody?: string;       // Text of that message
  ReplyToSender?: string;     // Who sent the original

  // --- Group Context ---
  GroupSubject?: string;      // "Engineering Team"
  GroupMembers?: string;      // "Alice, Bob, Charlie"
  WasMentioned?: boolean;     // Was the bot @mentioned?

  // --- Media ---
  MediaPath?: string;         // Local path to downloaded media
  MediaType?: string;         // "image/png", "audio/ogg", etc.
  MediaPaths?: string[];      // Multiple attachments
  Transcript?: string;        // Audio transcription

  // --- Forwarding ---
  ForwardedFrom?: string;     // If this was forwarded, who from?
  ForwardedFromType?: string; // "user" | "channel"

  // --- Security ---
  CommandAuthorized?: boolean; // Is this sender allowed to run commands?
  OwnerAllowFrom?: Array<string | number>; // The allowlist

  // --- Routing ---
  SessionKey?: string;        // How to identify this conversation
  OriginatingChannel?: OriginatingChannelType;

  // ... 40+ more fields
};
```

That's a LOT of fields — and that's intentional. Different channels provide different information. Telegram gives you forwarding info. Discord gives you thread data. iMessage gives you message effects. The `MsgContext` is the "universal envelope" that can hold information from any platform.

### Step 3: Dispatch to the AI

**File: `src/auto-reply/dispatch.ts`**

```typescript
async function dispatchInboundMessage(params: {
  ctx: MsgContext;           // The normalized message
  cfg: OpenClawConfig;       // System config
  dispatcher: ReplyDispatcher; // How to send the response back
}): Promise<DispatchInboundResult>
```

The dispatch flow:
1. **Finalize context** — set `CommandAuthorized` to `false` if not explicitly set (default-deny security)
2. **Route** — figure out which AI agent handles this conversation (Layer 1's routing)
3. **Memory search** — find relevant past conversations (Layer 7)
4. **Context assembly** — build the prompt within token budget (Layer 8)
5. **AI inference** — send to the AI provider (Layer 5)
6. **Tool execution** — if the AI wants to use tools (Layer 6)
7. **Reply dispatch** — send the response back through the channel's outbound adapter

### Step 4: Response Goes Back Out

The `ReplyDispatcher` sends the AI's response back through the channel's outbound adapter, which converts it into platform-specific format and sends it via the platform's API.

---

## Part 6: The Built-In Channels (All 22 of Them)

### Tier 1: Core Messaging (The Original Six)

These were the first channels, built before the plugin system even existed.

#### Telegram (`extensions/telegram/`)

The "golden child" — simplest to set up, most complete feature set.

- **Auth**: Bot token (from @BotFather)
- **Chat types**: DM, group, channel, thread
- **Features**: Polls, reactions, threads, media, native commands (slash commands)
- **Text limit**: 4,000 characters (markdown chunking)
- **Gateway**: Webhook + polling modes
- **Status**: Bot probe (username, rate limits), group audit (membership, permissions)
- **Unique**: Supports both webhook (production) and polling (development) modes. The probe can check if the bot token is still valid without sending a message.

```typescript
export const telegramPlugin: ChannelPlugin<ResolvedTelegramAccount, TelegramProbe> = {
  id: "telegram",
  capabilities: {
    chatTypes: ["direct", "group", "channel", "thread"],
    reactions: true, threads: true, media: true,
    polls: true, nativeCommands: true, blockStreaming: true,
  },
  outbound: {
    deliveryMode: "direct",
    chunkerMode: "markdown",
    textChunkLimit: 4000,
    pollMaxOptions: 10,
  },
  // ... full adapter suite
};
```

#### WhatsApp (`extensions/whatsapp/`)

- **Auth**: QR code web login (like WhatsApp Web)
- **Chat types**: DM, group only (no threads or channels)
- **Features**: Polls, reactions, media
- **Unique**: Forces account binding — your WhatsApp number IS the account. No bot tokens here, it's your actual phone.
- **Session management**: Web-based session that can expire (requires re-scan of QR code)
- **Phone normalization**: E164 format for allowlists

#### Discord (`extensions/discord/`)

- **Auth**: Bot token (from Discord Developer Portal)
- **Chat types**: DM, channel, thread
- **Features**: Polls, reactions, threads, media, slash commands, streaming
- **Streaming**: Coalesces responses (waits for 1500 chars or 1s idle before sending)
- **Unique**: Subagent hooks for advanced features. Supports both bot token and optional user token (read-only).
- **Rich actions**: Channel management, role management, kicks, bans, pins — Discord has the most actions of any channel

#### Slack (`extensions/slack/`)

- **Auth**: Bot token + optional user token
- **Chat types**: DM, channel, thread
- **Features**: Polls, reactions, threads, media, interactive replies
- **Modes**: Socket Mode (default, simpler) or HTTP webhook mode
- **Unique**: Dual token support. The bot token does most things, but a user token enables read-only access to more data.

#### Signal (`extensions/signal/`)

- **Auth**: E164 phone number (your Signal number)
- **Chat types**: DM, group only
- **Features**: Media (with size limits)
- **Backend**: Uses signal-cli (a command-line Signal client) or an HTTP wrapper
- **Unique**: Phone-number-based identity. The most privacy-focused channel.

#### iMessage (`extensions/imessage/`)

- **Auth**: Native macOS integration
- **Chat types**: DM, group
- **Features**: Media
- **Unique**: Only works on macOS. Simple wrapper around the system framework. If you want more features (reactions, effects, edit, unsend), use BlueBubbles instead.

### Tier 2: Enterprise/Team Platforms

#### Mattermost (`extensions/mattermost/`)
- Self-hosted team platform (like an open-source Slack)
- Webhook inbound + WebSocket for real-time
- Slash command registration after connection
- 9 source files — moderately complex

#### Feishu/Lark (`extensions/feishu/`)
- Chinese enterprise platform (Lark is the international name)
- **53 source files** — the most complex channel by far
- Multi-domain: Chat, Documents, Wiki, Drive, Permissions, Bitable (spreadsheet DB)
- Registers 6 agent tools for document/data operations
- Card messages with streaming updates (progressive loading)

#### Microsoft Teams (`extensions/msteams/`)
- Uses Microsoft Bot Framework
- Express.js HTTP server + ActivityHandler pattern
- 43 source files
- Reactions support

#### Google Chat (`extensions/googlechat/`)
- Google Cloud service account authentication
- Webhook inbound + REST API outbound
- Space-based routing (Google Chat uses "spaces" instead of "channels")
- 14+ source files

#### Matrix (`extensions/matrix/`)
- Open-source federated protocol
- **End-to-end encryption** support (via matrix-sdk-crypto)
- Live directory discovery
- Was rewritten from scratch in March 2026 (old version: `matrix-js`)

#### Nextcloud Talk (`extensions/nextcloud-talk/`)
- Self-hosted Nextcloud integration
- Webhook bot pattern
- Room-based routing

#### Synology Chat (`extensions/synology-chat/`)
- Synology NAS native chat
- Webhook-based, simple architecture

### Tier 3: Specialized/Regional Channels

#### LINE (`extensions/line/`)
- Dominant in Japan, Taiwan, Thailand
- Rich card/Flex messages for UI
- Card command registration

#### IRC (`extensions/irc/`)
- Classic Internet Relay Chat (oldest protocol supported!)
- Multi-network configuration
- SSL, SASL authentication
- 16+ source files — surprisingly complex for such an old protocol

#### Zalo (`extensions/zalo/`)
- Vietnam's dominant messaging platform
- Bot API (official business accounts)
- 2,000-character text limit

#### Zalo Personal (`extensions/zalouser/`)
- Zalo personal accounts (not bot)
- Uses `zca-js` library for direct account access
- Registers utility tools (send, image, link, friends)

#### Twitch (`extensions/twitch/`)
- Streaming platform chat
- Uses Twurple SDK
- Primarily group (channel chat)

#### BlueBubbles (`extensions/bluebubbles/`)
- iMessage via BlueBubbles macOS app
- REST API bridge to iMessage
- **Full iMessage features**: reactions, edit, unsend, reply, effects, group management
- Explicitly `preferOver: ["imessage"]` — if both are configured, BlueBubbles wins
- 28+ source files

#### Nostr (`extensions/nostr/`)
- Decentralized protocol
- NIP-04 encrypted DMs
- Relay-based messaging
- Public key authentication
- Custom HTTP route for profile management

#### Tlon/Urbit (`extensions/tlon/`)
- Decentralized social network (Urbit)
- CLI-driven (binary discovery for the `tlon` command)
- Command whitelisting for security (only 12 allowed subcommands)

### Tier 4: The Web UI (Built-In, Not an Extension)

**Location: `src/channels/web/`**

The Web UI channel is special — it's not in `extensions/` but built directly into the core. It provides a browser-based chat interface that connects via WebSocket to the gateway. Think of it as the "default" channel that always works, even if no other channels are configured.

---

## Part 7: The Plugin Channel Runtime SDK

**File: `src/plugins/runtime/types.ts`**

When a channel extension starts up, it gets access to a rich SDK via `channelRuntime`. This is what lets external channel plugins do everything built-in channels can do.

```typescript
type PluginRuntimeChannel = {
  // Text processing
  text: {
    chunkByNewline, chunkMarkdownText, chunkText,
    hasControlCommand, resolveChunkMode,
  };

  // Reply dispatching (the core AI response flow)
  reply: {
    dispatchReplyWithBufferedBlockDispatcher,
    createReplyDispatcherWithTyping,
    dispatchReplyFromConfig,
    withReplyDispatcher,
    finalizeInboundContext,
  };

  // Routing
  routing: {
    buildAgentSessionKey,
    resolveAgentRoute,
  };

  // Device pairing
  pairing: {
    buildPairingReply,
    readAllowFromStore,
    upsertPairingRequest,
  };

  // Media handling
  media: {
    fetchRemoteMedia,
    saveMediaBuffer,
  };

  // Session management
  session: {
    resolveStorePath,
    recordSessionMetaFromInbound,
    recordInboundSession,
    updateLastRoute,
  };

  // @mention handling
  mentions: {
    buildMentionRegexes,
    matchesMentionPatterns,
  };

  // Reaction handling
  reactions: {
    shouldAckReaction,
    removeAckReactionAfterReply,
  };

  // Group policy
  groups: {
    resolveGroupPolicy,
    resolveRequireMention,
  };

  // Command authorization
  commands: {
    resolveCommandAuthorizedFromAuthorizers,
    isControlCommandMessage,
    shouldHandleTextCommands,
  };
};
```

This SDK was added in February 2026 (commit `469cd5b46` by Gu XiaoBo) specifically because external channel plugin authors couldn't access the AI response dispatching pipeline. Before this, only built-in channels could process AI responses. After this commit, any extension can do everything a built-in channel does.

---

## How Layer 4 Evolved (Git History)

### Era 1: The Monolith (Before January 2026)

OpenClaw didn't start with a channel system at all. It started as a **Twilio webhook CLI** — a simple tool that received SMS messages and replied with AI-generated text. There was no concept of "channels" or "plugins."

The first messaging integrations were **hardcoded "providers"** living in `src/providers/plugins/`:
- `telegram.ts`
- `discord.ts`
- `slack.ts`
- `whatsapp.ts`
- `signal.ts`
- `imessage.ts`

Each file contained everything — configuration, sending, receiving, health checks — all in one big file. The type system was minimal. There was no adapter pattern.

MS Teams was even worse — it lived in its own top-level directory `src/msteams/` with no relationship to the other providers at all.

### Era 2: The Great Rename — January 13, 2026

**Commit `90342a4f3`** — `refactor!: rename chat providers to channels`

This was the first architectural awakening. Peter Steinberger realized that "provider" was confusing because the same word was used for both AI providers (OpenAI, Anthropic) and messaging platforms (Telegram, Discord). The fix: messaging platforms became "channels."

**Impact:** 393 files changed, 8,005 insertions, 6,738 deletions.

This wasn't just a search-and-replace. The renaming cascaded through:
- Source directory: `src/providers/` → `src/channels/`
- CLI commands: `providers status` → `channels status`
- Config schema: `providers: {}` → `channels: {}`
- Gateway methods: `providers.list` → `channels.list`
- Documentation: `docs/providers/` → `docs/channels/`

This was a **breaking change** (the `!` in the commit message), meaning all existing configs had to be updated.

### Era 3: The Plugin Architecture — January 11-15, 2026

Just two days before the great rename, the plugin system was born:

**January 11 — `cf0c72a55`** — `feat: add plugin architecture`
- Created `src/plugins/` with 37 new files
- First plugin: `extensions/voice-call/` (not a channel — a voice call tool)
- Established the pattern: extensions live in `extensions/`, register via `api.registerX()`

**January 15 — `2b4a68e27`** — `feat: load channel plugins`
- Connected the plugin system to channels
- Created `src/channels/plugins/index.ts` — the bridge between plugins and channels
- Now channels could be loaded as plugins instead of being hardcoded

**January 15 — `5abe3c214`** — `feat: add plugin HTTP hooks + Zalo plugin`
- Zalo was the **FIRST channel plugin** (not a built-in channel)
- Added HTTP hook support so channels could receive webhooks through the plugin system
- 36 files, 3,061 insertions

**January 15 — `725a6b71d`** — `feat: add matrix channel plugin`
- Matrix was the **second channel plugin**
- Very comprehensive: E2EE support, poll types, thread binding
- 51 files, 3,986 insertions

### Era 4: The Mass Migration — January 16-18, 2026

This is when the real transformation happened. Every built-in channel was extracted from `src/channels/plugins/` and moved to `extensions/`.

**January 16 — `d9f9e93de`** — MS Teams goes first:
- Extracted from `src/msteams/` to `extensions/msteams/`
- First built-in channel to become a plugin
- Created `src/channels/plugins/catalog.ts` to track bundled plugins

**January 16 — `390bd11f3`** — Zalo Personal + Adapter Types:
- Community contribution by @suminhthanh (first community channel!)
- Introduced `src/channels/plugins/types.adapters.ts` — the adapter pattern was born
- Added the directory CLI for listing contacts/groups

**January 18 — `c5e19f5c6`** — THE BIG MIGRATION:
- **All six original channels** moved to extensions in a single commit
- Created: `extensions/discord/`, `extensions/imessage/`, `extensions/signal/`, `extensions/slack/`, `extensions/telegram/`, `extensions/whatsapp/`
- Massively expanded `src/plugin-sdk/index.ts` (+160 lines of SDK exports)
- 63 files, 4,076 insertions

**January 18 — `ee6e534cc`** — The cleanup:
- **Deleted the old built-in files**: `src/channels/plugins/discord.ts`, `imessage.ts`, `signal.ts`, `slack.ts`, `telegram.ts`, `whatsapp.ts` — 2,576 lines removed
- Created `src/plugins/runtime/types.ts` (181 lines) — the runtime contract
- The monolith era was officially over

### Era 5: The Cambrian Explosion — January 20 to February 22, 2026

With the plugin architecture mature, new channels appeared at incredible speed:

| Date | Channel | Author | Notable |
|------|---------|--------|---------|
| Jan 20 | **Nostr** | Peter + @joelklabo | Decentralized protocol |
| Jan 21 | **Mattermost** | Dominic Damoah | Open-source Slack alternative |
| Jan 23 | **Google Chat** | iHildy | First "beta" channel |
| Jan 25 | **LINE** | @plum-dawg | Asian market expansion |
| Jan 27 | **Twitch** | Community | First streaming platform |
| Feb 2-3 | **Feishu/Lark** | Josh Palmer | Most complex channel (53 files) |
| Feb 10 | **IRC** | Vignesh | Oldest protocol, newest plugin |
| Feb 13 | **Linq/BlueBubbles** | George McCain | Rich iMessage bridge |
| Feb 22 | **Synology Chat** | Jean-Marc | NAS-native chat |

In one month, the channel count went from 6 to 20+. This was only possible because the plugin architecture made it easy to add channels without touching the core.

### Era 6: The Rename Cascade — January 27-30, 2026

During the explosion, the project itself was renamed twice:

1. **Jan 27 — `6d16a658e`** — `refactor: rename clawdbot to moltbot with legacy compat`
2. **Jan 30 — `9a7160786`** — `refactor: rename to openclaw`

Each rename touched hundreds of files (the second one exceeded 140KB of diff). Config keys, package names, CLI commands, environment variables — everything changed. Extensions had to update their imports and registrations.

### Era 7: SDK Maturation — February-March 2026

The plugin SDK evolved to give external channels feature parity with built-in ones:

**February 24 — `469cd5b46`** — `feat(plugin-sdk): Add channelRuntime support for external channel plugins`
- By Gu XiaoBo — enabled external plugins to dispatch AI responses
- Before this, only built-in channels could process the AI reply pipeline
- Added `channelRuntime` to the gateway context

**March 8 — `8e962668c`** — Matrix rewrite:
- 273 files changed, 7,179 insertions, 16,054 deletions
- Deleted the original `extensions/matrix-js/` entirely
- Replaced with a ground-up rewrite at `extensions/matrix/`
- Added migration docs and legacy crypto restore

### Era 8: The Great Deduplication — March 2026

With 20+ channels, there was a LOT of duplicated code. A sustained wave of refactoring shared common patterns:

- **Group policy** — Unified across all channels that support groups
- **Config adapters** — Shared base helpers for allowFrom, defaultTo, account accessors
- **Onboarding** — Unified DM/group policy wizard scaffolding
- **Allowlists** — Shared wildcard matching and compilation
- **Outbound** — Unified threading and send flows
- **Plugin SDK scoping** — Each channel migrated to "scoped imports" (import only what you need)
- **sendPayload** — Added to all adapters with chunking and empty-payload guards

This deduplication phase was critical — without it, fixing a bug in group policy would have required changes in 15+ files across 15+ extensions.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────┐
│                    Gateway (Layer 2)                     │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Channel Manager                       │  │
│  │  start / stop / restart / health check            │  │
│  │  exponential backoff (5s → 5min, max 10 retries)  │  │
│  └─────────────────────┬─────────────────────────────┘  │
│                        │                                 │
│  ┌─────────────────────┴─────────────────────────────┐  │
│  │            Channel Plugin Contract                 │  │
│  │                                                    │  │
│  │  Required:  id, meta, capabilities, config         │  │
│  │  Optional:  ~20 adapters (outbound, gateway,       │  │
│  │             security, status, threading, etc.)      │  │
│  └─────────────────────┬─────────────────────────────┘  │
└────────────────────────┼────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
    │Telegram │    │ Discord │    │  Slack  │    ... 19 more
    │         │    │         │    │         │
    │ Bot API │    │ Gateway │    │ Socket  │
    │ Webhook │    │ Socket  │    │  Mode   │
    │         │    │         │    │         │
    │ DM,grp, │    │ DM,chan,│    │ DM,chan,│
    │ chan,thr │    │ thread  │    │ thread  │
    │         │    │         │    │         │
    │ Polls,  │    │ Polls,  │    │ Polls,  │
    │ react,  │    │ react,  │    │ react,  │
    │ media   │    │ slash,  │    │ media   │
    │         │    │ stream, │    │         │
    │         │    │ roles,  │    │         │
    │         │    │ kicks   │    │         │
    └─────────┘    └─────────┘    └─────────┘

Inbound Flow:
  Platform → Channel Plugin → MsgContext → Dispatch → AI → Reply → Channel Plugin → Platform

Outbound Flow:
  AI Response → Outbound Adapter → Chunker → Platform API → User
```

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `src/channels/plugins/types.plugin.ts` | The ChannelPlugin contract (~20 adapters) |
| `src/channels/plugins/types.adapters.ts` | All adapter type definitions (383 lines) |
| `src/channels/plugins/types.core.ts` | Core types: capabilities, snapshots, policies (403 lines) |
| `src/channels/registry.ts` | Master channel list, aliases, metadata |
| `src/channels/plugins/index.ts` | Plugin loading, caching, lookup |
| `src/channels/plugins/message-action-names.ts` | 50+ action constants |
| `src/gateway/server-channels.ts` | Channel Manager lifecycle |
| `src/auto-reply/dispatch.ts` | Inbound message dispatch pipeline |
| `src/auto-reply/templating.ts` | MsgContext type definition |
| `src/plugins/runtime/types.ts` | Channel Runtime SDK types |
| `extensions/telegram/src/channel.ts` | Telegram plugin (reference implementation) |
| `extensions/discord/src/channel.ts` | Discord plugin (most feature-rich) |
| `extensions/feishu/` | Feishu plugin (most complex, 53 files) |
| `extensions/matrix/` | Matrix plugin (E2E encryption) |
