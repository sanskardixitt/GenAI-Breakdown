# 01 — Architecture Blueprints

> **Day 6 ka goal:** Demo se product tak ka gap. Ye file architecture patterns deti hai jo actually production mein chalte hain.
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/08-production-stack`](../08-diagrams/08-production-stack.excalidraw) — request ka poora safar.

---

## Part 1: Demo aur Production mein gap

| | Demo | Production |
|---|---|---|
| Error handling | try/except | Retries, fallbacks, circuit breakers, DLQ |
| Latency | "chal jaata hai" | P95 SLA, streaming, caching |
| Cost | "kitna hoga?" | Per-request tracking, budgets, alerts |
| Quality | "acha lag raha hai" | Eval suite, thresholds, CI |
| Security | Kuch nahi | ACL, injection defence, PII, audit |
| Observability | print() | Distributed tracing |
| Scale | 1 user | Concurrency, rate limits, queues |
| Failure | Crash | Graceful degradation |

> **90% AI projects demo pe ruk jaate hain** kyunki ye gap 5x kaam hai jitna log estimate karte hain.

---

## Part 2: Blueprint — RAG service (production shape)

```
                        CLIENT
                          │
                    ┌─────▼──────┐
                    │   API GW    │  auth, rate limit, request ID
                    └─────┬──────┘
                          │
                 ┌────────▼─────────┐
                 │  App (FastAPI)   │
                 └────────┬─────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌────────────┐  ┌────────────┐
    │ Semantic │   │ RAG        │  │ Trace      │
    │ Cache    │   │ Pipeline   │  │ Emitter    │
    │ (Redis)  │   └─────┬──────┘  └────────────┘
    └──────────┘         │
                ┌────────┼─────────┐
                ▼        ▼         ▼
          ┌─────────┐ ┌──────┐ ┌────────┐
          │Postgres │ │Redis │ │ LLM    │
          │+pgvector│ │cache │ │ API    │
          └─────────┘ └──────┘ └────────┘
                          │
              ┌───────────▼────────────┐
              │  Async: indexing jobs   │
              │  (Celery / RQ)          │
              └────────────────────────┘
```

### Request flow (code shape)

```python
@app.post("/query")
async def query(req: QueryRequest, user: User = Depends(auth)):
    trace_id = str(uuid4())
    with tracer.trace(trace_id, user_id=hash(user.id)):

        # 1. Rate limit
        await rate_limiter.check(user.id)

        # 2. Input validation
        if len(req.query) > MAX_QUERY_LEN:
            raise HTTPException(400, "Query too long")

        # 3. Semantic cache
        if cached := await semantic_cache.get(req.query, user.access_scope):
            return cached                          # ⭐ 15-30% requests

        # 4. Query rewrite (multi-turn ke liye)
        q = await rewrite(req.query, req.history) if req.history else req.query

        # 5. Route
        intent = await classify_intent(q)
        if intent == "aggregation":
            return await sql_pipeline(q, user)
        if intent == "realtime":
            return await api_pipeline(q, user)

        # 6. Retrieve (ACL filter!)
        chunks = await hybrid_search(q, k=50,
                                     access_levels=user.access_levels)  # 🔒
        chunks = await rerank(q, chunks, k=5)

        # 7. Quality gate
        if not chunks or chunks[0].score < MIN_SCORE:
            return NoAnswerResponse(reason="no_relevant_context")

        # 8. Generate (streaming)
        return StreamingResponse(generate_stream(q, chunks, trace_id))
```

---

## Part 3: Async / background jobs

**Kya sync hona chahiye vs async:**

| Sync (user wait karta hai) | Async (background) |
|---|---|
| Query answering | Document indexing |
| Retrieval | Memory extraction |
| Short generation | Long agent runs |
| | Eval runs |
| | Report generation |
| | Re-embedding |

### Long agent runs ka pattern

```python
@app.post("/agent/run")
async def start_run(req: AgentRequest, user: User = Depends(auth)):
    run_id = str(uuid4())
    await db.create_run(run_id, req.task, user.id, status="queued")
    await queue.enqueue(execute_agent, run_id, req.task)     # background
    return {"run_id": run_id, "status": "queued"}

@app.get("/agent/run/{run_id}")
async def get_run(run_id: str, user: User = Depends(auth)):
    run = await db.get_run(run_id)
    authorize(user, run)
    return {"status": run.status, "steps": run.steps,
            "output": run.output, "cost": run.cost}

@app.get("/agent/run/{run_id}/stream")
async def stream_run(run_id: str, user: User = Depends(auth)):
    return EventSourceResponse(subscribe_to_run(run_id))     # live updates
```

> 🔑 **Agents ko HTTP request ke andar mat chalao.** 5-minute agent run ek HTTP timeout hai. Job queue + polling/SSE pattern use karo.

---

## Part 4: Graceful degradation

Har dependency fail ho sakti hai. **Har failure ka ek fallback hona chahiye:**

```python
async def resilient_rag(query, user):
    # Reranker down? Bina rerank ke chalo (thodi quality kam, par service up)
    try:
        chunks = await rerank(query, candidates, k=5)
    except RerankerError:
        log.warning("reranker_down_fallback")
        chunks = candidates[:5]                    # vector order use karo

    # Primary model down? Fallback model
    try:
        return await generate(query, chunks, model=PRIMARY)
    except (OverloadedError, RateLimitError):
        log.warning("primary_model_fallback")
        return await generate(query, chunks, model=FALLBACK)

    # Sab down? Honest error
    except Exception as e:
        log.error("generation_failed", error=str(e))
        return ErrorResponse("Abhi service available nahi hai. Thodi der baad try karein.")
```

### Degradation ladder

```
Full service      : hybrid + rerank + best model
  ↓ reranker down
Degraded L1       : hybrid + best model
  ↓ vector DB slow
Degraded L2       : BM25 only + best model
  ↓ primary model down
Degraded L3       : BM25 + fallback model
  ↓ sab down
Honest failure    : "service unavailable" (fake answer NAHI)
```

> **Kabhi bhi failure ko chhupa ke fake answer mat do.** "Mujhe nahi pata" ek galat answer se better hai — especially enterprise mein.

---

## Part 5: Streaming architecture

```python
async def generate_stream(query, chunks, trace_id):
    full_text = []
    try:
        async with client.messages.stream(
            model=MODEL, max_tokens=2048,
            system=RAG_SYSTEM,
            messages=[{"role": "user", "content": build_prompt(query, chunks)}],
        ) as stream:
            # Metadata pehle bhejo (citations UI mein turant dikhein)
            yield sse({"type": "sources",
                       "sources": [c.metadata for c in chunks]})

            async for text in stream.text_stream:
                full_text.append(text)
                yield sse({"type": "delta", "text": text})

            final = await stream.get_final_message()

        answer = "".join(full_text)

        # Post-processing (stream ke BAAD)
        valid, err = validate_citations(answer, chunks)
        yield sse({"type": "done", "valid": valid,
                   "usage": final.usage.model_dump()})

        # Async logging (user ko wait na karao)
        asyncio.create_task(log_interaction(trace_id, query, answer, final))

    except Exception as e:
        log.error("stream_failed", trace_id=trace_id, error=str(e))
        yield sse({"type": "error", "message": "Generation failed"})
```

**Streaming ke practical points:**
- **Sources pehle bhejo** — UI turant citations dikha sake
- **Validation stream ke baad** — partial output validate nahi kar sakte
- **Logging async** — user ko block na karo
- **Error mid-stream handle karo** — partial output already dikh gaya hai

---

## Part 6: Multi-tenancy 🔒

```python
# Har layer pe tenant isolation
class TenantContext:
    tenant_id: str
    access_levels: list[str]
    rate_limit: int
    monthly_budget: float

async def query(req, ctx: TenantContext):
    # 1. Rate limit — per tenant
    await rate_limiter.check(ctx.tenant_id, ctx.rate_limit)

    # 2. Budget — per tenant
    if await get_month_spend(ctx.tenant_id) > ctx.monthly_budget:
        raise BudgetExceeded()

    # 3. Retrieval — TENANT FILTER MANDATORY
    chunks = await search(req.query, filter={
        "tenant_id": ctx.tenant_id,                  # 🔒 non-negotiable
        "access_level": {"$in": ctx.access_levels},
    })

    # 4. Cache key mein tenant
    cache_key = f"{ctx.tenant_id}:{hash(req.query)}"  # 🔒 cross-tenant leak roko

    # 5. Trace mein tenant
    tracer.set("tenant_id", ctx.tenant_id)
```

> 🚨 **Sabse khatarnak multi-tenant bug: cache mein tenant_id na daalna.** Tenant A ka answer Tenant B ko mil jaayega. Ye ek data breach hai. Eval mein iska test **zaroor** rakho.

---

## Part 7: Deployment topologies

### Topology 1: Monolith (chhoti team)
```
Ek container: FastAPI + RAG + worker threads
Postgres (managed)
Redis (managed)
```
**Kab:** < 5,000 req/day, team ≤ 5. **Simplest, aur mostly kaafi hai.**

### Topology 2: API + Workers
```
API containers (autoscaled)      ← sync queries
Worker containers                 ← indexing, agents, evals
Postgres + Redis + queue
```
**Kab:** Background work hai (indexing, agents). Ye sabse common production shape hai.

### Topology 3: Microservices
```
Gateway → Query service → Retrieval service → Generation service
                       → Indexing service
```
**Kab:** Badi team, alag scaling needs. **Warna over-engineering.**

> **Recommendation:** Topology 2 se shuru karo. Monolith se grow karna aasan hai; microservices se wapas aana mushkil.

---

## Part 8: Configuration management

```python
# ❌ Bura — magic numbers bikhre hue
chunks = search(query, k=5)
if score < 0.3: ...
response = llm(model="claude-opus-5", max_tokens=2048)

# ✅ Acha — ek jagah, versioned, environment-aware
class Config(BaseSettings):
    # Retrieval
    retrieve_k: int = 50
    rerank_k: int = 5
    min_relevance_score: float = 0.30

    # Generation
    model: str = "claude-opus-5"
    fallback_model: str = "claude-sonnet-5"
    max_tokens: int = 2048
    effort: str = "high"

    # Limits
    max_query_length: int = 2000
    request_timeout_s: int = 60

    # Agent
    agent_max_steps: int = 30
    agent_max_cost_usd: float = 2.0

    # Prompts (versioned files)
    prompt_version: str = "v3"

    class Config:
        env_prefix = "RAG_"

cfg = Config()
```

**Fayda:** Har parameter tune ho sakta hai bina deploy ke (env vars), aur trace mein log ho sakta hai.

---

## Khud se pooch 🤔

1. Demo aur production mein 8 gaps kya hain?
2. Long agent runs ko HTTP request mein kyun nahi chalana chahiye? Sahi pattern kya hai?
3. Degradation ladder kya hai? Aakhri step "honest failure" kyun hona chahiye?
4. Streaming mein sources pehle kyun bhejte hain? Validation baad mein kyun?
5. Multi-tenant mein sabse khatarnak bug kya hai? Usse kaise rokoge?
6. Kaunsi deployment topology se shuru karna chahiye aur kyun?
7. Tumhari RAG service P95 latency 12 seconds pe hai aur users complain kar rahe hain. Ek systematic plan banao.

<details>
<summary>Answer 7</summary>

**Kadam 1: Measure karo, guess mat karo.**
Tracing se per-stage latency breakdown nikalo. Typical culprits:
```
Query rewrite    :  300ms
Embedding        :  100ms
Vector search    :  200ms
Reranking        :  400ms   ← suspicious agar top-100 rerank kar rahe ho
LLM prefill      :  800ms   ← cache miss?
LLM decode       : 9,000ms  ← 🔥 usually yahi
Post-processing  :  200ms
```

**Kadam 2: Sabse bade lever pe kaam karo (usually decode).**

Decode time = output tokens × per-token time. To:
- **Output length cap karo** — prompt mein "2-3 sentences" + `max_tokens` reasonable. 400 → 150 tokens = 9s → 3.5s.
- **Streaming on karo** (agar nahi hai) — perceived latency 12s → ~1.5s. Ye sabse bada UX win hai, 1 line ka change.
- **Effort/model check karo** — kya reasoning model use ho raha hai jahan zaroorat nahi? Routing add karo.

**Kadam 3: Prefill optimize karo.**
- **Prompt caching** — `cache_read_input_tokens` check karo. Agar 0 hai, koi invalidator hai (timestamp? per-user system prompt?). Fix karne se 800ms → 150ms.
- **`top_k` kam karo** — 20 chunks se 5 chunks. Input tokens 4x kam, aur accuracy aksar *badhti* hai.

**Kadam 4: Parallel karo.**
```python
# Serial: 300 + 100 = 400ms
q = await rewrite(query)
emb = await embed(q)

# Parallel jahan possible ho (embedding original query pe bhi kar sakte ho)
emb_task = asyncio.create_task(embed(query))
q = await rewrite(query)
```

**Kadam 5: Reranking optimize karo.**
- Top-100 nahi, **top-50** rerank karo
- Chhota reranker model
- Ya reranking ko optional banao low-latency tier ke liye

**Kadam 6: Semantic cache.**
Common queries pe full skip. 15-30% requests 12s se 100ms ho jaayengi.

**Expected result:**
```
Before: P95 = 12s, perceived 12s
After:  P95 = 4s, perceived ~1.5s (streaming)
        + 20% requests cached (100ms)
```

**Aur ek important baat:** Latency ke saath **quality bhi measure karo**. Output length kam karne se latency to girti hai, par kya answers ab incomplete hain? Eval set pe check karo — warna tum latency fix karke quality tod doge, aur pata bhi nahi chalega.
</details>

---

**Next:** [`02-cost-aur-latency.md`](02-cost-aur-latency.md)
