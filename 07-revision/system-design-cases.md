# AI System Design Cases

> **Kaise use karo:** Har case ko **30 minutes** do. Pehle khud design karo (kagaz pe / doc mein), phir solution padho.
>
> Interview mein bhi yahi structure use karo.

---

## 🎯 Universal answering framework

Har AI system design question ko is order mein attack karo:

```
1. CLARIFY (5 min)
   - Users kaun, kitne? Query volume?
   - Success kaise measure hoga?
   - Constraints: latency, cost, privacy, compliance
   - Data: kaisa, kitna, kitni baar update?

2. PARADIGM CHOOSE KARO (2 min)
   - Single call / Workflow / Agent — aur KYUN
   - (Ye step log skip karte hain aur seedha agent bana dete hain)

3. HIGH-LEVEL ARCHITECTURE (10 min)
   - Data flow diagram
   - Har component ka justification

4. DEEP DIVE (8 min)
   - Retrieval strategy / orchestration detail
   - Failure modes aur handling

5. QUALITY (3 min) ⭐ — yahi tumhe alag karta hai
   - Eval strategy — kya measure karoge
   - Baseline aur success threshold

6. PRODUCTION (2 min)
   - Cost estimate, latency budget
   - Security (ACL, injection)
   - Monitoring aur alerts
```

> 🔑 **Step 5 (evals) mention karna sabse strong signal hai.** Zyadatar candidates architecture bata dete hain par measurement nahi. Wahi difference hai.

---

## Case 1: Enterprise Document Assistant

> **Prompt:** "Ek company ke 50,000 internal documents (Confluence, Google Drive, PDFs) pe employees sawaal pooch sakein. 2,000 employees, alag-alag access levels."

### Clarify (ye poocho)
- Query volume? (~2,000/day maan lo)
- Latency expectation? (< 5s acceptable)
- Access levels kitne? (department + confidentiality tiers)
- Documents kitni baar update hote hain? (daily)
- Citations mandatory? (**haan** — enterprise mein hamesha)
- Compliance? (audit log chahiye)

### Paradigm
**Classic RAG (single call)** 90% queries ke liye, **agentic RAG** 10% complex ke liye.
**Agent kyun nahi default?** Steps predictable hain (retrieve → generate). 10x cost, zero gain.

### Architecture

```
INGESTION (nightly + webhook)
  Confluence/Drive/PDF
    → Layout-aware parsing (Unstructured/Docling)
    → Structure-aware chunking (headers) + enrichment (header path)
    → ACL metadata mapping ⭐ (source permissions → chunk access_level)
    → Embed (batch) → pgvector + Postgres FTS
    → Incremental: doc hash tracking, cascade delete

QUERY
  Query rewrite (multi-turn) → Semantic cache (tenant-scoped)
    → Router (lookup / aggregation / summary / complex)
    → Hybrid retrieve (vector + BM25, RRF) + ACL PRE-FILTER 🔒
    → Rerank (cross-encoder, top-50 → top-5)
    → Quality gate (score < threshold → "nahi mila")
    → Generate with citations
    → Citation validation
    → Trace + audit log
```

### Key decisions (aur unke reasons)

| Decision | Reason |
|---|---|
| **pgvector** | 50K docs ≈ 500K chunks — comfortable. Already Postgres. SQL ACL filtering. |
| **Hybrid + rerank** | Internal docs mein bahut identifiers (project codes, ticket IDs, system names) jo vector miss karta hai |
| **ACL pre-filter** | Post-filter mein top-5 khaali ho sakta hai. Aur pre-filter = data model tak pahuncha hi nahi |
| **Header-based chunking** | Confluence/docs structured hain — free semantic boundaries, aur metadata |
| **Router** | "Kitne open tickets hain?" jaisi queries RAG se confidently galat aayengi |

### 🔒 Security (ye section zaroor bolo)

```python
chunks = search(query, k=50, filter={
    "access_level": {"$in": user.access_levels},
    "department": {"$in": user.departments + ["public"]},
})
```
- ACL **retrieval layer** mein, prompt mein nahi
- Source system se permissions sync (Confluence space perms → chunk metadata)
- Permission change = re-index affected chunks
- **ACL eval tests, zero tolerance**
- Audit log: kaun, kya poocha, kya chunks mile

### Evals
- 200 cases, **department-wise slices** (aggregate problems chhupa deta hai)
- Retrieval: Recall@5 > 0.85, MRR > 0.75
- Generation: RAG triad (context relevance, groundedness, answer relevance)
- **ACL leak tests** — har release pe, zero tolerance
- Regression set (fixed bugs)

### Production
- **Cost:** ~2,000 queries/day × ~$0.012 ≈ $720/mo + infra ≈ **$1,200/mo**
- **Latency:** P95 target 5s (streaming se perceived ~1.5s)
- **Monitoring:** no_context_rate, cache_hit_rate, citation_rate, per-dept accuracy

### 🚩 Common mistakes
- ACL prompt mein (security hole)
- Sirf vector search (identifiers miss)
- Agent banana (unnecessary)
- Aggregation queries RAG pe bhejna
- Eval strategy mention na karna

---

## Case 2: Customer Support Automation

> **Prompt:** "5,000 support tickets/day. AI se draft responses banao, categorize karo, escalate karo jahan zaroori ho."

### Clarify
- Kitne % auto-resolve karna hai? (target 40%?)
- Escalation criteria?
- Human review draft pe? (**haan, shuru mein zaroor**)
- SLA? (< 5s per ticket)
- Languages? (Hindi + English → multilingual embeddings)

### Paradigm ⭐
**WORKFLOW, agent nahi.** Steps known hain:
```
classify → history fetch (code) → RAG (similar tickets + KB)
→ draft → policy check (code) → route (code rules) → send/escalate
```
**Agent kyun nahi:** 5,000/day pe agent = 10-30x cost, non-deterministic routing, aur priority logic auditable nahi rahega.

### Architecture

```
Ticket arrives
  ↓
[1] Classify (fast model, structured output)
    → category, sentiment, urgency, language
  ↓
[2] Fetch history (CODE — CRM API)
  ↓
[3] RAG: similar resolved tickets + KB articles (hybrid + rerank)
  ↓
[4] Draft response (main model, grounded prompt with citations)
  ↓
[5] Policy check (CODE — regex + rules)
    → no promises, no PII leak, no unauthorized commitments
  ↓
[6] Route (CODE — business rules)
    → auto-send | human review | escalate
  ↓
[7] Log + feedback loop
```

### Key decisions

| Decision | Reason |
|---|---|
| **Business rules CODE mein** | Priority/escalation auditable hona chahiye. LLM ka mood nahi. |
| **Fast model for classify** | 5,000/day × classification — bada model 50x waste |
| **RAG over resolved tickets** | Past solutions best training data hain |
| **Human review gate** | Shuru mein 100%, phir confidence-based reduce karo |
| **Batch API where possible** | Non-urgent tickets → ~50% sasta |

### 🔒 Security (ye case mein critical)
Tickets **untrusted content** hain — koi bhi likh sakta hai:
- **Reader/actor separation**: reader ticket padhta hai → structured output → actor sirf validated fields leta hai
- Actor ke paas `issue_refund`, `upgrade_tier` jaise tools **nahi** hain
- Injection detection flag → human review
- PII redaction logs mein

### Evals
- 300 cases, category-wise
- **Draft quality** (LLM judge + human sample)
- **Routing accuracy** (kya sahi tickets escalate hue?)
- **Policy violations** (zero tolerance)
- **Business metrics:** auto-resolve rate, CSAT, handle time

### Production
- **Cost:** 5,000/day × ~$0.008 ≈ $1,200/mo
- **Rollout:** 5% → 25% → 100%, human review 100% → confidence-based
- **Monitoring:** escalation_rate, human_edit_rate ⭐ (draft kitna badla jaata hai — quality proxy), CSAT

---

## Case 3: Code Review Assistant

> **Prompt:** "PRs pe automatic code review. 50 developers, ~100 PRs/day."

### Clarify
- Repo size? Languages?
- Existing linters/SAST hai? (**haan hona chahiye** — LLM unhe replace na kare)
- Comment inline ya summary?
- False positive tolerance? (**bahut kam** — noisy bot ignore ho jaata hai)

### Paradigm
**Agent** ✅ — aur ye rare case hai jahan agent genuinely justified hai:
- Complexity: kaunsi files padhni hain, kaunsa context chahiye — pata nahi
- **Verification sasti hai** (tests, linter, compiler)
- Error cost kam (human review karega, comment ignore ho sakta hai)

### Architecture

```
PR webhook
  ↓
[1] Deterministic checks FIRST (CODE)
    linter, formatter, SAST, test results
    → ye LLM se pehle, kyunki fast + free + reliable
  ↓
[2] Context gather
    diff + changed files + related files (imports) + PR description
  ↓
[3] AGENT with tools:
    read_file, search_code, get_test_results, get_git_blame
    Guards: max_steps=15, max_cost=$0.50, read-only
  ↓
[4] Findings with confidence + severity
  ↓
[5] Filter (CODE) — confidence threshold, dedupe with linter findings
  ↓
[6] Post inline comments
```

### 🔑 Key insight (ye bolo)
> **Modern models severity filters ko literally follow karte hain.** Agar prompt kehta hai "only report high-severity issues", model thoroughly investigate karega, bugs identify karega, **aur phir report nahi karega** — measured recall gir jaayegi jabki actual bug-finding improve hui hai.
>
> **Fix:** Model ko bolo **sab report karo** confidence + severity ke saath, phir **downstream code se filter karo**.

### Key decisions

| Decision | Reason |
|---|---|
| **Deterministic checks pehle** | Linter/SAST fast, free, reliable. LLM unhe duplicate na kare. |
| **Read-only tools** | Agent code modify na kare — sirf comment kare |
| **Confidence + severity output** | Filtering downstream, model pe nahi |
| **Cost cap per PR** | 100 PRs/day × $0.50 = $50/day max |

### Evals
- **30 real PRs** with known bugs (ground truth)
- Precision aur recall dono (precision zyada important — noise se bot ignore ho jaata hai)
- **3 runs per PR** (non-determinism)
- Cost per PR, latency per PR
- Developer feedback: comment useful tha? (thumbs)

### Production
- **Cost:** 100 PRs/day × ~$0.30 ≈ $900/mo
- **Rollout:** opt-in repos pehle
- **Kill switch** — noisy ho to turant band

---

## Case 4: Multi-tenant SaaS AI Feature

> **Prompt:** "Tumhare B2B SaaS mein AI assistant add karna hai. 500 customers, har ek ka apna data."

### 🔒 Ye case **security-first** hai

### Critical decisions

**1. Tenant isolation — har layer pe**
```python
# Retrieval
chunks = search(query, filter={"tenant_id": ctx.tenant_id})    # MANDATORY

# Cache — sabse common bug yahan hota hai
cache_key = f"{ctx.tenant_id}:{hash(query)}"                   # 🔒

# Trace
tracer.set("tenant_id", ctx.tenant_id)

# Budget
if await month_spend(ctx.tenant_id) > ctx.budget: raise BudgetExceeded()
```

> 🚨 **Cache mein tenant_id na daalna = data breach.** Tenant A ka answer Tenant B ko. Eval mein iska test **zaroor**.

**2. Vector store: shared vs isolated**

| Approach | Pros | Cons |
|---|---|---|
| **Shared index + filter** ⭐ | Simple ops, cost-efficient | Filter bug = leak |
| Namespace per tenant | Better isolation | 500 namespaces manage |
| Separate DB per tenant | Best isolation | Ops nightmare at 500 |

**Recommendation:** Shared + strict filter + **automated leak tests**. Enterprise tier ke liye namespace option.

**3. Cost control per tenant**
```python
TIER_LIMITS = {
    "free":       {"queries_day": 50,   "budget_month": 5},
    "pro":        {"queries_day": 500,  "budget_month": 100},
    "enterprise": {"queries_day": 5000, "budget_month": 2000},
}
```
Unit economics **din 1 se** track karo — warna ek heavy tenant poora margin kha jaayega.

**4. Noisy neighbour**
- Per-tenant rate limits
- Priority queues (enterprise pehle)
- Per-tenant concurrency caps

### Evals
- **Cross-tenant leak tests** ⭐ — zero tolerance, har release
- Per-tenant quality (data quality vary karti hai!)
- Cost per tenant tier

### Production
- Per-tenant dashboards
- Budget alerts (80% pe warn)
- Tenant-level kill switch

---

## Case 5: Research Assistant (Agentic)

> **Prompt:** "Ek research assistant jo web + internal docs se comprehensive reports banaye."

### Paradigm
**Agent** ✅ — genuinely open-ended, steps unknown.

### Architecture

```
Query
  ↓
[1] Plan (decompose into sub-questions)
  ↓
[2] ORCHESTRATOR + WORKERS (parallel) ⭐
    Worker 1: web research on sub-Q1  → 300-word report
    Worker 2: internal docs on sub-Q2 → 300-word report
    Worker 3: data analysis on sub-Q3 → 300-word report
    (har worker ka apna context, apna budget)
  ↓
[3] Synthesize (orchestrator, clean context)
  ↓
[4] Verify claims against sources
  ↓
[5] Report with citations
```

### 🔑 Multi-agent yahan kyun justified hai
**Context isolation.** Har worker 50K tokens padhta hai, 500 tokens return karta hai. Single agent 150K tokens accumulate kar leta aur report quality gir jaati.
Plus parallelism (3x faster).

### Key decisions

| Decision | Reason |
|---|---|
| **Workers ko poora brief** | Unhone main conversation nahi dekhi |
| **Return format constrain** | "Max 300 words, har claim ka source" — warna 20K tokens wapas |
| **Nested delegation nahi** | Exponential cost |
| **Per-worker budget** | max_steps=15, max_cost=$0.30 |
| **Async, HTTP nahi** | 5-minute run = job queue + SSE progress |

### 🔒 Security
Web content **untrusted** hai:
- Workers ke paas outbound network tools **nahi** (sirf fetch, jo controlled hai)
- Reader/actor separation
- Egress allowlist

### Evals
- 20 research tasks with expert-written reference reports
- **3 runs each** (non-determinism)
- Metrics: factual accuracy, source quality, coverage, **cost per report** ⭐
- Human evaluation on sample

### Production
- **Cost:** ~$2-5 per report — **usage caps mandatory**
- Job queue, progress streaming
- Checkpointing (long runs crash hote hain)

---

## 📝 Interview mein kya bolna hai (aur kya nahi)

### ✅ Strong signals
- **"Pehle main measure karunga"** — eval strategy mention
- **"Ye workflow hai, agent nahi, kyunki..."** — paradigm justify
- **"ACL retrieval layer mein hogi"** — security awareness
- **"Hybrid search kyunki identifiers miss ho jaate hain"** — specific reasoning
- **Cost estimate** with numbers
- **Failure modes** aur handling
- **"Ye trade-off hai: X better hoga par Y kharab"**

### ❌ Weak signals
- Seedha "LangChain use karunga" (framework-first thinking)
- Har cheez ko agent banana
- Evals mention na karna
- Security skip karna
- Cost/latency estimate na dena
- "AI ye kar lega" (magic thinking)

### 🎯 Ek line jo har case mein bolni chahiye
> **"Main sabse simple approach se shuru karunga jo kaam kare, ek eval suite banaunga measure karne ke liye, aur phir data ke basis pe complexity add karunga."**

Ye ek line senior aur junior AI engineer ka farak dikhati hai.

---

## 🏋️ Khud practice karo

In cases ko khud design karo (solutions nahi diye):

1. **WhatsApp business bot** — 50,000 messages/day, Hindi+English, order tracking + FAQ
2. **Legal contract analyzer** — 200-page contracts, clause extraction, risk flagging
3. **Medical records assistant** — PHI, compliance, doctor-facing, India data residency
4. **E-commerce product search** — 1M products, natural language + filters, personalization
5. **DevOps incident assistant** — logs + metrics + runbooks, on-call support
6. **Educational tutor** — adaptive, student progress memory, multi-subject

Har ek ke liye **poora framework** use karo (clarify → paradigm → architecture → deep dive → evals → production).

---

**Next:** [`../GLOSSARY.md`](../GLOSSARY.md)
