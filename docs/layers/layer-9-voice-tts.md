# Layer 9: Voice / TTS — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Voice/TTS? (The Simple Version)

Imagine you have a super-smart friend who only communicates by text messages. That's your AI assistant. But sometimes you're driving, cooking, or just too lazy to read — you want your friend to **talk to you** with their actual voice. And sometimes you want to **call them on the phone** and have a real conversation.

That's what the Voice/TTS layer does:

1. **Text-to-Speech (TTS)**: Takes the AI's text reply and converts it to audio — like having someone read it out loud
2. **Voice Calls**: Lets you actually phone-call the AI through Twilio, Telnyx, or Plivo — with real-time back-and-forth conversation
3. **Discord Voice**: Lets the AI join a Discord voice channel and talk/listen in real-time

Think of it as three levels of voice:
- **Level 1**: AI sends you a voice note instead of text (TTS)
- **Level 2**: You call the AI on a real phone number (Voice Calls)
- **Level 3**: AI sits in a Discord voice channel chatting with everyone (Discord Voice)

---

## Part 1: Text-to-Speech Core — Making the AI Talk

### The TTS Configuration System

**File:** `src/tts/tts-core.ts` (719 lines)

TTS starts with configuration. There are four TTS backends, each with its own settings:

#### Backend 1: ElevenLabs (Cloud, Premium)
```
Default voice ID: pMsXgVXv3BLzUgSXRplE
Default model: eleven_multilingual_v2
API: POST to {baseUrl}/v1/text-to-speech/{voiceId}
Auth: xi-api-key header
```

ElevenLabs is the premium option — it supports voice cloning, multiple languages, and fine-grained control:
- **stability** (0–1): How consistent the voice sounds. Low = more expressive but variable
- **similarityBoost** (0–1): How close to the original voice clone
- **style** (0–1): How much style/emotion to add
- **useSpeakerBoost**: Amplify the speaker characteristics
- **speed** (0.5–2): Playback speed
- **seed**: For reproducible audio output
- **applyTextNormalization**: "auto", "on", or "off" — how to handle numbers, abbreviations, etc.
- **languageCode**: 2-letter ISO 639-1 code for multilingual voices

#### Backend 2: OpenAI TTS (Cloud, Easy)
```
Default model: gpt-4o-mini-tts
Default voice: alloy
Models: gpt-4o-mini-tts, tts-1, tts-1-hd
API: POST to {baseUrl}/audio/speech
```

OpenAI TTS has 14 voices: alloy, ash, ballad, cedar, coral, echo, fable, juniper, marin, onyx, nova, sage, shimmer, verse.

The `gpt-4o-mini-tts` model is special — it supports **instructions** (like "speak in a warm, friendly tone"), which the older tts-1 models don't.

Custom OpenAI-compatible endpoints are supported. When using a non-default base URL, any voice or model name is accepted (since third-party services may have different options).

#### Backend 3: Edge TTS (Free, Offline-ish)
```
Default voice: en-US-MichelleNeural
Default language: en-US
Default format: audio-24khz-48kbitrate-mono-mp3
Library: node-edge-tts
```

Edge TTS uses Microsoft's text-to-speech engine (the one built into Edge browser). It's free, requires no API key, and works as a fallback when cloud providers aren't configured.

#### Backend 4: Sherpa-ONNX (Fully Local)
A skill (not a code backend) that runs TTS entirely on your device using the Sherpa-ONNX inference engine with Piper VITS models. Zero cloud dependency. Supports macOS, Linux, and Windows.

### TTS Directive Tags — AI-Controlled Voice

The AI can embed special tags in its text to control how it's spoken:

```
[[tts:provider=elevenlabs]]
[[tts:voice=shimmer]]
[[tts:speed=1.5]]
[[tts:stability=0.8]]
[[tts:text]]This is spoken differently from the visible text.[[/tts:text]]
```

The directive parser (`parseTtsDirectives()`) uses a two-pass regex:
1. **Block tags**: `[[tts:text]]...[[/tts:text]]` — replacement text for TTS
2. **Inline tags**: `[[tts:key=value]]` — override provider, voice, model, speed, etc.

After parsing, directives are stripped from the visible text. The AI can write one thing and speak another.

**Security**: Model override permissions are controlled by a policy:
```typescript
{
  allowText: true,        // AI can provide alternate TTS text
  allowProvider: false,    // AI can't switch providers (opt-in only — billing concern)
  allowVoice: true,       // AI can pick a voice
  allowModelId: true,     // AI can pick a model
  allowVoiceSettings: true,// AI can adjust stability/speed
  allowNormalization: true,
  allowSeed: true,
}
```

`allowProvider` is false by default — because switching providers could incur unexpected costs.

### Text Preparation Pipeline

Before text reaches a TTS engine, it goes through several steps:

1. **Parse directives** — extract and remove `[[tts:...]]` tags
2. **Strip markdown** — remove `###`, `**bold**`, `` `code` ``, `> blockquotes`, `~~strikethrough~~`, and horizontal rules
3. **Check length** — if text > `maxTextLength` (default 1500 chars):
   - **Summarize** with AI (if enabled) — uses `completeSimple()` with temperature 0.3
   - **Truncate** if summarization is disabled or fails
4. **Minimum check** — skip TTS if text < 10 chars after cleaning

The summarization function (`summarizeText()`) supports configuring a separate `summaryModel` (e.g., a cheaper/faster model for summaries), and can even use Ollama for fully local summarization.

---

## Part 2: The Main TTS Engine

**File:** `src/tts/tts.ts` (1,004 lines)

### Output Format Selection

Different channels need different audio formats:

| Channel | Format | Sample Rate | Bitrate | Voice Bubble? |
|---------|--------|-------------|---------|---------------|
| Telegram | opus | 48kHz | 64kbps | Yes |
| Feishu | opus | 48kHz | 64kbps | Yes |
| WhatsApp | opus | 48kHz | 64kbps | Yes |
| Everything else | mp3 | 44.1kHz | 128kbps | No |
| Telephony (OpenAI) | pcm | 24kHz | — | — |
| Telephony (ElevenLabs) | pcm | 22.05kHz | — | — |

"Voice bubble" means the message appears as a playable voice note in the chat (like a WhatsApp voice message) rather than a file attachment.

### The Conversion Flow

**`textToSpeech(params)`** — the main function:

1. Resolve configuration (merge config + user prefs + directives)
2. Select provider (user preference, then fallback order)
3. Call the backend:
   - **Edge**: Use `node-edge-tts` library → generates file locally → retry with default format on failure
   - **ElevenLabs**: POST to API → receive audio buffer → write temp file
   - **OpenAI**: POST to API → receive audio buffer → write temp file
4. Write audio to temp file with appropriate extension (.opus, .mp3, etc.)
5. Schedule cleanup after 5 minutes
6. Return `TtsResult` with audioPath, provider, latency, etc.

**Provider fallback**: If the first provider fails, the next one is tried. This is how Edge TTS serves as a free fallback.

### Auto-TTS Modes

The system has four auto-TTS modes that control when replies become audio:

| Mode | Behavior |
|------|----------|
| `off` | Never auto-generate audio |
| `always` | Every reply becomes audio |
| `inbound` | Only when the user sent a voice message |
| `tagged` | Only when the AI includes `[[tts]]` directives |

**`maybeApplyTtsToPayload()`** — the integration point for message delivery:
1. Check auto-mode gates (off? skip. inbound? check if user sent audio. tagged? check for directives.)
2. Check mode gates — if mode="final", skip non-final streaming chunks
3. Skip if message already has media (mediaUrl/mediaUrls or "MEDIA:" prefix)
4. Process text through the preparation pipeline
5. Convert to audio
6. Attach `mediaUrl` to the payload; set `audioAsVoice=true` for Telegram/Feishu/WhatsApp

### User Preferences

TTS preferences persist in `~/.openclaw/settings/tts.json` using atomic writes (write to temp file, then rename). Users can control:
- Auto mode (off/always/inbound/tagged)
- Provider (openai/elevenlabs/edge)
- Max text length (100–4096 chars)
- Summarization on/off

### The `/tts` Command

**File:** `src/auto-reply/reply/commands-tts.ts` (280 lines)

Users can control TTS through chat commands:
- `/tts` or `/tts status` — show current settings
- `/tts on` / `/tts off` — enable/disable
- `/tts provider openai` — switch provider
- `/tts limit 2000` — change max text length
- `/tts summary on` — enable auto-summarization
- `/tts audio Hello world` — generate audio on demand
- `/tts help` — show usage

### The Agent Tool

**File:** `src/agents/tools/tts-tool.ts` (62 lines)

The AI can also explicitly generate audio via the `tts` tool:
```typescript
{
  name: "tts",
  schema: {
    text: String,          // Required: text to convert
    channel: Optional<String>  // Optional: channel for format selection
  }
}
```

On success, returns `MEDIA:{audioPath}` (with optional `[[audio_as_voice]]` tag). The tool description includes `SILENT_REPLY_TOKEN` — telling the AI to not send a text reply after generating audio, to avoid duplicate messages.

### Gateway API

**File:** `src/gateway/server-methods/tts.ts` (157 lines)

External clients (companion apps, web UI) can access TTS through:
- `tts.status` — current settings, provider, key availability
- `tts.convert` — convert text to audio
- `tts.enable` / `tts.disable` — toggle
- `tts.setProvider` — switch provider
- `tts.providers` — list available providers with models and voices

---

## Part 3: Voice Calls — Real Phone Conversations

**Directory:** `extensions/voice-call/` (60+ files)

This is where things get serious. The voice-call extension lets you have actual phone conversations with the AI — including outbound calls, inbound calls, and multi-turn dialogue.

### Call State Machine

Every call follows this state machine:

```
[initiated] → [ringing] → [answered] → [active]
                                           │
                                    ┌──────┴──────┐
                                    ▼              ▼
                               [speaking]  ←→  [listening]
                                    │
                                    ▼ (terminal)
        [completed | hangup-user | hangup-bot | timeout |
         error | failed | no-answer | busy | voicemail]
```

States only move **forward** (you can't go from "answered" back to "ringing"), except for the `speaking ↔ listening` cycle which repeats for multi-turn conversations. Any state can transition to a terminal state.

### Three Telephony Providers

#### Twilio
- **Auth**: Account SID + Auth Token
- **Webhook verification**: HMAC-SHA1 (URL + sorted POST params)
- **Media streams**: WebSocket-based real-time audio
- **Special**: TwiML-based call control, `<Gather>` for speech recognition, `<Say>` for TTS, `<Stream>` for bidirectional audio
- **Token auth**: Per-call crypto.randomBytes(16) tokens passed via `<Parameter>` (not query string, for security)

#### Telnyx
- **Auth**: Bearer token (API key)
- **Webhook verification**: Ed25519 signature with timestamp clock-skew check (5 min max)
- **Call control**: REST API v2 (`https://api.telnyx.com/v2`)
- **Special**: client_state field (Base64-encoded callId) for tracking
- **Hangup cause mapping**: Telnyx codes → normalized EndReason (normal_clearing→completed, no_answer→no-answer, etc.)

#### Plivo
- **Auth**: Basic auth (Base64 of authId:authToken)
- **Webhook verification**: HMAC-SHA256, supports V2 and V3 signatures
- **Call control**: REST API (`https://api.plivo.com/v1/Account/{authId}`)
- **Special**: XML-based flow routing (flow=xml-speak, flow=xml-listen), transfer between call legs

### Call Modes

**Notify mode**: The AI calls you, says a message, waits a few seconds (`notifyHangupDelaySec`, default 3s), and hangs up. One-way delivery.

**Conversation mode**: Full multi-turn dialogue. The AI speaks, then listens, then responds, in a loop until someone hangs up or a timeout is reached.

### The Multi-Turn Conversation Loop

When `continueCall()` is called:

1. **Speak** the prompt — transitions to `speaking` state
2. **Listen** — transitions to `listening` state, generates a `turnToken` (nonce for replay hardening)
3. **Wait** for transcript — `waitForFinalTranscript()` with timeout (180s default)
4. **Process** — the transcript is returned to the caller
5. **Generate response** — `generateVoiceResponse()` runs an embedded Pi agent
6. Loop back to step 1

Latency is tracked per turn: `lastTurnLatencyMs` (total) and `lastTurnListenWaitMs` (listening portion).

### Media Streaming (Real-Time Audio)

**File:** `extensions/voice-call/src/media-stream.ts` (528 lines)

For Twilio streaming, audio flows through WebSockets in real-time:

```
User speaks on phone
  → Twilio sends Opus audio frames via WebSocket
  → MediaStreamHandler receives "media" messages
  → Forwards audio to OpenAI Realtime STT session
  → STT transcribes to text
  → manager.processEvent(call.speech)
  → generateVoiceResponse() via embedded Pi agent
  → TelephonyTtsProvider synthesizes response audio
  → Convert PCM → mu-law 8kHz (telephony standard)
  → Queue for serialized playback (prevents overlap)
  → Chunk into 20ms frames (160 bytes @ 8kHz mono)
  → Send back through WebSocket
  → Twilio plays audio to caller
```

**Connection limits** (anti-abuse):
- Max pending connections: 32
- Max per IP: 4
- Max total: 128
- Pre-start timeout: 5s

**TTS Queue** (`queueTts()`): Serializes TTS playback — ensures audio chunks don't overlap. Uses an AbortController for barge-in (when the user starts talking mid-playback).

### Audio Processing

**File:** `extensions/voice-call/src/telephony-audio.ts` (91 lines)

Telephony uses a specific audio format: **8kHz mono mu-law** (G.711 standard). The system converts from the TTS output:

1. **Resample** — linear interpolation from source sample rate (24kHz for OpenAI, 22.05kHz for ElevenLabs) down to 8kHz
2. **PCM to mu-law** — compress 16-bit linear PCM to 8-bit mu-law encoding:
   - Sign bit extraction
   - Exponent field (3 bits) — logarithmic compression
   - Mantissa (4 bits) — fine detail
   - Final bit inversion: `~(sign | (exp << 4) | mantissa) & 0xff`
3. **Chunk** — split into 20ms frames (160 bytes at 8kHz mono)

### Webhook Server

**File:** `extensions/voice-call/src/webhook.ts` (488 lines)

The webhook server receives callbacks from telephony providers:

1. HTTP server on port 3334 (default), bound to 127.0.0.1
2. WebSocket upgrade support for media streams
3. Request pipeline:
   - Path matching (`/voice/webhook` or `/voice/hold-music`)
   - Signature verification (HMAC-SHA1/Ed25519/HMAC-SHA256 depending on provider)
   - Event parsing → normalized events
   - Replay detection (logged but still processed)
   - Pass to `manager.processEvent()`
   - Return provider-specific response (TwiML XML or JSON)

**Stale call reaper**: Optional periodic cleanup of calls stuck in non-terminal states (`staleCallReaperSeconds` config).

### Webhook Security

**File:** `extensions/voice-call/src/webhook-security.ts` (981 lines)

This is the most security-critical file in the voice system:

**Replay protection**:
- Per-provider replay cache mapping dedupeKey → expiration
- 10-minute replay window
- Auto-prune when cache exceeds 10,000 entries
- Prunes every 64th call for efficiency

**URL reconstruction** (for signature verification):
- Priority: X-Forwarded-Proto + X-Forwarded-Host → X-Original-Host → Ngrok-Forwarded-Host → Host header → fallback
- RFC 1123 hostname validation
- IPv6 support (`[::1]:port` format)
- Rejects `@` character (user info injection attack)
- Optional allowedHosts whitelist
- Optional trustedProxyIPs list

### Tunneling for Local Development

**File:** `extensions/voice-call/src/tunnel.ts` (315 lines)

Since telephony providers need to reach your webhook, local development requires tunneling:

**ngrok**:
- Spawns ngrok CLI process
- Parses JSON logs for public URL
- 30s startup timeout
- Supports custom domains

**Tailscale**:
- Two modes: `serve` (tailnet-private) or `funnel` (public HTTPS)
- Resolves Tailscale DNS name
- 10s timeout

### Inbound Call Policy

**File:** `extensions/voice-call/src/allowlist.ts` (20 lines)

Four policies for who can call the AI:
| Policy | Behavior |
|--------|----------|
| `disabled` | Block all inbound calls |
| `allowlist` | Only numbers in `allowFrom` list |
| `pairing` | Same as allowlist (future expansion) |
| `open` | Accept all inbound calls |

Phone numbers are normalized (strip non-digits) before comparison. Rejected calls are hung up immediately.

### Call Storage

**File:** `extensions/voice-call/src/manager/store.ts` (95 lines)

Calls are persisted in JSONL format at `~/.openclaw/voice-calls/calls.jsonl`. Each line is a complete `CallRecord` with:
- callId, providerCallId, provider, direction, state
- from, to, sessionKey, timestamps
- endReason, transcript array, metadata

Writes are fire-and-forget (async, non-blocking). On startup, the system reloads active calls and verifies them with the provider.

### Configuration

**File:** `extensions/voice-call/src/config.ts` (526 lines)

Key defaults:
| Setting | Default |
|---------|---------|
| enabled | false |
| maxDurationSeconds | 300 (5 minutes) |
| silenceTimeoutMs | 800ms |
| transcriptTimeoutMs | 180,000ms (3 minutes) |
| ringTimeoutMs | 30,000ms (30 seconds) |
| maxConcurrentCalls | 1 |
| webhook port | 3334 |
| webhook bind | 127.0.0.1 |
| responseModel | openai/gpt-4o-mini |
| responseTimeoutMs | 30,000ms |

Streaming config:
| Setting | Default |
|---------|---------|
| sttProvider | openai-realtime |
| sttModel | gpt-4o-transcribe |
| silenceDurationMs | 800ms |
| vadThreshold | 0.5 |
| preStartTimeoutMs | 5,000ms |
| maxPendingConnections | 32 |
| maxConnections | 128 |

### CLI Commands

**File:** `extensions/voice-call/src/cli.ts` (381 lines)

- `voicecall call --message "Hello" --to +1234567890` — initiate a call
- `voicecall continue --call-id <id> --message "How are you?"` — multi-turn
- `voicecall speak --call-id <id> --message "Goodbye"` — one-way speech
- `voicecall end --call-id <id>` — hang up
- `voicecall status --call-id <id>` — show call details
- `voicecall tail` — stream call logs
- `voicecall latency` — analyze turn latency (min/max/avg/p50/p95)
- `voicecall expose --mode funnel` — set up Tailscale tunnel

---

## Part 4: Discord Voice — AI in Voice Channels

**Directory:** `src/discord/voice/` (3 files)

### Discord Voice Manager

**File:** `src/discord/voice/manager.ts` (903 lines)

This lets the AI join Discord voice channels and participate in conversations.

**Audio specs**:
| Setting | Value |
|---------|-------|
| Sample rate | 48kHz (Discord standard) |
| Channels | 2 (stereo) |
| Bit depth | 16 (PCM) |
| Min segment | 0.35 seconds |
| Silence timeout | 1000ms |
| Playback ready timeout | 15,000ms |
| Speaking ready timeout | 60,000ms |

**Join flow**:
1. Fetch voice channel from Discord API
2. Create `VoiceConnection` via `@discordjs/voice`
3. Subscribe `AudioPlayer` to connection
4. Register speaking detection handlers
5. Handle disconnection (auto-reconnect or cleanup)

**Speaking detection**:
When a user speaks, the manager:
1. Detects speaking start via Discord's speaking event
2. Subscribes to the user's audio stream
3. Decodes Opus audio frames
4. Writes raw audio to WAV file
5. Transcribes using the media capability (likely Whisper)
6. Generates a response via the agent
7. Converts response to audio via TTS
8. Queues playback through the AudioPlayer

**DAVE protocol**: Discord's end-to-end encryption for voice. The manager sets `selfDeaf=false, selfMute=false` and tracks decryption failures. If too many failures occur, it performs auto-recovery (leave + rejoin the channel).

**Speaker context caching**: To avoid looking up the same user repeatedly, speaker info (label, isOwner) is cached for 60 seconds per guild:user pair.

### Voice Commands

**File:** `src/discord/voice/command.ts` (374 lines)

Discord slash commands:
- `/voice join -channel <channel>` — join a voice channel
- `/voice leave` — leave the current voice channel
- `/voice status` — show active voice sessions

**Authorization** checks:
- Guild membership
- Channel-level enabled/allowlist config
- Member roles
- Owner allowlist
- Access group policy

---

## Part 5: Voice Mapping — Cross-Provider Voice Names

**File:** `extensions/voice-call/src/voice-mapping.ts` (68 lines)

When using TwiML's `<Say>` element (Twilio's built-in TTS), OpenAI voice names are mapped to Amazon Polly voices:

| OpenAI | Polly |
|--------|-------|
| alloy | Polly.Joanna |
| echo | Polly.Matthew |
| fable | Polly.Amy |
| onyx | Polly.Brian |
| nova | Polly.Salli |
| shimmer | Polly.Kimberly |

Names starting with "Polly." or "Google." are passed through as-is. XML special characters are escaped.

---

## Part 6: Response Generation

**File:** `extensions/voice-call/src/response-generator.ts` (159 lines)

When the AI needs to respond during a voice call:

1. Load core agent dependencies (via core-bridge dynamic import)
2. Build session key: `voice:{normalizedPhone}`
3. Resolve model from `voiceConfig.responseModel` (default: openai/gpt-4o-mini)
4. Build system prompt with conversation history
5. Run embedded Pi agent with:
   - `messageProvider: "voice"`
   - `lane: "voice"` (separate processing lane)
   - Timeout: 30s default
6. Extract text from agent payloads, join with spaces

The agent runs in a dedicated "voice" lane to avoid competing with text message processing.

---

## Git History Evolution

### The Very Beginning: Twilio WhatsApp Relay (Nov 2025)

The project's **first-ever commit** on November 24, 2025, by **Peter Steinberger**, was called "Warelay" — a WhatsApp relay using Twilio webhooks. Twilio was the original telephony backbone. But by December 5, Twilio was dropped from the core runtime as the project pivoted to web-first.

### Voice Wake and Talk Mode (Dec 2025)

Starting December 6, Peter Steinberger built voice wake features for macOS — permission callbacks, voice wake overlays, and configurable wake words for iOS. On **December 30**, a burst of commits added:
- **ElevenLabs integration** — first TTS provider, wired as default for macOS talk mode
- Extracted `ElevenLabsKit` as its own module
- Hard timeouts for synthesis to prevent hangs
- **Talk mode key distribution** — gateway-level TTS polling for native apps
- **Streaming audio playback** — `PCMStreamingAudioPlayer` for real-time TTS (1,091 lines)

This was the foundation: voice on native apps via ElevenLabs streaming.

### Telegram Voice Notes (Jan 4, 2026)

**Manuel Maly** (@manmal) added the first channel-specific voice support: Telegram voice notes using `[[audio_as_voice]]` tags. This established the pattern of channel-aware audio formatting.

### The Voice-Call Plugin Explosion (Jan 11–14, 2026)

In a four-day sprint, Peter Steinberger created the entire voice-call extension:

- **Jan 11**: Created the plugin architecture and scaffolded the voice-call extension (24+ files)
- **Jan 12**: Massive implementation commit — Twilio provider (537 lines), Telnyx provider (364 lines), OpenAI Realtime STT (303 lines), OpenAI TTS provider (264 lines), media stream handling, tunnel support, webhook security, voice mapping. Thousands of lines in a single commit.
- **Jan 13**: Released voice-call plugin v0.1.0
- **Jan 13**: **vrknetha** contributed the **Plivo provider** (1,861 lines) — third telephony provider, community PR #846
- **Jan 14**: Refactored Twilio into api/webhook modules and split the call manager

### Sherpa-ONNX: Fully Local TTS (Jan 21, 2026)

Peter Steinberger added the sherpa-onnx-tts skill — a shell script wrapper around the Sherpa-ONNX inference engine for completely local, offline TTS using Piper VITS models.

### The Telegram-TTS → Core Migration (Jan 23–25, 2026)

This is a fascinating evolution story:

1. **Jan 23**: **Glucksberg** created a standalone `telegram-tts` extension — TTS voice responses just for Telegram
2. **Jan 24**: Glucksberg added auto-TTS hooks, provider switching, `/tts_limit` command, and auto-summarization
3. **Jan 24**: Peter Steinberger saw the value and **moved TTS into core** (PR #1559, crediting Glucksberg). Deleted the entire telegram-tts extension (1,042 lines removed) and created the core TTS system: `src/tts/tts.ts` (630 lines), tests (234 lines), agent tool (60 lines), gateway endpoint (138 lines). This was the architectural pivotTTS became a first-class feature available to ALL channels, not just Telegram.
4. **Jan 24**: **Seb Slight** added **auto-mode gating** on inbound voice notes (#1667) — when users send voice, replies auto-generate as audio
5. **Jan 25**: Peter Steinberger added **Edge TTS** as a free fallback (467 lines) — TTS now works without any API keys
6. **Jan 25**: **zhixian** added custom OpenAI-compatible TTS endpoints
7. **Jan 25**: **Dan Guido** fixed TTS audio overlap with an iterative queue (#1713) — preventing heap exhaustion from concurrent synthesis

### Community Hardening (Feb 2026)

February brought a wave of community contributions:

- **nyanjou**: Discord voice message send/receive support
- **danielwanwx**: Strip markdown before TTS (#13237) — removing `###` and `**` that sound terrible when read aloud
- **mcwigglesmcgee**: Fixed Twilio stream auth — tokens moved from query string to `<Parameter>` for security (#14029)
- **Marcus Castro**: Preserve TTS overrides across session resets
- **大猫子 (lailoo)**: Added 5 missing OpenAI voices: ballad, cedar, juniper, marin, verse (#11020)
- **David Cantu Martinez**: Hang up rejected inbound calls instead of leaving them ringing (#15892)
- **ikari**: Show ALL provider errors, not just the last one
- **zerone0x**: Updated tool description to prevent duplicate audio delivery (#18046)

### Discord Voice Channel Integration (Feb 20, 2026)

**Shadow** created the Discord voice channel system (#18774) — `command.ts` (339 lines) and `manager.ts` (670 lines). This was a major feature: the AI could now **join Discord voice channels** and participate in real-time conversations with Opus audio, DAVE protocol encryption, and integrated STT/TTS. 1,917 lines total.

### Late Additions (Feb–Mar 2026)

- **Feb 21**: Peter Steinberger made TTS model/provider overrides opt-in (preventing billing surprises)
- **Feb 27**: **Hershey Goldberger** added call-waiting queue for inbound Twilio calls
- **Feb 28**: **Greg Mousseau** added Android ElevenLabs streaming TTS via WebSocket (325 lines)
- **Feb 28**: **smthfoxy** enabled opus voice bubbles for Feishu and WhatsApp (#27366)
- **Mar 5**: **Kai** added OpenAI TTS baseUrl support for Discord voice (#34321)
- **Mar 11**: **ademczuk** added speed and instructions fields to voice-call OpenAI config (#39226)
- **Mar 13**: Peter Steinberger added **persisted replay protection** — a durable replay ledger for webhook dedup against replay attacks

### The Evolution in Four Sentences

Voice started as a Twilio WhatsApp relay in the first-ever commit (Nov 2025), then pivoted to ElevenLabs streaming for macOS talk mode (Dec 2025). The voice-call telephony plugin was built in a four-day sprint (Jan 11–14, 2026) with three providers (Twilio, Telnyx, Plivo). A community contribution (Glucksberg's telegram-tts) catalyzed TTS moving into core as a first-class feature available to all channels (Jan 24). Discord voice channels, Android streaming, and extensive community hardening rounded out the system through March 2026.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Voice / TTS Layer                            │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    TTS Core (src/tts/)                          │  │
│  │                                                                 │  │
│  │  Text Input → Strip Markdown → Parse Directives → Summarize?   │  │
│  │                                                                 │  │
│  │  ┌─────────────┐  ┌──────────┐  ┌─────────┐  ┌────────────┐  │  │
│  │  │ ElevenLabs  │  │ OpenAI   │  │ Edge    │  │ Sherpa-ONNX│  │  │
│  │  │ (Cloud)     │  │ TTS      │  │ (Free)  │  │ (Local)    │  │  │
│  │  │             │  │ (Cloud)  │  │         │  │            │  │  │
│  │  │ 11 models   │  │ 3 models │  │ MS      │  │ Piper VITS │  │  │
│  │  │ Voice clone │  │ 14 voices│  │ Voices  │  │ models     │  │  │
│  │  │ stability/  │  │ instruct │  │ No key  │  │ No cloud   │  │  │
│  │  │ similarity  │  │ (mini)   │  │ needed  │  │ needed     │  │  │
│  │  └──────┬──────┘  └────┬─────┘  └────┬────┘  └─────┬──────┘  │  │
│  │         └───────────┬──┴─────────────┴──────────────┘         │  │
│  │                     ▼                                          │  │
│  │        Output Format Selection                                 │  │
│  │        TG/Feishu/WA → opus (voice bubble)                     │  │
│  │        Others → mp3 (audio file)                               │  │
│  │        Telephony → pcm/mu-law 8kHz                            │  │
│  │                                                                 │  │
│  │  Triggers: Agent tool │ Auto-mode │ /tts command │ Gateway    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │            Voice Call Extension (extensions/voice-call/)        │  │
│  │                                                                 │  │
│  │  ┌─────────────────────────────────────────────────────────┐   │  │
│  │  │                  Call Manager                             │   │  │
│  │  │                                                          │   │  │
│  │  │  State Machine:                                          │   │  │
│  │  │  initiated → ringing → answered → active                 │   │  │
│  │  │                                     ↕                    │   │  │
│  │  │                              speaking ↔ listening         │   │  │
│  │  │                                     ↓                    │   │  │
│  │  │                              [terminal states]           │   │  │
│  │  │                                                          │   │  │
│  │  │  Storage: ~/.openclaw/voice-calls/calls.jsonl            │   │  │
│  │  │  Max concurrent: 1 (default)                             │   │  │
│  │  │  Max duration: 300s (default)                            │   │  │
│  │  └─────────────────────────────────────────────────────────┘   │  │
│  │                                                                 │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │  │
│  │  │ Twilio   │  │ Telnyx   │  │ Plivo    │  │ Mock         │  │  │
│  │  │          │  │          │  │          │  │ (testing)    │  │  │
│  │  │ HMAC-SHA1│  │ Ed25519  │  │HMAC-SHA256│ │              │  │  │
│  │  │ TwiML    │  │ REST v2  │  │ XML flow │  │              │  │  │
│  │  │ <Stream> │  │ Bearer   │  │ Basic    │  │              │  │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │  │
│  │                                                                 │  │
│  │  ┌─────────────────────────────────────────────────────────┐   │  │
│  │  │              Media Streaming Pipeline                     │   │  │
│  │  │                                                          │   │  │
│  │  │  Phone → Twilio → WebSocket → OpenAI Realtime STT       │   │  │
│  │  │       → Transcript → Pi Agent → TTS → mu-law 8kHz       │   │  │
│  │  │       → 20ms chunks → WebSocket → Twilio → Phone        │   │  │
│  │  │                                                          │   │  │
│  │  │  Limits: 32 pending, 4/IP, 128 total connections         │   │  │
│  │  └─────────────────────────────────────────────────────────┘   │  │
│  │                                                                 │  │
│  │  Webhook: port 3334 │ Tunnel: ngrok/tailscale │ CLI commands  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │           Discord Voice (src/discord/voice/)                   │  │
│  │                                                                 │  │
│  │  /voice join → VoiceConnection → Speaking Detection            │  │
│  │  → Opus Decode → WAV → Whisper STT → Agent → TTS              │  │
│  │  → AudioResource → AudioPlayer → Discord Voice Channel         │  │
│  │                                                                 │  │
│  │  48kHz stereo │ DAVE encryption │ Auto-reconnect               │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/tts/tts-core.ts` | 719 | Backend functions (ElevenLabs/OpenAI/Edge), directives, config types |
| `src/tts/tts.ts` | 1,004 | Main TTS engine, auto-mode, format selection, user prefs |
| `src/tts/tts.test.ts` | 748 | Comprehensive TTS tests |
| `src/tts/prepare-text.test.ts` | 68 | Markdown stripping tests |
| `src/config/types.tts.ts` | 90 | TTS configuration schema |
| `src/agents/tools/tts-tool.ts` | 62 | Agent TTS tool |
| `src/gateway/server-methods/tts.ts` | 157 | Gateway TTS API |
| `src/auto-reply/reply/commands-tts.ts` | 280 | `/tts` chat commands |
| `extensions/voice-call/index.ts` | 565 | Voice-call extension entry |
| `extensions/voice-call/src/types.ts` | 305 | Call types, state machine, events |
| `extensions/voice-call/src/config.ts` | 526 | Voice-call configuration |
| `extensions/voice-call/src/manager.ts` | 319 | Call manager |
| `extensions/voice-call/src/manager/state.ts` | 49 | State transitions |
| `extensions/voice-call/src/manager/events.ts` | 259 | Event processing pipeline |
| `extensions/voice-call/src/manager/outbound.ts` | 381 | Outbound calls, speaking, multi-turn |
| `extensions/voice-call/src/manager/store.ts` | 95 | JSONL persistence |
| `extensions/voice-call/src/manager/timers.ts` | 113 | Max duration and transcript timeouts |
| `extensions/voice-call/src/media-stream.ts` | 528 | WebSocket media streaming |
| `extensions/voice-call/src/telephony-audio.ts` | 91 | PCM→mu-law conversion, resampling |
| `extensions/voice-call/src/telephony-tts.ts` | 83 | TTS for telephony (mu-law output) |
| `extensions/voice-call/src/response-generator.ts` | 159 | AI response via embedded Pi agent |
| `extensions/voice-call/src/webhook.ts` | 488 | Webhook server |
| `extensions/voice-call/src/webhook-security.ts` | 981 | Signature verification, replay protection |
| `extensions/voice-call/src/tunnel.ts` | 315 | ngrok/Tailscale tunneling |
| `extensions/voice-call/src/allowlist.ts` | 20 | Caller allowlisting |
| `extensions/voice-call/src/voice-mapping.ts` | 68 | OpenAI→Polly voice mapping |
| `extensions/voice-call/src/providers/base.ts` | 79 | Provider interface |
| `extensions/voice-call/src/providers/twilio.ts` | 709 | Twilio implementation |
| `extensions/voice-call/src/providers/telnyx.ts` | 358 | Telnyx implementation |
| `extensions/voice-call/src/providers/plivo.ts` | 594 | Plivo implementation |
| `extensions/voice-call/src/providers/stt-openai-realtime.ts` | ~303 | OpenAI Realtime STT |
| `extensions/voice-call/src/providers/tts-openai.ts` | ~264 | OpenAI TTS for telephony |
| `extensions/voice-call/src/cli.ts` | 381 | CLI commands |
| `extensions/voice-call/src/runtime.ts` | 265 | Runtime initialization |
| `extensions/voice-call/src/core-bridge.ts` | 160 | Bridge to core agent system |
| `src/discord/voice/command.ts` | 374 | Discord voice slash commands |
| `src/discord/voice/manager.ts` | 903 | Discord voice channel manager |
| `skills/sherpa-onnx-tts/SKILL.md` | 104 | Local TTS skill documentation |
