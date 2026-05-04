# Prompt Cache Pattern for AI Applications

> AI Engineering Series | May 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Prompt Cache Matters](#why-prompt-cache-matters)
3. [How Prompt Caching Works](#how-prompt-caching-works)
4. [Anthropic Prompt Cache — Mechanics](#anthropic-prompt-cache--mechanics)
5. [Cache Breakpoints — Where to Cut](#cache-breakpoints--where-to-cut)
6. [Building a Cache-Optimised Prompt — Step by Step](#building-a-cache-optimised-prompt)
7. [Refresh & Invalidation](#refresh--invalidation)
8. [AI-Specific Use Cases](#ai-specific-use-cases)
9. [Advanced Patterns](#advanced-patterns)
10. [Monitoring & Observability](#monitoring--observability)
11. [Integration with Other Patterns](#integration-with-other-patterns)
12. [Production Best Practices](#production-best-practices)
13. [Common Mistakes](#common-mistakes)
14. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is Prompt Caching?

**Prompt caching** is a server-side optimisation provided by LLM APIs that lets you reuse the *processing* of a long prefix of your prompt across multiple requests. You mark a stable region of the prompt as cacheable, the provider hashes that region and stores the model's intermediate state (KV cache), and subsequent requests that share the same prefix skip re-encoding it.

```
Without prompt cache:
─────────────────────────────────────────────────────
each request: encode 50K input tokens → generate
              ↑ 100% input cost, 100% latency every call

With prompt cache (Anthropic):
─────────────────────────────────────────────────────
1st request:  encode 50K (write to cache, +25% surcharge)
2nd request:  reuse cached prefix (read at 10% of input cost)
3rd request:  reuse cached prefix (read at 10% of input cost)
              ↑ 90% input-token cost reduction, lower latency
```

This is a **provider-native** pattern — Anthropic, OpenAI, and Google all offer some flavour. We focus on Anthropic's because it has the most explicit control surface (manual breakpoints, 5-min and 1-hour TTLs).

### The Problem It Solves

Modern AI applications stuff enormous amounts of stable context into every request:

- System prompt with instructions, persona, output schema (~2–10K tokens)
- Tool definitions (~1–5K tokens)
- Few-shot examples (~5–30K tokens)
- Document context for RAG / "long context" apps (~10–500K tokens)
- Conversation history (~grows over session)

When 95% of every request is identical to the previous request, paying full input price for every token is waste. Prompt caching shifts that to **pay-once, read-cheaply**.

---

## Why Prompt Cache Matters

### 1. Massive Cost Reduction

Anthropic's pricing model (May 2026):

| Token type | Multiplier vs base input |
|---|---|
| Cache **write** (5-min TTL) | 1.25× |
| Cache **write** (1-hour TTL) | 2.0× |
| Cache **read** | 0.1× |
| Regular input | 1.0× |
| Output | (output rate) |

```
50K-token system prompt, 1000 calls/day, 5-min TTL refresh:
─────────────────────────────────────────────────────
Without cache: 50K × 1000 × $3/MTok    = $150/day
With cache:    50K × 12   × $3.75/MTok ($2.25 cache writes, refreshed every 5min during 24h × 12 active hours)
              + 50K × 988 × $0.30/MTok ($14.82 cache reads)
              = ~$15.50/day
Savings: ~90%
```

### 2. Latency Reduction

Reading from cache skips token encoding work. On 50K-token prefixes we measure:

- TTFT (time-to-first-token) drops from ~3s to ~600ms
- End-to-end latency drops 30–60%

### 3. Enables New Architectures

Without caching, you avoid putting 200K tokens in the prompt because cost is prohibitive. With caching, you can put your **entire knowledge base** in the prompt for a single user session — replacing RAG entirely for small/mid corpora.

### 4. Real Cost Comparison

```
B2B legal AI assistant — 5K calls/day with 80K-token contract context
─────────────────────────────────────────────────────────────────────
Before prompt cache:
  Input cost:  $1,200/day
  P50 TTFT:    4.2s
  Architecture: complex RAG to keep context small

After prompt cache:
  Input cost:  $130/day  (-89%)
  P50 TTFT:    0.9s
  Architecture: full contract in prompt, cached for the session
```

---

## How Prompt Caching Works

### The Mental Model

The model's transformer state for a sequence of tokens — the **KV cache** — is what prompt caching persists. For autoregressive transformers, processing token N requires the keys/values for tokens 1..N-1. Once you've computed those, the work is reusable for any continuation that starts with the same prefix.

```
prompt = [system] [tools] [examples] [docs] [history] [user_msg]
                                                        ▲
       ◄────── stable prefix (cacheable) ──────►       new
```

The provider:
1. Hashes the cacheable region.
2. Looks up that hash in its cache.
3. **Hit:** loads the KV cache, charges read price.
4. **Miss:** computes and stores the KV cache, charges write price (+surcharge).

### Cache Match Rules — Strict Prefix

Caches only match **exact prefix bytes**, in order, from token 0. If you change one byte in the system prompt, the entire downstream cache is invalidated.

```
Stable order matters:
─────────────────────────────────────────────────────
[system] [tools]  [docs] [user]   ← cache hits when [system][tools][docs] reused
[tools]  [system] [docs] [user]   ← DIFFERENT cache entry — order swap = miss
```

The implication: **put your stable content first, volatile content last**.

### Eligibility & Limits (Anthropic-specific)

| Constraint | Value |
|---|---|
| Minimum cacheable region | 1024 tokens (Sonnet/Opus), 2048 (Haiku) |
| Max cache breakpoints per request | 4 |
| TTL options | 5 minutes (default) or 1 hour |
| Requests required to populate | 1 (write happens on first call) |
| Models supporting it | All Claude 4.x and most 3.x |

---

## Anthropic Prompt Cache — Mechanics

### Marking a Cache Breakpoint

Add `cache_control` to the content block where you want the prefix to end:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": LARGE_SYSTEM_PROMPT,           # ~8K tokens
            "cache_control": {"type": "ephemeral"} # cache up to here
        }
    ],
    messages=[
        {"role": "user", "content": user_query}
    ],
)
```

`ephemeral` is the only type. The 5-minute TTL is the default; pass `{"type": "ephemeral", "ttl": "1h"}` for the 1-hour variant.

### Multiple Breakpoints

Up to 4 cache breakpoints per request. Use them to cache nested layers that change at different rates:

```python
system=[
    {"type": "text", "text": ROLE_AND_RULES,
     "cache_control": {"type": "ephemeral", "ttl": "1h"}},      # rarely changes
    {"type": "text", "text": TOOL_DEFINITIONS,
     "cache_control": {"type": "ephemeral", "ttl": "1h"}},      # rarely changes
    {"type": "text", "text": KNOWLEDGE_BASE_CHUNK,
     "cache_control": {"type": "ephemeral"}},                   # 5min — refreshed daily
]
messages=[
    *conversation_history_with_breakpoint,                       # 5min — grows
    {"role": "user", "content": new_user_message},               # not cached
]
```

Each breakpoint creates a *cumulative* cached prefix: `[A]`, `[A][B]`, `[A][B][C]`, etc. A miss at any layer invalidates all downstream layers.

### Reading the Response — Cache Hit Confirmation

The response `usage` block tells you exactly what happened:

```json
{
  "usage": {
    "input_tokens": 47,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 12834,
    "output_tokens": 412
  }
}
```

| Field | Meaning |
|---|---|
| `input_tokens` | Tokens billed at full rate (the new, uncached portion) |
| `cache_creation_input_tokens` | Tokens written to cache this call (1.25× / 2.0× rate) |
| `cache_read_input_tokens` | Tokens read from cache (0.1× rate) |

**Always log all three.** Cost regressions show up here before they show up on the bill.

---

## Cache Breakpoints — Where to Cut

The placement of cache breakpoints determines hit rate. The rule:

> **Cut between regions of different change-rate, with the slowest-changing first.**

### A Mental Layering Model

```
Layer                          Typical Change Rate    TTL
─────────────────────────────────────────────────────
1. Role / instructions          monthly                1h
2. Tool definitions             weekly                 1h
3. Few-shot examples            weekly                 1h
4. Static knowledge / docs      daily                  5min (refreshed often)
5. Conversation history         per turn               5min
6. Current user message         every call             not cached
```

A four-breakpoint design might be:

```
[L1+L2+L3] ──cache_control──► [L4] ──cache_control──► [L5] ──cache_control──► [L6]
   1h TTL                       5m TTL                   5m TTL                no cache
```

### Anti-pattern: Volatile Token Inside Stable Region

```
SYSTEM: "You are an assistant. Today is 2026-05-04. Be helpful."
                                       ↑ flips daily, invalidates entire system prompt
```

Move volatile substitutions out of cached regions:

```
SYSTEM (cached):  "You are an assistant. Be helpful."
USER (not cached): "<context_date>2026-05-04</context_date>\n\nActual question: ..."
```

### Conversation History — the Cumulative Breakpoint

Long-running chats benefit hugely. Add a cache breakpoint on the *last user/assistant turn* in history. The next turn's prefix becomes the previous full conversation, all cacheable.

```python
def add_history_breakpoint(messages):
    if not messages:
        return messages
    last = messages[-1].copy()
    if isinstance(last["content"], str):
        last["content"] = [{"type": "text", "text": last["content"]}]
    last["content"][-1]["cache_control"] = {"type": "ephemeral"}
    return messages[:-1] + [last]
```

Each new turn re-uses everything before it.

---

## Building a Cache-Optimised Prompt

### Step 1: Audit Tokens by Stability

Profile a real session and group every token by how often it changes.

```
component                tokens   change-rate    cacheable?
────────────────────────────────────────────────────────────
system role               300     never          yes (1h)
tool defs                 2400    weekly         yes (1h)
output schema             800     weekly         yes (1h)
RAG retrieved chunks      4500    per query      sometimes (5m if reused in turn)
conversation history     12000    per turn       yes (5m)
user message              80      every call     no
```

### Step 2: Reorder Stable → Volatile

Inviolable: stable content first, exactly the same bytes every call.

### Step 3: Insert Breakpoints at Boundaries

```python
def build_request(history, user_msg, retrieved_docs):
    return {
        "model": "claude-sonnet-4-6",
        "system": [
            {"type": "text", "text": ROLE_PROMPT,
             "cache_control": {"type": "ephemeral", "ttl": "1h"}},
            {"type": "text", "text": TOOL_DEFS_TEXT,
             "cache_control": {"type": "ephemeral", "ttl": "1h"}},
            {"type": "text", "text": OUTPUT_SCHEMA_TEXT,
             "cache_control": {"type": "ephemeral", "ttl": "1h"}},
        ],
        "messages": add_history_breakpoint([
            *history,
            {"role": "user", "content": format_with_docs(user_msg, retrieved_docs)},
        ]),
    }
```

### Step 4: Verify with `usage`

Run two requests back-to-back. The second should show non-zero `cache_read_input_tokens`. If it doesn't, you have a stability bug.

### Step 5: Pre-Warm

For deterministic prefixes (e.g., a daily-refreshed knowledge base), make a single throwaway call right after deploy to populate the cache before user traffic arrives.

---

## Refresh & Invalidation

### TTL & Implicit Refresh

The 5-minute clock starts on every **read**. Continuous traffic keeps the cache hot indefinitely; idle for 5 minutes and the next request is a miss (write).

The 1-hour TTL is the same — clock resets on every read.

### When to Pick 5min vs 1h

```
choose 1h when:
- Off-peak gaps regularly exceed 5 minutes
- Write surcharge (2.0×) is amortised over many reads
- Content is genuinely stable for an hour

choose 5min when:
- Steady high-throughput traffic keeps it warm anyway
- Content might change within the hour
- Cost-sensitive — 5min has a smaller write surcharge (1.25×)
```

### Manual Invalidation

There is no "flush" API. To invalidate, change one byte of the cached prefix — e.g., bump a version stamp:

```python
ROLE_PROMPT = f"<!--prompt_v=2026-05-04-->\n{ROLE_BODY}"
```

When you publish v3, every subsequent call misses on the old cache and writes a fresh one.

### Coordinated Refresh Across Replicas

Anthropic's cache is shared across an Anthropic API account, not per-client. So as soon as one replica writes the cache, all replicas reading the same prefix get hits. No coordination needed.

---

## AI-Specific Use Cases

### 1. Long-Context Document Q&A

Put the entire document (book, contract, codebase) in the system prompt with cache_control. User asks 30 questions in a session — only the first pays full price.

```
200K-token codebase, 30 user questions:
without cache: 30 × 200K = 6M input tokens
with cache:    1 write (200K × 1.25) + 29 reads (200K × 0.1)
             = 250K + 580K = 830K eq. input tokens (-86%)
```

### 2. Multi-Turn Chatbots

Cache the conversation history at every turn. By turn 50 the user is paying ~10% of the input rate on the entire backlog.

### 3. Few-Shot Heavy Classifiers

50 in-context examples (~10K tokens) cached once, used for every classification request that day.

### 4. Tool-Using Agents

Tool definitions are large and stable. Cache them at the 1-hour TTL. Save ~30% of input cost on tool-calling agents.

### 5. Batch Inference

Even in batch jobs, prompt caching pays off when the items share a common prefix (system + schema). Submit them in temporal proximity.

---

## Advanced Patterns

### Stacking with Semantic Cache

```
1. semantic cache hit?  → return    (best case: zero LLM cost)
2. miss → call LLM with prompt cache
   → input charged at 0.1× for the cached prefix (next-best case)
3. response → write to semantic cache
```

These optimise different stages and **compose multiplicatively** on cost.

### Layered TTL Strategy

```
Innermost cached layer:   1h TTL  (rarely changes)
Middle layer:             1h TTL
Outer cached layer:       5m TTL  (changes daily)
                           ↑ outer-layer miss does NOT invalidate inner reads —
                             provider still re-uses inner KV cache up to the divergence point
```

Layering minimises total write surcharge while keeping volatile content fresh.

### Cache-Aware Routing

If your LLM gateway routes requests across multiple Anthropic accounts (e.g., for rate limits), pin the same `(tenant, prompt_version)` to the same account so the cache stays hot. Round-robin routing across accounts kills cache hit rate.

### Prompt Version Pinning

```python
@dataclass
class PromptVersion:
    role_v: str          # "2026-05-04"
    tools_v: str         # "2026-04-29"
    schema_v: str        # "2026-05-01"

def cache_key_stamp(v: PromptVersion) -> str:
    return f"<!--p:{v.role_v}|{v.tools_v}|{v.schema_v}-->"
```

Putting the stamp inside the cached region gives you a clean invalidation lever without the cost of guesswork.

---

## Monitoring & Observability

### Required Metrics

| Metric | What to Track |
|---|---|
| `cache_read_tokens / total_input_tokens` | Hit rate by tokens (target ≥ 80% on stable workloads) |
| `cache_creation_tokens / total_input_tokens` | Write share (high = volatility you didn't expect) |
| `cache_savings_usd` | `(read_tokens × 0.9 + creation_tokens × -0.25) × rate` |
| `ttft_p50 / ttft_p99` | Latency win |
| `prompt_version_id` | Track invalidations over time |

### Logging Recipe

```python
def log_usage(usage, request_id):
    cache_total = usage.cache_read_input_tokens + usage.cache_creation_input_tokens
    total_in = usage.input_tokens + cache_total
    hit_pct = (usage.cache_read_input_tokens / total_in * 100) if total_in else 0
    log.info("llm.usage",
        request_id=request_id,
        input=usage.input_tokens,
        cache_read=usage.cache_read_input_tokens,
        cache_write=usage.cache_creation_input_tokens,
        output=usage.output_tokens,
        cache_hit_pct=round(hit_pct, 1),
    )
```

### Alerts

```
ALERT: cache_hit_pct < 60% for 10 min
       → prompt template changed accidentally OR
         volatile token slipped into cached region OR
         traffic dropped below TTL warmth threshold
```

---

## Integration with Other Patterns

### + Semantic Cache

Stacked: semantic cache eliminates the call; prompt cache discounts the call when it happens. Both at once is standard production practice.

### + RAG

Two interactions:

1. **Cache the retrieval-irrelevant scaffolding** — system prompt, schema, tools — as a stable prefix.
2. **Don't cache the retrieved chunks** unless they're reused (e.g., session-scoped retrieval). Inserting per-query chunks in the cached prefix kills hit rate.

### + Plan-and-Execute

The planner's prompt (instructions + tools + examples) is a perfect fit for 1h TTL. The executor's per-step prompt typically isn't — its inputs vary too much.

### + Cost Guardrails

Prompt caching reshapes the cost model. Recompute budgets after enabling, then attach guardrails (token-budget circuit breaker) to the residual spend.

### + Streaming

Prompt cache works transparently with streaming endpoints. TTFT improvements compound: cached prefix means the model starts emitting tokens much sooner.

---

## Production Best Practices

### 1. Stable Bytes, Always

Resist string interpolation in cached regions. Even whitespace differences create misses.

### 2. Order Stable → Volatile

If you can't put a region first, you can't cache it. Reorder the request.

### 3. Use Both TTLs

1h for the deep, rarely-changing layers; 5m for the fast-moving outer layers. Don't pick one.

### 4. Cache the Tools

Tool definitions are usually 1–5K tokens, stable for weeks. Almost always worth a breakpoint.

### 5. Pre-Warm After Deploy

Push one no-op request through the new prompt template to populate the cache before user traffic arrives.

### 6. Record `prompt_version` per Request

When you investigate a regression, you need to know which prompt version a given trace used.

### 7. A/B Test Cache Layouts

Two breakpoint placements can yield very different hit rates on real traffic. Run them in shadow before promoting.

### 8. Don't Try to Cache Below the Minimum

Anything under 1024 tokens (Sonnet/Opus) silently won't be cached. Combine small layers into one cacheable block.

---

## Common Mistakes

1. **Volatile token in cached region** (timestamp, request_id, user name) — invalidates everything downstream.
2. **Wrong order** — cached content not first means it's not actually cached.
3. **Reordering tool definitions** — different serialisation of the same tools = different bytes = miss.
4. **Embedding RAG chunks in cached prefix when they vary per query** — pure write surcharge, no reads.
5. **Ignoring usage metrics** — assuming caching works without verifying read tokens.
6. **Choosing 1h TTL for low-volume traffic** — write surcharge (2.0×) never amortises.
7. **Forgetting to bump version stamps** — old cached prompt continues serving after a deliberate change.
8. **Cross-account routing** — round-robin across Anthropic accounts splits the cache.

---

## Interview Cheat Sheet

**Q: How does Anthropic's prompt cache work?**
A: You mark a region of the prompt with `cache_control: ephemeral`. The provider hashes that prefix; reads cost 10% of input rate, writes cost 125% (5min TTL) or 200% (1h). TTL refreshes on each read. Up to 4 breakpoints per request.

**Q: How do I get a high hit rate?**
A: Put stable content first (system, tools, schema), volatile content last (user message). Don't interpolate anything that changes (timestamps, IDs) into cached regions. Verify with `usage.cache_read_input_tokens`.

**Q: When is the write surcharge not worth it?**
A: When the cache will be read fewer than ~3 times before TTL expiry. Steady traffic always wins; very bursty/sparse traffic on a 1h TTL can be a net loss.

**Q: Difference between prompt cache and semantic cache?**
A: Prompt cache reuses the model's intermediate KV state across requests with the same prefix — provider-side, exact-prefix match, discounts the call. Semantic cache reuses the response itself across semantically similar requests — application-side, eliminates the call. They compose.

**Q: Can I invalidate a cache entry on demand?**
A: No flush API. Change one byte (e.g., a version stamp comment) and the next call writes a fresh entry; the old one ages out.

**Q: Does it work with streaming and tool use?**
A: Yes, transparently. TTFT improvements are larger with streaming because the initial encode work is what gets skipped.

**Q: What if my prompts vary slightly per user (e.g., user name)?**
A: Move per-user fields out of the cached prefix into the final user message. Or use templated placeholders only in the *post-breakpoint* section.

---

**Document Version:** 1.0
**Last Updated:** May 2026
