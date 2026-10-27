# 05 — Agentic RAG aur GraphRAG

> **Is file ka goal:** RAG ka aakhri evolution level. Aur ek imaandaar reality check — **ye techniques hype se kam useful hain jitna log kehte hain**, par specific cases mein genuinely powerful.

---

## Part 1: Agentic RAG — retrieval ko tool bana do

### Core shift

```
CLASSIC RAG (retrieval fixed hai, pipeline mein):
  query → ALWAYS retrieve → ALWAYS generate → answer

AGENTIC RAG (retrieval ek TOOL hai, model decide karta hai):
  query → LLM sochta hai
            ├─ retrieve karna chahiye? kya search karun?
            ├─ tool call: search("...")
            ├─ result dekha — kaafi hai?
            ├─ nahi → alag query se dobara search
            └─ haan → answer generate karo
```

**Ye Day 2 ka tool calling hai, retrieval pe apply kiya hua.** Kuch naya nahi — ek reframing.

### Implementation (surprisingly simple)

```python
tools = [
    {
        "name": "search_docs",
        "description": (
            "Company documentation mein semantic search karta hai.\n"
            "KAB USE KARO: jab answer company-specific policies, processes, "
            "ya internal docs pe depend karta ho.\n"
            "KAB NA KARO: general knowledge ke liye, ya jab answer "
            "conversation mein already hai.\n"
            "TIP: alag-alag phrasings se kai baar search kar sakte ho.\n"
            "RETURNS: top 5 excerpts source ke saath. Kuch na mile to empty."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string",
                          "description": "Natural language search query"},
                "category": {"type": "string",
                             "enum": ["hr", "engineering", "legal", "all"]}
            },
            "required": ["query"]
        }
    }
]

SYSTEM = """Tum ek research assistant ho.

- Company-specific sawaalon ke liye search_docs use karo.
- Pehle search kaafi na ho, to ALAG phrasing se dobara search karo.
- Complex sawaal ko todo aur alag-alag searches karo.
- Sirf retrieved documents se answer do, citations ke saath.
- Kuch na mile to clearly bolo — bana ke na batao.
"""

# Standard agent loop (Day 2, file 03)
messages = [{"role": "user", "content": query}]
while True:
    resp = client.messages.create(model=MODEL, tools=tools,
                                  system=SYSTEM, messages=messages, max_tokens=4096)
    messages.append({"role": "assistant", "content": resp.content})
    if resp.stop_reason != "tool_use":
        break
    results = [execute_search(b) for b in resp.content if b.type == "tool_use"]
    messages.append({"role": "user", "content": results})
```

### Agentic RAG kya unlock karta hai

| Capability | Example |
|---|---|
| **Query reformulation** | Pehla search fail → khud better query banata hai |
| **Multi-hop** | "top customer ka contract" → 2 dependent searches |
| **Multi-source** | Docs + SQL + web, sab tools hain — model chunta hai |
| **Sufficiency judgement** | "ye kaafi nahi hai, aur dhundhta hoon" |
| **Selective retrieval** | "namaste" pe search nahi karega (cost bacha) |
| **Comparison** | Har entity ke liye alag search khud karta hai |

### 💰 Cost reality

```
Classic RAG:   1 embedding + 1 search + 1 LLM call     = ~$0.004, 3s
Agentic RAG:   1-4 LLM calls + 1-4 searches            = ~$0.02,  8-20s
               (aur reasoning model ke saath 3x zyada)
```

**5x cost, 4x latency.** Ye significant hai.

### Kab agentic RAG use karo

| ✅ Haan | ❌ Nahi |
|---|---|
| Complex analytical queries | Simple factual lookups |
| Multi-hop reasoning chahiye | Single-document answers |
| Kai data sources hain | Ek corpus |
| Queries unpredictable hain | Queries predictable pattern mein |
| Accuracy > cost/latency | High volume, cost-sensitive |
| Research/analysis use case | FAQ bot |

### 🔥 Best pattern: hybrid routing

```python
def smart_rag(query):
    complexity = classify(query)          # chhota fast model, ~100ms

    if complexity == "simple":
        return classic_rag(query)         # 90% queries, sasta, fast
    else:
        return agentic_rag(query)         # 10% queries, mehnga, powerful
```

**Ye production mein best economics deta hai** — 90% queries sasti rehti hain, 10% ko power milti hai. Blanket agentic RAG paisa jalata hai.

---

## Part 2: GraphRAG — relationships pe reasoning

### Problem jo ye solve karta hai

```
Query: "Kaunse projects un vendors pe depend karte hain jinke contracts
        Q3 mein expire ho rahe hain?"
```

Ye information **kisi ek document mein nahi hai.** Ye relationships mein chhupi hai:
```
Project A  ──uses──→  Vendor X  ──has──→  Contract 123  ──expires──→  2026-09-15
Project B  ──uses──→  Vendor X
Project C  ──uses──→  Vendor Y  ──has──→  Contract 456  ──expires──→  2027-01-01
```

Vector search yahan structurally fail hota hai — wo **similar text** dhundhta hai, **connected facts** nahi.

### GraphRAG kaise kaam karta hai

```
INDEXING (mehnga!):
  Documents
     ↓ LLM se extract karo
  Entities: [Project A, Vendor X, Contract 123, ...]
  Relations: [(Project A, uses, Vendor X), (Vendor X, has, Contract 123), ...]
     ↓
  Knowledge Graph banao
     ↓
  Communities detect karo (clusters), har community ka summary banao

QUERY:
  Local query  → relevant entities dhundo → graph traverse karo → neighbors lao
  Global query → community summaries use karo ("poore corpus ka theme kya hai")
```

### Entity extraction (indexing ka core)

```python
EXTRACT = """Is text se entities aur relationships extract karo.

JSON return karo:
{
  "entities": [{"name": "...", "type": "person|org|project|contract|...",
                "description": "..."}],
  "relations": [{"source": "...", "target": "...", "type": "...",
                 "description": "..."}]
}

Text: {chunk}"""
```

Har chunk pe ek LLM call. **Yahan cost aata hai.**

### 💰 Cost reality (imaandaar)

```
10,000 documents ka corpus:

Classic RAG indexing:
  Embedding: ~$5
  Time: ~30 minutes

GraphRAG indexing:
  Entity extraction: 10,000 chunks × LLM call     ≈ $200-800
  Community summarization: hundreds of calls      ≈ $50-200
  Time: hours to days
  → 50-150x MEHNGA
```

Aur **incremental updates painful hain** — naya document graph ke poore structure ko affect kar sakta hai.

### GraphRAG ka imaandaar assessment

| ✅ Genuinely better | ❌ Overkill / worse |
|---|---|
| Multi-hop relationship queries | Simple factual lookup |
| "Poore corpus ka theme kya hai" (global) | Single-document answers |
| Entity-centric domains (people, orgs, contracts) | Unstructured prose |
| Impact analysis ("X badla to kya affect hoga") | FAQ |
| Investigative/compliance work | High-volume production Q&A |
| Static corpus | Frequently updating corpus |

> 🚩 **Imaandaar advice:** GraphRAG **bahut** hyped hai. 90% RAG applications ko iski zaroorat **nahi** hai.
>
> **Sasta 80% alternative:** Structured metadata + filters. Agar tumhara data mein relationships hain, unhe **SQL table** mein daalo aur text-to-SQL use karo. 100x sasta, 90% capability, aur exact answers (aggregation bhi kar sakta hai!).

### Lightweight alternative (recommended)

```python
# Full graph ke bajaye — hybrid approach
# 1. Entities ko ek SQL table mein daalo (relationships ke saath)
# 2. Text ko normal RAG mein rakho
# 3. Do tools do agent ko:

tools = [
    {"name": "search_docs",       # semantic search
     "description": "Documents mein text search"},
    {"name": "query_entities",    # SQL
     "description": "Entity relationships query karo. Schema: "
                    "projects(id,name), vendors(id,name), "
                    "project_vendors(project_id,vendor_id), "
                    "contracts(id,vendor_id,expiry_date)"}
]
# → Model relationship queries SQL se karega, text queries RAG se
```

Ye **agentic RAG + SQL** pattern GraphRAG ka 80% value 5% cost pe deta hai. Aur exact aggregations bhi kar sakta hai, jo GraphRAG bhi struggle karta hai.

---

## Part 3: Multi-source RAG (real production shape)

Real systems mein ek corpus nahi hota:

```python
tools = [
    {"name": "search_internal_docs",
     "description": "Company policies, runbooks, HR docs (semantic)."},

    {"name": "query_database",
     "description": "Customer/order/revenue data pe SQL. Counts, sums, "
                    "rankings ke liye ISKO use karo. Schema: ..."},

    {"name": "search_codebase",
     "description": "Code aur technical docs search."},

    {"name": "get_realtime_metrics",
     "description": "Live system metrics, current status."},

    {"name": "web_search",
     "description": "Public/current information jo internal docs mein nahi."},
]
```

**Model router ban jaata hai** — wo decide karta hai kaunsa source. Ye ek elegant architecture hai.

### 🔑 Yahan tool descriptions **critical** ho jaati hain

Agar `search_internal_docs` aur `search_codebase` ki descriptions overlap karti hain, model galat chunega — aur tumhe pata bhi nahi chalega ki kyun answer kharab aaya.

```python
# ❌ Overlapping
"search_docs": "Documentation search karta hai"
"search_code": "Code documentation search karta hai"
#                     ↑ dono "documentation"! Model confused.

# ✅ Clear boundaries
"search_internal_docs":
    "HR policies, business processes, compliance docs. "
    "CODE ke liye ye NAHI — uske liye search_codebase use karo."
"search_codebase":
    "Source code, API references, technical architecture docs. "
    "BUSINESS policies ke liye ye NAHI — search_internal_docs use karo."
```

---

## Part 4: RAG evolution — final decision guide

```
                    START
                      │
        ┌─────────────┴─────────────┐
        │ Data < 50 pages?          │
        └──────┬──────────────┬─────┘
           YES │              │ NO
               ▼              ▼
      Poora context     ┌──────────────────────┐
      mein daalo        │ Queries mostly simple │
      (RAG hi na karo)  │ lookups?              │
                        └────┬───────────┬──────┘
                         YES │           │ NO
                             ▼           ▼
                  ┌────────────────┐  ┌─────────────────────┐
                  │ CLASSIC RAG    │  │ Multi-hop /         │
                  │ + hybrid       │  │ multi-source?       │
                  │ + reranking    │  └───┬─────────────┬───┘
                  │ ⭐ 80% cases   │  YES │             │ NO
                  └────────────────┘      ▼             ▼
                                  ┌───────────────┐  ┌──────────────┐
                                  │ AGENTIC RAG   │  │ Aggregation? │
                                  │ (+ routing)   │  └──┬───────┬───┘
                                  └───────────────┘ YES │       │ NO
                                                        ▼       ▼
                                                    SQL /    Classic RAG
                                                  text-to-SQL   theek hai
```

### Aur GraphRAG kahan hai?

**Us decision tree mein GraphRAG jaan-boojh kar nahi hai.** Wo tab consider karo jab:
1. Classic RAG + hybrid + rerank already chal raha hai ✅
2. Tumne measure kiya ki relationship queries specifically fail ho rahi hain ✅
3. SQL + agentic approach try kiya aur kaafi nahi tha ✅
4. Corpus mostly static hai ✅
5. Indexing cost justify ho sakti hai ✅

**Saare 5 tick nahi hue?** GraphRAG na karo.

---

## Part 5: Production RAG architecture (poora picture)

```
                        USER QUERY
                             │
                    ┌────────▼─────────┐
                    │  Query Rewriting  │  ← multi-turn ke liye MANDATORY
                    │  (pronouns fix)   │
                    └────────┬──────────┘
                             │
                    ┌────────▼──────────┐
                    │  Semantic Cache    │─── HIT ──→ return
                    └────────┬───────────┘
                             │ MISS
                    ┌────────▼──────────┐
                    │      ROUTER        │
                    └─┬────┬────┬────┬───┘
            ┌─────────┘    │    │    └────────┐
            ▼              ▼    ▼             ▼
        ┌────────┐   ┌────────┐ ┌────────┐ ┌──────────┐
        │  RAG   │   │  SQL   │ │  API   │ │Full-ctx  │
        └───┬────┘   └───┬────┘ └───┬────┘ │summarize │
            │            │          │      └────┬─────┘
    ┌───────▼───────┐    │          │           │
    │Hybrid retrieve│    │          │           │
    │ (BM25+vector) │    │          │           │
    │ + ACL filter🔒│    │          │           │
    └───────┬───────┘    │          │           │
    ┌───────▼───────┐    │          │           │
    │   Reranking   │    │          │           │
    └───────┬───────┘    │          │           │
    ┌───────▼───────┐    │          │           │
    │ Quality gate  │    │          │           │
    │ (low? abstain)│    │          │           │
    └───────┬───────┘    │          │           │
            └────────────┴──────────┴───────────┘
                             │
                    ┌────────▼─────────┐
                    │    GENERATION     │
                    │ (grounded prompt, │
                    │  citations)       │
                    └────────┬──────────┘
                    ┌────────▼──────────┐
                    │Citation validation│
                    └────────┬──────────┘
                    ┌────────▼──────────┐
                    │ Trace + eval log  │  ← Day 6
                    └────────┬──────────┘
                             ▼
                          ANSWER
```

**Ye "over-engineered" nahi hai** — ye production RAG ka realistic shape hai. Har box ek specific failure mode fix karta hai jo tumne Day 3 mein padha.

---

## Khud se pooch 🤔

1. Agentic RAG aur classic RAG mein **structural** farak kya hai? Kaun decide karta hai?
2. Agentic RAG ke 6 capabilities batao. Cost/latency ka trade-off?
3. Hybrid routing (simple→classic, complex→agentic) kyun best economics deta hai?
4. GraphRAG kaunsa problem solve karta hai jo vector search **structurally** nahi kar sakta?
5. GraphRAG ka indexing cost classic RAG se 50-150x kyun hai?
6. GraphRAG ka "80% value, 5% cost" alternative kya hai? Wo kaise kaam karta hai?
7. Multi-source RAG mein tool descriptions kyun critical ho jaati hain? Ek overlapping-description bug ka example do.
8. Ek startup kehta hai "hum GraphRAG use kar rahe hain" par unka Recall@5 0.6 hai. Tumhara diagnosis?

<details>
<summary>Answer 8</summary>

**Diagnosis: unhone foundations skip kar diye aur seedha shiny layer pe jump kar gaye.**

Recall@5 = 0.6 ka matlab hai ki **basic retrieval hi toota hua hai** — 40% queries mein sahi information top-5 mein nahi aa rahi. GraphRAG ke saath ye bilkul possible hai, kyunki GraphRAG **retrieval quality fix nahi karta** — wo ek alag *access pattern* (relationships) add karta hai.

**Kya likely hua:**
1. Naive chunking (fixed-size, koi structure/enrichment nahi)
2. Sirf vector search, koi BM25 nahi → exact terms, IDs, names miss ho rahe hain
3. Koi reranking nahi → sahi chunk rank 8-15 pe pada hai
4. Parsed text kabhi check nahi kiya
5. Aur ab wo ek 50-150x mehnga indexing layer maintain kar rahe hain jo unki actual problem address hi nahi karta

**Mera recommendation (order mein):**
1. **GraphRAG rok do** (delete na karo, par usme invest band). Bleeding stop.
2. **Golden set banao** (30-50 queries). Retrieval aur generation alag measure karo.
3. **Parsed chunks padho.** Yahan aksar shocking cheezein milti hain.
4. **Hybrid search + reranking add karo.** Expected: 0.6 → 0.85+, ~3 din mein.
5. **Chunk enrichment** (header paths prepend karo).
6. **Ab measure karo:** kaunsi queries *still* fail ho rahi hain? Agar wo genuinely multi-hop relationship queries hain, to GraphRAG (ya better: agentic RAG + SQL) revisit karo — **evidence ke saath**, hype ke saath nahi.

**Meta-lesson:** Advanced techniques **weak foundations ko fix nahi karti — wo unhe chhupa deti hain** aur maintenance cost badha deti hain. Naye AI engineers ki sabse common galti yahi hai: Level 1 skip karke Level 5 pe jump. Level 1 (hybrid + rerank) mein 3 din mein 20 points milte hain; Level 5 mein 3 mahine mein 3 points.
</details>

---

## 🎉 Day 3 complete!

Ab tum jaante ho:
- ✅ RAG ke 7 failure modes aur **systematic diagnosis** (Recall@5 vs Recall@20 table)
- ✅ Chunking strategies, parent-document retrieval, chunk enrichment
- ✅ **Hybrid search + reranking** — highest ROI techniques
- ✅ Generation prompting, citation validation, quality gates
- ✅ Agentic RAG aur GraphRAG — aur kab inhe **na** use karna hai

**Sabse important takeaway:** 80% RAG failures retrieval mein hote hain. Measure karo, guess na karo. Aur Level 1 (hybrid + rerank) sabse pehle.

**Next:** [`../03-agents/01-agent-kya-hai.md`](../03-agents/01-agent-kya-hai.md) — Day 4, agents.
