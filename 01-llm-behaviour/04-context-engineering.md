# 04 — Context Engineering (Prompt engineering ka bada bhai)

> **Is file ka goal:** Mindset shift. Prompt engineering "shabd" ke baare mein hai; context engineering **"model ko kaunsi information milegi"** ke baare mein hai. **Yahi actual AI engineering hai.**
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/03-context-window-aur-kv-cache`](../08-diagrams/03-context-window-aur-kv-cache.excalidraw)

---

## Part 1: Shift kyun hua?

### 2023: Prompt engineering ka daur
```
Context window: 4K-8K tokens
Model: instructions weakly follow karta tha
→ Problem: "Model ko manaao ki wo theek se kaam kare"
→ Solution: clever wording, magic phrases, emphasis
```

### 2026: Context engineering ka daur
```
Context window: 200K-1M tokens
Model: instructions bahut closely follow karta hai
→ Problem: "Is bade context ko sahi information se bharo"
→ Solution: retrieval, curation, compaction, tool design
```

> 🔑 **Ek line mein:** Pehle model **weak** tha, isliye prompt strong likhna padta tha. Ab model **strong** hai, isliye **context accurate** dena padta hai.
>
> Aaj ka failure mode "model ne samjha nahi" nahi hai — "model ke paas sahi information nahi thi" hai.

---

## Part 2: Context ka budget model

Context ko ek **budget** ki tarah dekho, storage ki tarah nahi:

```
┌────────────────────────────────────────────────────────┐
│  TOTAL BUDGET: 200,000 tokens                          │
│                                                         │
│  FIXED COSTS (har request pe, cacheable):              │
│    System prompt / instructions      1,500             │
│    Tool definitions (8 tools)        3,200             │
│    Few-shot examples                 2,000             │
│                                    ────────            │
│                          Subtotal:   6,700             │
│                                                         │
│  VARIABLE COSTS (per request):                         │
│    RAG chunks (top-5)                4,000             │
│    Conversation history              8,000  ← manage!  │
│    Tool results (accumulate hote hain) 12,000 ← manage!│
│    Current user message                200             │
│                                    ────────            │
│                          Subtotal:  24,200             │
│                                                         │
│  RESERVED:                                              │
│    Output (thinking + answer)        8,000             │
│                                                         │
│  TOTAL USED: ~39,000  (20% of window)                  │
└────────────────────────────────────────────────────────┘
```

### 🚩 Bada context ≠ better output

Ye counter-intuitive fact bahut important hai:

```
Context filled:     10%     30%     60%     90%
Task accuracy:      95%     93%     87%     76%
```

**Kyun girta hai:**
1. **Attention dilution** — 100 relevant tokens + 100K irrelevant tokens = signal drown
2. **Lost in the middle** — beech ka content weakly use hota hai
3. **Distractor effect** — model plausible-but-wrong chunk pe ground kar leta hai
4. **Instruction drift** — system prompt bahut door chala gaya, uska weight kam

> **Real-world observation:** `top_k` 20 se 5 karne pe RAG accuracy **badh** jaati hai, ghatti nahi. Zyada context ne noise add kiya, signal nahi.

---

## Part 3: Context engineering ke 5 levers

### Lever 1: Selection — kya include karna hai

**Sawaal:** "Kya sab kuch fit ho sakta hai?" ❌
**Sahi sawaal:** "Is task ke liye **minimum** kya chahiye?" ✅

```python
# ❌ Bura — sab daal do
context = get_all_documents()               # 50K tokens
history = get_full_conversation()           # 30K tokens

# ✅ Acha — sirf relevant
context = retrieve_relevant(query, top_k=5)  # 4K tokens
history = get_recent_turns(n=6) + summary_of_older()  # 3K tokens
```

**Selection ke rules:**
- Har piece ka **justification** hona chahiye ("ye kyun context mein hai?")
- Agar nahi bata sakte → hata do
- Doubt hai to **kam** rakho — measure karke badhao

### Lever 2: Ordering — kahan rakhna hai

Position matter karti hai (attention ki wajah se):

```
┌──────────────────────────────┐
│ 🟢 STRONG   System prompt     │  ← rules, role
│ 🟢 STRONG   Tool definitions  │
│ 🟢 STRONG   Examples          │
│ 🟡 WEAK     ...               │
│ 🟡 WEAK     Bulk documents    │  ← beech mein bulk daalo
│ 🟡 WEAK     ...               │
│ 🟢 STRONG   Key constraints   │  ← important cheez repeat karo
│ 🟢 STRONG   User query        │  ← sabse aakhir mein
└──────────────────────────────┘
```

**Do rules:**
1. **Important content top ya bottom** — beech mein nahi
2. **Stable content pehle, variable baad mein** — prompt caching ke liye critical

```python
# ✅ Cache-friendly ordering
system   = STABLE_INSTRUCTIONS       # never changes → cached
tools    = STABLE_TOOL_DEFS          # never changes → cached
examples = STABLE_EXAMPLES           # never changes → cached
# ── cache breakpoint yahan ──
messages = [retrieved_chunks, history, user_query]   # changes per request
```

### Lever 3: Compression — kam tokens mein zyada

| Technique | Kaise | Saving |
|---|---|---|
| **Summarize old turns** | Purane 20 turns → ek 200-token summary | 80-90% |
| **Extract, don't dump** | Poora doc nahi, sirf relevant fields | 70-95% |
| **Structured > prose** | Table/JSON prose se compact hota hai | 30-50% |
| **Deduplicate** | Same info kai chunks mein? Ek rakho | 20-40% |
| **Truncate tool output** | 10,000-line log nahi, sirf errors | 90%+ |

```python
# ❌ Poora API response daal diya
tool_result = json.dumps(api_response)     # 8,000 tokens of nested JSON

# ✅ Sirf jo chahiye
tool_result = json.dumps({
    "status": api_response["status"],
    "items": [{"id": i["id"], "name": i["name"], "price": i["price"]}
              for i in api_response["data"]["items"][:10]],
    "total_count": api_response["meta"]["total"],
    "note": f"Showing 10 of {api_response['meta']['total']} results"
})                                          # 400 tokens
```

> **Tool output ko design karo.** Ye sabse zyada ignore kiya jaane wala lever hai. Ek badly-designed tool poora context window ek call mein bhar deta hai.

### Lever 4: Isolation — alag context, alag kaam

Sab kuch ek context mein daalne ke bajaye **kaam baato**:

```
❌ Ek bada context:
   [instructions + 20 tools + poori history + saare docs] → ek model

✅ Isolated contexts:
   Router (chhota context)    → decide kare kaunsa handler
      ↓
   Handler A (sirf iske 3 tools + relevant docs)
   Handler B (sirf iske 2 tools + apna data)
```

**Aur subagents:**
```
Main agent (context clean rehta hai)
   ↓ "in 50 files ko analyze karo"
   ├─ Subagent 1 (files 1-17)  → ek 200-token summary return
   ├─ Subagent 2 (files 18-34) → summary
   └─ Subagent 3 (files 35-50) → summary
   ↓
Main agent ko 600 tokens milte hain, 50 files ke 100K tokens nahi
```

Ye **context isolation** hai — modern multi-agent systems ka **main** justification (na ki "specialization").

### Lever 5: Persistence — context ke bahar memory

Sab kuch context mein rakhne ki zaroorat nahi. **Bahar store karo, zaroorat pe laao:**

```
Context (short-term)          External (long-term)
─────────────────────         ─────────────────────
Current task                  Vector DB (documents)
Last 6 turns                  File system (notes, artifacts)
Active tool results           SQL DB (structured facts)
                              Memory store (user preferences)
                                      ↓
                              tool call se retrieve karo
```

**Modern agent pattern:** Agent ko ek **memory file** do (ek plain `.md` bhi chalega) jisme wo apne learnings likhta jaaye. Ye context ke bahar rehti hai, par tool se accessible hai.

---

## Part 4: Context rot — agents ki sabse badi problem

Long-running agents mein context **degrade** hota jaata hai:

```
Step 1:  [system + tools + task]                      5K tokens  ✅ clean
Step 5:  [... + 4 tool results]                      25K tokens  ✅ theek
Step 15: [... + 14 tool results, kai stale]          80K tokens  ⚠️ noisy
Step 30: [... + 29 tool results, mostly irrelevant] 180K tokens  ❌ model kho gaya
```

**Symptoms:**
- Agent purani baat repeat karta hai
- Already-fail-hui approach dobara try karta hai
- Original task bhool jaata hai
- Loop mein phas jaata hai
- Cost exponentially badhti hai

### Solutions (production mein sab combine hote hain)

#### (a) Tool result pruning
```python
# Purane tool results ko placeholder se replace karo
if step_number - result_step > 5:
    result_content = "[purana tool result hata diya gaya — zaroorat ho to dobara call karo]"
```
Structure preserve rehta hai (message ordering valid), content chala gaya.

#### (b) Compaction (summarization)
```python
if token_count > 100_000:
    summary = llm_summarize(messages[2:-6],
        instruction="Task progress, key findings, failed approaches, "
                    "aur pending steps preserve karo.")
    messages = messages[:2] + [{"role": "user", "content": f"[Earlier: {summary}]"}] \
               + messages[-6:]
```

**Compaction ke saath 3 rules:**
1. **Failed approaches preserve karo** — warna agent same galti dohrayega
2. **Original task verbatim rakho** — paraphrase mein drift ho jaata hai
3. Compaction ek **extra LLM call** hai — cost aur latency add hoti hai

#### (c) Externalize state
```python
# ❌ Context mein sab track karo
"Ab tak main ne ye files padhi: a.py, b.py, ... (50 files list)"

# ✅ File mein track karo
write_file("progress.md", "Analyzed: a.py, b.py, ...\nPending: x.py, y.py")
# Context mein sirf: "progress.md mein status hai"
```

Ye **massively** better scale karta hai. Modern reasoning models ise naturally use karte hain jab tumhe file tools do.

#### (d) Fresh context restart
Ek naya context banao, sirf essentials carry karo:
```python
fresh = [
    {"role": "user", "content": f"""
Task: {original_task}

Ab tak ka progress:
{concise_summary}

Ye approaches FAIL ho chuki hain (dobara na try karo):
{failed_list}

Agla step: continue karo.
"""}
]
```

> **Kai baar clean restart, compaction se better hota hai.** Compaction noise ko compress karta hai; restart use **delete** kar deta hai.

---

## Part 5: Prompt caching — context engineering ka best friend

Recall (file 02): caching = KV cache reuse = **prefix match**.

```
Request 1: [system][tools][examples][query A]
                  ↑ cache write (1.25x cost)

Request 2: [system][tools][examples][query B]
            └──── cache read (0.1x cost!) ────┘  ← 90% saving
```

### ✅ Karo

```python
system   = FROZEN_INSTRUCTIONS           # kabhi na badlo
tools    = sorted(TOOLS, key=lambda t: t["name"])   # deterministic order!
examples = FROZEN_EXAMPLES
# ── cache breakpoint ──
messages = [*history, {"role": "user", "content": query}]
```

### ❌ Cache killers (silent — koi error nahi aayega!)

```python
# 💀 System prompt mein timestamp
system = f"You are helpful. Current time: {datetime.now()}"
#         → har request pe naya prefix → cache hit rate 0%

# 💀 Non-deterministic serialization
system = f"Config: {json.dumps(config)}"       # dict order badal sakta hai
#         fix: json.dumps(config, sort_keys=True)

# 💀 Per-user content prefix mein
system = f"You help {user.name} from {user.company}..."
#         → har user ka apna cache → cross-user sharing zero

# 💀 Tools per-user vary karna
tools = build_tools_for(user)        # tools position 0 pe hain — sab invalidate

# 💀 Conditional system sections
if flag_a: system += "..."           # har flag combination = alag cache entry
```

### Verify karo (assume na karo)

```python
print(response.usage.cache_read_input_tokens)      # >0 hona chahiye
print(response.usage.cache_creation_input_tokens)
print(response.usage.input_tokens)                 # uncached portion
```

> **Agar `cache_read_input_tokens` repeated requests pe 0 hai, ek silent invalidator hai.** Do requests ka rendered prompt byte-by-byte diff karo. Ye ek 30-minute debug session hai jo mahine ka bill bacha deti hai.

---

## Part 6: Ek practical checklist 📋

Naya feature banate waqt, ye poocho:

### Selection
- [ ] Context mein har piece ka justification hai?
- [ ] Kya `top_k` measure karke set kiya, guess se nahi?
- [ ] History pe hard cap hai?
- [ ] Tool outputs truncated/summarized hain?

### Ordering
- [ ] Stable content pehle (cacheable), variable baad mein?
- [ ] Important constraints top/bottom pe (beech mein nahi)?
- [ ] User query aakhir mein?

### Compression
- [ ] Tool responses **designed** hain, raw dump nahi?
- [ ] Purani history compact hoti hai?
- [ ] Duplicate information hataayi?

### Caching
- [ ] System prompt frozen hai (no timestamps/UUIDs)?
- [ ] Tools deterministic order mein serialize hote hain?
- [ ] Cache hit rate actually **measure** kar rahe ho?

### Agent-specific
- [ ] Tool result pruning strategy hai?
- [ ] Compaction threshold set hai?
- [ ] State external file/DB mein hai, context mein nahi?
- [ ] Failed approaches track hote hain?
- [ ] `max_iterations` cap hai?

---

## Part 7: Prompt engineering vs Context engineering

| | Prompt Engineering | Context Engineering |
|---|---|---|
| **Sawaal** | "Kaise likhoon?" | "Kya doon?" |
| **Unit** | Shabd, phrases | Documents, tool results, memory |
| **Fail mode** | Model galat samjha | Model ke paas info nahi thi |
| **Tools** | Wording, examples, format | Retrieval, chunking, compaction, tool design |
| **Skill level** | Entry | **Core AI engineering** |
| **Time share** | ~10% | ~60% |

> 🔑 **Final insight:** Jab tumhara AI feature fail kare, **90% baar problem context mein hai, prompt ke shabdon mein nahi.**
>
> Prompt tweak karne se pehle ye poocho: **"Kya model ke paas is sawaal ka jawab dene ke liye zaroori information thi?"**
> Agar nahi — koi prompt magic use fix nahi karega.

---

## Khud se pooch 🤔

1. Prompt engineering se context engineering shift kyun hua? Model ke saath kya badla?
2. Bada context accuracy kyun **kam** kar sakta hai? 4 reasons.
3. Context engineering ke 5 levers naam se batao. Har ek ka ek example do.
4. "Context rot" kya hai? Uske 4 symptoms aur 4 solutions?
5. Compaction karte waqt failed approaches preserve karna kyun zaroori hai?
6. Ye code mein 3 cache killers dhundo:
```python
system = f"You are an assistant for {user.company}. Time: {datetime.now()}"
tools = build_tools(user.permissions)
messages = [{"role": "user", "content": query}]
```
7. Tumhara agent 30 steps ke baad "confused" ho jaata hai aur loop karta hai. 4 fixes batao (priority order mein).

<details>
<summary>Answer 6 aur 7</summary>

**6. Teen killers:**
1. **`{datetime.now()}`** — har request pe badalta hai → cache **hamesha** miss. Sabse bada.
2. **`{user.company}`** — per-user prefix → users ke beech koi sharing nahi. 1000 users = 1000 separate caches.
3. **`build_tools(user.permissions)`** — tools position 0 pe render hote hain. Different permissions = different tools = **poora** cache invalid, system prompt bhi.

**Fix:**
```python
system = FROZEN_SYSTEM_PROMPT              # koi interpolation nahi
tools  = ALL_TOOLS_SORTED                  # sab users ke liye same, deterministic order
# ── cache breakpoint ──
messages = [
    {"role": "user", "content":
        f"<context>Company: {user.company}\nDate: {date.today()}</context>\n\n{query}"}
]
# Permission enforcement EXECUTION layer pe karo, tool definitions pe nahi:
# execute_tool() mein check karo ki user ko ye tool allowed hai ya nahi.
```
Note: `date.today()` (time nahi) — din bhar stable rahega. Aur permission ko tool *definitions* se hatakar tool *execution* pe le gaye — security same, cache bach gaya.

**7. Fixes, priority order:**

1. **Tool result pruning** (sabse bada, sabse aasan) — 5 steps se purane results ko placeholder se replace karo. Ye usually 60-70% context bachata hai turant.
2. **State externalize karo** — agent ko `progress.md` file do jisme wo findings/pending/failed likhe. Context mein sirf pointer rahe. Ye "confusion" ka root fix hai — agent ka working memory context se bahar aa gaya.
3. **Compaction threshold** — 100K tokens pe summarize karo, aur **failed approaches explicitly preserve** karo (warna loop wahi se aayega).
4. **`max_iterations` cap + loop detection** — agar same tool same args ke saath 3 baar call hua, to break karo aur model ko batao: *"Tum ye call repeat kar rahe ho. Alag approach lo ya user se poochho."*

**Aur ek stepping-back sawaal:** 30 steps kya **zaroori** hain? Kai baar ye harness problem hai — better tools (ek tool jo 3 kaam kare), ya kuch steps ko deterministic code se replace karna, step count half kar deta hai. Aur agar steps predictable hain, to ye **workflow** hai, agent nahi.
</details>

---

## 🎉 Day 2 complete!

Ab tum jaante ho:
- ✅ Modern prompting — aur kaunse purane tricks **hataane** hain
- ✅ Reasoning models: kaise kaam karte hain, kab use karna hai
- ✅ Tool calling ka exact mechanism (JSON + loop, magic nahi)
- ✅ Context engineering — asli AI engineering skill

**Next:** [`../02-rag/01-rag-refresher-aur-kyun-toota.md`](../02-rag/01-rag-refresher-aur-kyun-toota.md) — Day 3, RAG deep dive.
