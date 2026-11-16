# Lab 6 — Capstone: Agentic RAG App 🏆

> **Day 7.** Time: 3 ghante.
> **Goal:** Sab kuch jodkar ek working, measured, production-shaped app banana.

---

## Kya banayenge

Ek **document Q&A assistant** jo:
- ✅ Hybrid RAG use karta hai (BM25 + vector + reranking)
- ✅ Router se decide karta hai: RAG, aggregation, ya direct answer
- ✅ Agentic mode complex queries ke liye
- ✅ Citations deta hai aur validate karta hai
- ✅ Cost/latency track karta hai
- ✅ Eval suite ke saath aata hai
- ✅ Guards ke saath (limits, allowlist, quality gate)

**Ye tumhara portfolio project ban sakta hai.**

---

## Architecture

```
                    USER QUERY
                        │
              ┌─────────▼──────────┐
              │  Query Rewrite     │  (multi-turn ke liye)
              └─────────┬──────────┘
              ┌─────────▼──────────┐
              │  Semantic Cache    │──HIT──→ return
              └─────────┬──────────┘
              ┌─────────▼──────────┐
              │      ROUTER        │
              └──┬──────┬───────┬──┘
        simple   │      │       │  complex
        ┌────────┘      │       └────────┐
        ▼               ▼                ▼
  ┌───────────┐  ┌──────────┐   ┌──────────────┐
  │ Classic   │  │ Direct   │   │ Agentic RAG  │
  │ RAG       │  │ answer   │   │ (agent loop) │
  └─────┬─────┘  └────┬─────┘   └──────┬───────┘
        └─────────────┴────────────────┘
                      │
            ┌─────────▼──────────┐
            │ Citation validate  │
            └─────────┬──────────┘
            ┌─────────▼──────────┐
            │  Trace + metrics   │
            └─────────┬──────────┘
                      ▼
                   ANSWER
```

---

## Step 1: Project structure (15 min)

```
capstone/
├── .env
├── config.py
├── corpus/                    # tumhare documents (PDF/MD/TXT)
├── src/
│   ├── ingest.py              # parse → chunk → enrich → index
│   ├── retrieve.py            # hybrid + rerank
│   ├── router.py              # intent classification
│   ├── rag.py                 # classic RAG
│   ├── agent.py               # agentic mode (Lab 4 ka harness)
│   ├── cache.py               # semantic cache
│   └── trace.py               # observability
├── evals/
│   ├── datasets/golden.jsonl
│   ├── run.py                 # Lab 5 se
│   └── baseline.json
├── app.py                     # FastAPI ya Streamlit
└── README.md
```

```python
# config.py
from pydantic_settings import BaseSettings

class Config(BaseSettings):
    # Models
    model: str = "claude-opus-5"
    fast_model: str = "claude-haiku-4-5"
    embed_model: str = "BAAI/bge-m3"              # multilingual (Hindi!)
    rerank_model: str = "BAAI/bge-reranker-v2-m3"

    # Retrieval
    chunk_size: int = 800
    chunk_overlap: int = 100
    retrieve_k: int = 30
    rerank_k: int = 5
    min_relevance: float = 0.20

    # Generation
    max_tokens: int = 2048
    effort: str = "medium"

    # Agent
    agent_max_steps: int = 10
    agent_max_cost: float = 0.50

    # Cache
    semantic_cache_threshold: float = 0.97
    cache_ttl_seconds: int = 3600

    prompt_version: str = "v1"

cfg = Config()
```

---

## Step 2: Ingestion (30 min)

```python
# src/ingest.py
import re, hashlib, json
from pathlib import Path
from sentence_transformers import SentenceTransformer
from config import cfg

embedder = SentenceTransformer(cfg.embed_model)

def parse_markdown(path: Path):
    """Header path track karo — enrichment aur metadata ke liye."""
    text = path.read_text(encoding="utf-8")
    sections, stack, buf = [], [], []

    for line in text.split("\n"):
        m = re.match(r'^(#{1,4})\s+(.+)$', line)
        if m:
            if buf:
                sections.append({"path": " > ".join(stack), "text": "\n".join(buf)})
                buf = []
            level, title = len(m.group(1)), m.group(2).strip()
            stack = stack[:level-1] + [title]
        else:
            buf.append(line)
    if buf:
        sections.append({"path": " > ".join(stack), "text": "\n".join(buf)})
    return [s for s in sections if s["text"].strip()]


def chunk_text(text, size=None, overlap=None):
    """Recursive split — separators ki hierarchy."""
    size = size or cfg.chunk_size
    overlap = overlap or cfg.chunk_overlap
    if len(text) <= size * 4:                    # rough char→token
        return [text]

    chunks, start = [], 0
    seps = ["\n\n", "\n", ". ", " "]
    while start < len(text):
        end = min(start + size * 4, len(text))
        if end < len(text):
            for sep in seps:                      # sahi jagah todo
                idx = text.rfind(sep, start + size * 2, end)
                if idx > 0:
                    end = idx + len(sep)
                    break
        chunks.append(text[start:end].strip())
        start = end - overlap * 4
    return [c for c in chunks if c]


def ingest(corpus_dir="corpus", out="index.json"):
    chunks = []
    for path in Path(corpus_dir).rglob("*.md"):
        doc_title = path.stem.replace("_", " ").title()
        for section in parse_markdown(path):
            for i, piece in enumerate(chunk_text(section["text"])):
                # ⭐ ENRICHMENT — header path prepend
                enriched = f"[{doc_title} | {section['path']}]\n{piece}"
                chunks.append({
                    "id": hashlib.md5(
                        f"{path}:{section['path']}:{i}".encode()).hexdigest()[:12],
                    "doc": str(path.relative_to(corpus_dir)),
                    "doc_title": doc_title,
                    "section_path": section["path"],
                    "text": piece,
                    "enriched": enriched,
                    "embed_model": cfg.embed_model,       # ⭐ rebuild ke liye
                })

    print(f"Embedding {len(chunks)} chunks...")
    embs = embedder.encode([c["enriched"] for c in chunks],
                           normalize_embeddings=True, batch_size=32,
                           show_progress_bar=True)
    for c, e in zip(chunks, embs):
        c["embedding"] = e.tolist()

    Path(out).write_text(json.dumps(chunks), encoding="utf-8")
    print(f"✅ Indexed {len(chunks)} chunks → {out}")
    return chunks

if __name__ == "__main__":
    ingest()
```

---

## Step 3: Retrieval (30 min)

```python
# src/retrieve.py
import json, re
import numpy as np
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer, CrossEncoder
from config import cfg

class Retriever:
    def __init__(self, index_path="index.json"):
        self.chunks = json.loads(open(index_path, encoding="utf-8").read())
        self.embeddings = np.array([c.pop("embedding") for c in self.chunks])
        self.by_id = {c["id"]: c for c in self.chunks}
        self.embedder = SentenceTransformer(cfg.embed_model)
        self.reranker = CrossEncoder(cfg.rerank_model)
        self.bm25 = BM25Okapi([self._tok(c["enriched"]) for c in self.chunks])

    @staticmethod
    def _tok(t): return re.findall(r'\w+', t.lower())

    def _vector(self, query, k):
        q = self.embedder.encode([query], normalize_embeddings=True)[0]
        scores = self.embeddings @ q
        return [self.chunks[i]["id"] for i in np.argsort(scores)[::-1][:k]]

    def _bm25(self, query, k):
        scores = self.bm25.get_scores(self._tok(query))
        return [self.chunks[i]["id"] for i in np.argsort(scores)[::-1][:k]]

    @staticmethod
    def _rrf(lists, kc=60):
        s = {}
        for lst in lists:
            for rank, cid in enumerate(lst, 1):
                s[cid] = s.get(cid, 0) + 1/(kc + rank)
        return sorted(s, key=s.get, reverse=True)

    def search(self, query, k=None, rerank=True):
        k = k or cfg.rerank_k
        rk = cfg.retrieve_k

        fused = self._rrf([self._vector(query, rk), self._bm25(query, rk)])
        candidates = [self.by_id[cid] for cid in fused[:rk]]

        if not rerank:
            return [(c, 1.0) for c in candidates[:k]]

        scores = self.reranker.predict([[query, c["enriched"]] for c in candidates])
        ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
        return [(c, float(s)) for c, s in ranked[:k]]
```

---

## Step 4: Router + RAG (30 min)

```python
# src/router.py
from common import client, text_of
from config import cfg

ROUTER_PROMPT = """Query ko classify karo:

- "lookup"      : ek specific fact/policy dhundhna (documents mein hoga)
- "complex"     : multi-hop, comparison, ya multi-part sawaal
- "aggregation" : count/sum/average/ranking (documents se nahi ho sakta)
- "chitchat"    : greeting, thanks, ya general baat

Query: {query}

Sirf ek word reply karo."""

def route(query: str) -> str:
    r = client.messages.create(
        model=cfg.fast_model, max_tokens=10,
        messages=[{"role": "user", "content": ROUTER_PROMPT.format(query=query)}],
    )
    intent = text_of(r).strip().lower()
    return intent if intent in {"lookup","complex","aggregation","chitchat"} else "lookup"
```

```python
# src/rag.py
import re
from common import client, text_of
from config import cfg

RAG_SYSTEM = """Tum ek document-grounded assistant ho.

RULES:
1. SIRF diye gaye documents se answer do. General knowledge use na karo.
2. Har claim ke saath source cite karo: [Doc 1], [Doc 2]
3. Agar documents mein answer nahi hai: "Diye gaye documents mein is sawaal
   ka jawab nahi mila." Guess na karo, deflect na karo.
4. Agar documents contradict karein, dono positions batao sources ke saath.
5. Concise raho — 2-4 sentences jab tak detail na maangi jaaye.
6. Jab sawaal ek specific value maange (number, date, naam), pehle wo value
   do, phir context.
"""

def format_docs(results):
    return "\n\n".join(
        f"[Doc {i+1}] (source: {c['doc']} | {c['section_path']})\n{c['text']}"
        for i, (c, _) in enumerate(results)
    )

def validate_citations(answer, n_docs):
    cited = {int(m) for m in re.findall(r'\[Doc (\d+)\]', answer)}
    invalid = cited - set(range(1, n_docs + 1))
    has_answer = "nahi mila" not in answer.lower()
    return {"invalid_citations": list(invalid),
            "uncited_answer": has_answer and not cited}

def classic_rag(query, retriever):
    results = retriever.search(query)

    # Quality gate
    if not results or results[0][1] < cfg.min_relevance:
        return {"answer": "Diye gaye documents mein is sawaal ka jawab nahi mila.",
                "sources": [], "gated": True}

    r = client.messages.create(
        model=cfg.model, max_tokens=cfg.max_tokens,
        system=RAG_SYSTEM,
        messages=[{"role": "user",
                   "content": f"<documents>\n{format_docs(results)}\n</documents>\n\n"
                              f"Sawaal: {query}"}],
    )
    answer = text_of(r)
    return {
        "answer": answer,
        "sources": [{"id": c["id"], "doc": c["doc"], "section": c["section_path"],
                     "score": round(s, 3)} for c, s in results],
        "validation": validate_citations(answer, len(results)),
        "usage": {"in": r.usage.input_tokens, "out": r.usage.output_tokens},
        "gated": False,
    }
```

---

## Step 5: Agentic mode (30 min)

```python
# src/agent.py
# Lab 4 ka harness use karo, search ko tool bana ke

from lab4b_harness import Agent, AgentConfig, Tool

def make_agentic_rag(retriever):
    def search_docs(query: str) -> str:
        results = retriever.search(query, k=4)
        if not results:
            return "Kuch nahi mila. Alag phrasing try karo."
        return "\n\n".join(
            f"[{c['doc']} | {c['section_path']}] (score {s:.2f})\n{c['text'][:600]}"
            for c, s in results
        )

    tools = [Tool(
        name="search_docs",
        description=(
            "Company documentation mein semantic + keyword search.\n\n"
            "KAB USE KARO: jab answer documents pe depend karta ho.\n"
            "TIP: pehli search kaafi na ho to ALAG phrasing se dobara search karo. "
            "Multi-part sawaal ke liye alag-alag searches karo.\n"
            "RETURNS: top 4 excerpts source aur relevance score ke saath."
        ),
        input_schema={"type": "object",
                      "properties": {"query": {"type": "string",
                                     "description": "Natural language search query"}},
                      "required": ["query"]},
        fn=search_docs,
    )]

    return Agent(tools, AgentConfig(max_steps=cfg.agent_max_steps,
                                    max_cost_usd=cfg.agent_max_cost), verbose=False)
```

---

## Step 6: Orchestrator (20 min)

```python
# src/pipeline.py
import time, uuid
from router import route
from rag import classic_rag
from agent import make_agentic_rag
from cache import SemanticCache
from config import cfg

class Pipeline:
    def __init__(self, retriever):
        self.retriever = retriever
        self.agent = make_agentic_rag(retriever)
        self.cache = SemanticCache(threshold=cfg.semantic_cache_threshold)

    def query(self, q: str, history=None):
        trace = {"trace_id": str(uuid.uuid4()), "query": q,
                 "prompt_version": cfg.prompt_version, "t0": time.time()}

        # 1. Cache
        if hit := self.cache.get(q):
            trace["path"] = "cache"
            trace["latency_s"] = round(time.time() - trace["t0"], 2)
            return {**hit, "trace": trace}

        # 2. Route
        intent = route(q)
        trace["intent"] = intent

        # 3. Dispatch
        if intent == "chitchat":
            result = {"answer": "Main aapke documents ke baare mein sawaalon ka "
                                "jawab de sakta hoon. Kya poochhna chahenge?",
                      "sources": []}
        elif intent == "aggregation":
            result = {"answer": "Ye sawaal counting/aggregation maangta hai, jo "
                                "document search se reliably nahi ho sakta. "
                                "Kripya structured data source use karein.",
                      "sources": []}
        elif intent == "complex":
            r = self.agent.run(q)
            result = {"answer": r.output or "Answer generate nahi kar paya.",
                      "sources": [], "agent_steps": r.steps, "agent_cost": r.cost}
        else:
            result = classic_rag(q, self.retriever)

        trace["latency_s"] = round(time.time() - trace["t0"], 2)
        self.cache.set(q, result)
        return {**result, "trace": trace}
```

---

## Step 7: Eval (25 min) ⭐

```python
# evals/golden.jsonl — apne corpus ke liye 25+ cases banao
# Mix karo: lookup, complex, aggregation, edge cases

# run_evals.py
from evals.run import run_evals
from src.retrieve import Retriever
from src.pipeline import Pipeline

retriever = Retriever()
pipe = Pipeline(retriever)

def eval_pipeline(query):
    r = pipe.query(query)
    retrieved = [{"id": s["id"], "text": ""} for s in r.get("sources", [])]
    return r["answer"], retrieved

run_evals("evals/datasets/golden.jsonl", eval_pipeline, "evals/baseline.json")
```

### Experiments zaroor chalao

```python
for name, kwargs in [
    ("baseline",     dict()),
    ("no_rerank",    dict(rerank=False)),
    ("topk_10",      dict(k=10)),
    ("no_enrich",    dict(use_enriched=False)),
    ("fast_model",   dict(model=cfg.fast_model)),
]:
    ...
```

**Ye table tumhara deliverable hai** — ye prove karta hai ki tumne engineering ki, guessing nahi.

---

## Step 8: UI (20 min)

```python
# app.py
import streamlit as st
from src.retrieve import Retriever
from src.pipeline import Pipeline

st.set_page_config(page_title="Doc Assistant", page_icon="📚")

@st.cache_resource
def load():
    return Pipeline(Retriever())

pipe = load()

st.title("📚 Document Assistant")

if "messages" not in st.session_state:
    st.session_state.messages = []

for m in st.session_state.messages:
    with st.chat_message(m["role"]):
        st.markdown(m["content"])
        if m.get("sources"):
            with st.expander(f"📎 {len(m['sources'])} sources"):
                for s in m["sources"]:
                    st.caption(f"**{s['doc']}** · {s['section']} · score {s['score']}")

if q := st.chat_input("Apna sawaal poochhein..."):
    st.session_state.messages.append({"role": "user", "content": q})
    with st.chat_message("user"): st.markdown(q)

    with st.chat_message("assistant"):
        with st.spinner("Soch raha hoon..."):
            r = pipe.query(q)
        st.markdown(r["answer"])

        if r.get("sources"):
            with st.expander(f"📎 {len(r['sources'])} sources"):
                for s in r["sources"]:
                    st.caption(f"**{s['doc']}** · {s['section']} · score {s['score']}")

        t = r["trace"]
        st.caption(f"⚡ {t['latency_s']}s · 🔀 {t.get('intent','-')} · "
                   f"🆔 {t['trace_id'][:8]}")

    st.session_state.messages.append({"role": "assistant", "content": r["answer"],
                                      "sources": r.get("sources", [])})
```

```powershell
streamlit run app.py
```

---

## ✅ Capstone checklist

**Core:**
- [ ] Ingestion pipeline (parse → chunk → enrich → embed → index)
- [ ] Hybrid retrieval (BM25 + vector + RRF)
- [ ] Reranking (cross-encoder)
- [ ] Router (intent classification)
- [ ] Classic RAG with citations
- [ ] Agentic mode for complex queries
- [ ] Quality gate (low score pe abstain)
- [ ] Citation validation

**Production shape:**
- [ ] Semantic cache
- [ ] Tracing (trace_id, latency, intent, prompt_version)
- [ ] Config centralized aur environment-driven
- [ ] Agent guards (steps, cost)
- [ ] Error handling har stage pe

**Quality:**
- [ ] **Eval suite (25+ cases)** ⭐
- [ ] Baseline recorded
- [ ] Experiment matrix chalayi (no_rerank, topk, model)
- [ ] Category-wise breakdown

**Extras:**
- [ ] Streamlit UI
- [ ] README with architecture diagram aur eval results

---

## 🏆 Ise portfolio project banao

README mein ye zaroor daalo:

```markdown
## Results

| Configuration      | Recall@5 | Groundedness | P95 Latency | Cost/query |
|--------------------|----------|--------------|-------------|------------|
| Vector only        |   0.68   |     3.9      |    4.1s     |  $0.008    |
| + Hybrid (RRF)     |   0.81   |     4.2      |    4.3s     |  $0.008    |
| + Reranking        |   0.89   |     4.4      |    4.6s     |  $0.008    |
| + Chunk enrichment |   0.92   |     4.5      |    4.6s     |  $0.008    |

## Key engineering decisions
- **Hybrid search** because pure vector missed exact identifiers (E4471 class of queries)
- **Reranking** because Recall@20 was 0.95 but Recall@5 only 0.68 — a ranking problem, not retrieval
- **Router** because aggregation queries produce confidently wrong answers via RAG
- **Quality gate** because "I don't know" beats a plausible fabrication
```

> **Ye README hiring managers ko dikhata hai ki tum measure karte ho, guess nahi.** Wahi asli signal hai.

---

## 🎉 Course complete!

Tumne 7 din mein:
- Transformer se lekar production deployment tak poora stack samjha
- Framework-free code likha — isliye har framework tumhe transparent hai
- Ek measured, production-shaped app banaya

**Next:** [`../07-revision/quick-revision-sheet.md`](../07-revision/quick-revision-sheet.md)
