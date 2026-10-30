# 02 — LlamaIndex

> **Ek line mein:** LlamaIndex **RAG-first** framework hai. LangChain broad hai; LlamaIndex **data → LLM** ke path pe deep hai.

---

## Part 1: Positioning

```
LangChain    : "LLM apps ka toolkit"      (broad, sab kuch)
LlamaIndex   : "Data framework for LLMs"  (deep, RAG-focused)
```

Agar tumhara problem **"apne data ko LLM tak pahunchana"** hai — LlamaIndex ka design usi ke around hai.

### Core abstractions

| Concept | Kya hai |
|---|---|
| **Document** | Raw source (file, page, record) |
| **Node** | Ek chunk + metadata + relationships (prev/next/parent) |
| **Index** | Nodes ka organized structure (vector, keyword, tree, graph) |
| **Retriever** | Index se nodes nikalne ka logic |
| **Node Postprocessor** | Reranking, filtering, metadata replacement |
| **Response Synthesizer** | Nodes → final answer (multiple modes) |
| **Query Engine** | Retriever + postprocessor + synthesizer, ek saath |
| **Agent** | Tools ke saath loop |

> **`Node` ka concept LangChain ke `Document` se richer hai** — usme relationships hote hain (`prev`, `next`, `parent`, `child`). Isi se parent-document aur sentence-window retrieval built-in milte hain.

---

## Part 2: 5-line RAG

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

documents = SimpleDirectoryReader("./data").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine()
print(query_engine.query("Refund policy kya hai?"))
```

**Ye impressive hai** — par yaad rakho ye **naive RAG** hai (Day 3, Level 0). Production ke liye hybrid + reranking chahiye.

---

## Part 3: LlamaIndex ki genuine strengths

### Strength 1: LlamaHub — connectors ⭐

```python
from llama_index.readers.notion import NotionPageReader
from llama_index.readers.google import GoogleDriveReader
from llama_index.readers.slack import SlackReader
from llama_index.readers.database import DatabaseReader
```

300+ ready-made data readers. Agar tumhe 10 sources se data laana hai, ye **weeks bachate hain**.

### Strength 2: Advanced retrieval patterns built-in ⭐

Day 3 ke advanced patterns yahan ready-made hain:

```python
# SENTENCE WINDOW (Day 3, file 02)
from llama_index.core.node_parser import SentenceWindowNodeParser
from llama_index.core.postprocessor import MetadataReplacementPostProcessor

parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,                                # 3 sentences aage-peeche
    window_metadata_key="window",
    original_text_metadata_key="original_text",
)
# Retrieval: sentence se match, window return

query_engine = index.as_query_engine(
    node_postprocessors=[MetadataReplacementPostProcessor(target_metadata_key="window")]
)
```

```python
# AUTO-MERGING (parent-document ka smart version)
from llama_index.core.node_parser import HierarchicalNodeParser
from llama_index.core.retrievers import AutoMergingRetriever

parser = HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128])
# Agar ek parent ke kai children retrieve hue → parent return karo
```

```python
# HYBRID + RERANK
from llama_index.core.retrievers import QueryFusionRetriever
from llama_index.core.postprocessor import SentenceTransformerRerank

retriever = QueryFusionRetriever(
    [vector_retriever, bm25_retriever],
    similarity_top_k=50,
    mode="reciprocal_rerank",                     # RRF!
    num_queries=4,                                # multi-query bhi
)
reranker = SentenceTransformerRerank(model="BAAI/bge-reranker-v2-m3", top_n=5)

query_engine = RetrieverQueryEngine.from_args(retriever, node_postprocessors=[reranker])
```

> **Ye LlamaIndex ka strongest pitch hai:** Day 3 ke saare advanced patterns (parent-doc, sentence-window, hybrid, RRF, reranking, multi-query) **already implemented** hain, tested hain. Tumhe sirf compose karna hai.

### Strength 3: Response synthesis modes

```python
query_engine = index.as_query_engine(response_mode="...")
```

| Mode | Kaise | Kab |
|---|---|---|
| `compact` (default) | Chunks stuff karo, jitne fit hon | Standard |
| `refine` | Ek-ek chunk pe answer refine karo | Bahut chunks, quality > cost |
| `tree_summarize` | Hierarchical summarization | **Poore document ka summary** |
| `accumulate` | Har chunk ka alag answer, phir join | Multi-part questions |
| `no_text` | Sirf nodes return karo, generate nahi | Custom generation |

`tree_summarize` genuinely useful hai — Day 3 ka "poore document ka summary" problem ye solve karta hai.

### Strength 4: Query engine composition

```python
from llama_index.core.tools import QueryEngineTool
from llama_index.core.query_engine import RouterQueryEngine

router = RouterQueryEngine.from_defaults(query_engine_tools=[
    QueryEngineTool.from_defaults(hr_engine,
        description="HR policies, leave, benefits ke sawaalon ke liye"),
    QueryEngineTool.from_defaults(eng_engine,
        description="Technical documentation, architecture ke liye"),
    QueryEngineTool.from_defaults(sql_engine,
        description="Counts, sums, aggregations ke liye (structured data)"),
])
```

Ye Day 3 ka **router pattern** hai, ready-made. Aur `sql_engine` include karna wahi fix hai jo aggregation queries ke liye chahiye.

---

## Part 4: LlamaIndex vs LangChain (RAG ke liye)

| Aspect | LlamaIndex | LangChain |
|---|---|---|
| **Focus** | RAG deep | Broad |
| **Retrieval patterns** | ✅ Zyada, ready-made | Basic + custom |
| **Node relationships** | ✅ First-class | Manual |
| **Connectors** | 300+ (LlamaHub) | 700+ (community) |
| **Agents** | Decent | Better (LangGraph) |
| **Learning curve** | Medium | Medium-high |
| **Documentation** | Good | Vast but scattered |
| **Ecosystem size** | Smaller | Larger |

### 🔑 Practical guidance

```
RAG-heavy product, complex retrieval  → LlamaIndex
Agent-heavy, complex orchestration    → LangGraph
Bahut varied integrations             → LangChain
Simple RAG                            → Custom (~200 lines)
Dono chahiye                          → LlamaIndex (RAG) + LangGraph (orchestration)
                                        ya dono ko chhod ke custom
```

> ⚠️ **LangChain + LlamaIndex ek saath mixing** common hai par debugging painful ho jaata hai — do abstraction systems, do vocabularies. Agar karna hi hai to **clear boundary** rakho: LlamaIndex sirf retrieval de, LangGraph sirf orchestration.

---

## Part 5: Kab LlamaIndex NA use karo

| Situation | Kyun nahi |
|---|---|
| Simple RAG (ek source, basic search) | 200 lines custom simpler hai |
| Full control chahiye har stage pe | Abstractions beech mein aayengi |
| Latest model features chahiye | Wrapper lag karta hai |
| Team ko already LangChain aata hai | Do frameworks = do learning curves |
| Sirf agents chahiye, RAG nahi | Ye uska core nahi hai |

---

## Part 6: Practical hybrid (recommended)

```python
# LlamaIndex ka BEST use: loading aur node parsing
from llama_index.readers.notion import NotionPageReader
from llama_index.core.node_parser import SentenceWindowNodeParser

docs = NotionPageReader(integration_token=TOKEN).load_data(page_ids=IDS)
nodes = SentenceWindowNodeParser.from_defaults(window_size=3).get_nodes_from_documents(docs)

# Ab APNA pipeline — full control
for n in nodes:
    my_vector_db.upsert(n.node_id, embed(n.text), n.metadata, n.text)
    my_bm25_index.add(n.node_id, n.text)

def my_rag(query):
    hits = rrf([my_bm25_index.search(query, 50), my_vector_db.search(embed(query), 50)])
    top5 = my_reranker.rerank(query, hits)[:5]
    windows = [h.metadata["window"] for h in top5]          # sentence-window!
    return anthropic_client.messages.create(                 # direct SDK
        model="claude-opus-5", max_tokens=2048,
        system=RAG_SYSTEM,
        messages=[{"role": "user", "content": build(query, windows)}],
    )
```

**Ye pattern kyun best hai:**
- ✅ Connectors + node parsing ka fayda (weeks bachate hain)
- ✅ Retrieval pipeline pe **poora control** (hybrid, rerank, tuning)
- ✅ Direct SDK — latest features turant (`effort`, structured outputs, caching)
- ✅ Debug karna aasan — har step tumhara code

---

## Khud se pooch 🤔

1. LlamaIndex aur LangChain ka positioning farak kya hai?
2. LlamaIndex ka `Node` LangChain ke `Document` se richer kaise hai? Isse kya unlock hota hai?
3. Sentence-window retrieval LlamaIndex mein kaise implement hoti hai?
4. `response_mode` ke 5 options kya hain? `tree_summarize` kab use karoge?
5. LlamaIndex + LangGraph mix karne ka sahi tarika kya hai?
6. "5-line RAG" impressive hai par production-ready kyun nahi?
7. Tumhare paas Notion, Confluence, Google Drive, aur Slack se data hai. 50,000 documents. Complex retrieval chahiye. Kya use karoge?

<details>
<summary>Answer 7</summary>

**Ye LlamaIndex ka strongest use case hai** — 4 sources, bade scale pe, complex retrieval ke saath.

**Architecture:**

```python
# 1. INGESTION — LlamaIndex ka best part (4 connectors ready-made)
from llama_index.readers.notion import NotionPageReader
from llama_index.readers.confluence import ConfluenceReader
from llama_index.readers.google import GoogleDriveReader
from llama_index.readers.slack import SlackReader

docs = []
docs += NotionPageReader(...).load_data(...)
docs += ConfluenceReader(...).load_data(...)
docs += GoogleDriveReader(...).load_data(...)
docs += SlackReader(...).load_data(...)

# Source metadata add karo — filtering aur citations ke liye CRITICAL
for d in docs:
    d.metadata["source_system"] = detect_source(d)
    d.metadata["access_level"] = derive_acl(d)      # 🔒 security

# 2. PARSING — hierarchical (auto-merging ke liye)
from llama_index.core.node_parser import HierarchicalNodeParser
nodes = HierarchicalNodeParser.from_defaults(chunk_sizes=[2048, 512, 128]).get_nodes_from_documents(docs)

# 3. RETRIEVAL — hybrid + rerank
retriever = QueryFusionRetriever([vector_retriever, bm25_retriever],
                                 similarity_top_k=50, mode="reciprocal_rerank")
reranker = SentenceTransformerRerank(model="BAAI/bge-reranker-v2-m3", top_n=5)
```

**Par teen critical decisions jo LlamaIndex nahi karega — tumhe karne hain:**

**(a) Access control 🔒** — Slack DMs, private Notion pages, restricted Drive folders. Har chunk pe `access_level` metadata **mandatory** hai, aur retrieval mein filter **mandatory**. Ye Day 3 ka #1 enterprise audit failure hai. LlamaIndex ye tumhare liye nahi karega.

**(b) Incremental sync** — 50,000 docs ka full re-index roz nahi kar sakte. Tumhe chahiye: document hash tracking, deletion handling, aur per-source incremental sync. Ye custom code hai.

**(c) Source-specific chunking** — Slack threads aur Confluence pages ko same tarah chunk karna galat hai. Slack: 5-10 messages per chunk. Confluence: heading-based. **Per-source parser** chahiye.

**Meri final recommendation:**
- **LlamaIndex:** connectors + node parsing + retrieval composition ✅
- **Custom:** ACL enforcement, incremental sync, per-source chunking rules ✅
- **Direct SDK:** generation (latest features + prompt caching) ✅
- **Eval:** Din 1 se — 4 sources ke saath retrieval quality ka blame assign karna eval ke bina impossible hai

Aur ek practical note: **shuru mein sirf 1-2 sources** karo, pipeline theek karo, phir baaki add karo. Chaaron ek saath = 4 tarah ke failure modes ek saath debug karna.
</details>

---

**Next:** [`03-agent-sdks.md`](03-agent-sdks.md)
