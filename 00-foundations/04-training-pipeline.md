# 04 — Training Pipeline: Raw model se Assistant tak

> **Is file ka goal:** Samajhna ki "GPT" aur "ChatGPT" mein farak kya hai. Aur ye ki tum **kab fine-tune karo aur kab na karo** — ye ek mehnga decision hai.

---

## Bade picture mein 3 stages

```
┌──────────────────────────────────────────────────────────────────┐
│  STAGE 1: PRETRAINING                                            │
│  Poore internet ka text → "agla token predict karo"              │
│  Result: BAHUT knowledge, par useless behaviour                  │
│  Cost: $10M - $500M+, mahine lagta hai, hazaar GPUs              │
├──────────────────────────────────────────────────────────────────┤
│  STAGE 2: SFT (Supervised Fine-Tuning)                           │
│  High-quality (instruction → good answer) pairs                  │
│  Result: Instructions follow karna seekh gaya                     │
│  Cost: $10K - $1M, din-hafte                                      │
├──────────────────────────────────────────────────────────────────┤
│  STAGE 3: PREFERENCE TUNING (RLHF / DPO / RLAIF)                 │
│  Human/AI preferences: "A better hai ya B?"                       │
│  Result: Helpful, honest, harmless — "assistant" ban gaya         │
│  Cost: $100K - $10M, ongoing process                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Stage 1: Pretraining — knowledge kahan se aati hai

### Objective (surprisingly simple)

```
Input:  "Bharat ki rajdhani"
Target: " Delhi"

Loss:   Model ne " Delhi" ko kitni probability di? Kam di to penalty.
```

Bas yahi. Trillions of tokens pe repeat. **Koi labels nahi, koi human annotation nahi** — text khud apna label hai (self-supervised).

### Data (roughly)

| Source | Share | Note |
|---|---|---|
| Web crawl (filtered) | 40-60% | CommonCrawl, heavily filtered |
| Code | 10-20% | GitHub. **Reasoning ke liye disproportionately important** |
| Books | 5-15% | Long-form coherence |
| Academic papers | 5-10% | Technical depth |
| Wikipedia / reference | 3-5% | High-quality facts |
| Curated / synthetic | Growing | 2025-26 mein bada shift |

> **Interesting fact:** Code training se model ka **general reasoning** better hota hai, sirf coding nahi. Theory: code mein strict logical structure hota hai — variables, dependencies, control flow — jo model ko step-by-step thinking sikhata hai.

### Scaling laws

Empirically discover hua ki performance predictably improve hoti hai:
```
Performance ~ f(parameters, data, compute)
```
Aur teeno ko **saath badhana** padta hai. Chinchilla paper (2022) ne dikhaya ki purane models **under-trained** the — parameters zyada the, data kam.

**Aaj ka trend:** Parameters ko wahi rakho, **data aur training time badhao**. Isliye "chhote par bahut zyada trained" models (jaise 8B models jo purane 70B se better hain) exist karte hain.

### Pretrained model actually kaisa lagta hai?

Ye samajhna bahut zaroori hai — raw pretrained model **assistant nahi hai**:

```
You: "Bharat ki rajdhani kya hai?"

Raw pretrained model output:
"Bharat ki rajdhani kya hai? Pakistan ki rajdhani kya hai?
Nepal ki rajdhani kya hai? Ye sab sawaal class 5 ke
geography exam mein aate hain. Neeche answer key hai..."
```

Model ne **jawab nahi diya** — usne "aisa text kaise continue hota hai" predict kiya! Kyunki internet pe aisa sawaal usually ek question-list ka part hota hai.

> 🔑 **Raw model ek "text continuation engine" hai, assistant nahi.** Assistant banane ke liye Stage 2 aur 3 chahiye.

---

## Stage 2: SFT — Instruction following

### Kya hota hai

Human experts (ya ab often, strong models) high-quality examples likhte hain:

```json
{
  "instruction": "Bharat ki rajdhani kya hai?",
  "response": "Bharat ki rajdhani New Delhi hai."
}
{
  "instruction": "Is Python code ka bug batao: for i in range(10): print(i]",
  "response": "Line mein syntax error hai — closing bracket galat hai.
               `print(i]` ki jagah `print(i)` hona chahiye..."
}
```

Same training objective (next token prediction!), par ab data **conversations** hai, random internet nahi.

### Quality > Quantity (bada insight)

LIMA paper ne dikhaya: **1,000 bahut high-quality examples**, 50,000 medium-quality se better result dete hain.

**Kyun?** Knowledge already pretraining se aa gayi hai. SFT model ko naya knowledge nahi sikha raha — bas **"kaise behave karna hai"** sikha raha hai. Ye ek style/format transfer hai, knowledge transfer nahi.

> 🔥 **Ye insight tumhare fine-tuning decisions ko drive karega:** fine-tuning **behaviour** badalne ke liye hai, **knowledge** daalne ke liye nahi. Knowledge ke liye RAG use karo.

### Chat template

SFT mein model ko **special tokens** sikhaye jaate hain jo roles mark karte hain:

```
<|system|>Tum ek helpful assistant ho.<|end|>
<|user|>Bharat ki rajdhani?<|end|>
<|assistant|>New Delhi.<|end|>
```

**Isliye "system prompt" kaam karta hai** — ye ek architectural feature nahi, ek **learned convention** hai. Model ne training mein seekha ki `<|system|>` ke baad ki baat ko zyada authority deni hai.

> Isse ek important security insight nikalta hai: system prompt ki "authority" seekhi gayi hai, hard-enforced nahi. Isliye **prompt injection possible hai** — Day 6 mein detail.

---

## Stage 3: Preference Tuning — "acha" jawab kya hai

SFT ke baad model instructions follow karta hai, par uska jawab "best" nahi hota. Problem: **"acha jawab" likhna mushkil hai, par do jawabon mein se better chunna aasan hai.**

### RLHF (classic 3-step)

```
Step 1: SFT model se ek prompt ke 4 answers generate karo

Step 2: Human labeler rank kare:  B > D > A > C

Step 3: Ek "Reward Model" train karo jo ye ranking predict kare
        (ek separate chhota model jo answer ko score deta hai)

Step 4: RL (PPO) se main model ko update karo taaki
        reward model ka score maximize ho
```

### DPO — simpler alternative (aaj zyada popular)

RLHF complex hai (2 models, unstable RL training). **DPO (Direct Preference Optimization)** reward model ko skip karta hai:

```
Data: (prompt, chosen_answer, rejected_answer)
Loss: Directly optimize karo ki chosen ki probability badhe, rejected ki ghate
```

Ek hi model, standard supervised training, kaafi stable. **Isliye open-source community mein DPO dominant hai.**

### RLAIF / Constitutional AI

Human labeling mehngi aur slow hai. Solution: **AI se hi feedback lo.**

Ek model ko ek "constitution" (principles ka set) diya jaata hai, aur wo khud outputs critique karta hai:

```
Principle: "Jawab helpful ho par harmful instructions na de"

Model A ka answer → Critic model: "ye principle todta hai kyunki..."
                  → Revised answer
                  → (revised, original) preference pair ban gaya
```

Ye scale karta hai aur consistent hota hai. Anthropic ka Constitutional AI isi idea pe hai.

### Alignment tax

Preference tuning se model helpful/safe banta hai, par sometimes **kuch raw capability kho deta hai** — creativity kam, zyada hedging ("As an AI, I cannot..."), zyada verbose.

Ye ek real trade-off hai jo lab-level pe balance kiya jaata hai. Isliye kabhi tumhe lagta hai "purana model isme better tha" — ho sakta hai wo alignment tax ho.

---

## Stage 4 (naya): Reasoning training

2024-25 mein ek naya stage add hua — model ko **lamba sochna** sikhaya jaata hai:

```
RL with verifiable rewards:

Math/code problems jinka answer VERIFY ho sakta hai:
  Model long chain-of-thought generate kare
  → Answer check karo (automatically!)
  → Sahi hai to us reasoning path ko reinforce karo
```

**Key insight:** Yahan human labeling ki zaroorat nahi — **reward objective hai** (code chala, test pass hua ya nahi; math ka answer sahi hai ya nahi). Isliye ye massively scale karta hai.

Result: Model **khud se** seekh jaata hai —
- Apna kaam verify karna
- Galti pakadke backtrack karna
- Problem ko sub-problems mein todna
- Zyada compute lagana jab problem hard ho

Iski detail Day 2 (`01-llm-behaviour/02-reasoning-models.md`) mein hai — ye 2026 ka sabse important shift hai.

---

## Practical: Tumhe fine-tune karna chahiye? 🎯

Ye ek **mehnga decision** hai. Ye decision tree follow karo:

```
Problem: model expected output nahi de raha
   │
   ├─→ Prompt behtar likha? (examples, format, clear instructions)
   │      └─ NAHI → PEHLE YE KARO. 60% problems yahan solve hote hain.
   │
   ├─→ Model ke paas zaroori INFORMATION hai?
   │      └─ NAHI → RAG / tools chahiye. Fine-tuning ye NAHI fix karega.
   │
   ├─→ Kya tum FORMAT/STYLE enforce karna chahte ho?
   │      └─ HAAN → structured outputs try karo. Phir few-shot. Phir fine-tune.
   │
   ├─→ Kya tumhare paas 500+ high-quality examples hain?
   │      └─ NAHI → fine-tuning se kuch nahi hoga. Data pehle banao.
   │
   └─→ Kya tumhare paas EVAL SUITE hai?
          └─ NAHI → 🛑 RUKO. Bina eval, fine-tuning andhere mein teer hai.
```

### Fine-tuning kya fix karta hai vs nahi karta

| ✅ Fine-tuning ACHA hai | ❌ Fine-tuning KHARAB hai |
|---|---|
| Consistent output format enforce karna | Naya factual knowledge daalna |
| Specific tone/voice (brand voice) | Frequently changing information |
| Narrow task pe chhota model bada match karna | "Model zyada smart ho jaaye" |
| Latency/cost kam karna (chhota model use karna) | Ek-off edge cases fix karna |
| Domain jargon/style internalize karna | Reasoning capability badhana |
| Long prompt ko internalize karna (cost saving) | Jab tumhare paas eval nahi hai |

### Cost comparison

| Approach | Setup effort | Per-request cost | Update speed |
|---|---|---|---|
| Prompting | Minutes | Baseline | Instant |
| Few-shot | Hours | Slightly zyada (tokens) | Instant |
| RAG | Days | Medium (retrieval + tokens) | Instant (index update) |
| **Fine-tuning** | **Weeks** | **Kam** (chhota prompt/model) | **Days** (retrain) |

> 🔑 **Practical rule:** Prompting → Few-shot → RAG → Fine-tuning. Is order mein try karo, ulta nahi.
> Fine-tuning **last resort** hai, first instinct nahi. 2026 mein ~5% production use-cases ko genuinely fine-tuning chahiye.

### PEFT / LoRA (agar fine-tune karna hi hai)

Poora model fine-tune karna (saare billions parameters) bahut mehnga hai. **LoRA** ka trick:

```
Original weights FREEZE karo.
Har layer mein 2 chhote "adapter" matrices add karo (rank r = 8, 16, 32).
Sirf adapters train karo — total ~0.1-1% parameters.
```

**Fayde:** 100x sasta, ek base model pe kai LoRA adapters swap kar sakte ho, adapter file chhoti (~10-100MB).
**Yahi 95% practical fine-tuning hai.** "Fine-tuning" bolne wale log usually LoRA hi karte hain.

---

## Ek imaandaar summary table

| Stage | Kya sikhta hai | Data | Kaun karta hai |
|---|---|---|---|
| Pretraining | Language + world knowledge | Trillions of web tokens | Sirf bade labs |
| SFT | Instructions follow karna | 1K-100K curated pairs | Labs + **tum (LoRA)** |
| Preference (RLHF/DPO) | "Acha" jawab dena | Preference pairs | Labs + advanced teams |
| Reasoning RL | Lamba sochna, verify karna | Verifiable problems | Sirf bade labs |

> **Tumhare liye realistic:** Tum SFT (LoRA se) kar sakte ho. Baaki stages tumhare liye "already done" hain — tum unke upar build karte ho. Aur wahi normal hai.

---

## Khud se pooch 🤔

1. Raw pretrained model se sawaal poocho to wo jawab kyun nahi deta? Kya karta hai?
2. LIMA paper ka insight kya tha? Wo tumhare fine-tuning decision ko kaise affect karta hai?
3. RLHF aur DPO mein farak? DPO kyun popular ho gaya?
4. Reasoning training mein "verifiable rewards" kyun game-changer hain?
5. System prompt ki "authority" kahan se aati hai? Ye security ke liye kya imply karta hai?
6. Tumhare boss kehte hain: "hamare 10,000 internal documents pe model fine-tune kar do taaki wo unke baare mein jawab de sake." Tumhara response?

<details>
<summary>Answer 6</summary>

**"Ye galat approach hai, aur mehnga fail hoga."** Reasons:

1. **Fine-tuning behaviour sikhata hai, facts nahi.** Model documents ka *style* pakad lega, par facts reliably retrieve nahi karega. Wo confidently plausible-sounding galat jawab dega — worst possible failure mode.
2. **Documents badalte hain.** Naya document aaya = poora retrain. RAG mein = ek index update (seconds).
3. **Citations impossible.** Fine-tuned model nahi bata sakta ki jawab kahan se aaya. Enterprise mein ye usually deal-breaker hai.
4. **Cost:** Fine-tuning + retraining cycle vs ek vector DB.

**Mera proposal:** RAG banao — chunk + embed + hybrid search + reranking. 1 hafte mein working prototype, citations ke saath, instant updates.
**Fine-tuning tab consider karenge** jab RAG chal raha ho aur specific gap dikhe — jaise "model hamare internal jargon ka format galat likhta hai". *Tab* ek chhota LoRA us specific behaviour ke liye. Aur tab tak humare paas eval suite bhi hoga jisse hum measure kar payenge ki fine-tuning ne actually help kiya ya nahi.
</details>

---

**Next:** [`05-inference-aur-decoding.md`](05-inference-aur-decoding.md) — jab model actually jawab deta hai.
