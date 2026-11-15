# Lab 5 — Eval Harness ⭐⭐

> **Day 6 ke baad karo.** Time: 60 min.
> **Goal:** Ek working eval harness banana jo tum apne asli project mein copy kar sako.

---

## Part A: Dataset format (10 min)

```python
# evals/schema.py
from dataclasses import dataclass, field, asdict
import json

@dataclass
class EvalCase:
    id: str
    input: str
    expected_chunks: list[str] = field(default_factory=list)
    expected_answer: str | None = None
    must_contain: list[str] = field(default_factory=list)
    must_not_contain: list[str] = field(default_factory=list)
    category: str = "general"
    difficulty: str = "medium"
    notes: str = ""

def load_cases(path: str) -> list[EvalCase]:
    with open(path, encoding="utf-8") as f:
        return [EvalCase(**json.loads(line)) for line in f if line.strip()]

def save_cases(cases: list[EvalCase], path: str):
    with open(path, "w", encoding="utf-8") as f:
        for c in cases:
            f.write(json.dumps(asdict(c), ensure_ascii=False) + "\n")
```

```jsonl
// evals/datasets/golden.jsonl
{"id": "hr_001", "input": "parental leave ke liye kitna notice?", "expected_chunks": ["hr_01"], "must_contain": ["60"], "must_not_contain": ["consult HR"], "category": "hr"}
{"id": "eng_001", "input": "error code E4471 ka fix?", "expected_chunks": ["eng_02"], "must_contain": ["connection pool"], "category": "engineering"}
{"id": "fin_001", "input": "Rs 8000 expense kaise approve hoga?", "expected_chunks": ["fin_01"], "must_contain": ["finance"], "category": "finance"}
```

---

## Part B: Metrics (20 min)

```python
# evals/metrics.py
import re, json
from common import client, MODEL, FAST_MODEL, text_of

# ═══════════════════════════════════════════════
# 1. RETRIEVAL METRICS (deterministic, free)
# ═══════════════════════════════════════════════
def retrieval_metrics(retrieved_ids: list[str], case) -> dict:
    if not case.expected_chunks:
        return {}
    expected = set(case.expected_chunks)
    got = list(retrieved_ids)

    hit = bool(expected & set(got))
    rr = 0.0
    for i, cid in enumerate(got, 1):
        if cid in expected:
            rr = 1.0 / i
            break
    precision = len(expected & set(got)) / len(got) if got else 0.0

    return {"recall_hit": hit, "reciprocal_rank": rr, "precision": precision}


# ═══════════════════════════════════════════════
# 2. RULE-BASED (deterministic, free) ⭐ pehle ye likho
# ═══════════════════════════════════════════════
def rule_metrics(answer: str, case) -> dict:
    a = answer.lower()
    return {
        "contains_required": all(m.lower() in a for m in case.must_contain)
                             if case.must_contain else True,
        "avoids_banned":     not any(m.lower() in a for m in case.must_not_contain)
                             if case.must_not_contain else True,
        "has_citation":      bool(re.search(r'\[Doc \d+\]', answer)),
        "length_ok":         20 <= len(answer) <= 3000,
        "no_hedging":        not any(p in a for p in
                                     ["as an ai", "i cannot", "consult a professional",
                                      "please contact hr", "may vary"]),
        "not_empty":         len(answer.strip()) > 0,
    }


# ═══════════════════════════════════════════════
# 3. LLM JUDGE (mehnga, nuanced)
# ═══════════════════════════════════════════════
JUDGE_SCHEMA = {
    "type": "object",
    "properties": {
        "accuracy":     {"type": "integer", "minimum": 1, "maximum": 5},
        "completeness": {"type": "integer", "minimum": 1, "maximum": 5},
        "groundedness": {"type": "integer", "minimum": 1, "maximum": 5},
        "relevance":    {"type": "integer", "minimum": 1, "maximum": 5},
        "reasoning":    {"type": "string"},
    },
    "required": ["accuracy", "completeness", "groundedness", "relevance", "reasoning"],
    "additionalProperties": False,
}

JUDGE_PROMPT = """Tum ek strict evaluator ho. Diye gaye answer ko rate karo.

<question>{question}</question>

<retrieved_context>
{context}
</retrieved_context>

<answer>{answer}</answer>

Rate 1-5 (5 = best):

ACCURACY: Facts sahi hain (context ke against)?
  5 = sab facts sahi | 3 = minor issues | 1 = major errors

COMPLETENESS: Sawaal ka poora jawab diya?
  5 = poora | 3 = partial | 1 = mostly missing

GROUNDEDNESS: Har claim context se supported hai?
  5 = sab supported | 3 = kuch unsupported | 1 = mostly hallucinated

RELEVANCE: Answer directly sawaal ka jawab deta hai (deflect nahi karta)?
  5 = direct answer | 3 = partially useful | 1 = off-topic ya deflection

Reasoning ek line mein."""

def judge_metrics(question, answer, context_text) -> dict:
    r = client.messages.create(
        model=MODEL,                      # ⭐ strong model use karo
        max_tokens=1024,
        output_config={"format": {"type": "json_schema", "schema": JUDGE_SCHEMA}},
        messages=[{"role": "user", "content": JUDGE_PROMPT.format(
            question=question, context=context_text[:6000], answer=answer)}],
    )
    scores = json.loads(text_of(r))
    return {f"judge_{k}": v for k, v in scores.items() if k != "reasoning"} | \
           {"judge_reasoning": scores["reasoning"]}
```

---

## Part C: Runner (15 min)

```python
# evals/run.py
import time, json, statistics
from dataclasses import asdict
from schema import load_cases
from metrics import retrieval_metrics, rule_metrics, judge_metrics

def run_evals(dataset_path, pipeline_fn, output_path, use_judge=True):
    """
    pipeline_fn(query) -> (answer: str, retrieved: list[dict])
    """
    cases = load_cases(dataset_path)
    results = []

    for i, case in enumerate(cases, 1):
        print(f"[{i}/{len(cases)}] {case.id}: {case.input[:50]}...")
        t0 = time.time()

        try:
            answer, retrieved = pipeline_fn(case.input)
            error = None
        except Exception as e:
            answer, retrieved, error = "", [], f"{type(e).__name__}: {e}"

        row = {
            "case_id": case.id,
            "category": case.category,
            "difficulty": case.difficulty,
            "error": error,
            "answer": answer,
            "retrieved_ids": [c.get("id") for c in retrieved],
            "latency_s": round(time.time() - t0, 2),
        }

        if not error:
            row |= retrieval_metrics(row["retrieved_ids"], case)
            row |= rule_metrics(answer, case)
            if use_judge:
                ctx = "\n\n".join(c.get("text", "") for c in retrieved)
                row |= judge_metrics(case.input, answer, ctx)

        results.append(row)

    summary = summarize(results)
    by_cat = summarize_by(results, "category")

    out = {"n": len(results), "summary": summary,
           "by_category": by_cat, "results": results}
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(out, f, indent=2, ensure_ascii=False)

    print_report(summary, by_cat)
    return out


def summarize(results):
    ok = [r for r in results if not r.get("error")]
    if not ok:
        return {"error_rate": 1.0}

    def avg(key):
        vals = [r[key] for r in ok if key in r]
        return round(statistics.mean(vals), 3) if vals else None

    return {
        "recall_hit":        avg("recall_hit"),
        "mrr":               avg("reciprocal_rank"),
        "contains_required": avg("contains_required"),
        "avoids_banned":     avg("avoids_banned"),
        "has_citation":      avg("has_citation"),
        "no_hedging":        avg("no_hedging"),
        "judge_accuracy":    avg("judge_accuracy"),
        "judge_groundedness":avg("judge_groundedness"),
        "judge_relevance":   avg("judge_relevance"),
        "p50_latency_s":     round(statistics.median(r["latency_s"] for r in ok), 2),
        "p95_latency_s":     round(sorted(r["latency_s"] for r in ok)[int(len(ok)*0.95)], 2)
                             if len(ok) > 1 else None,
        "error_rate":        round(1 - len(ok)/len(results), 3),
    }


def summarize_by(results, key):
    groups = {}
    for r in results:
        groups.setdefault(r[key], []).append(r)
    return {k: {"n": len(v), **summarize(v)} for k, v in groups.items()}


def print_report(summary, by_cat):
    print(f"\n{'='*62}\nEVAL SUMMARY\n{'='*62}")
    for k, v in summary.items():
        if v is not None:
            flag = ""
            if isinstance(v, float) and k.startswith(("recall", "contains", "judge")):
                flag = " ✅" if v >= 0.85 else (" ⚠️" if v >= 0.70 else " ❌")
            print(f"  {k:<22} {v}{flag}")

    print(f"\n{'BY CATEGORY':<20} {'n':>4} {'recall':>8} {'ground':>8} {'relev':>8}")
    print("-"*62)
    for cat, m in sorted(by_cat.items()):
        print(f"  {cat:<18} {m['n']:>4} {str(m.get('recall_hit','-')):>8} "
              f"{str(m.get('judge_groundedness','-')):>8} "
              f"{str(m.get('judge_relevance','-')):>8}")
```

---

## Part D: CI gate (10 min)

```python
# evals/check.py
import json, sys

TOLERANCES = {
    "recall_hit":         0.03,
    "contains_required":  0.05,
    "judge_groundedness": 0.20,     # 1-5 scale
    "judge_relevance":    0.20,
}
WORSE_IF_HIGHER = {"p95_latency_s": 0.25, "error_rate": 0.02}

def check(results_path, baseline_path):
    results  = json.load(open(results_path, encoding="utf-8"))
    baseline = json.load(open(baseline_path, encoding="utf-8"))
    failures = []

    for metric, tol in TOLERANCES.items():
        curr, base = results["summary"].get(metric), baseline["summary"].get(metric)
        if curr is None or base is None: continue
        if curr < base - tol:
            failures.append(f"REGRESSION {metric}: {base:.3f} → {curr:.3f}")

    for metric, tol in WORSE_IF_HIGHER.items():
        curr, base = results["summary"].get(metric), baseline["summary"].get(metric)
        if curr is None or base is None: continue
        if curr > base * (1 + tol):
            failures.append(f"WORSE {metric}: {base} → {curr}")

    # Category-wise (aggregate problems chhupa deta hai!)
    for cat, m in results["by_category"].items():
        b = baseline["by_category"].get(cat)
        if not b: continue
        if m.get("recall_hit") and b.get("recall_hit"):
            if m["recall_hit"] < b["recall_hit"] - 0.10:
                failures.append(f"CATEGORY {cat} recall: "
                                f"{b['recall_hit']:.2f} → {m['recall_hit']:.2f}")

    if failures:
        print("❌ EVAL FAILED:")
        for f in failures: print(f"   {f}")
        sys.exit(1)
    print("✅ EVAL PASSED")

if __name__ == "__main__":
    check(sys.argv[1], sys.argv[2])
```

---

## Part E: Use karo (5 min)

```python
# run_my_evals.py
from evals.run import run_evals
from lab3d_rerank import hybrid_rerank_search
from lab3e_rag import rag

def pipeline(query):
    answer, source_ids = rag(query, k=5)
    retrieved = hybrid_rerank_search(query, k=5)
    return answer, [{"id": c["id"], "text": c["text"]} for c, _ in retrieved]

# Baseline banao
run_evals("evals/datasets/golden.jsonl", pipeline, "evals/baseline.json")

# ── Ab kuch badlo (prompt, top_k, model) aur dobara chalao ──
run_evals("evals/datasets/golden.jsonl", pipeline, "results.json")

# Compare
# python evals/check.py results.json evals/baseline.json
```

### 🧪 Experiment matrix

```python
EXPERIMENTS = {
    "baseline":     dict(k=5, use_rerank=True),
    "no_rerank":    dict(k=5, use_rerank=False),
    "topk_10":      dict(k=10, use_rerank=True),
    "topk_3":       dict(k=3, use_rerank=True),
}

for name, cfg in EXPERIMENTS.items():
    out = run_evals("evals/datasets/golden.jsonl",
                    make_pipeline(**cfg), f"results_{name}.json", use_judge=True)
    s = out["summary"]
    print(f"{name:<14} recall={s['recall_hit']:.3f} "
          f"ground={s['judge_groundedness']:.2f} "
          f"relev={s['judge_relevance']:.2f} "
          f"p95={s['p95_latency_s']}s")
```

---

## Part F: Judge validation ⭐ (bahut important)

```python
# evals/validate_judge.py
"""
Judge ko validate karo — warna tum ek UNMEASURED metric measure kar rahe ho.
"""
import json

# 1. 15 cases chuno, unke answers generate karo
# 2. KHUD grade karo (1-5 groundedness), human_grades.json mein save karo
# 3. Judge se grade karao
# 4. Agreement measure karo

def validate(results_path, human_grades_path):
    results = json.load(open(results_path))["results"]
    human = json.load(open(human_grades_path))     # {case_id: score}

    pairs = [(human[r["case_id"]], r["judge_groundedness"])
             for r in results if r["case_id"] in human and "judge_groundedness" in r]

    exact = sum(1 for h, j in pairs if h == j) / len(pairs)
    within1 = sum(1 for h, j in pairs if abs(h - j) <= 1) / len(pairs)
    bias = sum(j - h for h, j in pairs) / len(pairs)

    print(f"Exact agreement:  {exact:.1%}")
    print(f"Within ±1:        {within1:.1%}")
    print(f"Judge bias:       {bias:+.2f}  ({'lenient' if bias > 0 else 'strict'})")

    if within1 < 0.80:
        print("❌ Judge unreliable hai — rubric fix karo!")
    else:
        print("✅ Judge acceptable")

    # Biggest disagreements dekho
    print("\nBiggest disagreements:")
    for r in sorted(results, key=lambda r: -abs(human.get(r["case_id"], 0) -
                                                 r.get("judge_groundedness", 0)))[:3]:
        print(f"  {r['case_id']}: human={human.get(r['case_id'])} "
              f"judge={r.get('judge_groundedness')} — {r.get('judge_reasoning','')[:80]}")
```

> 🔑 **Ye step skip mat karo.** Agar judge tumse 30% disagree karta hai, tumhara poora eval noise hai — aur tum us noise ke hisaab se optimize kar rahe ho.

---

## ✅ Lab 5 checklist

- [ ] Dataset format banaya (JSONL)
- [ ] Rule-based metrics likhe (sasta, deterministic)
- [ ] LLM judge likha rubric ke saath
- [ ] Runner chalaya, baseline banaya
- [ ] Category-wise breakdown dekha
- [ ] CI check likha (tolerances ke saath)
- [ ] Experiment matrix chalaya (no_rerank ka farak dekha)
- [ ] **Judge validate kiya** human grades ke against

---

## 🚀 Aage badho

1. **Apne project ka eval banao** — 20 real queries se shuru karo
2. **`regressions.jsonl`** banao — har fix kiya bug wahan daalo
3. **ACL tests** add karo (agar multi-tenant hai)
4. **Adversarial set** — injection attempts (Day 6, file 04)
5. **CI mein daalo** — GitHub Actions
6. **Cost/latency assertions** add karo
7. **Holdout set** — 20% cases alag rakho, sirf monthly use karo

---

**Next:** [`lab-06-capstone.md`](lab-06-capstone.md)
