# Modern AI — Study Notes Repo 🧠

> **Bhai, welcome!** Ye repo ek self-study course hai — "Modern AI kaise kaam karta hai" ko zero-fluff tarike se samajhne ke liye.
> Sab explanation **Hinglish** mein hai (Hindi + English mix), kyunki technical concepts apni bhasha mein jaldi click karte hain.

---

## Ye repo kiske liye hai?

Tum agar:
- RAG aur AI ke baare mein **moderate** knowledge rakhte ho ✅
- Lekin **"modern AI actually kaise chalta hai"** — transformers, reasoning models, agents, tool calling — wo deeply nahi jaante ❌
- Aur **production mein kaunse frameworks use hote hain** ye confusion hai ❌

...to ye repo exactly tumhare liye likha gaya hai.

---

## Structure — kaise padhna hai

Folders **order mein** padho. Har folder ka apna `flow` hai: pehle theory, phir practice, phir production.

```
knowledge/
│
├── README.md                    ← tum yahan ho
├── ROADMAP-7-DAYS.md            ← 🔥 1 week ka day-by-day plan (START HERE)
├── GLOSSARY.md                  ← Hinglish dictionary of all terms
│
├── 00-foundations/              ← LLM ke andar kya hota hai
│   ├── 01-modern-ai-kya-hai.md
│   ├── 02-transformer-aur-attention.md
│   ├── 03-tokens-embeddings-context.md
│   ├── 04-training-pipeline.md
│   └── 05-inference-aur-decoding.md
│
├── 01-llm-behaviour/            ← LLM ko control kaise karein
│   ├── 01-prompting-fundamentals.md
│   ├── 02-reasoning-models.md
│   ├── 03-structured-output-aur-tool-calling.md
│   └── 04-context-engineering.md
│
├── 02-rag/                      ← RAG deep dive (tumhara existing knowledge upgrade)
│   ├── 01-rag-refresher-aur-kyun-toota.md
│   ├── 02-chunking-aur-indexing.md
│   ├── 03-retrieval-hybrid-reranking.md
│   ├── 04-advanced-rag-patterns.md
│   └── 05-agentic-aur-graph-rag.md
│
├── 03-agents/                   ← 2026 ka main paradigm
│   ├── 01-agent-kya-hai.md
│   ├── 02-agent-loop-aur-harness.md
│   ├── 03-memory-systems.md
│   ├── 04-multi-agent-patterns.md
│   └── 05-mcp-model-context-protocol.md
│
├── 04-frameworks/               ← 🔥 "Production mein kya use hota hai"
│   ├── 00-framework-landscape-map.md
│   ├── 01-langchain-aur-langgraph.md
│   ├── 02-llamaindex.md
│   ├── 03-agent-sdks.md
│   ├── 04-crewai-autogen-multiagent.md
│   ├── 05-vector-databases.md
│   ├── 06-serving-aur-inference-stack.md
│   ├── 07-evals-aur-observability.md
│   └── 08-stack-kaise-choose-karein.md
│
├── 05-production/               ← real product banane ke liye
│   ├── 01-architecture-blueprints.md
│   ├── 02-cost-aur-latency.md
│   ├── 03-eval-driven-development.md
│   ├── 04-security-aur-guardrails.md
│   └── 05-deployment-aur-scaling.md
│
├── 06-hands-on-labs/            ← code likh ke seekho
│   └── README.md  + lab files
│
├── 07-revision/                 ← revise + interview prep
│   ├── quick-revision-sheet.md
│   ├── interview-questions.md
│   └── system-design-cases.md
│
└── 08-diagrams/                 ← 🎨 9 editable .excalidraw visual maps
    └── README.md  + 01…09 diagram files
```

---

## Padhne ka tarika (important!)

### 1. Passive reading se kuch nahi hoga
Har file ke end mein **"Khud se pooch"** section hai. Wahan rukh ke, khud ko answer do. Agar answer nahi aaya → wapas upar padho.

### 2. Notes ke saath code chalao
`06-hands-on-labs/` mein har concept ka chhota lab hai. Theory padhne ke turant baad lab karo — warna bhool jaoge.

### 3. Mental model > memorization
Tumhe ye yaad karne ki zaroorat nahi ki "attention formula kya hai". Tumhe ye samajhna hai ki **"attention kya problem solve karta hai"**. Interview aur real work — dono mein wahi kaam aata hai.

### 4. 7 din ka plan follow karo
`ROADMAP-7-DAYS.md` kholo. Har din ka fixed scope hai. Scope creep se bacho.

### 5. Diagram saath mein khol ke rakho
`08-diagrams/` mein har bade concept ka ek **editable** Excalidraw map hai. File padhte waqt uska diagram doosri window mein khula rakho — text aur picture ek saath jaldi click karte hain.

Revision ke waqt ulta karo: **diagram kholo, text notes hata do, aur khud se har box explain karo.** Jahan atko — wahi tumhara gap hai.

---

## Ye repo kya NAHI hai

| ❌ Ye nahi hai | ✅ Ye hai |
|---|---|
| Math-heavy research paper summary | Practitioner ka mental model |
| Ek framework ka tutorial | Poore landscape ka map + trade-offs |
| "Copy-paste karo aur chalao" | "Samjho ki kyun chal raha hai" |
| ML Engineer banne ka course | **AI Engineer** banne ka course |

> **AI Engineer vs ML Engineer:** ML Engineer model *banata* hai (training, fine-tuning, research). AI Engineer pre-trained models ke *upar product* banata hai (RAG, agents, orchestration, evals, prod). Ye repo AI Engineer track hai — 2026 mein 90% jobs isi mein hain.

---

## Progress Tracker

Jaise-jaise padho, tick karte jao:

- [ ] Day 1 — Foundations (transformers, tokens, training)
- [ ] Day 2 — LLM behaviour (prompting, reasoning, tool calling)
- [ ] Day 3 — RAG deep dive
- [ ] Day 4 — Agents
- [ ] Day 5 — Frameworks landscape
- [ ] Day 6 — Production concerns
- [ ] Day 7 — Labs + revision + mock interview

---

## Ek line mein poora course

> **Modern AI = ek pre-trained transformer (LLM) ko, context aur tools dekar, ek loop mein chalana — aur us loop ko evals se measure karke production-safe banana.**

Baaki sab detail hai. Chalo shuru karein → [`ROADMAP-7-DAYS.md`](ROADMAP-7-DAYS.md)
