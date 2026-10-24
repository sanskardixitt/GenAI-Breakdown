# 02 — Chunking aur Indexing

> **Is file ka goal:** Chunking RAG ka sabse under-appreciated part hai. Yahan ki galti aage har stage mein multiply hoti hai.

---

## Part 1: Chunking kyun karte hain? (4 reasons)

| Reason | Detail |
|---|---|
| **Embedding limit** | Embedding models ki max length hoti hai (512-8192 tokens) |
| **Precision** | 50-page doc ka ek vector = meaning dhul gaya. Chhota chunk = focused meaning |
| **Context budget** | Poora doc LLM ko nahi bhej sakte |
| **Citations** | "Page 12, para 3" batane ke liye granularity chahiye |

### Fundamental trade-off

```
CHHOTE CHUNKS (100-200 tokens)          BADE CHUNKS (1000-2000 tokens)
────────────────────────────           ─────────────────────────────
✅ Precise embeddings                   ✅ Poora context ek jagah
✅ Kam noise LLM ko                     ✅ Answer katta nahi
✅ Zyada chunks fit hote hain            ✅ Kam chunks manage karne
❌ Context kat jaata hai                 ❌ Embedding "dhundhli"
❌ Zyada chunks = zyada noise            ❌ Zyada irrelevant text LLM ko
❌ Cross-reference toot jaate hain       ❌ Kam chunks fit hote hain
```

> 🔑 **Golden insight:** Ye trade-off **jhoota** hai. Best strategies **dono** dete hain — chhote chunk se **search** karo, bade chunk se **answer** do. (Parent-document / sentence-window retrieval — Part 3 mein.)

---

## Part 2: Chunking strategies (simple se advanced)

### (1) Fixed-size — baseline

```python
def chunk_fixed(text, size=500, overlap=50):
    chunks, start = [], 0
    while start < len(text):
        chunks.append(text[start:start + size])
        start += size - overlap
    return chunks
```

**Kab:** Prototype, uniform text.
**Problem:** Shabd/sentence beech mein kat jaate hain. Structure ignore.

### (2) Recursive character splitting — practical default

Separators ki hierarchy try karo, order mein:

```python
separators = ["\n\n", "\n", ". ", " ", ""]
# Pehle paragraph pe todo. Bada hai? Line pe todo. Bada hai? Sentence pe. Phir shabd.
```

**Kab:** **Yahi tumhara default hona chahiye.** Simple, structure-aware enough, fast.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=count_tokens,   # characters nahi, TOKENS count karo!
)
```

### (3) Document-structure-aware — best jab structure hai

Document ke natural boundaries use karo:

| Doc type | Split on |
|---|---|
| Markdown | Headers (`#`, `##`, `###`) |
| HTML | `<section>`, `<div>`, `<h1-h6>` |
| Code | Functions, classes (AST se!) |
| PDF | Sections, pages |
| CSV/tables | Rows (header preserve karke!) |
| Legal | Clauses, numbered sections |

```python
# Markdown example — headers ko metadata bana do
headers = [("#", "h1"), ("##", "h2"), ("###", "h3")]
# Chunk metadata: {"h1": "HR Policies", "h2": "Leave", "h3": "Parental Leave"}
# → Ab tum metadata pe filter kar sakte ho AND context inject kar sakte ho!
```

> 🔥 **Ye sabse underrated technique hai.** Agar tumhare docs structured hain (markdown, HTML, legal), structure use karna semantic chunking se better aur 100x sasta hai.

### (4) Semantic chunking — meaning se todo

Consecutive sentences ke embeddings compare karo; jahan bada jump ho, wahan todo:

```python
def semantic_chunk(sentences, threshold=0.5):
    embs = embed(sentences)
    chunks, current = [], [sentences[0]]
    for i in range(1, len(sentences)):
        if cosine_sim(embs[i-1], embs[i]) < threshold:   # topic badal gaya
            chunks.append(" ".join(current)); current = []
        current.append(sentences[i])
    chunks.append(" ".join(current))
    return chunks
```

**Kab:** Unstructured prose (articles, transcripts, meeting notes).
**Cost:** Har sentence embed karna padta hai — indexing slow aur mehngi.
**Imaandaar assessment:** Fayda usually **chhota** hota hai vs recursive+overlap. **Pehle recursive try karo**, semantic tab jab measure karke gap dikhe.

### (5) LLM-based chunking

LLM se poochho "ye document kahan todna chahiye". Best quality, **bahut** mehnga. Sirf high-value static corpus pe worth hai.

### Comparison

| Strategy | Quality | Cost | Kab use karo |
|---|---|---|---|
| Fixed-size | ⭐⭐ | 💰 | Prototype only |
| **Recursive** | ⭐⭐⭐ | 💰 | **Default** |
| **Structure-aware** | ⭐⭐⭐⭐ | 💰 | **Jab structure ho — best ROI** |
| Semantic | ⭐⭐⭐⭐ | 💰💰💰 | Unstructured prose, measure karke |
| LLM-based | ⭐⭐⭐⭐⭐ | 💰💰💰💰💰 | High-value static corpus |

---

## Part 3: Advanced patterns (trade-off ko todne wale) ⭐

### Parent-Document Retrieval

**Idea:** Chhote chunk se **search** karo, bade parent se **answer** do.

```
Indexing:
  Document → Parent chunks (2000 tokens) → Child chunks (400 tokens)
             Parent store karo (docstore)   Child EMBED karo (vector DB)
             child → parent ka mapping rakho

Query:
  Query → child chunks search (precise!) → unke PARENTS fetch karo → LLM ko parents do
```

```python
# Simplified
for parent in parent_chunks:
    pid = save_parent(parent)                       # docstore
    for child in split(parent, size=400):
        vector_db.add(embed(child), metadata={"parent_id": pid})

def retrieve(query, k=5):
    children = vector_db.search(embed(query), k=k*3)
    parent_ids = dedupe([c.metadata["parent_id"] for c in children])[:k]
    return [get_parent(pid) for pid in parent_ids]
```

**Result:** Precise search + full context. **Ye ek best-value technique hai** — implement karna aasan, fayda bada.

### Sentence-Window Retrieval

Similar idea, finer granularity:

```
Index: har SENTENCE alag embed karo
Retrieve: matching sentence + uske aage-peeche ki N sentences do
```

```
Match:  "Parental leave 26 hafte ki hai."
Window: "Company family-friendly policies rakhti hai. Parental leave 26 hafte
         ki hai. Iske liye HR ko 60 din pehle notice dena hota hai."
```

**Best for:** Factual Q&A, dense reference documents.

### Multi-Representation Indexing

Ek chunk ke **kai** searchable versions banao:

```
Chunk (original text)
   ├─ Embedding of raw text
   ├─ Embedding of LLM-generated SUMMARY
   ├─ Embedding of HYPOTHETICAL QUESTIONS ("Is chunk se kaunse sawaal ban sakte hain?")
   └─ Extracted keywords (BM25 ke liye)
```

**Hypothetical questions wala trick powerful hai** — query-to-question matching, query-to-document matching se better hota hai (Failure 1 ko directly attack karta hai!).

```python
questions = llm(f"Is text se 3 sawaal banao jinke jawab ye text deta hai:\n{chunk}")
for q in questions:
    vector_db.add(embed(q), metadata={"chunk_id": cid, "type": "question"})
```

**Cost:** Indexing pe ek LLM call per chunk. Static corpus pe worth it.

### Contextual chunk enrichment 🔥

Chunk ko standalone samajhne layak banao — ye **bada difference** karta hai:

```
❌ Raw chunk (context ke bina meaningless):
"Ye 26 hafte ki hoti hai aur iske liye 60 din ka notice chahiye."
   → "Ye" kya? Kisko pata!

✅ Enriched chunk:
"[Document: HR Policy 2024 | Section: Leave Policies > Parental Leave]
Ye 26 hafte ki hoti hai aur iske liye 60 din ka notice chahiye."
   → Ab embedding "parental leave" ke saath match karega!
```

**Implementation:** Header path prepend karo (structure-aware chunking se free milta hai), ya LLM se ek chhota context line generate karao.

> **Ye ek chhota change hai jiska bada asar hota hai.** Kai teams isse alone 10-15 points accuracy gain report karte hain.

---

## Part 4: Metadata — free superpower

Chunk ke saath metadata store karo. Ye **filtering** enable karta hai, jo search ko dramatically better banata hai:

```python
{
    "text": "Parental leave 26 hafte...",
    "metadata": {
        # Source
        "source": "hr_policy_2024.pdf",
        "page": 12,
        "section_path": "Leave Policies > Parental Leave",

        # Filtering ke liye (ye gold hai)
        "doc_type": "policy",
        "department": "hr",
        "language": "en",
        "effective_date": "2024-01-01",
        "expiry_date": "2025-12-31",
        "access_level": "all_employees",     # ← SECURITY!

        # Ranking ke liye
        "version": 3,
        "last_updated": "2024-06-15",
    }
}
```

### Metadata filtering se kya milta hai

```python
# ❌ Bina filter — purani policy bhi aa jaayegi
results = search(query, k=5)

# ✅ Filter ke saath — sirf current, sirf allowed
results = search(query, k=5, filter={
    "department": "hr",
    "effective_date": {"$lte": today},
    "expiry_date": {"$gte": today},
    "access_level": {"$in": user.access_levels},   # 🔒 CRITICAL
})
```

### 🚨 Security: Access control **retrieval** mein hona chahiye

```python
# ❌ KHATARNAK — LLM pe bharosa
context = search(query, k=5)                # sab kuch
prompt = f"Confidential docs mention na karo.\n{context}"
# → Prompt injection se ye bypass ho jaayega. Guaranteed leak.

# ✅ SAHI — filter retrieval layer pe
context = search(query, k=5, filter={"access_level": {"$in": user.roles}})
# → Model ne wo data DEKHA hi nahi. Leak impossible.
```

> **Rule:** "Model ko bol diya ki na bataye" **security nahi hai**. Data model tak pahunchna hi nahi chahiye. Ye enterprise RAG ka #1 audit failure hai.

---

## Part 5: Embedding model selection

| Consideration | Detail |
|---|---|
| **Dimensions** | 384 (fast, chhota) → 1536 → 3072 (accurate, mehnga storage). 768-1024 sweet spot |
| **Max sequence** | 512 tokens (chhote chunks) vs 8192 (bade chunks OK) |
| **Multilingual** | Hindi/regional chahiye? Multilingual model lo (`bge-m3`, `multilingual-e5`) |
| **Domain** | Code, medical, legal ke specialized models exist karte hain |
| **API vs local** | API: easy, per-call cost. Local: free after setup, GPU chahiye |
| **MTEB score** | Leaderboard dekho — par apne data pe test karo, blindly na maano |

### Practical picks (2026)

| Need | Option |
|---|---|
| General, hosted, easy | OpenAI `text-embedding-3-large`, Voyage `voyage-3` |
| Open-source, self-host | `BAAI/bge-large-en-v1.5`, `intfloat/e5-large-v2` |
| Multilingual (Hindi!) | `BAAI/bge-m3`, `intfloat/multilingual-e5-large` |
| Fast/cheap prototype | `sentence-transformers/all-MiniLM-L6-v2` (384-dim, laptop pe chalega) |
| Code | `jina-embeddings-v2-base-code`, `voyage-code-3` |

### ⚠️ Do critical gotchas

**1. Query aur document ke liye SAME model, aur prefixes ka dhyaan.**
Kai models ko asymmetric prefix chahiye:
```python
# bge/e5 models mein:
doc_emb   = embed("passage: " + chunk)
query_emb = embed("query: " + user_query)
# Prefix bhoolna = silently 10-20% accuracy loss, koi error nahi
```

**2. Embedding model badla = POORA index rebuild.**
Different models ke vector spaces incompatible hote hain. `text-embedding-3` ka vector aur `bge-m3` ka vector compare karna meaningless hai.
```
Model change → poora corpus re-embed → hours/days + cost
```
Isliye **shuru mein hi soch ke choose karo**, aur model name metadata mein store karo.

---

## Part 6: Indexing pipeline (production shape)

```python
def index_document(file_path):
    # 1. PARSE (layout-aware!)
    doc = parse(file_path)                   # Unstructured/LlamaParse/Docling

    # 2. CLEAN
    doc.text = clean(doc.text)               # headers/footers/page numbers hatao

    # 3. CHUNK (structure-aware)
    chunks = structure_aware_chunk(doc, target_tokens=800, overlap=100)

    # 4. ENRICH (context add karo)
    for c in chunks:
        c.text = f"[{doc.title} | {c.section_path}]\n{c.text}"

    # 5. METADATA
    for c in chunks:
        c.metadata = {**doc.metadata, "section_path": c.section_path,
                      "chunk_index": c.i, "embedding_model": MODEL_NAME}

    # 6. EMBED (batch karo — 10x faster)
    embeddings = embed_batch([c.text for c in chunks], batch_size=64)

    # 7. UPSERT (idempotent! deterministic IDs)
    for c, e in zip(chunks, embeddings):
        c.id = hash(f"{doc.id}:{c.i}")       # re-index safe
        vector_db.upsert(c.id, e, c.metadata, c.text)

    # 8. BM25 index bhi update karo (hybrid search ke liye)
    keyword_index.add([c.text for c in chunks])
```

### Production concerns

| Concern | Solution |
|---|---|
| **Idempotency** | Deterministic chunk IDs (`hash(doc_id:index)`) — re-index duplicate na banaye |
| **Incremental updates** | Doc hash store karo; badla nahi to skip karo |
| **Deletion** | Doc delete = uske saare chunks delete. `doc_id` se query karo |
| **Versioning** | Purana version rakho (audit), par filter mein latest do |
| **Failures** | Ek doc fail = poora job fail na ho. Log karo, continue karo |
| **Cost** | Embedding calls batch karo, results cache karo (text hash se) |

---

## Part 7: Chunk size — practical guidance

**Ek "best" size nahi hai. Ye starting points hain, phir MEASURE karo:**

| Content type | Chunk size (tokens) | Overlap |
|---|---|---|
| FAQ / short answers | 200-400 | 20-50 |
| Policy / documentation | 500-800 | 50-100 |
| Technical docs / manuals | 800-1200 | 100-150 |
| Narrative / books | 1000-1500 | 150-200 |
| Code | Function/class boundary | Imports/signature repeat karo |
| Tables | Poora table (ya row groups + header) | Header har chunk mein |
| Chat transcripts | 5-10 turns | 1-2 turns |

**Overlap rule of thumb:** chunk size ka 10-15%. Zyada overlap = duplicate results aur storage waste.

> **Sabse important:** Chunk size ek **hyperparameter** hai. 3 sizes try karo (400, 800, 1200), apne golden set pe Recall@5 measure karo, best chuno. **Guess na karo — 30 minute ka experiment hai.**

---

## Khud se pooch 🤔

1. Chhote vs bade chunks ka trade-off kya hai? Parent-document retrieval usse kaise **todta** hai?
2. "Contextual chunk enrichment" kya hai aur wo kaam kyun karta hai?
3. Access control **retrieval layer** pe kyun hona chahiye, prompt mein kyun nahi?
4. Embedding model badalne pe kya karna padta hai? Ye kyun ek bada decision hai?
5. Multi-representation indexing mein "hypothetical questions" kyun powerful hain? (Failure mode 1 se jodo)
6. Ye chunk kharab kyun hai aur kaise sudhaaroge?
   ```
   "...aur iske liye 60 din ka notice dena hota hai. Exception cases
   mein HR se approval le sakte hain. Section 4.3 dekhein."
   ```
7. Tumhare paas 500 markdown docs hain nested headers ke saath. Kaunsi chunking strategy aur kyun?

<details>
<summary>Answers 6 aur 7</summary>

**6. Teen problems:**

1. **Standalone meaningless.** "iske liye" — kiske liye? Chunk apne aap mein self-contained nahi hai. Embedding "notice period" ke saath match karega par kisi bhi actual query ke saath weakly.
2. **Beech se shuru.** Chunk boundary ne sentence ka pehla hissa kaat diya.
3. **Dangling reference.** "Section 4.3 dekhein" — retrieval ke waqt 4.3 aayega hi nahi.

**Fix:**
```
[Document: HR Policy 2024 | Section: 4.2 Leave Policies > Parental Leave]

Parental leave 26 hafte ki paid hoti hai. Iske liye 60 din ka notice dena
hota hai. Exception cases mein HR se approval le sakte hain.

[Related: Section 4.3 — Leave Encashment]
```
- Header path prepend kiya → standalone meaningful, aur query se match karega
- Sentence boundary pe toda (recursive splitter with `". "` separator)
- Cross-reference ko metadata/note bana diya — ya better, parent-document retrieval use karo taaki 4.3 bhi context mein aa jaaye

**7. Structure-aware chunking (markdown headers pe).** Reasons:

1. **Free semantic boundaries.** Headers author ne khud lagaye hain — wo natural topic boundaries hain. Semantic chunking jo embeddings se *guess* karta hai, wo yahan already ground truth mein available hai. Aur 100x sasta.
2. **Metadata free milti hai.** Header path (`h1 > h2 > h3`) → filtering + enrichment dono ke liye.
3. **Enrichment trivial.** Header path prepend karna ek line hai.

**Concrete plan:**
```python
# 1. Markdown headers pe split, header path metadata mein
# 2. Bade sections ko recursive splitter se 800 tokens mein todo (100 overlap)
# 3. Har chunk ko header path se enrich karo
# 4. Parent-document retrieval: parent = section, child = 400-token pieces
# 5. Metadata: {h1, h2, h3, source, last_updated}
```

**Jo NA karo:** Semantic chunking. Yahan structure already hai — embeddings se boundaries guess karna paisa aur time waste hai, aur likely *worse* hoga.
</details>

---

**Next:** [`03-retrieval-hybrid-reranking.md`](03-retrieval-hybrid-reranking.md) — sabse zyada ROI wali file.
