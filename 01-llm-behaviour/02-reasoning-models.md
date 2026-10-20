# 02 — Reasoning Models ⭐ (2025-26 ka sabse bada shift)

> **Is file ka goal:** Samajhna ki "reasoning model" kya hai, wo kaise kaam karta hai, aur **kab use karna hai kab nahi.** Ye 2024 ke baad AI ka sabse important development hai — aur bahut log ise aaj bhi nahi samajhte.

---

## Part 1: Problem kya thi?

File 02 (transformer) se ek constraint yaad karo:

> **Har token ka compute equal hota hai.** Model "easy" token pe kam aur "hard" token pe zyada compute nahi kar sakta.

Iska matlab ek fundamental limitation:

```
Sawaal: "Bharat ki rajdhani?"           → 1 forward pass mein answer
Sawaal: "Ye 500-line code ka bug dhundo" → BHI 1 forward pass mein answer?!
```

Dono ko **same amount of "thinking"** milta tha. Ye clearly galat hai — hard problems ko zyada compute chahiye.

### Pehla hack: Chain-of-Thought (2022)

Log ne discover kiya ki agar model ko **"step by step socho"** bolo, wo apna reasoning likhta hai — aur accuracy badh jaati hai:

```
❌ Direct:
Q: Ravi ke paas 5 seb the, 2 diye, phir 3x kharide. Kitne hain?
A: 12    ← galat (jaldi mein)

✅ CoT:
Q: ... "Step by step socho"
A: Shuru: 5 seb
   2 diye: 5 - 2 = 3 seb
   "3x kharide" ka matlab current ka 3 guna: 3 × 3 = 9
   Total: 3 + 9 = 12... ruko, "3x kharide" ambiguous hai.
   Agar "3 aur kharide" hai to 3 + 3 = 6.
   Assuming 3x = 3 times current: 3 × 3 = 9, total 9.
A: 9
```

**Kyun kaam karta hai?** Ye ek beautiful insight hai:

> **Generated tokens hi compute hain.** Jab model reasoning likhta hai, wo actually **apne aap ko zyada forward passes de raha hai.** Har intermediate token ek naya forward pass hai jisme wo previous reasoning ko *attend* kar sakta hai.
>
> Reasoning tokens = model ka **scratchpad** = extra compute.

Ye "prompt trick" nahi tha — ye ek **computational** insight tha.

---

## Part 2: Reasoning models — trick ko architecture banana

Agar reasoning tokens compute hain, to **model ko RL se sikhao ki reasoning tokens acche kaise likhe.** Yahi reasoning models hain (o1, o3, Claude extended thinking, Gemini thinking, DeepSeek-R1, etc.).

### Training kaise hoti hai (recall file 04)

```
Verifiable problems lo (math, code — jinka answer AUTO-CHECK ho sake)
    ↓
Model long chain-of-thought generate kare
    ↓
Final answer verify karo (test chalao / math check karo)
    ↓
Sahi hai? → us reasoning path ko REINFORCE karo
Galat hai? → penalize
    ↓
Millions of problems pe repeat
```

**Key:** Human labeling ki zaroorat **nahi** — reward objective hai. Isliye ye massively scale karta hai.

### Model khud se kya seekh jaata hai 🤯

Ye emergent behaviours hain — koi explicitly sikhaya nahi:

| Behaviour | Kaisa dikhta hai |
|---|---|
| **Self-verification** | "Ruko, main check karta hoon... 3 × 3 = 9, haan sahi hai" |
| **Backtracking** | "Ye approach kaam nahi kar rahi. Alag tarike se try karta hoon" |
| **Decomposition** | "Ye 3 sub-problems mein todta hoon: pehle X, phir Y..." |
| **Adaptive effort** | Easy problem = 200 thinking tokens. Hard = 20,000 |
| **Alternative exploration** | "Do interpretations possible hain. Dono check karta hoon" |
| **Doubt expression** | "Mujhe yakeen nahi, par most likely..." |

> Ye behaviours **RL se emerge** hui, hardcode nahi ki gayi. DeepSeek-R1 paper mein ise "aha moment" kaha gaya — model ne training ke beech khud backtracking discover ki.

---

## Part 3: Test-time compute — naya scaling axis

Ye ek paradigm shift hai:

```
PURANA SCALING (2018-2023):
  Better model chahiye? → Bada model banao, zyada data, zyada training compute
  → Mehnga, slow, ek baar ka investment

NAYA SCALING (2024+):
  Better answer chahiye? → Model ko INFERENCE PE zyada sochne do
  → Same model, per-request tunable!
```

**Ye huge hai** kyunki:
1. Tum **per-request** decide kar sakte ho kitna kharch karna hai
2. Ek hi model easy aur hard dono kaam kar sakta hai
3. Improvement ke liye retrain nahi karna padta

### Effort / thinking budget

Modern APIs mein tum thinking depth control karte ho:

```python
# Conceptual — exact syntax provider-specific hai
response = client.messages.create(
    model="claude-opus-5",
    thinking={"type": "adaptive"},           # model khud decide kare kitna sochna hai
    output_config={"effort": "high"},        # low | medium | high | xhigh | max
    messages=[...]
)
```

| Effort | Kab use karo | Trade-off |
|---|---|---|
| `low` | Simple tasks, latency-sensitive, subagents | Fast, sasta, shallow reasoning ka risk |
| `medium` | Routine work | Balanced |
| `high` | Intelligence-sensitive kaam (default) | Acha balance |
| `xhigh` | Hardest coding + agentic tasks | Slow, mehnga, best quality |
| `max` | Correctness > cost, research-grade | Sabse slow; kabhi over-thinking |

### 🚨 Important 2026 API changes

Purane models mein `budget_tokens` se thinking budget set karte the. **Naye models mein:**
- `budget_tokens` **remove ho gaya** — bhejne pe 400 error
- Uski jagah **adaptive thinking + `effort`** parameter hai
- `temperature`/`top_p`/`top_k` bhi kai naye models mein remove hain
- Kuch models mein thinking **by default ON** hai (jaise Claude Opus 5) — omit karne pe bhi chalti hai
- Raw chain-of-thought often **return nahi hota** — sirf summary milta hai (`display: "summarized"`)

> **Lesson:** Reasoning ka API surface fast evolve kar raha hai. Provider docs check karo, training-data memory pe bharosa na karo.

---

## Part 4: Kab reasoning model use karo? (asli decision) 🎯

Ye sabse practical section hai.

### ✅ Reasoning model USE karo

| Task | Kyun |
|---|---|
| Multi-step math / quantitative analysis | Intermediate steps chahiye |
| Complex debugging (root cause dhundhna) | Hypothesis → test → backtrack |
| Multi-file code refactor | Dependencies track karna |
| Architecture design decisions | Trade-offs weigh karna |
| Ambiguity resolution | Multiple interpretations explore karna |
| Long-horizon agentic tasks | Planning + self-correction |
| Constraint satisfaction (scheduling, puzzles) | Search + backtrack |
| Legal/financial document analysis | Careful multi-clause reasoning |

### ❌ Reasoning model USE NA KARO

| Task | Kyun nahi | Better option |
|---|---|---|
| Classification | Ek pattern match hai | Chhota fast model, temp=0 |
| Summarization | Reasoning se better nahi hota | Standard model |
| Simple extraction | Structured output kaafi hai | Chhota model + schema |
| Translation | Pattern task | Standard model |
| Format conversion | Mechanical | Standard model / **code** |
| Simple RAG Q&A | Answer context mein hai | Standard model |
| High-volume, latency-critical | 10x slow, 5x mehnga | Chhota model |

### 🔥 Sabse common galti

> **Reasoning model ko "better model" samajhna.** Ye "better" nahi hai — ye **different tool** hai.
>
> Classification task pe reasoning model use karna = screw lagane ke liye hammer. Slow, mehnga, aur output bhi kabhi *kharab* (over-thinking se model simple task ko complicate kar deta hai).

### Cost/latency reality

Roughly, same task pe:

| | Standard model | Reasoning model (high effort) |
|---|---|---|
| Latency | 2 sec | 15-60+ sec |
| Output tokens | 300 | 3,000 (thinking + answer) |
| Cost | 1x | 5-15x |
| Accuracy (easy task) | 95% | 95% (koi fayda nahi!) |
| Accuracy (hard task) | 45% | 85% (huge fayda) |

**Insight:** Fayda **task difficulty ke saath scale** karta hai. Easy tasks pe reasoning ka ROI **zero ya negative** hai.

---

## Part 5: Reasoning models ke saath prompting — ULTA hai!

Ye counter-intuitive hai par bahut important:

| Standard model | Reasoning model |
|---|---|
| "Step by step socho" ✅ helpful | ❌ **Redundant/harmful** — already sochta hai |
| Detailed step-by-step instructions ✅ | ❌ **Over-constraining** — model ka apna plan better hai |
| Few-shot examples ✅ strong | ⚠️ Kam zaroori; over-constrain kar sakte hain |
| Verbose prompt often better | ✅ **Concise, goal-focused** better |
| "Verify your answer" ✅ | ❌ **Already verify karta hai** — bolne se over-verification |

### Reasoning models ke liye prompting rules

```
✅ KARO:
- GOAL clearly batao, method nahi
  "Is codebase mein memory leak dhundo aur fix karo"
  (na ki: "pehle ye file dekho, phir wo, phir...")

- CONSTRAINTS batao
  "Existing API contract na todo. Tests pass hone chahiye."

- SUCCESS CRITERIA batao
  "Done ka matlab: test suite green aur memory usage 100MB se kam"

- Poora task upfront do
  Ek well-specified pehla turn > 5 incremental turns

❌ NA KARO:
- "Think step by step"           → already karta hai
- Step-by-step method dictate    → uska plan usually better hai
- "Double-check your work"       → already karta hai; over-verification hoti hai
- Verbose scaffolding            → quality GIRTI hai
```

> 🔑 **Golden rule:** Purane model ke liye likhe gaye prompts reasoning model pe **often kharab perform karte hain** — wo bahut prescriptive hote hain. Migration ke waqt scaffolding **hatao** aur measure karo. Ye counter-intuitive hai, isliye log miss karte hain.

---

## Part 6: Reasoning tokens visibility

| Provider approach | Kya milta hai |
|---|---|
| **Hidden** | Reasoning hota hai, tumhe nahi dikhta. Bill hota hai. |
| **Summarized** | Ek readable summary milta hai (raw CoT nahi) |
| **Full** | Poora chain-of-thought (kuch open models) |

**2026 mein dominant pattern:** Summarized. Raw chain-of-thought usually **return nahi hota** (safety + competitive reasons).

### Practical implications

1. **Billing:** Thinking tokens **output tokens** ki tarah bill hote hain, chahe dikhein ya na dikhein. Cost estimate mein include karo!
2. **`max_tokens`:** Ye thinking + answer **dono** cover karta hai. Kam rakha to model soch ke khatam ho jaayega aur answer truncate ho jaayega.
   ```
   max_tokens = 1000, thinking = 900 tokens → answer ke liye 100 bache 💥
   ```
   **Rule:** Reasoning models ke saath generous `max_tokens` do (16K-64K+ high effort pe).
3. **Streaming UX:** Default mein thinking hidden ho to user ko ek **lamba pause** dikhta hai. `display: "summarized"` on karo ya UI mein "Thinking..." indicator dikhao.
4. **Multi-turn:** Same model pe conversation continue karte waqt thinking blocks **unchanged wapas bhejo** (edit na karo — signature validation fail hoti hai).

---

## Part 7: Reasoning ki limitations (imaandaar view)

Reasoning models magic nahi hain:

| Limitation | Detail |
|---|---|
| **Missing knowledge fix nahi hota** | Soch ke wo cheez nahi aa jaayegi jo nahi pata. RAG/tools ab bhi chahiye. |
| **Over-thinking** | Simple problem ko complicate kar dete hain. Kabhi galat answer *zyada* sochne se aata hai. |
| **Latency real hai** | 30-60 sec normal hai. Interactive UX design karna padta hai. |
| **Reasoning ≠ correct** | Beautiful reasoning + galat conclusion possible hai. Verify karna zaroori. |
| **Reasoning trace ≠ actual process** | Jo likha hai wo hamesha exactly wo nahi jo internally hua. Debugging ke liye fully bharosa na karo. |
| **Cost compounds in agents** | Har agent step pe thinking = 20 steps × 3000 thinking tokens = 60K tokens |

### Practical pattern: hybrid routing

```python
def route(task):
    complexity = classify_complexity(task)     # chhota fast model se

    if complexity == "simple":
        return call(fast_model, effort=None)               # classification, extraction
    elif complexity == "medium":
        return call(main_model, effort="medium")
    else:
        return call(reasoning_model, effort="xhigh")       # hard debugging, design
```

Ye **model routing** production mein sabse impactful cost optimization hai — often 60-80% cost reduction, accuracy loss ke bina.

---

## Part 8: Ek quick comparison chart

```
                    Standard Model          Reasoning Model
                    ──────────────          ───────────────
Latency             ⚡ 1-3 sec              🐢 15-60+ sec
Cost                💰 1x                   💰💰💰 5-15x
Easy tasks          ✅ Perfect              ⚠️ Overkill (kabhi worse)
Hard reasoning      ❌ Struggles            ✅ Excellent
Prompting style     Detailed, few-shot      Concise, goal-focused
Streaming UX        Smooth                  Long pause pehle
Best for            Volume, speed           Depth, correctness
```

---

## Khud se pooch 🤔

1. Chain-of-Thought **kyun** kaam karta hai? ("Generated tokens = compute" ko apne shabdon mein explain karo)
2. "Test-time compute" ko naya scaling axis kyun kehte hain? Purane scaling se farak?
3. Reasoning training mein "verifiable rewards" kyun game-changer hain?
4. Reasoning models ke saath prompting **ulta** kyun hai? 3 concrete examples do.
5. `max_tokens` reasoning model ke saath kyun khatarnak hai? Kya bug aayega?
6. Ek high-volume task hai: 1 million customer reviews ko sentiment classify karna. Reasoning model use karoge? Kyun/kyun nahi?
7. Tumhara agent 20 steps leta hai, har step pe reasoning model high effort pe. Kya problem hai aur kya karoge?

<details>
<summary>Answer 6 aur 7</summary>

**6. Bilkul nahi.** Sentiment classification ek pattern-matching task hai — reasoning se accuracy barely badhti hai (95% → 96% maybe).

Math karo:
- Reasoning model: 1M requests × ~2000 tokens (thinking+answer) × high cost
- Fast model: 1M requests × ~20 tokens × low cost
- **Difference: 50-100x cost, 10x latency**, ~1% accuracy gain ke liye

**Sahi approach:** Chhota fast model (Haiku-class), `temperature=0`, structured output (enum: positive/negative/neutral/mixed), aur **Batch API** (~50% extra discount, kyunki real-time zaroorat nahi).
**Bonus:** Ek 200-case eval set banao. Agar chhota model 93%+ karta hai, kaam khatam. Reasoning model us 2% ke liye 100x paisa nahi worth hai.

**7. Ye "reasoning cost compounding" problem hai.**
20 steps × 3000 thinking tokens = 60,000 thinking tokens **ek task** ke liye — plus har step pe poora growing context dobara bhejna.

**Fixes (priority order):**
1. **Effort mix karo.** Sirf **planning** steps pe high effort. Execution steps (file read, tool call, formatting) pe `low` — unme sochne ki zaroorat nahi.
2. **Steps kam karo.** 20 steps often ek harness problem hai, model problem nahi. Better tools (ek tool jo 3 kaam kare), ya kuch steps ko deterministic code se replace karo.
3. **Prompt caching on karo.** Agent loop mein prefix mostly stable hota hai — bada win.
4. **Context prune karo.** Purane tool results hata do, warna har step pe woh dobara bhej rahe ho.
5. **Task budget set karo** (jahan supported ho) taaki model khud pace kare aur gracefully finish kare.
6. Sawaal poocho: **kya isko agent hona chahiye?** 20 predictable steps = ye **workflow** hai. Workflow banao — sasta, fast, debuggable.
</details>

---

**Next:** [`03-structured-output-aur-tool-calling.md`](03-structured-output-aur-tool-calling.md) — agents ki actual neev.
