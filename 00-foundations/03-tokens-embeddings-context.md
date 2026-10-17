# 03 — Tokens, Embeddings aur Context Window

> **Is file ka goal:** Model "text" nahi dekhta — numbers dekhta hai. Ye conversion samajhna zaroori hai, kyunki tumhari **poori cost, latency aur kai weird bugs** isi layer se aate hain.
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/03-context-window-aur-kv-cache`](../08-diagrams/03-context-window-aur-kv-cache.excalidraw)

---

## Part 1: Tokens — model ki actual bhasha

### Token kya hai?

Token = text ka ek chunk. **Na exactly shabd, na exactly letter** — beech ka kuch.

```
"Hello world"           → ["Hello", " world"]                    (2 tokens)
"unbelievable"          → ["un", "bel", "iev", "able"]            (4 tokens)
"ChatGPT"               → ["Chat", "GPT"]                         (2 tokens)
"नमस्ते"                  → ["न", "म", "स", "्", "त", "े"]              (6+ tokens!)
"1234567"               → ["123", "45", "67"]                     (3 tokens)
"    "  (4 spaces)      → ["   ", " "]                            (2 tokens)
```

### Tokenizer kaise banta hai (BPE)

Modern models **BPE (Byte Pair Encoding)** ya uska variant use karte hain. Algorithm dumb-simple hai:

```
1. Shuru: har character ek token hai
2. Poore training corpus mein sabse frequent PAIR dhundo
3. Us pair ko merge karke ek naya token banao
4. Step 2-3 repeat karo, ~100,000 baar
```

Result: **common cheezein ek token, rare cheezein kai tokens.**
"the" ek token hai (bahut common). "Chandrashekhar" 5 tokens hai (rare).

### 🔥 Practical consequences (ye actually matter karte hain)

#### (a) Hindi/regional languages 2-4x mehngi hain

English-heavy training data ki wajah se:
```
"I am going home"          → ~4 tokens
"main ghar jaa raha hoon"  → ~7 tokens
"मैं घर जा रहा हूँ"            → ~15 tokens
```

**Same meaning, 4x cost.** Agar tumhara product Indian languages handle karta hai, ye tumhare budget planning mein aana chahiye.

#### (b) Model spelling mein weak hota hai

Classic example: *"strawberry mein kitne 'r' hain?"* Models yahan galti karte the.

**Kyun?** Model "strawberry" ko `["str", "aw", "berry"]` dekhta hai — individual letters nahi. Letters count karna uske liye ek indirect inference hai, direct observation nahi.

> Isliye character-level tasks (reverse karo, letters count karo, anagram) mein LLM struggle karta hai. Aisa kaam **code se karao**, model se nahi.

#### (c) Numbers unreliable hain

`1234567` → `["123", "45", "67"]`. Model ke liye ye ek "number" nahi, 3 arbitrary chunks hain. Isliye:
- Bade numbers ka arithmetic galat hota hai
- **Solution:** calculator tool do, ya code execution do. Model se math na karao.

#### (d) Rough conversion (yaad rakho)

| Language | Ratio |
|---|---|
| English | 1 token ≈ 4 characters ≈ 0.75 shabd |
| Code | 1 token ≈ 3 characters (symbols zyada) |
| Hindi (Devanagari) | 1 token ≈ 1-2 characters |

**Quick estimate:** English words × 1.3 ≈ tokens. Ek A4 page ≈ 500-700 tokens. Ek 300-page book ≈ 150K tokens.

> ⚠️ **Kabhi bhi `tiktoken` se Claude ke tokens count na karo** — wo OpenAI ka tokenizer hai, 15-20% galat count deta hai. Har provider ka apna token counting endpoint hota hai (Anthropic: `messages.count_tokens`), wahi use karo.

---

## Part 2: Embeddings — meaning ka geometry

### Token ID se embedding tak

```
"ghar"  →  token ID: 8921  →  embedding: [0.23, -0.81, 0.44, ..., 0.12]
                                          (4096 numbers ka vector)
```

Ye vector **seekha gaya** hai. Training ke dauraan model ne khud discover kiya ki kaunse shabd kahan hone chahiye.

### Magic: semantic geometry

Similar meaning wale shabd vector space mein **paas** hote hain:

```
        ↑ (royalty dimension, roughly)
        │
   raja │●        ●rani
        │
        │
  aadmi │●        ●aurat
        └──────────────────→ (gender dimension, roughly)
```

Aur famous arithmetic:
```
vector("raja") - vector("aadmi") + vector("aurat") ≈ vector("rani")
```

Ye **koi programmer ne code nahi kiya**. Model ne text se khud seekha ki "gender" ek direction hai vector space mein.

### Do tarah ke embeddings — inhe confuse na karo!

Ye distinction bahut log miss karte hain:

| | **Static / Input embeddings** | **Contextual embeddings** |
|---|---|---|
| Kya | Ek token = ek fixed vector | Layers se guzarne ke baad ka representation |
| "bank" | Ek hi vector, chahe river ho ya money | River ke context mein alag, money mein alag |
| Kahan | Model ka input layer (lookup table) | Transformer blocks ka output |
| RAG mein | ❌ Weak (context nahi) | ✅ Yahi use hota hai |

**RAG mein tum "embedding model" use karte ho** (jaise `text-embedding-3`, `voyage-3`, `bge-m3`, `all-MiniLM`). Wo ek chhota transformer hai jo **poore text ka ek single contextual vector** banata hai.

```
"Delhi mein baarish hui"  →  [Embedding model]  →  [0.1, 0.8, ..., 0.3]
                                                   (ek 1024-dim vector)
```

### Similarity kaise measure hoti hai

**Cosine similarity** — do vectors ke beech ka angle:

```
cos_sim(A, B) = (A · B) / (|A| × |B|)

 1.0  = exactly same direction  (identical meaning)
 0.0  = perpendicular            (unrelated)
-1.0  = opposite direction       (rare in practice)
```

**Practical note:** Real embedding models mein "unrelated" text bhi 0.6-0.7 similarity de deta hai. Isliye **absolute threshold** (jaise "0.8 se zyada hi lo") fragile hai. **Relative ranking** (top-K lo) zyada reliable hai.

### ⚠️ Embeddings ki 4 limitations (RAG debug karte waqt yaad rakho)

1. **Negation blind:** "Delhi mein baarish hui" aur "Delhi mein baarish nahi hui" ka similarity ~0.95 hoga. Embeddings "nahi" ko barely register karte hain.
2. **Exact keywords miss:** Product code `XR-4471B` dhundhna hai? Embedding ise semantic dhundhega, exact nahi. Isliye **BM25/keyword search** chahiye. (Ye hybrid search ka poora justification hai.)
3. **Fixed budget:** Chahe 10 shabd ho ya 500, output same size vector. 500 shabd ki nuance squeeze ho jaati hai — isliye chunking matters.
4. **Domain mismatch:** General-purpose embedding model tumhare medical/legal/internal jargon ko theek se nahi samajhta. Yahan fine-tuned embeddings ya domain models help karte hain.

> 🔑 **Ye 4 limitations 80% RAG failures explain karti hain.** Day 3 mein detail mein aayenge.

---

## Part 3: Context Window — sabse misunderstood concept

### Context window mein kya hota hai?

**Sab kuch:**
```
┌───────────────────────────────────────┐
│ System prompt          (500 tokens)   │
│ Tool definitions       (2,000)        │
│ RAG retrieved chunks   (4,000)        │
│ Conversation history   (8,000)        │
│ Current user message   (200)          │
│ ─────────────────────────────────     │
│ Model ka output        (2,000)        │  ← ye bhi context mein counts!
└───────────────────────────────────────┘
Total: 16,700 tokens
```

**Critical:** Output bhi window mein hi hai. Agar window 200K hai aur input 199K hai, tumhare paas output ke liye 1K bacha hai.

### Bade context ka jhoot 🚩

Log sochte hain: *"1M context window hai, poori database daal do!"*

**Teen problems:**

#### 1. Paisa
Input tokens bhi charge hote hain. Har turn pe. Example ke liye ek $5/M input rate lo:
```
1M token context, 100 conversations/day = 100M input tokens/day
100M × $5/M = $500/day = $15,000/month
...ek hi user ke liye. 🔥
```

#### 2. Quality girti hai
Ye counter-intuitive hai par proven hai. "Needle in a haystack" tests mein: jitna zyada irrelevant context, utna performance drop — especially **beech ka** content.

```
Context bharo:      10%    50%    90%
Accuracy:           95%    89%    76%   (roughly, task-dependent)
```

**Kyun?** Attention weights spread ho jaate hain. 100 relevant tokens ke saath 100K irrelevant tokens hain — signal-to-noise ratio gir gaya.

#### 3. Latency
Bada input = **prefill** phase lamba. Time-to-first-token seedha context size se scale karta hai.

### 🔑 Golden rule
> **Context window ek budget hai, storage nahi.**
>
> Sawaal ye nahi: "kitna fit ho sakta hai?"
> Sawaal ye hai: "**minimum kitna chahiye** is task ke liye?"

Ye mindset shift poore repo ka central theme hai — isse **"context engineering"** kehte hain (Day 2 mein detail).

---

## Part 4: Context management strategies

Jab context bharne lage, 5 options hain:

| Strategy | Kaise | Kab use karein | Trade-off |
|---|---|---|---|
| **Truncation** | Purane messages kaat do | Simple chat | Information poori kho gayi |
| **Sliding window** | Last N turns rakho + system prompt | Short-memory chat | Purani baat bhool jaate ho |
| **Summarization / Compaction** | Purane part ka summary banao, replace kar do | Long conversations, agents | Detail loss, ek extra LLM call |
| **RAG over history** | Poori history store karo, relevant part retrieve karo | Long-term memory | Complexity, retrieval quality pe dependent |
| **Context editing / pruning** | Purane tool results delete kar do (structure rakho) | Tool-heavy agents | Model wo detail dubara nahi dekh sakta |

**Modern practice (agents ke liye):** Ye combine hote hain —
- Tool results ka **pruning** (purane tool outputs hata do)
- Milestone pe **compaction** (summary bana do)
- Cross-session ke liye **external memory** (file/DB)

> Note: Kai providers ab ye server-side offer karte hain (Anthropic ki "compaction" aur "context editing" features). Par concept samajhna zaroori hai — feature ka naam nahi.

---

## Part 5: Practical cheat sheet 📋

### Token budget planning

```
System prompt        : 200 - 2,000      (fixed, cacheable)
Tool definitions     : 100 - 500 / tool (fixed, cacheable)
RAG chunks           : 2,000 - 8,000    (top-5 se top-20 chunks)
Conversation history : variable         (yahan control chahiye)
Output reserve       : 1,000 - 8,000    (reasoning models mein zyada)
```

### Cost optimization checklist

- [ ] **Prompt caching on** — stable prefix (system + tools) cache karo. 90% tak saving.
- [ ] **Stable content pehle, variable baad mein** — cache hit rate ke liye critical
- [ ] System prompt mein **koi timestamp/UUID nahi** — cache silently toot jaayega
- [ ] RAG mein `top_k` tune karo — 20 chunks se 5 chunks pe jaane se often accuracy *badhti* hai
- [ ] Simple tasks ko **chhote/sasta model** pe route karo
- [ ] History pe **hard token cap** lagao, phir compact karo
- [ ] Batch API use karo jahan real-time zaroori nahi (~50% sasta)

### Debug: "Mera model context bharne pe bewakoof ho gaya"

Symptoms → likely cause:

| Symptom | Cause | Fix |
|---|---|---|
| Beech ki information ignore kar raha hai | Lost in the middle | Important stuff top/bottom pe |
| Purani instructions bhool raha hai | System prompt bahut door hai | Instructions ko end pe repeat karo |
| Irrelevant chunks pe base kar raha hai | Poor retrieval, zyada `top_k` | Reranking + `top_k` kam karo |
| Output cut ho raha hai | `max_tokens` kam hai | Badhao, ya streaming use karo |
| Cost phat gaya | Cache miss | Cache hit metrics check karo |

---

## Khud se pooch 🤔

1. Hindi text English se 3x zyada tokens kyun leti hai? Iska tumhare product pe kya asar?
2. "strawberry mein kitne r" wala problem tokenization se kaise explain hota hai?
3. Static aur contextual embeddings mein farak? RAG mein kaunsa use hota hai?
4. Embeddings ki 4 limitations batao. Kaunsi limitation **hybrid search** ko justify karti hai?
5. Bada context window "sab daal do" ka solution kyun nahi hai? 3 reasons.
6. Tumhare paas 200K context window hai aur ek 500-page manual (250K tokens). Kya karoge?
7. Tumhare system prompt mein `f"Aaj ki date: {datetime.now()}"` hai. Kya problem hai?

<details>
<summary>Answer 6 aur 7</summary>

**6.** Poora manual daalna possible bhi nahi (250K > 200K), aur hota to bhi galat approach hota — mehnga aur quality-degrading.
**Sahi approach:** RAG. Manual ko chunk karo, embed karo, index karo. Query aane pe top 5-10 relevant chunks retrieve karo (~4K tokens). Cost 60x kam, accuracy zyada (kam noise), latency kam.
**Bonus:** Citations bhi de paoge — "ye answer page 47 se hai".

**7.** Ye **cache killer** hai. `datetime.now()` har request pe badalta hai. System prompt prefix ka pehla hissa hai — wo badla to **poora cache invalid**. Tum full price pay karoge, aur koi error nahi aayega — silently paisa jal raha hoga.
**Fix:** Date ko **end** mein daalo (last cache breakpoint ke baad), user message ke saath. Ya agar granularity chahiye to sirf date rakho (`2026-08-12`), time nahi — phir din bhar cache valid rahega.
</details>

---

**Next:** [`04-training-pipeline.md`](04-training-pipeline.md) — model "helpful" kaise banta hai.
