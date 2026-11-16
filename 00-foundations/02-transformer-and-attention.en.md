# 02 — Transformers and Attention ⭐ (Core file)

> **Goal of this file:** Understand attention not as a formula, but as a **solution to a problem**. Once you do, "context window", "KV cache", and "quadratic cost" will all make sense on their own.
>
> **Time:** 75 min. Don't rush. This is the foundation for the rest of the repo.
>
> 📊 **Keep the diagrams open alongside:** [`08-diagrams/01-transformer-forward-pass`](../08-diagrams/01-transformer-forward-pass.excalidraw) and [`08-diagrams/02-attention-qkv`](../08-diagrams/02-attention-qkv.excalidraw)
>
> *Model examples and numbers reflect publicly documented architectures as of 2025–26.*

---

## Part 1: What was the problem? (Before Transformers)

### The RNN story

Until around 2017, the best language models were RNNs (Recurrent Neural Networks). An RNN works like this:

```
"I"  →  [state] → "am"  →  [state] → "going"  →  [state] → "home"
          ↓                  ↓                     ↓
       h1 = f(h0, "I")    h2 = f(h1, "am")      h3 = f(h2, "going")
```

Read one word → update the state → read the next word. Very much like how a human reads.

**It had two fatal problems:**

#### Problem 1: Sequential = slow
To compute `h3` you need `h2`, and for that you need `h1`. **You can't parallelize across the sequence.**
The whole point of a GPU is parallelism, and RNNs wasted it. A 10,000-word document meant 10,000 sequential steps.

#### Problem 2: The information bottleneck (forgetting)
The entire past is squeezed into one fixed-size vector `h`. Five hundred words later, the information about word #3 is almost gone.

```
"Ravi lives in Delhi. [... 400 words ...] He is happy in HIS CITY."
                                                          ↑
                      the model needs to remember "Delhi" — but it has faded out of the vector
```

LSTMs and GRUs made this *somewhat* better (through gating), but didn't fundamentally solve it.

> **History note:** Attention itself predates Transformers. In 2014, Bahdanau et al. added an attention mechanism *on top of* an RNN for translation, so the decoder could look back at all encoder states. The 2017 breakthrough was showing that you could **drop the recurrence entirely** and build the whole model out of attention.

---

## Part 2: The core idea of attention

The 2017 paper "Attention Is All You Need" (Vaswani et al.) proposed:

> **"Drop sequential processing. Let every word look at every other word at the same time, and let the model learn for itself what to pay attention to."**

### An analogy that actually works 📚

Imagine you're taking an exam, and it's **open book**.

- **The RNN student** reads the whole book once, writes notes in a fixed-size notebook, then closes the book and answers from the notes. The notes on early chapters have become blurry.

- **The attention student** keeps the book **open**. For each question, they go straight to the *relevant pages*. Is the question about "Delhi"? They flip straight to that page. Distance doesn't matter: page 3 or page 300 takes the same effort.

**That's attention.** Every token can "see" the entire context and weight each part by relevance.

---

## Part 3: Q, K, V — the actual mechanism

Now let's get concrete. In attention, each token is projected into three vectors. The names sound intimidating, but the analogy is simple: a **library search**.

| Name | Stands for | Library analogy | Intuition |
|---|---|---|---|
| **Q** | Query | Your **question** | "What am I looking for?" |
| **K** | Key | The books' **labels / catalog entries** | "What am I about?" |
| **V** | Value | The book's **actual content** | "Here's the information I carry" |

Mechanically, each one is just the token's vector multiplied by a learned weight matrix: `Q = x·W_Q`, `K = x·W_K`, `V = x·W_V`. Training learns what makes a good query, key, and value.

### The process — 4 steps

Take the sentence: **"Ravi couldn't lift the box because it was too heavy."**

The model has to work out what **"it"** refers to: Ravi or the box?

```
Step 1: "it" forms its Query
        Q("it") ≈ "I'm a pronoun; I need to find what I refer to"

Step 2: Compare it against every token's Key (dot product, then scale)
        Q("it") · K("Ravi")  = 0.6
        Q("it") · K("lift")  = 0.1
        Q("it") · K("box")   = 2.4   ← strongest match!
        Q("it") · K("heavy") = 1.4

Step 3: Turn the scores into probabilities with softmax (they sum to 1.0)
        Ravi: 0.10 | lift: 0.06 | box: 0.61 | heavy: 0.23

Step 4: Take a weighted average of the Values
        output("it") = 0.10·V(Ravi) + 0.06·V(lift) + 0.61·V(box) + 0.23·V(heavy)
```

*(Numbers are illustrative but consistent: softmax of the Step 2 scores gives the Step 3 weights. In reality every token attends to every token, including itself and words like "the".)*

**Result:** The new representation of "it" mostly carries information about "box". The model has **learned** that something being "heavy" and "lifted" points to the box, not to Ravi.

### The formula (it won't look scary now)

```
Attention(Q, K, V) = softmax( Q·Kᵀ / √d_k ) · V
                      ↑         ↑      ↑      ↑
                   normalize  match  scale  weighted
                   to 1.0     score         average
```

The `√d_k` scaling exists only because, in high dimensions, dot products get very large, and softmax then "saturates" (0.999 weight on one token, ~0 on the rest). That kills the gradients. Dividing by `√d_k` keeps the scores in a healthy range. **That's all there is to it.**

---

## Part 4: Multi-Head Attention — another crucial idea

One attention "head" tends to capture one kind of relationship. But language has **many relationships at once**:

```
"Ravi drove his new car in Delhi"
```

In this one sentence:
- **Subject** relation: "drove" ← "Ravi" (who did it)
- **Object** relation: "drove" ← "car" (what was driven)
- **Location** relation: "drove" ← "Delhi" (where)
- **Possession** relation: "car" ← "his" (whose)

A single head can't capture all of these well. Hence **multi-head attention**: many heads in parallel (typically 32–128 in large models), each with its own Q/K/V projections, each learning its own kind of pattern. Their outputs are concatenated and mixed by one more linear layer.

> **Analogy:** 32 experts read the same document at the same time. One tracks grammar, one meaning, one references, one tone. Then they all hand in their reports, and the reports are combined.

### Modern twist: sharing Keys and Values (GQA, MLA)

Separate K and V for every head is expensive at inference time (see the KV cache in Part 8). Modern models cut this down:

| Variant | Idea | Used by |
|---|---|---|
| **MHA** (Multi-Head) | Every head has its own Q, K, V | Original Transformer, GPT-2/3 |
| **MQA** (Multi-Query) | All query heads share **one** K/V head | PaLM, Falcon |
| **GQA** (Grouped-Query) | Query heads are split into groups; each group shares a K/V head (e.g., 64 Q heads, 8 KV heads) | Llama 2 70B / Llama 3, Mistral, Qwen, Gemma: the current default |
| **MLA** (Multi-head Latent) | Compress K/V into a small latent vector and reconstruct them on the fly | DeepSeek-V2/V3/R1, Kimi K2 |

The quality loss is small, and the memory savings are large (often 4–8× less KV cache or more).

---

## Part 5: The full Transformer block

Attention alone isn't enough. A modern transformer "block" looks like this:

```
              Input x
                 │
         ┌───────┤
         │       ↓
         │    RMSNorm                 ← normalize first ("pre-norm")
         │       ↓
         │  Multi-Head Self-Attention  ← tokens "talk" to each other
         │  (RoPE applied to Q and K)    (information MIXING)
         │       ↓
         └────► (+)                   ← residual: add the input back
                 │
         ┌───────┤
         │       ↓
         │    RMSNorm
         │       ↓
         │  Feed-Forward Network       ← each token is processed ON ITS OWN
         │  (SwiGLU MLP, or MoE)         (THINKING)
         │       ↓
         └────► (+)                   ← residual
                 │
              Output
```

This block is **repeated** many times: 32 layers for Llama 3 8B, 80 for Llama 3 70B, 126 for Llama 3.1 405B.

> **What changed since 2017:** The original paper used **post-norm** (LayerNorm *after* each residual add) and a ReLU MLP. Almost every modern LLM uses **pre-norm** (normalize *before* attention/FFN, which trains much more stably at depth), **RMSNorm** instead of LayerNorm (simpler and faster), and **SwiGLU** feed-forward layers.

### What each part does (memorize this table)

| Component | Job | In one line |
|---|---|---|
| **Self-Attention** | Moves information between tokens | "Which token should pay attention to which": **communication** |
| **Feed-Forward (MLP)** | Transforms each token's information | "What does this information mean?": **computation**. It holds most of the parameters and much of the stored "knowledge" |
| **Residual connection** | Adds the input back to the output | Keeps gradients flowing through deep networks; acts as a "highway" |
| **Normalization** (RMSNorm/LayerNorm) | Rescales values | Keeps training stable; stops numbers from exploding |
| **Positional encoding** | Tells the model each token's **position** | Attention has no built-in notion of order! (details below) |

### 🔑 The most important intuition
> **Attention = communication. FFN = computation.**
>
> Inside each layer, tokens first talk to each other (attention), then think on their own (FFN). This repeats dozens of times. With each repetition the representations get "richer". Interpretability research suggests a rough progression: early layers handle surface features and syntax, middle layers build up meaning and facts, and later layers shape the specific prediction. It's a tendency, not a strict division.

---

## Part 6: Positional Encoding — a hidden but critical detail

Attention has a strange property: **it has no concept of order.**

```
"dog bites man"    →  same set of attention scores
"man bites dog"    →  same set of attention scores (just permuted)
```

That's because attention is just "a weighted average over all tokens". It's a **set** operation, not a **sequence** operation. (Causal masking, covered in Part 7, leaks a little order information, but not nearly enough.)

**The fix:** Inject each token's **position** into the computation.

| Method | How it works | Where it's used |
|---|---|---|
| Sinusoidal | Add sin/cos waves of different frequencies to the embeddings | Original 2017 paper |
| Learned absolute | Learn an embedding for each position | BERT, GPT-2, GPT-3 |
| **RoPE** (Rotary) | **Rotate** the Q and K vectors by an angle that depends on position | **Nearly every modern LLM** (Llama, Qwen, Mistral, Gemma, DeepSeek, gpt-oss…) |
| ALiBi | Penalize attention scores in proportion to distance | BLOOM, MPT |

**Why RoPE won:** Because Q and K are rotated, their dot product depends only on the **relative** distance between tokens (the relationship between tokens 5 and 7 is the same as between 105 and 107). It's also applied inside attention rather than added to embeddings, and it can be **stretched**.

**Context extension:** Techniques like Position Interpolation, NTK-aware scaling, **YaRN**, and LongRoPE rescale RoPE's frequencies, and the model is then fine-tuned briefly on long sequences. This is a big part of how context windows went from 2K–4K tokens (2022) to **128K as standard and 1M+ on several frontier models** today (Gemini, Claude, GPT-4.1; Llama 4 Scout advertises 10M).

> When you hear "the model's context window was extended", it almost always involves some form of RoPE scaling plus long-context training. Some recent models also interleave layers with **no** positional encoding (e.g., Llama 4's "iRoPE") to help generalize to longer lengths.

---

## Part 7: Decoder-only and Causal Masking

The original Transformer had both an encoder and a decoder (it was built for translation). Encoder-only models like BERT are still used for embeddings and classification. But today's generative LLMs (GPT, Llama, Qwen, DeepSeek, Mistral, and, as far as is publicly known, Claude and Gemini) are **decoder-only**.

### Causal masking — an essential trick

During training the model sees the whole text at once. But when it's predicting "the next word", it **must not see the words that come after**, or it would just copy the answer.

The fix: set the attention scores for future positions to `-infinity` (after softmax they become 0).

```
         I    am   going  home
I      [ ✓    ✗     ✗     ✗  ]   ← "I" can only see itself
am     [ ✓    ✓     ✗     ✗  ]   ← "am" can see itself and everything before it
going  [ ✓    ✓     ✓     ✗  ]
home   [ ✓    ✓     ✓     ✓  ]

✓ = can attend    ✗ = masked (blocked)
```

This gives you a beautiful efficiency: **a single forward pass trains on every next-token prediction in the sequence at once**, with no cheating.

---

## Part 8: Practical consequences 🔥 (this comes up in interviews)

Now for the real payoff. These practical facts follow **directly** from the architecture.

### (1) Quadratic cost — O(n²)

With `n` tokens, the attention score matrix is `n × n`: every token is compared with every other token.

| Tokens | Comparisons | Relative attention cost |
|---|---|---|
| 1,000 | 1,000,000 | 1× |
| 10,000 | 100,000,000 | 100× |
| 100,000 | 10,000,000,000 | 10,000× |

**What this means in practice:**

- **Total model cost doesn't grow purely quadratically at first.** The FFN and projection layers grow linearly with `n`, and at short and medium lengths they dominate. Roughly, attention takes over when `n` gets larger than about 6× the hidden size. For a model with hidden size ~8K, that's somewhere in the tens of thousands of tokens.
- **FlashAttention does not remove the O(n²) compute.** It computes *exact* attention in tiles without ever storing the full `n × n` matrix in GPU memory, so memory becomes linear and it runs much faster. The arithmetic is still quadratic.
- **What actually breaks the quadratic barrier** is changing the attention pattern:
  - **Sliding-window / local attention:** each token only sees the last *w* tokens. Usually it's mixed with some full-attention layers (Gemma 3 uses 5 local layers for every 1 global layer; gpt-oss alternates them).
  - **Sparse attention:** each token attends only to a selected subset (e.g., DeepSeek Sparse Attention in DeepSeek-V3.2).
  - **Linear attention / state-space hybrids:** most layers use a linear-time mechanism (Mamba, Gated DeltaNet, "lightning attention"), with a few full-attention layers kept in. Examples: Jamba, MiniMax-M1, Qwen3-Next.

Bottom line: "just stuff everything into the context" is still an **expensive** strategy, in cost, latency, and (see point 3) quality.

### (2) KV Cache — the most important generation optimization

Generation happens token by token. When a new token is generated, the **K and V of earlier tokens don't need to be recomputed**, because thanks to causal masking they never change!

```
Generate "I":      compute K,V for "I"                        → store in cache
Generate "am":     reuse "I" from cache, compute K,V for "am" only
Generate "going":  reuse "I","am" from cache, compute "going" only
```

Without a cache, each new token would mean reprocessing the entire sequence. With it, each new token only does **one row** of attention against the cached keys, which is linear in the context length.

This splits inference into two phases:

| Phase | What happens | Bottleneck |
|---|---|---|
| **Prefill** | Process the whole prompt in parallel and build the KV cache | **Compute-bound**. Shows up as *time to first token* |
| **Decode** | Generate one token at a time, reading the entire KV cache each step | **Memory-bandwidth-bound**. Shows up as *tokens per second* |

**The trade-off:** the KV cache consumes **GPU memory**, and a lot of it. You can estimate the size yourself:

```
KV cache per token = 2 (K and V) × layers × kv_heads × head_dim × bytes_per_value

Llama 3 70B (80 layers, 8 KV heads via GQA, head_dim 128, FP16 = 2 bytes):
  2 × 80 × 8 × 128 × 2 = 327,680 bytes ≈ 320 KB per token
  × 128K tokens ≈ 40 GB  for ONE sequence
  (with full MHA, i.e. 64 KV heads, it would be 8× that: ~320 GB)
```

That directly explains:
- Why **long-context inference is expensive**: the cache dominates memory and bandwidth
- Why **GQA / MLA**, **KV-cache quantization** (FP8/INT8), and **PagedAttention** (vLLM's paged memory for the cache) exist
- Why **batch size is limited** when you self-host: every concurrent request needs its own cache
- Why **prompt caching** exists: it is essentially reusing the KV cache *across requests*

> **Prompt caching** (Anthropic, OpenAI, Google, and inference servers like vLLM/SGLang all offer it) = "reuse the KV cache for a shared **prefix**". That's why it only works on an **exact prefix match**: change one token near the start, and everything after it is invalidated. Providers bill cached input tokens at a large discount and return the first token faster. This behavior comes straight from the architecture; it isn't an arbitrary API design choice. (Anthropic uses explicit cache breakpoints; OpenAI and Gemini also cache long prefixes automatically.)

### (3) "Lost in the middle" and context rot

With a long context, attention gets spread thin. The original "Lost in the Middle" study (Liu et al., 2023) found that models use information at the **start and end** of the context much better than information in the middle.

**What's changed since then:** Frontier models now score near-perfectly on simple "needle in a haystack" tests. But harder long-context benchmarks (RULER, NoLiMa, and Chroma's 2025 "context rot" study) show that performance still **degrades as input length grows**, especially when the model has to *reason over* or *combine* several facts rather than just retrieve one string.

**Practical rules:**
- Put the most important information (instructions, the key question) at the **top** or **bottom** of the context, not buried in the middle.
- **More context ≠ better answers.** Retrieving 5 relevant chunks often beats dumping in 50.

### (4) Every token gets the same amount of compute

The model doesn't spend less compute on an "easy" token and more on a "hard" one. Every token goes through the same forward pass.

**That's why reasoning models exist:** if the model needs to "think harder", the only way is to **generate more tokens** (thinking tokens) and spread the compute over time. The whole **"test-time compute"** shift (OpenAI o1/o3, DeepSeek-R1, Claude's extended thinking, Gemini's thinking models) comes from this constraint. More on this in Day 2.

---

## Part 9: MoE — Mixture of Experts (the modern default at scale)

Most of today's largest open-weight models are **Mixture of Experts**. The idea:

Instead of one big FFN per layer, split it into **many smaller "experts"**. For each token, a small learned **router** picks the top-*k* experts to use. Attention layers stay dense; only the FFN is sparse.

```
Dense model:   1 token → 1 big FFN (all parameters active)
MoE model:     1 token → router → top-k of N experts (+ often 1 always-on "shared" expert)
```

**Real examples:**

| Model | Total params | Active per token | Experts |
|---|---|---|---|
| Mixtral 8x7B (2023) | ~47B | ~13B | 8, top-2 |
| DeepSeek-V3 / R1 | 671B | 37B | 256 routed (top-8) + 1 shared |
| Qwen3-235B-A22B | 235B | 22B | 128, top-8 |
| Llama 4 Maverick | ~400B | 17B | 128 routed + 1 shared |
| gpt-oss-120b | ~117B | ~5.1B | 128, top-4 |
| Kimi K2 | ~1T | 32B | 384, top-8 + 1 shared |

The trend is toward **fine-grained** experts (many small ones, with more selected per token) plus a shared expert for common knowledge.

**Benefit:** Huge total parameter count (more capacity and knowledge) with low per-token compute. That's how very large models can be served cheaply.

**Costs:** All experts must sit in memory (lots of VRAM), and serving across GPUs needs "expert parallelism". The router also has to keep the load balanced across experts during training. This used to be handled with auxiliary losses; DeepSeek-V3 popularized an auxiliary-loss-free bias-adjustment method.

> When you read "235B-A22B" or "400B total, 17B active", that's an MoE.

---

## Part 10: One diagram that ties it all together

```
"Ravi went" → [Tokenizer] → [15234, 8921]          (IDs are illustrative)
                                  ↓
                         [Embedding lookup]
                                  ↓
        ┌──────────── Block 1 ─────────────┐
        │  RMSNorm → Attention (RoPE)      │  ← tokens talk to each other
        │  RMSNorm → FFN / MoE             │  ← each token thinks
        │  + residual around each          │
        └──────────────────────────────────┘
                                  ↓
        ┌──────────── Block 2 ─────────────┐   ← repeated 32–120+ times
        └──────────────────────────────────┘
                                  ↓
                                 ...
                                  ↓
                          [Final RMSNorm]
                                  ↓
             [Linear → vocabulary size (~100K–260K)]
                                  ↓
                     [Softmax → probabilities]
                                  ↓
            "home": 0.34  "to": 0.28  "back": 0.09  ...
                                  ↓
                       [Sampling] → "home"
                                  ↓
            Loop! Append it to the input (KV cache grows by one)
```

**That's it. That's the whole LLM.** Every ChatGPT, Claude, or Gemini response is the result of this loop.

---

## Ask yourself 🤔

1. What were the **two** problems with RNNs, and how does attention solve each one?
2. Explain Q, K, V with your own analogy (not the library one; come up with a new one).
3. Why is multi-head attention better than single-head? Give a concrete language example.
4. **Why** is positional encoding necessary? What would break without it?
5. Why is attention **quadratic**? Does FlashAttention change that? How does the KV cache help during generation?
6. Why does prompt caching work on a **prefix match** and not an arbitrary match? (Answer from the architecture.)
7. How should "lost in the middle" / context rot affect your RAG design?
8. Using the KV-cache formula, why does switching from MHA to GQA with 8 KV heads let you serve more users on the same GPU?

<details>
<summary>Answer to 6 (think it through yourself first!)</summary>

Prompt caching is essentially **reusing the KV cache across requests**.

The K and V for token #500 depend on tokens #1–499, because attention is causal and every token's representation is built from everything before it, at every layer. If token #10 changes, the K/V of every token from #10 onward **changes too**, so none of it can be reused.

That's why a cache is only valid for an **exact prefix match**. "Changed one word in the middle" = everything after that point is invalidated.

**Practical consequence:** Put **stable content first** (system prompt, tool definitions, few-shot examples, long documents) and **variable content last** (the user's query, timestamps, per-request IDs). If you put `datetime.now()` at the top of your system prompt, your cache hit rate drops to **zero**, and you won't even notice, because nothing throws an error. Check the cache-hit fields in the API's usage response to verify it's working.
</details>

---

**Next:** [`03-tokens-embeddings-context.md`](03-tokens-embeddings-context.md) — what the model actually sees.
