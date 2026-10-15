# 01 — Modern AI kya hai? (Bade picture se shuru)

> **Is file ka goal:** Tumhe "modern AI" ka ek clean mental model dena, taaki baaki sab files us map par fit ho jaayein.

---

## 1. Pehle ek imaandaar definition

Modern AI (2026 wala) ka core ek hi cheez hai:

> **Ek bahut bada neural network jo "agla token kya hoga" predict karna seekh gaya hai — aur wo prediction itni acchi ho gayi hai ki usse reasoning, coding, planning jaise kaam nikal aate hain.**

Bas. Isse zyada mystical kuch nahi hai. Lekin is ek line mein 3 surprising cheezein chhupi hain:

1. **Objective simple hai, capability emergent hai.** Model ko kisi ne "logic" nahi sikhaya. Usne itna text dekha ki logic ka pattern seekh gaya.
2. **Scale ne quality badli.** Same architecture, 100x data + compute → qualitatively naya behaviour. Ye "phase transition" jaisa hai — paani 99°C pe CDLUCLAUD
> Pehle humne **language** solve ki (transformer) → phir **instructions** (RLHF) → phir **knowledge** (RAG/tools) → phir **thinking** (reasoning) → ab **action** (agents).

Ye progression yaad rakho. Har naya development inhi 5 buckets mein se kisi ek mein aata hai.

---

## 3. Aaj ka AI stack — 6 layers

Jab bhi koi AI product dekho, use in 6 layers mein todo. Ye framework tumhe poore repo mein kaam aayega:

```
┌─────────────────────────────────────────────────┐
│  6. PRODUCT       UI, streaming, feedback loop  │
├─────────────────────────────────────────────────┤
│  5. EVAL + OBS    accuracy measure, tracing     │  ← 90% log ye skip karte hain
├─────────────────────────────────────────────────┤
│  4. ORCHESTRATION agent loop, routing, retries  │  ← LangGraph, Agent SDKs
├─────────────────────────────────────────────────┤
│  3. CONTEXT       RAG, memory, tools, MCP       │  ← yahan actual product value hai
├─────────────────────────────────────────────────┤
│  2. MODEL API     Claude / GPT / Gemini / OSS   │  ← commodity ban raha hai
├─────────────────────────────────────────────────┤
│  1. INFRA         GPUs, vLLM, serving           │  ← mostly rented
└─────────────────────────────────────────────────┘
```

**Sabse important insight:** Layer 2 (model) commodity ban raha hai — sab providers ek jaise ho rahe hain. Tumhara **competitive advantage layer 3 aur 5 mein hai** — kitna acha context de paate ho, aur kitna acha measure kar paate ho.

> Naye AI engineers galti ye karte hain ki wo layer 2 pe obsess karte hain ("kaunsa model best hai?") jabki paisa layer 3 aur 5 mein banta hai.

---

## 4. Teen paradigms — inko confuse na karo

Ye teen alag cheezein hain, aur log inhe mix kar dete hain:

### (a) Single call / "Completion"
```
User input → LLM → output
```
Ek request, ek jawab. Classification, summarization, extraction, rewriting.
**Kab use karo:** Jab task ek step ka ho. **90% production use-cases yahi hain** — aur log inhe unnecessarily agent bana dete hain.

### (b) Workflow (tum control karte ho)
```
Input → LLM step 1 → code → LLM step 2 → validate → output
```
Multi-step, lekin **steps tum decide karte ho** — code mein hardcoded hai. LLM sirf specific jagah pe "sochta" hai.
**Kab use karo:** Jab tumhe pata hai ki steps kya hain. Predictable, debuggable, sasta.

### (c) Agent (model control karta hai)
```
Input → LLM → tool → LLM → tool → LLM → ... → output
         ↑______________________|
      (loop, model decide karta hai ki kab rukna hai)
```
**Model decide karta hai** ki kaunsa tool, kitni baar, kab rukna hai.
**Kab use karo:** Jab steps pehle se pata nahi ho. Powerful, par mehnga, slow, aur non-deterministic.

### 🔑 Golden rule
> **Sabse simple paradigm choose karo jo kaam kar de.** Agent tab banao jab workflow genuinely fail ho raha ho — "cool lagta hai" isliye nahi.
>
> Reality: 70% cases (a) se ho jaate hain, 25% (b) se, sirf 5% ko (c) chahiye. Industry ka hype ulta ratio dikhata hai.

---

## 5. Modern AI ke 4 fundamental constraints

Ye 4 constraints tumhare **poore design ko drive karenge**. Inhe deeply internalize karo:

### Constraint 1: Model stateless hai
Model ko kuch yaad nahi rehta. Chat "history" ek illusion hai — tum har baar poori conversation dubara bhejte ho.

**Iska matlab:**
- Long conversation = har turn pe zyada tokens = zyada paisa + zyada latency
- "Memory" ek **engineering feature** hai, model ka feature nahi
- Isliye context management, compaction, summarization sab zaroori hai

### Constraint 2: Context window finite hai
Aaj ke best models 200K se 1M tokens tak jaate hain. Bada lagta hai, par:
- Bharne mein **paisa** lagta hai (input tokens bhi charge hote hain)
- Bharne se **quality girti hai** — "lost in the middle" problem (beech ki information ignore ho jaati hai)
- Bada context = **slow** response

> **Insight:** Bada context window "sab kuch daal do" ka license nahi hai. Ye "smartly bharo" ka challenge hai.

### Constraint 3: Model confidently galat ho sakta hai
Hallucination bug nahi hai — ye **architecture ka natural consequence** hai. Model probability se agla token chunta hai; usme "mujhe nahi pata" ka koi special signal nahi hai.

**Iska matlab:** Verification tumhari zimmedari hai. Citations, structured output validation, evals, human-in-the-loop.

### Constraint 4: Model non-deterministic hai
Same input, alag output (temperature > 0 pe guaranteed; 0 pe bhi hardware/batching se variance aata hai).

**Iska matlab:** Traditional unit tests kaam nahi karte. Tumhe **eval suites** chahiye — statistical, threshold-based testing.

---

## 6. AI Engineer ka actual kaam

Log sochte hain AI engineer ka kaam "prompt likhna" hai. Nahi. Actual kaam:

| % time | Kaam |
|---|---|
| 30% | **Context pipeline** — sahi data model tak pahunchana (RAG, tools, formatting) |
| 25% | **Evals** — measure karna ki kaam kar raha hai ya nahi |
| 20% | **Orchestration + error handling** — retries, fallbacks, timeouts, loops |
| 15% | **Cost/latency optimization** — caching, model routing, streaming |
| 10% | **Prompting** — haan, sirf 10% |

> Agar tum 80% time prompt tweak kar rahe ho, to tum galat layer pe kaam kar rahe ho. Problem lagbhag hamesha **context** mein hoti hai, prompt ke shabdon mein nahi.

---

## 7. Ek chhota reality check — 2026 mein kya badal gaya

| Purani soch (2023) | Aaj ki reality (2026) |
|---|---|
| "Bada context window sab solve kar dega" | Bada context ≠ better. Curation matters more |
| "Fine-tuning se best results" | Pehle prompting + RAG try karo. Fine-tuning 5% cases mein |
| "Agents production-ready nahi hain" | Agents production mein hain, par narrow scope ke saath |
| "LangChain hi standard hai" | Landscape split ho gaya — provider SDKs + LangGraph + custom |
| "Prompt engineering ek career hai" | Prompt engineering ek skill hai. Career = context + eval engineering |
| "GPU chahiye AI banane ke liye" | 95% AI products API calls hain. GPU sirf self-hosting mein |

---

## Khud se pooch (yahan ruko!) 🤔

Bina file dekhe, in sawaalon ka jawab do:

1. Model "stateless" hone ka tumhare architecture pe kya concrete asar padta hai? 2 examples do.
2. Workflow aur Agent mein exact farak kya hai? Kisne control kiya?
3. Hallucination ko "bug" kehna kyun galat hai?
4. AI stack ke 6 layers mein se **kaunsi 2 layers** mein tumhara competitive advantage hai, aur kyun?
5. Tumhare paas ek task hai: "customer support emails ko category mein daalo". Kya ye single call, workflow, ya agent hai? Kyun?

<details>
<summary>Answer 5 (dekhne se pehle khud socho)</summary>

**Single call.** Ek input, ek classification output, koi external data ki zaroorat nahi, koi multi-step decision nahi.
Isko agent banana = 10x cost, 5x latency, aur non-deterministic output — zero benefit ke liye.

Ha, agar requirement badle — "email padho, CRM se customer history nikalo, phir category decide karo aur ticket create karo" — tab **workflow** ban jaayega (steps pata hain). Agent tab chahiye jab "kya karna hai" bhi model ko decide karna pade.
</details>

---

**Next:** [`02-transformer-aur-attention.md`](02-transformer-aur-attention.md) — ab andar chalte hain. Ye poore repo ki sabse important file hai.
