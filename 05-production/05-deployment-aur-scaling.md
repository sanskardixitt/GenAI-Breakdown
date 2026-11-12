# 05 — Deployment aur Scaling

> **Day 6 ka finale:** Code se production tak. Aur production mein zinda rehna.

---

## Part 1: Deployment checklist

### Pre-launch

```
CODE
  □ Prompts files mein, git mein tracked (inline nahi)
  □ Config environment-driven (hardcoded values nahi)
  □ Secrets secret manager mein (env file commit nahi)
  □ Dependencies pinned (requirements.txt / lockfile)
  □ Health check endpoint

QUALITY
  □ Eval suite (30+ cases) passing
  □ Baseline metrics recorded
  □ ACL leak tests passing (zero tolerance)
  □ Adversarial eval set passing
  □ Load test kiya (expected 3x traffic pe)

RELIABILITY
  □ Retries + timeouts har external call pe
  □ Fallback model configured
  □ Graceful degradation tested (reranker down karke dekho!)
  □ stop_reason handling (max_tokens, refusal)
  □ Agent limits (steps, cost, time)

OBSERVABILITY
  □ Tracing on (prompt_version, cache_read included)
  □ Error tracking (Sentry ya equivalent)
  □ Metrics dashboard
  □ Alerts configured aur TEST kiye hue

SECURITY
  □ Rate limiting per user/tenant
  □ Input validation (length, content)
  □ PII redaction in logs
  □ ACL in retrieval
  □ Audit log

COST
  □ Prompt caching on, hit rate verified
  □ Budget caps per tenant
  □ Cost alerts
  □ Cost per request baseline recorded
```

---

## Part 2: Scaling dimensions

AI apps ka scaling normal apps se alag hai:

| Dimension | Bottleneck | Solution |
|---|---|---|
| **Concurrent requests** | Provider rate limits | Queue, backpressure, multiple keys/regions |
| **Token throughput** | TPM limits | Batching, caching, routing |
| **Vector search** | Index size, RAM | Sharding, quantization, better index params |
| **Indexing** | Embedding API rate | Batch, queue, off-peak |
| **Long-running agents** | Wall time | Job queue, not HTTP |
| **Cost** | 💰 | Routing, caching (Day 6, file 02) |

### 🔑 Sabse common scaling surprise

```
"Humne 10x traffic handle karne ke liye 10x servers add kiye —
 phir bhi 429 rate limit errors aa rahe hain"
```

**Kyun:** Bottleneck tumhare servers nahi, **provider ke rate limits** hain. Servers add karne se kuch nahi hoga.

**Fix:**
- Rate limit headroom pehle se maango (provider se)
- **Queue with backpressure** — requests drop na ho, wait karein
- Caching (requests hi kam ho jaayein)
- Multiple keys/regions (agar allowed ho)

---

## Part 3: Queue-based architecture

```python
# API layer — fast, non-blocking
@app.post("/query")
async def query(req: QueryRequest):
    if await queue.depth() > MAX_QUEUE_DEPTH:
        raise HTTPException(503, "Overloaded, thodi der baad try karein")

    job_id = await queue.enqueue(process_query, req, priority=req.priority)
    return {"job_id": job_id}

# Worker layer — controlled concurrency
class Worker:
    def __init__(self, max_concurrent=10):
        self.sem = asyncio.Semaphore(max_concurrent)     # provider limits respect

    async def process(self, job):
        async with self.sem:
            return await pipeline(job)
```

**Fayde:**
- API instantly respond karta hai
- Concurrency tumhare control mein (provider limits ke hisaab se)
- Spikes absorb ho jaate hain
- Retries centralized
- Priority queues possible (paid users pehle)

---

## Part 4: Rate limit handling

```python
class RateLimitedClient:
    def __init__(self, client, rpm=1000, tpm=500_000):
        self.client = client
        self.req_bucket = TokenBucket(rpm, 60)
        self.tok_bucket = TokenBucket(tpm, 60)

    async def create(self, **kwargs):
        estimated = estimate_tokens(kwargs)
        await self.req_bucket.acquire(1)
        await self.tok_bucket.acquire(estimated)

        for attempt in range(5):
            try:
                return await self.client.messages.create(**kwargs)
            except RateLimitError as e:
                # retry-after header respect karo
                wait = int(e.response.headers.get("retry-after", 2 ** attempt))
                log.warning("rate_limited", attempt=attempt, wait=wait)
                await asyncio.sleep(wait + random.uniform(0, 1))   # jitter!
            except OverloadedError:
                await asyncio.sleep(2 ** attempt + random.uniform(0, 1))
        raise MaxRetriesExceeded()
```

**Note:** Official SDKs mein retries built-in hote hain (`max_retries`). Pehle wo configure karo, custom logic tab likho jab genuinely zaroorat ho.

> ⚠️ **Jitter zaroori hai.** Bina jitter ke saare clients ek saath retry karenge — "thundering herd" — aur rate limit phir se hit hoga.

---

## Part 5: Caching layers (production stack)

```
┌──────────────────────────────────────────────────┐
│ L1: Response cache (exact query hash)            │
│     TTL: 1 hour | Hit: 5-15% | Saving: 100%      │
│     Key: hash(query + filters + TENANT_ID) 🔒    │
├──────────────────────────────────────────────────┤
│ L2: Semantic cache (embedding similarity)        │
│     TTL: 1 hour | Hit: 15-30% | Saving: 100%     │
│     Threshold: 0.97 | Tenant-scoped 🔒           │
├──────────────────────────────────────────────────┤
│ L3: Embedding cache (text hash → vector)         │
│     TTL: forever | Hit: high on re-index         │
├──────────────────────────────────────────────────┤
│ L4: Retrieval cache (query → chunk IDs)          │
│     TTL: 10 min | Careful: index update pe stale │
├──────────────────────────────────────────────────┤
│ L5: Prompt cache (provider-side)                 │
│     TTL: 5min-1hr | Hit: 60-90% | Saving: 90%    │
└──────────────────────────────────────────────────┘
```

### Cache invalidation

```python
async def on_document_updated(doc_id):
    # Retrieval caches stale ho gaye
    await cache.delete_pattern(f"retrieval:*")

    # Response caches jinme ye doc tha
    affected = await db.get_cached_responses_using_doc(doc_id)
    await cache.delete_many(affected)

    # Embedding cache theek hai (text hash based)
```

> 🚩 **Semantic cache ke saath document updates khatarnak hain.** Purana answer serve hota rahega. Ya TTL chhota rakho, ya document update pe aggressive invalidation.

---

## Part 6: Rollout strategy

### Progressive rollout

```python
def should_use_new_version(user_id: str, percentage: int) -> bool:
    return (int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 100) < percentage

async def query(req, user):
    if should_use_new_version(user.id, ROLLOUT_PERCENTAGE):
        result = await pipeline_v2(req)
        tracer.set("version", "v2")
    else:
        result = await pipeline_v1(req)
        tracer.set("version", "v1")
    return result
```

**Rollout schedule:**
```
Day 1:  5%   → metrics watch karo (quality, cost, latency, errors)
Day 3:  25%  → agar sab theek
Day 5:  50%
Day 7:  100%
```

**Rollback trigger:**
```python
ROLLBACK_IF = {
    "error_rate > 2x baseline",
    "p95_latency > 1.5x baseline",
    "cost_per_request > 1.5x baseline",
    "thumbs_down_rate > 1.3x baseline",
    "cache_hit_rate < 0.5x baseline",
}
```

### Shadow mode (best for risky changes)

```python
async def query(req, user):
    result = await pipeline_v1(req)                # ye user ko jaata hai

    # V2 background mein chalao, compare karo, user ko na dikhao
    asyncio.create_task(shadow_compare(req, result))

    return result

async def shadow_compare(req, v1_result):
    try:
        v2_result = await pipeline_v2(req)
        await log_comparison(req, v1_result, v2_result)
    except Exception as e:
        log.warning("shadow_failed", error=str(e))    # user affect na ho
```

**Fayda:** Real traffic pe real comparison, **zero user risk**. Mehnga hai (2x cost) par risky changes ke liye worth hai.

---

## Part 7: Incident response

### Common incidents aur first response

| Incident | Symptom | First action |
|---|---|---|
| **Provider outage** | 5xx spike | Fallback model pe switch |
| **Rate limited** | 429 spike | Queue depth badhao, non-critical traffic throttle |
| **Cost spike** | Bill alert | Cache hit rate check ⭐, recent deploys check |
| **Quality regression** | Thumbs down spike | Last deploy rollback, eval chalao |
| **Latency spike** | P95 alert | Stage breakdown dekho, degraded mode on |
| **Vector DB down** | Retrieval errors | BM25-only fallback |
| **Injection detected** | Security alert | Affected content quarantine, user pattern check |
| **ACL leak** | 🚨 Critical | Feature disable, audit log review, incident report |

### Runbook template

```markdown
## Incident: Cost spike

### Detect
- Alert: cost_per_request > 2x baseline

### Diagnose (in order)
1. cache_hit_rate check karo — 40% se kam? → cache invalidator
   → Recent deploys mein prompt/system changes dekho
2. Model distribution — routing toota?
3. Token breakdown — input ya output badha?
4. Top 20 expensive requests — outliers?
5. Traffic — genuinely badha?

### Mitigate
- Cache invalidator → revert us change ko
- Routing broken → routing thresholds fix karo
- Agent loop → cost caps tighten karo
- Traffic → expected hai to budget adjust karo

### Prevent
- CI mein cost regression test
- Cache hit rate alert threshold review
```

---

## Part 8: Ongoing operations

### Weekly
- [ ] Eval metrics review (trend)
- [ ] Cost per request trend
- [ ] Top failure cases → eval set mein add
- [ ] User feedback (thumbs, regeneration) review

### Monthly
- [ ] Eval set expand karo (naye cases)
- [ ] Holdout set pe run (over-fitting check)
- [ ] Model version review (naya model aaya? test karo)
- [ ] Cost optimization review
- [ ] Security: adversarial set update

### Quarterly
- [ ] Architecture review (kya simplify ho sakta hai?)
- [ ] Dependency updates
- [ ] Red team exercise
- [ ] Retention/compliance audit

---

## Part 9: Model version upgrades

Naya model aaya. Kya karo?

```
1. EVAL PE TEST KARO (production pe nahi!)
   → Same eval suite, naya model
   → Quality, cost, latency compare karo

2. BREAKING CHANGES CHECK KARO
   → Naye models mein parameters remove/change hote hain
     (temperature hata, budget_tokens hata, thinking default badla,
      prefill 400 dene laga)
   → Provider ka migration guide padho

3. PROMPTS RE-TUNE KARO ⭐
   → Purane model ke liye likhe prompts naye pe often KHARAB karte hain
   → Scaffolding hatao ("think step by step", "CRITICAL:", verbose rules)
   → Measure karo

4. SHADOW MODE
   → Real traffic pe compare karo

5. PROGRESSIVE ROLLOUT
   → 5% → 25% → 50% → 100%

6. BASELINE UPDATE
```

> 🚩 **Sabse common upgrade mistake:** Sirf model string badalna aur ship kar dena. Naye models ka **prompting behaviour alag** hota hai — purane prompts often under-perform karte hain. Ye eval ke bina invisible rehta hai.

---

## Khud se pooch 🤔

1. "10x servers add kiye phir bhi 429" — kya ho raha hai? Fix?
2. Queue-based architecture ke 5 fayde kya hain?
3. Retry mein jitter kyun zaroori hai?
4. Semantic cache aur document updates ka conflict kya hai?
5. Shadow mode kya hai? Progressive rollout se better kab hai?
6. Cost spike incident mein pehla diagnostic step kya hai aur kyun?
7. Naya model aaya — 6 steps kya hain? Sabse common mistake kya hai?
8. Tumhara AI feature launch hone wala hai kal. Aakhri 24 ghante mein kya check karoge?

<details>
<summary>Answer 8</summary>

**Priority order mein (agar time kam pade to upar wale zaroori):**

**🔴 Blockers (in ke bina launch mat karo):**

1. **Eval suite pass ho rahi hai?** Baseline recorded hai? Agar eval hi nahi hai — **launch postpone karo** aur aaj 20 cases banao. Ye 3 ghante ka kaam hai aur mahine bachaayega.

2. **ACL / security tests pass?** Multi-tenant hai to cross-tenant leak test **zaroor**. Ek leak = compliance incident. Zero tolerance.

3. **Cost caps lage hain?** Per-tenant budget, agent cost limits, rate limits. Bina iske ek bad actor (ya ek bug) raat bhar mein bill phaad sakta hai.

4. **Kill switch hai?** Feature flag jisse tum 30 seconds mein feature band kar sako. Ye non-negotiable hai.

**🟡 Important:**

5. **Graceful degradation TEST karo** (padho nahi, actually test karo):
   - Reranker band karke request bhejo → chalna chahiye
   - Vector DB band karke → BM25 fallback ya honest error
   - Primary model unavailable simulate karo → fallback

6. **Alerts test karo.** Alert configure karna aur alert **kaam karna** do alag cheezein hain. Ek fake spike trigger karke dekho ki notification actually aati hai.

7. **Load test** — expected traffic ka 3x. Rate limits kahan hit honge?

8. **Tracing verify karo** — ek request bhejo, trace mein `prompt_version`, `cache_read_input_tokens`, `stop_reason` sab dikh rahe hain?

9. **Prompt caching working?** Do identical requests bhejo, doosri mein `cache_read_input_tokens > 0` hona chahiye. Agar 0 hai, tum din 1 se full price pay karoge.

**🟢 Good to have:**

10. **Rollout plan** — 5% se shuru karo, 100% se nahi
11. **Runbook** — team ko pata ho ki cost spike / quality drop pe kya karna hai
12. **On-call** — kaun available hai launch ke pehle 48 ghante?

**Aur ek soft check:**

13. **Khud 20 real queries chalao.** Metrics sab green ho sakti hain aur product phir bhi bura lag sakta hai. 20 minutes apne product ko **use** karo, jaise ek user karta.

**Agar in me se koi bhi #1-4 miss hai — launch date push karo.** Ek mahina late launch karna, ek security incident ya ₹5 lakh ke bill se sasta hai.
</details>

---

## 🎉 Day 6 complete!

Ab tum jaante ho:
- ✅ Production architecture patterns (degradation, streaming, multi-tenancy)
- ✅ Cost aur latency optimization — priority order mein
- ✅ **Eval-driven development** — ek workflow, tool nahi
- ✅ Security: prompt injection, ACL, credentials, PII
- ✅ Deployment, scaling, rollout, incident response

**Next:** [`../06-hands-on-labs/README.md`](../06-hands-on-labs/README.md) — Day 7, code likhne ka waqt.
