# 02 — Cost aur Latency Optimization ⭐

> **Is file ka goal:** Concrete, high-impact optimizations — priority order mein. Ye wo file hai jo tumhare AWS/API bill ko half kar sakti hai.

---

## Part 1: Pehle measure karo

```python
# Har request pe ye track karo
{
    "trace_id": "...",
    "input_tokens": 6_200,
    "cached_input_tokens": 5_100,      # ⭐ cache hit
    "output_tokens": 380,
    "model": "claude-opus-5",
    "cost_usd": 0.0142,
    "latency_ms": {"total": 4_800, "retrieve": 300, "rerank": 200,
                   "prefill": 700, "decode": 3_600},
}
```

**Bina ye data ke optimize karna guess hai.** Pehla kadam hamesha instrumentation hai.

### Cost formula

```
cost = (input_tokens_uncached × input_price
        + input_tokens_cached × input_price × 0.1
        + output_tokens × output_price) / 1_000_000
```

**Key facts:**
- Output tokens input se **4-5x mehnge** hote hain
- Cached input ~**90% sasta** hota hai
- Isliye: **output kam karo, cache maximize karo**

---

## Part 2: Cost optimization — priority order 💰

### #1 Model routing (60-80% saving) 🔥

```python
COMPLEXITY_RULES = {
    "trivial": ("haiku-class", None),        # classify, extract, route
    "simple":  ("sonnet-class", "low"),      # summarize, draft
    "medium":  ("sonnet-class", "medium"),
    "hard":    ("opus-class", "high"),       # analysis, debugging
}

async def route_and_call(task):
    complexity = await classify(task)         # chhota model, ~50 tokens
    model, effort = COMPLEXITY_RULES[complexity]
    return await call(model, task, effort=effort)
```

**Cascade pattern (aur better):**
```python
async def cascade(task):
    result = await call(CHEAP_MODEL, task)
    if validates(result) and result.confidence > 0.7:
        return result                          # 80% requests yahin khatam
    return await call(STRONG_MODEL, task)      # 20% escalate
```

> **Ye #1 lever hai.** Har task ke liye sabse bada model use karna = paisa jalana. Apne eval set pe har tier test karo — aksar chhota model 93% karta hai jahan bada 95% karta hai.

### #2 Prompt caching (50-90% input cost) 🔥

```python
# ✅ Cache-friendly structure
system = FROZEN_SYSTEM_PROMPT               # kabhi na badle
tools  = sorted(TOOLS, key=lambda t: t["name"])   # deterministic order
# ── cache breakpoint ──
messages = [*history, {"role": "user", "content": query}]
```

**Cache killers (silent!):**
```python
❌ system = f"You are helpful. Time: {datetime.now()}"     # har request naya
❌ system = f"Helping {user.name} from {user.company}"     # per-user cache
❌ tools  = build_tools(user.permissions)                  # position 0 pe
❌ system = json.dumps(config)                             # sort_keys=True bhoola
❌ if flag: system += "..."                                # har combination alag
```

**Verify karo:**
```python
assert response.usage.cache_read_input_tokens > 0, "Cache miss! Investigate."
```

> 🚨 **`cache_hit_rate` ko monitor karo aur uspe alert lagao.** Ek chhota prompt change cache tod deta hai, **koi error nahi aata**, aur bill 10x ho jaata hai. Ye sabse common silent cost disaster hai.

### #3 Output length (30-50% cost + latency) 🔥

```python
# Prompt mein
"Jawab concise rakho — 2-3 sentences. Detail tabhi do jab explicitly maanga jaaye."

# API mein
max_tokens=1024        # reasonable cap, 4096 nahi "just in case"
```

**Ye dono cost aur latency ek saath fix karta hai** — kyunki output tokens mehnge bhi hain aur slow bhi.

### #4 Semantic caching (15-30% requests skip)

```python
async def cached_query(query, tenant_id, threshold=0.97):
    q_emb = await embed(query)
    hits = await cache_index.search(q_emb, k=1,
                                    filter={"tenant": tenant_id})   # 🔒
    if hits and hits[0].score > threshold:
        metrics.incr("cache.semantic.hit")
        return hits[0].answer

    answer = await full_pipeline(query)
    await cache_index.add(q_emb, answer, tenant_id, ttl=3600)
    return answer
```

⚠️ **Threshold high rakho (0.95-0.98)** — negation aur numbers pe embeddings blind hain:
```
"Delhi mein baarish hui?" vs "Delhi mein baarish NAHI hui?"  → 0.95 similarity
→ galat cache hit → ULTA answer 💥
```

### #5 `top_k` reduction (20-30% input)

```
top_k = 20 → 4,000 tokens context
top_k = 5  → 1,000 tokens context

Aur accuracy aksar BADHTI hai (kam noise)!
```
Reranking ke saath ye safe hai — top-50 retrieve karo, top-5 pass karo.

### #6 Batch API (~50% saving)

Real-time zaroorat nahi? Batch API use karo:
```python
# Nightly classification, bulk extraction, eval runs
batch = client.messages.batches.create(requests=[...])
# Results within 24h, ~50% sasta
```

### #7 Context pruning (agents mein 30-60%)

```python
# Purane tool results hata do — structure preserve karo
if step - result_step > 5:
    content = "[purana result hataya gaya]"
```

---

## Part 3: Latency optimization — priority order ⚡

### Latency budget

```
Typical RAG request:
  Network + auth        :   50 ms
  Semantic cache lookup :   30 ms
  Query rewrite         :  300 ms   (skip agar single-turn)
  Embedding             :   80 ms
  Vector search         :  200 ms
  Reranking             :  200 ms
  ──────────────────────────────
  LLM prefill           :  700 ms   → cached: 100ms
  LLM decode            : 3,600 ms  🔥 SABSE BADA
  ──────────────────────────────
  Post + logging        :   50 ms
  TOTAL                 : ~5.2 sec
```

### #1 Streaming (perceived 10x) 🔥

```
Without: [────── 5.2s wait ──────] → answer
With:    [1.2s] → t→o→k→e→n→s...
```
**Total time same, perceived latency 4x better.** Ek line ka change.

### #2 Output length (linear impact)

400 tokens → 150 tokens = decode 3.6s → 1.4s.

### #3 Prompt caching (prefill 5x)

700ms → 100ms. Free lunch.

### #4 Parallelize independent work

```python
# ❌ Serial: 380ms
q_rewritten = await rewrite(query)
embedding = await embed(q_rewritten)

# ✅ Parallel jahan possible
async with asyncio.TaskGroup() as tg:
    intent_task = tg.create_task(classify_intent(query))
    embed_task  = tg.create_task(embed(query))
# → max(80, 100) = 100ms instead of 180ms
```

### #5 Chhota model (2-3x)

Routing se automatically milta hai.

### #6 Reranking optimize

- Top-50 rerank karo, top-100 nahi
- Chhota reranker
- Low-latency tier ke liye skip karo

### #7 Connection pooling / warm connections

```python
# Client ko module level pe banao, har request pe nahi
client = AsyncAnthropic(max_retries=2, timeout=60.0)
```

---

## Part 4: Cost monitoring

```python
class CostTracker:
    async def record(self, trace_id, tenant_id, usage, model):
        cost = compute_cost(usage, model)
        await db.insert_usage(trace_id, tenant_id, usage, cost, model)

        # Real-time budget check
        spend = await self.month_spend(tenant_id)
        budget = await self.get_budget(tenant_id)

        if spend > budget:
            raise BudgetExceeded(tenant_id)
        if spend > budget * 0.8:
            await alert(f"{tenant_id} 80% budget pe hai")
```

### Dashboards jo actually chahiye

| Metric | Kyun |
|---|---|
| **Cost per request** (P50, P95) | Unit economics |
| **Cost per tenant/user** | Kaun mehnga hai |
| **Cache hit rate** ⭐ | Silent cost regression pakadta hai |
| **Model distribution** | Routing kaam kar raha hai? |
| **Tokens: input/output/cached** | Kahan ja rahe hain |
| **Cost trend (daily)** | Anomaly detection |

### Alerts

```python
ALERTS = {
    "cost_per_request > 2x_baseline":  "critical",   # kuch badla
    "cache_hit_rate < 0.4":            "critical",   # cache toota
    "daily_cost > 1.5x_avg":           "warning",
    "tenant_spend > 0.8 * budget":     "warning",
    "output_tokens_p95 > 2x_baseline": "warning",    # verbosity creep
}
```

---

## Part 5: Ek real optimization case study

**Starting point:**
```
5,000 requests/day
Model: top-tier for everything
No caching, no routing
top_k = 20
Avg: 8,000 input + 600 output tokens
Cost: ~$0.024/request = $3,600/month
P95 latency: 9 seconds
```

**Optimization sequence:**

| Step | Change | Cost after | Latency after |
|---|---|---|---|
| 0 | Baseline | $3,600/mo | 9.0s |
| 1 | **Streaming on** | $3,600 | 9.0s (perceived 1.5s) ✅ |
| 2 | **Output cap** (600→250) | $2,900 | 5.5s |
| 3 | **Prompt caching** (system+tools) | $1,700 | 4.8s |
| 4 | **top_k 20→5** + reranking | $1,300 | 4.6s |
| 5 | **Model routing** (70% cheap) | $520 | 3.2s |
| 6 | **Semantic cache** (20% hit) | $420 | 2.6s avg |

**Result: 88% cost reduction, 71% latency reduction, quality same (eval verified).**

> 🔑 **Note:** Har step ke baad **eval chalayi gayi**. Bina uske ye "optimization" nahi, "degradation" ho sakti thi. Step 2 aur 4 specifically quality risk carry karte hain.

---

## Part 6: Anti-patterns 🚩

| Anti-pattern | Kyun bura |
|---|---|
| Bina measure kiye optimize karna | Galat cheez fix karoge |
| Bina eval ke optimize karna | Quality silently gir jaayegi |
| `max_tokens=8192` "safety" ke liye | Model bharne ki koshish karega |
| Har task pe sabse bada model | 5-10x waste |
| Cache hit rate monitor na karna | Silent cost explosion |
| Har turn pe poori history bhejna | Quadratic cost growth |
| Har request pe naya client banana | Connection overhead |
| Retry bina backoff | Rate limit death spiral |
| Infra optimize karna jab prompt fix chahiye | Galat layer |

---

## Khud se pooch 🤔

1. Output tokens input se kyun mehnge hain? (Architecture se answer)
2. Cost optimization ke top 3 levers? Wo milkar kitna bacha sakte hain?
3. 5 cache killers batao. Wo "silent" kyun hote hain?
4. Semantic cache mein threshold high kyun? Ek concrete failure example.
5. Latency ka sabse bada component kaunsa hai? Uspe 2 fixes?
6. Case study mein har step ke baad eval kyun chalayi gayi?
7. Tumhara monthly bill achanak 3x ho gaya. Systematic debug?

<details>
<summary>Answer 7</summary>

**Kadam 1: Kya badla? (5 minutes)**
- Deploy history dekho — kaunsa change bill spike se pehle gaya?
- Traffic dekho — requests badhe? (Agar 3x traffic hai to bill 3x normal hai!)
- Model distribution dekho — routing toot gaya? Sab requests bade model pe jaa rahi hain?

**Kadam 2: Cache hit rate check karo** ⭐ (sabse likely culprit)
```sql
SELECT date, AVG(cached_input_tokens::float / NULLIF(input_tokens,0)) AS hit_rate
FROM usage GROUP BY date ORDER BY date DESC LIMIT 14;
```
Agar hit rate 0.85 se 0.05 pe gir gaya — **mil gaya.** Koi ne prompt prefix mein kuch dynamic daal diya:
- `datetime.now()` system prompt mein
- Per-user content prefix mein
- Tools ka order non-deterministic ho gaya
- Ek conditional system section add hua

**Fix:** Do requests ka rendered prompt byte-by-byte diff karo. Invalidator mil jaayega.

**Kadam 3: Token breakdown**
```sql
SELECT date,
       AVG(input_tokens) AS avg_in,
       AVG(output_tokens) AS avg_out,
       AVG(cost_usd) AS avg_cost
FROM usage GROUP BY date ORDER BY date DESC LIMIT 14;
```
- **Input badha?** → RAG `top_k` badha? History cap toot gaya? Tool definitions add hue?
- **Output badha?** → Prompt change ne verbosity badha di? `max_tokens` badha? Reasoning/effort on ho gaya?

**Kadam 4: Model distribution**
```sql
SELECT model, COUNT(*), SUM(cost_usd) FROM usage
WHERE date > now() - interval '7 days' GROUP BY model;
```
Routing classifier fail ho raha hai? Sab "hard" classify ho raha hai?

**Kadam 5: Outliers**
```sql
SELECT trace_id, cost_usd, input_tokens, output_tokens FROM usage
ORDER BY cost_usd DESC LIMIT 20;
```
Kuch requests $5 kharch kar rahi hain? Agent loop? Ek user abuse kar raha hai? Ek badly-formed input jo loop bana raha hai?

**Sabse common causes (frequency order):**
1. **Cache invalidator** (ek chhota prompt edit) — 40%
2. **Model routing toota** ya effort default badla — 25%
3. **Context growth** (history cap hata, `top_k` badha, tools add hue) — 20%
4. **Agent loop** bina cost cap ke — 10%
5. **Traffic genuinely badha** — 5%

**Prevention (aage ke liye):**
- `cache_hit_rate < 0.4` pe **alert** ⭐
- `cost_per_request > 2x baseline` pe alert
- CI mein cost regression test (eval ke saath cost bhi assert karo)
- Per-tenant budget caps
</details>

---

**Next:** [`03-eval-driven-development.md`](03-eval-driven-development.md)
