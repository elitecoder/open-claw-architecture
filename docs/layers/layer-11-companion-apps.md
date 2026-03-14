# Layer 11: Companion Apps — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Are the Companion Apps? (The Simple Version)

Imagine you have a super-smart robot brain sitting in a box on your desk (that's the OpenClaw gateway). This brain can think, talk to AI models, manage your messages, and run tools. But it has no eyes, no ears, no hands. It can't see what's on your screen, hear your voice, take a photo, or check your location.

That's where the companion apps come in. They're like the robot's **eyes, ears, and hands** on your phone, tablet, or laptop:

- **macOS app**: Lives in your menu bar, lets you talk to the AI with voice wake, shows a canvas panel, and provides a CLI tool
- **iOS app**: Full iPhone/iPad app with camera, location, motion sensors, contacts, Live Activities, widgets, and push notifications
- **Android app**: Similar to iOS but with Kotlin, foreground service, SMS, and Google's APIs

The critical thing to understand: these apps are **thin clients**. They don't run the AI. They don't process messages. They just:

1. **Connect** to the gateway via WebSocket
2. **Expose** platform capabilities (camera, location, mic) the gateway can invoke
3. **Render** UI the gateway sends them (canvas/A2UI)
4. **Relay** user messages to the gateway and display responses

Think of it like a spy movie: the gateway is mission control, and the apps are field agents with special equipment. Mission control tells a field agent "take a photo," the agent takes it and sends it back.

---

## Part 1: The Shared Library — OpenClawKit

**Directory:** `apps/shared/OpenClawKit/`
**Platforms:** iOS 18+, macOS 15+

The shared library prevents code duplication between the Apple apps. It has three modules:

### Module 1: OpenClawProtocol — The Wire Format

Auto-generated Swift types that match the gateway's WebSocket protocol:

```swift
let GATEWAY_PROTOCOL_VERSION = 3

// The first message a client sends
struct ConnectParams: Codable, Sendable {
    let minprotocol: Int, maxprotocol: Int
    let client: [String: AnyCodable]      // Client metadata
    let caps: [String]?                    // Capabilities (camera, location, etc.)
    let commands: [String]?               // Supported invoke commands
    let permissions: [String: AnyCodable]?
    let role: String?                     // "node" or "operator"
    let scopes: [String]?                // Permission scopes
    let device: [String: AnyCodable]?    // Device identity
    let auth: [String: AnyCodable]?      // Auth credentials
}

// Gateway's hello response
struct HelloOk: Codable, Sendable {
    let _protocol: Int
    let server: [String: AnyCodable]
    let features: [String: AnyCodable]
    let snapshot: Snapshot              // Current gateway state
    let canvashosturl: String?          // Canvas URL for this client
    let policy: [String: AnyCodable]    // Policy/permissions
}

// RPC request/response frames
struct RequestFrame: Codable  { let id: String, method: String, params: AnyCodable? }
struct ResponseFrame: Codable { let id: String, ok: Bool, payload: AnyCodable?, error: [String: AnyCodable]? }
struct EventFrame: Codable    { let event: String, payload: AnyCodable?, seq: Int? }
```

### Module 2: OpenClawKit — Core Types & Gateway Client

**Gateway Channel (`GatewayChannel.swift`):**
A generic WebSocket abstraction with:
- `WebSocketTasking` protocol (wraps `URLSessionWebSocketTask`, mockable for tests)
- `GatewayConnectOptions` struct with role, scopes, capabilities, commands, permissions
- Auth sources: `deviceToken`, `sharedToken`, `bootstrapToken`, `password`, `none`

**Gateway Node Session (`GatewayNodeSession.swift`, 536 lines):**
The main connection actor for companion apps:
```swift
public actor GatewayNodeSession {
    func connect(url, token?, bootstrapToken?, password?,
                 connectOptions, onConnected, onDisconnected, onInvoke) async throws

    func currentCanvasHostUrl() -> String?
    func refreshNodeCanvasCapability(timeoutMs: Int) async -> Bool
    func subscribeServerEvents(bufferingNewest: Int) -> AsyncStream<EventFrame>
}
```

The `onInvoke` callback is how the gateway commands the device — it sends a `BridgeInvokeRequest`, and the app returns a `BridgeInvokeResponse`.

**Canvas A2UI Action (`CanvasA2UIAction.swift`, 105 lines):**
Formats A2UI actions as structured agent messages:
```
CANVAS_A2UI action=<name> session=<key> surface=<surfaceId>
  component=<componentId> host=<machineName> instance=<instanceId>
  ctx=<contextJSON> default=update_canvas
```

**Gateway Push (`GatewayPush.swift`):**
```swift
enum GatewayPush: Sendable {
    case snapshot(HelloOk)     // Initial state
    case event(EventFrame)     // Server push events
    case seqGap(expected, received)  // Missed frames
}
```

**Capabilities:**
```swift
enum OpenClawCapability: String, Codable {
    case canvas, browser, camera, screen, voiceWake
    case location, device, watch
    case photos, contacts, calendar, reminders, motion
}
```

**Error Types (`GatewayErrors.swift`, 147 lines):**
- `GatewayConnectAuthError` — auth failures with detail codes (AUTH_REQUIRED, TOKEN_MISMATCH, PAIRING_REQUIRED)
- `GatewayResponseError` — RPC errors with method, code, message
- `GatewayDecodingError` — JSON decode failures

**Other shared types:**
- Command parameter types: `CameraCommands`, `LocationCommands`, `MotionCommands`, `PhotosCommands`, `ContactsCommands`, `CalendarCommands`, `SystemCommands`, `BrowserCommands`, `TalkCommands`
- Platform support: `CameraAuthorization`, `LocationServiceSupport`, `AudioStreamingProtocols`
- Auth & identity: `DeviceAuthPayload`, `DeviceAuthStore`, `DeviceIdentity`, `InstanceIdentity`, `GatewayTLSPinning`
- Talk/TTS: `TalkDirective` (voice params), `TalkPromptBuilder`, `TalkSystemSpeechSynthesizer`

### Module 3: OpenClawChatUI — Shared Chat Components

SwiftUI chat UI shared between macOS and iOS:
- `ChatView` — the main chat interface
- `ChatViewModel` — message state management
- `ChatComposer` — input area with attachments
- `ChatMarkdownRenderer` — renders AI responses with markdown
- `AssistantTextParser` — parses assistant output
- `OpenClawChatTransport` protocol — abstract RPC for sending/receiving messages

---

## Part 2: macOS Menu Bar App

**Directory:** `apps/macos/` (232 Swift files)
**Platform:** macOS 15+

### Architecture

The macOS app is a **menu bar app** — it lives in the menu bar (top-right of your screen) with an animated critter icon. No dock icon by default (configurable).

**Entry Point (`MenuBar.swift`):**
```swift
@main
struct OpenClawApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var delegate
    @State private var state: AppState

    var body: some Scene {
        MenuBarExtra { MenuContent(...) } label: {
            CritterStatusLabel(...)  // Animated icon
        }
        .menuBarExtraStyle(.menu)

        Settings { SettingsRootView(...) }
    }
}
```

**Interactions:**
- **Left-click** the icon → toggle web chat panel
- **Right-click** → show menu with settings, status, quick actions
- **Hover** → HUD overlay for notifications

### App State

```swift
@MainActor @Observable final class AppState {
    var connectionMode: ConnectionMode    // .unconfigured, .local, .remote
    var isPaused: Bool                    // Pause/resume agent
    var launchAtLogin: Bool               // Auto-start
    var swabbleEnabled: Bool              // Voice wake on/off
    var swabbleTriggerWords: [String]     // Wake words
    var voiceWakeMicID: String            // Selected mic
    var voiceWakeLocaleID: String         // Speech lang
    var voicePushToTalkEnabled: Bool      // PTT mode
    var talkEnabled: Bool                 // TTS on/off
    var showDockIcon: Bool                // Show in dock
    var seamColorHex: String?             // UI accent from gateway
}
```

### Gateway Connection

**`GatewayConnection.swift` (800+ lines):**

A shared actor managing the WebSocket connection with:
- **Single shared instance**: `GatewayConnection.shared`
- **Auto-reconnection**: Exponential backoff (150ms, 400ms, 900ms)
- **Push subscription**: `AsyncStream<GatewayPush>` (replaces NotificationCenter)
- **Tailnet fallback**: Falls back from local to Tailscale gateway
- **30+ RPC methods**: agent, status, config, chat, channels, voicewake, cron, skills, health

**`ControlChannel.swift` (425+ lines):**

Higher-level API wrapping GatewayConnection:
```swift
@MainActor @Observable final class ControlChannel {
    enum ConnectionState { case disconnected, connecting, connected, degraded(String) }

    private(set) var state: ConnectionState
    private(set) var lastPingMs: Double?
}
```

Routes events to the right handlers:
- "agent" events → `AgentEventStore`
- "heartbeat" events → NotificationCenter
- Friendly error messages for common failures

### Canvas Manager

**`CanvasManager.swift` (340+ lines):**

Manages canvas WebView windows:
```swift
@MainActor final class CanvasManager {
    func show(sessionKey, path?, placement?) -> String
    func hide(sessionKey)
    func eval(sessionKey, javaScript) -> String
    func snapshot(sessionKey, outPath?) -> String
}
```

Each canvas gets its own `CanvasWindowController` with:
- Session-specific directory (`~/Library/Application Support/OpenClaw/canvas/<sessionKey>/`)
- `WKWebView` with custom URL schemes (`openclaw://`, `openclawlocal://`)
- Injected A2UI action bridge (WKUserScript at document start)
- File watcher for auto-reload
- Panel or fullscreen presentation

### Voice Wake

The macOS app has a sophisticated voice wake system (19 files):

- **`VoiceWakeRuntime.swift`** — On-device speech recognition engine (using Apple's Speech framework via the Swabble library)
- **`VoiceWakeOverlay*.swift`** — Real-time transcription overlay UI
- **`VoiceWakeForwarder.swift`** — Sends transcripts to the AI agent:
  ```
  "User talked via voice recognition on [MachineName] - <transcript>"
  ```
- **`VoiceWakeChime.swift`** — Audio feedback: trigger chime (listening) + send chime (submitted)
- **`VoicePushToTalk.swift`** — Push-to-talk mode
- **`VoiceWakeGlobalSettingsSync.swift`** — Syncs wake words to gateway

### IPC (Inter-Process Communication)

**`OpenClawIPC/IPC.swift` (417 lines):**

Unix domain socket protocol at `~/Library/Application Support/OpenClaw/control.sock`:

```swift
enum Request: Codable {
    case notify(title, body, sound?, priority?)
    case ensurePermissions([Capability], interactive)
    case runShell(command, cwd?, env?, timeout?)
    case agent(message, thinking?, session?, deliver, to?)
    case canvasPresent(session, path?, placement?)
    case canvasEval(session, javaScript)
    case canvasSnapshot(session, outPath?)
    case canvasA2UI(session, command, jsonl?)
    case cameraSnap(facing?, maxWidth?, quality?, outPath?)
    case cameraClip(facing?, duration?, includeAudio, outPath?)
    case screenRecord(screenIndex?, duration?, fps?, includeAudio, outPath?)
    case nodeList, nodeDescribe(nodeId), nodeInvoke(nodeId, command, params?)
    case status, rpcStatus
}
```

This lets the CLI tool (`openclaw-mac`), external scripts, and even the gateway's shell tools communicate with the menu bar app.

### CLI Tool

**`OpenClawMacCLI/EntryPoint.swift` (57 lines):**

```bash
openclaw-mac connect [--url ws://host:port] [--token <token>] [--mode local|remote]
openclaw-mac discover [--timeout ms] [--json] [--include-local]
openclaw-mac wizard [--url ws://host:port] [--token <token>]
```

### Gateway Discovery

Multiple discovery mechanisms:
- **Bonjour/mDNS**: Local network gateway discovery
- **Tailscale Serve**: Auto-discover via Tailscale's serve feature
- **Wide-area**: Manual remote gateway configuration
- **TCP probe**: Direct connection testing

### Other Features

- **Sparkle auto-updater**: Automatic update checks
- **Node pairing approval**: Prompts when new devices try to pair
- **Device pairing approval**: Prompts for new device connections
- **Exec approval prompting**: Shows prompts when the AI wants to run commands
- **Cron settings**: Manage scheduled tasks
- **Channel status**: View connected messaging channels

---

## Part 3: iOS App

**Directory:** `apps/ios/` (32+ directories)
**Platform:** iOS 18+

### Architecture

The iOS app uses SwiftUI with the Observation framework (`@Observable`). It maintains **two parallel gateway connections**:

1. **Node session** (role=node) — device capabilities, node.invoke
2. **Operator session** (role=operator) — chat, config, talk

### Main Model

**`NodeAppModel.swift` (3,020 lines):**

This is the heart of the iOS app — a massive `@MainActor @Observable` class that coordinates everything:
- Gateway connection lifecycle
- Device capability routing (camera, location, motion, SMS, contacts, calendar, reminders)
- Live Activity updates
- Chat session management
- Voice wake coordination
- Talk mode state & TTS playback
- Deep link handling
- Push notification handling
- Background wake triggers

### Platform Capabilities

#### Camera (`CameraController.swift`, 354 lines)
- **snap**: AVFoundation → JPEG/PNG with quality control, max width 1600px, device selection
- **clip**: Video capture → MOV → MP4 export (medium quality), 250ms–60s duration
- **Permissions**: Video + Audio via `AVCaptureDevice`

#### Location (`LocationService.swift`, 179 lines)
- One-shot location with timeout
- Streaming location updates (async stream)
- Significant change monitoring (background capable)
- When-in-use or Always authorization modes
- Background location updates support

#### Motion (`MotionService.swift`, 101 lines)
- Activity classification: walking, running, cycling, automotive, stationary, unknown
- Pedometer data: steps, distance, floors ascended/descended
- Uses `CMMotionActivityManager` and `CMPedometer`

#### Contacts (`ContactsService.swift`)
- Search and read contacts with framework permissions

#### Calendar & Reminders (`CalendarService.swift`, `RemindersService.swift`)
- List, create, modify events and reminders via EventKit

#### Photos (`PhotosService.swift`)
- Access latest photo metadata via Photos framework

### Canvas & WebView

**`ScreenWebView.swift` (195 lines):**

UIViewRepresentable wrapping WKWebView:
- A2UI action handler intercepts `window.webkit.messageHandlers.openclawCanvasA2UIAction`
- Deep link interception for `openclaw://` URLs
- Non-persistent data store (no cookies)
- Full-bleed layout (no safe area insets)
- Black background

### Live Activities

**`LiveActivityManager.swift` (126 lines):**

Uses iOS ActivityKit for Dynamic Island and Lock Screen:
```swift
struct OpenClawActivityAttributes: ActivityAttributes {
    var agentName: String, sessionKey: String

    struct ContentState: Codable, Hashable {
        var statusText: String
        var isIdle, isDisconnected, isConnecting: Bool
        var startedAt: Date
    }
}
```

The Dynamic Island shows:
- Green = idle (ready)
- Red = disconnected
- Gray = connecting
- Elapsed time or spinner

### Voice & Talk Mode

**`TalkModeManager.swift` (2,000+ lines):**

The largest file in the iOS app:
- **Microphone**: `AVAudioEngine` with tap
- **Speech recognition**: `SFSpeechRecognizer` (on-device or cloud)
- **TTS**: ElevenLabs streaming (PCM 24kHz preferred, MP3 fallback)
- **Modes**: Continuous listening, push-to-talk with auto-stop
- **Silence detection**: Configurable timeout (~1.5s default), noise floor calibration
- **Incremental speech**: Streams partial recognition results to the gateway
- **Directive support**: Can switch voices/models, interrupt TTS on speech

### Push Notifications

The iOS app supports APNs for:
- **Silent push wakes**: Gateway can wake the app in background
- **Watch prompt mirroring**: Prompts from the gateway shown as rich notifications
- **Background refresh**: Scheduled reconnection attempts
- **Action buttons**: Up to 4 configurable action buttons per notification

### Chat Transport

**`IOSGatewayChatTransport.swift` (150 lines):**

Implements `OpenClawChatTransport` for the shared ChatUI:
```swift
func sendMessage(sessionKey, message, thinking, attachments) -> OpenClawChatSendResponse
    // RPC: "chat.send" with 35s timeout
func requestHistory(sessionKey) -> OpenClawChatHistoryPayload
    // RPC: "chat.history"
func events() -> AsyncStream<OpenClawChatTransportEvent>
    // Subscribes to "tick", "seqGap", "health", "chat", "agent" events
```

### Onboarding

- QR code scanner for gateway pairing
- Wizard-based setup flow
- TLS fingerprint trust prompt (TOFU — trust on first use)

---

## Part 4: Android App

**Directory:** `apps/android/`
**Platform:** Android SDK 31+ (Android 12+), Target SDK 36

### Build Configuration

- **Namespace:** `ai.openclaw.app`
- **NDK ABIs:** armeabi-v7a, arm64-v8a, x86, x86_64
- **ProGuard**: Enabled for release builds
- **Version**: `2026.3.13` (versionCode: 202603130)

### Architecture

Jetpack Compose + Kotlin Coroutines with MVVM:
- `NodeApp` (Application) → `NodeRuntime` (953 lines, main logic)
- `MainViewModel` (150 lines) → exposes StateFlows to Compose UI
- `MainActivity` (63 lines) → Compose entry with `RootScreen`

### Two Gateway Sessions

Like iOS, the Android app maintains two parallel connections:
```kotlin
nodeSession: GatewaySession    // role=node, device capabilities
operatorSession: GatewaySession // role=operator, chat/config/talk
```

**`GatewaySession.kt` (250+ lines):**
- Uses OkHttp3 WebSocket
- RPC via `ConcurrentHashMap<String, CompletableDeferred<RpcResponse>>`
- Auth sources: DEVICE_TOKEN, SHARED_TOKEN, BOOTSTRAP_TOKEN, PASSWORD
- Device token retry fallback (1 retry budget)

### Invoke Command Registry

**`InvokeCommandRegistry.kt`:**

Maps gateway commands to handlers:

| Category | Commands |
|----------|---------|
| Canvas | present, hide, navigate, eval, snapshot |
| A2UI | reset, push, pushJsonl |
| Camera | list, snap, clip |
| Location | get |
| Device | status, info, permissions, health |
| Notifications | list, actions |
| System | notify |
| Photos | latest |
| Contacts | search |
| Calendar | list, get |
| Motion | activities, pedometer |
| SMS | send |

### Platform Capabilities

#### Camera (`CameraCaptureManager.kt`, 150+ lines)
- Uses **CameraX** (androidx.camera)
- snap: EXIF rotation handling, bitmap scaling, JPEG compression
- clip: MP4 video recording
- Smart size limiter: dynamically reduces quality to stay under 5MB
- Permissions: CAMERA + RECORD_AUDIO

#### Location (`LocationCaptureManager.kt`)
- One-shot and streaming location
- FusedLocationProviderClient integration

#### Motion (`MotionHandler.kt`)
- Step counter, accelerometer, gyroscope via SensorManager
- Activity recognition via `ACTIVITY_RECOGNITION` permission

#### SMS (`SmsHandler.kt`, `SmsManager.kt`)
- Send SMS programmatically (with `SEND_SMS` permission)
- Not available on iOS (Apple restriction)

#### Contacts (`ContactsHandler.kt`)
- Search via `ContactsContract` provider

#### Calendar (`CalendarHandler.kt`)
- List and get events via `CalendarContract` provider

#### Photos (`PhotosHandler.kt`)
- Query MediaStore for latest images

### Canvas WebView

**`CanvasController.kt` (200 lines):**
```kotlin
class CanvasController {
    fun attach(webView: WebView)
    fun navigate(url: String)
    suspend fun eval(javaScript: String): String
    suspend fun snapshotBase64(format, quality?, maxWidth?): String
}
```

- Loads scaffold from `file:///android_asset/CanvasScaffold/scaffold.html`
- A2UI via `WebView.evaluateJavascript()`
- Snapshot: Bitmap → scale → JPEG → Base64

### Voice & Talk Mode

**`TalkModeManager.kt` (200+ lines):**
- Speech recognition: `android.speech.SpeechRecognizer`
- TTS: ElevenLabs WebSocket streaming (`ElevenLabsStreamingTts.kt`) with system TTS fallback
- Audio focus management via `AudioFocusRequest`
- Modes: continuous listening, push-to-talk
- State flows: isEnabled, isListening, isSpeaking, statusText, micInputLevel

### Foreground Service

**`NodeForegroundService.kt` (150 lines):**

Android requires a foreground service to keep the WebSocket alive when the app is backgrounded:
- Persistent notification showing connection status + mic state
- `FOREGROUND_SERVICE_TYPE_DATA_SYNC`
- Observes runtime state and updates notification text dynamically

### Permissions

```xml
<!-- From AndroidManifest.xml -->
INTERNET, ACCESS_NETWORK_STATE, NEARBY_WIFI_DEVICES
ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION
CAMERA, RECORD_AUDIO
READ_CONTACTS, WRITE_CONTACTS
READ_CALENDAR, WRITE_CALENDAR
SEND_SMS
READ_MEDIA_IMAGES, READ_MEDIA_VISUAL_USER_SELECTED
ACTIVITY_RECOGNITION
POST_NOTIFICATIONS, FOREGROUND_SERVICE, FOREGROUND_SERVICE_DATA_SYNC
```

### Secure Storage

**`SecurePrefs.kt`:**
- Uses `EncryptedSharedPreferences` for credentials
- Gateway tokens, TLS fingerprints, device identity

---

## Part 5: The Thin Client Pattern

### What the Apps Do vs. What the Gateway Does

| Responsibility | App | Gateway |
|---------------|-----|---------|
| Run the AI agent | | ✓ |
| Process messages | | ✓ |
| Manage sessions | | ✓ |
| Execute tools | | ✓ |
| Take photos | ✓ | |
| Read location | ✓ | |
| Access contacts | ✓ | |
| Read motion data | ✓ | |
| Send SMS | ✓ (Android) | |
| Render UI | ✓ | |
| Play TTS audio | ✓ | |
| Recognize speech | ✓ | |
| Show notifications | ✓ | |
| Manage Live Activities | ✓ (iOS) | |

### Connection Architecture

```
Companion App
├── nodeSession (role=node)
│   ├── Capabilities: [camera, location, motion, SMS, contacts, ...]
│   ├── Commands: [camera.snap, location.get, device.status, ...]
│   └── Handles: node.invoke requests FROM the gateway
│
└── operatorSession (role=operator)
    ├── Scopes: [operator.read, operator.write, operator.talk.secrets]
    └── Methods: chat.send, chat.history, sessions.list, talk.get.config
```

### Message Flow: User → AI → Action → Response

```
1. User types "take a photo of my desk" in chat
   ↓
2. App sends chat.send via operatorSession
   ↓
3. Gateway receives message, runs AI agent
   ↓
4. AI decides to use camera tool
   ↓
5. Gateway sends node.invoke(command: "camera.snap") via nodeSession
   ↓
6. App's CameraController takes photo → JPEG → Base64
   ↓
7. App returns BridgeInvokeResponse with image data
   ↓
8. Gateway gives image to AI for analysis
   ↓
9. AI generates text response: "I can see a desk with..."
   ↓
10. Gateway sends chat event via operatorSession
    ↓
11. App displays response in ChatView
```

### Capability Advertisement

On connect, apps advertise what they can do:

```swift
// iOS
GatewayConnectOptions(
    role: "node",
    caps: ["camera", "location", "motion", "contacts", "calendar", "reminders", "canvas"],
    commands: ["camera.snap", "camera.clip", "location.get", "device.status",
               "canvas.present", "canvas.eval", "canvas.snapshot", ...],
    permissions: ["camera": true, "location": true, "microphone": true]
)
```

The gateway only sends invoke commands the app has advertised. If the app doesn't support `sms.send` (like iOS), the gateway won't try.

---

## Git History Evolution

### The December Sprint (Dec 5–14, 2025)

Peter Steinberger created all three apps and the shared library in a 10-day sprint:

**Dec 5**: The **macOS app** was born — 1,236 lines creating the menu bar app with CLI, IPC, and animated critter icon. Settings, onboarding, and menu bar icon animations followed within hours.

**Dec 6–8**: **Voice wake** was added to macOS — live speech recognition listener with animated ears, mic picker, voice wake overlay (291 lines), and adaptive delays.

**Dec 9**: **Relay renamed to gateway** — establishing the term used throughout the project.

**Dec 12**: Two major milestones:
- **Shared library created** as `ClawdisNodeKit` (1,002 lines, 17 files) — Bonjour types, bridge frames, screen commands
- **iOS app created** (1,348 lines, 17 files) — bridge client, discovery, session, keychain, voice wake, screen controller

**Dec 13**: Shared library renamed to `ClawdisKit` for broader scope. Bonjour beacons and bridge presence added.

**Dec 14**: Three more milestones in one day:
- **Shared chat UI** — `ClawdisChatUI` module (1,997 lines) with ChatComposer, ChatView, ChatViewModel
- **Camera integration** — snap/clip capture for iOS + macOS (1,669 lines)
- **Android app created** — 2,518 lines, 27 files: Jetpack Compose, bridge discovery, camera, canvas, chat, settings, and foreground service

By December 14, all three platforms had functional apps connecting to the gateway.

### The Three Renames (Jan–Feb 2026)

The project went through three name changes:

1. **Clawdis → Clawdbot** (Jan 4, 2026) — `246adaa11`
2. **Clawdbot → Moltbot** (Jan 27, 2026) — `6d16a658e`, 110KB diff
3. **Moltbot → OpenClaw** (Jan 30, 2026) — `9a7160786`, 140KB diff. Package IDs changed from `bot.molt` to `ai.openclaw`

Each rename required updating bundle identifiers, package names, file paths, and references across all three apps plus the shared library. Residual cleanup continued into March.

### Talk Mode & Streaming TTS (Feb–Mar 2026)

- **Feb 21**: **Nimrod Gutman** added provider-agnostic talk mode config with legacy compatibility
- **Feb 28**: **Greg Mousseau** created **Android ElevenLabs streaming TTS** — WebSocket-based streaming with PCM playback (325 lines), wired into NodeRuntime, with thread safety fixes and TTS fallback
- **Feb 28**: **Ayaan Zaidi** added speaker toggle in Android voice tab
- **Mar 3**: Nimrod Gutman added talk mode latency tracing

### iOS Security Hardening (Mar 2–3, 2026)

**Mariano** delivered a 5-part security stack for iOS:
1. **Keychain Migrations + Tests** — secure credential storage
2. **Concurrency Locks** — thread-safe access to shared state
3. **Runtime Security Guards** — validation at critical boundaries
4. **TTS PCM→MP3 Fallback** — graceful degradation when PCM fails
5. (Part 5 delivered separately)

**Rocuts** moved gateway connection metadata to Keychain, and **Josh Avant** expanded SecretRef coverage across credentials.

### iOS Live Activity (Mar 4, 2026)

**Mariano** added Live Activity support — Dynamic Island and Lock Screen status display showing connection state (connected/disconnected/connecting) with elapsed time. Created `ActivityWidget` bundle, `LiveActivityManager` (126 lines), and `OpenClawActivityAttributes`.

### Android UI Redesign (Mar 13, 2026)

**Ayaan Zaidi** delivered a complete UI overhaul in a single day:
- Redesigned onboarding flow
- New Connect tab with unified status cards
- Speaker label and status pill for Voice tab
- Compact chat composer layout
- Google Code Scanner for QR onboarding

### iOS Push Notifications & App Store (Mar 7–12, 2026)

- **Nimrod Gutman** added iOS APNs relay gateway (#43369) — enabling push notifications for the iOS app
- **Nimrod Gutman** prepared App Store Connect release assets
- **Nimrod Gutman** added iOS beta release flow
- **Nimrod Gutman** added onboarding welcome pager

### The Evolution in Four Sentences

Peter Steinberger created all three companion apps (macOS, iOS, Android) and the shared library in a 10-day sprint in December 2025, establishing the thin-client architecture from day one. The project went through three name changes (Clawdis → Clawdbot → Moltbot → OpenClaw) by January 30, 2026. February–March brought community contributions: Greg Mousseau built Android streaming TTS, Mariano delivered the iOS security stack and Live Activities, Ayaan Zaidi redesigned the Android UI, and Nimrod Gutman prepared the iOS App Store release. Throughout it all, the core design remained constant: apps are thin clients exposing platform capabilities, while the gateway runs the AI.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      OpenClaw Gateway                                │
│                                                                      │
│  AI Agent  │  Sessions  │  Channels  │  Tools  │  Memory  │  TTS   │
│                                                                      │
│  WebSocket Server (port 18789)                                       │
│  ├── node.invoke → commands device capabilities                     │
│  ├── chat.send/history → chat RPCs                                  │
│  ├── events → push (agent, heartbeat, chat)                        │
│  └── canvas.* → A2UI push/reset/eval                               │
└───────────┬──────────────────┬──────────────────┬────────────────────┘
            │                  │                  │
     ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐
     │   macOS App   │  │   iOS App    │  │ Android App  │
     │  (SwiftUI)    │  │  (SwiftUI)   │  │ (Compose)    │
     │               │  │              │  │              │
     │ ┌───────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
     │ │ nodeSession│ │  │ │nodeSession│ │  │ │nodeSession│ │
     │ │ (role=node)│ │  │ │(role=node)│ │  │ │(role=node)│ │
     │ └─────┬─────┘ │  │ └─────┬────┘ │  │ └─────┬────┘ │
     │       │        │  │      │       │  │       │      │
     │ ┌─────▼─────┐  │  │ ┌────▼────┐  │  │ ┌─────▼────┐ │
     │ │ operSession│  │  │ │operSess │  │  │ │operSess  │ │
     │ │(role=oper) │  │  │ │(r=oper) │  │  │ │(r=oper)  │ │
     │ └───────────┘  │  │ └─────────┘  │  │ └──────────┘ │
     │                │  │              │  │              │
     │ Capabilities:  │  │ Capabilities:│  │ Capabilities:│
     │ • Voice wake   │  │ • Camera     │  │ • Camera    │
     │ • Camera       │  │ • Location   │  │ • Location  │
     │ • Screen rec   │  │ • Motion     │  │ • Motion    │
     │ • Canvas       │  │ • Contacts   │  │ • Contacts  │
     │ • IPC socket   │  │ • Calendar   │  │ • Calendar  │
     │ • CLI tool     │  │ • Reminders  │  │ • SMS       │
     │                │  │ • Photos     │  │ • Photos    │
     │ UI:            │  │ • Push (APNs)│  │ • Notif.    │
     │ • Menu bar     │  │ • Live Act.  │  │             │
     │ • Chat panel   │  │ • Widgets    │  │ UI:         │
     │ • Canvas panel │  │              │  │ • Compose   │
     │ • Voice overlay│  │ UI:          │  │ • Canvas WV │
     │ • Settings     │  │ • SwiftUI    │  │ • Voice tab │
     │                │  │ • Canvas WV  │  │ • Settings  │
     │                │  │ • Talk orb   │  │ • FG Service│
     │                │  │ • Onboarding │  │ • Onboarding│
     └────────────────┘  └──────────────┘  └─────────────┘

     Shared Library: OpenClawKit
     ├── OpenClawProtocol  (wire types)
     ├── OpenClawKit       (gateway client, commands, auth)
     └── OpenClawChatUI    (shared chat components)
```

---

## Key Files Reference

### Shared Library (OpenClawKit)

| File | Purpose |
|------|---------|
| `apps/shared/OpenClawKit/Package.swift` | SPM manifest (iOS 18+, macOS 15+) |
| `Sources/OpenClawProtocol/GatewayModels.swift` | Auto-generated wire types |
| `Sources/OpenClawKit/GatewayChannel.swift` | WebSocket abstraction |
| `Sources/OpenClawKit/GatewayNodeSession.swift` | Main connection actor (536 lines) |
| `Sources/OpenClawKit/GatewayPush.swift` | Push subscription types |
| `Sources/OpenClawKit/GatewayErrors.swift` | Error types (147 lines) |
| `Sources/OpenClawKit/CanvasA2UIAction.swift` | A2UI action formatting (105 lines) |
| `Sources/OpenClawKit/TalkDirective.swift` | TTS directive parsing (202 lines) |
| `Sources/OpenClawKit/Capabilities.swift` | Capability enum |

### macOS App

| File | Lines | Purpose |
|------|-------|---------|
| `Sources/OpenClaw/MenuBar.swift` | — | @main entry, menu bar app |
| `Sources/OpenClaw/AppState.swift` | 150+ | Observable app state |
| `Sources/OpenClaw/GatewayConnection.swift` | 800+ | Gateway WebSocket actor |
| `Sources/OpenClaw/ControlChannel.swift` | 425+ | High-level gateway API |
| `Sources/OpenClaw/CanvasManager.swift` | 340+ | Canvas window management |
| `Sources/OpenClaw/CanvasWindowController.swift` | 200+ | WKWebView canvas |
| `Sources/OpenClaw/VoiceWakeForwarder.swift` | 74 | Speech → agent messages |
| `Sources/OpenClawIPC/IPC.swift` | 417 | Unix socket IPC protocol |
| `Sources/OpenClawMacCLI/EntryPoint.swift` | 57 | CLI entry point |

### iOS App

| File | Lines | Purpose |
|------|-------|---------|
| `Sources/OpenClawApp.swift` | 549 | App delegate, push, background |
| `Sources/Model/NodeAppModel.swift` | 3,020 | Main state model |
| `Sources/Voice/TalkModeManager.swift` | 2,000+ | Speech recognition + TTS |
| `Sources/Camera/CameraController.swift` | 354 | AVFoundation camera |
| `Sources/Location/LocationService.swift` | 179 | CoreLocation wrapper |
| `Sources/Motion/MotionService.swift` | 101 | CoreMotion activities + pedometer |
| `Sources/Chat/IOSGatewayChatTransport.swift` | 150 | Chat RPC adapter |
| `Sources/Screen/ScreenWebView.swift` | 195 | WKWebView + A2UI |
| `Sources/LiveActivity/LiveActivityManager.swift` | 126 | Dynamic Island |
| `ActivityWidget/OpenClawLiveActivity.swift` | 85 | Widget UI |

### Android App

| File | Lines | Purpose |
|------|-------|---------|
| `NodeRuntime.kt` | 953 | Main runtime, handler coordination |
| `gateway/GatewaySession.kt` | 250+ | OkHttp WebSocket RPC |
| `node/InvokeDispatcher.kt` | 150+ | Command → handler routing |
| `node/CanvasController.kt` | 200 | Android WebView canvas |
| `node/CameraCaptureManager.kt` | 150+ | CameraX integration |
| `chat/ChatController.kt` | 150+ | Chat state management |
| `voice/TalkModeManager.kt` | 200+ | Speech + ElevenLabs TTS |
| `voice/ElevenLabsStreamingTts.kt` | — | WebSocket streaming TTS |
| `NodeForegroundService.kt` | 150 | Persistent notification service |
| `MainActivity.kt` | 63 | Compose entry point |
| `MainViewModel.kt` | 150 | MVVM ViewModel |
