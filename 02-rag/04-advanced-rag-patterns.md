# 04 — Advanced RAG Patterns

> **Is file ka goal:** Generation stage ko theek karna, aur specific problems ke liye specific patterns. Retrieval theek ho gaya (file 03) — ab baaki 20%.

---

## Part 1: Generation stage — prompt design

Retrieval perfect ho gaya, par generation bigaad sakta hai. Ek battle-tested RAG prompt:

```python
RAG_SYSTEM = """Tum ek document-grounded assistant ho.

RULES:
1. SIRF neeche diye gaye documents se answer do. Apna general knowledge use na karo.
2. Har claim ke saath source cite karo: [Doc 1], [Doc 2]
3. Agar documents mein answer nahi hai, to bolo:
   "Diye gaye documents mein is sawaal ka jawab nahi mila."
   Guess na karo, aur na hi general knowledge se bharo.
4. Agar documents aapas mein CONTRADICT karte hain, dono positions batao
   sources ke saath. Ek chun ke doosra chhupa na do.
5. Agar documents partially answer karte hain, jo hai wo do aur clearly
   batao ki kya missing hai.
6. Concise raho — 2-4 sentences jab tak detail na maangi jaaye.
"""

def build_prompt(query, chunks):
    docs = "\n\n".join(
        f"[Doc {i+1}] (source: {c.metadata['source']}, "
        f"section: {c.metadata.get('section_path', 'N/A')})\n{c.text}"
        for i, c in enumerate(chunks)
    )
    return f"""<documents>
{docs}
</documents>

Sawaal: {query}"""
```

### Har rule kis failure ko fix karta hai

| Rule | Fix karta hai |
|---|---|
| "SIRF documents se" | Model apna training knowledge use kar leta tha |
| Citations | Traceability, aur hallucination reduce hoti hai (accountability effect) |
| "nahi mila" escape hatch | **Sabse important** — iske bina model plausible bakwaas banayega |
| Contradiction handling | Model chup-chaap ek side chun leta tha |
| Partial answer | Model sab-ya-kuch-nahi behave karta tha |
| Concise | Cost + latency |

> 🔑 **Rule 3 (escape hatch) sabse critical hai.** Model ko explicit permission chahiye "nahi pata" bolne ke liye. Bina iske, wo *hamesha* kuch generate karega — kyunki uska training objective "text produce karna" hai, "abstain karna" nahi.

### Citation validation (trust, but verify)

```python
import re

def validate_citations(answer, chunks):
    cited = set(int(m) for m in re.findall(r'\[Doc (\d+)\]', answer))
    valid = set(range(1, len(chunks) + 1))

    if cited - valid:                         # hallucinated citation!
        log.warning(f"Invalid citations: {cited - valid}")
        return False, "Citation error detected"
    if not cited and "nahi mila" not in answer:
        log.warning("Answer diya par koi citation nahi")
        return False, "Uncited answer"
    return True, None
```

**Ye tumhara hallucination canary hai.** Agar model `[Doc 7]` cite kar raha hai jab sirf 5 docs the — wo confabulate kar raha hai, aur tumhe **automatically pata chal jaayega**.

---

## Part 2: Chunk ordering — ek chhota bada trick

Recall karo: "lost in the middle" — beech ka content weakly use hota hai.

```python
def reorder_for_attention(chunks):
    """Best chunks ko edges pe rakho, worst beech mein."""
    # chunks relevance-sorted aate hain: [best, 2nd, 3rd, 4th, worst]
    result = []
    for i, c in enumerate(chunks):
        if i % 2 == 0:
            result.append(c)        # even → front
        else:
            result.insert(0, c)     # odd → back
    return result
    # [best, 2nd, 3rd, 4th, 5th] → [4th, 2nd, best, 3rd, 5th]
    #                                          ↑ hmm
```

**Simpler aur better:**
```python
def reorder(chunks):
    # Best pehla, doosra-best AAKHIRI, baaki beech mein
    if len(chunks) <= 2: return chunks
    return [chunks[0]] + chunks[2:] + [chunks[1]]
```

**Impact:** Chhota (2-5%), par free hai. Kai libraries ise "long context reorder" kehte hain.

---

## Part 3: Self-RAG / Corrective RAG

**Idea:** Retrieval ko grade karo, aur kharab ho to correct karo.

```python
def corrective_rag(query, max_attempts=2):
    for attempt in range(max_attempts):
        chunks = retrieve_and_rerank(query)

        # GRADE: ye chunks kaafi hain?
        grade = llm_grade(query, chunks)
        # → "sufficient" | "insufficient" | "irrelevant"

        if grade == "sufficient":
            return generate(query, chunks)

        if grade == "irrelevant" and attempt == 0:
            query = rewrite_query(query)      # naya angle try karo
            continue

        if grade == "insufficient":
            extra = retrieve(decompose(query))   # aur retrieve karo
            chunks += extra
            return generate(query, chunks)

    return "Sufficient information nahi mila."
```

### Grading prompt

```python
GRADER = """Tum ek retrieval quality grader ho.

Documents dekho aur judge karo ki ye sawaal ka jawab dene ke liye kaafi hain:

- "sufficient"   : jawab in documents mein poora hai
- "insufficient" : partially relevant hai, par kuch missing
- "irrelevant"   : ye documents sawaal se related hi nahi

Sirf ek word return karo."""
```

### Verification loop (hallucination catch karo)

```python
def generate_with_verification(query, chunks):
    answer = generate(query, chunks)

    verdict = llm(f"""Kya ye answer POORI TARAH in documents se supported hai?
Koi bhi claim jo documents mein nahi hai, wo unsupported hai.

Documents: {format(chunks)}
Answer: {answer}

Reply: "supported" ya "unsupported: <kaunsa claim>" """)

    if verdict.startswith("unsupported"):
        answer = generate(query, chunks, extra_instruction=
            "Pichhli koshish mein unsupported claims the. SIRF wo likho jo "
            "documents mein explicitly hai.")
    return answer
```

### Trade-off (imaandaar view)

| | Naive RAG | Corrective RAG |
|---|---|---|
| LLM calls | 1 | 2-4 |
| Latency | 3s | 8-15s |
| Cost | 1x | 3-4x |
| Hallucination rate | Baseline | **60-70% kam** |

> **Kab worth hai:** High-stakes domains (medical, legal, financial) jahan galat answer ka cost bahut zyada hai. Casual chat mein overkill.
>
> **Ek sasta 80% version:** Grading skip karo, sirf **verification** rakho. 2 calls, aur most hallucinations pakad leta hai.

---

## Part 4: Specific problem patterns

### (a) Multi-hop questions

```
"Hamare highest-revenue customer ka contract kab renew hota hai?"
```
Ek search se possible nahi — do dependent facts chahiye.

```python
def multi_hop(query, max_hops=3):
    known_facts, current = [], query
    for hop in range(max_hops):
        chunks = retrieve(current)
        fact = llm(f"Ye extract karo: {current}\nContext: {chunks}")
        known_facts.append(fact)

        next_q = llm(f"""Original: {query}
Ab tak pata: {known_facts}
Agar aur information chahiye to next search query do, warna "DONE".""")
        if next_q.strip() == "DONE": break
        current = next_q

    return llm(f"Original: {query}\nFacts: {known_facts}\nFinal answer:")
```

Ye essentially **agentic RAG** hai (file 05).

### (b) Comparison questions

```
"Product A aur Product B mein kya farak hai?"
```

Ek query se **dono** ke chunks nahi milenge (ek dominate karega).

```python
def comparison_rag(query):
    entities = llm(f"Is comparison query mein kaunse entities compare ho rahe hain? "
                   f"Comma-separated list do.\n{query}").split(",")

    per_entity = {}
    for e in entities:
        aspect = llm(f"Query '{query}' mein '{e}' ka kaunsa aspect poocha hai?")
        per_entity[e.strip()] = retrieve(f"{e.strip()} {aspect}")

    return llm(f"Query: {query}\n" +
               "\n".join(f"## {e}\n{format(c)}" for e, c in per_entity.items()))
```

**Key insight:** Har entity ke liye **alag retrieval**. Ye pattern comparison queries pe dramatically better hai.

### (c) Aggregation questions

```
"Hamare paas kitne active contracts hain?"
```

**RAG ye NAHI kar sakta.** Vector search 5 chunks laayega, model unse count karke **confidently galat** number dega.

**Sahi approach — routing:**
```python
def route(query):
    intent = llm(f"""Query classify karo:
- "lookup"      : ek specific fact/policy dhundhna → RAG
- "aggregation" : count/sum/average/ranking → SQL
- "realtime"    : current/live data → API
- "summary"     : poore document ka summary → full context

Query: {query}
Reply: ek word""")

    return {"lookup": rag_pipeline, "aggregation": sql_pipeline,
            "realtime": api_pipeline, "summary": full_context_pipeline}[intent]
```

> 🚩 **Production RAG systems mein router hona chahiye.** "Sab kuch RAG" ek architecture mistake hai. Ye Day 3 ka sabse important structural lesson hai.

### (d) Tables aur numbers

RAG tables pe kamzor hai. Options:

| Approach | Kab |
|---|---|
| Table ko markdown mein convert, poora chunk banao | Chhote tables |
| Table ko **SQL/dataframe** mein daalo, text-to-SQL karo | **Numeric queries — best** |
| Table + LLM-generated summary dono index karo | Mixed queries |
| Table image ko vision model se padhwao | Complex/scanned layouts |

**Rule:** Agar user table pe **calculate** karna chahta hai → SQL. Agar table se ek value **dhundhna** hai → RAG theek hai.

### (e) "Poore document ka summary do"

RAG **wrong tool** hai — retrieval information kaat deta hai.

```python
def summarize_document(doc_id):
    doc = get_full_document(doc_id)

    if count_tokens(doc) < 150_000:
        return llm(f"Is document ka summary do:\n{doc}")     # poora context

    # Hierarchical (map-reduce)
    section_summaries = [llm(f"Summarize:\n{s}") for s in doc.sections]
    return llm(f"In section summaries se ek unified summary banao:\n"
               f"{section_summaries}")
```

---

## Part 5: Caching layers

```
┌────────────────────────────────────────────────────┐
│ L1: EXACT QUERY CACHE                              │
│     hash(query + filters) → answer                  │
│     Hit rate: 5-15%   |  Saving: 100%               │
├────────────────────────────────────────────────────┤
│ L2: SEMANTIC CACHE                                  │
│     query embedding ~ cached query (sim > 0.95)     │
│     Hit rate: 15-30%  |  Saving: 100%               │
│     ⚠️ Threshold dhyaan se — galat hit = galat answer │
├────────────────────────────────────────────────────┤
│ L3: EMBEDDING CACHE                                 │
│     hash(text) → embedding                          │
│     Hit rate: high (re-indexing pe)  | Saving: embed cost│
├────────────────────────────────────────────────────┤
│ L4: PROMPT CACHE (provider-side)                    │
│     Stable prefix ka KV cache reuse                 │
│     Hit rate: 60-90% | Saving: 90% prefill          │
└────────────────────────────────────────────────────┘
```

### Semantic cache — dhyaan se

```python
def semantic_cache_get(query, threshold=0.97):
    q_emb = embed(query)
    hits = cache_index.search(q_emb, k=1)
    if hits and hits[0].score > threshold:
        return hits[0].cached_answer
    return None
```

⚠️ **Threshold high rakho (0.95-0.98).** Kyun:
```
"Delhi mein baarish hui?"     vs  "Delhi mein baarish nahi hui?"
similarity: 0.95 → cache hit → ULTA answer! 💥
```
Embeddings negation blind hain (yaad hai?) — semantic cache is limitation ko inherit karta hai. Numbers, dates, negation wale queries pe careful raho.

---

## Part 6: RAG evaluation

### Retrieval metrics (file 03 mein cover hue)
Recall@K, MRR, nDCG, Precision@K

### Generation metrics — RAG triad

```
        Query
       /     \
      /       \
  Answer ←── Context

1. CONTEXT RELEVANCE : Retrieved context query se relevant hai?
2. GROUNDEDNESS      : Answer context se supported hai? (faithfulness)
3. ANSWER RELEVANCE  : Answer actually query ka jawab deta hai?
```

Teeno alag failure modes hain:

| Failure | Kya hua |
|---|---|
| Low context relevance | **Retrieval** fail — wrong docs |
| Low groundedness | **Hallucination** — model ne bana diya |
| Low answer relevance | Model ne sawaal ka jawab hi nahi diya (off-topic, ya over-hedged) |

### LLM-as-judge (practical)

```python
def judge_groundedness(answer, chunks):
    return llm(f"""Answer ko atomic claims mein todo. Har claim ke liye check
karo ki wo documents se supported hai.

Documents: {format(chunks)}
Answer: {answer}

JSON return karo:
{{"total_claims": N, "supported": N, "unsupported": ["claim1", ...],
  "groundedness_score": 0.0-1.0}}""")
```

**Tips:**
- Judge ke liye **strong model** use karo (weak judge = noisy metric)
- Judge ko **rubric** do, "acha hai kya?" nahi
- Judge ko human labels ke against **validate** karo (50 cases) — warna tum ek unmeasured metric measure kar rahe ho
- Reference-free metrics (groundedness) reference-based se cheaper hain

### Frameworks

| Tool | Kya deta hai |
|---|---|
| **Ragas** | RAG-specific metrics (faithfulness, relevance, context precision) |
| **TruLens** | RAG triad, tracing ke saath |
| **DeepEval** | Pytest-style LLM tests |
| **Promptfoo** | Config-driven eval, CI friendly |
| **LangSmith / Langfuse / Braintrust** | Tracing + eval + dataset management |

> **Realistic advice:** Pehle **apna** chhota eval script likho (30 lines). Framework tab lo jab tumhe pata ho ki tumhe kya measure karna hai. Framework pehle lena = tumhe uske metrics ke hisaab se sochna padega, apne problem ke hisaab se nahi.

---

## Part 7: RAG anti-patterns 🚩

| Anti-pattern | Kyun bura | Fix |
|---|---|---|
| Sab queries RAG pe bhejna | Aggregation/realtime fail | **Router** |
| `top_k = 20` "safety" ke liye | Noise ↑, accuracy ↓ | `top_k = 5` + reranking |
| Reranking skip karna | Ranking problems | Rerank add karo (biggest win) |
| Metadata na store karna | Filter nahi kar sakte, citations weak | Metadata daalo |
| Access control prompt mein | **Security hole** | Retrieval filter |
| Parsed text kabhi na dekhna | Garbage in, garbage out | Chunks manually padho |
| Eval set na hona | Improvement measure nahi hota | 30-case golden set |
| Chunk size guess karna | Suboptimal | 3 sizes test karo |
| "nahi pata" allow na karna | Hallucination guaranteed | Escape hatch do |
| Naive → agentic jump | Foundations kharab, cost 5x | Pehle hybrid+rerank |
| Sirf accuracy measure karna | Cost/latency blow up | Teeno track karo |

---

## Khud se pooch 🤔

1. RAG prompt mein "nahi mila" escape hatch **sabse critical** kyun hai?
2. Citation validation kaise hallucination canary ban jaata hai?
3. Corrective RAG ka cost/benefit? Kab worth hai, kab nahi? Sasta 80% version kya hai?
4. Comparison query ("A vs B") ke liye normal RAG kyun fail hota hai? Fix?
5. RAG triad ke 3 metrics kya hain? Har ek kaunsa failure catch karta hai?
6. Semantic cache mein threshold high kyun rakhna chahiye? Concrete failure example?
7. Aggregation query RAG se kyun nahi ho sakti? Model kya karega?
8. Tumhare RAG mein: Recall@5 = 0.91, groundedness = 0.94, par users complain karte hain ki answers "useless" hain. Kya problem ho sakti hai?

<details>
<summary>Answer 8 (sabse interesting)</summary>

**Retrieval acha hai (0.91), hallucination nahi hai (0.94) — to problem ANSWER RELEVANCE mein hai.** Ye RAG triad ka teesra leg hai jo tum measure nahi kar rahe ho.

Likely causes:

1. **Over-hedging.** Answer technically grounded hai par actionable nahi:
   > *"Documents suggest that leave policies may vary and it's advisable to consult HR for specifics."*
   Groundedness 1.0 (kuch claim hi nahi kiya!), utility 0. Ye ek classic trap hai — **groundedness aur usefulness ek dusre ke against optimize ho sakte hain.**

2. **Sawaal ka jawab hi nahi.** User ne "kitne din?" poocha, answer policy ke baare mein general information de raha hai. Grounded hai, relevant nahi.

3. **Partial answers.** Answer ek hissa deta hai, doosra chhod deta hai — kyunki wo chunk retrieve nahi hua (Recall@5 = 0.91 ka matlab 9% miss).

4. **Format mismatch.** User ek number chahta hai, model 3 paragraph likh raha hai.

**Fixes:**
- **Answer relevance metric add karo** — "kya ye answer sawaal ka direct jawab deta hai? 1-5"
- Prompt mein: *"Direct answer do jab context clear ho. 'Consult HR' jaisa deflection na do jab documents mein answer hai."*
- Prompt mein: *"Agar sawaal ek specific value maangta hai (number, date, naam), to pehle wo value do, phir context."*
- **Sabse important: real user queries se golden set banao**, synthetic se nahi. Ye gap hi isliye chhupa raha ki tumhara eval set real usage represent nahi kar raha.

**Meta-lesson:** Do metrics green hone ka matlab nahi ki system acha hai. **Metric jo tum measure nahi kar rahe, wahi toota hua hota hai.** Isliye RAG triad ke teeno legs chahiye.
</details>

---

**Next:** [`05-agentic-aur-graph-rag.md`](05-agentic-aur-graph-rag.md)
