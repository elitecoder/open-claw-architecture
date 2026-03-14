# Layer 7: Memory System — Deep Dive

> Explained like you're 12 years old, with full git history evolution.

---

## What Is Layer 7?

Every time you have a conversation with the robot, it forgets everything when the conversation ends. It's like talking to someone with amnesia — brilliant, helpful, but no memory of yesterday.

Layer 7 fixes that. It gives the robot a **long-term memory** — a journal it can write in and search through later. When you tell it "I prefer dark mode" or "our deployment server is at 10.0.0.5," it can write that down. Next week, when you ask "what's our server IP?", it searches its journal and finds the answer.

This is called **RAG** (Retrieval-Augmented Generation) — a fancy way of saying "search your notes before answering."

Think of it like a student with a really good filing system. They can't remember everything they've ever learned, but they keep organized notes and know how to search them quickly.

Layer 7 has **seven major parts**:

1. **Embeddings** — Converting text into searchable numbers
2. **Storage** — Where the memory lives (SQLite + vector tables)
3. **Indexing & Sync** — How new information gets filed away
4. **Search & Retrieval** — Finding relevant memories
5. **Ranking Algorithms** — Deciding which results are best (hybrid search, MMR, temporal decay)
6. **Memory Extensions** — Plugin-based memory backends (LanceDB)
7. **Configuration & CLI** — Managing the memory system

---

## Part 1: Embeddings (Converting Text to Searchable Numbers)

### The Core Idea

Computers can't search by meaning — they can only compare numbers. So to search "things related to deployment," we need to convert text into numbers that capture its meaning. These numbers are called **embeddings** — a list of numbers (a "vector") that represents what a piece of text is about.

Texts about similar topics produce similar vectors. "Our deployment server is at 10.0.0.5" and "How do I deploy to production?" would have vectors pointing in roughly the same direction, even though they share few words.

### The Embedding Provider System

**File: `src/memory/embeddings.ts`**

OpenClaw supports **6 embedding providers**:

| Provider | File | Default Model | Dimensions | Notes |
|----------|------|--------------|------------|-------|
| **OpenAI** | `embeddings-openai.ts` | text-embedding-3-small | 1,536 | Most popular, cloud-based |
| **Google Gemini** | `embeddings-gemini.ts` | gemini-embedding-001 | 768 | Multimodal support (images!) |
| **Voyage AI** | `embeddings-voyage.ts` | voyage-large-2 | 1,024 | Specialized for retrieval |
| **Mistral** | `embeddings-mistral.ts` | mistral-embed | 1,024 | European provider |
| **Ollama** | `embeddings-ollama.ts` | nomic-embed-text | varies | Fully local, runs on your machine |
| **Local** | (node-llama-cpp) | embeddinggemma-300m | 300 | Built-in, no network needed |

**Dimensions** is the size of the vector. More dimensions = more nuanced understanding, but more storage and slower search. 1,536 dimensions (OpenAI) means each text chunk is represented by 1,536 numbers.

### The Provider Interface

```typescript
type EmbeddingProvider = {
  id: string;                                     // "openai", "gemini", etc.
  model: string;                                  // "text-embedding-3-small"
  maxInputTokens?: number;                        // Max text length (8192 for OpenAI)
  embedQuery: (text: string) => Promise<number[]>; // Embed a single search query
  embedBatch: (texts: string[]) => Promise<number[][]>; // Embed many texts at once
  embedBatchInputs?: (inputs: EmbeddingInput[]) => Promise<number[][]>; // Multimodal
};
```

### Auto-Selection and Fallback

When the provider is set to `"auto"`, the system tries providers in order:

```
1. Local model (if model file exists on disk)
       │ (not available)
2. OpenAI (if OPENAI_API_KEY set)
       │ (not available)
3. Gemini (if GEMINI_API_KEY set)
       │ (not available)
4. Voyage AI (if VOYAGE_API_KEY set)
       │ (not available)
5. Mistral (if MISTRAL_API_KEY set)
       │ (not available)
6. FTS-only mode (no embeddings, just keyword search)
```

If the primary provider fails at runtime, a **fallback provider** kicks in automatically. If the fallback also fails with auth errors, the system degrades gracefully to FTS-only mode (keyword search without vectors).

### Multimodal Embeddings

Gemini's newer models can embed **images and audio**, not just text:

```typescript
type EmbeddingInput = {
  text: string;
  parts?: [
    { type: "text"; text: string },
    { type: "inline-data"; mimeType: string; data: string } // base64
  ];
};
```

This means the memory system can index screenshots, diagrams, and audio files — and find them by searching with text queries like "that architecture diagram we discussed."

### The Embedding Cache

Computing embeddings costs money (API calls) and time. So OpenClaw caches them:

```sql
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,  -- Hash of provider config (invalidates on changes)
  hash TEXT NOT NULL,          -- Hash of input text
  embedding TEXT NOT NULL,     -- JSON array of floats
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

If you re-index a file that hasn't changed, the embeddings are loaded from cache instead of recomputed. The cache uses LRU (Least Recently Used) eviction — when it's full, the oldest entries are removed.

The `provider_key` is clever: it's a hash of the provider configuration. If you switch from OpenAI to Gemini, the old cache entries aren't used (because Gemini's vectors mean different things than OpenAI's).

### Batch Embedding

For large indexing jobs, making one API call per text chunk would be painfully slow. Batch embedding processes hundreds of chunks in a single API call:

**Supported batch APIs:**
- **OpenAI Batches** (`batch-openai.ts`): Up to 50,000 requests per batch, 24h completion window
- **Gemini Batches** (`batch-gemini.ts`): Immediate response via `batchEmbedContents`
- **Voyage Batches** (`batch-voyage.ts`): Similar to OpenAI flow

**Failure handling:** If batch embedding fails twice in a row, it's auto-disabled and the system falls back to one-at-a-time embedding. A success resets the failure counter.

```typescript
BATCH_FAILURE_LIMIT = 2;  // Auto-disable after 2 consecutive failures
```

---

## Part 2: Storage (Where Memory Lives)

### The SQLite Database

All memory data lives in a SQLite database at `~/.openclaw/memory/{agentId}.sqlite`. SQLite was chosen because it's a single file, requires no server, and handles concurrent reads well.

### The Database Schema

```sql
-- Metadata table (key-value store)
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

-- Tracked files (what's been indexed)
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',  -- "memory" or "sessions"
  hash TEXT NOT NULL,          -- Content hash for change detection
  mtime INTEGER NOT NULL,     -- Last modification time
  size INTEGER NOT NULL
);

-- Text chunks (the actual indexed content)
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,         -- Which embedding model was used
  text TEXT NOT NULL,          -- The actual text
  embedding TEXT NOT NULL,     -- JSON array of floats
  updated_at INTEGER NOT NULL
);

-- Full-text search index (BM25 keyword search)
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text, id UNINDEXED, path UNINDEXED, source UNINDEXED,
  model UNINDEXED, start_line UNINDEXED, end_line UNINDEXED
);

-- Vector search index (cosine similarity)
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[{dims}]    -- dims varies by provider (1536 for OpenAI)
);
```

There are **four key tables**:
1. **`files`** — Tracks which files have been indexed and their content hashes (for change detection)
2. **`chunks`** — The actual text chunks with their embeddings
3. **`chunks_fts`** — A FTS5 virtual table for fast keyword search (BM25 algorithm)
4. **`chunks_vec`** — A sqlite-vec virtual table for fast vector similarity search

### Memory File Organization

The robot reads from files in your workspace:

```
your-project/
├── MEMORY.md                    -- Primary memory file (evergreen, no decay)
└── memory/
    ├── 2026-03-10.md           -- Dated entry (temporal decay applies)
    ├── 2026-03-11.md           -- More recent = higher score
    ├── architecture/
    │   └── decisions.md        -- Topical subdirectory
    └── images/
        └── diagram.png         -- Multimodal (Gemini only)
```

**Two types of memory files:**
- **Evergreen** (`MEMORY.md`, non-dated files): Never decay in relevance. Good for permanent facts.
- **Dated** (`memory/YYYY-MM-DD.md`): Decay over time. Good for daily notes and session summaries.

---

## Part 3: Indexing & Sync (Filing Away Information)

### Chunking — Breaking Text Into Pieces

Before text can be embedded, it needs to be split into chunks (each embedding provider has a maximum input size).

**File: `src/memory/internal.ts`**

```
Algorithm:
1. Split content into lines
2. Accumulate lines until reaching the token threshold (default: 400 tokens)
3. When a chunk is full, save it (with start/end line numbers)
4. Carry forward some overlap (default: 80 tokens) for continuity
5. Continue until all lines are processed
```

**Why overlap?** Without overlap, if a sentence spans two chunks, neither chunk would have the full sentence. Overlap ensures that information at chunk boundaries is captured in both adjacent chunks.

**Default settings:**
- Chunk size: 400 tokens (~300 words)
- Overlap: 80 tokens (~60 words)

### The Sync Pipeline

**File: `src/memory/manager-sync-ops.ts` (1,391 lines)**

Sync is the process of detecting changes and re-indexing affected files.

```
Trigger (one of):
  ├── File watcher detects a change
  ├── Session transcript updated
  ├── Periodic interval (default: 5 minutes)
  ├── Manual "openclaw memory index"
  └── Search requested (auto-sync before searching)
         │
Step 1: Scan all memory directories
         │
Step 2: For each file:
  ├── Compute content hash
  ├── Compare with stored hash in `files` table
  │     │
  │     Same hash → Skip (no changes)
  │     Different → Re-index this file
  │     New file → Index for first time
  │     Missing → Remove from index
         │
Step 3: For files that need indexing:
  ├── Split into chunks
  ├── Check embedding cache for each chunk
  ├── Embed uncached chunks (batch if possible)
  ├── Store chunks + embeddings in database
  └── Update FTS and vector tables
         │
Step 4: Mark sync as clean
```

### Session Transcript Indexing

OpenClaw can also index **past conversations** so the robot can remember what you talked about:

**File: `src/memory/session-files.ts`**

Session files are JSONL (one JSON object per line):
```json
{"type": "message", "message": {"role": "user", "content": "Deploy to staging"}}
{"type": "message", "message": {"role": "assistant", "content": "Done! Deployed v2.3"}}
```

The indexer:
1. Reads each line
2. Extracts user/assistant messages
3. Formats as "User: ..." and "Assistant: ..."
4. Hashes the content for change detection
5. Creates a line map (maps chunk lines → JSONL source lines for citations)
6. Chunks and embeds like any other memory file

Session indexing is **optional** (configured via `sources: ["memory", "sessions"]`).

### File Watching

**Uses chokidar** to watch the `memory/` directory for changes:
- Ignores `.git`, `node_modules`, `.venv`, `__pycache__`
- Debounces rapid changes (1.5 seconds) — if you save a file 5 times in 2 seconds, it only re-indexes once
- Marks the index as "dirty" and triggers a sync

### Readonly Recovery

SQLite databases can become readonly (disk full, permissions, NFS issues). The system detects this and auto-recovers:

```
1. Detect "readonly" in error message
2. Close current database connection
3. Reopen database
4. Retry the failed operation
5. Track attempts/successes/failures for status reporting
```

---

## Part 4: Search & Retrieval (Finding Relevant Memories)

### The Search Entry Point

**File: `src/memory/manager.ts`**

```typescript
async search(query: string, opts?: {
  maxResults?: number;   // Default: 6
  minScore?: number;     // Default: 0.35
  sessionKey?: string;   // For session-scoped search
}): Promise<MemorySearchResult[]>
```

### Three Search Modes

The system operates in one of three modes depending on what's available:

#### Mode 1: Hybrid Search (Best Quality)

When both vector and FTS are available, the system runs **both** searches and merges results:

```
Query: "deployment server configuration"
         │
    ┌────┴────┐
    │         │
    ▼         ▼
Vector     Keyword
Search     Search (BM25)
    │         │
    │    ┌────┘
    ▼    ▼
  Merge Results
  (weighted combination)
    │
    ▼
  Apply Temporal Decay (optional)
    │
    ▼
  Apply MMR Re-ranking (optional)
    │
    ▼
  Filter by minScore
    │
    ▼
  Return top maxResults
```

#### Mode 2: Vector-Only Search

When FTS isn't available:
1. Embed the query
2. Search the vector table for nearest neighbors
3. Filter by minScore, return top results

#### Mode 3: FTS-Only Search (Fallback)

When no embedding provider is available (no API key, local model missing):
1. Extract keywords from query (remove stop words)
2. Search each keyword separately via BM25
3. Merge and deduplicate results

### Vector Search (Cosine Similarity)

**File: `src/memory/manager-search.ts`**

```sql
-- If sqlite-vec extension is loaded:
SELECT c.id, c.path, c.start_line, c.end_line, c.text, c.source,
       vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
 WHERE c.model = ? AND c.source IN (...)
 ORDER BY dist ASC
 LIMIT ?
```

The score is: `score = 1 - distance`. A distance of 0 means identical (score = 1.0). A distance of 1 means completely different (score = 0.0).

**Fallback (no sqlite-vec):** Load all embeddings into memory and compute cosine similarity manually:

```typescript
cosineSimilarity(a, b) = dot(a, b) / (||a|| * ||b||)
```

### Keyword Search (BM25)

```sql
SELECT id, path, source, start_line, end_line, text,
       bm25(chunks_fts) AS rank
  FROM chunks_fts
 WHERE chunks_fts MATCH ?
 ORDER BY rank ASC
 LIMIT ?
```

BM25 is a classic information retrieval algorithm that scores documents by:
- How often the search terms appear in the chunk (term frequency)
- How rare those terms are across all chunks (inverse document frequency)
- How long the chunk is (shorter chunks with the terms score higher)

### Hybrid Merging

**File: `src/memory/hybrid.ts`**

When both vector and keyword results come back:

```typescript
combinedScore = vectorWeight * vectorScore + textWeight * keywordScore
// Default: 0.7 * vector + 0.3 * keyword
```

Vector search is weighted higher (0.7) because it understands **meaning**, not just word matching. But keyword search (0.3) helps catch exact matches that vector search might miss.

### Search Result Structure

```typescript
type MemorySearchResult = {
  path: string;        // "memory/2026-03-10.md"
  startLine: number;   // Line 15
  endLine: number;     // Line 28
  score: number;       // 0.82
  snippet: string;     // Truncated to 700 chars
  source: "memory" | "sessions";
  citation?: string;   // Optional source attribution
};
```

---

## Part 5: Ranking Algorithms (Deciding What's Best)

### Query Expansion — Making Search Smarter

**File: `src/memory/query-expansion.ts` (810 lines)**

Before searching, the query is cleaned up and expanded:

```
Input: "that thing we discussed about the API"
         │
Step 1: Tokenize
  → ["that", "thing", "we", "discussed", "about", "the", "API"]
         │
Step 2: Remove stop words (language-aware!)
  → ["discussed", "API"]
         │
Step 3: Filter invalid keywords
  (< 3 chars, numbers only, punctuation only)
  → ["discussed", "API"]
         │
Step 4: Deduplicate
  → ["discussed", "API"]
```

**Multilingual stop words** — The system detects the language and applies appropriate filtering:
- **English**: "the", "is", "that", "we", "about", "thing" etc.
- **Chinese**: Character n-grams (unigrams + bigrams for phrase matching)
- **Japanese**: Mix of kanji, hiragana, katakana handling
- **Korean**: Strip trailing particles (정렬 → 정)
- **Arabic, Spanish, Portuguese**: Language-specific stop words

### Maximal Marginal Relevance (MMR) — Diversity Ranking

**File: `src/memory/mmr.ts` (198 lines)**

**Problem:** Without MMR, if you search "deployment configuration," you might get 6 results that all say roughly the same thing (because they're all similar to the query AND to each other).

**Solution:** MMR balances **relevance** (how well does this match the query?) with **diversity** (how different is this from results we've already picked?).

```
Algorithm:
1. Start with the highest-scoring result
2. For each remaining slot:
   a. For each candidate:
      - Compute: MMR = λ × relevance - (1-λ) × max_similarity_to_already_selected
   b. Pick the candidate with the highest MMR score
3. Repeat until we have enough results

Where:
  λ = 0.7 (default) — higher = more relevant, lower = more diverse
  similarity = Jaccard similarity on tokenized text
```

**Jaccard similarity** measures overlap between two sets of tokens:
```
Jaccard(A, B) = |words in both A and B| / |words in A or B|
```

**Example:**
- Result 1: "Deploy to server 10.0.0.5 using SSH" (score: 0.9)
- Result 2: "Deploy to server 10.0.0.5 via SSH key" (score: 0.88) — very similar to #1!
- Result 3: "Server configuration uses nginx proxy" (score: 0.75) — different info

Without MMR: Returns #1, #2, #3 (two near-duplicates)
With MMR: Returns #1, #3, then maybe #2 (diverse first, then fill)

### Temporal Decay — Recency Boosting

**File: `src/memory/temporal-decay.ts` (166 lines)**

**Problem:** A memory from 6 months ago about "server configuration" shouldn't rank as highly as one from yesterday — the newer one is more likely to be current.

**Solution:** Exponential decay based on age:

```
Decay Multiplier = e^(-λ × ageInDays)

Where λ = ln(2) / halfLifeDays
Default halfLifeDays = 30
```

**What this means in practice:**
- Today's memory: multiplier = 1.0 (full score)
- 30-day-old memory: multiplier ≈ 0.5 (half score)
- 60-day-old memory: multiplier ≈ 0.25 (quarter score)
- 90-day-old memory: multiplier ≈ 0.125 (eighth of score)

**How timestamps are determined:**
1. Dated files (`memory/2026-03-10.md`): date extracted from filename
2. Evergreen files (`MEMORY.md`, undated): **no decay applied** — they're permanently relevant
3. Other files: file modification time (mtime)

### The Full Search Pipeline (Putting It All Together)

```
User asks: "What's our deployment process?"
         │
1. Normalize & clean query
         │
2. Extract keywords: ["deployment", "process"]
         │
3. PARALLEL:
   ├─→ Embed query → [1536 floats via OpenAI]
   │   └─→ Vector search: top 200 candidates (cosine similarity)
   │
   └─→ FTS query: "\"deployment\" AND \"process\""
       └─→ Keyword search: top 200 candidates (BM25)
         │
4. Hybrid merge: score = 0.7 × vector + 0.3 × keyword
         │
5. Temporal decay: multiply by age-based factor
   - memory/2026-03-12.md (1 day old) → × 0.98
   - memory/2026-02-10.md (31 days) → × 0.49
   - MEMORY.md (evergreen) → × 1.0
         │
6. MMR re-ranking: maximize relevance + diversity
         │
7. Filter: score ≥ 0.35
         │
8. Return top 6 results
         │
9. Inject into AI context as <memory_search_results>
```

---

## Part 6: Memory Extensions (Plugin-Based Backends)

### memory-core Extension

**Location: `extensions/memory-core/`**

This is the **default memory plugin**. It's a thin wrapper that provides:
- `memory_search` tool — Semantic search for memory files
- `memory_get` tool — Read a specific snippet from a memory file
- CLI command: `openclaw memory` (status, index, search)

It delegates all real work to the core `MemoryIndexManager` via the plugin runtime API.

### memory-lancedb Extension

**Location: `extensions/memory-lancedb/`**

This is an **alternative memory plugin** using LanceDB (a vector database) instead of SQLite-vec. It adds features the core memory doesn't have:

**Auto-Recall:** Before the AI starts thinking, this plugin automatically searches memory for relevant context and injects it:

```typescript
// Hook: before_agent_start
1. Extract the user's latest message
2. Embed it
3. Search LanceDB for top 3 matching memories
4. Inject as <relevant-memories> in the context
```

**Auto-Capture:** After the AI finishes, this plugin analyzes the conversation and saves important information:

```typescript
// Hook: agent_end
1. Scan all messages for "trigger patterns"
   - "remember", "prefer", "important", "always", "never"
   - Phone numbers, emails
   - "my X is", "is my"
2. Filter out noise (emoji-heavy, system-generated, too short)
3. Check for duplicates (0.95 similarity threshold)
4. Store top 3 new memories with categories
```

**Memory Categories:**
- `preference` — "I prefer dark mode"
- `fact` — "The server runs Ubuntu 22.04"
- `decision` — "We decided to use PostgreSQL"
- `entity` — "John's phone is 555-0123"
- `other` — Everything else

**Memory Entry Schema:**
```typescript
type MemoryEntry = {
  id: string;          // UUID
  text: string;        // The memory content
  vector: number[];    // Embedding vector
  importance: number;  // 0 to 1
  category: string;    // "preference", "fact", etc.
  createdAt: number;   // Timestamp
};
```

**Tools Provided:**
- `memory_recall` — Search long-term memories
- `memory_store` — Save information to memory
- `memory_forget` — Delete memories (GDPR-compliant!)

**Prompt Injection Protection:** The plugin escapes special characters and wraps injected memories in `<relevant-memories>` tags with warnings to prevent memory content from being treated as instructions.

### The QMD Backend

**File: `src/memory/qmd-manager.ts` (2,069 lines — the largest source file!)**

QMD (Query Markdown) is an **alternative backend** that uses an external CLI tool (`qmd`) instead of the built-in SQLite system.

```typescript
type MemoryBackend = "builtin" | "qmd";
```

**QMD Features:**
- External process for search (spawns `qmd` CLI per query)
- Combines BM25 keyword search + vector search
- MCP runtime integration
- Multiple collections (memory, sessions, custom)
- Default search timeout: 4 seconds
- Update interval: 5 minutes

**When to use QMD:** It's designed for users who want a local-first search sidecar that runs independently of the OpenClaw process. The built-in backend is fine for most users.

### Backend Selection

**File: `src/memory/search-manager.ts` (254 lines)**

The `FallbackMemoryManager` wraps the primary backend with automatic fallback:

```
1. Try QMD backend (if configured)
       │ (fails?)
2. Fall back to builtin SQLite backend
       │ (also fails?)
3. Return null (no memory available)
```

---

## Part 7: Configuration & CLI

### Memory Configuration

**File: `src/agents/memory-search.ts` (407 lines)**

The full resolved configuration:

```typescript
type ResolvedMemorySearchConfig = {
  enabled: boolean;

  // What to index
  sources: ("memory" | "sessions")[];
  extraPaths: string[];              // Additional directories to index

  // Embedding provider
  provider: "openai" | "gemini" | "voyage" | "mistral" | "ollama" | "local" | "auto";
  model: string;
  fallback: "openai" | "gemini" | "local" | "voyage" | "mistral" | "ollama" | "none";

  // Storage
  store: {
    driver: "sqlite";
    path: string;                    // ~/.openclaw/memory/{agentId}.sqlite
    vector: {
      enabled: boolean;
      extensionPath?: string;        // Path to sqlite-vec binary
    };
  };

  // Chunking
  chunking: {
    tokens: 400;                     // Chunk size
    overlap: 80;                     // Overlap between chunks
  };

  // Search
  query: {
    maxResults: 6;
    minScore: 0.35;
    hybrid: {
      enabled: boolean;
      vectorWeight: 0.7;
      textWeight: 0.3;
      candidateMultiplier: 4;        // Search 4× more candidates than needed
      mmr: { enabled: boolean; lambda: 0.7 };
      temporalDecay: { enabled: boolean; halfLifeDays: 30 };
    };
  };

  // Sync
  sync: {
    onSessionStart: boolean;
    onSearch: boolean;               // Auto-sync before searching
    watch: boolean;                  // Watch filesystem for changes
    watchDebounceMs: 1500;
    intervalMinutes: 0;              // Periodic sync interval
  };

  // Batch
  batch: {
    enabled: boolean;
    wait: boolean;
    concurrency: number;
    pollIntervalMs: number;
    timeoutMs: number;
  };

  // Embedding cache
  cache: {
    enabled: boolean;
    maxEntries?: number;
  };

  // Multimodal
  multimodal: MemoryMultimodalSettings;
};
```

### Example Config

```json5
{
  agents: [{
    id: "main",
    memorySearch: {
      enabled: true,
      provider: "openai",
      sources: ["memory", "sessions"],
      extraPaths: ["~/notes"],
      query: {
        maxResults: 10,
        minScore: 0.3,
        hybrid: {
          mmr: { enabled: true, lambda: 0.7 },
          temporalDecay: { enabled: true, halfLifeDays: 14 },
        },
      },
    },
  }],
}
```

### CLI Commands

**File: `src/cli/memory-cli.ts` (818 lines)**

```bash
# Check memory status for all agents
openclaw memory status
openclaw memory status --deep      # Probe embedding provider availability
openclaw memory status --json      # Machine-readable output

# Force re-index
openclaw memory index
openclaw memory index --force      # Full reindex (not incremental)

# Search memory
openclaw memory search "deployment process"
openclaw memory search --max-results 20 --min-score 0.2 --json
```

The status command shows:
- Provider + model in use
- Files indexed / total files
- Chunks indexed
- Sources active (memory/sessions)
- Store path
- Vector extension status (loaded/missing)
- FTS status
- Cache status (entries count)
- Batch processing status
- Fallback reason (if degraded)

### Gateway Methods

```
doctor.memory.status    — Get memory health report
skills.status           — Includes memory tool availability
```

---

## How Layer 7 Evolved (Git History)

### The Genesis Week — January 12-18, 2026

The memory system was built in a **single intense week** by Peter Steinberger.

**January 12 — `bf11a42c3`** — `feat: add memory vector search`

The foundational commit. Created the entire memory system from scratch:
- `src/memory/manager.ts` — the core MemoryIndexManager
- OpenAI embedding support
- Memory CLI commands
- Memory search tool for agents
- Configuration system
- 1,300+ lines of new code

**January 15 — `cb78fa46a`** — Made `node-llama-cpp` optional. The local embedding model was too heavy as a required dependency. Now it's a fallback.

**January 17** — Three major features in one day:
- `5a08471dc` — **sqlite-vec acceleration** — Vector search went from loading all embeddings into memory to using a proper vector index. Massive speed improvement for large memory stores.
- `a31a79396` — **OpenAI batch indexing** — Batch embedding via the OpenAI Batch API (+327 lines in manager.ts)
- Multiple fixes: split embedding batches, parallelize indexing, session memory source

**January 18** — The system expanded dramatically:
- `be7191879` — **Gemini embeddings + auto provider selection** — Second embedding provider. Created `embeddings-gemini.ts` and `embeddings-openai.ts` (previously everything was in `embeddings.ts`). The auto-selection logic was born.
- `946477413` — **Gemini batch embedding** — Gemini batch support
- Memory extensions created: `memory-core` and `memory-lancedb` plugins

### The QMD Backend — January 27, 2026

**`5d3af3bc6`** — `feat(memory): Implement new (opt-in) QMD memory backend`

Author: Vignesh Natarajan. A major milestone introducing QMD as an alternative to the built-in memory:
- Created `qmd-manager.ts` (612 lines initially, now 2,069)
- Created `backend-config.ts` (245 lines)
- Expanded `search-manager.ts`
- Extracted `types.ts`

This was the first time the memory system had **two backends**. The architecture was refactored to support a common `MemorySearchManager` interface with pluggable implementations.

### Search Quality Revolution — January 26 to February 19, 2026

A community contributor, **Rodrigo Uroz**, transformed search quality:

**January 26 — `fa9420069`** — **MMR re-ranking** — Created `mmr.ts` (198 lines) with Jaccard similarity and the MMR algorithm. This was the first step beyond "just sort by score."

**February 10 — `6b3e0710f`** — **Temporal decay** — Created `temporal-decay.ts` (166 lines). Exponential decay with configurable half-life. 1,372 insertions across 13 files — a significant integration effort.

**February 19 — `a87b5fb00`** — Schema and config additions for MMR + temporal decay.

### Voyage AI — February 7, 2026

**`6965a2cc9`** — `feat(memory): native Voyage AI support`

Author: Jake (mcinteerj). Third embedding provider. Created:
- `embeddings-voyage.ts` (86 lines)
- `batch-voyage.ts` (363 lines)
- 879 insertions across 11 files

### The Great Decomposition — February 13-17, 2026

The memory system had grown too large. Two major refactors broke it apart:

**February 13 — `4c401d336`** — `refactor(memory): extract manager sync and embedding ops`

Split the monolithic `manager.ts` (which had grown to 1,777+ lines) into:
- `manager-embedding-ops.ts` (803 lines) — embedding generation, caching, batch processing
- `manager-sync-ops.ts` (998 lines) — file watching, sync pipeline, indexing

The `MemoryIndexManager` now extends `MemoryManagerEmbeddingOps` which extends `MemoryManagerSyncOps` — a clear inheritance chain separating concerns.

**February 17 — Two refactors:**
- `ddef3cadb` — Removed prototype mixing pattern (a code smell)
- `9bfd3ca19` — Consolidated batch helpers into shared infrastructure: `batch-runner.ts`, `batch-upload.ts`, `embeddings-remote-client.ts`

### Query Expansion Goes Multilingual — February 16-22, 2026

**February 16 — `de98782c3`** — `feat: LLM-based query expansion for FTS mode`

Author: kangxi. Created `query-expansion.ts` (357 lines initially) with keyword extraction for FTS-only mode. English and Chinese stop words.

**February 22** — Four language PRs merged on the same day:
- Korean support (Andrew Jeon)
- Japanese support (Vincent Koc)
- Spanish and Portuguese stop words
- Arabic stop words

The query expansion system went from English-only to **7 languages** in a single day.

### More Embedding Providers — February-March 2026

| Date | Provider | Author | Commit |
|------|----------|--------|--------|
| Feb 22 | **Mistral** | Vincent Koc | `d92ba4f8a` |
| Mar 3 | **Ollama** | nico-hoff | `3eec79bd6` |
| Mar 11 | **Gemini v2 preview** | Bill Chirico | `60aed9534` |

### Multimodal Memory — March 11, 2026

**`d79ca5296`** — `Memory: add multimodal image and audio indexing`

Author: Gustavo Madeira Santana. Created `multimodal.ts` (118 lines) and `embedding-inputs.ts` (34 lines). Extended Gemini embeddings to handle images and audio files.

### Post-Compaction Memory Sync — March 12, 2026

**`143e593ab`** — `Compaction Runner: wire post-compaction memory sync`

Author: Rodrigo Uroz. When the context engine (Layer 8) compacts conversation history, the memory system now automatically re-syncs to capture any information that was summarized away.

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory Files                             │
│  MEMORY.md  memory/*.md  sessions/*.jsonl  extra paths      │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         │    File Watcher + Sync     │
         │                            │
         │  chokidar watches memory/  │
         │  Session transcript events │
         │  Periodic interval (5min)  │
         │  Debounce: 1.5 seconds     │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │     Chunking Pipeline      │
         │                            │
         │  Split text: 400 tokens    │
         │  Overlap: 80 tokens        │
         │  Hash for deduplication    │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │   Embedding Generation     │
         │                            │
         │  6 providers:              │
         │  OpenAI, Gemini, Voyage,   │
         │  Mistral, Ollama, Local    │
         │                            │
         │  Auto-selection + fallback │
         │  Batch API support         │
         │  Embedding cache (LRU)     │
         └─────────────┬─────────────┘
                       │
         ┌─────────────┴─────────────┐
         │      SQLite Storage        │
         │                            │
         │  chunks table (text+embed) │
         │  chunks_fts (BM25 FTS5)    │
         │  chunks_vec (sqlite-vec)   │
         │  embedding_cache (LRU)     │
         └─────────────┬─────────────┘
                       │
                  (on search)
                       │
         ┌─────────────┴──────────────────────────┐
         │          Search Pipeline                 │
         │                                          │
         │  ┌──────────┐    ┌──────────────┐       │
         │  │  Vector   │    │   Keyword    │       │
         │  │  Search   │    │   Search     │       │
         │  │ (cosine)  │    │   (BM25)     │       │
         │  └─────┬─────┘    └──────┬───────┘       │
         │        │                 │                │
         │        └────────┬────────┘                │
         │                 │                         │
         │         Hybrid Merge                      │
         │     (0.7 vector + 0.3 keyword)            │
         │                 │                         │
         │         Temporal Decay                    │
         │     (exp decay, 30-day half-life)         │
         │                 │                         │
         │         MMR Re-ranking                    │
         │     (λ=0.7 relevance/diversity)           │
         │                 │                         │
         │         Filter minScore ≥ 0.35            │
         │         Return top 6 results              │
         └─────────────────────────────────────────┘

Alternative Backends:
  ├── QMD (external CLI, local-first sidecar)
  └── LanceDB (plugin, auto-recall/capture)
```

---

## Key Files Reference

| File | Lines | Purpose |
|------|-------|---------|
| `src/memory/manager.ts` | 840 | Core MemoryIndexManager class |
| `src/memory/manager-sync-ops.ts` | 1,391 | File watching, sync pipeline, indexing |
| `src/memory/manager-embedding-ops.ts` | 925 | Embedding generation, caching, batch processing |
| `src/memory/qmd-manager.ts` | 2,069 | QMD alternative backend (largest file!) |
| `src/memory/query-expansion.ts` | 810 | Multilingual keyword extraction |
| `src/memory/embeddings.ts` | ~300 | Provider orchestration + auto-selection |
| `src/memory/embeddings-openai.ts` | ~100 | OpenAI embedding provider |
| `src/memory/embeddings-gemini.ts` | ~200 | Gemini embedding provider (multimodal) |
| `src/memory/embeddings-voyage.ts` | ~90 | Voyage AI embedding provider |
| `src/memory/embeddings-mistral.ts` | ~50 | Mistral embedding provider |
| `src/memory/embeddings-ollama.ts` | ~140 | Ollama local embedding provider |
| `src/memory/mmr.ts` | 198 | Maximal Marginal Relevance algorithm |
| `src/memory/temporal-decay.ts` | 166 | Exponential recency decay |
| `src/memory/hybrid.ts` | ~160 | Hybrid search merging (vector + keyword) |
| `src/memory/search-manager.ts` | 254 | Backend abstraction + fallback |
| `src/memory/backend-config.ts` | 355 | Backend selection (builtin vs QMD) |
| `src/memory/internal.ts` | ~470 | Chunking, cosine similarity, file scanning |
| `src/memory/session-files.ts` | ~200 | Session transcript indexing |
| `src/memory/multimodal.ts` | 118 | Image/audio indexing |
| `src/memory/types.ts` | ~80 | Core type definitions |
| `src/memory/batch-openai.ts` | ~200 | OpenAI batch embedding |
| `src/memory/batch-gemini.ts` | ~200 | Gemini batch embedding |
| `src/memory/batch-voyage.ts` | ~360 | Voyage batch embedding |
| `src/memory/batch-runner.ts` | ~150 | Generic batch orchestration |
| `src/agents/memory-search.ts` | 407 | Agent-level memory config resolution |
| `src/agents/tools/memory-tool.ts` | ~200 | memory_search and memory_get agent tools |
| `src/cli/memory-cli.ts` | 818 | CLI commands (status, index, search) |
| `extensions/memory-core/index.ts` | ~50 | Default memory plugin |
| `extensions/memory-lancedb/index.ts` | ~500 | LanceDB plugin with auto-recall/capture |
| **Total: 99 files** | **21,343 lines** | |
