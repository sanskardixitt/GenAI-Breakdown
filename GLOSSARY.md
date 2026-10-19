# Glossary — Hinglish AI Dictionary 📖

> Alphabetical. Har term ka **ek line ka matlab** + **kyun matter karta hai**.

---

## A

**ACL (Access Control List)**
Kaun kaunsa document dekh sakta hai. **Kyun:** RAG mein ye **retrieval layer** mein hona chahiye, prompt mein nahi — warna prompt injection se bypass ho jaayega. Enterprise ka #1 audit failure.

**Adaptive thinking**
Model khud decide karta hai kitna sochna hai (fixed token budget ke bajaye). **Kyun:** `budget_tokens` ki jagah aa gaya hai; naye models mein `budget_tokens` bheja to 400 error.

**Agent**
LLM + loop + tools, jahan **model** decide karta hai kab rukna hai. **Kyun:** Workflow se farak — wahan *tum* steps decide karte ho. 5% cases ko genuinely agent chahiye.

**Agentic RAG**
Retrieval ko ek **tool** bana dena, taaki model decide kare kab aur kya search karna hai. **Kyun:** Multi-hop queries handle karta hai, par 5x cost.

**ANN (Approximate Nearest Neighbour)**
Vector search ka algorithm jo "approximately" best matches dhundhta hai. **Kyun:** "Approximate" ka matlab hai wo miss kar sakta hai — Recall@10 = 0.95 matlab 5% baar sahi answer nahi mila.

**Attention**
Mechanism jisse har token baaki sab tokens ko dekh sakta hai aur relevance ke hisaab se weight de sakta hai. **Kyun:** Transformer ka core; O(n²) hai isliye long context mehnga hai.

---

## B

**Batch API**
Non-realtime requests ka bulk processing, ~50% sasta. **Kyun:** Classification, extraction, eval runs — jahan turant answer nahi chahiye.

**BM25**
Statistical keyword search (term frequency × inverse document frequency). **Kyun:** Exact IDs, product codes, rare terms pakadta hai jo vector search miss karta hai. **Hybrid search ka doosra half.**

**BPE (Byte Pair Encoding)**
Tokenizer algorithm — frequent character pairs ko merge karke tokens banata hai. **Kyun:** Isliye common words 1 token hain aur Hindi 3-4x mehngi hai.

---

## C

**Causal masking**
Training mein future tokens ko block karna taaki model cheat na kare. **Kyun:** Ek forward pass mein poore sentence ke predictions train ho jaate hain.

**Chunk enrichment**
Chunk ke aage header path / context prepend karna. **Kyun:** Standalone meaningless chunk ("ye 26 hafte ki hoti hai") ko searchable banata hai. Chhota change, bada asar.

**Chunking**
Documents ko chhote pieces mein todna indexing ke liye. **Kyun:** Trade-off — chhote = precise search par context kat jaata hai. Parent-document retrieval isse todta hai.

**Compaction**
Purani conversation ko summarize karke replace karna. **Kyun:** Long agents mein context rot rokta hai. **Failed approaches preserve karna zaroori hai** warna agent same galti dohrayega.

**Context engineering**
"Model ko kaunsi information milegi" ka design. **Kyun:** Prompt engineering ka bada bhai. 90% failures context mein hote hain, prompt ke shabdon mein nahi.

**Context rot**
Long agent runs mein context ka degrade hona (purane tool results, stale exploration). **Kyun:** Agent repeat karne lagta hai, task bhool jaata hai, loop mein phas jaata hai.

**Context window**
Model ek call mein kitne tokens dekh sakta hai (input + output). **Kyun:** Ye ek **budget** hai, storage nahi. Bharne se quality girti hai.

**Cross-encoder**
Query aur document ko **ek saath** process karke relevance score deta hai. **Kyun:** Reranking ka engine. Bi-encoder se accurate, par slow — isliye sirf top-50 pe.

---

## D

**Decode phase**
Token-by-token generation. Memory-bandwidth-bound. **Kyun:** Latency ka sabse bada component. Output length kam karna sabse bada lever hai.

**DPO (Direct Preference Optimization)**
RLHF ka simpler alternative — reward model skip karke directly preferences pe optimize. **Kyun:** Stable, ek model, isliye open-source mein dominant.

---

## E

**Effort parameter**
Model ki thinking depth control (`low`/`medium`/`high`/`xhigh`/`max`). **Kyun:** `budget_tokens` ki jagah. Cost/quality ka main knob.

**Embedding**
Text ka numerical vector representation jisme similar meaning = paas ke vectors. **Kyun:** RAG ka foundation. Par 4 limitations: negation blind, exact IDs miss, fixed budget, domain mismatch.

**Eval (evaluation)**
Statistical quality measurement (`assert metric > threshold`, exact match nahi). **Kyun:** AI mein traditional tests kaam nahi karte. **Bina eval ke tum andhere mein ho.**

---

## F

**Few-shot**
Prompt mein examples dena. **Kyun:** Sabse strong prompting technique — model pattern matching mein exceptional hai. 3 examples > 3 paragraphs instructions.

**FFN (Feed-Forward Network)**
Transformer block ka wo hissa jahan har token akele apni information process karta hai. **Kyun:** Attention = communication, FFN = computation.

**Fine-tuning**
Model ke weights ko apne data pe adjust karna. **Kyun:** **Behaviour** sikhata hai, **knowledge** nahi. Last resort — prompting → few-shot → RAG → fine-tuning.

---

## G

**GraphRAG**
Documents se entities+relationships nikaal ke knowledge graph banana. **Kyun:** Multi-hop relationship queries solve karta hai, par indexing 50-150x mehngi. 90% cases ko nahi chahiye.

**Groundedness**
Answer ke saare claims retrieved context se supported hain ya nahi. **Kyun:** RAG triad ka ek leg. Hallucination ka direct metric.

**Guardrails**
Safety checks — input validation, output filtering, approval gates. **Kyun:** LLM output untrusted hai.

---

## H

**Hallucination**
Model ka confidently galat information generate karna. **Kyun:** Ye **bug nahi hai** — architecture ka natural result. Model probability se token chunta hai; "mujhe nahi pata" ka special signal nahi hai.

**HNSW**
Vector index algorithm (multi-layer graph). **Kyun:** Default choice — fast + accurate, par memory-hungry. `ef_search` recall control karta hai.

**Hybrid search**
Vector + keyword (BM25) search combine karna, usually RRF se. **Kyun:** ⭐ RAG ka highest-ROI upgrade. Do fundamentally alag failure modes cover karta hai.

**HyDE (Hypothetical Document Embeddings)**
Query ki jagah ek fake **answer** generate karke usko embed karna. **Kyun:** Answer-to-answer matching question-to-answer se better hai — query-document asymmetry attack karta hai.

---

## K

**KV cache**
Purane tokens ke Key/Value vectors ko store karke reuse karna generation mein. **Kyun:** Generation nearly linear banata hai. **Prompt caching isi se aata hai** — isliye wo prefix-match pe kaam karta hai.

---

## L

**LoRA (Low-Rank Adaptation)**
Fine-tuning ka efficient tarika — base weights freeze, chhote adapters train. **Kyun:** 100x sasta, adapters swap ho sakte hain. 95% practical fine-tuning yahi hai.

**Lost in the middle**
Model context ke shuru aur ant ko better use karta hai, beech ko weakly. **Kyun:** Important information top ya bottom pe rakho.

---

## M

**MCP (Model Context Protocol)**
AI apps aur tools ke beech standard protocol. **Kyun:** M×N integrations ko M+N banata hai. Par context cost aur security surface add karta hai.

**MoE (Mixture of Experts)**
FFN ko kai "experts" mein todna, per token sirf 2-3 active. **Kyun:** Total parameters zyada (capacity), per-token compute kam. "400B total, 30B active" = MoE.

**MRR (Mean Reciprocal Rank)**
Pehla sahi result kis rank pe aaya (1/rank ka average). **Kyun:** Recall se better signal jab ranking quality measure karni ho.

**Multi-head attention**
Kai parallel attention heads, har ek alag relationship pakadta hai. **Kyun:** Ek sentence mein grammar, meaning, references — sab ek saath.

---

## P

**PagedAttention**
KV cache ko OS virtual memory ki tarah pages mein manage karna (vLLM). **Kyun:** Memory fragmentation khatam → 2-4x throughput same GPU pe.

**Parent-document retrieval**
Chhote chunks se **search**, bade parents se **answer**. **Kyun:** Chunking trade-off ko todta hai — precision aur context dono.

**pgvector**
Postgres extension for vector search. **Kyun:** ⭐ 90% projects ka default hona chahiye. Ek DB (data + vectors + full-text), SQL filtering, ops simple.

**Prefill phase**
Poora input parallel process karna. Compute-bound. **Kyun:** TTFT determine karta hai. Prompt caching isko 5x kam karta hai.

**Prompt caching**
Stable prefix ka KV cache reuse karna requests ke beech. **Kyun:** 50-90% input cost saving. **Prefix-match** hai — ek byte badla shuru mein, poora cache invalid, **bina error ke**.

**Prompt injection**
Malicious instructions data ke through inject karna. **Kyun:** **AI ka SQL injection.** Fix architecture mein hai (privilege separation, least privilege), prompt mein nahi.

---

## Q

**Quantization**
Model weights ki precision kam karna (FP16 → FP8 → 4-bit). **Kyun:** Memory kam, throughput zyada. Quality loss task-dependent — apne eval pe test karo.

**Query rewriting**
User query ko retrieval ke liye reformulate karna. **Kyun:** Multi-turn chat mein **mandatory** — "aur uske liye?" standalone meaningless hai.

---

## R

**RAG (Retrieval Augmented Generation)**
Relevant documents retrieve karke prompt mein daalna. **Kyun:** Knowledge cutoff, private data, citations, updatability — 4 problems solve karta hai.

**RAG triad**
Context relevance + Groundedness + Answer relevance. **Kyun:** Teeno alag failure modes hain. Do green ho sakti hain aur teesri toot sakti hai.

**Recall@K**
Sahi document top-K results mein hai ya nahi. **Kyun:** Retrieval quality ka main metric. Recall@20 vs Recall@5 ka gap = ranking problem.

**Reranking**
Cross-encoder se retrieved candidates ko re-order karna. **Kyun:** ⭐ Single highest-impact RAG technique. "Aboutness" se "answerhood" pe shift.

**RLHF (Reinforcement Learning from Human Feedback)**
Human preferences se model ko align karna. **Kyun:** Isse raw model "assistant" banta hai.

**RoPE (Rotary Position Embedding)**
Position ko vector rotation se encode karna. **Kyun:** Aaj ka standard. Relative positions naturally encode karta hai — **isliye 1M context possible hai**.

**RRF (Reciprocal Rank Fusion)**
Kai ranked lists ko merge karna: `score = Σ 1/(k+rank)`, `k=60`. **Kyun:** Score normalization ki zaroorat nahi, scale-free, almost zero tuning.

---

## S

**Semantic cache**
Query embedding similarity se cached answers reuse karna. **Kyun:** 15-30% requests skip. Par threshold high rakho (0.97) — negation pe galat hit khatarnak hai.

**SFT (Supervised Fine-Tuning)**
Instruction-response pairs pe training. **Kyun:** Raw model ko instructions follow karna sikhata hai. Quality > quantity (LIMA insight).

**Streaming**
Response ko token-by-token bhejena. **Kyun:** Total time same, **perceived latency 10x better**. Plus timeout protection.

**Structured output**
JSON Schema se output format guarantee karna (constrained decoding). **Kyun:** Prompt request nahi, decoding-level **guarantee**. Assistant prefill ka replacement.

**System prompt**
Model ko role/rules dene wala message. **Kyun:** Uski "authority" **seekhi hui** hai (training se), architecturally enforced nahi — isliye prompt injection possible hai.

---

## T

**Temperature**
Sampling randomness control. **Kyun:** 0 = deterministic, 1 = raw distribution. **Naye models mein often removed** (400 error) — behaviour prompting aur effort se control hota hai.

**Test-time compute**
Inference pe zyada compute lagana (reasoning tokens). **Kyun:** Naya scaling axis — per-request tunable, retrain nahi karna padta.

**Token**
Text ka chunk (~4 chars English, 1-2 chars Hindi). **Kyun:** Cost, latency, aur context window sab tokens mein measure hote hain.

**Tool calling**
Model ka structured JSON generate karna jo kehta hai "ye tool ye args ke saath chalao". **Kyun:** **Model tool NAHI chalata** — tum chalate ho. Agents ki neev.

**TTFT (Time To First Token)**
Pehla token aane mein kitna time. **Kyun:** Prefill phase ka metric. Prompt caching isse 5x kam karta hai.

---

## V

**Vector database**
Embeddings store aur ANN search karne wala system. **Kyun:** Ise **index** samjho, source of truth nahi — kabhi bhi rebuild ho sakta hai.

**vLLM**
Popular open-source LLM serving engine. **Kyun:** PagedAttention + continuous batching = 2-4x throughput.

---

## W

**Workflow**
Multi-step pipeline jahan **tum** steps decide karte ho (code mein). **Kyun:** 25% cases yahan fit hote hain. Agent se sasta, fast, deterministic, debuggable. **Log ise agent bana dete hain — sabse common galti.**

---

## Quick disambiguation 🔍

Ye jodiyan log confuse karte hain:

| A | B | Farak |
|---|---|---|
| **Workflow** | **Agent** | Tum decide karte ho vs model decide karta hai |
| **Prompt engineering** | **Context engineering** | Shabd vs information |
| **Fine-tuning** | **RAG** | Behaviour vs knowledge |
| **Bi-encoder** | **Cross-encoder** | Separate encoding vs joint encoding |
| **Long-term memory** | **RAG** | User ke facts vs documents |
| **Tool runner** | **Coding agent SDK** | Loop over *tumhare* tools vs full harness + built-in tools |
| **Harness-only** | **Managed platform** | Tum host karte ho vs provider host karta hai |
| **Offline eval** | **Production monitoring** | Eval set pe vs live traffic pe |
| **Direct injection** | **Indirect injection** | User likhta hai vs data mein chhupa hua |
| **Pre-filter** | **Post-filter** | Search se pehle vs baad mein (post mein top-k khaali ho sakta hai) |

---

**Wapas jao:** [`README.md`](README.md) · [`ROADMAP-7-DAYS.md`](ROADMAP-7-DAYS.md) · [`07-revision/quick-revision-sheet.md`](07-revision/quick-revision-sheet.md)
