# OpenClaw Architecture — Deep Dive Documentation

> A comprehensive, layer-by-layer technical deep dive into the [OpenClaw](https://github.com/openclaw/openclaw) self-hosted AI assistant — explained like you're 12 years old, with full git history evolution for every layer.

---

## What Is OpenClaw?

OpenClaw is a **self-hosted, local-first personal AI assistant** that connects to messaging platforms (WhatsApp, Telegram, Discord, Slack, Signal, iMessage, and more) and runs on your own devices. It's a monorepo built with TypeScript (Node.js 22+), Swift (Apple apps), and Kotlin (Android).

## Architecture Overview

The system is organized into 12 architectural layers, from the foundational runtime up to deployment infrastructure:

| Layer | Name | Required? | Description |
|-------|------|-----------|-------------|
| 1 | [Core Runtime](layers/layer-1-core-runtime.md) | **Yes** | Boot sequence, config system, session management, routing |
| 2 | [Gateway](layers/layer-2-gateway.md) | **Yes** | Central nervous system — HTTP/WebSocket server, message orchestration |
| 3 | [Plugin System](layers/layer-3-plugin-system.md) | **Yes** | Extension loading, lifecycle management, SDK |
| 4 | [Channel Integrations](layers/layer-4-channel-integrations.md) | Need ≥1 | Telegram, WhatsApp, Discord, Slack, Signal, iMessage, and 16 more |
| 5 | [AI Providers](layers/layer-5-ai-providers.md) | Need ≥1 | OpenAI, Anthropic, Google, Ollama, Bedrock, and 50+ more |
| 6 | [Skills](layers/layer-6-skills.md) | Optional | 53 bundled SKILL.md documentation files teaching the AI capabilities |
| 7 | [Memory System](layers/layer-7-memory-system.md) | Optional | RAG-based long-term memory with embeddings, SQLite, hybrid search |
| 8 | [Context Engine](layers/layer-8-context-engine.md) | Optional | Pluggable conversation context management, compaction, overflow recovery |
| 9 | [Voice / TTS](layers/layer-9-voice-tts.md) | Optional | Text-to-speech (4 backends), telephony (3 providers), Discord voice |
| 10 | [Canvas & A2UI](layers/layer-10-canvas-a2ui.md) | Optional | Interactive UI rendering via HTML canvas and Google's A2UI protocol |
| 11 | [Companion Apps](layers/layer-11-companion-apps.md) | Optional | macOS menu bar, iOS, Android thin-client apps |
| 12 | [Infrastructure](layers/layer-12-infrastructure.md) | Optional | Docker, daemon (launchd/systemd/schtasks), sandbox, Tailscale, cloud deploy |

## How to Read These Docs

Each layer document follows the same structure:

1. **ELI12 Explanation** — What the layer does, explained simply
2. **Detailed Technical Breakdown** — Every type, function, and algorithm documented with line numbers
3. **Git History Evolution** — How the layer evolved over time, who built it, and why decisions were made
4. **Architecture Diagram** — ASCII diagram showing component relationships
5. **Key Files Reference** — Quick-reference table of important source files

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

*Generated with deep research into the OpenClaw source code and git history.*
