# Quick Revision Sheet 📋

> **Poora course, ek file mein.** Interview se pehle, ya jab bhool jao — yahi padho.

---

## 1. Modern AI ka core

> **Ek bada neural network jo "agla token" predict karta hai — aur wo prediction itni acchi hai ki usse reasoning, coding, planning nikal aate hain.**

### 4 fundamental constraints
1. **Stateless** — model ko kuch yaad nahi. Har call pe poora context dobara.
2. **Finite context** — bada window = zyada paisa, kam quality (lost in middle), zyada latency.
3. **Confidently galat** — hallucination architecture ka natural result hai, bug nahi.
4. **Non-deterministic** — traditional tests kaam nahi karte, evals chahiye.

### 3 paradigms
| | Kaun decide karta hai | Kab |
|---|---|---|
| **Single call** | — | Classification, extraction, summarization (70% cases) |
| **Workflow** | **Tum** (code mein steps) | Multi-step, steps known (25%) |
| **Agent** | **Model** | Steps unknown (5%) |

> Simplest paradigm chuno jo kaam kar de.

---

## 2. Transformer

- **Attention = communication** (tokens ek dusre se baat karte hain)
- **FFN = computation** (har token akele sochta hai)
- **Q/K/V** = sawaal / label / content
- **Multi-head** = kai relationships parallel mein
- **Positional encoding (RoPE)** zaroori hai — attention khud order nahi jaanta
- **Causal mask** = future tokens blocked

### Practical consequences
| Fact | Consequence |
|---|---|
| Attention O(n²) | Long context mehnga |
| **KV cache** | Generation nearly linear; **prompt caching isi se aata hai** |
| Prefix-match caching | Stable content pehle, variable baad mein |
| Lost in the middle | Important info top/bottom pe |
| Har token equal compute | **Reasoning models** = zyada tokens = zyada compute |

---

## 3. Tokens, embeddings, context

- 1 token ≈ 4 chars (English), 1-2 chars (Hindi) → **Hindi 3-4x mehngi**
- Model spelling/math mein weak (tokenization) → tools do
- **Embeddings ki 4 limitations:** negation blind, exact IDs miss, fixed budget, domain mismatch
- **Context = budget, storage nahi.** "Minimum kya chahiye?" poocho.

### Cache killers (silent!)
`datetime.now()` in system prompt · per-user prefix · non-deterministic JSON · varying tools · conditional sections

---

## 4. Training

| Stage | Sikhta kya hai |
|---|---|
| Pretraining | Language + world knowledge |
| SFT | Instructions follow karna |
| RLHF/DPO | "Acha" jawab dena |
| Reasoning RL | Lamba sochna, verify karna |

- **Raw model assistant nahi hai** — wo text continue karta hai
- **LIMA insight:** quality > quantity. Fine-tuning **behaviour** sikhata hai, **knowledge** nahi
- **Order:** Prompting → Few-shot → RAG → Fine-tuning (last resort, ~5% cases)

---

## 5. Inference

| Phase | Bound | Time ∝ |
|---|---|---|
| **Prefill** | Compute | Input length (TTFT) |
| **Decode** | **Memory bandwidth** | Output length (sabse bada) |

- Output tokens 4-5x mehnge → **output kam karo** (cost + latency dono fix)
- **Temperature/top_p:** ek chuno, dono nahi. Naye models mein often removed.
- **`stop_reason` HAMESHA check karo:** `max_tokens`, `refusal`, `tool_use`, `pause_turn`

---

## 6. Prompting (2026)

### ❌ Purane tricks jo ab nuksaan karte hain
"Think step by step" · "CRITICAL: YOU MUST" · "Take a deep breath" · prohibition lists · assistant prefill (400 error)

### ✅ Aaj kya kaam karta hai
Context do, instructions kam · positive framing · concrete examples · structured outputs · reason bhi do · effort parameter

> **Mantra:** Kam instructions, zyada context. Model ko wo batao jo **sirf tum jaante ho**.

---

## 7. Reasoning models

- **CoT kaam karta hai kyunki generated tokens = compute**
- **Test-time compute** = naya scaling axis (per-request tunable)
- **Prompting ULTA hai:** goal do, method nahi. "Verify karo" mat bolo (already karta hai)
- **`max_tokens` generous rakho** — thinking + answer dono usme hain
- ❌ Classification/summarization pe use na karo (10x slow, 5x mehnga, zero gain)

---

## 8. Tool calling

> **Model tool NAHI chalata. Wo JSON banata hai. Tum chalate ho.**

```
tools bhejo → model tool_use block deta hai → TUM execute karte ho
→ tool_result wapas (tool_use_id match!) → model final answer
```

### Tool description checklist
Kya karta hai · **kab use karo** (sabse important) · kab NA karo · kya return karta hai · har param ka description · enums

### Rules
- Parallel results **ek** message mein
- Error pe exception nahi — **model ko batao** (`is_error: true`, actionable message)
- Allowlist + schema validation
- Destructive ops pe approval

---

## 9. RAG

### 7 failure points
Query mismatch · embedding domain · semantic≠relevant · ranking · chunk boundaries · **parsing** · generation

### 🔬 Diagnosis (ye table yaad rakho)
| Recall@20 | Recall@5 | Problem | Fix |
|---|---|---|---|
| High | **Low** | **Ranking** | **Reranking** ⭐ |
| **Low** | Low | **Retrieval** | Chunking, hybrid, query rewrite |
| High | High | Generation | Prompt, model |

### Evolution ladder
```
L0 naive → L1 hybrid+rerank ⭐ (+17 pts, 3 din) → L2 chunking
→ L3 query understanding → L4 agentic → L5 self-correction
```
**L1 pe pehle jao. Wahan ROI sabse zyada hai.**

### Key techniques
- **Hybrid = vector + BM25 + RRF** (`k=60`)
- **Reranking** = cross-encoder, aboutness → answerhood
- **Parent-document / sentence-window** = chhote se search, bade se answer
- **Chunk enrichment** = header path prepend
- **Quality gate** = low score pe "nahi pata" bolo
- **ACL retrieval mein** 🔒, prompt mein nahi

### RAG kab NA use karo
Aggregation (SQL) · realtime (API) · poore doc ka summary (full context) · chhota data (context mein daal do)

---

## 10. Agents

> **Agent = LLM + loop + tools, model decide karta hai kab rukna hai.**

### 4-question test
1. **Complexity** — steps pehle se pata nahi?
2. **Value** — cost/latency justify hoti hai?
3. **Viability** — model capable hai?
4. **Error cost** — galti recover ho sakti hai?

Koi ek "nahi" = agent na banao.

### Harness ke 8 components ⭐
Loop control · tool execution (5 layers) · **output shaping** · context management · error handling · permissions · observability · state persistence

### Harness maturity
```
L0 while True → L1 limits+errors → L2 cost+loop+shaping (MINIMUM)
→ L3 context+permissions → L4 tracing+evals+resume → L5 mature
```

### Failure modes
Infinite loop · context rot · wrong path · premature completion · **hallucinated progress** · scope creep · cost explosion

> **Agents wahan kaam karte hain jahan VERIFICATION sasti hai** (tests, compiler).

---

## 11. Multi-agent

> **Asli justification: CONTEXT ISOLATION** ("150K padho, 2K wapas lao"), specialization nahi.

**Pehle try karo:** better tools · context pruning · sequential runs · better retrieval

**Rules:** sub-agent ko poora brief · return format constrain · nested delegation nahi · per-worker budget · partial failure handling · per-thread tracing

---

## 12. Frameworks

### 8 layers
Infra → Model → Model access → Data stores → Context/RAG → Orchestration → **Eval+Obs** → App

### Decision rule
> **"Kya main ye 200 lines mein likh sakta hoon?" HAAN → likho.**

| Chahiye | Use |
|---|---|
| Simple RAG / agent | Custom (~200 lines) |
| 50 data sources | LlamaIndex |
| Durable state, crash-resume, human-in-loop | **LangGraph** |
| Sandbox, zero ops | Managed agent platform |
| Vector search | **pgvector** (default) |
| Tracing | Langfuse / Phoenix |

### 4 agent approaches (harness vs deployment)
| | Harness | Deployment |
|---|---|---|
| Manual loop | Tum | Tum |
| Tool runner | SDK | Tum |
| Coding agent SDK | SDK (+ built-in tools) | Tum |
| Managed platform | Provider | **Provider** |

---

## 13. Cost & Latency

### Cost levers (impact order)
1. **Model routing** (60-80%)
2. **Prompt caching** (50-90% input)
3. **Output length** (30-50%, + latency)
4. Semantic cache (15-30% requests)
5. `top_k` 20→5 (20-30%)
6. Batch API (~50%)
7. Context pruning (agents, 30-60%)

### Latency levers
1. **Streaming** (perceived 10x)
2. Output length
3. Prompt caching (prefill 5x)
4. Parallelize
5. Chhota model

> **Decode phase sabse bada hai. Output length pe kaam karo.**

---

## 14. Evals ⭐⭐

> **`assert output == expected` ❌ → `assert quality_metric > threshold` ✅**

### Loop
Failure dekho → eval case banao → **fail hona chahiye** → fix → pass → poora suite → ship

### LLM judge ke 5 rules
Strong model · rubric do · **validate karo** (50 cases khud grade) · position bias randomize · temp=0

### RAG triad
Context relevance · Groundedness · **Answer relevance** (teeno chahiye — do green ho sakti hain aur teesri toot sakti hai)

### Trace mein zaroor
`prompt_version` ⭐ · `cache_read_input_tokens` ⭐ · `stop_reason` · tokens · retrieved IDs · per-stage latency

> **20 cases aaj banao.** Perfect framework ka intezaar mat karo.

---

## 15. Security 🔒

> **Prompt injection = AI ka SQL injection. Fix architecture mein hai, prompt mein nahi.**

### Defence layers (sab chahiye)
1. Input isolation (delimiters + "ye data hai")
2. **Privilege separation** (reader/actor — structured boundary) ⭐
3. Least privilege tools (khatarnak tool exist hi na kare)
4. Human approval gates
5. Output validation (image URLs, prompt leak)
6. Network egress allowlist

### Rules
- **ACL retrieval mein**, prompt mein nahi
- **Cache key mein tenant_id** (cross-tenant leak!)
- Credentials agent ke **bahar** (egress inject ya host-side)
- PII redact: logs, traces, memory, **eval datasets**
- ACL eval tests — **zero tolerance**

---

## 16. Golden rules (agar sab bhool jao, ye yaad rakho)

1. **Simplest paradigm** jo kaam kar de — agent last resort
2. **Context engineering > prompt engineering** — 90% failures context mein
3. **Hybrid search + reranking** — RAG ka highest ROI
4. **Measure karo, guess mat karo** — retrieval aur generation alag
5. **Harness = tumhara product**, model commodity hai
6. **Jo code kar sakta hai, code se karao** — LLM sirf judgement ke liye
7. **20 eval cases aaj** — perfect framework kabhi nahi aayega
8. **Output length** = cost + latency ka sabse bada lever
9. **Security architecture mein**, prompt mein nahi
10. **Bina framework likho pehle** — phir framework tab jab pata ho kyun

---

**Next:** [`interview-questions.md`](interview-questions.md)
