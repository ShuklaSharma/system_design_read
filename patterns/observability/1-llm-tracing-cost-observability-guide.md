# LLM Tracing & Cost Observability Pattern

> AI Engineering Series | May 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why LLM Observability is Different](#why-llm-observability-is-different)
3. [The Three Pillars (and a Fourth)](#the-three-pillars-and-a-fourth)
4. [Trace Model — What to Capture](#trace-model--what-to-capture)
5. [OpenTelemetry GenAI Semantic Conventions](#opentelemetry-genai-semantic-conventions)
6. [Building an Instrumented LLM Client — Step by Step](#building-an-instrumented-llm-client)
7. [Cost Accounting](#cost-accounting)
8. [Quality & Eval Observability](#quality--eval-observability)
9. [Tracing Agents and Tool Calls](#tracing-agents-and-tool-calls)
10. [Backends: Self-Hosted vs Managed](#backends-self-hosted-vs-managed)
11. [AI-Specific Use Cases](#ai-specific-use-cases)
12. [Integration with Other Patterns](#integration-with-other-patterns)
13. [Production Best Practices](#production-best-practices)
14. [Common Mistakes](#common-mistakes)
15. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## Introduction

### What is LLM Observability?

**LLM observability** is the practice of capturing, structuring, and querying every signal needed to understand and operate LLM-based systems in production:

- **Traces** of every model call, tool call, retrieval step, and guardrail decision.
- **Token-level cost** attributed to user, feature, tenant, model, and prompt version.
- **Quality signals** — automated evals, user feedback, human review labels.
- **Latency** broken down by component (TTFT, TPOT, end-to-end).

It is to LLM apps what APM (Datadog, New Relic) is to web apps: not optional, not a future-state goal, the prerequisite for running anything seriously.

### Why a Dedicated Pattern

Generic APM was designed for short, deterministic, cheap requests. LLM workloads break those assumptions:

- A single user request may invoke 5–50 model calls in a chain.
- Costs vary 1000× across models and prompt sizes.
- The same input produces different outputs.
- "Errors" include not just exceptions but wrong, unsafe, or hallucinated outputs.
- Tokens — not requests — drive cost and latency.

A pattern dedicated to LLM observability gives you the right primitives: token-aware spans, cost rollups, eval results-as-data, prompt versioning.

---

## Why LLM Observability is Different

### 1. Cost is in the Data Plane

A single request can cost $0.0001 or $5.00 depending on model, tokens, tools, retries, and reflection loops. Without per-request cost capture you cannot:

- Bill or rate-limit by tenant.
- Detect runaway cost regressions.
- Decide when caching/routing changes pay off.

Cost is a first-class signal, alongside latency and errors.

### 2. Quality is Probabilistic

A request can return HTTP 200 with a wrong, biased, or unsafe answer. "Did it work?" requires evaluating output, not just the response code.

### 3. Causality is Multi-Step

An agent loop has dozens of steps: planning, retrieval, tool calls, reflection, synthesis. Debugging "why did it answer wrong" requires reconstructing that whole tree, not just the final call.

### 4. Reproduction is Hard

Same prompt + same model = different output. Without captured request/response pairs and a `seed`/`temperature`/`prompt_version`, regressions are impossible to reproduce.

### 5. Real Incident Without Observability

```
Day 1: prompt template change goes out
Day 2: user complaints uptick — "the bot keeps refusing me"
Day 3: support escalates, but no idea which calls are bad
Day 5: someone notices Anthropic spend doubled (but quality is reportedly worse)
Day 7: rollback all prompt changes blindly
Day 10: bisected to a single line in the system prompt — by reading 200 random log lines
```

With proper observability, this is a 30-minute investigation: filter traces by `prompt_version`, look at quality-eval delta, blame the regressing change.

---

## The Three Pillars (and a Fourth)

Generic observability has three pillars: **logs, metrics, traces**. LLM systems add a fourth: **evals**.

```
┌──────────────────────────────────────────────────────────┐
│                          LOGS                            │
│   structured request/response pairs, prompts, tools      │
├──────────────────────────────────────────────────────────┤
│                         METRICS                          │
│   token count, cost, latency, hit rate, block rate       │
├──────────────────────────────────────────────────────────┤
│                         TRACES                           │
│   parent/child spans for chains, agents, tools, retries  │
├──────────────────────────────────────────────────────────┤
│                          EVALS                           │
│   automated + human quality labels per output            │
└──────────────────────────────────────────────────────────┘
```

Each pillar must be **joinable** by `request_id` (and ideally `tenant_id`, `user_id`, `prompt_version`). Without joins, you have four piles of data and no insight.

---

## Trace Model — What to Capture

### The LLM Span

For every model call, a span with these attributes:

| Field | Example | Why |
|---|---|---|
| `request_id` | `r_91kf...` | Top-level request correlation |
| `parent_span_id` | `s_4a2...` | Reconstruct chain |
| `span_kind` | `llm.completion` | Filter by stage |
| `gen_ai.system` | `anthropic` | Vendor |
| `gen_ai.request.model` | `claude-sonnet-4-6` | Cost / capability |
| `gen_ai.usage.input_tokens` | `2,341` | Cost driver |
| `gen_ai.usage.output_tokens` | `478` | Cost driver |
| `gen_ai.usage.cache_read_input_tokens` | `1,800` | Prompt-cache savings |
| `gen_ai.usage.cache_creation_input_tokens` | `0` | Cache writes |
| `gen_ai.response.finish_reason` | `end_turn`, `tool_use`, `max_tokens` | Outcome |
| `gen_ai.request.temperature` | `0.0` | Reproduction |
| `gen_ai.prompt.hash` | `sha256(...)` | Prompt-version correlation |
| `prompt_version` | `2026-05-04` | What changed |
| `tenant_id`, `user_id`, `feature` | — | Attribution |
| `latency.ttft_ms` | `412` | Streaming UX metric |
| `latency.total_ms` | `2,180` | End-to-end |
| `cost_usd` | `0.0123` | Pre-computed dollars |
| `error` | optional | Status |

### What NOT to Put in Span Attributes

- Full prompt and response bodies — too big, may contain PII. Store in a separate blob store keyed by `prompt.hash` / `response.hash`. Spans get the hash; bodies get a TTL.
- Unhashed PII.
- Raw embeddings (large, low-value in spans).

### Hierarchy of Spans

```
[request: /chat]                                       (root)
  ├─ [guardrail.input]
  ├─ [retrieval.vector_search]
  │    ├─ [embed]
  │    └─ [vector.query]
  ├─ [llm.completion model=opus tools=true]
  │    ├─ [tool.get_order]
  │    └─ [llm.completion model=opus]                  (continuation after tool)
  ├─ [reflection.critique model=sonnet]
  ├─ [llm.completion model=opus]                       (revision)
  ├─ [guardrail.output]
  └─ [render]
```

This tree, viewable in any tracing UI (Jaeger, Tempo, Honeycomb, Langfuse), is the unit of debugging.

---

## OpenTelemetry GenAI Semantic Conventions

OpenTelemetry has a **GenAI semconv** standardising the attribute names above. Use it from day one — it makes your traces interoperable with Langfuse, Phoenix, Honeycomb, Datadog, Grafana Cloud, and self-hosted Jaeger.

Key namespaces:

```
gen_ai.system              vendor (anthropic, openai, bedrock, ...)
gen_ai.request.model
gen_ai.request.temperature
gen_ai.request.max_tokens
gen_ai.request.top_p
gen_ai.usage.input_tokens
gen_ai.usage.output_tokens
gen_ai.usage.cache_read_input_tokens     (extension; widely used)
gen_ai.usage.cache_creation_input_tokens (extension)
gen_ai.response.id
gen_ai.response.finish_reasons
gen_ai.tool.name
gen_ai.tool.call.id
```

### Example: OpenTelemetry Span (Python)

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind
import anthropic, time, hashlib, json

tracer = trace.get_tracer("llm.app")
client = anthropic.Anthropic()

PRICE = {  # $ per million tokens (May 2026 illustrative)
    "claude-opus-4-7":   {"in": 15.00, "out": 75.00, "cache_read": 1.50, "cache_write": 18.75},
    "claude-sonnet-4-6": {"in":  3.00, "out": 15.00, "cache_read": 0.30, "cache_write":  3.75},
    "claude-haiku-4-5-20251001": {"in": 0.80, "out": 4.00, "cache_read": 0.08, "cache_write": 1.00},
}

def cost_usd(model, usage):
    p = PRICE[model]
    m = 1_000_000
    return (
        usage.input_tokens               * p["in"]          / m
      + usage.output_tokens              * p["out"]         / m
      + usage.cache_read_input_tokens    * p["cache_read"]  / m
      + usage.cache_creation_input_tokens * p["cache_write"] / m
    )

def prompt_hash(messages, system):
    h = hashlib.sha256()
    h.update(json.dumps({"s": system, "m": messages}, sort_keys=True).encode())
    return h.hexdigest()[:16]

def traced_complete(model, system, messages, **kwargs):
    with tracer.start_as_current_span("llm.completion", kind=SpanKind.CLIENT) as span:
        span.set_attribute("gen_ai.system", "anthropic")
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.request.temperature", kwargs.get("temperature", 1.0))
        span.set_attribute("gen_ai.prompt.hash", prompt_hash(messages, system))

        t0 = time.perf_counter()
        try:
            resp = client.messages.create(
                model=model, system=system, messages=messages, **kwargs
            )
        except anthropic.APIError as e:
            span.set_attribute("error", True)
            span.set_attribute("error.type", type(e).__name__)
            raise

        latency_ms = (time.perf_counter() - t0) * 1000
        u = resp.usage
        span.set_attribute("gen_ai.usage.input_tokens", u.input_tokens)
        span.set_attribute("gen_ai.usage.output_tokens", u.output_tokens)
        span.set_attribute("gen_ai.usage.cache_read_input_tokens", u.cache_read_input_tokens)
        span.set_attribute("gen_ai.usage.cache_creation_input_tokens",
                           u.cache_creation_input_tokens)
        span.set_attribute("gen_ai.response.id", resp.id)
        span.set_attribute("gen_ai.response.finish_reasons", [resp.stop_reason or "unknown"])
        span.set_attribute("latency.total_ms", latency_ms)
        span.set_attribute("cost_usd", cost_usd(model, u))
        return resp
```

That single decorator gives you cost, tokens, latency, vendor, and prompt-version correlation — emitted via OTLP to whatever backend you choose.

---

## Building an Instrumented LLM Client

### Step 1: Wrap the SDK

Don't sprinkle tracing across business logic. Wrap the SDK once, all callers get instrumentation for free. Pattern shown above.

### Step 2: Capture Streaming Properly

```python
def traced_stream(model, system, messages, **kwargs):
    with tracer.start_as_current_span("llm.stream") as span:
        t0 = time.perf_counter()
        ttft_ms = None
        usage = None
        with client.messages.stream(model=model, system=system,
                                    messages=messages, **kwargs) as stream:
            for event in stream:
                if ttft_ms is None and event.type == "content_block_delta":
                    ttft_ms = (time.perf_counter() - t0) * 1000
                    span.set_attribute("latency.ttft_ms", ttft_ms)
                yield event
            usage = stream.get_final_message().usage
        if usage:
            span.set_attribute("gen_ai.usage.input_tokens", usage.input_tokens)
            span.set_attribute("gen_ai.usage.output_tokens", usage.output_tokens)
            span.set_attribute("cost_usd", cost_usd(model, usage))
        span.set_attribute("latency.total_ms", (time.perf_counter() - t0) * 1000)
```

TTFT is a real UX metric; total latency alone hides it.

### Step 3: Body Storage Separately

```python
def store_body(hash_, body, ttl_days=7):
    blob_store.put(f"prompts/{hash_}", body, ttl=ttl_days * 86400)
```

Spans reference bodies by hash. PII / cost reasons; also lets traces stay small.

### Step 4: Propagate Context

Use OpenTelemetry context propagation across services so a single user request keeps one `trace_id` end-to-end (frontend → API → vector DB → LLM gateway → callbacks).

### Step 5: Sample Smartly

Don't trace 100% in high-traffic systems — too expensive. Apply tail-based sampling:

- **Always** sample errors, blocked requests, and slow requests.
- **Always** sample a fixed % of all requests for baseline.
- **Sometimes** sample ordinary success cases.

```
sampling rule:
  IF error OR latency > p99 OR cost > $0.50 OR guardrail.blocked: 100%
  ELSE: 5%
```

---

## Cost Accounting

### Compute Cost in the Hot Path

Compute `cost_usd` per span at emission time (as in the wrapper above). Don't try to back-compute from logs later — pricing changes, you'll get drift.

### Roll Up Dimensions

You need to slice cost by every dimension you might bill or budget on. Recommend OLAP on your trace store:

```sql
SELECT
  tenant_id,
  feature,
  prompt_version,
  gen_ai_request_model AS model,
  date_trunc('hour', ts) AS hour,
  SUM(cost_usd) AS spend,
  SUM(gen_ai_usage_input_tokens) AS in_tokens,
  SUM(gen_ai_usage_output_tokens) AS out_tokens,
  SUM(gen_ai_usage_cache_read_input_tokens) AS cache_read,
  COUNT(*) AS calls
FROM llm_spans
WHERE ts >= now() - INTERVAL '24 hours'
GROUP BY 1, 2, 3, 4, 5
ORDER BY spend DESC;
```

### Budgets and Guardrails

Hard caps per dimension, enforced before the call:

```python
def check_budget(tenant_id, feature):
    spent_today = budget_store.get(tenant_id, feature)
    cap = budgets[(tenant_id, feature)]
    if spent_today >= cap:
        raise BudgetExceeded(tenant=tenant_id, feature=feature, cap=cap)
```

Persist budget consumption *atomically* with span emission so a crash can't double-count or drop.

### Cost Anomaly Detection

```
ALERT  per-tenant cost > 3× rolling 7-day median for that tenant
ALERT  per-feature cost > daily budget × 80% before noon
ALERT  cache_read_share drops below 50% of its 7-day mean
       (someone broke prompt caching)
```

### Showback / Chargeback

Tag every span with `business_unit`, `cost_center`, `feature` if you internally bill departments. The cost rollups become invoices.

---

## Quality & Eval Observability

### Online Evaluation

Score every (or every-Nth) production response in flight, async:

```python
def online_eval(request_id, prompt, response):
    score = anthropic.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=100,
        system="Score the response 1-5 on: helpfulness, factuality, safety. JSON only.",
        messages=[{"role": "user", "content": f"PROMPT:{prompt}\nRESPONSE:{response}"}],
    )
    emit_metric("eval_score", score, request_id=request_id)
```

Write the score back as a metric tagged with `request_id`. Now you can:

- Track quality drift over time.
- Correlate quality with `prompt_version`, `model`, `cache_hit_pct`.
- Trigger alerts on quality regressions.

### Offline Evaluation

Maintain a **golden dataset** of (input, expected) pairs. On every prompt or model change, replay against the golden set:

```
prompt_version=2026-05-04:
   golden_pass_rate    87.3%
   p95 latency         1.4s
   avg cost            $0.012

prompt_version=2026-05-05 (proposed):
   golden_pass_rate    79.1%   ← regression — DO NOT SHIP
   p95 latency         1.1s
   avg cost            $0.009
```

CI gates the deploy on `golden_pass_rate >= baseline - epsilon`.

### User Feedback

Capture explicit (👍/👎, 1–5 stars, free-text) and implicit (regenerate clicks, abandonment, conversion) feedback. Join via `request_id`.

### Eval-as-Data

Store eval results in the same warehouse as traces. Dashboards correlate everything:

```
quality(week N) vs cost(week N) vs cache_hit(week N) vs prompt_version
```

This is how you find that "the new prompt is 12% cheaper but 8% lower quality — net negative."

---

## Tracing Agents and Tool Calls

Agent traces are deeper and noisier than chat traces. A few rules:

### Span Per Step, Not Per Token

A reasonable cap is one span per LLM call, one span per tool call, one root span per user request. Avoid per-token spans — the volume kills your trace store.

### Tool Spans Carry Args & Results (Hashed)

```python
with tracer.start_as_current_span("tool.get_order") as span:
    span.set_attribute("gen_ai.tool.name", "get_order")
    span.set_attribute("gen_ai.tool.call.id", call_id)
    span.set_attribute("tool.args.hash", sha256(args_json))
    result = tools.get_order(**args)
    span.set_attribute("tool.result.size_bytes", len(json.dumps(result)))
    return result
```

### Loop Iteration Counts

For ReAct / Plan-and-Execute / Reflection, attribute the iteration count to the root span — `agent.iterations: 4`. Easy filter for runaway loops.

### Cost Per Agent Run

Sum `cost_usd` across all child spans, attach to root. A single dashboard column shows you the most expensive agent runs.

```sql
SELECT request_id, SUM(cost_usd) total, COUNT(*) spans
FROM llm_spans
GROUP BY request_id
ORDER BY total DESC LIMIT 50;
```

---

## Backends: Self-Hosted vs Managed

### Self-Hosted OpenTelemetry Stack

```
app  ──► OTel Collector ──► [Tempo for traces]  + [Prometheus for metrics] + [ClickHouse for spans-as-data]
                          ──► [Loki for logs]
                                   │
                                   ▼
                                Grafana
```

**Pros:** No vendor lock-in, raw data ownership, cheap at scale.
**Cons:** You operate it. Schema design is on you.

### Managed Providers

| Provider | Strength |
|---|---|
| Langfuse (self-host or cloud) | Purpose-built for LLM tracing + evals + prompt mgmt |
| Arize Phoenix | OSS LLM tracing with evals, integrates with OTel |
| Helicone | LLM gateway + observability in one |
| Honeycomb / Datadog / New Relic | Generic APM with GenAI semconv support |
| Grafana Cloud | Managed Tempo/Prometheus/Loki, OSS-compatible |

### Decision Guide

```
Just starting + small team:         Langfuse (cloud) or Phoenix
Already on Datadog/Honeycomb:       Use existing + GenAI semconv
Strict data residency:              self-host OTel + Tempo + ClickHouse
Want LLM-specific eval UI:          Langfuse / Phoenix on top of existing OTel
```

A common production setup: emit OTel-format spans → fork to (a) Tempo for traces, (b) Langfuse for LLM-specific eval & prompt management. Both consume the same emission.

---

## AI-Specific Use Cases

### Chatbots / Customer Support

- Daily cost per tenant.
- TTFT and total latency per intent.
- Quality eval score weekly trend.
- Top-10 most expensive sessions.

### Agents with Tools

- Iterations per request distribution.
- Tool call success rate per tool.
- Cost per agent run; flag outliers.
- Time-to-first-tool-call (planning latency).

### RAG Pipelines

- Retrieval recall@k (offline eval).
- Groundedness rate (online eval).
- Cache hit rate at semantic / prompt layers.
- Per-document utilisation (which chunks ever get cited).

### Code-Generating Apps

- Test-pass rate by prompt version.
- Iteration distribution if reflection is on.
- Cost per accepted suggestion.

### Multi-Tenant SaaS

- Showback dashboard per tenant.
- Anomaly detection on tenant cost spikes.
- Per-tenant quality (some tenants have harder queries).

---

## Integration with Other Patterns

### + Prompt Caching

Track `cache_read_input_tokens` and `cache_creation_input_tokens` as first-class metrics. They are the only way to verify caching is actually working.

### + Semantic Cache

Emit a span for every cache lookup with `hit=true|false`, `similarity`, `matched_query_hash`. Combined view: cost saved by semantic cache vs cost spent on embedding lookups.

### + Reflection

`agent.iterations` and per-iteration cost rollup. Without it, reflection loops are invisible cost.

### + Guardrails

Every block / transform decision is a span event. Searchable: "how many requests did we block last week, by reason?"

### + Cost Guardrails / Budgets

Budget enforcement runs *off the same metrics* this pattern emits. The observability layer is the substrate, the guardrail is the consumer.

### + Plan-and-Execute / Tool Use

Trace tree mirrors the agent loop exactly. The tracing UI becomes the agent debugger.

---

## Production Best Practices

### 1. Use OpenTelemetry GenAI Semconv

Don't invent attribute names. Future-you and every tool you might integrate need standard names.

### 2. Cost in the Span, Not Reconstructed Later

Compute `cost_usd` at emit time using a canonical price table. Pricing changes break post-hoc reconstruction.

### 3. Hash Bodies, Store Separately

Span size matters. Hashes also let you join "all calls with this prompt" easily.

### 4. Tag with `prompt_version` Always

Single most useful filter for diagnosing regressions.

### 5. Tag with `tenant_id` Always

Multi-tenant cost & quality slicing requires it. Even single-tenant systems benefit from tagging environment.

### 6. Sample Smartly, Always 100% on Errors

Tail-based sampling keeps cost bounded without losing the interesting cases.

### 7. Track Cache Metrics as First-Class

`cache_read_share`, `cache_write_share`, `semantic_hit_rate`. These tell you whether your optimisations are working.

### 8. Online Eval at Low Sample Rate

10% online-eval coverage is enough to track quality drift. 100% is wasteful.

### 9. Maintain a Golden Eval Set

Versioned. CI-gated. The single most leveraged practice for not regressing quality.

### 10. Dashboards Per Audience

Engineers see latency/error/iteration. Product sees quality/feedback. Finance sees cost-per-feature/tenant. Same data, three views.

### 11. Reverse-Lookup From Body

When a user reports a bad answer, hashing their copy-pasted text and grepping `gen_ai.response.hash` finds the exact trace. Wire this lookup into your support tool.

### 12. PII Hygiene in Spans

Even attribute fields can leak (e.g., long `user_input` strings). Lint your span emission for sensitive shapes.

---

## Common Mistakes

1. **No request_id propagation.** Pillars don't join. Debugging is impossible.
2. **Cost reconstructed from logs.** Pricing drift makes numbers wrong; nobody trusts the dashboard.
3. **Full prompts in span attributes.** Span store balloons; PII risk.
4. **No prompt_version tag.** Regressions can't be traced to a change.
5. **Sampling everything at the same rate.** Either cost runs away or you miss errors.
6. **Treating evals as a release-time activity.** Without online eval, drift goes unnoticed for weeks.
7. **No TTFT for streaming.** Total latency hides the perceived UX.
8. **Per-token spans.** Trace store explodes.
9. **Custom attribute names.** Vendor lock-in or rebuild later when integrating standard tools.
10. **No golden set.** "It looks fine in eyeballs review" — until it doesn't.

---

## Interview Cheat Sheet

**Q: What's different about LLM observability?**
A: Cost varies 1000× per request and is a first-class signal; "errors" include wrong/unsafe answers, not just exceptions; causality spans many model+tool steps; same input produces different outputs. So the model adds tokens/cost as core dimensions, makes evals a fourth pillar, and emphasises trace trees over flat logs.

**Q: What do you put in a model-call span?**
A: Vendor, model, input/output token counts, cache read/write tokens, finish reason, latency (TTFT + total), cost in USD, prompt hash, prompt version, tenant id, feature, request id. All using OpenTelemetry GenAI semantic conventions for portability.

**Q: How do you compute cost?**
A: At span-emission time, with a canonical price table indexed by (model, token-type). Store `cost_usd` as a numeric attribute. Don't rebuild from raw token counts later — pricing drifts.

**Q: How do you debug a "the bot gave a wrong answer" report?**
A: Find the trace by `request_id` (or hash of the user's quoted output). Walk the span tree: which retrieved chunks, which tool calls, which prompt version, which model, which final response. Compare against the golden set, look at the eval score.

**Q: Online vs offline eval?**
A: Online eval scores live production traffic at a sample rate (e.g., 10%) to track drift. Offline eval runs on a versioned golden set in CI, gating prompt/model deploys. You need both: offline catches regressions before prod, online catches drift in prod.

**Q: How do you avoid blowing up trace storage?**
A: Tail-based sampling (100% on errors / slow / expensive, fixed-% baseline elsewhere), one span per LLM call (not per token), hash-and-store bodies separately with TTL, trim attributes to GenAI semconv standards.

**Q: Self-host or managed?**
A: For LLM-specific UI (prompt mgmt, eval views) use Langfuse or Phoenix on top of the OTel stack. If you already have Datadog/Honeycomb, add GenAI semconv to the existing pipeline. Self-host the full OTel stack when data residency or cost at scale matters.

---

**Document Version:** 1.0
**Last Updated:** May 2026
