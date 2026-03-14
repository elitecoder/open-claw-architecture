# Layer 2: Gateway — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Layer 2?

If Layer 1 (Core Runtime) is the robot's brain stem, Layer 2 (Gateway) is its **entire nervous system**. It's the central switchboard that connects everything together — your phone, your Discord, your web browser, your AI models, and any other devices.

The gateway is a **server** that runs on your machine. Everything talks to it:
- Your iPhone app connects to it over WebSocket
- Discord messages arrive through it
- AI responses flow back through it
- Your web browser talks to it via HTTP

Think of it as a phone exchange operator from the 1950s — every call (message) goes through them, and they plug the right cables together.

**By the numbers:** `src/gateway/` contains 247 files and ~4.8MB of code. It has received **1,720 commits** in just 3 months — about 19 commits per day.

---

## Part 1: The 10-Phase Startup Sequence

When you run `openclaw gateway`, the server boots up in 10 carefully ordered phases. Each phase depends on the ones before it. If any critical phase fails, the whole startup stops.

### Phase 1: Environment Setup (Lines 271-283)

```typescript
process.env.OPENCLAW_GATEWAY_PORT = String(port);
```

- Detects if we're running in test mode (minimal startup for unit tests)
- Stores the gateway port in the environment so child processes (browser, canvas) know where to connect
- Logs which environment options are active

**Like:** Writing your address on a sticky note so everyone in the house knows where to send mail.

### Phase 2: Config Validation & Migration (Lines 285-332)

```typescript
let configSnapshot = await readConfigFileSnapshot();
if (configSnapshot.legacyIssues.length > 0) {
  const { config: migrated } = migrateLegacyConfig(configSnapshot.parsed);
  if (migrated) await writeConfigFile(migrated);
}
```

- Reads `config.json5` and takes a snapshot
- Checks for legacy (outdated) config patterns
- Auto-migrates old config formats to new ones
- Re-reads config to ensure a fresh, validated snapshot
- Sets the runtime config snapshot so all other code uses this version

**Like:** Opening your instruction manual and checking if it's from an old version. If so, translate it to the current format before continuing.

### Phase 3: Secrets Runtime Activation (Lines 333-360)

- Boots the secrets manager (API keys, tokens, passwords)
- Resolves secret references in the config (e.g., `${OPENAI_API_KEY}`)
- Makes secrets available to all downstream systems

**Like:** Unlocking your safe and laying out all the keys you'll need for the day.

### Phase 4: Auth Bootstrap (Lines 361-420)

- Generates gateway tokens if none exist (first-run scenario)
- Loads TLS certificates if HTTPS is configured
- Sets up the auth resolver (determines which auth method to use)
- Configures rate limiters for brute-force protection

**Like:** Setting up the front door lock and making spare keys for family members.

### Phase 5: Load Plugins + Channel Plugins (Lines 421-490)

- Scans for plugins in `~/.openclaw/extensions/`, bundled `extensions/`, and workspace paths
- Loads each plugin module using `jiti` (dynamic TypeScript loader)
- Registers channels (Discord, Telegram, etc.), providers, tools, hooks, and HTTP routes
- Channel plugins get their own registration lifecycle

**Like:** Installing all your apps — Discord, Telegram, Slack — so you can receive messages on each platform.

### Phase 6: Create HTTP/HTTPS + WebSocket Servers (Lines 491-580)

```typescript
const httpServer = createHttpServer(tlsOptions);
const wss = new WebSocketServer({ noServer: true });
```

- Creates the HTTP server (or HTTPS if TLS is configured)
- Creates a WebSocket server attached to the HTTP server
- Binds to the configured port and address
- Sets up the request pipeline for HTTP endpoints

**Like:** Opening your shop — putting up the sign, unlocking the doors, and setting up the counter.

### Phase 7: Start Channel Manager (Lines 581-660)

- Iterates through all configured channels (Discord, Telegram, WhatsApp, etc.)
- Starts each channel with its account credentials
- Sets up exponential backoff restart for failed channels (5 seconds → 5 minutes max)
- Registers channel health monitors

**Like:** Logging into all your messaging apps one by one.

### Phase 8: Start Discovery (Lines 661-730)

Three discovery mechanisms run in parallel:

1. **mDNS/Bonjour**: Advertises the gateway on your local network so devices can find it automatically (like AirDrop discovery)
2. **Tailscale**: If you use Tailscale VPN, advertises via Tailnet DNS so remote devices can find the gateway
3. **Wide-area DNS-SD**: For discovering the gateway across the internet via DNS

**Like:** Putting up a "we're open" sign that's visible from the street (mDNS), from across town (Tailscale), and from anywhere in the world (wide-area DNS).

### Phase 9: Start Timers & Monitors (Lines 731-850)

- **Cron scheduler**: Runs scheduled tasks (like "check my email every hour")
- **Health monitor**: Periodically checks if channels are healthy
- **Heartbeat timer**: Sends `tick` events to all connected clients so they know the server is alive
- **Maintenance timer**: Cleans up stale connections, expired sessions, etc.

**Like:** Setting alarms for regular chores — take out the trash, check the mailbox, make sure the lights are on.

### Phase 10: Attach WebSocket Handlers & Config Watcher (Lines 851-1070)

- Attaches the WebSocket message handler (processes incoming client messages)
- Registers plugin lifecycle hooks
- Sets up the config file watcher for hot-reload (change config → gateway auto-updates without restart)
- Logs the startup complete message with URLs

**Like:** The final check — making sure the phone is plugged in, the intercom works, and writing down the phone number for everyone to call.

---

## Part 2: The Protocol (How Devices Talk to the Gateway)

### Frame Types

Every message between a client and the gateway is a **frame** — a JSON object with a `type` field. There are three types:

#### Request Frame (Client → Server)
```json
{
  "type": "req",
  "id": "abc-123",
  "method": "chat.send",
  "params": { "message": "Hello!", "sessionKey": "agent:main:main" }
}
```

- `id`: A unique ID so the server can match the response to this request
- `method`: What you want to do (like `chat.send`, `config.get`, `health`)
- `params`: The details (different for each method)

**Like:** Writing a letter with a tracking number, a subject line, and the actual message.

#### Response Frame (Server → Client)
```json
{
  "type": "res",
  "id": "abc-123",
  "ok": true,
  "payload": { "runId": "run-456", "status": "started" }
}
```

- `id`: Matches the request's `id` so you know which question this answers
- `ok`: Did it work? `true` = success, `false` = error
- `payload`: The result data (only if `ok: true`)
- `error`: Error details (only if `ok: false`)

**Like:** Getting a letter back with your tracking number, a checkmark (or X), and the answer.

#### Event Frame (Server → Client, unsolicited)
```json
{
  "type": "event",
  "event": "chat.event",
  "payload": { "runId": "run-456", "text": "Here's my response..." },
  "seq": 42
}
```

- Not a response to a request — the server sends these proactively
- `event`: What happened (like `tick`, `chat.event`, `node.pair.requested`)
- `seq`: Sequence number so clients can detect missed events
- Events can be dropped for slow clients (marked with `dropIfSlow`)

**Like:** A news ticker — the server announces things as they happen, whether you asked or not.

### The Connect Handshake

When a client first connects via WebSocket, there's a careful dance:

```
Client                              Gateway
  |                                    |
  |--- WebSocket upgrade request ----->|
  |                                    |
  |<--- "connect.challenge" event -----|  (sends a random nonce)
  |                                    |
  |--- connect request --------------->|
  |    (credentials + device info)     |
  |                                    |
  |    [Server validates everything]   |
  |                                    |
  |<--- "hello-ok" response ----------|
  |    (token, role, features,         |
  |     available methods, snapshot)   |
  |                                    |
  |--- normal requests/events -------->|
```

The `hello-ok` response tells the client everything it needs to know:
- What methods are available
- What role/permissions they have
- A device token for future connections
- The current state snapshot (so the UI can render immediately)
- The canvas host URL (for interactive UI)
- Policy limits (max payload size, heartbeat interval)

### Schema Validation with AJV

Every frame is validated against a strict JSON schema using the AJV library. This means:
- If you send a request with missing fields → rejected immediately
- If you send the wrong data type → rejected
- If you include unknown fields → they're kept but flagged

There are 100+ compiled validators — one for each method's params. Validation errors are formatted into human-readable messages like: `"at .auth: unexpected property 'unknown'"`.

**Why?** Because WebSocket messages are just raw text. Without validation, a buggy client could crash the server by sending garbage data. The schema is the bouncer at the door.

---

## Part 3: Authentication (Who Are You?)

The gateway supports **7 different ways** to prove your identity. They're tried in priority order — the first one that works wins.

### Method 1: Token Authentication (Most Common)

The simplest method. You share a secret string (like a password for an API).

```json
{ "auth": { "token": "my-secret-token-12345" } }
```

The server compares your token to the configured one using **constant-time comparison** (`safeEqualSecret()`). This prevents timing attacks — where an attacker measures how long the comparison takes to guess the token character by character.

**Like:** A secret knock on the door. If you knock the right pattern, you're in.

### Method 2: Password Authentication

Similar to token, but designed for interactive scenarios (web UI, browser).

```json
{ "auth": { "password": "my-password" } }
```

Also uses constant-time comparison. Rate-limited per IP address.

### Method 3: Device Pairing (Mobile/Desktop Apps)

For companion apps (iPhone, macOS menu bar), the device proves its identity using cryptography:

1. The device has an **Ed25519 key pair** (public + private key)
2. The server sends a random challenge (nonce)
3. The device signs the challenge with its private key
4. The server verifies the signature with the device's public key

```json
{
  "device": {
    "id": "sha256-of-public-key",
    "publicKey": "base64url-encoded-key",
    "signature": "base64url-signed-challenge",
    "signedAt": 1710000000000,
    "nonce": "server-provided-random-string"
  }
}
```

The signature includes: device ID, client ID, client mode, role, scopes, timestamp, and nonce. The timestamp must be within 2 minutes of server time (prevents replay attacks).

**Like:** A fingerprint scanner. Only YOUR finger opens the lock, and the scanner changes its challenge every time so nobody can replay a recording.

### Method 4: Bootstrap Token (First-Time Setup)

When you pair a new device for the first time, it uses a one-time bootstrap token:

```json
{
  "auth": { "bootstrapToken": "one-time-setup-code" },
  "device": { ... }
}
```

The bootstrap token is typically displayed as a QR code on the server. You scan it with your phone, the phone sends it with its device identity, and the server pairs them permanently.

**Like:** A golden ticket. Use it once to get through the door, then you get a permanent membership card (device token).

### Method 5: Trusted Proxy

For enterprise setups where a reverse proxy (like nginx) handles authentication:

- The proxy authenticates the user (via SSO, OIDC, SAML, etc.)
- The proxy adds a header like `X-Forwarded-User: jane@company.com`
- The gateway trusts the proxy (verified by IP address) and accepts the user

No client-side credential needed — the proxy vouches for you.

**Like:** A security guard at the office building who checks your badge. Once past the guard, the receptionist trusts you automatically.

### Method 6: Tailscale Identity

If you use Tailscale (a VPN that creates a private network):

- Tailscale's proxy adds `Tailscale-User-Login` headers
- The gateway detects the Tailscale headers on loopback connections
- Your Tailscale identity IS your auth — no password needed

**Like:** Being on a private family Wi-Fi. Everyone on the network is trusted by definition.

### Method 7: No Authentication

For development/testing only. All connections are allowed.

### Rate Limiting (Brute-Force Protection)

Authentication failures are rate-limited per IP address:
- **Two separate buckets**: one for token/password failures, one for device token failures
- Browser clients on loopback use a synthetic IP (`198.18.0.1`) to prevent abuse
- After too many failures, the server responds with `429 Too Many Requests` and a `retryAfterMs` delay
- Rate counters reset on successful auth

### Role-Based Access Control (RBAC)

After authenticating, you're assigned a **role** that determines what you can do:

| Role | Who | Can Do |
|------|-----|--------|
| **Operator** | CLI, Control UI, web apps | Everything — full admin, read, write, approvals, pairing |
| **Node** | Backend worker nodes | Only node-specific operations (invoke, events, pending work) |
| **Guest** | Web widgets, anonymous users | Read-only, limited operations |

Within the Operator role, there are **5 permission scopes**:

| Scope | Examples |
|-------|---------|
| `operator.admin` | Logout channels, create agents, install skills, manage secrets, reset sessions |
| `operator.read` | Health checks, list sessions, view logs, model catalog, usage stats |
| `operator.write` | Send messages, chat, control TTS, invoke nodes, browser requests |
| `operator.approvals` | Request/resolve execution approvals |
| `operator.pairing` | Pair devices and nodes, manage device tokens |

The `admin` scope is a **master key** — it bypasses all method-specific checks.

---

## Part 4: HTTP Endpoints (The Web-Facing Side)

The gateway isn't just WebSocket — it also serves HTTP endpoints. Requests flow through a **staged pipeline** of 14 handlers, checked in order:

```
Request arrives
  → hooks          (/hooks/*)
  → tools-invoke   (tool invocation)
  → slack          (Slack integrations)
  → openresponses  (/v1/responses)
  → openai         (/v1/chat/completions)
  → canvas-auth    (canvas authorization)
  → a2ui           (canvas A2UI WebSocket)
  → canvas-http    (canvas HTTP routes)
  → plugin-auth    (plugin route auth)
  → plugin-http    (plugin HTTP routes)
  → control-ui     (web UI avatar + SPA)
  → gateway-probes (/health, /ready)
  → 404 fallback
```

First handler that returns `true` (handled) stops the pipeline.

### Health Probes

```
GET /health   → {"ok": true, "status": "live"}
GET /healthz  → Same (Kubernetes alias)
GET /ready    → {"ready": true, "failing": [], "uptimeMs": 12345}
GET /readyz   → Same (Kubernetes alias)
```

Used by Docker, Kubernetes, and monitoring tools to check if the gateway is alive and ready.

### OpenAI-Compatible Chat Completions

```
POST /v1/chat/completions
```

This makes the gateway act as a **drop-in replacement for the OpenAI API**. Any tool that works with ChatGPT can point to your OpenClaw gateway instead.

**Request format** (same as OpenAI):
```json
{
  "model": "openclaw",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What's the weather?"}
  ],
  "stream": true
}
```

**Features:**
- Supports both streaming (SSE) and non-streaming responses
- Handles image attachments via `image_url` parts (base64 or HTTP URLs)
- Converts OpenAI message format to OpenClaw's internal format
- Returns responses wrapped in OpenAI's format:

```json
{
  "id": "chatcmpl_abc123",
  "object": "chat.completion",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "The weather is..."},
    "finish_reason": "stop"
  }]
}
```

**Streaming** uses Server-Sent Events (SSE):
```
data: {"id":"...","choices":[{"delta":{"content":"The"}}]}
data: {"id":"...","choices":[{"delta":{"content":" weather"}}]}
data: {"id":"...","choices":[{"delta":{"content":" is..."}}]}
data: [DONE]
```

### OpenResponses Endpoint

```
POST /v1/responses
```

A second OpenAI-compatible endpoint using the newer "Responses" API format.

### Webhook Hooks

```
POST /hooks/wake    → Wake the agent
POST /hooks/agent   → Send a message to a specific agent
POST /hooks/{mapped} → Custom mapped hooks
```

Webhooks let external services (GitHub, Zapier, etc.) trigger actions. Features include:
- Bearer token authentication
- Rate limiting (20 attempts/60 seconds)
- Idempotency keys (prevent duplicate execution)
- Max body size: 20MB
- Deduplication via SHA256 fingerprinting

---

## Part 5: The Chat Flow (The Heart of Everything)

The `chat.send` handler in `src/gateway/server-methods/chat.ts` (1,500+ lines) is the most complex and important handler. Here's what happens when you send a message:

### Step 1: Validation Gauntlet

Your message goes through **12 validation checks**:

1. Schema validation (correct field types)
2. System provenance check (only authorized bridge clients)
3. Message sanitization (Unicode NFC normalization, control character stripping)
4. System receipt validation
5. Stop command detection (messages starting with `/` may be commands)
6. Attachment normalization
7. Empty message check
8. Attachment parsing with image extraction
9. Session key resolution and canonicalization
10. Timeout resolution
11. Send policy check (is this session allowed to receive messages?)
12. Idempotency check (already processed this exact request? Return cached response)

### Step 2: Abort Controller Setup

```typescript
context.chatAbortControllers.set(clientRunId, {
  controller: new AbortController(),
  sessionId: entry?.sessionId ?? clientRunId,
  sessionKey: rawSessionKey,
  startedAtMs: now,
  expiresAtMs: resolveChatRunExpiresAtMs({now, timeoutMs}),
  ownerConnId, ownerDeviceId
});
```

Every chat run gets an `AbortController`. This is like a "cancel button" — if you want to stop the AI mid-response, the abort controller signals all downstream operations to stop.

### Step 3: Immediate Acknowledgment

The server immediately responds:
```json
{"runId": "run-456", "status": "started"}
```

This happens BEFORE the AI starts thinking. The client knows its message was received and can show a "thinking..." indicator.

### Step 4: Message Construction

The raw message is enriched into multiple formats:
- **Body**: The raw user message (for UI display)
- **BodyForAgent**: Timestamped version (so the AI knows when you sent it)
- **BodyForCommands**: May include `/think <thinking>` prefix if thinking mode is requested
- **Delivery context**: Which channel, which account, which thread

### Step 5: Fire-and-Forget Dispatch

```typescript
void dispatchInboundMessage({ ctx, cfg, dispatcher, replyOptions })
  .then(() => { /* success handling */ })
  .catch((err) => { /* error broadcast */ })
  .finally(() => { context.chatAbortControllers.delete(clientRunId); });
```

The message is dispatched **asynchronously** — the server doesn't wait for the AI to finish. Instead, it:
1. Fires the message into the agent pipeline
2. The agent streams responses back via **broadcast events**
3. Each text chunk becomes a `chat.event` sent to all connected clients

### Step 6: Streaming Response

As the AI generates text, the gateway broadcasts events:
```json
{
  "type": "event",
  "event": "chat.event",
  "payload": {
    "runId": "run-456",
    "sessionKey": "agent:main:main",
    "seq": 1,
    "state": "delta",
    "text": "Here's what I found..."
  }
}
```

When the AI finishes:
```json
{
  "state": "final",
  "message": { "role": "assistant", "content": "Here's what I found..." }
}
```

### Step 7: Transcript Persistence

The complete exchange is appended to the session's JSONL transcript file:
```
~/.openclaw/agents/main/sessions/agent-main-main.jsonl
```

Each line is a JSON object representing one message.

### Chat Abort (Cancellation)

Two abort modes:

1. **Abort single run**: Cancel a specific AI response by `runId`
2. **Abort all runs for session**: Cancel everything in a session (the "stop" button)

Abort authorization:
- **Admin scope**: Can abort anything
- **Device ID match**: Can abort your own device's runs
- **Connection ID match**: Can abort your own connection's runs

Partial text is preserved — if the AI wrote half an answer before you cancelled, that partial response is saved to the transcript.

### Chat History

```
Method: chat.history
Returns: Last N messages (default 200, max 1000)
```

Messages are sanitized for display:
- Text truncated to 12K characters
- Image data replaced with placeholders
- Messages over 128KB replaced with "[message too large]"
- Internal metadata stripped
- Silent-only replies filtered out
- Total response size capped

---

## Part 6: Runtime State (The Server's Working Memory)

The gateway keeps all its live state in a single `RuntimeState` object:

```typescript
{
  canvasHost: CanvasHostHandler | null   // The embedded browser canvas server
  httpServer: HttpServer                 // Main HTTP/HTTPS server
  wss: WebSocketServer                   // WebSocket server
  clients: Set<GatewayWsClient>         // All connected WebSocket clients
  broadcast: GatewayBroadcastFn          // Function to send events to all clients
  agentRunSeq: Map<string, number>       // Sequence numbers per agent run (for ordering)
  dedupe: Map<string, DedupeEntry>       // Request deduplication cache
  chatRunState: {
    chatAbortControllers: Map            // AbortControllers for active chat runs
    chatRunBuffers: Map                  // Partial text buffers for streaming
    chatDeltaSentAt: Map                 // Timestamps of last streamed chunk
    chatAbortedRuns: Map                 // Recently aborted run IDs
  }
  chatAbortControllers: Map              // Direct access to abort controllers
  nodeRegistry: NodeRegistry             // All paired/connected nodes
  channelManager: ChannelManager         // All active channel connections
  cronService: CronService               // Scheduled task runner
  healthMonitor: HealthMonitor           // Periodic health checker
}
```

**Like:** The operator's desk — with a list of everyone who's connected, a notepad of ongoing conversations, a switchboard of active channels, and a schedule of upcoming tasks.

---

## Part 7: Broadcasting (The PA System)

Broadcasting is how the gateway pushes events to all connected clients simultaneously. Think of it as a PA system — when something happens, everyone hears about it.

### Event Types

| Event | When |
|-------|------|
| `chat.event` | AI generates text, completes, or errors |
| `agent` | Agent starts/stops/changes state |
| `presence` | Client connects/disconnects |
| `health` | Channel health changes |
| `tick` | Periodic heartbeat (every ~30 seconds) |
| `node.pair.requested` | A new device wants to pair |
| `exec.approval.requested` | Agent needs human approval |
| `shutdown` | Server is shutting down |

### Broadcast Mechanics

```typescript
broadcast(event: string, payload: unknown, options?: {
  dropIfSlow?: boolean      // Skip slow clients
  stateVersion?: object     // Include state version hints
})
```

Each event is serialized once and sent to every client in the `clients` Set. For slow clients (whose WebSocket buffer is backing up), events marked `dropIfSlow: true` are skipped to prevent memory exhaustion.

State version hints let clients detect when they've missed events and need to resync.

---

## Part 8: Channel Manager (Keeping the Lines Open)

The Channel Manager (`src/gateway/server-channels.ts`) handles the lifecycle of all messaging channels — starting them, monitoring them, and restarting them when they fail.

### Lifecycle for Each Channel

```
Configure → Start → Monitor → (Failure?) → Backoff → Restart
                      ↑                                  |
                      +----------------------------------+
```

### Exponential Backoff Restart

When a channel crashes (Discord disconnects, WhatsApp session expires, etc.):

```
Attempt 1: Wait 5 seconds  → Restart
Attempt 2: Wait 10 seconds → Restart
Attempt 3: Wait 20 seconds → Restart
Attempt 4: Wait 40 seconds → Restart
...
Max wait: 5 minutes → Restart
```

The wait time doubles each attempt (exponential backoff) up to a maximum of 5 minutes. This prevents hammering a broken service while still recovering automatically.

### Health Monitoring

Each channel reports its health via `probeAccount()`:
- **connected**: Everything working
- **disconnected**: Not connected but configured
- **error**: Something is wrong
- **not_configured**: Channel not set up

Health changes are broadcast to all clients as `health` events, so the UI can show red/green indicators.

---

## Part 9: Discovery (Finding the Gateway)

How do your devices find the gateway on the network?

### mDNS / Bonjour (Local Network)

The gateway advertises itself on your local network using Bonjour (Apple's zero-config networking):

**Service name:** `_openclaw-gw._tcp.local`

**TXT records advertised:**
```
role=gateway
gatewayPort=18789
lanHost=openclaw.local
displayName=My MacBook
canvasPort=18793
```

**On macOS:** Uses `dns-sd -B _openclaw-gw._tcp local` to browse for instances
**On Linux:** Uses `avahi-browse -rt _openclaw-gw._tcp` to discover

**Modes:**
- **minimal** (default): Advertises basic info only (security-conscious)
- **full**: Also includes SSH port and CLI path (for troubleshooting)
- **off**: No mDNS advertising

A 60-second watchdog checks if the service is still advertised and re-advertises if needed.

### Tailscale Discovery (VPN/Remote)

If you use Tailscale, the gateway advertises via DNS on your Tailnet:

1. Resolves your Tailnet hostname (e.g., `macbook.tail1234.ts.net`)
2. Writes a DNS zone file to `~/.openclaw/dns/`
3. Other devices query Tailnet DNS for `_openclaw-gw._tcp.<domain>`
4. SRV and TXT records point them to the gateway

### Wide-Area DNS-SD (Global)

For finding gateways across the internet:

```
Zone file format:
$ORIGIN example.ts.net
_openclaw-gw._tcp IN PTR my-instance._openclaw-gw._tcp
my-instance._openclaw-gw._tcp IN SRV 0 0 18789 macbook.example.ts.net
my-instance._openclaw-gw._tcp IN TXT "displayName=My MacBook" "gatewayPort=18789"
```

The zone file uses a YYYYMMDDNN serial format, and content-hash deduplication skips writes if nothing changed.

---

## Part 10: Nodes (Remote Workers)

Nodes are external devices that can execute commands on behalf of the gateway — like your iPhone's camera, a remote server's GPU, or an Android phone's sensors.

### Node Pairing Flow

```
1. Node sends: "node.pair.request" (I want to join!)
2. Gateway broadcasts: "node.pair.requested" (Hey operator, someone wants in)
3. Operator clicks: "Approve" or "Reject"
4. Gateway sends: "node.pair.resolved" (Welcome! / Go away.)
5. Node stores: Authentication token for future connections
```

### Node Invocation

Once paired, the gateway can ask nodes to do things:

```typescript
nodeRegistry.invoke({
  nodeId: "iphone-jane",
  command: "camera.capture",
  params: { resolution: "1080p" },
  timeoutMs: 30000
})
```

The node receives the command via WebSocket, executes it (takes a photo), and sends back the result.

### iOS Background Challenges

iPhones can't run background tasks easily. When a node command fails with `NODE_BACKGROUND_UNAVAILABLE`:
1. The command is queued in a **pending action queue** (max 64 items, 10-minute TTL)
2. An APNS push notification wakes the iPhone app
3. The app comes to foreground, pulls pending actions with `node.pending.pull`
4. Executes each action and acknowledges with `node.pending.ack`

Wake notifications are throttled (15 seconds between background wakes, 10 minutes between foreground nudges).

---

## Part 11: The Gateway Client (Connecting TO the Gateway)

The `GatewayClient` class (`src/gateway/client.ts`) is the other side — code that connects TO the gateway.

### Connection Flow

1. Create WebSocket connection to `ws://127.0.0.1:18789` (or `wss://` for TLS)
2. Security check: Blocks plaintext `ws://` to non-loopback addresses (prevents MITM attacks)
3. Receive `connect.challenge` event with nonce
4. Sign the challenge with device identity (Ed25519)
5. Send `connect` request with credentials
6. Receive `hello-ok` response with token, role, scopes
7. Store device token for future connections

### Reconnection with Backoff

If the connection drops:
- Exponential backoff: 1s → 2s → 4s → ... → 30s max
- Certain auth failures pause reconnection entirely (wrong password, rate limited, pairing required)

### Tick Watchdog

The server sends `tick` events at regular intervals (default 30s). The client watches for these:
- If no tick for 2x the interval → connection is stale → close and reconnect

### Request-Response Pattern

```typescript
const result = await client.request("chat.send", {
  message: "Hello!",
  sessionKey: "agent:main:main"
});
```

Each request gets a UUID. The client tracks pending requests in a Map, and resolves/rejects the Promise when the matching response arrives.

---

## Part 12: Server Methods (The Complete Menu)

The gateway has **69+ method files** organized into categories:

| Category | Methods | Purpose |
|----------|---------|---------|
| **Chat** | `chat.send`, `chat.abort`, `chat.history`, `chat.inject` | Sending/receiving messages |
| **Config** | `config.get`, `config.set`, `config.patch`, `config.apply` | Reading/writing config |
| **Channels** | `channels.status`, `channels.logout` | Channel management |
| **Sessions** | `sessions.list`, `sessions.get`, `sessions.patch`, `sessions.reset`, `sessions.delete` | Session management |
| **Agents** | `agents.list`, `agents.create`, `agents.update`, `agents.delete` | Agent CRUD |
| **Nodes** | `node.list`, `node.pair.*`, `node.invoke`, `node.describe` | Node management |
| **Cron** | `cron.list`, `cron.add`, `cron.update`, `cron.remove`, `cron.run` | Scheduled tasks |
| **Models** | `models.list` | Available AI models |
| **Skills** | `skills.status`, `skills.install`, `skills.update` | Skill management |
| **TTS** | `tts.status`, `tts.enable`, `tts.disable`, `tts.convert` | Text-to-speech |
| **System** | `health`, `logs.tail`, `usage.status`, `usage.cost` | System info |
| **Devices** | `device.pair.*`, `device.token.*` | Device management |
| **Exec** | `exec.approval.request`, `exec.approval.resolve` | Execution approvals |
| **Web** | `web.login.start`, `web.login.wait` | Web auth flows |
| **Browser** | `browser.request` | Browser automation |
| **Wizard** | `wizard.*` | Onboarding flows |

### Method Dispatch

When a request arrives:
1. **Authorization check**: Does this role + scopes allow this method?
2. **Rate limiting**: For control-plane writes (`config.apply`, `config.patch`, `update.run`), max 3 requests per 60 seconds per client
3. **Handler lookup**: Find the handler function for this method
4. **Invocation**: Call the handler within a plugin scope (so plugins can hook into it)

Unknown methods → error response. Unauthorized methods → error with missing scope info.

---

## How Layer 2 Evolved: The Git History

### Phase 0: The Predecessor — Control Channel (December 8, 2025)

Before the gateway existed, there was a simple **control channel** at `src/infra/control-channel.ts`. It was a basic heartbeat relay — nothing more than a way for the server to announce "I'm alive" to connected clients. It lasted barely a day.

### Phase 1: The Big Bang (December 9, 2025)

On a single afternoon, the gateway was created from scratch in commit `b2e7fb01a`:

- +5,200 lines, -2,486 lines
- Created: `server.ts`, `client.ts`, `protocol/index.ts`, `protocol/schema.ts`
- Deleted: `control-channel.ts` (the predecessor)
- Added: protocol code generator, server tests, Swift macOS client code, architecture docs

20 minutes later, commit `172ce6c79` refined the protocol with discriminated unions and generated a JSON schema (1,162 lines).

By the end of that day, the gateway had a typed protocol, a client library, test coverage, and provider routing. **The gateway was born fully formed in an afternoon.**

### Phase 2: Protocol v2 and Early Features (December 10-23, 2025)

- **Dec 12**: Protocol v2 — the handshake changed from ad-hoc to a formal `req:connect` message. Only 3 days after creation.
- **Dec 13**: Bonjour/mDNS discovery added for LAN device finding
- **Dec 13-14**: Webchat integration
- **Dec 21**: Auth module extracted from server.ts (200 lines). Initial auth: Tailscale + PAM
- **Dec 23**: PAM dropped after only 2 days. Replaced by password auth for Tailscale Funnel.

### Phase 3: The Great Decomposition (January 3-4, 2026)

The monolithic `server.ts` had grown too large:

- **Jan 3**: First split — `server-http.ts` (HTTP endpoints) and `server-providers.ts` (channel lifecycle) extracted
- **Jan 3**: Method dispatcher extracted into `server-methods.ts` (3,269 lines!)
- **Jan 4**: The giant `server-methods.ts` was decomposed into **19 individual files**: `chat.ts`, `sessions.ts`, `nodes.ts`, `agent.ts`, `cron.ts`, `config.ts`, etc.

### Phase 4: OpenAI-Compatible API (January 10-19, 2026)

- **Jan 10**: `/v1/chat/completions` added — making the gateway an OpenAI-compatible API server
- **Jan 10 (30 min later)**: Immediately disabled by default (security concern?)
- **Jan 10 (1 hour later)**: Made configurable via config toggle
- **Jan 19**: `/v1/responses` endpoint added (OpenAI Responses API)
- **Jan 20**: Community contribution expanded `/v1/responses` input support

### Phase 5: Plugin Architecture + Channels Rename (January 11-15, 2026)

- **Jan 11**: Providers refactored into plugin architecture
- **Jan 13**: **Breaking rename**: "providers" → "channels" throughout the codebase. `server-providers.ts` → `server-channels.ts`.
- **Jan 15**: Channel plugins and HTTP hooks added

### Phase 6: Second Great Decomposition (January 14, 2026)

Commit `d19bc1562` created **25 new files** from the server:
- `server-broadcast.ts` — event broadcasting
- `server-runtime-state.ts` — runtime state
- `server-close.ts`, `server-cron.ts`, `server-discovery-runtime.ts`, `server-lanes.ts`, `server-maintenance.ts`, `server-plugins.ts`, `server-reload-handlers.ts`, etc.

This was when the server truly became modular.

### Phase 7: Device Auth + TLS (January 19-20, 2026)

- **Jan 19**: Unified device authentication + pairing system
- **Jan 19**: Native TLS support added
- **Jan 19**: Older "bridge" protocol removed
- **Jan 20**: Device token auth + CLI management

### Phase 8: Project Renames (January 27-30, 2026)

- **Jan 27**: Renamed to "moltbot"
- **Jan 30**: Final rename to "openclaw"

### Phase 9: Security Hardening (February 2026)

February was dominated by security:

- **Feb 13**: Auth rate-limiting and brute-force protection
- **Feb 14**: Trusted-proxy auth mode for reverse proxies
- **Feb 16-18**: Mesh/DAG orchestration experiment added... and **reverted 2 days later** (failed experiment)
- **Feb 19**: Auth became on-by-default (explicit opt-out required)
- **Feb 19**: Plaintext WebSocket blocked for non-loopback connections
- **Feb 19**: Security headers added (CORS, CSP)
- **Feb 22**: Legacy v1 device-auth handshake removed

### Phase 10: Container-Native + Polish (March 2026)

- **Mar 1**: `healthz`/`readyz` probe endpoints for Kubernetes
- **Mar 6**: Image support in OpenAI chat completions endpoint
- **Mar 6**: Channel-backed readiness probes
- **Mar 9**: Pending node work primitives
- **Mar 12**: Preauth WebSocket handshake limits tightened (latest security hardening)

### Evolution Timeline

| Date | Event | Commits |
|------|-------|---------|
| Dec 8, 2025 | Control channel predecessor | ~3 |
| **Dec 9, 2025** | **Gateway created (Big Bang)** | **5 commits in one afternoon** |
| Dec 12 | Protocol v2 | 1 breaking change |
| Dec 13 | Bonjour discovery | 3 |
| Dec 21-23 | Auth module created, PAM dropped | 4 |
| **Jan 3-4, 2026** | **First decomposition (19 method files)** | ~8 |
| Jan 10 | OpenAI-compatible endpoint | 3 |
| Jan 11-13 | Plugin architecture + channels rename | 5 |
| **Jan 14** | **Second decomposition (25 new files)** | 2 |
| Jan 19-20 | Device auth + TLS + bridge removal | 5 |
| Jan 27-30 | Renames (moltbot → openclaw) | 2 |
| Feb 13-22 | Security hardening wave | ~15 |
| Mar 1-12 | Container support + polish | ~10 |

### Key Takeaways

1. **Born fully formed**: Unlike Layer 1 (which grew from a monolith), the gateway was created as a deliberate, planned system in a single afternoon. It replaced the control channel wholesale.

2. **Two massive decompositions**: Jan 3-4 (monolith → 19 method files) and Jan 14 (server → 25 helper modules). The gateway was growing too fast for a single file.

3. **OpenAI compatibility was a turning point**: Adding `/v1/chat/completions` transformed the gateway from a pure WebSocket RPC server into a dual-protocol (WS + HTTP REST) API server that any OpenAI-compatible tool can use.

4. **Security evolved from "trust everything" to "trust nothing"**: Started with no auth → PAM (dropped in 2 days) → token/password → device pairing → TLS → rate limiting → default-on auth → plaintext blocking. Each step was driven by real security concerns.

5. **One failed experiment**: Mesh/DAG orchestration was added Feb 16 and reverted Feb 18. Not every feature survives.

6. **Extraordinary velocity**: 1,720 commits in 3 months, with 657 in February alone. The gateway was the primary focus of development.

---

## File Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/gateway/server.impl.ts` | ~1070 | 10-phase startup sequence |
| `src/gateway/server-http.ts` | ~992 | HTTP endpoint pipeline (14 stages) |
| `src/gateway/server-methods.ts` | ~158 | RPC method dispatcher |
| `src/gateway/server-methods/chat.ts` | ~1504 | Chat send/abort/history/inject |
| `src/gateway/openai-http.ts` | ~613 | OpenAI-compatible /v1/chat/completions |
| `src/gateway/auth.ts` | ~250+ | 7 auth methods + resolution |
| `src/gateway/method-scopes.ts` | ~134 | RBAC scope → method matrix |
| `src/gateway/server-broadcast.ts` | ~56 | Event broadcasting to all clients |
| `src/gateway/server-runtime-state.ts` | ~150 | Runtime state structure |
| `src/gateway/server-channels.ts` | ~300 | Channel lifecycle + backoff restart |
| `src/gateway/client.ts` | ~724 | WebSocket client + reconnection |
| `src/gateway/protocol/index.ts` | ~200 | Protocol frame schemas + AJV validators |
| `src/gateway/protocol/schema/frames.ts` | varies | Request/Response/Event frame definitions |
| `src/gateway/server/ws-connection.ts` | ~200 | WebSocket connection handler |
| `src/gateway/server/ws-connection/message-handler.ts` | ~700+ | Full handshake flow |
| `src/gateway/server-discovery-runtime.ts` | ~100 | Discovery startup (mDNS/Tailscale/DNS-SD) |
| `src/gateway/server-cron.ts` | ~500+ | Cron/scheduled task service |
| `src/gateway/node-registry.ts` | ~230+ | Node registration and invocation |
| `src/gateway/server-methods/nodes.ts` | ~1200+ | Node pairing, wake, invocation |
| `src/gateway/server-methods/sessions.ts` | ~400+ | Session list/preview/patch/reset |
| `src/gateway/server-methods/config.ts` | ~600+ | Config get/patch/apply/reload |
| `src/infra/bonjour.ts` | ~282 | mDNS/Bonjour service advertiser |
| `src/infra/bonjour-discovery.ts` | ~591 | mDNS/Bonjour service discovery |
| `src/infra/widearea-dns.ts` | ~200 | Wide-area DNS-SD zone generation |
