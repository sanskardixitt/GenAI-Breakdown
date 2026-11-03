# 05 — Vector Databases

> **Is file ka goal:** Vector DB choose karna. Aur ek important message: **zyadatar log ko dedicated vector DB ki zaroorat nahi hai.**

---

## Part 1: Vector DB karta kya hai

```
Input:  query vector [0.2, -0.8, ..., 0.4]   (1024 dimensions)
Task:   10 million stored vectors mein se sabse similar 10 dhundo
Output: [(id, score), ...]
```

**Naive approach:** Har vector se compare karo — 10M comparisons. Slow (seconds).
**Vector DB:** **ANN (Approximate Nearest Neighbour)** index — milliseconds mein.

### ANN algorithms

| Algorithm | Kaise | Trade-off |
|---|---|---|
| **HNSW** ⭐ | Multi-layer graph, "skip list" jaisa | Fast + accurate, **memory-hungry**. Default choice. |
| **IVF** | Vectors ko clusters mein baato, relevant clusters search karo | Kam memory, tuning chahiye (`nprobe`) |
| **IVF-PQ** | IVF + Product Quantization (vector compress) | Bahut kam memory, thodi accuracy loss |
| **DiskANN** | SSD-based | Bahut bade datasets, RAM se bada |
| **Flat** | Brute force, exact | 100% accurate, chhote datasets (<100K) |

### 🔑 Recall — ANN ka "A"

```
ANN = APPROXIMATE. Wo best matches MISS kar sakta hai.

Recall@10 = 0.95  →  95% baar top-10 mein sahi answer hai
                      5% baar index ne miss kar diya!
```

**Ye tumhare RAG failure ka ek chhupa hua source hai.** Agar tumne HNSW parameters aggressively tune kiye (speed ke liye), tum silently recall kho rahe ho.

```python
# HNSW parameters
M = 16                    # graph connections per node (zyada = better recall, zyada memory)
ef_construction = 200     # build-time search depth (zyada = better index, slow build)
ef_search = 100           # query-time search depth (zyada = better recall, slow query)
```

**Debug tip:** Chhote dataset pe **flat (exact)** search se compare karo. Agar ANN aur exact ke results bahut alag hain, `ef_search` badhao.

---

## Part 2: Options — imaandaar comparison

### 🥇 Postgres + pgvector (default recommendation)

```sql
CREATE EXTENSION vector;

CREATE TABLE chunks (
    id          BIGSERIAL PRIMARY KEY,
    doc_id      TEXT NOT NULL,
    text        TEXT NOT NULL,
    embedding   vector(1024),
    metadata    JSONB,
    tsv         tsvector GENERATED ALWAYS AS (to_tsvector('english', text)) STORED
);

-- Vector index
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200);

-- Full-text index (HYBRID SEARCH ke liye!)
CREATE INDEX ON chunks USING GIN (tsv);

-- Metadata filtering
CREATE INDEX ON chunks USING GIN (metadata);
```

**Hybrid search ek query mein:**
```sql
WITH vec AS (
  SELECT id, RANK() OVER (ORDER BY embedding <=> $1) AS rank
  FROM chunks WHERE metadata->>'access_level' = ANY($3) LIMIT 50
),
kw AS (
  SELECT id, RANK() OVER (ORDER BY ts_rank(tsv, plainto_tsquery($2)) DESC) AS rank
  FROM chunks WHERE tsv @@ plainto_tsquery($2) LIMIT 50
)
SELECT c.*, COALESCE(1.0/(60+vec.rank), 0) + COALESCE(1.0/(60+kw.rank), 0) AS rrf
FROM chunks c
LEFT JOIN vec ON c.id = vec.id
LEFT JOIN kw  ON c.id = kw.id
WHERE vec.id IS NOT NULL OR kw.id IS NOT NULL
ORDER BY rrf DESC LIMIT 20;
```

**Ye ek query mein hybrid search + RRF + ACL filtering kar raha hai.** 🔥

| ✅ Fayde | ❌ Nuksaan |
|---|---|
| **Ek DB** — data + vectors + full-text ek jagah | ~10M vectors ke aage tuning chahiye |
| **ACID transactions** — chunk aur metadata sync mein | Dedicated DBs se thoda slow at extreme scale |
| **Joins!** — vectors ko business data se join karo | HNSW build memory-heavy |
| **Ops simple** — already Postgres jaante ho | |
| **Metadata filtering strong** — SQL WHERE | |
| **Backup/replication** — Postgres ka mature tooling | |
| **Sasta** — koi extra vendor nahi | |

> 🔑 **Meri recommendation: 90% projects ke liye pgvector se shuru karo.**
>
> "Vector DB chahiye" ek assumption hai, requirement nahi. 10 million vectors tak pgvector comfortably chalta hai. Zyadatar products kabhi wahan pahunchte hi nahi.

### Qdrant

Rust mein likha, purpose-built vector DB.

| ✅ | ❌ |
|---|---|
| Excellent filtering (payload-based, pre-filtering) | Ek aur system to run |
| Fast, memory-efficient | Data duplication (source of truth kahan?) |
| Self-host friendly (single binary) | |
| Good hybrid support | |
| Quantization built-in | |

**Kab:** Dedicated vector DB chahiye, self-host karna hai, filtering-heavy workload.

### Weaviate

| ✅ | ❌ |
|---|---|
| Hybrid search **built-in** (BM25 + vector) | Heavier to operate |
| GraphQL API | Learning curve |
| Modules (embedding generation built-in) | |
| Multi-tenancy | |

**Kab:** Hybrid search out-of-box chahiye, multi-tenant SaaS.

### Pinecone

| ✅ | ❌ |
|---|---|
| **Fully managed** — zero ops | Vendor lock-in |
| Serverless option | Cost scale pe badh sakti hai |
| Reliable, mature | Data provider ke paas |
| Good metadata filtering | Self-host option nahi |

**Kab:** Team chhoti, ops bandwidth nahi, managed prefer karte ho.

### Milvus / Zilliz

**Kab:** Genuinely huge scale (100M-1B+ vectors), distributed setup chahiye. Overkill zyadatar cases mein.

### Elasticsearch / OpenSearch

| ✅ | ❌ |
|---|---|
| **BM25 + vector dono** natively | Heavy to operate |
| Already hai to free win | Vector performance dedicated DBs se kam |
| Mature filtering, aggregations | JVM tuning |

**Kab:** Already Elastic hai. Tab naya system lena galat hoga.

### Chroma / FAISS / LanceDB

**Kab:** Prototype, local dev, notebooks. Production ke liye upar wale options.

- **FAISS:** Library hai, DB nahi. Persistence/filtering khud handle karo.
- **Chroma:** Dev-friendly, easy start.
- **LanceDB:** Embedded, file-based, serverless-friendly. Interesting for edge cases.

---

## Part 3: Decision framework 🎯

```
Kitne vectors?
   │
   ├─ < 100K
   │    → In-memory (FAISS/numpy) ya pgvector. Kuch bhi chalega.
   │
   ├─ 100K - 10M
   │    → Kya already Postgres hai?
   │         HAAN → pgvector ⭐ (naya system mat lo)
   │         NAHI → Kya already Elastic hai?
   │                   HAAN → Elastic
   │                   NAHI → Qdrant (self-host) ya Pinecone (managed)
   │
   ├─ 10M - 100M
   │    → Qdrant / Weaviate / Pinecone
   │      (pgvector possible hai par tuning chahiye)
   │
   └─ > 100M
        → Milvus / Vespa / managed at scale
```

### Aur ye poocho

| Sawaal | Asar |
|---|---|
| **Filtering kitni heavy hai?** | Heavy → Qdrant ya pgvector (SQL WHERE) |
| **Hybrid search chahiye?** | Haan → pgvector+FTS, Weaviate, ya Elastic |
| **Multi-tenant?** | Namespace/tenant isolation dekho |
| **Ops bandwidth?** | Nahi → managed (Pinecone) |
| **Data residency/compliance?** | Self-host mandatory ho sakta hai |
| **Update frequency?** | Frequent updates → index rebuild cost dekho |

---

## Part 4: Practical concerns (jo docs mein nahi hote)

### Concern 1: Source of truth

```
❌ Vector DB source of truth hai
   → Text sirf vector DB mein, phir tum embedding model badalna chahte ho
   → Re-embed karne ke liye original text kahan se laoge? 💥

✅ Vector DB ek INDEX hai
   → Source of truth: Postgres / S3 / doc store
   → Vector DB derived hai — kabhi bhi rebuild ho sakta hai
```

**Ye ek architectural rule hai:** Vector DB ko **cache/index** samjho, database nahi. Rebuild script hamesha ready rakho.

### Concern 2: Embedding model change = full rebuild

```
Model badla → saare vectors invalid → poora corpus re-embed
```
Isliye:
- Embedding model ka **naam metadata mein store karo**
- Rebuild pipeline **din 1 se** ready rakho
- Blue-green pattern: naya index build karo, phir switch karo

### Concern 3: Deletion aur updates

```python
# ❌ Document delete kiya par chunks reh gaye
db.delete_document(doc_id)     # vector DB mein 47 chunks abhi bhi hain!

# ✅ Cascade
def delete_document(doc_id):
    vector_db.delete(filter={"doc_id": doc_id})   # saare chunks
    keyword_index.delete(doc_id)
    docstore.delete(doc_id)
```

**Ye ek real production bug hai** — deleted documents RAG mein answer dete rehte hain. Especially dangerous jab document access-revoked ho.

### Concern 4: Metadata filtering strategy

```
PRE-filtering:  pehle filter, phir vector search
                → accurate, par slow (index ka fayda kam)
POST-filtering: pehle vector search (k=100), phir filter
                → fast, par top-k filter ke baad khaali ho sakta hai!
```

```python
# ❌ Post-filter ka bug
results = search(query, k=10)                     # 10 mile
filtered = [r for r in results if r.access_ok]    # 2 bache 💥
```

**Fix:** Over-fetch karo (`k=50`) ya native pre-filtering use karo (Qdrant/pgvector isme strong hain).

### Concern 5: Cost model

| DB | Cost driver |
|---|---|
| pgvector | Tumhara Postgres instance (RAM matters — HNSW memory-hungry) |
| Pinecone | Pods/serverless units + storage |
| Qdrant Cloud | RAM + storage |
| Managed generally | Vectors count × dimensions |

**Dimension reduction ka fayda:** 3072-dim se 1024-dim jaane se storage 3x kam, aur accuracy loss aksar minimal. Kai modern embedding models **Matryoshka** support karte hain — vector ko truncate kar sakte ho.

---

## Part 5: Ek production setup (pgvector)

```python
class VectorStore:
    def upsert(self, chunks, embeddings):
        """Idempotent — deterministic IDs se re-index safe."""
        with self.conn.transaction():
            for c, e in zip(chunks, embeddings):
                self.conn.execute("""
                    INSERT INTO chunks (id, doc_id, text, embedding, metadata, embed_model)
                    VALUES (%s, %s, %s, %s, %s, %s)
                    ON CONFLICT (id) DO UPDATE SET
                        text = EXCLUDED.text,
                        embedding = EXCLUDED.embedding,
                        metadata = EXCLUDED.metadata,
                        updated_at = now()
                """, (c.id, c.doc_id, c.text, e, Json(c.metadata), EMBED_MODEL))

    def delete_document(self, doc_id):
        self.conn.execute("DELETE FROM chunks WHERE doc_id = %s", (doc_id,))

    def hybrid_search(self, query, embedding, k=50, access_levels=None):
        # Upar wala RRF query
        ...

    def health(self):
        """Recall check — production mein ye monitor karo."""
        exact = self.flat_search(TEST_QUERY, k=10)
        ann   = self.search(TEST_QUERY, k=10)
        return len(set(exact) & set(ann)) / 10        # recall@10
```

> **`health()` wala pattern underrated hai.** ANN recall silently degrade ho sakti hai jab index badhta hai. Ise monitor karo.

---

## Khud se pooch 🤔

1. ANN mein "A" ka kya matlab hai? Ye RAG failure ka chhupa hua source kaise ban sakta hai?
2. HNSW ke 3 parameters kya hain? `ef_search` badhane se kya hota hai?
3. pgvector 90% cases ke liye default kyun hona chahiye?
4. Vector DB ko "source of truth" banana kyun galat hai?
5. Pre-filtering aur post-filtering mein farak? Post-filtering ka bug kya hai?
6. Embedding model badalne pe kya karna padta hai? Uske liye din 1 se kya ready rakhna chahiye?
7. Document deletion ka production bug kya hai? Wo security issue kaise ban sakta hai?
8. Tumhari team 500K documents (~5M chunks) index karna chahti hai. Already Postgres hai. Team 4 log ki hai, dedicated infra engineer nahi. Kya recommend karoge?

<details>
<summary>Answer 8</summary>

**pgvector. Bina hichkichahat.**

**Reasoning:**

1. **Scale fit hai.** 5M vectors pgvector ke liye comfortable range hai (10M tak theek chalta hai proper tuning ke saath).

2. **Already Postgres hai** — ye sabse bada factor hai. Naya system lene ka matlab:
   - Naya deployment, monitoring, backup, upgrade path
   - Data sync problem (Postgres aur vector DB consistent kaise rahenge?)
   - Naya failure mode team ke liye
   - 4 log ki team mein ye **real** cost hai

3. **Dedicated infra engineer nahi** — ye decisive hai. Qdrant/Weaviate self-host karna matlab koi us system ko own karega. Kaun?

4. **Hybrid search free mil raha hai** — Postgres FTS + pgvector ek query mein (upar ka RRF query). Weaviate ka main selling point yahan already available hai.

5. **ACL filtering SQL se** — `WHERE metadata->>'access_level' = ANY(...)` — mature, tested, aur pre-filtering hai (post-filtering ka bug nahi).

**Setup recommendations:**

```sql
-- Index (5M vectors ke liye)
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 200);
-- Build time: ~30-60 min. maintenance_work_mem badhao build ke liye.

SET maintenance_work_mem = '4GB';    -- build ke liye
SET hnsw.ef_search = 100;             -- query time (tune karo recall ke hisaab se)
```

**Sizing:** 5M × 1024 dims × 4 bytes ≈ 20GB raw vectors, plus HNSW graph overhead (~1.5x) ≈ **30GB**. Instance mein enough RAM chahiye — warna disk se read hoga aur slow ho jaayega.

**Cost optimization:** Agar 30GB RAM mehnga lag raha hai, embedding dimensions kam karo (1024 → 512 Matryoshka se). Storage half, accuracy loss usually 1-2%.

**Kab migrate karo (aur kab tak intezaar karo):**
- Vectors 20M+ ho jaayein, **aur** latency P95 target miss kar rahi ho
- Ya team mein koi dedicated infra owner aa jaaye
- **Tab tak nahi.** Premature migration ek classic time sink hai.

**Din 1 se ye ready rakho:**
1. **Rebuild script** — poora corpus re-embed + re-index (embedding model change ke liye)
2. **`health()` check** — ANN recall monitor karo
3. **Cascade delete** — document delete = chunks delete
4. **`embed_model` column** — pata rahe kaunse model se vectors bane
</details>

---

**Next:** [`06-serving-aur-inference-stack.md`](06-serving-aur-inference-stack.md)
