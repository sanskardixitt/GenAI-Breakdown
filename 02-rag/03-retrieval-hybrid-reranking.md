# 03 — Retrieval, Hybrid Search aur Reranking ⭐ (Highest ROI file)

> **Is file ka goal:** Ye repo ki sabse **practically valuable** file hai. Yahan ke 2 techniques (hybrid + rerank) typically RAG accuracy 15-20 percentage points badha deti hain, ~3 din ke kaam mein.

---

## Part 1: Vector search kaafi kyun nahi hai

Recall karo (Day 1, file 03) — embeddings ki limitations:

### Failure 1: Exact keywords / identifiers

```
Query: "error code E4471 ka fix"

Vector search laata hai:
  "error handling best practices..."       ← "error" ka semantic match
  "troubleshooting common issues..."       ← topically similar
  "debugging guide for developers..."      ← topically similar

Actual answer (nahi mila):
  "E4471: Database connection timeout. Fix: connection pool badhao."
```

**Kyun:** `E4471` ek arbitrary string hai. Embedding space mein uska koi "meaning" nahi hai. Semantic search iske liye blind hai.

**Ye category real-world mein bahut badi hai:**
- Product SKUs, part numbers, error codes
- Person names, company names
- API/function names, variable names
- Version numbers, dates
- Legal citations, case numbers

### Failure 2: Negation

```
"Delhi mein baarish hui"  vs  "Delhi mein baarish NAHI hui"
Cosine similarity: ~0.95   ← almost identical!
```

Embeddings "nahi" ko barely register karte hain.

### Failure 3: Rare terms

Rare terms embedding space mein poorly represented hote hain (training mein kam dikhe). "Project Falcon" ka koi meaningful vector nahi hoga.

---

## Part 2: Keyword search (BM25) — purana par zaroori

BM25 ek statistical scoring hai (semantic nahi):

```
BM25 score = Σ (term frequency × inverse document frequency × length normalization)
              ↑ term docs mein kitni baar hai
                            ↑ term kitna RARE hai poore corpus mein
                                                    ↑ chhote docs ko boost
```

### Ye kya karta hai

| Feature | Kaise |
|---|---|
| **Exact match** | `E4471` mil jaayega — guaranteed |
| **Rare term boost** | Rare word ka weight zyada (IDF) — ye important hai |
| **No training** | Corpus se hi statistics nikal jaati hain |
| **Fast** | Inverted index — milliseconds |
| **Explainable** | Kaunse terms match hue, dikha sakte ho |

### Limitations

| Problem | Example |
|---|---|
| Synonyms nahi samajhta | "car" ≠ "automobile" ≠ "gaadi" |
| Paraphrase nahi samajhta | "chhutti kitni" ≠ "leave entitlement" |
| Word order ignore | Bag of words hai |
| Spelling mistakes | Typo = zero match |

### 🔑 Vector aur BM25 **complementary** hain

| | Vector | BM25 |
|---|---|---|
| Synonyms/paraphrase | ✅ | ❌ |
| Exact IDs/codes | ❌ | ✅ |
| Negation | ❌ | ⚠️ (kuch) |
| Rare terms | ❌ | ✅ |
| Conceptual queries | ✅ | ❌ |
| Typos | ⚠️ | ❌ |

**Isliye HYBRID.** Ye "nice to have" nahi hai — ye **do fundamentally different failure modes** cover karta hai.

---

## Part 3: Hybrid Search — implementation

### Reciprocal Rank Fusion (RRF) — best practical method

Problem: BM25 score (0-30 range) aur cosine similarity (0-1 range) ko kaise combine karein? Scale different hai, normalize karna fragile hai.

**RRF ka elegant solution: scores bhool jao, sirf RANKS use karo.**

```python
def rrf(ranked_lists, k=60):
    """
    ranked_lists: [[doc_id ranked by BM25], [doc_id ranked by vector]]
    k=60: standard constant (top results ko dominate karne se rokta hai)
    """
    scores = {}
    for lst in ranked_lists:
        for rank, doc_id in enumerate(lst, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

**Example:**
```
BM25 ranking:    [D5, D2, D9, D1]
Vector ranking:  [D2, D7, D5, D3]

RRF scores (k=60):
  D5: 1/61 + 1/63 = 0.0323   ← dono mein hai, top 3 mein → jeeta
  D2: 1/62 + 1/61 = 0.0325   ← dono mein hai, top 2 mein → jeeta
  D9: 1/63         = 0.0159
  D7: 1/62         = 0.0161
  ...
Final: [D2, D5, D7, D9, D1, D3]
```

**RRF kyun acha hai:**
- Score normalization ki zaroorat nahi (scale-free)
- Jo docs **dono** methods mein aate hain, wo naturally top pe aa jaate hain
- Tuning almost zero (`k=60` universally kaam karta hai)
- Kitne bhi retrievers combine kar sakte ho

### Poora hybrid retriever

```python
from rank_bm25 import BM25Okapi

class HybridRetriever:
    def __init__(self, chunks):
        self.chunks = chunks
        self.bm25 = BM25Okapi([c.text.lower().split() for c in chunks])
        self.vector_db = build_vector_index(chunks)

    def retrieve(self, query, k=50):
        # 1. BM25
        bm25_scores = self.bm25.get_scores(query.lower().split())
        bm25_ranked = [self.chunks[i].id
                       for i in np.argsort(bm25_scores)[::-1][:k]]

        # 2. Vector
        vec_ranked = [r.id for r in self.vector_db.search(embed(query), k=k)]

        # 3. RRF merge
        fused_ids = rrf([bm25_ranked, vec_ranked])
        return [self.get(i) for i in fused_ids[:k]]
```

### Alternative: weighted sum (agar tuning karni ho)

```python
final = alpha * normalize(vector_score) + (1 - alpha) * normalize(bm25_score)
```

| `alpha` | Behaviour |
|---|---|
| 0.0 | Pure BM25 |
| 0.3 | BM25-heavy (technical docs, IDs zyada) |
| **0.5** | Balanced — acha default |
| 0.7 | Vector-heavy (conversational queries) |
| 1.0 | Pure vector |

**Kab weighted use karo:** Jab tumhe pata ho ki tumhara corpus ek side jhukta hai, aur tumhare paas tune karne ke liye eval set hai. **Warna RRF use karo** — kam tuning, robust.

---

## Part 4: Reranking ⭐⭐ (single highest-impact technique)

### Bi-encoder vs Cross-encoder — core difference

```
BI-ENCODER (embeddings, jo tum use kar rahe ho):
  query  → [Encoder] → vector_q  ┐
                                  ├→ cosine similarity
  doc    → [Encoder] → vector_d  ┘

  Docs PEHLE se embed ho sakte hain (offline) → FAST
  Par query aur doc ek dusre ko "dekh" nahi sakte!

CROSS-ENCODER (reranker):
  [query + doc EK SAATH] → [Encoder] → relevance score (0-1)

  Query aur doc ka JOINT processing → tokens ek dusre pe attend karte hain
  → Bahut ZYADA ACCURATE
  Par offline precompute impossible → SLOW
```

### Kyun cross-encoder better hai

```
Query: "2024 mein revenue kitna tha?"

Chunk A: "Revenue growth strategies mein hum focus karenge..."
  Bi-encoder:    0.82  ← "revenue" topic match
  Cross-encoder: 0.11  ← "ye sawaal ka JAWAB nahi deta"

Chunk B: "FY2024 revenue: ₹450 crore"
  Bi-encoder:    0.71  ← chhota chunk, kam similarity
  Cross-encoder: 0.96  ← "haan, ye EXACTLY sawaal ka jawab hai"
```

**Cross-encoder "answerhood" judge karta hai, "aboutness" nahi.** Yahi wo exact gap hai jo Failure Mode 3 (file 01) mein tha.

### Do-stage retrieval pattern

```
Stage 1: RETRIEVE (fast, high recall)
  Hybrid search → top 50-100 candidates
  Goal: sahi answer in 50 mein HO (recall maximize)

Stage 2: RERANK (slow, high precision)
  Cross-encoder se 50 candidates score karo → top 5 chuno
  Goal: 50 mein se BEST 5 (precision maximize)

→ LLM ko sirf top 5 do
```

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")   # multilingual!

def retrieve_and_rerank(query, retrieve_k=50, final_k=5):
    candidates = hybrid_retriever.retrieve(query, k=retrieve_k)
    pairs = [[query, c.text] for c in candidates]
    scores = reranker.predict(pairs)
    ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
    return [c for c, s in ranked[:final_k]]
```

### Typical impact (measured)

```
                        Recall@5    Answer accuracy
Vector only               0.62          58%
+ Hybrid (RRF)            0.74          71%
+ Reranking               0.89          84%   ← 🔥
```

> 🔑 **Agar tum is file se ek cheez le jao:** **Hybrid search + reranking.** Ye ~3 din ka kaam hai aur typically 20+ points deta hai. Iske bina RAG production mein ship na karo.

### Reranker options

| Option | Type | Latency | Note |
|---|---|---|---|
| `BAAI/bge-reranker-v2-m3` | Local, multilingual | ~200ms/50 docs (GPU) | **Best free option, Hindi support** |
| `BAAI/bge-reranker-base` | Local, English | ~100ms | Lighter |
| Cohere Rerank | API | ~150ms | Easy, per-call cost |
| Voyage rerank | API | ~150ms | Strong quality |
| **LLM-as-reranker** | API | ~1-2s | Best quality, mehnga, sirf top-10 pe |

### Latency budget

Reranking **200ms** add karti hai. Ye significant hai — par:
- LLM generation usually **4000ms** hai. 200ms = 5% overhead.
- Aur accuracy 13 points badhti hai.
- **Clearly worth it.**

**Agar latency critical hai:**
- Rerank sirf top-20 (top-50 nahi)
- Chhota reranker model
- Reranking ko async/parallel karo baaki kaam ke saath
- Common queries ke results cache karo

---

## Part 5: Query transformation (Failure 1 ka fix)

### (a) Query rewriting

```python
def rewrite(query, history):
    return llm(f"""User ke sawaal ko document-search ke liye rewrite karo.
Pronouns resolve karo, formal terminology use karo, jargon expand karo.
Sirf rewritten query do, kuch nahi.

History: {history}
Query: {query}""")

# "aur uski kya limit hai?"
#   → "parental leave policy ki maximum duration limit kitni hai"
```

**Sabse zaroori use case: follow-up questions.** "uski", "wo", "iske liye" — inko resolve karna mandatory hai, warna retrieval garbage hoga.

### (b) Multi-query (recall badhane ke liye)

```python
variants = llm(f"""Is query ke 3 alag-alag versions banao (alag wording,
alag specificity). Ek per line.

Query: {query}""").split("\n")

all_results = []
for v in [query] + variants:
    all_results.append([c.id for c in retrieve(v, k=20)])

final = rrf(all_results)      # RRF se merge — perfect fit
```

**Cost:** 1 LLM call + 4x retrieval. **Fayda:** Recall mein bada jump ambiguous queries pe.

### (c) HyDE (Hypothetical Document Embeddings)

Clever idea: **query ki jagah ek fake ANSWER embed karo.**

```python
def hyde_retrieve(query, k=10):
    fake_answer = llm(f"Is sawaal ka ek short paragraph answer likho "
                      f"(factual accuracy ki chinta na karo):\n{query}")
    return vector_db.search(embed(fake_answer), k=k)
```

**Kyun kaam karta hai:** Documents **answers** hote hain. Answer-to-answer similarity, question-to-answer similarity se **better** hoti hai. Ye directly query-document asymmetry (Failure 1) attack karta hai.

**Trade-off:** Ek extra LLM call (latency). Aur agar LLM ka fake answer topic se hat gaya to retrieval kharab. **Domain-specific corpora pe well kaam karta hai.**

### (d) Query decomposition (multi-hop ke liye)

```python
# "Hamare top customer ka contract kab renew hota hai?"
#   → Ek search se nahi milega! Do facts chahiye.

sub_queries = llm(f"Agar is sawaal ke liye kai facts chahiye, use "
                  f"sub-questions mein todo. Ek per line.\n{query}")
# 1. "top customer by revenue kaun hai"
# 2. "[customer] ka contract renewal date"
#     ↑ ye SEQUENTIAL hai — pehle ka answer doosre mein chahiye
```

Ye **agentic RAG** ki taraf le jaata hai (file 05).

### Comparison

| Technique | Latency add | Recall gain | Kab use karo |
|---|---|---|---|
| **Query rewriting** | +300ms | Medium | **Multi-turn chat mein MANDATORY** |
| Multi-query | +400ms, 4x retrieval | High | Ambiguous queries |
| HyDE | +500ms | Medium-High | Domain-specific corpora |
| Decomposition | +500ms, N searches | High (multi-hop) | Complex analytical queries |

---

## Part 6: Poora production retrieval pipeline

```python
def production_retrieve(query, history, user, k=5):
    # 1. Query understanding
    if history:
        query = rewrite_with_history(query, history)     # pronouns resolve

    # 2. Routing (sab kuch RAG nahi hai!)
    intent = classify_intent(query)
    if intent == "aggregation":  return sql_tool(query)
    if intent == "realtime":     return api_tool(query)

    # 3. Hybrid retrieve (wide)
    candidates = hybrid_retrieve(
        query, k=50,
        filter={                                          # 🔒 SECURITY
            "access_level": {"$in": user.roles},
            "expiry_date": {"$gte": today()},
        }
    )

    # 4. Rerank (narrow)
    reranked = cross_encoder_rerank(query, candidates, k=k)

    # 5. Quality gate
    if not reranked or reranked[0].score < 0.3:
        return NoRelevantContext()                        # "mujhe nahi pata" better hai

    # 6. Expand to parents (full context)
    return [get_parent(c) for c in reranked]
```

### Step 5 pe dhyaan do — quality gate

```python
if best_score < threshold:
    return "Mujhe available documents mein is sawaal ka jawab nahi mila."
```

**Ye counter-intuitive par bahut important hai.** Kharab chunks LLM ko dene se wo unpe hallucinate karega. **"Mujhe nahi pata" ek galat jawab se better hai** — especially enterprise mein, jahan galat jawab ka cost bahut zyada hai.

---

## Part 7: Retrieval metrics (measure karna seekho)

```python
def evaluate_retrieval(golden_set, retriever, k=5):
    hits = mrr_total = ndcg_total = 0

    for case in golden_set:
        retrieved = [c.id for c in retriever(case["query"], k=k)]
        expected = set(case["expected_chunk_ids"])

        # Recall@K — sahi chunk top-K mein hai?
        if expected & set(retrieved):
            hits += 1

        # MRR — pehla sahi result kis rank pe hai?
        for i, cid in enumerate(retrieved, 1):
            if cid in expected:
                mrr_total += 1 / i
                break

    n = len(golden_set)
    return {"recall@k": hits/n, "mrr": mrr_total/n}
```

| Metric | Kya batata hai | Target |
|---|---|---|
| **Recall@5** | Sahi chunk top-5 mein hai? | > 0.85 |
| **Recall@20** | Retrieval capability (ranking se independent) | > 0.95 |
| **MRR** | Sahi chunk kitna upar hai? | > 0.7 |
| **nDCG@5** | Ranking quality (graded relevance) | > 0.8 |
| **Precision@5** | 5 mein se kitne relevant? | > 0.6 |

### Diagnostic reading

| Recall@20 | Recall@5 | Diagnosis | Fix |
|---|---|---|---|
| High (0.95) | **Low (0.6)** | **Ranking problem** | **Reranking add karo** ⭐ |
| **Low (0.7)** | Low (0.5) | **Retrieval problem** | Chunking, hybrid, query rewriting |
| High | High | Retrieval theek | Generation dekho |

> **Ye table tumhe batata hai kaunsa fix lagana hai.** Guess ki zaroorat nahi.

---

## Khud se pooch 🤔

1. Vector search "E4471" jaisa error code kyun miss karta hai? BM25 kaise pakadta hai?
2. RRF score normalization se better kyun hai? `k=60` ka kaam kya hai?
3. Bi-encoder aur cross-encoder mein architectural farak kya hai? Cross-encoder offline precompute kyun nahi ho sakta?
4. Reranking "aboutness vs answerhood" gap ko kaise fix karta hai? Ek example do.
5. HyDE kaam kyun karta hai? Wo query-document asymmetry ko kaise attack karta hai?
6. Recall@20 = 0.95, Recall@5 = 0.60. Diagnosis? Fix?
7. Multi-turn chat mein query rewriting **mandatory** kyun hai? Ek concrete failure example do.
8. Quality gate (low score pe "nahi pata" bolna) kyun important hai?

<details>
<summary>Answers 6 aur 7</summary>

**6. Diagnosis: RANKING problem, retrieval problem nahi.**

Kyunki Recall@20 = 0.95 ka matlab hai: **sahi chunk 95% cases mein retrieve HO raha hai** — wo top-20 mein hai. Par Recall@5 = 0.60 ka matlab: **35% cases mein wo rank 6-20 ke beech hai**, top-5 mein nahi.

To retrieval (candidate generation) theek hai. Problem **ordering** mein hai.

**Fix: Reranking.** Exactly ye scenario hai jiske liye cross-encoder banaya gaya hai:
```python
candidates = hybrid_retrieve(query, k=50)         # recall high hai, use karo
top5 = cross_encoder_rerank(query, candidates, k=5)   # ordering fix karo
```
Expected: Recall@5 0.60 → ~0.88.

**Jo NA karo:** Chunking strategy badalna, embedding model badalna, `top_k` badhana. Wo sab **retrieval** fix hain — par retrieval already theek hai. Time waste hoga.

**7. Mandatory hai kyunki follow-up queries standalone meaningless hoti hain.**

```
Turn 1: "Parental leave policy kya hai?"
        → retrieval: "parental leave policy" ✅ perfect

Turn 2: "aur uske liye notice kitna dena padta hai?"
        → retrieval query: "aur uske liye notice kitna dena padta hai"
        → Embedding "notice period" ke saath weakly match karega
        → "uske" ka koi meaning nahi — retrieval mein "parental leave" ka
          zikr hi nahi hai!
        → Random notice-related chunks aayenge (resignation notice,
          termination notice, meeting notice) 💥
```

**Rewriting ke saath:**
```
Turn 2 rewritten: "parental leave ke liye kitne din ka advance notice
                   dena hota hai"
        → ✅ perfect retrieval
```

**Implementation note:** Rewriting ke liye last 2-3 turns kaafi hain, poori history nahi. Aur ek chhota fast model use karo — ye reasoning task nahi hai, ek reference-resolution task hai. Latency ~200ms, aur ye multi-turn RAG mein **sabse bada single fix** hai.
</details>

---

**Next:** [`04-advanced-rag-patterns.md`](04-advanced-rag-patterns.md)
