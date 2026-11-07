# 08 — Stack kaise choose karein (Decision Guide)

> **Day 5 ka finale:** Sab kuch jodkar ek practical decision process.

---

## Part 1: Ek universal decision process

```
STEP 1: Problem define karo (technology se pehle!)
   - Users kaun hain? Kitne?
   - Kya task hai? (classification / RAG / agent)
   - Success kya dikhta hai? (measurable!)
   - Constraints: latency, cost, privacy, compliance

STEP 2: Simplest paradigm chuno
   Single call → Workflow → Agent
   (Day 4 ka 4-question test)

STEP 3: Layer by layer decide karo
   Model → Context/RAG → Orchestration → Storage → Eval → UI

STEP 4: Har layer pe poocho:
   "Kya main ye 200 lines mein likh sakta hoon?"
   HAAN → likho. NAHI → framework.

STEP 5: EVAL LAYER PEHLE BANAO
   (Ye step 3 ka hissa nahi, alag hai — kyunki log ise skip kar dete hain)

STEP 6: Ship karo, measure karo, iterate karo
```

---

## Part 2: Constraints se stack derive karo

Har constraint ek decision force karti hai. Ye table use karo:

| Constraint | Force karta hai |
|---|---|
| **Data desh se bahar nahi jaa sakta** | Self-hosting ya region-locked cloud. Model choice limited. |
| **Latency < 2 sec** | Chhota model, no reranking, no agentic, streaming mandatory |
| **Cost < ₹1 per query** | Model routing, caching, chhota model, batch jahan possible |
| **Citations mandatory** | RAG (fine-tuning nahi), citation validation, chunk-level metadata |
| **Compliance audit** | Tracing mandatory, ACL retrieval mein, audit log, PII handling |
| **Team = 2 log** | Managed services, kam moving parts, boring stack |
| **Team = 20 log** | Framework standardization worth hai |
| **Traffic spiky** | Serverless / API (self-host nahi) |
| **Offline/air-gapped** | Self-host mandatory |

> **Ye tarika better hai** "kaunsa framework popular hai" se. Constraints se shuru karo, technology usse derive hogi.

---

## Part 3: 5 concrete blueprints

### Blueprint 1: Internal docs Q&A bot

```
Scale: 50 users, 500 queries/day, 5,000 documents
Constraints: internal only, citations chahiye, budget chhota
```

```
Model      : Mid-tier API model (sonnet-class)
RAG        : Custom (~200 lines)
             Parsing: Unstructured
             Chunking: structure-aware + enrichment
             Search: pgvector + Postgres FTS (hybrid, RRF)
             Rerank: bge-reranker (local)
Orchestr.  : Single call (agent nahi!)
Storage    : Postgres (sab kuch)
Eval       : Custom script, 40 cases
Obs        : Langfuse (self-hosted)
UI         : Streamlit ya simple Next.js
Deploy     : Ek container (Railway/Render)
```
**Cost:** ~₹8,000/month | **Build time:** 2 hafte

---

### Blueprint 2: Customer support automation

```
Scale: 5,000 tickets/day
Constraints: latency < 5s, escalation path, compliance logging
```

```
Model      : Routing — Haiku-class (classify) + Sonnet-class (draft)
RAG        : Custom, past tickets + KB
Orchestr.  : WORKFLOW (agent nahi!)
             classify → history (code) → RAG → draft → policy check (code) → route
Storage    : Postgres + pgvector, Redis (cache)
Eval       : 200 cases, category-wise slices
Obs        : Langfuse + custom metrics (escalation rate, CSAT)
Guardrails : PII redaction, policy validator, human escalation
Deploy     : Kubernetes ya ECS, autoscaling
```
**Cost:** ~₹40,000/month | **Build time:** 6 hafte

> **Note:** Ye agent **nahi** hai. Steps known hain. Day 4 ka lesson.

---

### Blueprint 3: Coding agent (internal dev tool)

```
Scale: 50 developers
Constraints: repo access, tests se verify, PR-based output
```

```
Model      : Top-tier reasoning model, effort=xhigh
Orchestr.  : Coding agent SDK, ya custom loop
Tools      : read, write, edit, bash, grep, glob, run_tests
Harness    : max_steps=40, max_cost=$3, loop detection,
             context pruning + compaction, workspace-scoped writes
Sandbox    : Docker container per run
Eval       : 30 real tasks, 3 runs each, success rate + cost
Obs        : Full tracing, per-run cost
Safety     : Read-only prod access, PR output (direct push nahi),
             human review mandatory
Deploy     : Background job queue
```
**Cost:** Variable (~₹150-800 per task) | **Build time:** 4 hafte

---

### Blueprint 4: Multi-source enterprise RAG

```
Scale: 2,000 users, 50,000 documents, 6 sources
Constraints: ACL enforcement, audit, SSO, 99.9% uptime
```

```
Model      : Enterprise API (Bedrock/Vertex/Foundry, ya direct)
Ingestion  : LlamaIndex connectors + custom ACL mapping
             Per-source chunking rules
             Incremental sync (hash-based)
RAG        : Hybrid (vector + BM25) + reranking + ACL pre-filter 🔒
Orchestr.  : Router (RAG / SQL / API) + agentic RAG for complex
Storage    : Managed vector DB ya pgvector cluster
Eval       : 300 cases, per-source slices, ACL leak tests ⭐
Obs        : Enterprise tracing + SIEM integration
Security   : ACL in retrieval, PII redaction, prompt injection defence,
             audit log har query ka
Deploy     : Multi-region, HA
```
**Cost:** ₹3-8 lakh/month | **Build time:** 4-6 mahine

---

### Blueprint 5: Consumer AI product

```
Scale: 100,000 users, spiky traffic
Constraints: cost per user tight, latency critical, quality visible
```

```
Model      : Aggressive routing — 80% chhota model, 20% bada
             Semantic cache (15-30% full skip)
             Prompt caching mandatory
Orchestr.  : Mostly single call, streaming
Storage    : Managed (Pinecone/Supabase) — ops minimize
Eval       : Continuous, user feedback loop (thumbs, regeneration rate)
Obs        : Cost per user tracking, cohort analysis
Deploy     : Serverless (spiky traffic ke liye)
Guardrails : Rate limiting per user, content moderation, abuse detection
```
**Cost:** Highly variable — **unit economics** track karna critical | **Build:** 8 hafte

---

## Part 4: 🚩 Red flags (in stacks se bacho)

| Red flag | Kyun bura |
|---|---|
| **Koi eval layer nahi** | Sabse mehngi galti. Tum blind ho. |
| **3+ frameworks ek saath** | Debugging impossible |
| **Din 1 pe multi-provider abstraction** | YAGNI + feature lag |
| **Agent jahan workflow kaafi tha** | 10-30x cost, non-deterministic |
| **GraphRAG bina basic RAG theek kiye** | Foundations skip |
| **Self-hosting bina genuine reason** | Ops burden |
| **Vector DB jab Postgres kaafi tha** | Extra system |
| **Fine-tuning pehle attempt mein** | Prompting/RAG try karo pehle |
| **ACL prompt mein** | Security hole |
| **Koi tracing nahi** | Debug nahi kar sakte |
| **Prompts code mein hardcoded** | Version nahi, regression trace nahi |
| **Multi-agent bina context justification** | Cost + complexity, zero gain |

---

## Part 5: Cost estimation (planning ke liye)

```python
def estimate_monthly_cost(
    requests_per_day, input_tokens, output_tokens,
    input_price_per_m, output_price_per_m,
    cache_hit_rate=0.0, cached_discount=0.9,
):
    monthly = requests_per_day * 30

    effective_input = input_tokens * (1 - cache_hit_rate * cached_discount)
    input_cost  = monthly * effective_input / 1_000_000 * input_price_per_m
    output_cost = monthly * output_tokens   / 1_000_000 * output_price_per_m

    return {
        "llm": input_cost + output_cost,
        "embeddings": monthly * 0.00002,        # rough
        "reranking": monthly * 0.0001,          # rough
        "infra": 5000,                           # DB, hosting (₹)
    }
```

### Cost levers (impact order)

| # | Lever | Typical saving |
|---|---|---|
| 1 | **Model routing** (chhota model jahan chalega) | 60-80% |
| 2 | **Prompt caching** | 50-90% input cost |
| 3 | **Output length kam** | 30-50% (aur latency!) |
| 4 | **Semantic caching** | 15-30% requests |
| 5 | **`top_k` kam (20→5)** | 20-30% input |
| 6 | **Batch API** (non-realtime) | ~50% |
| 7 | Context pruning (agents) | 30-60% |

> **Pehle 3 levers milkar aksar 80%+ cost cut kar dete hain.** Infra optimize karne se pehle ye karo.

---

## Part 6: Build vs Buy

| Component | Build | Buy | Recommendation |
|---|---|---|---|
| Model | ❌ | ✅ | **Buy** (API) — self-host sirf compliance/volume pe |
| RAG pipeline | ✅ | ⚠️ | **Build** — ye tumhara differentiator hai |
| Vector DB | ❌ | ✅ | **Buy/use** — pgvector ya managed |
| Agent loop | ✅ | ⚠️ | **Build** (~200 lines) unless durable state chahiye |
| Reranker | ❌ | ✅ | **Use** existing model (local ya API) |
| Tracing | ❌ | ✅ | **Buy/self-host** Langfuse — mat banao |
| **Evals** | ✅ | ⚠️ | **Build** apne metrics, tool optional |
| Guardrails | ⚠️ | ⚠️ | Hybrid — apne rules + existing PII/moderation tools |
| UI | ⚠️ | ⚠️ | Depends |

> 🔑 **Build wahi jo tumhara differentiator hai (context pipeline, evals). Buy wo jo commodity hai (model, storage, tracing).**

---

## Part 7: Ek final checklist ✅

Production mein jaane se pehle:

### Core
- [ ] Simplest paradigm use kiya (agent sirf jab zaroori)
- [ ] Har layer ka choice **justify** kar sakte ho
- [ ] Framework count minimum hai

### Quality
- [ ] **Eval set hai** (minimum 30 cases, real queries se)
- [ ] Retrieval aur generation alag measure hote hain
- [ ] Baseline metrics recorded hain
- [ ] CI mein eval chalti hai

### Reliability
- [ ] `stop_reason` check hota hai (truncation, refusal)
- [ ] Retries + timeouts set hain
- [ ] Fallback model configured
- [ ] Agents mein: step/cost/time limits, loop detection

### Cost
- [ ] Prompt caching on, **cache hit rate measured**
- [ ] Model routing implemented
- [ ] Cost per request tracked
- [ ] Budget alerts set

### Security 🔒
- [ ] **ACL retrieval layer mein** (prompt mein nahi)
- [ ] Tool allowlist + schema validation
- [ ] Destructive ops pe human approval
- [ ] Untrusted content delimited
- [ ] PII redaction (logs + memory)
- [ ] Credentials agent ke bahar

### Observability
- [ ] Tracing on (`prompt_version`, `cache_read` included)
- [ ] Alerts configured
- [ ] User feedback loop

---

## Khud se pooch 🤔

1. Universal decision process ke 6 steps kya hain? Step 5 alag kyun rakha gaya?
2. Constraints se stack derive karna "popular framework choose karne" se better kyun hai?
3. Blueprint 2 (customer support) mein agent kyun **nahi** hai?
4. Top 3 cost levers kaunse hain? Wo milkar kitna bacha sakte hain?
5. Build vs Buy ka rule kya hai? Kya build karna chahiye aur kyun?
6. 12 red flags mein se kaunsa sabse mehnga hai?
7. Tumhe ek naya project mila hai. Pehle 3 din mein kya karoge?

<details>
<summary>Answer 7</summary>

**Din 1: Problem samjho, technology ko haath mat lagao.**

Sawaal poochho:
- **Users kaun hain, kitne?** (10 internal vs 100K consumer = bilkul alag stack)
- **Exact task kya hai?** "AI chatbot" ek answer nahi hai. "Users apne past invoices ke baare mein sawaal poochte hain" — ye answer hai.
- **Success kaise measure hoga?** Agar iska answer nahi hai, wo pehle define karo.
- **Constraints:** latency budget? cost per query? data residency? compliance? team size?
- **Data kaisa hai?** Format, size, update frequency, structure. **Sample data khud dekho.**

Aur ek critical sawaal: **"Kya isme genuinely AI chahiye?"** Kabhi-kabhi ek SQL query aur ek form kaafi hota hai.

**Din 2: Eval set banao (haan, code se pehle).**

- 20-30 real examples collect karo (users se, logs se, ya domain expert se)
- Har case ke liye expected output ya must-contain facts likho
- **Ab tumhare paas success ki definition hai** — ye din 1 ke "success kaise measure hoga" ka concrete answer hai

Ye counter-intuitive lagta hai par ye **sabse high-leverage din** hai. Iske bina tum har baad ka decision guess se loge.

**Din 3: Simplest possible version banao (spike).**

```python
# Ek file, ~100 lines, koi framework nahi
def answer(query):
    chunks = simple_search(query)      # basic vector search
    return llm(build_prompt(query, chunks))
```

- Eval set pe chalao → **baseline number** mil gaya
- Ab tumhe pata hai ki problem **kahan** hai (retrieval? generation? dono?)

**Din 4+ (ab informed decisions):**

Baseline aur failure analysis ke basis pe:
- Recall kharab? → hybrid + reranking (Day 3 Level 1)
- Generation kharab? → prompt, ya bada model
- Latency kharab? → streaming, output length, routing
- Cost kharab? → caching, routing

**Ab framework decisions bhi informed hain:** "Mujhe LlamaIndex chahiye kyunki 6 connectors chahiye" — evidence ke saath, hype se nahi.

**Jo NA karo pehle 3 din mein:**
- ❌ Framework choose karna
- ❌ Vector DB compare karna
- ❌ Architecture diagram banana
- ❌ "Best practices" blog padhna

**Ye sab tab karo jab tumhe pata ho ki tumhara actual problem kya hai.** Pehle 3 din problem samajhne ke hain, solution architect karne ke nahi.

**Meta-lesson:** Zyadatar AI projects isliye fail nahi hote ki galat framework chuna — isliye fail hote hain ki **problem theek se define nahi kiya aur success measure nahi kiya.**
</details>

---

## 🎉 Day 5 complete!

Ab tum jaante ho:
- ✅ AI stack ke 8 layers, aur har layer ka tooling
- ✅ LangChain / LangGraph / LlamaIndex — kab kaunsa, kab koi nahi
- ✅ Agent SDKs ke 4 types (harness vs deployment axes)
- ✅ Multi-agent frameworks ka imaandaar assessment
- ✅ Vector DBs — aur ki pgvector default hona chahiye
- ✅ Serving stack — aur ki tumhe shayad self-hosting nahi chahiye
- ✅ **Evals + observability** — sabse important, sabse skipped layer
- ✅ Constraints se stack derive karna

**Next:** [`../05-production/01-architecture-blueprints.md`](../05-production/01-architecture-blueprints.md) — Day 6.
