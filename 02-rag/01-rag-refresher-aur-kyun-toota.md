# 01 — RAG Refresher aur "kyun toota"

> **Tumhe basics aate hain**, isliye ye file jaldi refresh karegi aur phir asli value pe jaayegi: **RAG kahan aur kyun fail hota hai.**
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/04-rag-pipeline`](../08-diagrams/04-rag-pipeline.excalidraw) — poori pipeline + "kyun tootta hai" ek jagah.

---

## Part 1: 60-second refresher

```
INDEXING (offline, ek baar):
  Documents → Chunks → Embeddings → Vector DB

QUERY TIME (online, har request):
  User query → Embedding → Similarity search → Top-K chunks
             → Prompt mein daalo → LLM → Answer (+ citations)
```

**RAG kyun exist karta hai — 4 reasons:**

| Reason | Detail |
|---|---|
| **Knowledge cutoff** | Model ko training ke baad ki cheezein nahi pata |
| **Private data** | Tumhare internal docs model mein nahi hain |
| **Citations** | "Ye answer document X, page 12 se hai" — enterprise mein mandatory |
| **Updatability** | Naya doc = index update (seconds). Fine-tune = retrain (days) |

---

## Part 2: Naive RAG ka anatomy — aur uske 7 failure points

Yahan asli value hai. Ye diagram dekho — **har arrow ek failure point hai:**

```
User query
    │  ❶ Query embedding-friendly nahi hai
    ▼
Embed query
    │  ❷ Embedding model domain samajhta nahi
    ▼
Vector search  ←──────── Vector DB ←── Chunks ←── Documents
    │  ❸ Semantic ≠ relevant        ❺ Chunk    ❻ Parsing
    ▼                                boundaries    kharab
Top-K chunks
    │  ❹ Ranking galat (best chunk #8 pe hai, K=5)
    ▼
Prompt assemble
    │  ❼ Chunks ka order/format kharab
    ▼
LLM generate
    │  ❽ Model context ignore karke apna knowledge use karta hai
    ▼
Answer
```

---

## Part 3: Saat failure modes — diagnosis ke saath

### Failure 1: Query-Document mismatch 🔥 (sabse common)

**Problem:** User ka sawaal aur document ki language different hoti hai.

```
User query:     "kitni chhutti milti hai?"
Document text:  "Employees are entitled to 24 days of annual paid leave
                 per calendar year as per the Leave Policy 2024."
```
Embedding similarity: **kam** — casual Hindi query vs formal English policy text.

**Diagnosis:**
```python
# Query embed karo, expected chunk embed karo, similarity dekho
print(cosine_sim(embed(query), embed(known_correct_chunk)))
# Agar 0.5 se kam hai → ye failure mode hai
```

**Fixes:**
1. **Query rewriting** — LLM se query ko document-style mein rewrite karao
2. **HyDE** — LLM se ek *fake answer* generate karao, usko embed karo (answer, answer se match karta hai)
3. **Hybrid search** — keyword search ("chhutti"/"leave") semantic gap bhar deta hai
4. **Multi-query** — ek query ke 3-4 variations generate karo, sabke results merge karo

### Failure 2: Embedding model domain nahi samajhta

**Problem:** General-purpose embedding tumhare domain jargon ko nahi jaanta.

```
Medical: "MI" = myocardial infarction? ya "Michigan"?
Legal:   "consideration" ka legal meaning ≠ normal meaning
Internal: "Project Falcon" — model ko kya pata ye kya hai
```

**Diagnosis:** Domain-specific query pe retrieval fail, general query pe kaam kare.

**Fixes:**
1. **Domain-specific embedding model** use karo (medical/legal/code models exist karte hain)
2. **Embedding fine-tuning** (tumhare query-document pairs pe) — mehnga par effective
3. **Glossary expansion** — query mein abbreviations expand kar do before embedding
4. **Hybrid search** — exact jargon keyword se mil jaayega

### Failure 3: Semantic similar ≠ relevant

**Problem:** Chunk topic-wise similar hai, par **sawaal ka jawab nahi** deta.

```
Query: "2024 mein revenue kitna tha?"

Retrieved (high similarity, useless):
  "Revenue growth strategies mein hum focus karenge..."     ← revenue ka zikr, data nahi
  "Revenue recognition policy ke hisaab se..."               ← revenue ka zikr, data nahi
  "Historically revenue seasonal raha hai..."                ← revenue ka zikr, data nahi

Actual answer (lower similarity, kho gaya):
  "FY2024: ₹450 crore"                                       ← chhota chunk, kam similarity!
```

**Ye embeddings ki fundamental limitation hai** — wo "aboutness" measure karti hain, "answerhood" nahi.

**Fixes:**
1. **Reranking** ⭐ — cross-encoder actually *query-document relevance* judge karta hai. **Ye sabse high-ROI fix hai.**
2. **Zyada retrieve, kam pass** — top-50 retrieve karo, rerank karo, top-5 LLM ko do
3. Metadata filters (year=2024)

### Failure 4: Ranking galat hai

**Problem:** Sahi chunk retrieve hua, par rank #12 pe hai. Tum `top_k=5` use kar rahe ho.

**Diagnosis (ye metric measure karo!):**
```python
# Recall@K — sahi chunk top-K mein hai ya nahi?
recall_at_5  = 0.62      # 38% queries mein sahi chunk top-5 mein hi nahi!
recall_at_20 = 0.91      # top-20 mein hai
recall_at_50 = 0.96
```

**Interpretation:** Recall@50 acha hai (0.96) par Recall@5 kharab (0.62) → **retrieval theek hai, ranking kharab hai.**

**Fix:** Reranking. `top_k=50` retrieve karo, cross-encoder se rerank karo, top-5 pass karo.
```
Recall@5 after reranking: 0.62 → 0.89   🔥
```

### Failure 5: Chunk boundaries ne answer kaat diya

```
Chunk 12 ends: "...parental leave policy ke hisaab se employee ko"
Chunk 13 starts: "26 hafte ki paid leave milti hai, jiske liye..."
```

Koi bhi single chunk poora answer nahi rakhta. Retrieval "correct" hai par answer adhoora.

**Fixes:** Overlap, semantic chunking, parent-document retrieval, sentence-window retrieval → agli file mein detail.

### Failure 6: Document parsing kharab thi

**Sabse underrated failure mode.** Garbage in, garbage out.

```
PDF table:
┌──────────┬────────┐
│ Product  │ Price  │
├──────────┼────────┤
│ Widget A │ ₹500   │
└──────────┴────────┘

Naive text extraction:
"Product Price Widget A ₹500"     ← structure gaya, ya worse:
"Product\nPrice\nWidget A\n₹500"  ← column order mix
```

**Diagnosis:** Extracted text **khud padho**. Ye sabse simple aur sabse skipped debug step hai.

**Fixes:**
- Better parsers (layout-aware: Unstructured, LlamaParse, Docling, Azure Document Intelligence)
- Tables ko markdown/HTML mein convert karo (structure preserve)
- Vision model se page images parse karao (scanned docs ke liye)
- Headers/footers/page numbers strip karo

### Failure 7: Generation stage failure

Sahi chunks mil gaye, phir bhi answer galat:

| Sub-failure | Symptom | Fix |
|---|---|---|
| **Context ignore** | Model apna training knowledge use karta hai | Prompt: "SIRF diye gaye context se answer do" |
| **Hallucinated citations** | "[Doc 3]" jab Doc 3 mein wo nahi hai | Chunks ko IDs do, citation validate karo code se |
| **Over-hedging** | "Documents suggest possibly..." | Prompt: "Direct answer do jab context clear ho" |
| **Wrong chunk pick** | 5 chunks the, model ne galat chuna | Chunks kam karo, reranking, chunks mein source label do |
| **Conflict handling** | Do chunks contradict, model ek chun leta hai | Prompt: "Agar conflict ho to dono batao source ke saath" |

---

## Part 4: 🔬 Diagnosis framework (ye sabse zaroori hai)

**Sabse bada mistake:** "RAG kaam nahi kar raha" bolke random cheezein badalna.

**Sahi approach: retrieval aur generation ko ALAG measure karo.**

```python
# Step 1: Ek golden set banao (20-50 queries)
golden = [
    {"query": "parental leave kitni?",
     "expected_chunk_ids": ["hr_policy_c12", "hr_policy_c13"],
     "expected_answer": "26 hafte paid"},
    ...
]

# Step 2: RETRIEVAL measure karo (LLM ko bilkul mat chhedo)
for case in golden:
    retrieved_ids = [c.id for c in retrieve(case["query"], k=5)]
    hit = any(e in retrieved_ids for e in case["expected_chunk_ids"])
    ...
print(f"Recall@5: {recall_5}")

# Step 3: GENERATION measure karo (PERFECT chunks manually do)
for case in golden:
    answer = generate(case["query"], context=get_chunks(case["expected_chunk_ids"]))
    # answer sahi hai?
print(f"Generation accuracy (perfect context): {gen_acc}")
```

### Decision matrix

| Recall@5 | Gen accuracy (perfect context) | Diagnosis | Kya fix karo |
|---|---|---|---|
| **Low** (< 0.7) | High (> 0.9) | **Retrieval problem** | Chunking, hybrid, reranking, query rewriting |
| High (> 0.9) | **Low** (< 0.7) | **Generation problem** | Prompt, model, output format |
| **Low** | **Low** | Dono | Pehle retrieval fix karo (upstream) |
| High | High | Pipeline theek hai | Problem edge cases/scope mein hai |

> 🔑 **Ye ek table poore RAG debugging ko systematic bana deta hai.** Iske bina tum guess karte rehte ho.
>
> **Empirical reality:** 80% RAG failures **retrieval** mein hote hain. Log 80% time **prompt** pe kharch karte hain. Ulta hai.

---

## Part 5: RAG evolution ladder

```
Level 0: NAIVE RAG
   chunk → embed → cosine search → top-5 → LLM
   ✅ 2 din mein ban jaata hai
   ❌ Production mein 50-65% accuracy

Level 1: + HYBRID SEARCH + RERANKING       ⭐ SABSE ZYADA ROI
   BM25 + vector → merge → cross-encoder rerank → top-5
   ✅ 65% → 82% accuracy, ~3 din ka kaam

Level 2: + BETTER CHUNKING + METADATA
   semantic chunking, parent-doc retrieval, metadata filters
   ✅ 82% → 88%

Level 3: + QUERY UNDERSTANDING
   query rewriting, decomposition, HyDE, multi-query
   ✅ 88% → 91%

Level 4: AGENTIC RAG
   LLM khud decide kare: search karna hai? kya search? kaafi hai?
   ✅ 91% → 94% (complex multi-hop queries pe bada jump)
   ❌ 3-5x cost, 3x latency

Level 5: + SELF-CORRECTION
   retrieved chunks grade karo, kaafi na ho to dobara search
   ✅ Reliability best, cost highest
```

> **Practical advice:** **Level 1 pe jao pehle.** Wahan ROI sabse zyada hai (17 percentage points, 3 din ka kaam). Level 4-5 pe log jump kar jaate hain hype ki wajah se — aur unka Level 1 bhi theek nahi hota.

---

## Part 6: Kab RAG use NA karo

RAG hammer hai, sab nail nahi hai:

| Situation | Better option |
|---|---|
| Data chhota hai (< 50 pages) | Poora context mein daal do — retrieval hi na karo |
| Structured queries ("Q3 mein top 10 customers") | **SQL / text-to-SQL** — vector search numbers pe bekaar hai |
| Aggregation ("total kitna", "average kya") | SQL. RAG count nahi kar sakta. |
| Real-time data (stock price, weather) | **API tool call** |
| "Poore document ka summary do" | Full document context mein daalo — retrieval information kaat dega |
| Behaviour/format problem | Prompting ya fine-tuning, RAG nahi |
| Multi-hop reasoning across many docs | Agentic RAG ya GraphRAG |

> 🚩 **RAG ka sabse common misuse:** Aggregation queries. "Hamare paas kitne customers hain?" — vector search 5 chunks laayega jinme customer ka zikr hai. Model 5 se count karke galat number dega, **confidently**. Iske liye SQL chahiye.
>
> Isliye production systems mein often ek **router** hota hai: query dekho → RAG, SQL, ya API pe bhejo.

---

## Khud se pooch 🤔

1. RAG ke 7 failure points naam se batao (pipeline order mein).
2. "Semantic similar ≠ relevant" ko ek example se explain karo. Best fix kya hai?
3. Recall@5 = 0.6 aur Recall@50 = 0.95 hai. Diagnosis kya? Fix kya?
4. Retrieval aur generation ko **alag** measure karna kyun zaroori hai? Kaise karoge?
5. Query-document mismatch ke 4 fixes batao.
6. Ye queries RAG ke liye theek hain? Kyun/kyun nahi?
   - (a) "Hamari refund policy kya hai?"
   - (b) "Pichhle quarter mein kitne refunds hue?"
   - (c) "Is 200-page contract ka summary do"
   - (d) "Aaj Nifty kitna hai?"
7. Tumhari RAG app 55% accuracy pe hai. Pehle 3 kadam kya honge, kis order mein?

<details>
<summary>Answers 6 aur 7</summary>

**6.**
- **(a) ✅ RAG perfect.** Semantic lookup, answer ek document mein hai, citation possible.
- **(b) ❌ RAG galat.** Ye **aggregation** hai. Vector search "refund" wale chunks laayega, count nahi karega. Model ka answer confidently galat hoga. → **SQL / analytics query** chahiye.
- **(c) ⚠️ RAG galat approach.** Summary ke liye **poora** document chahiye. Retrieval 5 chunks dega = 5 chunks ka summary, document ka nahi. → Full doc context mein daalo (200 pages ≈ 100K tokens, fits), ya hierarchical summarization (chunk-wise summary → summary of summaries).
- **(d) ❌ RAG galat.** Real-time data. → **API tool call**.

**Lesson:** Production system ko ek **router** chahiye jo query type classify kare aur sahi backend pe bheje. Sab kuch RAG mein daalna common architecture mistake hai.

**7. 55% accuracy pe, is order mein:**

**Kadam 1 — MEASURE karo (fix karne se pehle!).** 30-query golden set banao expected chunks + answers ke saath. Retrieval aur generation **alag** measure karo. Bina iske tum andhere mein change kar rahe ho aur pata nahi chalega ki improve hua ya nahi.

**Kadam 2 — Parsed text khud padho.** 5 random chunks print karo. Aksar yahan shocking cheezein milti hain — tables mashed hain, headers repeat ho rahe hain, poora page khaali aa raha hai. Ye **sabse sasta, sabse high-yield** debug step hai aur sab log skip karte hain.

**Kadam 3 — Level 1 pe jao: hybrid search + reranking.**
- BM25 add karo (`rank_bm25` library, 20 lines) + vector search, RRF se merge
- Cross-encoder reranker add karo (`bge-reranker` ya provider API)
- `top_k` 50 retrieve karo → rerank → top-5 LLM ko

Ye combination typically 55% → 78-82% karta hai, ~3 din mein.

**Kadam 4 (uske baad, measure karke):** chunking strategy, query rewriting, metadata filters.

**Jo NA karo:** Agentic RAG ya GraphRAG pe jump. Wo Level 1 ke bina paisa aur time waste hai.
</details>

---

**Next:** [`02-chunking-aur-indexing.md`](02-chunking-aur-indexing.md)
