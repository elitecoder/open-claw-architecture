# Layer 5: AI Providers — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Layer 5?

If the channels (Layer 4) are the robot's ears and mouth, then the AI providers are its **actual brain cells**. When someone sends a message, the robot needs to *think* about it and come up with a response. That thinking is done by a large language model (LLM) — like Claude, GPT, or Gemini — running on some server somewhere.

Layer 5 is the system that lets OpenClaw talk to **any** AI brain. Want to use Anthropic's Claude? OpenAI's GPT? Google's Gemini? A model running on your own laptop with Ollama? A model on AWS Bedrock? OpenClaw doesn't care — it has a universal translator for all of them.

Think of it like a universal TV remote. Each TV brand uses a different signal, but the universal remote knows how to talk to all of them. Layer 5 is that universal remote for AI brains.

Layer 5 has **seven major parts**:

1. **Model & Provider Types** — The data structures that describe AI brains
2. **Provider Discovery** — Finding which AI brains are available
3. **Authentication** — Proving you're allowed to use each brain
4. **Model Selection & Routing** — Picking the right brain for the job
5. **The Inference Pipeline** — Actually sending messages to the brain and getting responses
6. **Usage Tracking & Cost** — Counting tokens and dollars
7. **Provider Capabilities & Compatibility** — Handling each brain's quirks

---

## Part 1: Model & Provider Types (The Data Structures)

### What Is a "Provider" vs a "Model"?

A **provider** is a company or service that hosts AI models. Think of it like a TV network (NBC, Netflix, HBO). A **model** is a specific AI brain offered by that provider. Think of it like a specific TV show on that network.

```
Provider: Anthropic          Provider: OpenAI          Provider: Ollama (local)
  ├── claude-opus-4-6          ├── gpt-5.4                ├── llama3
  ├── claude-sonnet-4.5        ├── gpt-5.2                ├── mistral
  └── claude-haiku-4.5         └── gpt-5-mini             └── codellama
```

### API Protocols — The Languages AI Brains Speak

**File: `src/config/types.models.ts`**

Not all AI brains speak the same language. OpenClaw supports **8 different API protocols**:

```typescript
export const MODEL_APIS = [
  "openai-completions",       // The most common — OpenAI's chat/completions format
  "openai-responses",         // OpenAI's newer responses API
  "openai-codex-responses",   // ChatGPT/Codex backend API
  "anthropic-messages",       // Anthropic's native Messages API
  "google-generative-ai",     // Google's Generative AI API
  "github-copilot",           // GitHub Copilot's API
  "bedrock-converse-stream",  // AWS Bedrock's streaming API
  "ollama",                   // Ollama's native /api/chat API
] as const;
```

**Why so many?** Because each AI company designed their API differently. OpenAI was first, so many others copied their format (that's why `openai-completions` is the most common). But Anthropic, Google, and AWS each have their own formats with unique features. Ollama even has its own native protocol that's more efficient for local models.

Most providers fall into one of two camps:
- **OpenAI-compatible** (`openai-completions`): Ollama, vLLM, SGLang, Moonshot, Together, HuggingFace, OpenRouter, NVIDIA, and many more
- **Anthropic-compatible** (`anthropic-messages`): MiniMax, Kimi Coding, Xiaomi, and others that adopted Anthropic's format

### Model Definition — What Describes a Single AI Brain

```typescript
export type ModelDefinitionConfig = {
  id: string;                       // "claude-opus-4-6"
  name: string;                     // "Claude Opus 4.6"
  api?: ModelApi;                   // Override provider's default protocol
  reasoning: boolean;               // Can it "think out loud"? (chain-of-thought)
  input: Array<"text" | "image">;   // What can it accept?
  cost: {
    input: number;                  // $/million input tokens
    output: number;                 // $/million output tokens
    cacheRead: number;              // $/million cached input tokens
    cacheWrite: number;             // $/million cache creation tokens
  };
  contextWindow: number;            // Max tokens it can "see" at once
  maxTokens: number;                // Max tokens it can generate
  headers?: Record<string, string>; // Custom HTTP headers
  compat?: ModelCompatConfig;       // Quirks and compatibility flags
};
```

**Context window** is like the AI's short-term memory. A 200,000-token context window means the AI can "see" about 150,000 words of conversation at once. Bigger is better (but more expensive).

**Cost** is measured per million tokens. A token is roughly 3/4 of a word. So if a model costs $3/million input tokens, processing a 1,000-word message costs about $0.004 (less than half a cent).

### Provider Configuration — Where to Find an AI Brain

```typescript
export type ModelProviderConfig = {
  baseUrl: string;                          // "https://api.anthropic.com"
  apiKey?: SecretInput;                     // The password/key
  auth?: "api-key" | "aws-sdk" | "oauth" | "token";  // How to authenticate
  api?: ModelApi;                           // Default protocol for all models
  injectNumCtxForOpenAICompat?: boolean;    // Ollama-specific: inject context window
  headers?: Record<string, SecretInput>;    // Custom HTTP headers
  authHeader?: boolean;                     // Use Authorization header?
  models: ModelDefinitionConfig[];          // Available models
};
```

### Model Compatibility Flags — Handling Each Brain's Quirks

This is where it gets interesting. Every AI provider has slightly different behavior, even when they use the same protocol. The compatibility flags handle these differences:

```typescript
export type ModelCompatConfig = {
  supportsStore?: boolean;                     // Can store conversations?
  supportsDeveloperRole?: boolean;             // Supports "developer" role?
  supportsReasoningEffort?: boolean;           // Can adjust thinking depth?
  supportsUsageInStreaming?: boolean;           // Reports token usage while streaming?
  supportsTools?: boolean;                      // Can use tools/functions?
  supportsStrictMode?: boolean;                 // Strict validation?
  maxTokensField?: "max_completion_tokens" | "max_tokens";  // API field name differs!
  thinkingFormat?: "openai" | "zai" | "qwen";  // How reasoning blocks are formatted
  requiresToolResultName?: boolean;             // Mistral needs this
  requiresAssistantAfterToolResult?: boolean;   // Some providers need filler messages
  requiresMistralToolIds?: boolean;             // Mistral's tool IDs are max 9 chars
  requiresThinkingAsText?: boolean;             // Some providers can't handle thinking blocks
  requiresOpenAiAnthropicToolPayload?: boolean; // Need tool format conversion
};
```

**Real example:** OpenAI uses the field name `max_completion_tokens` for the maximum output length. Anthropic uses `max_tokens`. Same concept, different name. The `maxTokensField` flag tells OpenClaw which name to use.

**Another example:** Mistral's API requires tool call IDs to be exactly 9 characters. Other providers allow longer IDs. The `requiresMistralToolIds` flag triggers special ID truncation.

### Top-Level Models Configuration

```typescript
export type ModelsConfig = {
  mode?: "merge" | "replace";                         // Merge with built-ins or replace entirely?
  providers?: Record<string, ModelProviderConfig>;     // Your custom providers
  bedrockDiscovery?: BedrockDiscoveryConfig;           // AWS auto-discovery settings
};
```

The `mode` field is important:
- **`merge`** (default): Your providers are added on top of the built-in ones. If you define a model that already exists, yours wins.
- **`replace`**: Throw away all built-in providers and use ONLY yours. For advanced users who want total control.

---

## Part 2: Provider Discovery (Finding Available AI Brains)

### The Discovery Problem

OpenClaw can't just assume which providers you have. You might have an OpenAI API key. You might have Ollama running locally. You might have AWS Bedrock configured. OpenClaw needs to figure out what's available.

### The Discovery Pipeline

**File: `src/agents/models-config.providers.ts` (~1,000 lines)**

Discovery runs in **7 ordered phases**:

```
Phase 1: Simple Loaders ──── API key providers (check env vars)
    │    minimax, moonshot, kimi-coding, venice, xiaomi,
    │    together, huggingface, qianfan, openrouter, nvidia, ...
    │
Phase 2: Plugin Loaders ──── Plugin-registered providers
    │    (extensions like ollama, vllm, sglang)
    │
Phase 3: Profile Loaders ─── OAuth-based providers
    │    minimax-portal, qwen-portal, openai-codex
    │
Phase 4: Paired Loaders ──── Multi-variant providers
    │    volcengine + volcengine-plan (regular + coding)
    │    byteplus + byteplus-plan
    │
Phase 5: Cloudflare AI Gateway ── Profile-based
    │
Phase 6: Plugin Late Loaders ──── Late-phase plugin discovery
    │
Phase 7: Special Providers ─── GitHub Copilot + Amazon Bedrock
```

### Phase 1: Simple Loaders (Check Your API Keys)

The simplest discovery: "Do you have an API key for this provider?" Each provider has a known environment variable:

| Provider | Environment Variable |
|----------|---------------------|
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google | `GOOGLE_API_KEY` or `GEMINI_API_KEY` |
| MiniMax | `MINIMAX_API_KEY` |
| Moonshot | `MOONSHOT_API_KEY` |
| Together | `TOGETHER_API_KEY` |
| HuggingFace | `HUGGINGFACE_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |

If the env var exists, the provider is added with its predefined models and base URL.

### Phase 2: Plugin Discovery

Provider plugins (like Ollama, vLLM, SGLang) register themselves through the plugin system. They can dynamically discover models at runtime.

**Ollama Discovery Example:**
```typescript
async function discoverOllamaModels(baseUrl?: string): Promise<ModelDefinitionConfig[]>
// 1. Call GET /api/tags to list all pulled models
// 2. For each model, call POST /api/show to get context window size
// 3. Build ModelDefinitionConfig for each
// Max 200 models, concurrent requests
```

**vLLM/SGLang Discovery:**
```typescript
async function discoverOpenAICompatibleLocalModels(params: {
  baseUrl: string;     // "http://127.0.0.1:8000/v1"
  apiKey?: string;
  label: string;       // "vLLM" or "SGLang"
}): Promise<ModelDefinitionConfig[]>
// Call GET /v1/models (OpenAI-compatible endpoint)
// Build model definitions from response
```

### Phase 7: Special Providers

**GitHub Copilot** is discovered by checking for GitHub tokens (`COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`) and resolving the Copilot proxy URL.

**Amazon Bedrock** uses the AWS SDK to auto-discover available models:
```typescript
type BedrockDiscoveryConfig = {
  enabled?: boolean;              // Turn on auto-discovery
  region?: string;                // AWS region (default: us-east-1)
  providerFilter?: string[];      // Only show certain providers, e.g., ["anthropic"]
  refreshInterval?: number;       // How often to re-scan
  defaultContextWindow?: number;
  defaultMaxTokens?: number;
};
```

It calls the `ListFoundationModels` API to find all models available in your AWS account.

---

## Part 3: Authentication (Proving You're Allowed)

### The Auth Challenge

Every AI provider requires authentication — you need to prove you're a paying customer. But different providers use different auth methods:

- **API Key**: A static string (like a password). Most common.
- **OAuth**: A web-based login flow with refresh tokens. More secure.
- **Device Code**: You get a code, visit a URL, enter the code. Like pairing a TV.
- **AWS SDK**: Uses your AWS credentials (environment variables or config files).

### The Auth Resolution Waterfall

**File: `src/agents/model-auth.ts` (521 lines)**

When OpenClaw needs to authenticate with a provider, it tries **7 sources** in order:

```
1. Explicit profile ID ──── "Use this specific auth profile"
       │ (not found)
2. Auth override ──────── AWS SDK mode
       │ (not found)
3. Auth profile store ──── Saved credentials (ordered by preference)
       │ (not found)
4. Environment variable ── OPENAI_API_KEY, ANTHROPIC_API_KEY, etc.
       │ (not found)
5. Custom provider key ── From models.json config
       │ (not found)
6. Synthetic local auth ── For localhost providers (Ollama, vLLM)
       │ (not found)
7. Amazon Bedrock default ── AWS SDK credential chain
       │ (not found)
       └──→ ERROR: No credentials found
```

The result is a `ResolvedProviderAuth`:

```typescript
export type ResolvedProviderAuth = {
  apiKey?: string;     // The actual credential
  profileId?: string;  // Which profile it came from
  source: string;      // How it was found (e.g., "env: OPENAI_API_KEY")
  mode: "api-key" | "oauth" | "token" | "aws-sdk";
};
```

### Auth Profiles — Saved Credentials with Rotation

**File: `src/agents/auth-profiles.ts`**

Auth profiles are like saved passwords in your browser, but smarter:

```typescript
export type AuthProfileStore = {
  profiles: Record<string, AuthProfileCredential>;  // Saved credentials
  order?: string[];                                  // Preference order
  usageStats?: Record<string, ProfileUsageStats>;    // How often each is used
  failures?: Record<string, AuthProfileFailureReason>; // Which ones failed
  cooldowns?: Record<string, { until: number }>;     // Temporarily disabled
};
```

**Why multiple profiles?** Imagine you have two OpenAI API keys — one personal and one from work. If the personal one hits its rate limit, OpenClaw automatically switches to the work one. This is **auth profile rotation**.

Profile ordering is smart:
1. Explicitly configured order (if you set one)
2. Round-robin based on last used
3. Stored order from previous sessions
4. Profile creation order (oldest first)

**Cooldowns and failures:** If a profile fails (bad key, billing issue, rate limit), it's put on cooldown. The system tries the next profile instead of hammering a broken one.

### OAuth Flows

Three provider extensions implement full OAuth:

#### Google Gemini CLI Auth (PKCE + Localhost Callback)
```
1. Generate PKCE code verifier + challenge
2. Open browser to Google's auth page
3. User logs in and grants permission
4. Google redirects to http://localhost:8085/oauth2callback
5. Exchange auth code for access + refresh tokens
6. Store tokens in auth profile
```

#### MiniMax & Qwen Portal Auth (Device Code Flow)
```
1. Request a device code from the provider
2. Show user: "Go to https://... and enter code: ABCD-1234"
3. Poll the provider every 2-10 seconds: "Did they enter the code yet?"
4. When they do, receive access + refresh tokens
5. Store tokens in auth profile
```

### Secret Management

OpenClaw never stores plaintext secrets in config files if it can help it:

```typescript
// Environment reference (stored as-is)
apiKey: "env:OPENAI_API_KEY"

// Keychain reference (macOS Keychain, etc.)
apiKey: "keychain:openclaw-openai"

// Direct value (only if user explicitly sets it)
apiKey: "sk-abc123..."
```

When writing `models.json`, the system:
- Preserves env references as-is
- Replaces non-env secrets with placeholder markers
- Never persists plaintext secrets from temporary sources

---

## Part 4: Model Selection & Routing (Picking the Right Brain)

### The Model Reference

**File: `src/agents/model-selection.ts` (728 lines)**

Every model is identified by a **provider/model** pair:

```typescript
export type ModelRef = {
  provider: string;    // "anthropic"
  model: string;       // "claude-opus-4-6"
};
```

In config, you write this as a string: `"anthropic/claude-opus-4-6"`.

### Provider ID Normalization

Users might type provider names in many ways. OpenClaw normalizes them:

```typescript
"z.ai" or "z-ai"           → "zai"
"opencode-zen"              → "opencode"
"qwen"                      → "qwen-portal"
"kimi-code"                 → "kimi-coding"
"bedrock" or "aws-bedrock"  → "amazon-bedrock"
"bytedance" or "doubao"     → "volcengine"
```

### Model ID Normalization

Similarly, model names get normalized:

```typescript
// Anthropic aliases
"opus-4.6"                  → "claude-opus-4-6"

// Google preview versions
"gemini-3-pro"              → "gemini-3-pro-preview"

// OpenRouter prefix
"aurora-alpha"              → "openrouter/aurora-alpha"
```

### The Default Model

If you don't specify a model, OpenClaw has a hardcoded default:

```typescript
const DEFAULT_MODEL = "claude-opus-4-6";
const DEFAULT_PROVIDER = "anthropic";
```

### Model Resolution Flow

When the system needs to pick a model:

```
1. Check agent-specific model override
   (each agent can have its own preferred model)
       │
2. Check configured default model
   (from config.json5)
       │
3. Parse "provider/model" format
       │
4. Check model alias index
   (case-insensitive lookup)
       │
5. Verify provider exists and has the model
       │
6. If provider unavailable, try fallback providers
       │
7. Ultimate fallback: anthropic/claude-opus-4-6
```

### Model Allowlists

You can restrict which models are available:

```typescript
function buildAllowedModelSet(): Set<string>
// If no allowlist configured: allow ALL catalog + fallback models
// If allowlist configured: only allow those specific models
// The default model is ALWAYS allowed (can't lock yourself out)
```

### Thinking Levels

For reasoning models (models that "think out loud"), you can control how much thinking they do:

```typescript
export type ThinkLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "adaptive";
```

- **`off`**: No thinking, just answer immediately
- **`minimal`** to **`xhigh`**: Increasing amounts of reasoning (more tokens, better answers, higher cost)
- **`adaptive`**: Let the model decide how much thinking is needed

Resolution order:
1. Per-model override from config
2. Global default setting
3. Catalog recommendation for that model

---

## Part 5: The Inference Pipeline (Actually Talking to the Brain)

This is the heart of Layer 5 — the code that actually sends your message to an AI model and streams back the response.

### The Core Runtime: pi-embedded-runner

**Directory: `src/agents/pi-embedded-runner/`**

OpenClaw uses a library called `@mariozechner/pi-ai` (and related packages `pi-coding-agent`, `pi-agent-core`) as the foundation for AI inference. The `pi-embedded-runner` wraps this library with OpenClaw-specific logic.

### The Run Loop

**File: `src/agents/pi-embedded-runner/run.ts`**

The run loop is the master controller. It handles retries, auth failover, context management, and error recovery.

```
┌─────────────────────────────────────────────┐
│                  RUN LOOP                    │
│                                              │
│  Retry Budget: 24 base + 8 per auth profile  │
│  Max: 160 iterations                         │
│                                              │
│  For each attempt:                           │
│    1. Resolve auth (pick profile)            │
│    2. Check context window guard             │
│    3. Build session + system prompt          │
│    4. Construct tool schemas                 │
│    5. Call streamSimple() ──────────────┐    │
│    6. Process streaming response       │    │
│    7. Accumulate usage tokens          │    │
│                                        │    │
│  On failure:                           │    │
│    - Auth error? → next profile        │    │
│    - Rate limit? → backoff + retry     │    │
│    - Billing error? → next profile     │    │
│    - Context overflow? → compact + retry│    │
│    - Network error? → retry            │    │
│                                        │    │
│  Backoff for overload:                 │    │
│    initial: 250ms                      │    │
│    max: 1,500ms                        │    │
│    factor: 2x                          │    │
│    jitter: ±20%                        │    │
└────────────────────────────────────────┘
```

Key constants:
```typescript
const BASE_RUN_RETRY_ITERATIONS = 24;
const RUN_RETRY_ITERATIONS_PER_PROFILE = 8;   // 8 extra retries per auth profile
const MAX_RUN_RETRY_ITERATIONS = 160;          // Absolute ceiling
```

### A Single Inference Attempt

**File: `src/agents/pi-embedded-runner/run/attempt.ts`**

Each attempt within the run loop does:

1. **Create session** from pi-coding-agent SDK
2. **Build system prompt** with agent personality, skills, context
3. **Construct tool schemas** — validate and transform for the target provider
4. **Apply provider-specific transforms:**
   - Google: sanitize tool schemas, fix turn ordering
   - Anthropic: wrap payloads when going through OpenAI-format providers
   - Mistral: truncate tool IDs to 9 characters
5. **Map reasoning/thinking level** to provider-specific format
6. **Sanitize images** — check dimensions, format
7. **Set cache TTL** for prompt caching (Anthropic, Google)
8. **Load plugin extensions** — pre/post hooks
9. **Call `streamSimple()`** — the actual API call

### The Core Streaming Call

```typescript
import { streamSimple } from "@mariozechner/pi-ai";

const result = await streamSimple({
  model: resolvedModel,           // The AI model to use
  messages: preparedMessages,     // Conversation history
  tools: sdkTools,                // Available tool definitions
  temperature?: number,           // Creativity (0 = deterministic, 1 = creative)
  maxTokens?: number,             // Max response length
  thinking?: {                    // Reasoning configuration
    type: "enabled",
    budgetTokens: number          // How many tokens for thinking
  },
}, onStream);  // Callback for each streamed chunk
```

### Model Resolution at Runtime

**File: `src/agents/pi-embedded-runner/model.ts`**

When resolving a model for actual use:

```
1. Check Pi SDK registry (built-in models from the library)
       │
2. If not found, check configured inline models (from models.json)
       │
3. If OpenRouter, allow ANY model ID (pass-through)
       │
4. Apply forward-compatibility aliases
       │
5. Apply configured overrides (custom headers, cost, context window)
       │
6. Return Model<Api> object ready for streaming
```

### Stream Wrappers — Provider-Specific Streaming

Different providers stream responses differently. OpenClaw has specialized stream wrappers:

| Wrapper | File | Purpose |
|---------|------|---------|
| OpenAI HTTP | `openai-stream-wrappers.ts` | Standard SSE streaming |
| OpenAI WebSocket | `openai-ws-stream.ts` | WebSocket-first (faster) |
| Anthropic | `anthropic-stream-wrappers.ts` | Anthropic's event stream |
| Moonshot | `moonshot-stream-wrappers.ts` | Moonshot/Kimi-specific |
| Proxy | `proxy-stream-wrappers.ts` | Generic proxy wrapper |

The OpenAI WebSocket streaming deserves special mention — it's **WebSocket-first with HTTP fallback**. WebSocket is faster because it reuses a persistent connection instead of creating a new HTTP connection for each request. If WebSocket fails, it automatically falls back to standard HTTP streaming.

```
WebSocket (preferred, faster)
    │ fails?
    └──→ HTTP SSE (fallback, always works)
```

### Context Window Guard

**File: `src/agents/context-window-guard.ts`**

The context window guard prevents the "conversation too long" crash:

```typescript
CONTEXT_WINDOW_HARD_MIN_TOKENS = 10_000;    // Absolute minimum
CONTEXT_WINDOW_WARN_BELOW_TOKENS = 15_000;  // Warning threshold
```

Before each attempt, it checks: "Is the conversation approaching the model's context limit?" If yes, it triggers **compaction** — summarizing older messages to free up space. This is done in collaboration with Layer 8 (Context Engine).

The guard uses the **last API call's cache fields** (not accumulated totals) to avoid inflation from multi-turn tool calls. If the AI makes 5 tool calls in a row, each reports `cacheRead ≈ current_context_size`. Summing all 5 would give 5× the real context usage. The guard smartly uses only the most recent measurement.

---

## Part 6: Usage Tracking & Cost (Counting Tokens and Dollars)

### The Token Counting Problem

Every AI provider reports token usage differently. OpenClaw normalizes everything into a common format.

**File: `src/agents/usage.ts` (191 lines)**

### Raw Provider Usage — The Babel Tower

```typescript
export type UsageLike = {
  // Standard fields
  input?: number;
  output?: number;
  cacheRead?: number;
  cacheWrite?: number;
  total?: number;

  // OpenAI-style aliases
  promptTokens?: number;
  prompt_tokens?: number;
  completionTokens?: number;
  completion_tokens?: number;

  // Anthropic-style aliases
  inputTokens?: number;
  outputTokens?: number;
  cache_read_input_tokens?: number;
  cache_creation_input_tokens?: number;

  // Moonshot/Kimi-style
  cached_tokens?: number;
  prompt_tokens_details?: { cached_tokens?: number };  // Kimi K2

  // Generic aliases
  totalTokens?: number;
  total_tokens?: number;
};
```

That's **20+ different field names** for basically 4 numbers: input tokens, output tokens, cache reads, cache writes. Every provider picked different names.

### Normalized Usage — The Common Language

```typescript
export type NormalizedUsage = {
  input?: number;       // Tokens the AI "read"
  output?: number;      // Tokens the AI "wrote"
  cacheRead?: number;   // Tokens read from cache (cheaper!)
  cacheWrite?: number;  // Tokens written to cache (for future savings)
  total?: number;
};
```

The `normalizeUsage()` function handles all the conversions, including edge cases:
- **Negative input tokens**: When Moonshot reports `cached_tokens > prompt_tokens`, input goes negative. The normalizer clamps to 0.
- **Missing totals**: Computed as `input + output + cacheRead + cacheWrite`.

### Full Usage Snapshot with Costs

```typescript
export type AssistantUsageSnapshot = {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;       // Dollars spent on input
    output: number;      // Dollars spent on output
    cacheRead: number;   // Dollars saved via cache reads
    cacheWrite: number;  // Dollars spent on cache creation
    total: number;       // Total dollars
  };
};
```

Cost is calculated by multiplying token counts by the model's per-million-token prices from the `ModelDefinitionConfig.cost` field.

### Cost Tracking Fixes (A Story of Getting It Right)

The git history reveals how tricky cost tracking was:
- **Feb 21**: "drop stale pre-compaction usage snapshots" — old usage data was contaminating totals
- **Feb 26**: "clamp negative input token counts to zero" — Moonshot bug
- **Mar 4**: "parse Kimi K2 cached_tokens from prompt_tokens_details" — Kimi changed their format
- **Mar 12**: "stop guessing session model costs" — removed speculative cost estimation that was inaccurate
- **Mar 12**: "correct MiniMax M2.5 pricing (was ~50x too high)" — a decimal point error
- **Mar 13**: "cache OpenRouter pricing for configured models" — OpenRouter's dynamic pricing

---

## Part 7: Provider Capabilities & Compatibility (Handling Quirks)

### The Provider Capabilities System

**File: `src/agents/provider-capabilities.ts` (169 lines)**

Each provider has unique behaviors. The capabilities system tracks these:

```typescript
export type ProviderCapabilities = {
  // Tool schema format
  anthropicToolSchemaMode: "native" | "openai-functions";
  anthropicToolChoiceMode: "native" | "openai-string-modes";

  // Provider family (affects many behaviors)
  providerFamily: "default" | "openai" | "anthropic";

  // Reasoning/thinking block handling
  preserveAnthropicThinkingSignatures: boolean;
  geminiThoughtSignatureSanitization: boolean;
  dropThinkingBlockModelHints: string[];

  // Turn validation (message ordering rules)
  openAiCompatTurnValidation: boolean;

  // Tool call ID format
  transcriptToolCallIdMode: "default" | "strict9";
  transcriptToolCallIdModelHints: string[];
};
```

### Provider-Specific Quirks

| Provider | Quirk | How OpenClaw Handles It |
|----------|-------|------------------------|
| **Mistral** | Tool call IDs must be ≤9 characters | `transcriptToolCallIdMode: "strict9"` — truncates IDs |
| **Anthropic/Bedrock** | Thinking blocks must be dropped on replay | `dropThinkingBlockModelHints: ["claude"]` |
| **Kimi Coding** | Preserves Anthropic thinking signatures | `preserveAnthropicThinkingSignatures: true` |
| **OpenRouter** | Gemini models have thought signature artifacts | `geminiThoughtSignatureSanitization: true` |
| **OpenAI** | Requires specific turn ordering | `openAiCompatTurnValidation: true` |
| **Google** | Tool schemas need sanitization | Special transforms in attempt.ts |

### Transcript Hygiene — Cleaning Up Conversation History

**File: `src/agents/pi-embedded-runner/transcript-hygiene.ts` (470 lines)**

When replaying conversation history to the AI, different providers have different rules about what's valid. Transcript hygiene cleans up the history:

- **Drop orphaned reasoning blocks** — If a previous response had thinking blocks, some providers can't handle them on replay
- **Sanitize Gemini thought signatures** — Google's models leave artifacts that confuse other providers
- **Self-heal Anthropic replay thinking** — Fix broken thinking block structure
- **Ensure proper turn ordering** — Some providers require strict assistant→user→assistant alternation

---

## Part 8: The Complete Provider Ecosystem (All 50+ Providers)

### Tier 1: Major Cloud Providers

| Provider | Protocol | Auth | Key Models | Context | Special Features |
|----------|----------|------|-----------|---------|-----------------|
| **Anthropic** | anthropic-messages | API key | Claude Opus 4.6, Sonnet 4.5, Haiku 4.5 | 200K | Prompt caching, thinking blocks |
| **OpenAI** | openai-completions | API key | GPT-5.4, GPT-5.2, GPT-5-mini | 128K | WebSocket streaming, function calling |
| **Google** | google-generative-ai | API key / OAuth | Gemini 3 Pro, Gemini 3 Flash | 1M+ | Massive context, multimodal |
| **AWS Bedrock** | bedrock-converse-stream | AWS SDK | All hosted models | Varies | Auto-discovery, multi-model |
| **GitHub Copilot** | github-copilot | GitHub token | Copilot models | Varies | Code-focused |

### Tier 2: Regional/Specialized Cloud Providers

| Provider | Protocol | Region | Key Models |
|----------|----------|--------|-----------|
| **MiniMax** | anthropic-messages | China/Global | MiniMax-M2.5 (reasoning) |
| **Moonshot/Kimi** | openai-completions | China | Kimi K2.5 (256K context!) |
| **Kimi Coding** | anthropic-messages | China | K2P5 (coding-focused, reasoning) |
| **Qwen Portal** | openai-completions | China (Aliyun) | Coder model, Vision model |
| **Qianfan (Baidu)** | openai-completions | China | DeepSeek-v3.2, Ernie 5.0 |
| **ModelStudio (Aliyun)** | openai-completions | China | Qwen 3.5 Plus (1M context!) |
| **Volcengine/Doubao** | openai-completions | China (ByteDance) | Doubao models |
| **BytePlus** | openai-completions | Global (ByteDance) | BytePlus models |
| **Xiaomi** | anthropic-messages | China | MiMo v2 Flash |
| **NVIDIA** | openai-completions | Global | Nemotron 70B |

### Tier 3: Aggregator/Proxy Providers

| Provider | Protocol | What It Does |
|----------|----------|--------------|
| **OpenRouter** | openai-completions | Routes to 100+ models from various providers |
| **Together** | openai-completions | Hosts open-source models (Llama, Mixtral, etc.) |
| **HuggingFace** | openai-completions | Text generation inference |
| **Vercel AI Gateway** | openai-completions | Edge-deployed AI proxy |
| **Cloudflare AI Gateway** | openai-completions | CDN-level AI proxy |
| **Kilocode** | openai-completions | Unified gateway |

### Tier 4: Self-Hosted/Local Providers

| Provider | Protocol | Extension | How It Works |
|----------|----------|-----------|-------------|
| **Ollama** | ollama (native) | `extensions/ollama/` | Run models on your machine. Discovery via `/api/tags` |
| **vLLM** | openai-completions | `extensions/vllm/` | High-performance model serving. Discovery via `/v1/models` |
| **SGLang** | openai-completions | `extensions/sglang/` | Fast inference server. Discovery via `/v1/models` |
| **Copilot Proxy** | openai-completions | `extensions/copilot-proxy/` | Local proxy for Copilot models |
| **LocalAI** | openai-completions | (built-in) | OpenAI-compatible local server |

### Provider Auth Flow Summary

| Auth Type | Providers | How It Works |
|-----------|-----------|-------------|
| **API Key** | Most providers | Set env var or paste key in config |
| **OAuth (PKCE)** | Google Gemini CLI | Browser login → localhost callback |
| **Device Code** | MiniMax Portal, Qwen Portal | Get code → visit URL → enter code |
| **AWS SDK** | Amazon Bedrock | AWS credentials from env/profile |
| **GitHub Token** | GitHub Copilot | `GH_TOKEN` or `GITHUB_TOKEN` |
| **Local/None** | Ollama, vLLM, SGLang | Just needs the base URL |

---

## Part 9: The models.json File (Persistence)

### How Provider Config Is Saved

**File: `src/agents/models-config.ts`**

All discovered and configured providers are written to `~/.openclaw/agents/<agentId>/models.json`. This file is the "cache" of provider configuration.

### The Write Lock

```typescript
const MODELS_JSON_WRITE_LOCKS = new Map<string, Promise<void>>();
```

Only one write at a time. If two processes try to write `models.json` simultaneously, the second waits for the first to finish. This prevents file corruption.

### The Planning System

Before writing, OpenClaw creates a "plan":

```typescript
export type ModelsJsonPlan =
  | { action: "skip" }     // No providers configured at all
  | { action: "noop" }     // File already matches what we'd write
  | { action: "write"; contents: string }; // Write new content
```

Planning steps:
1. Resolve implicit providers (auto-discovered)
2. Merge with explicit providers (from config)
3. Normalize everything
4. Apply mode (`merge` vs `replace`)
5. Merge with existing file (if `merge` mode)
6. Enforce secret management (no plaintext leaks)
7. Compare with existing file
8. Return plan

If the plan is `noop`, no disk write happens. This avoids unnecessary filesystem churn.

---

## How Layer 5 Evolved (Git History)

### Era 1: Single Embedded Runtime (December 2025)

OpenClaw started with a **single AI provider** — whatever the `pi` agent runtime supported.

**December 17, 2025 — `fece42ce0`** — `feat: embed pi agent runtime`

This was the foundational commit. Before this, OpenClaw spawned a separate CLI process to run AI inference. This commit **embedded** the AI runtime directly into the Node.js process. The file `src/agents/pi-embedded.ts` (507 lines) handled:
- System prompt construction
- OAuth token management
- Direct model interaction

There was no concept of "providers" or "model selection." It was hardcoded to one model.

**December 23, 2025 — `082c87246`** — `feat: support custom model providers`

Created `src/agents/models-config.ts` (64 lines). For the first time, users could configure custom providers with different base URLs and API keys. But it was basic — no discovery, no auth profiles, no protocol translation.

### Era 2: The Provider Explosion (January 2026)

January was when OpenClaw went from 1 provider to 10+ in three weeks.

**January 6** — `dbfa316d1` — Multi-agent routing + multi-account providers. The system could now route different conversations to different AI models.

**January 11** — `3da1afed6` — **GitHub Copilot** added as the first third-party provider. This required solving the "GitHub token resolution" problem.

**January 12** — `05ac67c52` — `models-config.providers.ts` split from `models-config.ts`. The provider code was already outgrowing its original file.

**January 13** — `8b5cd97ce` — **Synthetic provider** added for testing. 22 files, +937 lines. This let developers test the provider system without burning real API credits.

**January 17** — Three major additions in one day:
- `e93a1d813` — **Kimi Code provider** (Moonshot)
- `8eb80ee40` — **Qwen Portal with OAuth** — first OAuth provider! +945 lines including the full `extensions/qwen-portal-auth/` with device code flow
- `a6deb0d9d` — **Bundled provider auth plugins** — Created the extensions: `copilot-proxy/`, `google-antigravity-auth/`, `google-gemini-cli-auth/`

**January 20** — Two landmarks:
- `7f25523d8` — **`bedrock-converse-stream`** added to `MODEL_APIS`. AWS Bedrock got its own protocol.
- `a5adedea9` — **`aws-sdk` auth mode** added. Now providers could authenticate via AWS credentials instead of API keys.

**January 23** — `8effb557d` — **Dynamic Bedrock model discovery**. +944 lines. Added `@aws-sdk/client-bedrock` to discover all available models in your AWS account automatically.

**January 24** — `51e3d16be` — **Ollama with automatic model discovery**. Called `/api/tags` and `/api/show` to find all locally pulled models. This made `resolveImplicitProviders` async (it now needed to make network calls).

### Era 3: Protocol Diversification (February 2026)

Up until now, most providers pretended to be OpenAI-compatible. February changed that.

**February 14** — `11702290f` — **Native Ollama streaming**. Created `src/agents/ollama-stream.ts` (+760 lines). Ollama was no longer forced through the OpenAI-compatible path — it got its own native `/api/chat` handler with proper tool calling support. This was a major milestone: the system could now handle truly different API protocols, not just "everything pretends to be OpenAI."

**February 22** — `ded9a59f7` — **OpenRouter: allow any model ID**. Previously, OpenRouter models had to be in a static catalog. Now any model ID was passed through directly. This was important because OpenRouter aggregates hundreds of models from dozens of providers.

**February 23** — `382fe8009` — **Removed google-antigravity provider**. First provider deprecation. Not all experiments succeed.

**February 26** — `861b90f79` — **`openai-codex-responses`** added as a distinct ModelApi type.

### Era 4: WebSocket-First Streaming (March 1, 2026)

A single day brought a transformative change.

**March 1** — `7ced38b5e` — **OpenAI WebSocket-first streaming with HTTP fallback**.

This was a massive 1,260-line addition:
- `src/agents/openai-ws-connection.ts` (528 lines) — WebSocket connection manager
- `src/agents/openai-ws-stream.ts` (680 lines) — WebSocket streaming handler

Why WebSocket? HTTP SSE (Server-Sent Events) creates a new connection for every request. WebSocket keeps one connection open and reuses it. For rapid back-and-forth conversations (especially with tool calls), this is dramatically faster.

The implementation is smart: it tries WebSocket first, and if that fails (corporate firewall, proxy issues), it silently falls back to HTTP SSE. Users don't even notice the switch.

Same day: WebSocket warm-up was added — pre-establishing the connection before it's needed, so the first request doesn't pay the connection setup cost.

### Era 5: Reasoning Model Support (Ongoing, Jan-Mar 2026)

Reasoning models (AI that "thinks out loud" before answering) were supported incrementally:

- **Early January** — `b05981ef2` — "add reasoning tag hint for local providers" — basic reasoning tag parsing
- **February 16** — `46bf210e0` — "always drop orphaned OpenAI reasoning blocks in session history" — cleanup
- **February 20** — `feccac672` — "sanitize thinking blocks for GitHub Copilot Claude models" — Copilot-specific
- **March 8** — `d6d04f361` — "preserve local limits and native thinking fallback" for Ollama
- **March 13** — `de35fba9b` — **The big split**: Created dedicated `transcript-hygiene.ts` (470 lines) and `thinking.ts` for proper reasoning block management. This was the "we need to do this properly" moment.
- **March 13** — `0f15cfe21` — "self-heal anthropic replay thinking history" — automatic repair of corrupted thinking blocks

### Era 6: Architecture Maturity (March 8-13, 2026)

The final week before our snapshot saw massive architectural cleanup:

**March 8** — Four refactoring commits in one day:
- `609403505` — **Split static provider builders** into `models-config.providers.static.ts` (440 lines)
- `52bc80914` — **Extract stream wrappers** from a 696-line monolith into `anthropic-stream-wrappers.ts`, `moonshot-stream-wrappers.ts`, `proxy-stream-wrappers.ts`
- `f66bd105a` — Decompose implicit provider resolution
- `65623e155` — Split provider discovery helpers

**March 12** — `d83491e75` — **Provider plugin architecture**. The biggest milestone:
- Created `src/plugins/provider-discovery.ts` — plugin-based provider discovery
- Created `src/plugins/provider-wizard.ts` (243 lines) — onboarding wizard for providers
- Extracted Ollama, SGLang, vLLM into proper extensions
- 41 files, +1,731 lines

Before this commit, all provider discovery was hardcoded in `models-config.providers.ts`. After this commit, providers could be **plugins** just like channels. Any extension could register a provider with custom auth flows and model discovery.

**March 12-13** — Fast Mode / Service Tiers:
- `35aafd7ca` — **Anthropic fast mode support** — `src/agents/fast-mode.ts`
- `d5bffcdea` — Fast mode for OpenAI models
- `2bcef8405` — Explicit Anthropic service tier wrapper
- `5c7a640f8` — Wire service tiers into the runner

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    User's Config (config.json5)              │
│  models:                                                     │
│    mode: "merge"                                             │
│    providers:                                                │
│      my-ollama: { baseUrl: "http://localhost:11434" }        │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         │    Provider Discovery      │
         │    (7-phase pipeline)      │
         │                            │
         │  Phase 1: Check env vars   │
         │  Phase 2: Plugin providers │
         │  Phase 3: OAuth profiles   │
         │  Phase 4: Paired providers │
         │  Phase 5: Cloudflare       │
         │  Phase 6: Late plugins     │
         │  Phase 7: Copilot/Bedrock  │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │     Auth Resolution        │
         │  (7-source waterfall)      │
         │                            │
         │  1. Explicit profile       │
         │  2. AWS SDK override       │
         │  3. Auth profile store     │
         │  4. Environment variable   │
         │  5. Config API key         │
         │  6. Synthetic local auth   │
         │  7. Bedrock default        │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │     Model Selection        │
         │                            │
         │  Normalize provider ID     │
         │  Normalize model ID        │
         │  Check alias index         │
         │  Verify availability       │
         │  Apply thinking level      │
         │  Fallback: claude-opus-4-6 │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴──────────────────────────┐
         │         Inference Pipeline               │
         │                                          │
         │  ┌──────────────────────────────────┐   │
         │  │          Run Loop                 │   │
         │  │  (24-160 retry iterations)        │   │
         │  │                                    │   │
         │  │  ┌────────────────────────────┐   │   │
         │  │  │    Single Attempt           │   │   │
         │  │  │                              │   │   │
         │  │  │  1. Build system prompt      │   │   │
         │  │  │  2. Construct tool schemas   │   │   │
         │  │  │  3. Provider transforms      │   │   │
         │  │  │  4. streamSimple() ──────┐   │   │   │
         │  │  │  5. Process stream       │   │   │   │
         │  │  │  6. Accumulate usage     │   │   │   │
         │  │  └──────────────────────────┘   │   │   │
         │  │                                    │   │
         │  │  On failure → next profile/retry   │   │
         │  └──────────────────────────────────┘   │
         │                                          │
         │  Stream Wrappers:                        │
         │   ├── OpenAI WebSocket (preferred)       │
         │   ├── OpenAI HTTP SSE (fallback)         │
         │   ├── Anthropic Messages stream          │
         │   ├── Ollama native /api/chat            │
         │   ├── Moonshot-specific                  │
         │   └── Generic proxy                      │
         └─────────────┬──────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         │     Usage & Cost           │
         │                            │
         │  Normalize 20+ field names │
         │  into 4: input, output,    │
         │  cacheRead, cacheWrite     │
         │                            │
         │  Cost = tokens × $/1M      │
         └────────────────────────────┘
```

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/config/types.models.ts` | 76 | Core type definitions (ModelApi, ModelDefinitionConfig, ModelProviderConfig) |
| `src/agents/models-config.providers.ts` | ~1,000 | Provider discovery pipeline (7 phases) |
| `src/agents/models-config.providers.static.ts` | ~550 | Static provider builder functions |
| `src/agents/models-config.providers.discovery.ts` | ~240 | Dynamic model discovery (Ollama, vLLM, etc.) |
| `src/agents/models-config.ts` | ~400 | models.json read/write/merge logic |
| `src/agents/model-auth.ts` | 521 | Auth resolution waterfall (7 sources) |
| `src/agents/model-selection.ts` | 728 | Model routing, normalization, aliases |
| `src/agents/model-catalog.ts` | ~200 | Model catalog management |
| `src/agents/usage.ts` | 191 | Token usage normalization and cost |
| `src/agents/provider-capabilities.ts` | 169 | Provider-specific quirks and capabilities |
| `src/agents/pi-embedded-runner/run.ts` | ~500 | Main retry/failover run loop |
| `src/agents/pi-embedded-runner/run/attempt.ts` | ~400 | Single inference attempt |
| `src/agents/pi-embedded-runner/model.ts` | ~300 | Runtime model resolution |
| `src/agents/pi-embedded-runner/transcript-hygiene.ts` | 470 | Conversation history sanitization |
| `src/agents/pi-embedded-runner/thinking.ts` | ~200 | Reasoning/thinking block handling |
| `src/agents/openai-ws-connection.ts` | 528 | OpenAI WebSocket connection manager |
| `src/agents/openai-ws-stream.ts` | 680 | OpenAI WebSocket streaming |
| `src/agents/ollama-stream.ts` | 760 | Native Ollama streaming |
| `src/agents/fast-mode.ts` | ~100 | Anthropic/OpenAI fast mode |
| `src/plugins/provider-discovery.ts` | ~150 | Plugin-based provider discovery |
| `src/plugins/provider-wizard.ts` | 243 | Provider onboarding wizard |
| `extensions/ollama/index.ts` | 115 | Ollama provider plugin |
| `extensions/vllm/index.ts` | 92 | vLLM provider plugin |
| `extensions/sglang/index.ts` | 92 | SGLang provider plugin |
| `extensions/google-gemini-cli-auth/` | ~400 | Google OAuth (PKCE) |
| `extensions/minimax-portal-auth/` | ~300 | MiniMax device code OAuth |
| `extensions/qwen-portal-auth/` | ~300 | Qwen device code OAuth |
