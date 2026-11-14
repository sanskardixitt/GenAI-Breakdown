# Lab 3 — Hybrid RAG + Reranking ⭐

> **Day 3 ke baad karo.** Time: 90 min.
> **Goal:** Vector-only se hybrid+rerank tak — aur **measure karke** dekhna ki kitna farak padta hai.

---

## Setup: sample corpus

```python
# corpus.py
DOCS = [
    # HR
    ("hr_01", "hr", "Parental Leave", "Employees are entitled to 26 weeks of paid parental leave. A minimum of 60 days advance written notice must be submitted to HR before commencement."),
    ("hr_02", "hr", "Annual Leave", "All full-time employees receive 24 days of annual paid leave per calendar year. Unused leave may be carried forward up to 10 days."),
    ("hr_03", "hr", "Sick Leave", "Employees may take up to 12 days of sick leave annually. Medical certificate required for absences exceeding 3 consecutive days."),
    ("hr_04", "hr", "Work From Home", "Employees may work remotely up to 2 days per week with manager approval. Full remote requires VP approval."),

    # Engineering
    ("eng_01", "engineering", "Deployment Process", "Production deployments require approval from two senior engineers. All deployments must occur between 10 AM and 4 PM IST on weekdays."),
    ("eng_02", "engineering", "Error Codes", "E4471: Database connection timeout. Resolution: increase connection pool size in config.yaml. E4472: Redis unavailable. E4473: Rate limit exceeded."),
    ("eng_03", "engineering", "Code Review", "All pull requests require at least one approval. PRs touching payment code require two approvals including a security team member."),
    ("eng_04", "engineering", "On-Call", "On-call rotation is weekly. Primary responder must acknowledge alerts within 15 minutes during business hours, 30 minutes otherwise."),

    # Finance
    ("fin_01", "finance", "Expense Policy", "Expenses under Rs 5000 require manager approval. Above Rs 5000 requires finance team approval. Receipts must be submitted within 30 days."),
    ("fin_02", "finance", "Travel Policy", "Domestic flights in economy class only. International travel requires VP approval. Per diem is Rs 2000 for domestic, Rs 5000 for international."),

    # Legal
    ("leg_01", "legal", "NDA Policy", "All contractors must sign an NDA before repository access. NDAs remain in effect for 3 years after contract termination."),
    ("leg_02", "legal", "Data Retention", "Customer data is retained for 7 years. Deletion requests must be processed within 30 days as per DPDP Act requirements."),
]

# Golden eval set — YE SABSE IMPORTANT HISSA HAI
GOLDEN = [
    {"q": "parental leave ke liye kitna notice dena padta hai?",
     "expected": ["hr_01"], "must_contain": ["60"]},
    {"q": "How many days of annual leave do I get?",
     "expected": ["hr_02"], "must_contain": ["24"]},
    {"q": "error code E4471 ka fix kya hai?",
     "expected": ["eng_02"], "must_contain": ["connection pool"]},
    {"q": "kya main ghar se kaam kar sakta hoon?",
     "expected": ["hr_04"], "must_contain": ["2 days"]},
    {"q": "production deploy karne ke liye kitne approvals chahiye?",
     "expected": ["eng_01"], "must_contain": ["two", "2"]},
    {"q": "Rs 8000 ka expense kaise approve hoga?",
     "expected": ["fin_01"], "must_contain": ["finance"]},
    {"q": "sick leave ke liye medical certificate kab chahiye?",
     "expected": ["hr_03"], "must_contain": ["3"]},
    {"q": "payment code ke PR ko kitne approvals chahiye?",
     "expected": ["eng_03"], "must_contain": ["two", "security"]},
    {"q": "international travel ka per diem kitna hai?",
     "expected": ["fin_02"], "must_contain": ["5000"]},
    {"q": "customer data kitne saal rakha jaata hai?",
     "expected": ["leg_02"], "must_contain": ["7"]},
    {"q": "on-call alert kitne minute mein acknowledge karna hai?",
     "expected": ["eng_04"], "must_contain": ["15"]},
    {"q": "NDA kitne saal valid rehta hai?",
     "expected": ["leg_01"], "must_contain": ["3 years", "3"]},
]
```

---

## Part A: Vector-only baseline (20 min)

```python
# lab3a_vector.py
import numpy as np
from sentence_transformers import SentenceTransformer
from corpus import DOCS, GOLDEN

model = SentenceTransformer("all-MiniLM-L6-v2")

# Chunk enrichment — header path prepend (Day 3 ka lesson)
def enrich(doc_id, category, title, text):
    return f"[{category.upper()} | {title}]\n{text}"

chunks = [{"id": d[0], "category": d[1], "title": d[2], "text": d[3],
           "enriched": enrich(*d)} for d in DOCS]

embeddings = model.encode([c["enriched"] for c in chunks], normalize_embeddings=True)

def vector_search(query, k=5):
    q = model.encode([query], normalize_embeddings=True)[0]
    scores = embeddings @ q                       # normalized → dot = cosine
    idx = np.argsort(scores)[::-1][:k]
    return [(chunks[i], float(scores[i])) for i in idx]

# ── EVALUATE ──
def evaluate(search_fn, k=5, name="baseline"):
    hits, mrr = 0, 0.0
    misses = []
    for case in GOLDEN:
        got = [c["id"] for c, _ in search_fn(case["q"], k=k)]
        exp = set(case["expected"])
        if exp & set(got):
            hits += 1
            for i, cid in enumerate(got, 1):
                if cid in exp:
                    mrr += 1/i
                    break
        else:
            misses.append((case["q"], case["expected"], got))

    n = len(GOLDEN)
    print(f"\n{'='*60}\n{name}")
    print(f"  Recall@{k}: {hits/n:.3f}  ({hits}/{n})")
    print(f"  MRR:       {mrr/n:.3f}")
    if misses:
        print(f"\n  ❌ MISSES ({len(misses)}):")
        for q, exp, got in misses:
            print(f"     Q: {q}")
            print(f"        expected {exp}, got {got[:3]}")
    return hits/n, mrr/n

evaluate(vector_search, k=5, name="VECTOR ONLY")
```

**Expected output:** Recall@5 ~0.75-0.85. Note karo ki **kaunse queries miss hue** — usually "E4471" wala.

---

## Part B: BM25 add karo (15 min)

```python
# lab3b_bm25.py
from rank_bm25 import BM25Okapi
import re

def tokenize(text):
    return re.findall(r'\w+', text.lower())

bm25 = BM25Okapi([tokenize(c["enriched"]) for c in chunks])

def bm25_search(query, k=5):
    scores = bm25.get_scores(tokenize(query))
    idx = np.argsort(scores)[::-1][:k]
    return [(chunks[i], float(scores[i])) for i in idx]

evaluate(bm25_search, k=5, name="BM25 ONLY")
```

**Observe:** BM25 vector se **alag** queries pe jeetega — especially "E4471". Aur **alag** queries pe haarega (paraphrase wale).

**Yahi complementarity hai.** Ab combine karte hain.

---

## Part C: Hybrid with RRF (20 min) ⭐

```python
# lab3c_hybrid.py
def rrf(ranked_lists, k_const=60):
    """Reciprocal Rank Fusion — scores nahi, RANKS use karo."""
    scores = {}
    for lst in ranked_lists:
        for rank, doc_id in enumerate(lst, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k_const + rank)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)

by_id = {c["id"]: c for c in chunks}

def hybrid_search(query, k=5, retrieve_k=10):
    vec_ids  = [c["id"] for c, _ in vector_search(query, k=retrieve_k)]
    bm25_ids = [c["id"] for c, _ in bm25_search(query, k=retrieve_k)]
    fused = rrf([vec_ids, bm25_ids])
    return [(by_id[cid], score) for cid, score in fused[:k]]

evaluate(hybrid_search, k=5, name="HYBRID (RRF)")
```

**Expected:** Recall@5 badhna chahiye. Aur — critically — **E4471 wala query ab pass hona chahiye.**

### 🧪 Experiment
```python
# Alag k_const try karo
for kc in [10, 30, 60, 100]:
    def h(q, k=5): 
        v = [c["id"] for c,_ in vector_search(q, 10)]
        b = [c["id"] for c,_ in bm25_search(q, 10)]
        return [(by_id[i], s) for i,s in rrf([v,b], kc)[:k]]
    print(f"\nk_const={kc}")
    evaluate(h, name=f"RRF k={kc}")
```
**Observe:** `k=60` usually robust hai. Ye ek reason hai ki wo standard bana.

---

## Part D: Reranking (25 min) ⭐⭐

```python
# lab3d_rerank.py
from sentence_transformers import CrossEncoder

# Pehli baar download hoga (~500MB)
reranker = CrossEncoder("BAAI/bge-reranker-base")     # ya bge-reranker-v2-m3 (multilingual)

def hybrid_rerank_search(query, k=5, retrieve_k=10):
    # Stage 1: WIDE retrieve (recall maximize)
    candidates = hybrid_search(query, k=retrieve_k, retrieve_k=retrieve_k)

    # Stage 2: RERANK (precision maximize)
    pairs = [[query, c["enriched"]] for c, _ in candidates]
    scores = reranker.predict(pairs)

    ranked = sorted(zip([c for c,_ in candidates], scores),
                    key=lambda x: x[1], reverse=True)
    return [(c, float(s)) for c, s in ranked[:k]]

evaluate(hybrid_rerank_search, k=5, name="HYBRID + RERANK")
```

### 🔍 Bi-encoder vs cross-encoder ko live dekho

```python
QUERY = "parental leave ke liye kitna notice dena padta hai?"

print(f"Query: {QUERY}\n")
print(f"{'Doc':<10} {'Bi-encoder':>12} {'Cross-encoder':>15}  Title")
print("-" * 65)

cands = hybrid_search(QUERY, k=6, retrieve_k=6)
pairs = [[QUERY, c["enriched"]] for c, _ in cands]
cross_scores = reranker.predict(pairs)

for (c, bi_score), cross in zip(cands, cross_scores):
    print(f"{c['id']:<10} {bi_score:>12.4f} {cross:>15.4f}  {c['title']}")
```

**Observe:** Cross-encoder scores mein **bahut zyada separation** hoga. Relevant doc ka score bahut upar, irrelevant ka bahut neeche. Bi-encoder scores compressed range mein hote hain.

**Yahi "aboutness vs answerhood" ka farak hai** — live dikh gaya.

---

## Part E: End-to-end RAG (10 min)

```python
# lab3e_rag.py
from common import client, MODEL, text_of

RAG_SYSTEM = """Tum ek document-grounded assistant ho.

RULES:
1. SIRF diye gaye documents se answer do. General knowledge use na karo.
2. Har claim ke saath source cite karo: [Doc 1], [Doc 2]
3. Agar documents mein answer nahi hai, bolo: "Diye gaye documents mein
   is sawaal ka jawab nahi mila." Guess na karo.
4. Concise raho — 2-3 sentences.
"""

def rag(query, k=5, min_score=0.0):
    results = hybrid_rerank_search(query, k=k)

    # Quality gate
    if not results or results[0][1] < min_score:
        return "Relevant information nahi mili.", []

    docs = "\n\n".join(
        f"[Doc {i+1}] (source: {c['id']}, section: {c['category']}/{c['title']})\n{c['text']}"
        for i, (c, _) in enumerate(results)
    )

    r = client.messages.create(
        model=MODEL, max_tokens=1024,
        system=RAG_SYSTEM,
        messages=[{"role": "user",
                   "content": f"<documents>\n{docs}\n</documents>\n\nSawaal: {query}"}],
    )
    return text_of(r), [c["id"] for c, _ in results]

# Test
for case in GOLDEN[:5]:
    answer, sources = rag(case["q"])
    ok = all(str(m).lower() in answer.lower() for m in case["must_contain"][:1])
    print(f"\n{'✅' if ok else '❌'} Q: {case['q']}")
    print(f"   A: {answer}")
    print(f"   Sources: {sources[:3]}")
```

---

## Part F: Final comparison table

```python
# lab3f_compare.py
print("\n" + "="*70)
print("FINAL COMPARISON")
print("="*70)

results = {}
for name, fn in [
    ("Vector only",        vector_search),
    ("BM25 only",          bm25_search),
    ("Hybrid (RRF)",       hybrid_search),
    ("Hybrid + Rerank",    hybrid_rerank_search),
]:
    recall, mrr = evaluate(fn, k=5, name=name)
    results[name] = (recall, mrr)

print(f"\n{'Method':<22} {'Recall@5':>10} {'MRR':>8}")
print("-"*42)
for name, (r, m) in results.items():
    print(f"{name:<22} {r:>10.3f} {m:>8.3f}")
```

**Expected shape (exact numbers vary karenge):**
```
Method                  Recall@5      MRR
------------------------------------------
Vector only                0.750    0.681
BM25 only                  0.667    0.583
Hybrid (RRF)               0.917    0.812
Hybrid + Rerank            0.917    0.903   ← MRR mein bada jump
```

> 🔑 **Chhote corpus mein Recall@5 saturate ho jaata hai. Real corpus (10,000+ chunks) mein reranking ka fayda Recall@5 mein bhi dikhta hai.** MRR yahan better signal hai — wo dikhata hai ki sahi doc kitna **upar** aaya.

---

## ✅ Lab 3 checklist

- [ ] Vector-only baseline measure kiya
- [ ] BM25 ne alag queries jeeti (E4471!)
- [ ] RRF se dono combine kiye — recall badha
- [ ] Reranking se MRR mein bada jump dekha
- [ ] Bi-encoder vs cross-encoder scores ka farak dekha
- [ ] End-to-end RAG chalaya citations ke saath
- [ ] Comparison table banayi

---

## 🚀 Aage badho

1. **Corpus badhao** — 100+ chunks add karo, dekho ki reranking ka fayda badhta hai
2. **Query rewriting add karo** — multi-turn queries test karo ("aur uske liye?")
3. **Metadata filtering** — category filter add karo, ACL simulate karo
4. **Chunk size experiment** — DOCS ko chhote chunks mein todo, measure karo
5. **HyDE add karo** — Lab 1 ka HyDE code use karke compare karo
6. **Failure analysis** — jo queries ab bhi fail ho rahi hain, unhe manually debug karo

---

**Next:** [`lab-04-agent-loop.md`](lab-04-agent-loop.md) — sabse important lab
