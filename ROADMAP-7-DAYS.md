# 🗓️ 7-Day Plan — Modern AI Mastery

> **Realistic expectation set karte hain:** 7 din mein tum "expert" nahi banoge. Lekin 7 din mein tum
> **poora landscape samajh jaoge, sahi vocabulary bol paoge, aur khud se architecture decide kar paoge.**
> Wahi actual "mastery ka entry point" hai. Baaki mastery build karne se aati hai.

**Daily commitment:** 3–5 ghante (2–3 ghante padhna + 1–2 ghante code).
Kam time hai? Har din ko 2 din mein baato → 14-day plan ban jaayega. Order na todo.

---

## Day 0 — Setup (30 min, aaj hi kar lo)

```bash
# Python env
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1

pip install anthropic openai python-dotenv
pip install sentence-transformers chromadb rank-bm25
pip install fastapi uvicorn
```

`.env` file banao (repo root mein, `.gitignore` mein add karo):
```
ANTHROPIC_API_KEY=sk-ant-...
```

> Ek hi provider se shuru karo. Multi-provider abstraction pehle din se add karna = premature complexity.

---

## Day 1 — Foundations: "LLM ke andar kya hai?"

**Goal:** Tum bina Google kiye samjha sako ki ek LLM ek token kaise predict karta hai.

| Time | Kya karna hai |
|---|---|
| 60 min | `00-foundations/01-modern-ai-kya-hai.md` |
| 75 min | `00-foundations/02-transformer-aur-attention.md` ⭐ (sabse important) |
| 45 min | `00-foundations/03-tokens-embeddings-context.md` |
| 45 min | `00-foundations/04-training-pipeline.md` |
| 45 min | `00-foundations/05-inference-aur-decoding.md` |
| 45 min | **Lab 1** — tokenizer + embeddings ke saath khelo |

**Day 1 khatam hone par tum ye bata paoge:**
- Attention "kya problem" solve karta hai (aur kyun RNN fail hua)
- Pretraining vs SFT vs RLHF mein farak
- Temperature badhane par actually kya hota hai
- Context window kyun paisa aur latency dono kharch karta hai

**Red flag:** Agar attention samajh nahi aaya to Day 2 pe mat jao. Ek din extra lelo. Ye foundation hai.

---

## Day 2 — LLM Behaviour: "Model ko control kaise karein?"

**Goal:** Prompting ko "jugaad" se "engineering" mein convert karna.

| Time | Kya karna hai |
|---|---|
| 60 min | `01-llm-behaviour/01-prompting-fundamentals.md` |
| 60 min | `01-llm-behaviour/02-reasoning-models.md` ⭐ (2025-26 ka biggest shift) |
| 75 min | `01-llm-behaviour/03-structured-output-aur-tool-calling.md` ⭐⭐ (agents ki neev) |
| 60 min | `01-llm-behaviour/04-context-engineering.md` |
| 60 min | **Lab 2** — structured output + ek tool call khud likho |

**Day 2 khatam hone par:**
- Tum "reasoning model" aur "normal model" ka farak jaante ho, aur kab kaunsa use karna hai
- Tumhe pata hai ki tool calling ek **JSON schema game** hai, magic nahi
- "Context engineering" > "prompt engineering" kyun hai, ye samajh gaye ho

---

## Day 3 — RAG Deep Dive: "Tumhara existing knowledge upgrade"

**Goal:** Naive RAG se production RAG tak. Tumhe already basics aate hain — yahan hum *kyun toota* pe focus karenge.

| Time | Kya karna hai |
|---|---|
| 45 min | `02-rag/01-rag-refresher-aur-kyun-toota.md` |
| 60 min | `02-rag/02-chunking-aur-indexing.md` |
| 75 min | `02-rag/03-retrieval-hybrid-reranking.md` ⭐ (yahan sabse zyada ROI hai) |
| 60 min | `02-rag/04-advanced-rag-patterns.md` |
| 45 min | `02-rag/05-agentic-aur-graph-rag.md` |
| 60 min | **Lab 3** — hybrid search (BM25 + vector) + rerank |

**Day 3 khatam hone par:**
- Tum diagnose kar sakte ho ki RAG ka failure **retrieval** mein hai ya **generation** mein
- Tum hybrid search + reranking ke bina RAG ship nahi karoge (kabhi)
- Tumhe pata hai ki 80% RAG problems chunking aur query rewriting se fix hoti hain

---

## Day 4 — Agents: "2026 ka main paradigm"

**Goal:** Agent ko demystify karna. Agent = while loop + tools. Bas.

| Time | Kya karna hai |
|---|---|
| 45 min | `03-agents/01-agent-kya-hai.md` |
| 75 min | `03-agents/02-agent-loop-aur-harness.md` ⭐⭐ (poore repo ka core) |
| 60 min | `03-agents/03-memory-systems.md` |
| 60 min | `03-agents/04-multi-agent-patterns.md` |
| 45 min | `03-agents/05-mcp-model-context-protocol.md` |
| 90 min | **Lab 4** — scratch se agent loop likho (framework ke bina!) |

**Day 4 khatam hone par:**
- Tum framework ke bina agent likh sakte ho — isliye framework ka value samajh paoge
- Tumhe pata hai kab agent **nahi** banana chahiye (ye zyada important hai)
- Multi-agent hype vs reality clear hai

---

## Day 5 — Frameworks: "Production mein kya use hota hai?" 🔥

**Goal:** Tumhara original sawaal. Poora landscape map + kab kya choose karna hai.

| Time | Kya karna hai |
|---|---|
| 60 min | `04-frameworks/00-framework-landscape-map.md` ⭐⭐⭐ (bade picture ka file) |
| 45 min | `04-frameworks/01-langchain-aur-langgraph.md` |
| 30 min | `04-frameworks/02-llamaindex.md` |
| 45 min | `04-frameworks/03-agent-sdks.md` |
| 30 min | `04-frameworks/04-crewai-autogen-multiagent.md` |
| 40 min | `04-frameworks/05-vector-databases.md` |
| 40 min | `04-frameworks/06-serving-aur-inference-stack.md` |
| 45 min | `04-frameworks/07-evals-aur-observability.md` ⭐ |
| 30 min | `04-frameworks/08-stack-kaise-choose-karein.md` |

**Day 5 khatam hone par:**
- Tum kisi bhi AI product ka stack diagram bana sakte ho
- Tum "LangChain use karein ya nahi" jaise sawaal ka **engineering answer** de sakte ho
- Tumhe pata hai ki eval + observability layer skip karna = production mein blind udna

---

## Day 6 — Production: "Ship karne ke liye"

**Goal:** Demo se product tak ka gap.

| Time | Kya karna hai |
|---|---|
| 60 min | `05-production/01-architecture-blueprints.md` |
| 50 min | `05-production/02-cost-aur-latency.md` ⭐ |
| 60 min | `05-production/03-eval-driven-development.md` ⭐⭐ |
| 50 min | `05-production/04-security-aur-guardrails.md` ⭐ |
| 45 min | `05-production/05-deployment-aur-scaling.md` |
| 60 min | **Lab 5** — 10-case eval harness banao |

**Day 6 khatam hone par:**
- Tum prompt caching + streaming se cost/latency optimize kar sakte ho
- Prompt injection ko tum "AI ka SQL injection" ki tarah treat karte ho
- Tumhare paas eval suite hai — vibes nahi

---

## Day 7 — Consolidation + Mock Interview

**Goal:** Bikhre hue dots ko jodna.

| Time | Kya karna hai |
|---|---|
| 45 min | `07-revision/quick-revision-sheet.md` (poora padho) |
| 60 min | `07-revision/interview-questions.md` — likhkar answer do, dekh ke nahi |
| 75 min | `07-revision/system-design-cases.md` — 2 case khud design karo |
| 120 min | **Capstone**: Lab 6 — chhota agentic RAG app end-to-end |
| 30 min | Apna gap list banao → next 30 din ka plan |

---

## 7 din ke baad kya? (Real mastery yahan se shuru)

| Week | Focus |
|---|---|
| Week 2 | Ek real project ship karo. Choti scope, poori depth. Evals mandatory. |
| Week 3 | Fine-tuning + embeddings fine-tuning explore karo (jab RAG kaafi na ho) |
| Week 4 | Ek eval-driven refactor karo — apne project ki accuracy measure karke badhao |
| Ongoing | Papers padho: "Attention Is All You Need", "ReAct", "Self-RAG", "Toolformer" |

---

## Study rules (in rules ko na todo)

1. **Ek din mein ek folder.** Aage nahi bhaagna.
2. **Har file ke "Khud se pooch" section** ko skip nahi karna. Wahi actual learning hai.
3. **Lab theory ke turant baad.** 24 ghante ka gap = 50% bhool jaoge.
4. **Notes apne shabdon mein likho.** Copy-paste se kuch retain nahi hota.
5. **Confusion ko note karo, chhupao mat.** Ek `MY-DOUBTS.md` file banao aur likhte jao.

> **Sabse important rule:** Jab tum kisi concept ko kisi aur ko Hinglish mein 2 minute mein samjha sako — tabhi tumhe wo aaya hai. Warna sirf padha hai.
