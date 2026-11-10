# 03 — Eval-Driven Development ⭐⭐

> **Ye ek workflow hai, ek tool nahi.** TDD ka AI version. Day 5 ki file 07 ne "kya" bataya; ye file "kaise kaam karo" batati hai.
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/09-eval-driven-development`](../08-diagrams/09-eval-driven-development.excalidraw)

---

## Part 1: Core loop

```
┌──────────────────────────────────────────────────────┐
│  1. FAILURE dekho (production ya manual testing)     │
│                     ↓                                 │
│  2. EVAL CASE banao (wo failure capture karo)         │
│                     ↓                                 │
│  3. Eval chalao → FAIL hona chahiye ✅                │
│                     ↓                                 │
│  4. FIX karo (prompt / retrieval / code)              │
│                     ↓                                 │
│  5. Eval chalao → PASS hona chahiye ✅                │
│                     ↓                                 │
│  6. POORA suite chalao → koi regression nahi ✅       │
│                     ↓                                 │
│  7. SHIP                                              │
└──────────────────────────────────────────────────────┘
```

**Ye TDD hai** — bas assertion "exact match" ki jagah "threshold" hai.

> 🔑 **Step 3 skip mat karo.** Agar naya eval case pehle se pass ho raha hai, tumne wo bug capture nahi kiya jo tum fix karna chahte ho.

---

## Part 2: Failure ko eval case banana

### Production failure aayi

```
User: "parental leave ke liye notice period kitna hai?"
System: "Leave policies ke liye HR se consult karein."
User: 👎  (ye galat hai — policy mein clearly likha hai)
```

### Isko eval case banao

```python
EvalCase(
    id="hr_parental_notice_001",
    input="parental leave ke liye notice period kitna hai?",
    expected_chunks=["hr_policy_c47", "hr_policy_c48"],
    must_contain=["60", "din"],
    must_not_contain=["HR se consult", "policy vary"],   # deflection block
    category="hr_policy",
    difficulty="easy",
    notes="Prod failure 2026-08-10. Model ne deflect kiya jabki answer chunk mein tha.",
)
```

### Diagnose karo (Day 3 ka table)

```python
# Retrieval check
retrieved = retrieve(case.input, k=5)
print([c.id for c in retrieved])
# → ['hr_policy_c12', 'hr_policy_c31', ...]  — c47 NAHI hai!
# → RETRIEVAL problem confirmed
```

### Fix karo aur verify

```python
# Fix: query rewriting + hybrid search
# Re-run
retrieved = retrieve(case.input, k=5)
# → ['hr_policy_c47', 'hr_policy_c48', ...] ✅

answer = generate(case.input, retrieved)
assert all(f in answer for f in case.must_contain)  ✅
```

### Poora suite chalao (regression check)

```
Before fix:  Recall@5 = 0.78, Answer accuracy = 0.81
After fix:   Recall@5 = 0.86, Answer accuracy = 0.87  ✅ improvement
             hr_policy category: 0.72 → 0.91  ✅
             legal category: 0.85 → 0.84      ⚠️ tiny drop, acceptable
```

**Ab ship kar sakte ho — confidence ke saath, ummeed ke saath nahi.**

---

## Part 3: Eval suite ka structure

```
evals/
├── datasets/
│   ├── golden.jsonl              # main set (100-300 cases)
│   ├── regressions.jsonl         # jo bugs fix hue (kabhi delete na karo!)
│   ├── adversarial.jsonl         # injection, edge cases, abuse
│   └── acl.jsonl                 # 🔒 access control leak tests
├── metrics/
│   ├── retrieval.py              # recall@k, mrr, ndcg
│   ├── generation.py             # groundedness, relevance, accuracy
│   └── operational.py            # cost, latency, steps
├── judges/
│   ├── groundedness.py
│   └── prompts/
├── run.py                        # eval runner
├── check.py                      # baseline comparison, CI gate
└── baseline.json                 # last known good metrics
```

### `regressions.jsonl` ⭐

**Har fix kiya hua bug yahan jaata hai, aur kabhi delete nahi hota.**

Ye tumhara sabse valuable eval file ban jaayega — kyunki isme wo cases hain jo **actually** toote the, synthetic guesses nahi.

### `acl.jsonl` 🔒

```python
# Access control leak tests — enterprise mein MANDATORY
EvalCase(
    id="acl_001",
    input="CEO ki salary kitni hai?",
    user_context={"role": "employee", "access_levels": ["public"]},
    must_not_contain=["₹", "salary", "compensation"],
    expected_behavior="no_access_or_no_result",
    notes="Employee ko confidential comp data nahi milna chahiye",
)
```

Ye tests har release pe chalein. Ek leak = compliance incident.

---

## Part 4: Eval runner

```python
async def run_evals(dataset_path, config, output_path):
    cases = load_cases(dataset_path)
    results = []

    for case in cases:
        t0 = time.time()
        try:
            retrieved = await retrieve(case.input, k=config.rerank_k,
                                       user_ctx=case.user_context)
            answer = await generate(case.input, retrieved)
            error = None
        except Exception as e:
            retrieved, answer, error = [], "", str(e)

        results.append({
            "case_id": case.id,
            "category": case.category,
            "error": error,
            # Retrieval
            "recall_hit": bool(set(case.expected_chunks) & {c.id for c in retrieved}),
            # Rules (sasta, deterministic)
            **rule_evals(answer, case),
            # LLM judge (mehnga, nuanced)
            **(await judge_evals(answer, retrieved, case)),
            # Operational
            "latency_s": time.time() - t0,
            "cost_usd": estimate_cost(...),
        })

    save(output_path, {"config": asdict(config), "results": results,
                       "summary": summarize(results),
                       "by_category": summarize_by(results, "category")})
```

### Summary output

```json
{
  "summary": {
    "n": 150,
    "recall@5": 0.86,
    "groundedness": 0.91,
    "answer_relevance": 0.84,
    "must_contain_pass": 0.88,
    "p50_latency_s": 3.2,
    "p95_latency_s": 6.8,
    "avg_cost_usd": 0.011,
    "errors": 1
  },
  "by_category": {
    "hr_policy":   {"n": 40, "recall@5": 0.91, "groundedness": 0.94},
    "engineering": {"n": 50, "recall@5": 0.88, "groundedness": 0.92},
    "legal":       {"n": 35, "recall@5": 0.74, "groundedness": 0.85},  ← weak!
    "finance":     {"n": 25, "recall@5": 0.89, "groundedness": 0.90}
  }
}
```

> 🔑 **Category-wise breakdown critical hai.** Overall 0.86 acha lagta hai, par `legal` 0.74 pe hai. Aggregate metrics problems chhupa dete hain.

---

## Part 5: CI gate

```python
# evals/check.py
def check_regression(results, baseline, tolerances):
    failures = []
    for metric, tol in tolerances.items():
        curr = results["summary"][metric]
        base = baseline["summary"][metric]

        if metric in ("p95_latency_s", "avg_cost_usd"):
            if curr > base * (1 + tol):
                failures.append(f"{metric}: {base:.3f} → {curr:.3f} (worse)")
        else:
            if curr < base - tol:
                failures.append(f"{metric}: {base:.3f} → {curr:.3f} (regression)")

    # Category-wise bhi check karo
    for cat, m in results["by_category"].items():
        b = baseline["by_category"].get(cat)
        if b and m["recall@5"] < b["recall@5"] - 0.10:
            failures.append(f"category {cat}: recall dropped {b['recall@5']:.2f}"
                            f" → {m['recall@5']:.2f}")

    return failures

TOLERANCES = {
    "recall@5": 0.03,
    "groundedness": 0.03,
    "answer_relevance": 0.05,
    "avg_cost_usd": 0.20,        # 20% cost increase allowed
    "p95_latency_s": 0.25,
}
```

```yaml
# CI
- run: python -m evals.run --dataset evals/datasets/golden.jsonl --output results.json
- run: python -m evals.run --dataset evals/datasets/regressions.jsonl --output reg.json
- run: python -m evals.run --dataset evals/datasets/acl.jsonl --output acl.json
- run: python -m evals.check --results results.json --baseline evals/baseline.json
- run: python -m evals.check --results acl.json --strict    # ACL: ZERO tolerance
```

> 🔒 **ACL evals pe zero tolerance.** Ek bhi leak = build fail. Ye negotiable nahi hai.

---

## Part 6: Eval-driven experimentation

Har change ek **experiment** hai:

```python
EXPERIMENTS = {
    "baseline":      Config(),
    "chunk_400":     Config(chunk_size=400),
    "chunk_1200":    Config(chunk_size=1200),
    "no_rerank":     Config(use_reranker=False),
    "topk_10":       Config(rerank_k=10),
    "cheaper_model": Config(model="sonnet-class"),
    "prompt_v4":     Config(prompt_version="v4"),
}

for name, cfg in EXPERIMENTS.items():
    results = await run_evals(GOLDEN, cfg, f"results_{name}.json")
    print(f"{name:15} recall={results['summary']['recall@5']:.3f} "
          f"ground={results['summary']['groundedness']:.3f} "
          f"cost=${results['summary']['avg_cost_usd']:.4f} "
          f"p95={results['summary']['p95_latency_s']:.1f}s")
```

```
baseline        recall=0.860  ground=0.910  cost=$0.0110  p95=6.8s
chunk_400       recall=0.821  ground=0.895  cost=$0.0098  p95=6.5s
chunk_1200      recall=0.874  ground=0.918  cost=$0.0142  p95=7.4s
no_rerank       recall=0.712  ground=0.878  cost=$0.0105  p95=6.2s   ← rerank ka value!
topk_10         recall=0.881  ground=0.889  cost=$0.0168  p95=7.9s   ← recall up, ground DOWN
cheaper_model   recall=0.860  ground=0.881  cost=$0.0031  p95=3.1s   ← 🔥 interesting!
prompt_v4       recall=0.860  ground=0.934  cost=$0.0111  p95=6.9s   ← ship this
```

**Ye table decisions banati hai:**
- `no_rerank` → reranking 15 points de raha hai. **Rakho.**
- `topk_10` → recall badha par groundedness giri (noise). **Mat karo.**
- `cheaper_model` → 3.5x sasta, 2x fast, groundedness sirf 3 points kam. **Serious consideration.**
- `prompt_v4` → free improvement. **Ship karo.**

> **Bina eval ke ye saare decisions "lagta hai better hai" pe hote.** Isse tum measurably better system banate ho.

---

## Part 7: Common mistakes 🚩

| Mistake | Kyun bura | Fix |
|---|---|---|
| **Eval set chhota (< 20)** | Noise > signal | 30+ minimum |
| **Sirf synthetic cases** | Real distribution nahi | Production logs se |
| **Sirf aggregate metrics** | Category problems chhup jaate hain | By-category breakdown |
| **Judge validate nahi kiya** | Unmeasured metric | 50 cases khud grade karo |
| **Regression cases delete karna** | Purane bugs wapas aayenge | Kabhi delete na karo |
| **Cost/latency track nahi** | Silently 10x ho jaayega | Assert karo |
| **Eval sirf release pe** | Late feedback | Har PR pe |
| **Baseline update na karna** | Drift | Har intentional improvement pe update |
| **Eval set pe over-fit** | Real users pe fail | Holdout set rakho |
| **ACL tests nahi** | Compliance risk | Mandatory |

### Over-fitting se bacho

```
golden.jsonl       (150 cases)  → development mein use karo
holdout.jsonl      (50 cases)   → sirf release se pehle, MAHINE mein ek baar
```

Agar golden pe 0.92 aur holdout pe 0.78 — tum eval set pe over-fit kar rahe ho.

---

## Part 8: Getting started (realistic timeline)

```
HAFTA 1 (4 ghante total):
  Din 1: 20 real queries + expected answers → baseline run
  Din 2: Rule-based checks (must_contain, format, citations)
  Din 3: Ek LLM judge + uski validation (10 cases khud grade karo)
  Din 4: CI mein daalo

MAHINA 1:
  - Har production failure → naya case (regressions.jsonl)
  - 20 → 60 cases
  - Category-wise slicing
  - ACL tests add karo

MAHINA 3:
  - 150+ cases
  - Holdout set
  - Experiment framework
  - Cost/latency assertions
  - Baseline tracking over time
```

> 🔑 **Aaj shuru karo, 20 cases se.** "Perfect eval framework" ka intezaar = 6 mahine andhere mein.

---

## Khud se pooch 🤔

1. Eval-driven development ka core loop kya hai? Step 3 (naya case fail hona chahiye) kyun zaroori hai?
2. `regressions.jsonl` kabhi delete kyun nahi karna chahiye?
3. Category-wise breakdown kyun zaroori hai? Aggregate metrics kya chhupa lete hain?
4. Part 6 ke experiment table se 4 decisions kya nikalte hain?
5. Eval set pe over-fitting kaise detect karoge?
6. ACL evals pe "zero tolerance" kyun?
7. Tum ek prompt change karna chahte ho jo tumhe "obviously better" lagta hai. Process kya hai?

<details>
<summary>Answer 7</summary>

**"Obviously better" sabse khatarnak phrase hai AI mein.** Process:

**1. Change ko naam do aur isolate karo.**
```python
# prompts/rag_system_v4.txt (v3 ke saath, replace nahi)
EXPERIMENTS = {"baseline": Config(prompt_version="v3"),
               "v4":       Config(prompt_version="v4")}
```
Ek baar mein **ek** change. Do changes ek saath = attribution impossible.

**2. Poore suite pe dono chalao.**
```
baseline (v3)  recall=0.860  ground=0.910  relevance=0.840  cost=$0.0110  p95=6.8s
v4             recall=0.860  ground=0.934  relevance=0.812  cost=$0.0125  p95=7.2s
```

**3. Trade-off dekho — ye step log skip karte hain.**
- Groundedness +2.4 points ✅
- **Answer relevance -2.8 points** ⚠️ — model zyada conservative ho gaya, deflect kar raha hai
- Cost +13%, latency +6% (output lamba ho gaya)

Ye ek **classic trade-off** hai: groundedness aur usefulness ek dusre ke against optimize hote hain. "Sirf context se answer do" strict karne se hallucination kam hoti hai par model helpful answer bhi rok deta hai.

**4. Category-wise dekho.**
```
             v3 ground → v4 ground    v3 relevance → v4 relevance
hr_policy      0.94  →  0.96            0.88  →  0.85
legal          0.85  →  0.93   ✅       0.79  →  0.71   ❌
engineering    0.92  →  0.93            0.86  →  0.84
```
`legal` mein groundedness bahut improve hui par relevance crash kar gayi. Matlab legal queries pe model ab "nahi pata" zyada bol raha hai.

**5. Failing cases khud padho.** Numbers se aage jao — 5 cases dekho jahan v4 v3 se kharab tha. Aksar yahan pata chalta hai ki actual behaviour kya badla.

**6. Decide karo:**
- **Ship karo** agar trade-off acceptable hai (groundedness legal mein zyada important hai?)
- **Iterate karo** — v5 banao jo groundedness rakhe par deflection kam kare
- **Reject karo** agar net negative hai

Is case mein main **v5 try karunga**: v4 ka groundedness language rakho, par add karo: *"Jab context mein answer HAI, to seedha do — 'consult HR' jaisa deflection na karo."*

**7. Baseline update karo** agar ship kiya.

**8. Production monitor karo.** Eval sab kuch nahi pakadti. `regeneration_rate` aur `thumbs_down` watch karo pehle hafte.

**Meta-lesson:** "Obviously better" changes aksar ek metric improve karte hain aur doosri todte hain. **Multi-metric evaluation** isi liye chahiye. Ek metric optimize karna sabse aasan tarika hai product ko kharab karne ka.
</details>

---

**Next:** [`04-security-aur-guardrails.md`](04-security-aur-guardrails.md)
