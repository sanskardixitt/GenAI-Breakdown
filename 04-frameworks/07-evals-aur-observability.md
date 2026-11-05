# 07 — Evals aur Observability ⭐⭐

> **Ye layer sabse zyada skip ki jaati hai — aur sabse zyada mehngi padti hai.**
>
> Bina evals ke tum AI product nahi bana rahe ho; tum **andhere mein prompt tweak** kar rahe ho aur ummeed kar rahe ho.

---

## Part 1: Problem — AI mein testing kaam kyun nahi karti

```python
# ❌ Traditional test — AI pe kaam nahi karta
def test_summarize():
    assert summarize(article) == "Expected summary text"
    # 💥 Har run pe alag output. Yeh test kabhi pass nahi hoga.
```

**Teen fundamental problems:**

| Problem | Detail |
|---|---|
| **Non-determinism** | Same input, alag output |
| **No single correct answer** | 10 alag summaries sab "sahi" ho sakte hain |
| **Quality is graded, not binary** | "Kitna acha hai" — pass/fail nahi |

### Solution: statistical evaluation

```python
# ✅ Eval — threshold pe, exact match pe nahi
def test_summarize_quality():
    results = [evaluate(summarize(case.input), case) for case in eval_set]
    assert mean(r.score for r in results) > 0.85
    assert min(r.score for r in results) > 0.60      # koi disaster nahi
```

> 🔑 **Mental shift:** `assert output == expected` → `assert quality_metric > threshold`

---

## Part 2: Eval dataset banana (sabse important kadam)

### Kahan se cases laao

| Source | Quality | Note |
|---|---|---|
| **Real user queries** ⭐ | Best | Production logs se — actual distribution |
| **Failure cases** ⭐ | Best | Jo cases fail hue, wo eval mein daalo |
| Domain experts | High | Wo edge cases jaante hain |
| Synthetic (LLM-generated) | Medium | Coverage ke liye, par real distribution nahi |
| Public benchmarks | Low | Tumhare use case se match nahi |

> 🚩 **Sabse common galti:** Synthetic eval set banana aur usse khush ho jaana. Synthetic queries **clean** hote hain; real users typos karte hain, adhoore sawaal poochte hain, Hinglish mein likhte hain. Tumhara eval real distribution reflect kare, warna wo ek jhoothi confidence hai.

### Dataset ka shape

```python
@dataclass
class EvalCase:
    id: str
    input: str
    expected_output: str | None = None      # reference (agar hai)
    expected_chunks: list[str] | None = None # retrieval eval ke liye
    must_contain: list[str] = field(default_factory=list)    # facts jo aane chahiye
    must_not_contain: list[str] = field(default_factory=list) # jo NA aayein
    category: str = "general"                # slice-wise analysis ke liye
    difficulty: str = "medium"
    notes: str = ""
```

**Size guidance:**

| Stage | Cases |
|---|---|
| Shuru mein | **20-30** (aaj hi bana lo) |
| Working product | 100-200 |
| Mature | 500+, categories mein sliced |

> **20 cases se shuru karo. "Perfect eval set" ka intezaar mat karo.** 20 cases ke saath tum already 10x better position mein ho zero cases se.

---

## Part 3: Eval methods

### Method 1: Rule-based (sasta, fast, deterministic) ⭐

```python
def rule_evals(output, case):
    return {
        "contains_facts":  all(f.lower() in output.lower() for f in case.must_contain),
        "avoids_banned":   not any(b.lower() in output.lower() for b in case.must_not_contain),
        "valid_json":      is_valid_json(output) if case.expects_json else True,
        "length_ok":       50 < len(output) < 2000,
        "has_citation":    bool(re.search(r'\[Doc \d+\]', output)),
        "no_hedging":      not any(p in output.lower() for p in
                                   ["as an ai", "i cannot", "consult a professional"]),
    }
```

**Ye pehle likho.** Sasta, fast, deterministic — aur surprisingly kaafi failures pakad leta hai.

### Method 2: LLM-as-judge (nuanced quality)

```python
JUDGE_PROMPT = """Tum ek strict evaluator ho.

Question: {question}
Reference answer: {reference}
Actual answer: {actual}

In dimensions pe 1-5 rate karo:
1. ACCURACY: Facts sahi hain? (5 = sab sahi, 1 = major errors)
2. COMPLETENESS: Sawaal ka poora jawab? (5 = poora, 1 = mostly missing)
3. GROUNDEDNESS: Sab claims reference se supported? (5 = haan, 1 = hallucinated)
4. RELEVANCE: Directly sawaal ka jawab? (5 = direct, 1 = off-topic)

JSON return karo:
{{"accuracy": N, "completeness": N, "groundedness": N, "relevance": N,
  "reasoning": "brief", "critical_issues": ["..."]}}"""
```

### 🚨 LLM-judge ke 5 rules (inhe follow na karo to metric bekaar hai)

1. **Strong model use karo** — weak judge = noisy metric. Judge pe kanjoosi mat karo.
2. **Rubric do, "acha hai kya?" nahi** — specific dimensions, specific scales
3. **Judge ko VALIDATE karo** ⭐ — 50 cases khud grade karo, judge se compare karo. Agreement < 80%? Rubric fix karo.
4. **Position bias se bacho** — A/B comparison mein order randomize karo
5. **`temperature=0`** — judge deterministic hona chahiye

> **Rule 3 sabse important hai aur sabse zyada skip hota hai.** Bina validation ke tum ek **unmeasured metric** measure kar rahe ho. Judge tumse disagree kar sakta hai systematically — aur tum uske hisaab se optimize kar rahe hoge.

### Method 3: Pairwise comparison (A/B)

```python
def compare(question, output_a, output_b):
    # Order randomize karo — position bias real hai
    flip = random.random() < 0.5
    first, second = (output_b, output_a) if flip else (output_a, output_b)

    verdict = llm(f"""Question: {question}
Answer 1: {first}
Answer 2: {second}

Kaunsa better hai? Reply: "1", "2", ya "tie". Phir ek line ka reason.""")

    if flip:
        verdict = {"1": "2", "2": "1", "tie": "tie"}[verdict]
    return verdict
```

**Kab:** Do prompt versions compare karne mein. Absolute scoring se **zyada reliable** — LLMs comparison mein better hain absolute judgement se.

### Method 4: Human eval

Sabse accurate, sabse slow. Use karo:
- Judge validation ke liye (Rule 3)
- High-stakes releases pe
- Jab automated metrics disagree karein

---

## Part 4: RAG-specific evals (Day 3 recap + code)

### Retrieval metrics

```python
def eval_retrieval(cases, retriever, k=5):
    hits, mrr = 0, 0.0
    for c in cases:
        got = [x.id for x in retriever(c.input, k=k)]
        exp = set(c.expected_chunks)
        if exp & set(got): hits += 1
        for i, cid in enumerate(got, 1):
            if cid in exp: mrr += 1/i; break
    return {"recall@k": hits/len(cases), "mrr": mrr/len(cases)}
```

### RAG triad (teeno chahiye)

```python
def eval_rag(case, retrieved, answer):
    return {
        "context_relevance": judge_context(case.input, retrieved),   # retrieval sahi?
        "groundedness":      judge_grounded(answer, retrieved),      # hallucination?
        "answer_relevance":  judge_relevance(case.input, answer),    # sawaal ka jawab?
    }
```

**Yaad rakho (Day 3, file 04):** Do metrics green ho sakti hain aur teesri toot sakti hai. Groundedness 0.94 + Recall 0.91 hone ke baawajood answers useless ho sakte hain agar **answer relevance** kharab ho.

---

## Part 5: Agent evals

Agents alag hain — process bhi matter karta hai, sirf output nahi:

```python
def eval_agent_run(case, run):
    return {
        # Outcome
        "task_completed":   verify_outcome(case, run),        # ⭐ sabse important
        "output_correct":   judge_output(case, run.output),

        # Process
        "steps":            run.steps,
        "used_right_tools": set(case.expected_tools) <= set(run.tools_used),
        "no_forbidden":     not (set(case.forbidden_tools) & set(run.tools_used)),
        "no_loops":         not detect_loop(run.tool_calls),

        # Efficiency
        "cost_usd":         run.cost,
        "duration_s":       run.duration,
    }

def test_agent():
    # Non-determinism ke liye: HAR case N baar chalao
    runs = [run_agent(c) for c in cases for _ in range(3)]
    metrics = [eval_agent_run(c, r) for c, r in zip(cases*3, runs)]

    assert mean(m["task_completed"] for m in metrics) > 0.80
    assert median(m["cost_usd"] for m in metrics) < 0.50      # ⭐ cost bhi assert karo!
    assert percentile([m["steps"] for m in metrics], 95) < 20
```

> 🔑 **Cost aur steps bhi assert karo.** Warna accuracy improve hoti rahegi aur bill silently 10x ho jaayega. Ye bahut common hai.

---

## Part 6: Observability — tracing

### Kya trace karna hai

```python
@trace
def rag_pipeline(query, user):
    with span("query_rewrite") as s:
        q = rewrite(query); s.set("original", query); s.set("rewritten", q)

    with span("retrieve") as s:
        chunks = hybrid_search(q, k=50)
        s.set("candidates", len(chunks)); s.set("top_score", chunks[0].score)

    with span("rerank") as s:
        top = rerank(q, chunks, k=5)
        s.set("final_ids", [c.id for c in top])

    with span("generate") as s:
        r = llm(build_prompt(q, top))
        s.set("input_tokens", r.usage.input_tokens)
        s.set("output_tokens", r.usage.output_tokens)
        s.set("cache_read", r.usage.cache_read_input_tokens)   # ⭐
        s.set("stop_reason", r.stop_reason)
        s.set("model", MODEL); s.set("prompt_version", PROMPT_VERSION)

    return r
```

### Trace mein ye fields **zaroor** hon

| Field | Kyun |
|---|---|
| `trace_id` / `run_id` | Correlation |
| `user_id` (hashed) | Per-user debugging |
| `prompt_version` ⭐ | **Regression debugging** — kaunsi version se kharab hua |
| `model` | Model change ka asar |
| Input/output tokens | Cost |
| **`cache_read_input_tokens`** ⭐ | Cache hit rate — silent invalidator pakadne ke liye |
| `stop_reason` | Truncation/refusal detect |
| Retrieved chunk IDs + scores | RAG debugging |
| Latency per stage | Bottleneck |
| Error/exception | Obviously |

> **`prompt_version` aur `cache_read` ko log karna** — ye do fields tumhe baad mein bahut bachaayengi.

### Tools

| Tool | Note |
|---|---|
| **Langfuse** ⭐ | Open-source, self-hostable, tracing + evals + prompt management |
| **LangSmith** | LangChain ka, par framework-agnostic bhi kaam karta hai |
| **Arize Phoenix** | Open-source, OpenTelemetry-native |
| **Braintrust** | Evals-first, strong CI integration |
| **Helicone** | Proxy-based (ek line change) |
| **Custom + OTel** | Poora control, koi lock-in nahi |

**Recommendation:** **Langfuse ya Phoenix** — open-source, self-hostable, aur OTel-compatible. Vendor lock-in kam.

---

## Part 7: Production monitoring (evals se alag)

Evals **offline** hain (eval set pe). Production monitoring **online** hai:

### Metrics

```python
# Quality proxies (kyunki ground truth nahi hai production mein)
metrics = {
    "no_context_rate":    "kitni baar 'jawab nahi mila' bola",
    "citation_rate":      "kitne answers mein citations the",
    "avg_retrieval_score": "top chunk ka average score",
    "refusal_rate":       "stop_reason == refusal",
    "truncation_rate":    "stop_reason == max_tokens",
    "empty_retrieval":    "0 chunks mile",

    # User signals ⭐
    "thumbs_up_rate":     "explicit feedback",
    "regeneration_rate":  "user ne 'try again' click kiya",
    "copy_rate":          "user ne answer copy kiya (positive signal!)",
    "escalation_rate":    "user human support pe gaya",

    # Ops
    "p50/p95/p99_latency": "",
    "cost_per_request":   "",
    "cache_hit_rate":     "",
    "error_rate":         "",
}
```

### 🚩 Alerts jo actually useful hain

```python
alerts = {
    "no_context_rate > 0.25":      "Retrieval toot raha hai",
    "cache_hit_rate < 0.4":        "Cache invalidator aa gaya — cost spike aayegi",
    "cost_per_request > 2x baseline": "Kuch badla — investigate",
    "p95_latency > 15s":           "UX toot raha hai",
    "truncation_rate > 0.05":      "max_tokens kam hai",
    "regeneration_rate > 0.3":     "Quality gir gayi",
}
```

> **`cache_hit_rate` alert underrated hai.** Ek chhota prompt change cache tod deta hai, koi error nahi aata, aur bill 10x ho jaata hai. Ye alert wo pakad leta hai.

---

## Part 8: CI/CD integration

```yaml
# .github/workflows/eval.yml
name: Eval
on: [pull_request]

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: python -m evals.run --dataset evals/golden.jsonl --output results.json
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      - run: python -m evals.check --results results.json --baseline baseline.json
        # Fail agar koi metric baseline se >5% gira
      - uses: actions/upload-artifact@v4
        with: { name: eval-results, path: results.json }
```

**Rule:** **Prompt change = eval run.** Prompt code hai. Code CI se guzarta hai.

---

## Part 9: Practical getting-started (aaj hi karo)

```
DIN 1 (2 ghante):
  1. Production logs se 20 real queries nikalo
  2. Unke expected answers likho (khud, ya expert se)
  3. Ek 30-line eval script likho
  4. Current system pe chalao → BASELINE mil gaya ✅

DIN 2:
  5. Rule-based checks add karo (must_contain, format, citations)
  6. Ek LLM judge add karo — aur usse VALIDATE karo (10 cases khud grade karo)

HAFTA 1:
  7. Tracing add karo (Langfuse/Phoenix)
  8. CI mein eval daalo
  9. Failure cases eval set mein add karte jao

ONGOING:
  10. Har production failure → naya eval case
  11. Har prompt change → eval run
  12. Weekly: metrics review karo
```

> 🔑 **Sabse important line is poore repo ki:**
>
> **20 cases ka eval set aaj banao. "Perfect eval framework" ke intezaar mein 6 mahine mat gawao.**
>
> 20 cases ke saath tum measure kar sakte ho. Zero cases ke saath tum guess kar rahe ho. Ye farak har baaki cheez se bada hai.

---

## Khud se pooch 🤔

1. Traditional tests AI pe kyun kaam nahi karte? 3 reasons.
2. Eval cases kahan se laane chahiye? Synthetic-only eval set ka khatra kya hai?
3. LLM-judge ke 5 rules kya hain? Rule 3 (validation) sabse important kyun hai?
4. Pairwise comparison absolute scoring se better kyun hai? Position bias kaise handle karo?
5. Agent evals mein process kyun matter karta hai? Cost assert karna kyun zaroori hai?
6. Trace mein `prompt_version` aur `cache_read_input_tokens` kyun log karne chahiye?
7. Offline evals aur production monitoring mein farak? Dono kyun chahiye?
8. Tumhari team 6 mahine se AI product bana rahi hai, koi eval nahi hai. Users complain kar rahe hain "pehle better tha". Kya karoge?

<details>
<summary>Answer 8</summary>

**"Pehle better tha" ek classic silent regression hai — aur bina eval ke tum ye prove ya fix nahi kar sakte.**

**Immediate (aaj, 3 ghante):**

1. **Production logs se 30 real queries nikalo** — mix karo: common queries, complaints wale queries, aur edge cases.
2. **Expected answers likho.** Domain expert se help lo. Perfect na ho — "must_contain facts" bhi kaafi hai shuru mein.
3. **Baseline measure karo.** Current system pe chalao. Ab tumhare paas ek **number** hai.

**Kal (4 ghante):**

4. **Git history se purane versions nikalo.** Agar prompts version-controlled hain (ya logs mein `prompt_version` hai), 2-3 purane versions pe **same eval chalao**.
   → Ab tum **prove** kar sakte ho ki regression hui ya nahi, aur **kahan** hui.
   → Agar prompts hardcoded the aur history nahi hai — ye ek painful lesson hai; aaj se version karo.

5. **Retrieval aur generation alag measure karo** (Day 3 ka table). Regression usually retrieval mein hoti hai — koi ne chunking badli, ya `top_k` badla, ya embedding model update hua.

**Is hafte:**

6. **Tracing add karo** (Langfuse/Phoenix). Bina iske next regression bhi invisible rahegi.
7. **CI mein eval** — ab koi prompt change eval ke bina merge na ho.
8. **Har complaint → naya eval case.** Eval set organically badhega.

**Agar regression confirm ho jaaye:**
- Purane version pe rollback karo (agar possible ho) — turant relief
- Phir bisect karo: kaunsa change ne toda?
- Fix karo, eval se verify karo, ship karo

**Agar eval kehta hai ki regression NAHI hui:**
Ye bhi valuable hai! Matlab:
- Users ka **expectation** badh gaya (product better hua, bar upar chala gaya)
- Ya **query distribution** badal gayi (naye use cases jo tumhara system handle nahi karta)
- Ya problem ek **specific slice** mein hai jo tumhare 30 cases mein nahi hai → category-wise eval add karo

**Team ko kaise batau:**

> "Hum 6 mahine se andhere mein ship kar rahe hain. Aaj se hamare paas ek number hai. Har change se pehle wo number check hoga. Ye humein slow nahi karega — ye humein **confidently fast** banayega, kyunki ab hum jaante honge ki humne kuch toda ya nahi."

**Meta-lesson:** Eval ka sabse bada value "quality measure karna" nahi hai — **confidence se change karna** hai. Bina eval ke har change ek gamble hai, isliye teams change karne se darti hain aur product ruk jaata hai.
</details>

---

**Next:** [`08-stack-kaise-choose-karein.md`](08-stack-kaise-choose-karein.md)
