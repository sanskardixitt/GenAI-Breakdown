# 05 — Inference aur Decoding: jab model jawab deta hai

> **Is file ka goal:** Temperature, top-p, streaming, latency — ye sab parameters kya *actually* karte hain. Ye knowledge tumhe production mein rozana kaam aayegi.

---

## Part 1: Inference ke do phases

Ye distinction bahut log nahi jaante, par ye **poori latency aur cost economics** explain karta hai:

```
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1: PREFILL (prompt processing)                       │
│                                                              │
│  Poora input ek saath process hota hai (PARALLEL!)           │
│  Saare tokens ke K,V compute hote hain                       │
│  → Compute-bound (GPU ke math units busy)                    │
│  → Time ∝ input length                                       │
│  → Ye "Time To First Token" (TTFT) hai                       │
├─────────────────────────────────────────────────────────────┤
│  PHASE 2: DECODE (generation)                                │
│                                                              │
│  Ek-ek token, SEQUENTIALLY (parallel nahi ho sakta!)          │
│  Har step pe KV cache se purana K,V padhna padta hai          │
│  → MEMORY-bandwidth-bound (GPU math units idle rehte hain!)   │
│  → Time ∝ output length                                       │
│  → Ye "tokens per second" hai                                 │
└─────────────────────────────────────────────────────────────┘
```

### 🔑 Ye insight sab explain karta hai

| Observation | Kyun |
|---|---|
| Lamba prompt = pehla token late aata hai | Prefill compute-heavy hai |
| Lamba output = poora response slow | Decode sequential hai, parallelize nahi hota |
| Output tokens input se 4-5x mehnge hain | Decode inefficient hai (memory-bound, GPU idle) |
| Prompt caching se latency 5x kam ho jaati hai | Prefill skip ho gaya! |
| Streaming se app "fast" lagta hai | TTFT dikha diya, total time same |
| Bigger batch = better throughput, same latency | Decode memory-bound hai; batching se compute reuse hota hai |

> **Practical rule:** Output tokens minimize karo. "Concise raho" instruction dena sirf UX nahi — ye **direct latency + cost optimization** hai.

---

## Part 2: Decoding strategies — token kaise chunta hai

Model ka output actually ek **probability distribution** hai poore vocabulary pe:

```
Input: "Mumbai mein aaj bahut"

Model output (raw probabilities):
  "garmi"    : 0.31
  "baarish"  : 0.24
  "traffic"  : 0.15
  "bheed"    : 0.09
  "acha"     : 0.05
  ... (128,000 more tokens with tiny probabilities)
```

Ab isme se **ek chunna** hai. Ye "decoding strategy" hai.

### (a) Greedy — always highest

```python
next_token = argmax(probabilities)   # "garmi"
```

**Deterministic** (almost — hardware/batching variance rehta hai).
**Problem:** Repetitive aur boring. Aur "locally best" choice globally best nahi hoti — model ek boring path mein phas jaata hai.

### (b) Temperature sampling

Temperature probabilities ko "sharpen" ya "flatten" karta hai:

```
probability_new = softmax(logits / temperature)
```

| Temperature | Effect | Distribution |
|---|---|---|
| `0.0` | Greedy (deterministic) | "garmi": 1.00, baaki 0 |
| `0.3` | Focused, thoda variety | garmi: 0.62, baarish: 0.30, traffic: 0.06 |
| `0.7` | Balanced (chat default) | garmi: 0.34, baarish: 0.28, traffic: 0.19 |
| `1.0` | Raw model distribution | garmi: 0.31, baarish: 0.24, traffic: 0.15 |
| `1.5+` | Chaotic, incoherent | almost flat — nonsense aayega |

**Intuition:** Temperature ek "creativity knob" hai. Kam = predictable/safe, zyada = creative/risky.

### (c) Top-p (nucleus sampling) — usually better

Fixed number of candidates lene ki jagah, **cumulative probability** se cut karo:

```
top_p = 0.9  →  "sabse likely tokens lo jab tak unka total 90% ho jaaye"

Case A (model confident hai):
  garmi: 0.85 → cumulative 0.85
  baarish: 0.08 → cumulative 0.93 ✓ ruk gaye
  → sirf 2 candidates

Case B (model unsure hai):
  a: 0.15, b: 0.13, c: 0.12, d: 0.11, e: 0.10, f: 0.09... → 8 candidates
```

**Adaptive hai** — jab model confident ho to kam options, jab confused ho to zyada. Isliye ye top-k se better hai.

### (d) Top-k

Simply top K tokens lo (jaise K=40), baaki zero. Simple par rigid — confident case mein bhi 40 options rakhta hai.

### Practical settings (yaad rakho) 📋

| Task | temperature | top_p | Kyun |
|---|---|---|---|
| **Classification / extraction** | 0.0 | — | Determinism chahiye |
| **Structured output (JSON)** | 0.0 – 0.2 | — | Format break nahi hona chahiye |
| **Code generation** | 0.0 – 0.3 | 0.95 | Correctness > creativity |
| **Factual Q&A / RAG** | 0.0 – 0.3 | — | Hallucination kam ho |
| **Chat / assistant** | 0.7 | 0.9 | Natural lage |
| **Creative writing** | 0.9 – 1.0 | 0.95 | Variety chahiye |
| **Brainstorming (N options)** | 1.0 | 0.98 | Diversity chahiye |

> ⚠️ **Temperature aur top_p dono ek saath tune na karo.** Ek chuno. Dono ek hi cheez control karte hain (randomness) — dono badalne se debug karna impossible ho jaata hai.
>
> ⚠️ **2026 note:** Kai naye reasoning models (jaise Claude Opus 5, Sonnet 5, Opus 4.7+) mein `temperature`/`top_p`/`top_k` **remove ho gaye hain** — bhejne pe 400 error aata hai. Wahan behaviour **prompting** aur **effort parameter** se control hota hai. Provider docs check karo — ye pattern aur models mein bhi phailega.

---

## Part 3: Stop conditions — model kab rukta hai

4 tarike:

| Reason | Kya hua | Tumhara action |
|---|---|---|
| `end_turn` / `stop` | Model ne natural end kar diya | ✅ Normal, aage badho |
| `max_tokens` | Tumhara output limit lag gaya | ⚠️ Output **truncated** hai! Handle karo |
| `stop_sequence` | Tumhara diya string aaya | ✅ Expected |
| `tool_use` | Model tool call karna chahta hai | 🔧 Tool chalao, result wapas do |
| `refusal` | Safety policy triggered | 🚫 Handle karo (fallback / user message) |

### 🚩 `max_tokens` wala bug jo sab karte hain

```python
# GALAT — silently truncated output use kar rahe ho
response = client.messages.create(..., max_tokens=500)
data = json.loads(response.content[0].text)   # 💥 crash, ya worse: partial data
```

```python
# SAHI
response = client.messages.create(..., max_tokens=4096)
if response.stop_reason == "max_tokens":
    # Output adhoora hai — retry with higher limit, ya continue karo
    handle_truncation()
```

> **Rule:** `stop_reason` **hamesha** check karo output use karne se pehle. Ye ek line ka check hai jo production mein ghante bachata hai.

---

## Part 4: Streaming

### Kyun matter karta hai

```
Without streaming:  [.................. 12 sec wait ..................] → poora jawab
With streaming:     [1.2s] → token → token → token → ... (12 sec total)
                       ↑
                  user ko yahin se lagta hai "chal raha hai"
```

**Total time same hai**, par perceived latency 10x better. Ye ek psychology win hai, engineering nahi — par bahut bada win hai.

### Kaise kaam karta hai (SSE)

Server-Sent Events — server ek open connection pe chunks bhejta rehta hai:

```
event: message_start
event: content_block_start
event: content_block_delta   → {"text": "Mumbai"}
event: content_block_delta   → {"text": " mein"}
event: content_block_delta   → {"text": " aaj"}
event: content_block_stop
event: message_delta         → {"stop_reason": "end_turn"}
event: message_stop
```

### Streaming ke real fayde (UX se aage)

1. **Timeout protection.** Bade `max_tokens` ke saath non-streaming request HTTP timeout kha sakti hai. Streaming mein connection alive rehta hai. (Isliye kai SDKs bade `max_tokens` pe non-streaming refuse kar dete hain.)
2. **Early cancellation.** User ko jawab galat lag raha hai? Stream cancel karo, tokens bach gaye.
3. **Progressive rendering.** Markdown/code render karte jao.

### Streaming ki complexity (imaandaari se)

- ❌ Structured output (JSON) stream karna painful hai — partial JSON invalid hota hai
- ❌ Output pe validation/guardrail lagana mushkil (poora dekhe bina judge nahi kar sakte)
- ❌ Error mid-stream aa sakta hai — partial output already user ko dikh gaya
- ❌ Tool calls ke saath logic complex ho jaati hai

**Practical:** Chat UI ke liye stream karo. Backend/pipeline calls mein zaroorat nahi — wahan `.get_final_message()` type helper use karo (timeout protection mil jaayega, complexity nahi).

---

## Part 5: Latency budget — kahan time jaata hai

Ek typical RAG request ka breakdown:

```
Network + auth               :   50 ms
Query embedding              :   80 ms
Vector search                :   30 ms
Reranking (cross-encoder)    :  200 ms   ← often ignore hota hai, bada hai!
─────────────────────────────────────
LLM prefill (6K tokens)      :  600 ms   ← prompt caching se ~100ms ho sakta
LLM decode (400 tokens)      : 4000 ms   ← 🔥 SABSE BADA
─────────────────────────────────────
Total                        : ~5.0 sec
```

### Optimization priority (impact ke hisaab se)

| # | Optimization | Impact | Effort |
|---|---|---|---|
| 1 | **Output length kam karo** | 🔥🔥🔥 Huge | Low (prompt instruction) |
| 2 | **Streaming on** | 🔥🔥🔥 Perceived | Low |
| 3 | **Prompt caching** | 🔥🔥 Prefill 5x | Low (prompt reorder) |
| 4 | **Chhota model jahan chalega** | 🔥🔥 2-3x | Medium (routing logic) |
| 5 | Parallel karo (embed + other work) | 🔥 100-200ms | Medium |
| 6 | Reranking optional banao | 🔥 200ms | Medium |
| 7 | Semantic cache (common queries) | 🔥🔥 Full skip | High |

> **Sabse bada lever: output length.** "Answer in 2-3 sentences" jaisi ek line prompt mein daalne se latency half ho sakti hai. Log iske liye infra optimize karne lag jaate hain jabki fix prompt mein hai.

---

## Part 6: Reliability patterns (production must-haves)

```python
# 1. RETRY with exponential backoff
#    Note: most official SDKs ye by default karte hain (max_retries).
#    Pehle wo check karo, phir custom likho.

# 2. TIMEOUT — hamesha set karo
client = Anthropic(timeout=60.0)   # seconds (TS mein milliseconds!)

# 3. FALLBACK model
try:
    resp = call(primary_model)
except (RateLimitError, OverloadedError):
    resp = call(cheaper_fallback_model)

# 4. STOP REASON check
if resp.stop_reason == "max_tokens":
    handle_truncation()
if resp.stop_reason == "refusal":
    handle_refusal()

# 5. OUTPUT VALIDATION
data = parse_and_validate(resp)      # schema validate karo, blindly trust na karo
```

### Error codes jo tumhe pata hone chahiye

| Code | Meaning | Retry? |
|---|---|---|
| 400 | Invalid request (galat params) | ❌ Nahi — code fix karo |
| 401 | Auth fail | ❌ Nahi — key check karo |
| 404 | Galat model ID | ❌ Nahi — model ID fix karo |
| 413 | Request bahut badi | ❌ Nahi — input chhota karo |
| **429** | Rate limit | ✅ Haan — backoff + `retry-after` header |
| **500** | Server error | ✅ Haan — backoff |
| **529** | Overloaded | ✅ Haan — backoff, ya fallback model |

> **Note:** Typed exception classes use karo (`RateLimitError`, `NotFoundError`), error message pe string-matching **nahi**. Message text badal sakta hai, class nahi.

---

## Part 7: Self-hosting ki jhalak (context ke liye)

Agar tum open-source model khud serve karo, to ye stack dikhta hai:

| Layer | Options | Kya karta hai |
|---|---|---|
| **Serving engine** | **vLLM**, TGI, SGLang, TensorRT-LLM | Batching, KV cache management, scheduling |
| **Quantization** | GGUF, AWQ, GPTQ, FP8 | Model chhota karo (memory bacha) |
| **Local/dev** | Ollama, llama.cpp, LM Studio | Laptop pe chalao |
| **Gateway** | LiteLLM, OpenRouter | Provider-agnostic API |

### vLLM ka killer feature: PagedAttention

KV cache ko OS ki virtual memory ki tarah **pages** mein manage karta hai. Isse memory fragmentation khatam ho jaati hai → **2-4x zyada throughput** same GPU pe.

> Dekha? KV cache ka concept (jo tumne file 02 mein padha) yahan directly production optimization ban gaya. **Foundations useless nahi hote.**

**Kab self-host karo:**
- ✅ Data privacy hard requirement hai (healthcare, defence, finance)
- ✅ Bahut high volume (API bill > GPU cost — usually millions of requests/day pe)
- ✅ Custom fine-tuned model serve karna hai
- ❌ Warna **na karo.** GPU ops ek poora full-time job hai. 95% products ke liye API better hai.

---

## Khud se pooch 🤔

1. Prefill aur decode mein kya farak? Kaunsa compute-bound, kaunsa memory-bound, aur kyun?
2. Output tokens input se mehnge kyun hain? (Architecture se answer do)
3. `temperature=0.2` aur `top_p=0.9` mein kya farak hai? Dono ek saath kyun na set karein?
4. Streaming se total time kam hota hai? Agar nahi, to fayda kya hai? 3 fayde batao.
5. Tumhari RAG app 5 second leti hai. Optimize karne ka **pehla** kadam kya hoga aur kyun?
6. `stop_reason` check na karne se kya bug aa sakta hai? Ek concrete example do.
7. Ek user JSON output maang raha hai aur streaming bhi. Kya problem hai?

<details>
<summary>Answer 5 aur 7</summary>

**5.** Sabse pehle **measure karo** ki time kahan jaa raha hai (instrument karo — Day 6 observability). Par bina measure kiye bhi, statistically sabse bada culprit **decode phase** hai (output length).

Do quick wins, is order mein:
1. **Streaming on karo** — perceived latency turant theek (5s → 1s feel). Effort: 1 line.
2. **Output length cap karo** — prompt mein "2-3 sentences mein jawab do" + `max_tokens` reasonable rakho. Agar 400 tokens se 150 tokens ho gaye, decode time 4s → 1.5s.

Ye dono milkar 5s ko ~1.5s actual aur ~0.5s perceived bana dete hain. **Infra ko chhua bhi nahi.**
Uske baad prompt caching (prefill 600ms → 100ms), phir baaki.

**7.** Conflict hai:
- Streaming mein tum **partial** JSON receive karte ho — `{"name": "Ra` invalid JSON hai. Tum parse nahi kar sakte, validate nahi kar sakte.
- Structured output features (`output_config.format` / strict schema) streaming ke saath work karte hain, par tumhe poora accumulate karke tab parse karna padega — to streaming ka real-time fayda gaya.

**Options:**
1. **Best:** Structured data ke liye streaming chhod do — non-streaming + schema validation. User ko spinner dikha do.
2. Hybrid: ek human-readable text part stream karo (UX ke liye), structured data alag non-streaming call/tool se lo.
3. Agar UI incremental hai (jaise ek list bharti jaa rahi hai), partial-JSON parser use karo — par ye complexity ka pitara hai, sirf tab karo jab UX genuinely demand kare.
</details>

---

## 🎉 Day 1 complete!

Ab tum jaante ho:
- ✅ Transformer/attention **kya problem solve karta hai**
- ✅ Tokens, embeddings, context window ki practical reality
- ✅ Model "assistant" kaise banta hai — aur fine-tuning kab karna hai
- ✅ Inference ke phases, decoding params, aur latency kahan jaati hai

**Ye foundation aage sab kuch aasan bana dega.** Jab Day 5 mein tum frameworks dekhoge, tumhe samajh aayega ki wo *kya* abstract kar rahe hain.

**Next:** [`../01-llm-behaviour/01-prompting-fundamentals.md`](../01-llm-behaviour/01-prompting-fundamentals.md)
