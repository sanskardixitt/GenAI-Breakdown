# 00 — Framework Landscape Map ⭐⭐⭐

> **Ye file tumhare original sawaal ka jawab hai:** *"Production development mein kaunse frameworks use hote hain?"*
>
> **Pehle ek warning:** Framework landscape **fast badalta hai**. Isliye ye file **categories aur trade-offs** sikhati hai, specific version numbers nahi. Category samajh gaye to naya framework aane pe tum use 5 minute mein place kar paoge.

---

## Part 1: Pehla sach — "AI framework" ek category nahi hai

Log poochte hain "kaunsa AI framework use karun?" — ye sawaal hi galat hai, jaise "kaunsa web tool use karun?" poochna.

**AI stack mein 8 alag layers hain, har ek ka apna tooling hai:**

```
┌────────────────────────────────────────────────────────────────┐
│ 8. APP / UI          Streamlit, Next.js, Chainlit, AI SDK      │
├────────────────────────────────────────────────────────────────┤
│ 7. EVAL + OBS  ⭐    Langfuse, LangSmith, Braintrust, Ragas,   │
│                      Phoenix, Promptfoo, DeepEval               │
├────────────────────────────────────────────────────────────────┤
│ 6. ORCHESTRATION     LangGraph, Agent SDKs, CrewAI, AutoGen,   │
│                      Temporal, Prefect, custom                  │
├────────────────────────────────────────────────────────────────┤
│ 5. CONTEXT/RAG       LlamaIndex, LangChain, Haystack, custom   │
├────────────────────────────────────────────────────────────────┤
│ 4. DATA STORES       Postgres+pgvector, Qdrant, Weaviate,      │
│                      Pinecone, Milvus, Elastic, OpenSearch      │
├────────────────────────────────────────────────────────────────┤
│ 3. MODEL ACCESS      Provider SDKs, LiteLLM, OpenRouter        │
├────────────────────────────────────────────────────────────────┤
│ 2. MODELS            Claude, GPT, Gemini, Llama, Qwen, Mistral │
├────────────────────────────────────────────────────────────────┤
│ 1. SERVING (self-host) vLLM, SGLang, TGI, Ollama, TensorRT-LLM │
└────────────────────────────────────────────────────────────────┘
```

> 🔑 **Sahi sawaal:** *"Har layer pe main kya use karun?"*
>
> Aur zyadatar products ko layers 2, 3, 5, 6, 7 chahiye. Layer 1 sirf self-hosting mein.

---

## Part 2: Layer-by-layer — kya use hota hai

### Layer 2-3: Model access

| Option | Kab | Note |
|---|---|---|
| **Provider SDK direct** (`anthropic`, `openai`) | ⭐ **Default.** Ek provider, full features | Sabse kam abstraction, sabse zyada control, naye features turant milte hain |
| **LiteLLM** | Kai providers, unified interface chahiye | 100+ providers, ek API. Latest features mein lag ho sakta hai |
| **OpenRouter** | Model routing, fallbacks, ek billing | Hosted gateway |
| **Provider gateway** (Bedrock, Vertex, Foundry) | Enterprise cloud commitment | Feature parity vary karti hai |

> **Practical advice:** **Provider SDK se shuru karo.** "Multi-provider abstraction" pehle din se banana premature hai — aur wo abstraction hamesha latest features ko block karta hai (naya parameter aaya, abstraction mein nahi hai).
>
> LiteLLM tab lo jab tumhe **genuinely** kai providers chahiye (cost routing, fallback, ya customer choice).

### Layer 4: Vector/data stores

| Option | Kab |
|---|---|
| **Postgres + pgvector** ⭐ | **Default.** Already Postgres hai? Ek naya system mat lao |
| **Qdrant** | Dedicated vector DB, acha filtering, self-host friendly |
| **Weaviate** | Hybrid search built-in, GraphQL |
| **Pinecone** | Fully managed, zero ops |
| **Milvus** | Bahut bada scale (billions) |
| **Elasticsearch / OpenSearch** | Already hai? BM25 + vector dono milega |
| **Chroma / FAISS / LanceDB** | Prototype, local dev |

Detail: [`05-vector-databases.md`](05-vector-databases.md)

### Layer 5: Context / RAG

| Option | Kab |
|---|---|
| **Custom (khud likho)** ⭐ | Simple RAG. **~200 lines. Poora control.** |
| **LlamaIndex** | RAG-focused, bahut ready-made connectors aur retrievers |
| **LangChain** | Broad ecosystem, bahut integrations |
| **Haystack** | Production pipelines, modular, NLP-heritage |

Detail: [`02-llamaindex.md`](02-llamaindex.md), [`01-langchain-aur-langgraph.md`](01-langchain-aur-langgraph.md)

### Layer 6: Orchestration ⭐ (yahan sabse zyada confusion hai)

| Option | Model | Kab |
|---|---|---|
| **Custom loop** ⭐ | Tum sab likhte ho | Simple agents, learning, full control |
| **LangGraph** | Explicit graph + state | Complex flows, checkpointing, human-in-loop |
| **Provider Agent SDKs** | Harness provided, tum host karte ho | Provider-native, batteries included |
| **Managed agent platforms** | Provider harness **aur** infra | Zero ops, hosted sandbox, scheduled runs |
| **CrewAI** | Role-based crew | Fast prototyping, opinionated |
| **AutoGen / AG2** | Conversational agents | Research, experimentation |
| **Temporal / Prefect** | Durable workflow engine | Long-running, retry-critical, **AI-agnostic** |

Detail: [`03-agent-sdks.md`](03-agent-sdks.md), [`04-crewai-autogen-multiagent.md`](04-crewai-autogen-multiagent.md)

### Layer 7: Eval + Observability ⭐⭐ (sabse zyada skip kiya jaane wala)

| Option | Kya deta hai |
|---|---|
| **Langfuse** | Open-source tracing + evals + prompt management |
| **LangSmith** | LangChain ka, tracing + datasets + evals |
| **Braintrust** | Evals-first, CI integration |
| **Arize Phoenix** | Open-source, OpenTelemetry-native |
| **Ragas** | RAG-specific metrics |
| **Promptfoo** | Config-driven eval, CI friendly |
| **DeepEval** | Pytest-style LLM tests |
| **Custom + OpenTelemetry** | Poora control, vendor lock-in nahi |

Detail: [`07-evals-aur-observability.md`](07-evals-aur-observability.md)

### Layer 8: App / UI

| Option | Kab |
|---|---|
| **Streamlit / Gradio** | Internal tools, demos, fast |
| **Chainlit** | Chat UI specifically |
| **Next.js + Vercel AI SDK** | Production web app, streaming built-in |
| **FastAPI + custom frontend** | Full control |

---

## Part 3: 🔑 Framework ka value kya hai (aur kya nahi)

### Framework ye deta hai ✅

| Value | Detail |
|---|---|
| **Integrations** | 100+ document loaders, 50+ vector stores — ye genuinely time bachate hain |
| **Patterns** | Battle-tested implementations (RAG pipelines, agent loops) |
| **Standard vocabulary** | Team mein "retriever", "chain", "node" ka common meaning |
| **Ops features** | Checkpointing, retries, durable execution |
| **Speed to prototype** | 1 din vs 1 hafta |

### Framework ye NAHI deta ❌

| Non-value | Detail |
|---|---|
| **Better accuracy** | Framework tumhare retrieval ko better nahi banata |
| **Understanding** | Framework problem chhupa deta hai, solve nahi karta |
| **Production readiness** | Guards, evals, monitoring — wo tumhe likhne hain |
| **Debuggability** | Aksar **kam** deta hai (abstraction layers) |
| **Latest features** | Abstraction naye API features ko block karta hai |

### 🚩 Framework ki hidden cost

```
1. ABSTRACTION LEAK
   Kuch weird ho raha hai → framework ka source padhna padega
   → "Simple" cheez debug karne mein 3 ghante

2. VERSION CHURN
   Fast-moving frameworks mein breaking changes aam hain
   → Upgrade = ek din ka kaam

3. FEATURE LAG
   Provider ne naya feature diya → framework mein 2-8 hafte baad aayega
   → Ya tum escape hatch use karte ho (jo framework ka point kill karta hai)

4. PERFORMANCE
   Extra layers = extra latency, extra memory

5. LEARNING CURVE
   Framework seekhna ≠ AI engineering seekhna
```

---

## Part 4: Decision framework 🎯

### Ye 5 sawaal poocho

```
1. Kya main is layer ko 200 lines mein khud likh sakta hoon?
   HAAN → Khud likho. (RAG basic, agent loop — dono ~200 lines hain)

2. Kya mujhe 20+ integrations chahiye (loaders, stores, providers)?
   HAAN → Framework use karo (LlamaIndex/LangChain).

3. Kya mujhe durable execution chahiye (crash-resume, long-running)?
   HAAN → LangGraph ya Temporal.

4. Kya meri team ko ek shared vocabulary chahiye?
   HAAN → Ek framework standardize karo (consistency > optimality).

5. Kya main framework ka source padh sakta hoon jab kuch toote?
   NAHI → Chhota/simpler framework chuno, ya khud likho.
```

### Practical recommendation by stage

| Stage | Recommendation |
|---|---|
| **Seekhna** | ❌ **Koi framework nahi.** Sab kuch khud likho. Tabhi samjhoge. |
| **Prototype (1 hafta)** | ✅ Framework — jitni jaldi ho sake demo banao |
| **MVP (production v1)** | ⚠️ Hybrid — provider SDK + kuch framework pieces |
| **Scale** | ⚠️ Core custom, edges framework |
| **Enterprise** | ✅ Framework/platform (support, compliance, team consistency) |

> 🔑 **Ye repo ka framework advice, ek line mein:**
>
> **Pehle bina framework likho. Phir framework tab lo jab tumhe pata ho ki wo kaunsa specific problem solve kar raha hai.**
>
> Framework pehle lena = tumhe uske abstractions mein sochna padega, apne problem ke hisaab se nahi.

---

## Part 5: Ek "boring" production stack (jo actually kaam karta hai)

Ye ek real, defensible stack hai jise main recommend karunga ek naye product ke liye:

```
┌──────────────────────────────────────────────────────────┐
│ UI          : Next.js + Vercel AI SDK (streaming free)   │
│               ya Streamlit (internal tools)               │
├──────────────────────────────────────────────────────────┤
│ API         : FastAPI (Python) — async, typed, simple    │
├──────────────────────────────────────────────────────────┤
│ ORCHESTRATION: Custom agent loop (~200 lines)            │
│               → LangGraph JAB durable state chahiye      │
├──────────────────────────────────────────────────────────┤
│ RAG         : Custom pipeline                             │
│               - Parsing: Unstructured / Docling           │
│               - Chunking: recursive + structure-aware      │
│               - Hybrid: pgvector + BM25 (Postgres FTS)    │
│               - Rerank: bge-reranker (local) ya API       │
├──────────────────────────────────────────────────────────┤
│ MODEL       : Provider SDK direct (anthropic / openai)    │
│               + chhota model routing layer                │
├──────────────────────────────────────────────────────────┤
│ STORAGE     : Postgres (data + pgvector + FTS) ⭐        │
│               Redis (cache, queues)                       │
│               S3/R2 (files)                               │
├──────────────────────────────────────────────────────────┤
│ EVAL + OBS  : Langfuse (ya Phoenix) + custom eval script │
├──────────────────────────────────────────────────────────┤
│ DEPLOY      : Docker + Railway/Render/Fly/ECS             │
│               Background jobs: Celery / Temporal          │
└──────────────────────────────────────────────────────────┘
```

### Ye stack kyun?

| Choice | Justification |
|---|---|
| **Custom orchestration** | Agent loop 200 lines hai. Framework ka fayda < debug cost. |
| **Postgres for everything** | Ek DB. pgvector + full-text search dono. Ops simple. |
| **Provider SDK direct** | Naye features turant. Prompt caching, structured outputs, effort — sab available. |
| **Local reranker** | 200ms, free, koi extra vendor nahi |
| **Langfuse** | Open-source, self-host kar sakte ho, tracing + evals |
| **Custom eval script** | Framework se pehle apna eval likhna — samajh aata hai kya measure kar rahe ho |

> **"Boring" ek compliment hai.** Ye stack teen saal chalega. Har layer ko tum samajhte ho. Har layer replace kar sakte ho bina baaki toote.

---

## Part 6: Ek "framework-heavy" stack (kab sahi hai)

```
┌──────────────────────────────────────────────────────────┐
│ ORCHESTRATION: LangGraph (durable, checkpointed)          │
│ RAG          : LlamaIndex (100+ connectors)               │
│ MODEL        : LiteLLM (multi-provider routing)           │
│ EVAL/OBS     : LangSmith / Braintrust                     │
│ STORE        : Pinecone / Qdrant (managed)                │
└──────────────────────────────────────────────────────────┘
```

**Kab ye sahi hai:**
- ✅ Bahut saare data sources (Notion, Slack, Drive, Confluence, ...) — connectors ka value real hai
- ✅ Long-running workflows jinhe crash-resume chahiye
- ✅ Badi team, standard vocabulary chahiye
- ✅ Enterprise, jahan vendor support matter karta hai
- ✅ Speed to market > long-term control

---

## Part 7: 🚩 Framework anti-patterns

| Anti-pattern | Kyun bura |
|---|---|
| **Framework pehle, samajh baad mein** | Tum framework seekh rahe ho, AI engineering nahi |
| **Ek framework sab layers ke liye** | Har layer ka best tool alag hai |
| **Framework ke andar framework** | LangChain + LlamaIndex + CrewAI ek saath = debugging hell |
| **Multi-provider abstraction din 1 pe** | YAGNI. Aur naye features block hote hain |
| **Eval layer skip karna** | Sabse mehngi galti. Tum blind ho. |
| **Framework hype se choose karna** | GitHub stars ≠ tumhare use case ke liye fit |
| **Framework upgrade na karna** | Security fixes miss, phir bada painful jump |
| **Framework ko black box treat karna** | Jab toote, tum stuck ho |

---

## Part 8: Ek quick reference table

| Mujhe chahiye... | Use karo |
|---|---|
| Ek LLM call | Provider SDK |
| Multi-provider fallback | LiteLLM / OpenRouter |
| Simple RAG | Custom (~200 lines) |
| 50 data sources se RAG | LlamaIndex |
| Simple agent | Custom loop (~200 lines) |
| Agent with checkpointing/resume | LangGraph |
| Provider-native agent harness | Agent SDK |
| Zero-ops hosted agent + sandbox | Managed agent platform |
| Multi-agent, fast prototype | CrewAI |
| Durable long-running workflow | Temporal / LangGraph |
| Vector search | pgvector (default) / Qdrant |
| Hybrid search | pgvector + Postgres FTS, ya Weaviate/Elastic |
| Reranking | bge-reranker (local) / Cohere / Voyage |
| Tracing | Langfuse / Phoenix / LangSmith |
| RAG evals | Ragas + custom |
| CI evals | Promptfoo / DeepEval / Braintrust |
| Chat UI | Vercel AI SDK / Chainlit |
| Self-host model | vLLM |
| Local dev model | Ollama |
| Third-party tools | MCP servers |

---

## Khud se pooch 🤔

1. "Kaunsa AI framework use karun" ye sawaal galat kyun hai? Sahi sawaal kya hai?
2. AI stack ke 8 layers batao. Zyadatar products ko kaunse chahiye?
3. Framework ke 5 fayde aur 5 non-fayde batao.
4. Framework ki 5 hidden costs kya hain? "Feature lag" kyun hoti hai?
5. Framework choose karne ke 5 sawaal kya hain?
6. "Boring stack" mein Postgres sab kuch kyun kar raha hai? Ye acha decision kyun hai?
7. Multi-provider abstraction din 1 pe kyun premature hai?
8. Ek startup 2 mahine se LangChain + LlamaIndex + CrewAI ke saath struggle kar rahi hai, accuracy 60% hai. Tumhara advice?

<details>
<summary>Answer 8</summary>

**Diagnosis: teen frameworks = teen abstraction layers = koi bhi problem debug nahi ho sakti.** Aur 60% accuracy ka framework se koi lena-dena nahi hai — wo ek **retrieval/eval** problem hai.

**Mera advice, is order mein:**

**1. Ruko. Eval banao (Day 1 ka kaam).**
60% accuracy ka matlab hai unhe pata bhi nahi ki kahan toota hai. Ek 30-case golden set banao. Retrieval aur generation **alag** measure karo (Day 3 ka table). Bina iske har change ek guess hai.

Yeh probably reveal karega: Recall@5 ~0.55, generation fine. Matlab **retrieval problem hai, framework problem nahi.**

**2. Ek spike likho — bina framework ke, 200 lines mein.**
```python
# ~200 lines, ek file
def rag(query):
    q = rewrite(query)                                  # 10 lines
    bm25_hits = bm25.search(q, k=50)                    # 15 lines
    vec_hits = vecdb.search(embed(q), k=50)             # 10 lines
    merged = rrf([bm25_hits, vec_hits])                 # 10 lines
    top5 = reranker.rerank(q, merged)[:5]               # 10 lines
    return llm(RAG_PROMPT.format(context=top5, q=q))    # 15 lines
```
Isko golden set pe chalao. Meri prediction: **ye teen frameworks se better score karega**, kyunki isme hybrid + reranking hai jo unke stack mein shayad nahi hai.

**3. Frameworks ko selectively wapas laao — evidence ke saath.**
- LlamaIndex ke connectors chahiye? Sirf **loading** ke liye use karo, poore pipeline ke liye nahi.
- CrewAI — poocho: *"multi-agent kyun hai?"* 90% chance ye zaroori nahi hai (Day 4 ka lesson).
- LangChain — agar LangGraph ka durable state chahiye to rakho, warna nahi.

**4. Eval + tracing layer add karo (jo abhi missing hai).**
Langfuse ya Phoenix. Ye wo layer hai jo unhe 2 mahine pehle chahiye tha.

**Ek imaandaar baat unse:** 2 mahine "framework struggle" mein gaye kyunki wo **galat problem** solve kar rahe the. Unka problem retrieval quality tha; wo orchestration tooling debug kar rahe the. Ye ek bahut common trap hai.

**Expected outcome:** 1 hafte mein 60% → 80%+, aur ek stack jo wo samajhte hain.
</details>

---

**Next:** [`01-langchain-aur-langgraph.md`](01-langchain-aur-langgraph.md)
