# Layer 10: Canvas & A2UI — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Canvas & A2UI? (The Simple Version)

Imagine you're texting with a super-smart friend, but sometimes text isn't enough. You want them to show you something visual — a chart, a form to fill out, a photo gallery, or an interactive map. In a normal chat, you'd just get a link to a website. But what if your friend could create a **mini app right inside your conversation** that you could see and interact with?

That's what the Canvas and A2UI systems do:

- **Canvas** is like a tiny web browser built into your chat. The AI can create an HTML page and show it to you — on your Mac, iPhone, Android, or even in Discord. The AI can also take screenshots of it, run JavaScript in it, and update it on the fly.

- **A2UI** (Agent-to-UI) is a special language for describing user interfaces using JSON — no HTML needed. The AI sends a JSON description like "show a button labeled 'Submit' next to a text field", and the client renders it natively. It's like the AI is an architect drawing blueprints, and each platform builds the actual building using its own materials.

Think of Canvas as a **whiteboard** the AI can draw on, and A2UI as a **blueprint language** for building interactive forms and dashboards.

---

## Part 1: The Canvas Host Server

**File:** `src/canvas-host/server.ts` (479 lines)

The Canvas Host is an HTTP server that serves web content the AI creates. It's like a tiny web server running on your machine, just for the AI's visual outputs.

### Configuration

```typescript
type CanvasHostOpts = {
  runtime: RuntimeEnv;
  rootDir?: string;        // Where canvas files live (default: ~/.openclaw/canvas)
  port?: number;           // HTTP port (default: 18793)
  listenHost?: string;     // Bind address (default: 127.0.0.1)
  allowInTests?: boolean;
  liveReload?: boolean;    // Watch files and auto-refresh browser (default: true)
}
```

The canvas host can run either standalone (on port 18793) or integrated into the gateway (on the gateway port 18789). In practice, it's almost always integrated — the gateway handles authentication and proxies requests to the canvas handler.

### The Handler Interface

```typescript
type CanvasHostHandler = {
  rootDir: string;          // Real path to canvas root directory
  basePath: string;         // URL mount point (/__openclaw__/canvas)
  handleHttpRequest(req, res): Promise<boolean>;   // Serve files
  handleUpgrade(req, socket, head): boolean;        // WebSocket upgrade
  close(): Promise<void>;                           // Cleanup
}
```

### How File Serving Works

When a request arrives at `/__openclaw__/canvas/mypage.html`:

1. **Parse URL** and extract the path
2. **Strip base path** (`/__openclaw__/canvas`) to get `mypage.html`
3. **Validate method** — only GET and HEAD allowed
4. **Resolve file safely** — using `resolveFileWithinRoot()` with path traversal protection
5. **Read file content** from disk
6. **Detect MIME type** (html, js, css, png, etc.)
7. **For HTML files**: inject the live reload script
8. **Send response** with proper Content-Type header

If no file exists at the requested path and it's a directory, `index.html` is automatically looked up inside it.

### The Default Test Page

When the canvas root directory is empty, the server creates a default `index.html` — an interactive test page with:
- Action buttons (hello, time, photo, dalek) that send test actions
- Bridge detection showing whether iOS (`window.webkit.messageHandlers`) or Android (`window.openclawCanvasA2UIAction`) bridges are available
- Status display showing action delivery results

This helps developers verify the canvas is working.

### Live Reload

**How it works:**

1. **File watcher**: Uses `chokidar` to monitor the canvas root directory
2. **Debounce**: File change events are debounced by 75ms (12ms in tests) to batch rapid saves
3. **WebSocket server**: Listens at `/__openclaw__/ws` for browser connections
4. **Broadcast**: On file change, sends `"reload"` message to all connected browsers
5. **Browser reload**: Injected script receives the message and calls `location.reload()`

The live reload script is **injected into every HTML file** served by the canvas host. This means you can edit a canvas file in your text editor, save it, and see the changes instantly in the app — no manual refresh needed.

Dotfiles and `node_modules` directories are ignored by the watcher.

---

## Part 2: File Security

**File:** `src/canvas-host/file-resolver.ts` (50 lines)

Security is critical here — the canvas host serves files from disk, so it must prevent attackers from reading arbitrary files on your system.

### Path Traversal Protection

The `resolveFileWithinRoot()` function enforces:

1. **No ".." in path components** — rejects any attempt to escape the root directory
2. **No symbolic links** — symlinks could point outside the root
3. **Directory → index.html** — directories automatically serve their index file
4. **SafeOpenResult wrapper** — prevents information leaks in error responses

Example attacks blocked:
- `/__openclaw__/a2ui/%2e%2e%2fpackage.json` (URL-encoded `../`) → **404**
- `/__openclaw__/a2ui/symlink-to-etc-passwd` → **404**
- `/__openclaw__/canvas/../../../etc/shadow` → **404**

### URL Normalization

```typescript
function normalizeUrlPath(raw: string): string {
  // 1. Decode URI component (%2e → .)
  // 2. Normalize with path.posix.normalize (resolves ../ and ./)
  // 3. Ensure leading /
}
```

---

## Part 3: A2UI — The Declarative UI Language

### What Is A2UI?

**A2UI** stands for **Agent-to-UI** — it's an open standard created by Google (Apache 2.0 license, vendored at `vendor/a2ui/`) that lets AI agents generate rich, interactive user interfaces using declarative JSON.

The key insight is **security through declarative data**: instead of the AI sending HTML/JavaScript (which could be malicious), it sends JSON describing a UI. The client has a pre-approved "catalog" of components and only renders those. The AI can't inject arbitrary code — it can only request components from the trusted catalog.

### Architecture Flow

```
1. AI Agent generates A2UI JSON payload
2. Transport: sent via gateway WebSocket to client
3. Client's A2UI Renderer parses JSON
4. Renderer maps abstract components → concrete native widgets
5. User interacts → actions sent back to AI
```

### The Component Catalog

A2UI v0.8 includes these components:

**Leaf/Display Components:**

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| **Text** | Display text | usage (h1-h5, caption, body), markdown support |
| **Image** | Display images | fit (contain, cover, fill, scale-down), usage hints (icon, avatar, feature, header) |
| **Icon** | Material Design icons | 40+ predefined icons |
| **Video** | Play video | URL source |
| **AudioPlayer** | Play audio | description, title |
| **Divider** | Separator line | orientation (horizontal/vertical), color, thickness |

**Form/Input Components:**

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| **TextField** | Text input | validation regex, label, type hints (shortText, number, date, longText) |
| **CheckBox** | Boolean toggle | label, data-bound value |
| **Slider** | Range picker | min, max, data binding, label |
| **DateTimeInput** | Date/time picker | format configuration |
| **MultipleChoice** | Multi-select | options array, max selections |

**Container/Layout Components:**

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| **Row** | Horizontal flex | distribution (start, center, end, spaceBetween, spaceAround, spaceEvenly), alignment |
| **Column** | Vertical flex | same distribution/alignment options |
| **List** | Scrollable container | direction (vertical/horizontal) |
| **Card** | Elevated box | child content |
| **Tabs** | Tabbed interface | title + content per tab |
| **Modal** | Dialog/popup | entry point component, content component |
| **Button** | Interactive button | action handler |

### JSON Protocol (Server → Client)

The AI sends messages in these formats:

**1. beginRendering** — Initialize a new surface (UI container):
```json
{
  "beginRendering": {
    "surfaceId": "dashboard-1",
    "rootComponentId": "main-column",
    "components": [
      {
        "id": "main-column",
        "component": {
          "Column": {
            "children": { "explicitList": ["header-text", "submit-btn"] }
          }
        }
      },
      {
        "id": "header-text",
        "component": {
          "Text": { "text": { "literalString": "Hello!" }, "usage": "h1" }
        }
      },
      {
        "id": "submit-btn",
        "component": {
          "Button": {
            "text": { "literalString": "Submit" },
            "action": { "name": "submit_form" }
          }
        }
      }
    ]
  }
}
```

**2. surfaceUpdate** — Update existing components
**3. dataModelUpdate** — Modify data model values at a path
**4. deleteSurface** — Remove a surface

### Data Binding

Components can reference data from a shared data model:

```typescript
// Literal values
{ "literalString": "Hello" }
{ "literalNumber": 42 }
{ "literalBoolean": true }

// Data-bound values (resolved at runtime)
{ "path": "/user/name" }           // Absolute path
{ "path": "./item/description" }    // Relative to component's data context
```

Paths support dot notation (`object.property`), bracket notation (`array[0]`), and mixed notation. The model processor normalizes all formats.

### Template-Based Dynamic Lists

Components can generate children dynamically from data:

```json
{
  "children": {
    "template": {
      "componentId": "item-template",
      "dataBinding": "/data/items"
    }
  }
}
```

This creates one instance of `item-template` for each entry in `/data/items`, with each instance's data context bound to its respective item.

### The Action System

When a user taps a button or interacts with a component:

```typescript
interface ClientToServerMessage {
  userAction?: {
    name: string;                    // Action identifier
    surfaceId: string;               // Which surface
    sourceComponentId: string;       // Which component triggered it
    timestamp: number;
    context?: Array<{                // Dynamic context data
      key: string;
      value: { path?: string; literalString?: string; ... }
    }>
  }
}
```

The action flows through the bridge:
1. Component fires action event
2. Bridge script calls `window.openclawSendUserAction()`
3. On iOS: forwards to `window.webkit.messageHandlers.openclawCanvasA2UIAction`
4. On Android: forwards to `window.openclawCanvasA2UIAction.postMessage()`
5. Native code relays to gateway → AI agent
6. Status feedback via `CustomEvent("openclaw:a2ui-action-status")`

### Renderers

A2UI has two renderer implementations in the vendor:

**Lit Renderer** (primary, used by OpenClaw):
- Web Components using the Lit framework
- Each A2UI component maps to a Lit custom element
- CSS-in-JS with structural styles
- Supports theming (component-level, element-level, markdown-level styles)
- Component registry prevents duplicate registration

**Angular Renderer**:
- Directive-based rendering (`ng-container[a2ui-renderer]`)
- Signals-based reactivity
- Dynamic component loading
- Injection-based catalog configuration

### Theming

The theme system styles components at three levels:
- **Component level**: AudioPlayer, Button, Card, Column, etc.
- **Element level**: `a`, `audio`, `body`, `button`, `h1`–`h5`, `input`, `textarea`, etc.
- **Markdown level**: Styled rendering for `p`, `h1`–`h5`, `ul`, `ol`, `li`, `a`, `strong`, `em`

---

## Part 4: The A2UI Handler

**File:** `src/canvas-host/a2ui.ts` (209 lines)

This file bridges the vendored A2UI library into the canvas host.

### Key Constants

```typescript
A2UI_PATH = "/__openclaw__/a2ui"           // A2UI asset URL prefix
CANVAS_HOST_PATH = "/__openclaw__/canvas"   // Canvas host URL prefix
CANVAS_WS_PATH = "/__openclaw__/ws"         // WebSocket path
CANVAS_CAPABILITY_QUERY_PARAM = "oc_cap"    // Capability token query param
```

### A2UI Root Resolution

`resolveA2uiRoot()` searches for the bundled A2UI assets in order:
1. Source build location (`src/canvas-host/a2ui`)
2. Multiple dist locations
3. `process.argv[1]` entry directory fallbacks
4. Repository root fallback

It validates by checking for `index.html` and `a2ui.bundle.js`. The result is cached, with a 10-second negative cache for "not found" to avoid repeated filesystem scans.

### Bridge Injection

`injectCanvasLiveReload()` injects a script into every HTML page that:

1. **Detects native bridges**: iOS WebKit message handler or Android bridge object
2. **Defines `window.OpenClaw.postMessage()`**: Universal messaging function
3. **Defines `window.openclawSendUserAction()`**: Action dispatch function
4. **Opens WebSocket** to `ws://host/__openclaw__/ws?oc_cap={capability}` for live reload
5. **Handles reload messages**: Calls `location.reload()` on "reload" events

### HTTP Request Handling

`handleA2uiHttpRequest()` serves A2UI assets at `/__openclaw__/a2ui/*`:
1. Check if path matches the A2UI prefix
2. Resolve A2UI root directory
3. Extract relative path
4. Use safe file resolution (same traversal protection as canvas)
5. Detect MIME type and serve file
6. Inject live reload for HTML files

---

## Part 5: Canvas Security — The Capability System

### The Problem

The canvas host serves web content that runs in a WebView on native apps. But the gateway is accessible from the network (especially with Tailscale). Without authentication, anyone who can reach the gateway could browse canvas files, which might contain sensitive data.

### The Solution: Capability Tokens

**File:** `src/gateway/canvas-capability.ts` (88 lines)

Each connected "node" (native app) gets a **capability token** — a random 18-byte base64url string that acts as a session key for canvas access.

```typescript
function mintCanvasCapabilityToken(): string {
  return randomBytes(18).toString("base64url");
}
```

**Scoped URLs**: The capability is embedded in the URL path:
```
/__openclaw__/cap/{capability}/__openclaw__/canvas/index.html
```

This means the URL itself carries the authorization — no cookies or headers needed. The token expires after 10 minutes (`CANVAS_CAPABILITY_TTL_MS = 10 * 60_000`) with a sliding window (refreshed on each valid request).

### Authorization Flow

**File:** `src/gateway/server/http-auth.ts` (117 lines)

When a canvas request arrives:

1. **Check if it's a canvas path** (`isCanvasPath()`)
2. **Extract capability** from scoped URL path or `?oc_cap=` query param
3. **Authorization checks** (in order):
   - Reject if the scoped path is malformed
   - Accept if it's a local/direct request
   - Try bearer token auth
   - Try node capability match — check if any connected WebSocket client has this capability and it hasn't expired
4. **Deny** if none of the above succeed (401 response)

### Capability Refresh

Nodes can refresh their expired capabilities via `node.canvas.capability.refresh`:
1. Mint a new token
2. Build a new scoped URL
3. Update the client record
4. Return the scoped URL to the node

This is used by Android when re-opening the canvas after the app was backgrounded.

### Canvas URL Resolution

**File:** `src/infra/canvas-host-url.ts` (93 lines)

The gateway resolves the canvas host URL for each connected node:

**Host resolution priority:**
1. `hostOverride` (if non-loopback)
2. `requestHost` hostname from the HTTP upgrade request
3. `localAddress` of the connection

**Special cases:**
- IPv6 addresses wrapped in brackets (`[::1]:port`)
- HTTPS scheme → default port 443
- HTTP scheme → default port 80
- Gateway port 18789 as the primary canvas port (since canvas is integrated into gateway)

---

## Part 6: Gateway Integration

### Startup

When the gateway starts (`server-runtime-state.ts`):
1. If `canvasHostEnabled` in config, create the canvas handler
2. Mount it at `/__openclaw__/canvas/`
3. Log: `canvas host mounted at http://127.0.0.1:18789/__openclaw__/canvas/ (root ~/.openclaw/canvas)`

### HTTP Request Pipeline

In the gateway HTTP handler (`server-http.ts`), canvas requests go through:

```
Stage 1: canvas-auth
  → Is this a canvas path?
  → Authorize via capability/bearer/local
  → Reject with 401 if unauthorized

Stage 2: a2ui
  → handleA2uiHttpRequest() for /__openclaw__/a2ui/* paths

Stage 3: canvas-http
  → canvasHost.handleHttpRequest() for /__openclaw__/canvas/* paths
```

### WebSocket Upgrade

For live reload WebSocket connections at `/__openclaw__/ws`:
1. Extract capability from query params
2. Authorize the request
3. If authorized, pass to `canvasHost.handleUpgrade()`

### Node Connection Setup

When a native app connects to the gateway WebSocket:
1. Resolve the canvas host URL for this client
2. Mint a capability token
3. Build a scoped canvas URL
4. Store on the client: `canvasHostUrl`, `canvasCapability`, `canvasCapabilityExpiresAtMs`
5. Send the scoped URL in the hello-ok response

---

## Part 7: The Canvas Agent Tool

**File:** `src/agents/tools/canvas-tool.ts` (215 lines)

The AI interacts with the canvas through these tool actions:

| Action | Description |
|--------|-------------|
| **present** | Show canvas at a target URL with optional placement |
| **hide** | Hide the canvas |
| **navigate** | Load a new URL in the canvas |
| **eval** | Execute JavaScript in the canvas and return the result |
| **snapshot** | Capture the canvas as an image (PNG/JPEG) |
| **a2ui_push** | Push A2UI JSON-L data to update the UI |
| **a2ui_reset** | Reset the A2UI surface to blank |

**Snapshot flow:**
1. AI calls the `snapshot` action
2. Gateway invokes `canvas.snapshot` on the connected node
3. Node captures the WebView as a base64 image
4. Gateway writes to a temp file
5. Returns image data to the AI (which can then analyze it)

**A2UI push flow:**
1. AI generates A2UI JSON describing a UI
2. Calls `a2ui_push` with the JSON payload
3. Gateway forwards to the node as `canvas.a2ui` command
4. Node's A2UI renderer processes the JSON and renders native components
5. User interacts → actions sent back through the bridge → AI receives them

### Canvas Skill

**File:** `skills/canvas/SKILL.md` (158 lines)

The canvas skill documentation teaches the AI how to use the canvas system, including:
- HTML/CSS/JS canvas creation
- A2UI JSON generation
- Tailscale integration for remote access
- Architecture overview
- Available tool actions

---

## Part 8: Discord A2UI — Activities

Discord has a special feature called **Activities** — interactive embedded experiences within Discord. OpenClaw uses this to render A2UI inside Discord:

**File created by Shadow:** `a2ui-activity.ts` (120 lines)

This creates a Discord Activity that hosts the A2UI renderer, allowing A2UI surfaces to render directly inside Discord channels. The Activity HTML loads the bundled A2UI renderer and connects to the gateway for data.

---

## Part 9: URL Paths and Ports

### Default URL Structure

| Path | Purpose |
|------|---------|
| `/__openclaw__/canvas/` | Canvas file serving |
| `/__openclaw__/a2ui/` | A2UI bundled assets |
| `/__openclaw__/ws` | Live reload WebSocket |
| `/__openclaw__/cap/{token}/...` | Capability-scoped access |

### Ports

| Port | Use |
|------|-----|
| 18789 | Gateway (canvas integrated here) |
| 18793 | Canvas standalone (when not using gateway) |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `OPENCLAW_SKIP_CANVAS_HOST` | Disable canvas host entirely |
| `OPENCLAW_STATE_DIR` | Override state directory (canvas root at `$STATE_DIR/canvas`) |
| `NODE_ENV=test` / `VITEST` | Auto-disable canvas in test environments |

---

## Part 10: Configuration

**In `config.json5`:**

```json5
{
  canvasHost: {
    enabled: true,              // Enable/disable canvas host
    root: "~/.openclaw/canvas", // Canvas file directory
    port: 18793,                // Standalone port (when not on gateway)
    liveReload: true            // File watching + auto-refresh
  }
}
```

---

## Git History Evolution

### Phase 1: Canvas on Native Platforms (Dec 12–14, 2025)

The Canvas system was born as a **native WebView** feature on macOS. On **December 12, 2025**, Peter Steinberger fixed voice overlay z-ordering above the canvas window — implying a canvas WebView already existed in the macOS app.

The next day (**Dec 13**), he created `RootCanvas.swift` for iOS — a full-screen canvas WebView — and wired `node.invoke` commands through the gateway. On **Dec 14**, a massive 2,518-line commit created the entire Android node app with `CanvasController.kt`, establishing canvas on all three platforms in just three days.

At this point, canvas was just a WebView that the server could command — no A2UI, no file serving, no security.

### Phase 2: A2UI Arrives (Dec 17, 2025)

On **December 17**, Peter Steinberger added the entire A2UI vendor library in a single commit (`cdb5ddb2d` — 73,598 lines): the v0.8/v0.9 JSON schemas, the Lit web-component renderer with the full component catalog (audio, button, card, checkbox, column, datetime-input, divider, icon, image, list, modal, multiple-choice, row, slider, surface, tabs, text-field, text, video), the Angular renderer, and `CanvasA2UI` bundle resources for macOS.

This was followed by seven more commits on the same day: fixing A2UI v0.8 rendering, push rendering, action forwarding, click handling, macOS forwarding, multi-display panel anchoring, and canvas toggle behavior. A2UI went from zero to functional in a single day.

### Phase 3: Protocol Rename and Canvas Host (Dec 18, 2025)

December 18 was another massive day with **16+ commits**:

1. **Protocol rename**: All commands renamed from `screen.*` to `canvas.*` across gateway, CLI, iOS, and Android
2. **Canvas Host creation**: `src/canvas-host/server.ts` (247 lines) — an HTTP/WebSocket server for serving canvas files from the filesystem
3. **Action bridge injection**: Bidirectional communication between canvas UI and the gateway/agent
4. **Cross-platform A2UI**: iOS and Android both got `canvas.a2ui` push/reset commands
5. **A2UI bundle sharing**: Moved from per-platform copies to shared ClawdisKit framework, deleting 18,655 lines of duplication
6. **Canvas snapshots**: JPEG/PNG capture support on iOS and Android

### Phase 4: Architecture Consolidation (Dec 19–21, 2025)

The canvas host was quickly refined:
- **Dec 20**: A2UI now served through the gateway rather than a separate endpoint
- **Dec 20**: `a2ui.ts` extracted as its own module
- **Dec 20**: Canvas host consolidated onto the gateway port (eliminating the separate port 18793 for most users)
- **Dec 20**: Better MIME type detection using `file-type` library
- **Dec 21**: CSS custom properties for safe area insets, debug overlay behind feature flag

### Phase 5: Live Reload and Stability (Jan 4–6, 2026)

**January 4**: The live reload feature was added — `chokidar` file watching triggers WebSocket-based browser refresh. This made canvas development much more pleasant: edit a file, save, see changes instantly.

January also saw build pipeline work for A2UI bundling, requiring Bun for TypeScript execution.

### Phase 6: Canvas Skill and Documentation (Jan 18, 2026)

Peter Steinberger created `skills/canvas/SKILL.md` (158 lines) — documentation teaching the AI how to use the canvas system, including HTML/CSS/JS canvas creation, A2UI JSON generation, and Tailscale integration.

### Phase 7: Security Hardening (Feb 2026)

February brought a wave of critical security fixes:

- **Feb 5**: **Coy Geek** reported that the canvas host served files without authentication. **George Pickett** fixed it the same day (#9518) — adding auth requirements for canvas and A2UI routes, plus a 212-line test suite.
- **Feb 13**: **Abdel Fane** added path traversal prevention using `openFileWithinRoot()` (#10525) — closing the directory escape vector
- **Feb 14**: **David Rudduck** changed the default bind from `0.0.0.0` to `127.0.0.1` (#13184) — loopback only
- **Feb 14**: **Yi Liu** restricted canvas IP-based auth to private networks (#14661)
- **Feb 14**: Peter Steinberger extracted `file-resolver.ts` to share safe file resolution
- **Feb 19**: Peter Steinberger added **canvas capability tokens** (`canvas-capability.ts`, 87 lines) — the scoped-URL session token system with 10-minute sliding expiration. 354 lines of auth hardening.
- **Feb 19**: Abdel Fane added baseline security headers to all gateway HTTP responses (#10526)
- **Feb 20**: Shadow restricted canvas file reads by jsonlPath

### Phase 8: Discord A2UI and Android Polish (Feb–Mar 2026)

- **Feb 17**: **Shadow** added A2UI rendering inside Discord Activities — the AI can now show interactive UI directly inside Discord channels
- **Feb 25**: **Ayaan Zaidi** submitted 7 commits in one day polishing Android canvas: TLS URL normalization, screen-tab restore, mobile viewport, narrow-screen A2UI layout overrides, and scoped URL suffix preservation
- **Feb 27**: Ayaan Zaidi added runtime canvas capability refresh for Android
- **Mar 8–10**: **Mariano** and **Nimrod Gutman** added iOS auto-loading of scoped gateway canvas, welcome home canvas, and toolbar refresh
- **Mar 13**: Peter Steinberger added test coverage for JSON file handling and canvas host helpers (most recent canvas commit)

### The Evolution in Four Sentences

Canvas started as a native WebView on macOS/iOS/Android in December 2025, with all three platforms up in just three days. A2UI — Google's declarative UI protocol — was vendored and integrated on December 17, giving the AI the ability to generate rich interactive UIs via JSON. The canvas-host HTTP server was created the next day, evolving from a separate port to an integrated gateway handler with live reload within a week. A major security hardening push in February 2026 added capability tokens, path traversal protection, auth requirements, and loopback-only defaults — transforming canvas from a development convenience into a production-safe feature.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Canvas & A2UI Layer                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                  AI Agent (tools)                               │  │
│  │                                                                 │  │
│  │  canvas.present  │ canvas.navigate │ canvas.eval               │  │
│  │  canvas.snapshot │ canvas.a2ui_push │ canvas.a2ui_reset        │  │
│  │  canvas.hide                                                    │  │
│  └─────────────────────────────┬──────────────────────────────────┘  │
│                                │                                     │
│                                ▼                                     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │               Gateway (port 18789)                              │  │
│  │                                                                 │  │
│  │  HTTP Pipeline:                                                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │  │
│  │  │ canvas-auth   │→│ a2ui handler  │→│ canvas-host handler  │  │  │
│  │  │               │  │              │  │                      │  │  │
│  │  │ Capability    │  │ Serves from  │  │ Serves from          │  │  │
│  │  │ token check   │  │ vendor/a2ui/ │  │ ~/.openclaw/canvas/  │  │  │
│  │  │ 10-min TTL    │  │              │  │                      │  │  │
│  │  │ Sliding window│  │ Path: /a2ui  │  │ Path: /canvas        │  │  │
│  │  └──────────────┘  └──────────────┘  └─────────────────────┘  │  │
│  │                                                                 │  │
│  │  WebSocket:                                                     │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │ /__openclaw__/ws  — Live reload broadcast               │  │  │
│  │  │ chokidar watcher → 75ms debounce → "reload" message     │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  │                                                                 │  │
│  │  Capability URLs:                                               │  │
│  │  /__openclaw__/cap/{18-byte-token}/__openclaw__/canvas/...     │  │
│  │  Token minted on node connect, refreshable via RPC             │  │
│  └───────────────────────────┬────────────────────────────────────┘  │
│                              │                                       │
│            ┌─────────────────┼─────────────────┐                    │
│            ▼                 ▼                  ▼                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐      │
│  │ macOS App    │  │ iOS App      │  │ Android App          │      │
│  │              │  │              │  │                       │      │
│  │ WKWebView    │  │ WKWebView    │  │ Android WebView      │      │
│  │ canvas.swift │  │ canvas.swift │  │ CanvasController.kt  │      │
│  │              │  │              │  │                       │      │
│  │ webkit.msg   │  │ webkit.msg   │  │ openclawCanvas       │      │
│  │ Handlers     │  │ Handlers     │  │ A2UIAction           │      │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘      │
│         │                  │                      │                  │
│         └──────────────────┼──────────────────────┘                  │
│                            ▼                                         │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                   A2UI Renderer (Lit)                           │  │
│  │                                                                 │  │
│  │  Component Catalog:                                             │  │
│  │  ┌────────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌───────┐ ┌──────┐ │  │
│  │  │ Text   │ │Button│ │ Card │ │TextField│ │Slider │ │Image │ │  │
│  │  └────────┘ └──────┘ └──────┘ └────────┘ └───────┘ └──────┘ │  │
│  │  ┌────────┐ ┌──────┐ ┌──────┐ ┌────────┐ ┌───────┐ ┌──────┐ │  │
│  │  │ Row    │ │Column│ │ List │ │CheckBox │ │ Tabs  │ │Modal │ │  │
│  │  └────────┘ └──────┘ └──────┘ └────────┘ └───────┘ └──────┘ │  │
│  │  ┌────────┐ ┌──────┐ ┌────────────┐ ┌───────────────┐       │  │
│  │  │Divider │ │ Icon │ │AudioPlayer │ │MultipleChoice │       │  │
│  │  └────────┘ └──────┘ └────────────┘ └───────────────┘       │  │
│  │                                                                 │  │
│  │  Data Binding: path-based resolution, literals, templates       │  │
│  │  Events: userAction → bridge → gateway → AI agent              │  │
│  │  Themes: component / element / markdown level styling          │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │              Discord Activities (A2UI)                          │  │
│  │              Renders A2UI inside Discord channels               │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Security: Path traversal protection │ Symlink rejection │           │
│  Capability tokens (18-byte random) │ Loopback-only default │        │
│  Private network IP restriction │ No arbitrary code execution        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/canvas-host/server.ts` | 479 | Canvas host HTTP server, live reload, file serving |
| `src/canvas-host/a2ui.ts` | 209 | A2UI asset serving, bridge injection, root resolution |
| `src/canvas-host/file-resolver.ts` | 50 | Safe file resolution with traversal protection |
| `src/canvas-host/server.test.ts` | 314 | Canvas host tests (serving, reload, security) |
| `src/canvas-host/server.state-dir.test.ts` | 28 | State directory default tests |
| `src/gateway/canvas-capability.ts` | 88 | Capability token minting, scoped URLs |
| `src/gateway/server/http-auth.ts` | 117 | Canvas authorization (capability/bearer/local) |
| `src/gateway/server-runtime-state.ts` | 237 | Canvas host initialization on gateway startup |
| `src/gateway/server-http.ts` | 866+ | HTTP pipeline (auth → a2ui → canvas) |
| `src/gateway/server/ws-connection.ts` | 319 | Canvas URL resolution, capability assignment |
| `src/infra/canvas-host-url.ts` | 93 | Canvas URL resolution (host, port, scheme) |
| `src/agents/tools/canvas-tool.ts` | 215 | Agent tool (present, eval, snapshot, a2ui_push) |
| `src/config/types.gateway.ts` | 49 | CanvasHostConfig type definition |
| `vendor/a2ui/README.md` | — | A2UI overview and philosophy |
| `vendor/a2ui/renderers/lit/src/0.8/` | — | Lit renderer (core, types, components, events) |
| `vendor/a2ui/renderers/angular/` | — | Angular renderer |
| `skills/canvas/SKILL.md` | 158 | Canvas skill documentation for the AI |
