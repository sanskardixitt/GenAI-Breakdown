# Lab 1 — Tokens, Embeddings, Similarity

> **Day 1 ke baad karo.** Time: 45 min.
> **Goal:** Tokens aur embeddings ko *dekhna*, sirf padhna nahi.

---

## Part A: Token counting (15 min)

```python
# lab1a_tokens.py
from common import client, MODEL

def count(text: str) -> int:
    """Provider ka apna token counter — tiktoken NAHI."""
    r = client.messages.count_tokens(
        model=MODEL,
        messages=[{"role": "user", "content": text}],
    )
    return r.input_tokens

SAMPLES = {
    "English short":  "The quick brown fox jumps over the lazy dog.",
    "Hinglish":       "Main ghar jaa raha hoon aur phir office jaunga.",
    "Hindi":          "मैं घर जा रहा हूँ और फिर ऑफिस जाऊँगा।",
    "Code":           "def hello(name: str) -> str:\n    return f'Hi {name}'",
    "Numbers":        "1234567890 9876543210 3.14159265358979",
    "Repeated":       "the the the the the the the the the the",
    "Rare words":     "Chandrashekhar Venkataraman Subrahmanyan",
    "JSON":           '{"name": "Sanskar", "role": "AI Engineer", "city": "Delhi"}',
}

print(f"{'Sample':<16} {'Chars':>6} {'Tokens':>7} {'Chars/Token':>12}")
print("-" * 45)
for name, text in SAMPLES.items():
    t = count(text)
    print(f"{name:<16} {len(text):>6} {t:>7} {len(text)/t:>12.2f}")
```

### Kya observe karo

Run karke ye dekho aur **likho**:

1. **Hindi vs English** — same meaning ke liye kitne guna tokens? (Expected: 3-4x)
2. **Chars per token** — English mein ~4, Hindi mein ~1-2, code mein ~3
3. **Repeated words** — "the" 10 baar = 10 tokens? Ya kam? (Common word = 1 token each)
4. **Rare names** — kitne tokens mein tut rahe hain?

### 💰 Cost implication nikalo

```python
# Ek chat app: 1000 messages/day, avg 200 words per message
english_tokens = count("word " * 200)
hindi_tokens   = count("शब्द " * 200)

print(f"English: {english_tokens} tokens/message")
print(f"Hindi:   {hindi_tokens} tokens/message")
print(f"Hindi is {hindi_tokens/english_tokens:.1f}x costlier")

# Monthly (input tokens only, $3/M rate example ke liye)
for lang, tok in [("English", english_tokens), ("Hindi", hindi_tokens)]:
    monthly = 1000 * 30 * tok
    print(f"{lang}: {monthly:,} tokens/month = ${monthly/1e6*3:.2f}")
```

> **Ye number yaad rakho.** Agar tumhara product Indian languages support karta hai, tumhara budget planning isse affect hoga.

---

## Part B: Embeddings aur similarity (20 min)

```python
# lab1b_embeddings.py
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")     # chhota, laptop pe chalega

def sim(a: str, b: str) -> float:
    ea, eb = model.encode([a, b])
    return float(np.dot(ea, eb) / (np.linalg.norm(ea) * np.linalg.norm(eb)))

PAIRS = [
    # (text A, text B, expected)
    ("Delhi mein baarish hui",     "Delhi mein baarish hui",       "identical"),
    ("Delhi mein baarish hui",     "Delhi mein barish ho rahi hai","paraphrase"),
    ("Delhi mein baarish hui",     "Delhi mein baarish NAHI hui",  "NEGATION ⚠️"),
    ("Delhi mein baarish hui",     "Mumbai mein dhoop thi",        "different"),
    ("How much parental leave?",   "24 days annual paid leave",    "q→a"),
    ("error code E4471",           "E4471: connection timeout",    "exact ID"),
    ("error code E4471",           "error handling best practices","topical only"),
    ("Product XR-4471B ka price",  "XR-4471B costs ₹5000",         "product code"),
    ("machine learning",           "deep learning",                 "related"),
    ("machine learning",           "washing machine",               "word overlap ⚠️"),
]

print(f"{'Similarity':>10}  {'Type':<16} Pair")
print("-" * 80)
for a, b, label in PAIRS:
    s = sim(a, b)
    bar = "█" * int(s * 20)
    print(f"{s:>10.3f}  {label:<16} {bar}")
    print(f"{'':>10}  {'':<16} A: {a}")
    print(f"{'':>10}  {'':<16} B: {b}\n")
```

### 🔑 Kya observe karo (ye Day 3 ke liye critical hai)

| Observation | Kya seekha |
|---|---|
| **Negation pair ~0.90+** | Embeddings "nahi" ko barely dekhti hain → **hybrid search kyun chahiye** |
| **Error code pair kam** | Exact IDs semantic search se nahi milte → **BM25 kyun chahiye** |
| **"machine learning" vs "washing machine"** | Word overlap se false similarity |
| **"Different" pair ~0.3-0.5** | Unrelated bhi 0 nahi hota → **absolute threshold fragile hai** |

> **Likho:** Negation pair ka score kya aaya? Agar wo 0.85+ hai, tumne abhi khud prove kar diya ki pure vector search RAG mein kyun fail hota hai.

---

## Part C: Query-document asymmetry (10 min)

```python
# lab1c_asymmetry.py
QUERY = "parental leave ke liye kitna notice dena padta hai?"

DOCS = [
    "Employees must provide a minimum of 60 days advance written notice "
    "prior to commencement of parental leave, as stipulated in Section 4.2.",

    "Parental leave ke liye 60 din pehle notice dena hota hai.",

    "Leave policies are governed by the HR handbook. Various leave types "
    "are available to employees including annual, sick, and parental leave.",
]

print(f"Query: {QUERY}\n")
for i, d in enumerate(DOCS, 1):
    print(f"Doc {i} similarity: {sim(QUERY, d):.3f}")
    print(f"  {d[:80]}...\n")
```

### Observe karo

- **Doc 2** (same language, direct answer) ka score sabse zyada hoga
- **Doc 1** (English formal, sahi answer!) ka score kam ho sakta hai
- **Doc 3** (topical, koi answer nahi) ka score decent ho sakta hai 😬

**Yahi Failure Mode 1 aur 3 hai** (Day 3, file 01) — live dekh liya.

### HyDE try karo

```python
from common import client, MODEL, text_of

fake_answer = text_of(client.messages.create(
    model=MODEL, max_tokens=150,
    messages=[{"role": "user", "content":
        f"Is sawaal ka ek short paragraph answer likho (factual accuracy "
        f"ki chinta na karo, bas typical answer likho):\n{QUERY}"}],
))

print(f"HyDE fake answer:\n{fake_answer}\n")
for i, d in enumerate(DOCS, 1):
    print(f"Doc {i}: query_sim={sim(QUERY,d):.3f}  hyde_sim={sim(fake_answer,d):.3f}")
```

**Observe:** HyDE se Doc 1 aur Doc 2 ka score badhna chahiye (answer-to-answer matching). Doc 3 ka relatively kam.

---

## ✅ Lab 1 checklist

Ye sab observe kar liya?

- [ ] Hindi English se 3-4x zyada tokens leti hai
- [ ] Negation pair ka similarity 0.85+ hai (embeddings negation-blind hain)
- [ ] Error codes semantic search se nahi milte
- [ ] Unrelated text bhi 0.3-0.5 similarity deta hai
- [ ] Query-document asymmetry live dekha
- [ ] HyDE se scores badle

---

## 🚀 Aage badho

1. **Apne domain ka data test karo** — apne actual documents se 5 pairs banao
2. **Multilingual model try karo** — `BAAI/bge-m3` load karo, Hindi pairs pe compare karo
3. **Prefix ka asar dekho** — `bge` models mein `"query: "` aur `"passage: "` prefix add karke difference measure karo
4. **Dimension ka asar** — 384-dim vs 1024-dim model compare karo

---

**Next:** [`lab-02-structured-output-tools.md`](lab-02-structured-output-tools.md)
