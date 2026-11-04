# 06 — Serving aur Inference Stack

> **Is file ka goal:** Self-hosting ka landscape samajhna — **aur ye jaanna ki tumhe shayad iski zaroorat nahi hai.** Par jab zaroorat ho, tab clarity ho.

---

## Part 1: Pehla sach — 95% products ko self-hosting nahi chahiye

```
Sawaal: "Kya main model self-host karun?"
Default answer: NAHI.
```

**Kyun:**

| Reality | Detail |
|---|---|
| **GPU ops ek full-time job hai** | Driver issues, OOM, batching tuning, monitoring, upgrades |
| **API sasta hai chhote-medium volume pe** | Ek A100 ~$2/hr = $1,440/month, 24×7. Kya tum us capacity ka use kar rahe ho? |
| **Frontier models self-host nahi hote** | Best models API-only hain. Open models strong hain par ek gap hai. |
| **Utilization problem** | Traffic spiky hai. GPU idle mein bhi paisa khaata hai. |
| **Feature lag** | Prompt caching, structured outputs, server-side tools — ye sab API features hain |

---

## Part 2: Kab self-hosting genuinely sahi hai

| Reason | Detail |
|---|---|
| **Data privacy hard requirement** | Healthcare, defence, kuch financial — data bahar nahi jaa sakta. Ye #1 valid reason hai. |
| **Volume genuinely bada** | Millions of requests/day. Tab GPU economics jeet jaati hai. |
| **Custom fine-tuned model** | Tumhara LoRA/fine-tune serve karna hai |
| **Latency floor** | Network hop hataana hai (rare — model inference network se zyada time leta hai) |
| **Air-gapped environment** | Internet hi nahi hai |
| **Cost predictability** | Fixed GPU cost vs variable API bill (accounting reason) |

### Break-even math (roughly)

```
API cost:  1M requests × 2K tokens avg × $3/M input ≈ $6,000/month
           (plus output tokens, jo mehnge hote hain)

Self-host: 2× A100 (redundancy) = ~$3,000/month
           + engineer time (0.3 FTE = ~$4,000/month)
           = ~$7,000/month

→ Break-even ~1-2M requests/month, AUR tumhe engineer chahiye
```

> **Insight:** Break-even sirf compute pe nahi hota — **engineer time** dominant cost hai. Wo aksar bhula diya jaata hai.

---

## Part 3: Serving engines

### vLLM ⭐ (de facto standard)

```bash
pip install vllm
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 4 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90
# OpenAI-compatible API on :8000
```

**Killer feature: PagedAttention**

Recall karo (Day 1, file 02) — KV cache GPU memory khaata hai. Traditional serving mein har request ke liye **contiguous** memory reserve hoti thi:

```
❌ Traditional:
   Request A: 2048 tokens reserve, 300 use → 1748 WASTED
   Request B: 2048 reserve, 1900 use → 148 wasted
   → 60-80% memory waste, fragmentation

✅ PagedAttention (OS virtual memory jaisa):
   Memory ko fixed "pages" (16 tokens) mein baato
   Jab zaroorat ho tab allocate karo
   → 96%+ utilization
   → 2-4x zyada concurrent requests, SAME GPU pe
```

**Aur features:**
- **Continuous batching** — naya request aate hi batch mein add ho jaata hai (poora batch khatam hone ka intezaar nahi)
- **Prefix caching** — shared prefix ka KV cache reuse (system prompt!)
- **Speculative decoding** — chhota "draft" model tokens guess kare, bada verify kare
- **Quantization** — AWQ, GPTQ, FP8
- **Multi-LoRA** — ek base model, kai LoRA adapters swap

> **Dekha?** Day 1 ka KV cache concept yahan direct production optimization ban gaya. **Foundations useless nahi hote.**

### Doosre engines

| Engine | Note |
|---|---|
| **SGLang** | vLLM ka competitor. RadixAttention (better prefix caching), structured output mein strong |
| **TensorRT-LLM** | NVIDIA ka. Sabse fast NVIDIA GPUs pe, par compilation step aur complexity |
| **TGI** (HuggingFace) | Mature, HF ecosystem ke saath integrated |
| **llama.cpp** | CPU + consumer GPU, GGUF format. Edge/local ke liye |
| **Ollama** | llama.cpp ke upar friendly wrapper. **Local dev ke liye best** |
| **LM Studio** | GUI, non-developers ke liye |

### Quick pick

```
Production GPU serving        → vLLM (ya SGLang)
Absolute max NVIDIA perf      → TensorRT-LLM (complexity ke saath)
Local dev / laptop            → Ollama
CPU / edge / embedded         → llama.cpp
```

---

## Part 4: Quantization — model chhota karo

Model weights ki precision kam karo:

| Format | Bits | Size (70B model) | Quality loss |
|---|---|---|---|
| FP16 / BF16 | 16 | ~140 GB | Baseline |
| **FP8** | 8 | ~70 GB | Minimal (modern GPUs pe) |
| **INT8** | 8 | ~70 GB | Very small |
| **AWQ / GPTQ (4-bit)** | 4 | ~35 GB | Small (2-5%) |
| GGUF Q4_K_M | ~4.5 | ~40 GB | Small |
| 2-bit | 2 | ~18 GB | Noticeable |

### Practical guidance

```
Production, quality matters      → FP8 (agar GPU support kare) ya INT8
Cost-sensitive, quality flexible → AWQ 4-bit
Laptop / local                   → GGUF Q4_K_M
Research / max quality           → BF16
```

> ⚠️ **Quantization quality loss task-dependent hai.** Simple tasks pe 4-bit almost identical hota hai; complex reasoning pe noticeable drop. **Apne eval set pe test karo**, benchmark blindly na maano.

---

## Part 5: GPU sizing

### Memory requirement

```
Total VRAM ≈ (Model weights) + (KV cache) + (activations + overhead)

Model weights = params × bytes_per_param
  70B @ FP16  = 140 GB
  70B @ FP8   = 70 GB
  70B @ 4-bit = 35 GB

KV cache = 2 × layers × heads × head_dim × seq_len × batch × bytes
  → Ye SEQ_LEN aur BATCH ke saath scale karta hai
  → Long context + high concurrency = KV cache weights se BADA ho sakta hai!
```

### GPU reference

| GPU | VRAM | Kya chalega |
|---|---|---|
| RTX 4090 | 24 GB | 7-13B FP16, 30B 4-bit |
| A100 40GB | 40 GB | 13B FP16, 70B 4-bit |
| A100 80GB | 80 GB | 70B FP8, 34B FP16 |
| H100 80GB | 80 GB | Same, ~2-3x faster |
| H200 / B200 | 141+ GB | 70B FP16 comfortably |

**Multi-GPU:**
- **Tensor parallelism (TP)** — ek layer ko GPUs mein baato. Fast interconnect chahiye (NVLink).
- **Pipeline parallelism (PP)** — layers ko GPUs mein baato. Slower, par network pe chalta hai.

---

## Part 6: Gateways aur routing

Ye layer **API users ke liye bhi** relevant hai — self-hosting se independent.

### LiteLLM

```python
from litellm import completion

# Ek interface, 100+ providers
response = completion(model="anthropic/claude-opus-5", messages=[...])
response = completion(model="openai/gpt-...",          messages=[...])
response = completion(model="ollama/llama3.3",         messages=[...])
```

**LiteLLM Proxy** (server mode) deta hai:
- Unified API
- **Virtual keys** with budgets per team/user
- Rate limiting
- Fallbacks aur retries
- Cost tracking
- Caching
- Logging/tracing hooks

**Kab useful:** Multi-team org jahan tumhe central control chahiye — kaun kitna kharch kar raha hai, kis model pe.

**⚠️ Trade-off:** Abstraction naye provider features ko lag karta hai. Agar tumhe latest features chahiye (naye thinking params, structured output shapes), gateway beech mein aa sakta hai. Escape hatch rakho.

### OpenRouter

Hosted gateway — ek API key se 200+ models. Fallbacks, cost routing.
**Kab:** Prototyping, ya jab customers ko model choice deni ho.

---

## Part 7: Model routing (cost optimization) 🔥

Ye **API users ke liye sabse impactful** optimization hai:

```python
def route(task, complexity=None):
    complexity = complexity or classify_complexity(task)   # chhota fast model

    return {
        "trivial": ("haiku-class",  {"effort": None}),      # classification, extraction
        "simple":  ("sonnet-class", {"effort": "low"}),     # summarization, drafting
        "medium":  ("sonnet-class", {"effort": "medium"}),
        "hard":    ("opus-class",   {"effort": "high"}),    # analysis, debugging
        "extreme": ("opus-class",   {"effort": "xhigh"}),   # multi-file refactor
    }[complexity]
```

**Typical impact:** 60-80% cost reduction, accuracy loss ~0-2%.

**Kaise implement karo:**
1. **Classifier** — ek chhota fast model, ya simple heuristics (query length, keywords, task type)
2. **Measure karo** — har tier ka accuracy apne eval set pe. Agar chhota model 93% de raha hai vs bade ka 95%, wo 2% ka cost kya hai?
3. **Escalation path** — agar chhota model fail kare (low confidence, validation error), bade pe retry karo

```python
def with_escalation(task):
    result = call(cheap_model, task)
    if not validates(result) or result.confidence < 0.7:
        result = call(strong_model, task)      # escalate
    return result
```

> **Ye "cascade" pattern bahut powerful hai** — 80% requests sasti, 20% mehngi, overall accuracy strong model ke barabar.

---

## Part 8: Ek self-hosting checklist

Agar self-host kar hi rahe ho:

- [ ] **Load test karo** — realistic concurrency pe throughput/latency measure karo
- [ ] **Continuous batching on** (vLLM default)
- [ ] **Prefix caching on** — system prompt shared hai, bada win
- [ ] **`--max-model-len` reasonable** — bahut bada = KV cache memory waste
- [ ] **`--gpu-memory-utilization` tune karo** (0.85-0.92 typical)
- [ ] **Quantization test karo apne eval pe**, benchmark pe nahi
- [ ] **Monitoring:** GPU util, memory, queue depth, TTFT, TPOT, throughput
- [ ] **Autoscaling** — GPU nodes scale karna slow hai (model load time!). Warm pool rakho.
- [ ] **Fallback to API** — GPU down ho to API pe fail over karo
- [ ] **Cost tracking** — GPU-hours vs requests served

### Metrics jo matter karti hain

| Metric | Kya |
|---|---|
| **TTFT** | Time To First Token — prefill speed |
| **TPOT** | Time Per Output Token — decode speed |
| **Throughput** | Tokens/sec across all requests |
| **Queue depth** | Kitne requests wait kar rahe hain |
| **GPU utilization** | 60-80% healthy; 95%+ = queuing, 30% = overprovisioned |

---

## Khud se pooch 🤔

1. 95% products ko self-hosting kyun nahi chahiye? 5 reasons.
2. Self-hosting ke 6 valid reasons kya hain? Break-even math mein kaunsa cost log bhool jaate hain?
3. PagedAttention kya problem solve karta hai? Ye Day 1 ke KV cache concept se kaise juda hai?
4. Continuous batching kya hai? Traditional batching se better kyun?
5. Quantization ke levels aur unke trade-offs? Test kaise karoge?
6. KV cache model weights se bada kab ho sakta hai?
7. Model routing kya hai? Typical impact? "Cascade" pattern kaise kaam karta hai?
8. Ek healthcare startup patient records pe RAG bana rahi hai. Data India mein rehna chahiye, PHI cloud pe nahi jaa sakta. 5,000 queries/day. Kya architecture?

<details>
<summary>Answer 8</summary>

**Ye self-hosting ka #1 valid reason hai — data privacy hard requirement.** Volume kam hai (5K/day), par volume yahan decision driver nahi hai; compliance hai.

**Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│  ON-PREM / INDIA REGION VPC (koi data bahar nahi)       │
│                                                          │
│  ┌──────────────┐    ┌─────────────┐   ┌─────────────┐ │
│  │ App (FastAPI)│───→│  Postgres   │   │  vLLM       │ │
│  │              │    │  + pgvector │   │  (GPU)      │ │
│  └──────┬───────┘    └─────────────┘   └──────┬──────┘ │
│         │                                      │        │
│         └──────────────────────────────────────┘        │
│                                                          │
│  Embedding model: local (bge-m3 — multilingual)         │
│  Reranker: local (bge-reranker-v2-m3)                    │
└─────────────────────────────────────────────────────────┘
```

**Model choice:**
- Open-weights model self-hosted — Llama/Qwen/Mistral family, medical-domain fine-tune agar available ho
- Size: **8B-30B range** shuru mein. 5K queries/day pe 70B overkill hai aur mehnga.
- Quantization: **FP8 ya AWQ 4-bit** — apne eval set pe test karke decide karo

**GPU sizing (5,000 queries/day = ~4 queries/min peak):**
- Ye **bahut kam load** hai. Ek A100 40GB ya do RTX 4090 kaafi hain.
- Redundancy ke liye 2 nodes (healthcare mein downtime acceptable nahi)
- Continuous batching se ye load trivially handle ho jaayega

**Critical decisions:**

**1. Model quality gap accept karna padega.** Open models frontier API models se peeche hain, especially medical reasoning mein. Mitigation:
- Strong RAG (hybrid + reranking) — model ko kam "sochna" pade, zyada "padhna"
- Narrow scope — general medical advice nahi, sirf **in patient records se retrieval**
- **Human-in-the-loop mandatory** — output clinician review ke liye, direct patient ko nahi

**2. Eval suite din 1 se.** Healthcare mein "vibes se ship" acceptable nahi. Golden set with clinician-validated answers. Groundedness metric especially critical — hallucination yahan patient safety issue hai.

**3. Audit trail.** Har query, retrieved chunks, generated answer, aur user — sab log. Compliance requirement, aur debugging ke liye bhi.

**4. Access control retrieval layer pe** (Day 3 ka lesson) — ek doctor sirf apne patients ke records dekh sake. Prompt mein nahi, `WHERE` clause mein.

**5. PHI redaction** — logs mein PHI na jaaye. Traces mein bhi.

**Team requirement (imaandaar baat):**
Ye setup ek **dedicated infra person** demand karta hai. GPU nodes, model updates, monitoring — ye kaam koi own karega. Agar team ke paas wo bandwidth nahi hai, to:
- **Alternative:** India-region ka managed cloud (jahan provider data residency guarantee kare) — agar compliance team us par sign off kare
- Ya ek healthcare-specific compliant AI vendor

**Jo NA karo:** PHI ko public API pe bhejna, chahe "anonymized" ho. Re-identification risk real hai, aur compliance audit mein ye nahi chalega.
</details>

---

**Next:** [`07-evals-aur-observability.md`](07-evals-aur-observability.md) — sabse skip kiya jaane wala, sabse important layer.
