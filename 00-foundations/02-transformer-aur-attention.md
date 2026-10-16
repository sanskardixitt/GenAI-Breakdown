# 02 — Transformer aur Attention ⭐ (Core file)

> **Is file ka goal:** Attention ko formula ki tarah nahi, **problem-solution** ki tarah samajhna. Iske baad tum "context window", "KV cache", "quadratic cost" — sab kuch naturally samajh jaoge.
>
> **Time:** 75 min. Jaldi na karo. Ye poore repo ki neev hai.
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/01-transformer-forward-pass`](../08-diagrams/01-transformer-forward-pass.excalidraw) aur [`08-diagrams/02-attention-qkv`](../08-diagrams/02-attention-qkv.excalidraw)

---

## Part 1: Problem kya thi? (Attention se pehle)

### RNN ki kahani

2016 mein language models RNN (Recurrent Neural Network) the. RNN aise kaam karta tha:

```
"main"  →  [state] → "ghar"  →  [state] → "jaa"  →  [state] → "raha"
             ↓                    ↓                   ↓
          h1 = f(h0, "main")   h2 = f(h1,"ghar")   h3 = f(h2,"jaa")
```

Ek shabd padho → state update karo → agla shabd padho. Bilkul human reading jaisa.

**Do fatal problems the:**

#### Problem 1: Sequential = slow
`h3` chahiye to pehle `h2` chahiye, uske liye `h1` chahiye. **Parallel nahi kar sakte.**
GPU ka poora point parallelism hai — RNN us power ko waste karta tha. 10,000 words ka document = 10,000 sequential steps.

#### Problem 2: Information bottleneck (bhoolna)
Poora past ek fixed-size vector `h` mein squeeze hota hai. 500 shabd baad, shabd #3 ki information almost gayab.

```
"Ravi Delhi mein rehta hai. [... 400 shabd ...] Wo APNE SHEHAR mein khush hai."
                                                       ↑
                        model ko "Delhi" yaad hona chahiye — par vector mein wo dhul gaya
```

LSTM/GRU ne isse *thoda* better kiya (gates ke through), par fundamentally solve nahi kiya.

---

## Part 2: Attention ka core idea

2017 ka paper — "Attention Is All You Need" — ne kaha:

> **"Sequential processing chhodo. Har shabd ko ek saath, sab dusre shabdon ko dekhne do — aur khud decide karne do ki kis pe dhyaan dena hai."**

### Analogy jo actually kaam karti hai 📚

Socho tum ek exam de rahe ho — **open book exam**.

- **RNN wala student:** Poori kitab ek baar padhta hai, notes banata hai (fixed size notebook), phir kitab band karke notes se answer likhta hai. Purane chapters ke notes dhundhle ho gaye.

- **Attention wala student:** Kitab **khuli** hai. Har sawaal ke liye wo *relevant pages* pe seedha jaata hai. Sawaal "Delhi" ke baare mein hai? Seedha us page pe. Distance matter nahi karta — page 3 ho ya page 300, ek hi effort.

**Yahi attention hai.** Har token, poore context ko "dekh" sakta hai, aur relevance ke hisaab se weight de sakta hai.

---

## Part 3: Q, K, V — asli mechanism

Ab thoda concrete. Attention mein har token ke 3 versions bante hain. Ye 3 naam intimidating lagte hain, par analogy simple hai — **library search**:

| Naam | Full form | Library analogy | Intuition |
|---|---|---|---|
| **Q** | Query | Tumhara **sawaal** | "Mujhe kya dhundhna hai?" |
| **K** | Key | Kitabon ke **labels/index** | "Main kis baare mein hoon?" |
| **V** | Value | Kitab ka **actual content** | "Mera actual matlab ye hai" |

### Process — 4 steps

Ek sentence lete hain: **"Ravi ne ball maari kyunki wo bhaari thi"**

Model ko decide karna hai: **"wo"** kis cheez ko refer kar raha hai — Ravi ya ball?

```
Step 1: "wo" apna Query banata hai
        Q("wo") = "main ek pronoun hoon, mujhe apna antecedent chahiye"

Step 2: Har dusre token ke Key se compare karo (dot product)
        Q("wo") · K("Ravi")   = 0.2   ← kam match
        Q("wo") · K("ball")   = 0.9   ← zyada match!
        Q("wo") · K("maari")  = 0.1
        Q("wo") · K("bhaari") = 0.4

Step 3: Scores ko softmax se probability banao (total = 1.0)
        Ravi: 0.10 | ball: 0.62 | maari: 0.06 | bhaari: 0.22

Step 4: Sab Values ka weighted average lo
        output("wo") = 0.10*V(Ravi) + 0.62*V(ball) + 0.06*V(maari) + 0.22*V(bhaari)
```

**Result:** "wo" ka naya representation mostly "ball" ki information rakhta hai. Model ne **seekh liya** ki "bhaari" hone ki wajah se "wo" = ball.

### Formula (ab ye darawna nahi lagega)

```
Attention(Q, K, V) = softmax( Q·Kᵀ / √d ) · V
                      ↑         ↑      ↑     ↑
                   normalize  match  scale  weighted
                   to 1.0     score        average
```

`√d` (scaling) sirf isliye hai ki bade dimensions mein dot products bahut bade ho jaate hain, aur softmax "saturate" ho jaata hai (ek token pe 0.999 weight, baaki 0). Scaling gradients ko healthy rakhta hai. **Bas itni hi baat hai.**

---

## Part 4: Multi-Head Attention — ek aur crucial idea

Ek attention "head" ek tarah ka relationship pakadta hai. Par language mein ek saath **kai relationships** hote hain:

```
"Ravi ne  Delhi  mein  apni  nayi  car  chalayi"
```

Ek hi sentence mein:
- **Grammar** relation: "chalayi" ← "Ravi" (kaun kar raha hai)
- **Object** relation: "chalayi" ← "car" (kya chala rahe hain)
- **Location** relation: "chalayi" ← "Delhi" (kahan)
- **Possession** relation: "car" ← "apni" (kiski)

Ek head sab nahi pakad sakta. Isliye **multi-head**: 32, 64, ya 128 parallel heads — har ek apna Q/K/V projection seekhta hai, apne tarah ka pattern pakadta hai. Phir sabke outputs concatenate hote hain.

> **Analogy:** Ek document ko 32 experts ek saath padh rahe hain. Ek grammar dekh raha hai, ek meaning, ek references, ek tone. Phir sab apni report deti hain aur combine ho jaati hai.

---

## Part 5: Poora Transformer block

Attention akela kaafi nahi. Ek transformer "block" mein ye hota hai:

```
        Input tokens (+ positional info)
                  ↓
        ┌─────────────────────┐
        │  Multi-Head         │  ← tokens ek dusre se "baat" karte hain
        │  Self-Attention     │     (information MIXING)
        └─────────────────────┘
                  ↓
            (+) Residual  ← input ko output mein wapas add karo
                  ↓
             LayerNorm     ← numbers ko stable rakho
                  ↓
        ┌─────────────────────┐
        │  Feed-Forward       │  ← har token AKELE apni information
        │  Network (MLP)      │     process karta hai (THINKING)
        └─────────────────────┘
                  ↓
            (+) Residual
                  ↓
             LayerNorm
                  ↓
              Output
```

Aur ye block **repeat** hota hai — 32, 80, 120 baar (model size ke hisaab se).

### Har part kya karta hai (ye table yaad rakho)

| Component | Kaam | Ek line mein |
|---|---|---|
| **Self-Attention** | Tokens ke beech information move karna | "Kis token ko kis pe dhyaan dena hai" — **communication** |
| **Feed-Forward (MLP)** | Har token ki information ko transform karna | "Ab is information ka kya matlab nikaalein" — **computation** |
| **Residual connection** | Input ko output mein add karna | Deep network mein gradient bachata hai; "highway" ban jaata hai |
| **LayerNorm** | Values ko normalize karna | Training stable rehti hai, numbers explode nahi hote |
| **Positional encoding** | Token ka **position** batana | Attention khud order nahi jaanta! (neeche detail) |

### 🔑 Sabse important intuition
> **Attention = communication. FFN = computation.**
>
> Layer ke andar tokens pehle ek dusre se baat karte hain (attention), phir akele sochte hain (FFN). Ye 80 baar repeat hota hai. Har repetition pe representation "richer" hota jaata hai — pehli layers syntax pakadti hain, beech ki layers meaning, aakhiri layers task-specific reasoning.

---

## Part 6: Positional Encoding — ek chhupa hua critical detail

Attention mein ek weird property hai: **usme order ka koi concept nahi hai.**

```
"kutta aadmi ko kaata"    →  same attention math
"aadmi kutta ko kaata"    →  same attention math
```

Kyunki attention sirf "sab tokens ka weighted average" hai — set operation hai, sequence operation nahi!

**Solution:** Har token ke embedding mein uski **position ki information** add kar dete hain.

| Method | Kaise | Kahan use hota hai |
|---|---|---|
| Sinusoidal | Sin/cos waves of different frequencies | Original 2017 paper |
| Learned | Position ka bhi embedding seekho | GPT-2 era |
| **RoPE** (Rotary) | Vector ko position ke hisaab se **rotate** karo | **Aaj ke almost sab models** |
| ALiBi | Distance ke hisaab se attention pe penalty | Some OSS models |

**RoPE kyun jeeta:** Ye *relative* position naturally encode karta hai (token 5 aur 7 ka relationship = token 105 aur 107 ka relationship), aur training se lambi sequences pe better extrapolate karta hai. **Isliye aaj 1M context windows possible hain** — RoPE ko scale/interpolate kiya jaa sakta hai.

> Jab tum sunte ho "model ka context window extend kiya gaya" — 90% baar wo RoPE scaling ki baat hai.

---

## Part 7: Decoder-only aur Causal Masking

Original transformer mein encoder + decoder dono the (translation ke liye). Aaj ke LLMs (GPT, Claude, Llama) **decoder-only** hain.

### Causal masking — ek zaroori trick

Training mein model ko poora text dikhta hai. Par agar model "agla shabd" predict kar raha hai, to use **aage ka shabd nahi dekhna chahiye** — warna wo cheat kar lega!

Solution: attention matrix mein future positions ko `-infinity` kar do (softmax ke baad 0 ho jaayenge).

```
        main  ghar  jaa   raha
main  [  ✓    ✗     ✗     ✗  ]   ← "main" sirf khud ko dekh sakta hai
ghar  [  ✓    ✓     ✗     ✗  ]   ← "ghar" apne se pehle wale dekh sakta hai
jaa   [  ✓    ✓     ✓     ✗  ]
raha  [  ✓    ✓     ✓     ✓  ]

✓ = dekh sakta hai    ✗ = masked (blocked)
```

Isse ek beautiful efficiency milti hai: **ek forward pass mein poore sentence ke saare predictions ek saath train ho jaate hain**, aur cheating bhi nahi hoti.

---

## Part 8: Practical consequences 🔥 (ye interview mein poocha jaata hai)

Ab asli value. Architecture se **directly** ye 4 practical facts nikalte hain:

### (1) Quadratic cost — O(n²)

`n` tokens hain to attention matrix `n × n` hoti hai. Har token har token ko dekhta hai.

| Tokens | Comparisons | Relative cost |
|---|---|---|
| 1,000 | 1,000,000 | 1x |
| 10,000 | 100,000,000 | 100x |
| 100,000 | 10,000,000,000 | 10,000x |

**Iska practical matlab:** Context 10x karne se cost 100x nahi badhti (kai optimizations hain — FlashAttention, sliding window, sparse attention), par **linear se zyada** badhti hai. Isliye "sab kuch context mein daal do" strategy mehngi hai.

### (2) KV Cache — generation ka sabse important optimization

Jab model token-by-token generate karta hai, har naye token pe **purane tokens ke K aur V dobara compute karne** ki zaroorat nahi — wo change nahi hote!

```
Generate "main":     compute K,V for "main"                 → cache mein daalo
Generate "ghar":     "main" ka K,V cache se lo, "ghar" ka naya compute karo
Generate "jaa":      "main","ghar" cache se, "jaa" naya
```

**Isse generation nearly linear ho jaata hai** instead of quadratic per token.

**Trade-off:** KV cache **GPU memory** khaata hai — aur bahut. Ye directly explain karta hai:
- Kyun long context inference mehnga hai (memory bound, compute bound nahi)
- Kyun **prompt caching** feature exist karta hai (ye essentially KV cache ko requests ke beech reuse karna hai)
- Kyun batch size limited hota hai self-hosting mein

> **Prompt caching** (Claude/OpenAI dono mein) = "prefix ka KV cache reuse karo". Isliye wo **prefix match** pe kaam karta hai — ek byte badla shuru mein, poora cache invalid. Ye architecture se directly aata hai, arbitrary API design nahi hai.

### (3) "Lost in the middle"

Attention weights spread ho jaate hain lambe context mein. Empirically models context ke **shuru aur ant** ko better use karte hain, beech ko weakly.

**Practical rule:** Sabse important information context ke **top** ya **bottom** pe rakho — beech mein nahi.

### (4) Har token ka compute equal hai

Model "easy" token pe kam aur "hard" token pe zyada compute nahi karta — har token same forward pass se guzarta hai.

**Isliye reasoning models exist karte hain:** agar model ko zyada "sochna" hai, to use **zyada tokens generate karne** padenge (thinking tokens). Compute ko time mein spread karne ka yahi ek tarika hai. Ye poori "test-time compute" revolution isi constraint se aayi hai — detail Day 2 mein.

---

## Part 9: MoE — ek modern variant (bonus)

Aaj ke bade models often **Mixture of Experts** hote hain. Idea:

FFN ko ek bade block ki jagah **N chhote "experts"** mein todo. Har token ke liye ek chhota "router" decide karta hai ki kaunse 2 experts use karne hain.

```
Dense model:   1 token → 1 bada FFN (saare parameters active)
MoE model:     1 token → router → 2 out of 64 experts (~3% parameters active)
```

**Fayda:** Total parameters bahut zyada (capacity zyada), par per-token compute kam. Isliye bade models sasta serve ho paate hain.

**Nuksaan:** Memory mein saare experts load karne padte hain (VRAM zyada), routing training unstable ho sakti hai, load balancing tricky hai.

> Jab tum padho "400B total params, 30B active params" — wo MoE hai.

---

## Part 10: Ek diagram jo sab jodta hai

```
"Ravi ghar" → [Tokenizer] → [15234, 8921]
                                  ↓
                          [Embedding lookup]
                          + [Positional (RoPE)]
                                  ↓
        ┌──────────── Block 1 ─────────────┐
        │  Attention (tokens baat karein)  │
        │  FFN (har token soche)           │
        └──────────────────────────────────┘
                                  ↓
        ┌──────────── Block 2 ─────────────┐   ← ye 80 baar
        └──────────────────────────────────┘      repeat hota hai
                                  ↓
                                 ...
                                  ↓
                        [Final LayerNorm]
                                  ↓
                    [Linear → vocab size (~128K)]
                                  ↓
                        [Softmax → probabilities]
                                  ↓
              "jaa": 0.34  "mein": 0.28  "ka": 0.09  ...
                                  ↓
                        [Sampling] → "jaa"
                                  ↓
                    Loop! Wapas input mein add karo
```

**Bas. Yahi poora LLM hai.** Har ChatGPT/Claude response is loop ka result hai.

---

## Khud se pooch 🤔

1. RNN ki **do** problems kya thi, aur attention ne dono ko kaise solve kiya?
2. Q, K, V ko apni analogy mein samjhao (library wali nahi — nayi banao).
3. Multi-head attention single-head se better kyun hai? Ek concrete language example do.
4. Positional encoding **kyun** zaroori hai? Uske bina kya toot jaayega?
5. Attention **quadratic** kyun hai? Aur generation ke waqt KV cache usse kaise bachata hai?
6. Prompt caching **prefix match** pe kyun kaam karta hai, arbitrary match pe nahi? (Architecture se answer do)
7. "Lost in the middle" ka tumhare RAG design pe kya asar padega?

<details>
<summary>Answer 6 (khud socho pehle!)</summary>

Prompt caching essentially **KV cache ko requests ke beech reuse** karna hai.

Token #500 ka K aur V, tokens #1–499 pe depend karte hain (attention causal hai — har token apne se pehle sab dekhta hai). Agar token #10 badal gaya, to #11 se aage sab tokens ke K,V **badal jaayenge** — unhe reuse nahi kar sakte.

Isliye cache sirf **exact prefix match** pe valid hai. "Beech mein ek shabd badla" = us point ke aage ka poora cache invalid.

**Practical consequence:** Apne prompt mein **stable content pehle** (system prompt, tools, examples) aur **variable content baad mein** (user query, timestamp) rakho. Agar tum system prompt mein `datetime.now()` daal do, tumhara cache hit rate **zero** ho jaayega — aur tumhe pata bhi nahi chalega, koi error nahi aayega.
</details>

---

**Next:** [`03-tokens-embeddings-context.md`](03-tokens-embeddings-context.md) — jo cheez model actually dekhta hai.
