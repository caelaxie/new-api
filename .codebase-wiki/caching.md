# Caching in LLM Routing

This document explains how caching works in the relay layer — the part of the system that proxies requests to upstream LLM providers (OpenAI, Claude, Gemini, etc.).

There are **three distinct concepts** all called "caching" in this codebase. Don't confuse them:

| Concept | What it does | Where |
|---------|-------------|-------|
| **Request Body Cache** | Keeps request bodies alive across middleware/adaptors without re-reading | `common/body_storage.go` |
| **Channel Affinity Cache** | Routes repeat requests to the same upstream channel | `service/channel_affinity.go` |
| **Provider Prompt Caching** | Adjusts billing when providers report cached tokens | `relay/channel/*/` + `service/text_quota.go` |

---

## 1. The Big Picture

```
                          CLIENT REQUEST
                               │
                               ▼
                     ┌───────────────────┐
                     │   HTTP Middleware  │
                     │  (auth, rate-limit │
                     │   distribute)     │
                     └────────┬──────────┘
                              │
                              ▼
                     ┌───────────────────┐
                     │   Body Storage    │  ◄── Cache #1: Request body
                     │  (memory or disk) │      kept in context for reuse
                     └────────┬──────────┘
                              │
                              ▼
                     ┌───────────────────┐
                     │ Channel Affinity  │  ◄── Cache #2: Sticky routing
                     │  (pick channel)   │      "send to same upstream"
                     └────────┬──────────┘
                              │
                              ▼
                     ┌───────────────────┐
                     │  Relay Adaptor    │
                     │  (OpenAI/Claude/  │
                     │   Gemini/etc.)    │
                     └────────┬──────────┘
                              │
                     ┌────────┴──────────┐
                     │  UPSTREAM LLM     │
                     │  (OpenAI, Claude, │
                     │   Gemini, etc.)   │
                     └────────┬──────────┘
                              │
                     Response comes back with usage stats
                              │
                              ▼
                     ┌───────────────────┐
                     │ Usage Extraction  │  ◄── Cache #3: Extract cached
                     │ (normalize cache  │      token counts from response
                     │  token counts)    │
                     └────────┬──────────┘
                              │
                              ▼
                     ┌───────────────────┐
                     │  Billing / Quota  │  ◄── Apply different rates
                     │  (cache_ratio *   │      for cached vs fresh tokens
                     │   token_count)    │
                     └───────────────────┘
```

---

## 2. Request Body Cache (BodyStorage)

### The Problem

An HTTP request body can only be read once. But in this system, the body is read multiple times:
- Once to count tokens (for billing estimation)
- Once to extract the model name (for channel selection)
- Once to forward to the upstream provider
- Once for logging

### The Solution

The first read wraps the body in a `BodyStorage` object and stashes it in the Gin context. Every subsequent read pulls from the cache instead of re-reading the HTTP stream.

```
  HTTP Body (can only read once!)
       │
       ▼
  ┌─────────────────────────────────────────────┐
  │            BodyStorage (first read)          │
  │                                              │
  │  Is body >= 10MB AND disk cache enabled?     │
  │         │                    │               │
        YES ▼                 NO ▼               │
  ┌──────────────┐    ┌───────────────┐          │
  │ DiskStorage  │    │ MemoryStorage │          │
  │ (temp file)  │    │ ([]byte)      │          │
  └──────┬───────┘    └───────┬───────┘          │
         │                    │                  │
         └────────┬───────────┘                  │
                  ▼                               │
         Stored in gin.Context                   │
         under key "KeyBodyStorage"              │
  └─────────────────────────────────────────────┘
       │         │         │         │
       ▼         ▼         ▼         ▼
   Token      Model     Forward    Log
   Count     Extract   to Upstream
   (all reuse the same BodyStorage)
```

### Key Files

| File | What it does |
|------|-------------|
| `common/body_storage.go` | `BodyStorage` interface, `memoryStorage`, `diskStorage` implementations |
| `common/disk_cache.go` | Temp file lifecycle: create, read, cleanup (files older than 5 min are deleted) |
| `common/disk_cache_config.go` | Config + atomic counters (active files, bytes used, hits) |
| `common/gin.go` | `GetBodyStorage()` / `CleanupBodyStorage()` — context integration |
| `setting/performance_setting/config.go` | Admin settings: `DiskCacheEnabled`, `DiskCacheThresholdMB` (default 10), `DiskCacheMaxSizeMB` (default 1024) |

---

## 3. Channel Affinity Cache (Sticky Routing)

### The Problem

LLM providers like OpenAI and Claude have **server-side prompt caching**. If you send the same prompt prefix to the same endpoint repeatedly, the provider caches it and charges you less. But this only works if consecutive requests hit the **same upstream channel** (same API key + same endpoint).

Without sticky routing, the gateway round-robins across channels, breaking provider-side caches.

### The Solution

The affinity cache remembers which channel handled a request for a given "affinity key" (e.g., `prompt_cache_key` or `user_id`), and routes the next request with the same key to the same channel.

```
  Request A: prompt_cache_key = "abc123"
       │
       ▼
  ┌──────────────────┐     ┌──────────────────┐
  │ Affinity Cache   │     │ Channel Pool     │
  │                  │     │                  │
  │ abc123 → Chan 5  │────▶│ Chan 1  Chan 2   │
  │                  │     │ Chan 3  Chan 4   │
  │ (LRU or Redis)   │     │ Chan 5  ◄─── GO  │
  └──────────────────┘     └──────────────────┘
       │
       ▼
  Request B: prompt_cache_key = "abc123"
       │
       ▼
  Cache hit! Route to Chan 5 again
  (provider sees same endpoint → cache hit → cheaper billing)
```

### How Keys Are Extracted

Affinity rules are configured in `setting/operation_setting/channel_affinity_setting.go`. Each rule specifies:
- **model pattern** (regex): e.g., `^gpt-.*$`
- **path**: e.g., `/v1/responses`
- **key source**: JSON path in the request body, e.g., `prompt_cache_key` or `metadata.user_id`

Default rules:
- OpenAI: key = `prompt_cache_key` from request body
- Claude: key = `metadata.user_id` from request body

### Key Files

| File | What it does |
|------|-------------|
| `service/channel_affinity.go` | `HybridCache[int]` storing channel IDs, LRU fallback (100k capacity, 1h TTL), Redis path |
| `setting/operation_setting/channel_affinity_setting.go` | Rule configuration |
| `controller/channel_affinity_cache.go` | Admin API: stats, clear cache |

---

## 4. Provider Prompt Caching (Billing)

This is **not** about caching responses. It's about **billing adjustments** when upstream providers report that part of your prompt was served from their internal cache.

### The Concept

LLM providers charge different rates:
- **Cache read**: tokens the provider already had cached (cheaper)
- **Cache write/creation**: tokens the provider is caching for the first time (more expensive)
- **Regular input**: tokens with no caching involved (base price)

The gateway extracts these counts from provider responses and applies different billing ratios.

### Provider Differences

Each provider reports cache stats differently. The gateway normalizes them all into a common format.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    UPSTREAM RESPONSES                        │
  ├──────────────────────────────────────────────────────────────┤
  │                                                              │
  │  OpenAI / DeepSeek / Zhipu / Moonshot                       │
  │  ┌─────────────────────────────────────────────┐            │
  │  │ usage.prompt_tokens_details.cached_tokens   │ ──┐        │
  │  │ OR usage.prompt_cache_hit_tokens            │   │        │
  │  └─────────────────────────────────────────────┘   │        │
  │                                                    │        │
  │  Claude (Anthropic)                                │        │
  │  ┌─────────────────────────────────────────────┐   │        │
  │  │ usage.cache_read_input_tokens               │   │        │
  │  │ usage.cache_creation_input_tokens           │   ├──▶ NORMALIZE
  │  │ usage.cache_creation.ephemeral_5m_input_tokens│  │        │
  │  │ usage.cache_creation.ephemeral_1h_input_tokens│  │        │
  │  └─────────────────────────────────────────────┘   │        │
  │                                                    │        │
  │  Gemini                                            │        │
  │  ┌─────────────────────────────────────────────┐   │        │
  │  │ usageMetadata.cachedContentTokenCount       │ ──┘        │
  │  └─────────────────────────────────────────────┘            │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
                               │
                               ▼
                  ┌─────────────────────────┐
                  │  Normalized Usage DTO   │
                  │                         │
                  │  CachedTokens           │  (cache reads)
                  │  CachedCreationTokens   │  (cache writes)
                  │  ClaudeCacheCreation5m  │  (Claude 5-min TTL)
                  │  ClaudeCacheCreation1h  │  (Claude 1-hour TTL)
                  └─────────────────────────┘
```

### Billing Math

Here's a simplified version of how billing works:

```
  Total Prompt Tokens = 1000
  Cached Tokens       =  400  (cache read)
  Cache Creation      =  200  (cache write)
  Regular Input       =  400  (1000 - 400 - 200)

  Cache Read Ratio    =  0.1  (Claude: 10% of input price)
  Cache Creation Ratio = 1.25 (Claude: 125% of input price)

  ┌─────────────────────────────────────────────────┐
  │  Cost = Regular × 1.0                           │
  │       + Cached  × 0.1    ← reads are cheap      │
  │       + Created × 1.25   ← writes cost more     │
  │                                                 │
  │  Cost = 400 × 1.0                               │
  │       + 400 × 0.1                               │
  │       + 200 × 1.25                              │
  │                                                 │
  │  Cost = 400 + 40 + 250 = 690 "quota units"      │
  │                                                 │
  │  Compare to no caching:                         │
  │  Cost = 1000 × 1.0 = 1000 "quota units"         │
  │                                                 │
  │  Savings: 31%                                   │
  └─────────────────────────────────────────────────┘
```

### Default Ratios (from `setting/ratio_setting/cache_ratio.go`)

| Provider | Cache Read Ratio | Cache Creation Ratio |
|----------|-----------------|---------------------|
| Claude (all models) | 0.1 (10%) | 1.25 (125%) |
| GPT-4o, o1, o3 | 0.5 (50%) | 1.0 (100%) |
| DeepSeek | 0.25 (25%) | 1.0 (100%) |
| Gemini | 0.1 (10%) | 1.0 (100%) |

"1.0" means same as regular input price. Ratios are configurable by admins.

### Claude's Extra Complexity

Claude has two cache creation tiers based on TTL:
- **5-minute ephemeral**: cheaper (uses `cacheCreationRatio5m`)
- **1-hour ephemeral**: more expensive (uses `cacheCreationRatio1h = cacheCreationRatio × 1.6`)

The constant `1.6` comes from Anthropic's pricing: 1-hour cache costs 6x the base, while 5-minute costs 3.75x, so `6/3.75 ≈ 1.6`.

### Key Files

| File | What it does |
|------|-------------|
| `relay/channel/openai/relay-openai.go:595-718` | `applyUsagePostProcessing()` — normalizes cache tokens from DeepSeek, Zhipu, Moonshot, OpenAI, llama.cpp |
| `relay/channel/claude/relay-claude.go:593-795` | Claude cache token extraction + stream patching |
| `relay/channel/gemini/relay-gemini.go` | Gemini `cachedContentTokenCount` extraction |
| `dto/openai_response.go:223-261` | `Usage` struct with `CachedTokens`, `CachedCreationTokens`, `ClaudeCacheCreation5m/1hTokens` |
| `dto/claude.go:557-596` | `ClaudeUsage` with `CacheReadInputTokens`, `CacheCreationInputTokens`, split 5m/1h |
| `setting/ratio_setting/cache_ratio.go` | Default cache ratios per model |
| `relay/helper/price.go:67-164` | `ModelPriceHelper()` — loads ratios into `PriceData` |
| `service/text_quota.go:159-310` | `calculateTextQuotaSummary()` — final billing math |
| `service/tiered_settle.go` | Tiered billing: maps cache tokens to expression variables `CR`, `CC`, `CC1h` |

---

## 5. Entity Caches (Supporting Infrastructure)

These aren't part of the relay path directly, but they support it by reducing database load.

```
  ┌─────────────────────────────────────────────────────┐
  │              Entity Cache Layer                      │
  ├──────────────┬──────────────┬───────────────────────┤
  │  User Cache  │ Token Cache  │  Channel Cache        │
  │  (Redis)     │ (Redis)      │  (In-Memory)          │
  │              │              │                       │
  │  Key:        │  Key:        │  Key:                 │
  │  user:{id}   │  token:{hmac}│  channelsIDM map      │
  │              │              │                       │
  │  TTL: 60s    │  TTL: 60s    │  TTL: sync frequency  │
  │              │              │                       │
  │  Fallback:   │  Fallback:   │  Fallback:            │
  │  DB query    │  DB query    │  DB query             │
  └──────────────┴──────────────┴───────────────────────┘
```

| Cache | Backend | File |
|-------|---------|------|
| User | Redis hash | `model/user_cache.go` |
| Token | Redis hash | `model/token_cache.go` |
| Channel | In-memory map | `model/channel_cache.go` |
| Subscription Plan | HybridCache (Redis or LRU) | `model/subscription.go` |
| Tiktoken Encoder | In-memory map | `service/tokenizer.go` |

### HybridCache (`pkg/cachex/hybrid_cache.go`)

A generic cache abstraction used by channel affinity and subscription plans:
- **Redis available**: uses Redis with codec-based serialization (Int, String, JSON)
- **No Redis**: falls back to `hot.HotCache` — an LRU in-memory cache with TTL and background cleanup

---

## 6. HTTP Response Caching (Dashboard Only)

These are standard HTTP cache headers, unrelated to LLM caching:

| Middleware | Applied To | Header |
|-----------|-----------|--------|
| `middleware/cache.go` | Static assets (JS, CSS) | `Cache-Control: max-age=604800` (1 week) |
| `middleware/disable-cache.go` | Sensitive API routes | `Cache-Control: no-store, no-cache` |
| `relay/helper/common.go` | SSE streaming responses | `Cache-Control: no-cache` |

---

## 7. Other Caches Worth Knowing

| Cache | File | Purpose |
|-------|------|---------|
| Vertex Access Token | `relay/channel/vertex/service_account.go` | Caches Google Cloud tokens (35min refresh, 30min TTL) |
| File Source Data | `service/file_service.go` | Caches loaded file data (URL or base64) in context |
| Exposed Ratio Cache | `setting/ratio_setting/exposed_cache.go` | 30s TTL cache for pricing data exposed to API consumers |
| Stream Cache Queue | `setting/sensitive.go` | Configurable buffer length for stream responses (default: disabled) |

---

## 8. Request Lifecycle Cheat Sheet

Here's a single request flowing through all cache layers:

```
  1. CLIENT ──POST /v1/chat/completions──▶ Router
                                              │
  2. Middleware Chain                          │
     ├─ RouteTag("relay")                     │
     ├─ SystemPerformanceCheck()              │
     ├─ TokenAuth() ──▶ TokenCache (Redis)    │
     ├─ ModelRequestRateLimit()               │
     └─ Distribute() ──▶ ChannelCache (mem)   │
                                              │
  3. controller.Relay()                       │
     ├─ Parse body ──▶ BodyStorage (mem/disk) │ ◄── Body cached in context
     ├─ Estimate tokens                       │
     ├─ ModelPriceHelper() ──▶ Load cache ratios
     ├─ PreConsumeBilling()                   │
     └─ ChannelAffinity ──▶ Pick channel      │ ◄── Sticky routing
                                              │
  4. Adaptor.DoResponse()                     │
     ├─ Forward to upstream LLM               │
     └─ Extract cache tokens from response    │ ◄── Provider-specific
                                              │
  5. PostTextConsumeQuota()                   │
     ├─ calculateTextQuotaSummary()           │
     │   ├─ Apply CacheRatio to read tokens   │ ◄── Billing math
     │   └─ Apply CreateCacheRatio to writes  │
     └─ RecordConsumeLog()                    │
                                              │
  6. CleanupBodyStorage()                     │ ◄── Delete temp files
```

---

## Glossary

| Term | Meaning |
|------|---------|
| **Cache read** | Tokens the provider served from its own cache (cheaper for you) |
| **Cache write/creation** | Tokens the provider is caching for the first time (costs more) |
| **Cache ratio** | Multiplier applied to cached token price (e.g., 0.1 = 10% of base price) |
| **Affinity key** | A value extracted from the request used for sticky routing (e.g., `prompt_cache_key`) |
| **HybridCache** | Generic cache that uses Redis if available, otherwise in-memory LRU |
| **BodyStorage** | Abstraction over request body that supports multiple reads |
| **Ephemeral 5m/1h** | Claude's two cache TTL tiers with different pricing |
