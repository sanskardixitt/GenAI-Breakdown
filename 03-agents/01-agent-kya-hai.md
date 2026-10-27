# 01 — Agent kya hai? (Hype hataake)

> **Is file ka goal:** Agent ko demystify karna, aur — **zyada important** — samajhna ki **kab agent NAHI banana chahiye.** Ye doosra part 90% log miss karte hain.

---

## Part 1: Definition — bilkul simple

> **Agent = ek LLM jo ek loop mein chal raha hai, tools ke saath, aur khud decide kar raha hai ki kab rukna hai.**

Bas. Itna hi.

```python
# Ye ek POORA agent hai. 12 lines.
def agent(task, tools, max_steps=20):
    messages = [{"role": "user", "content": task}]
    for _ in range(max_steps):
        response = llm(messages, tools=tools)
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return response                                  # model ne rok diya
        results = [execute(tc) for tc in response.tool_calls]
        messages.append({"role": "user", "content": results})
    raise MaxStepsExceeded()
```

**Koi framework nahi. Koi magic nahi.** Ek `for` loop, ek LLM call, ek dictionary lookup.

Jab tum "AI agent framework" ki 50,000 lines dekho, yaad rakho: **wo ye 12 lines ke aage-peeche ki convenience hai.** Kuch convenience genuinely valuable hai; kuch complexity hai jo tumhe nahi chahiye.

---

## Part 2: Teen paradigms — precise definitions

Ye distinction poore repo ka sabse practical framework hai:

### (a) Single call
```
input → LLM → output
```
| | |
|---|---|
| Control | Tum (koi loop nahi) |
| Steps | 1 |
| Determinism | High |
| Cost | 1x |
| Debug | Trivial |
| **Use for** | Classification, extraction, summarization, rewriting, translation |

### (b) Workflow (chain / pipeline)
```
input → LLM₁ → code → LLM₂ → validate → output
```
| | |
|---|---|
| Control | **Tum** — steps code mein hardcoded hain |
| Steps | Fixed, known |
| Determinism | Medium-high (path fixed hai) |
| Cost | N× (predictable) |
| Debug | Easy — har step inspect kar sakte ho |
| **Use for** | Multi-step tasks jinke steps tum jaante ho |

```python
# Workflow example — steps FIXED hain
def process_invoice(pdf):
    text     = extract_text(pdf)                    # code
    fields   = llm_extract(text, schema=InvoiceSchema)  # LLM step 1
    validate(fields)                                 # code — fail fast
    vendor   = db.lookup(fields.vendor_id)           # code
    decision = llm_approve(fields, vendor.rules)     # LLM step 2
    return decision
```

### (c) Agent
```
input → LLM ⇄ tools (loop) → output
         ↑ MODEL decide karta hai: kaunsa tool, kitni baar, kab rukna
```
| | |
|---|---|
| Control | **Model** |
| Steps | Variable, unknown upfront |
| Determinism | **Low** |
| Cost | Variable, **unpredictable** (10-100x) |
| Debug | Hard — har run alag path |
| **Use for** | Tasks jinke steps pehle se pata nahi |

### 🔑 Ek line mein farak

> **Workflow:** *Tum* decide karte ho ki kya karna hai.
> **Agent:** *Model* decide karta hai ki kya karna hai.

Yahi single distinction hai. Sab baaki isse follow hota hai.

---

## Part 3: Kab agent banao? (4-question test)

Ye 4 sawaal poocho. **Koi ek "nahi" = agent na banao.**

### 1. Complexity: Task multi-step aur pehle se unspecifiable hai?

```
❌ "PDF se invoice fields extract karo"
   → Steps pata hain. Workflow.

✅ "Ye bug production mein kyun aa raha hai — dhundo aur fix karo"
   → Steps pata nahi. Kaunsi files padhni hain? Kaunse logs? Depends.
```

### 2. Value: Outcome extra cost/latency justify karta hai?

```
❌ Support ticket categorize karo (₹0.01 value)
   → Agent ka cost ₹2 hai. Absurd.

✅ Ek din ka debugging kaam automate karo (₹50,000 value)
   → Agent ka ₹200 cost trivial hai.
```

### 3. Viability: Model is task type mein actually capable hai?

```
❌ "Hamara poora legacy COBOL system Rust mein migrate karo"
   → Model reliably ye nahi kar sakta. Agent 3 din ghoomega aur fail hoga.

✅ "In 20 files mein deprecated API calls update karo"
   → Well-scoped, model capable.
```

### 4. Cost of error: Galti pakdi aur recover ki jaa sakti hai?

```
❌ Agent production database pe direct writes kare
   → Galti irreversible. Bahut risky.

✅ Agent code likhe, tests chalaye, PR banaye
   → Tests catch karenge. Human review karega. Git revert possible.
```

### Decision table

| Task | Complexity | Value | Viability | Error cost | Verdict |
|---|---|---|---|---|---|
| Email classify | ❌ Simple | ❌ Low | ✅ | ✅ | **Single call** |
| Invoice process | ⚠️ Known steps | ✅ | ✅ | ⚠️ | **Workflow** |
| Codebase refactor | ✅ | ✅ | ✅ | ✅ (tests) | **Agent** ✅ |
| Research report | ✅ | ✅ | ✅ | ✅ | **Agent** ✅ |
| Auto-approve loans | ⚠️ | ✅ | ⚠️ | ❌ Irreversible | **Workflow + human** |
| DB migration | ✅ | ✅ | ⚠️ | ❌ | **Human + AI assist** |

---

## Part 4: 🚩 Agents ki asli lagat (jo demos mein nahi dikhti)

### Cost multiplication

```
Single call:  1 LLM call    = ~$0.01
Workflow:     3 LLM calls   = ~$0.03    (predictable)
Agent:        8-30 calls    = ~$0.15-1.50  (UNPREDICTABLE!)
```

Aur — critical — **context har step pe badhta hai:**

```
Step 1:  5,000 tokens input
Step 5:  25,000 tokens input     ← har step pe poori history dobara!
Step 10: 60,000 tokens input
Step 20: 150,000 tokens input

Total input tokens across run: ~800,000  (ek task ke liye!)
```

Ye **quadratic-ish** growth hai. Ek agent run ka bill 50x single call se ho sakta hai.

### Latency

```
Single call:  2 sec
Workflow:     6 sec
Agent:        30-300 sec   (reasoning model ke saath 10 min bhi normal hai)
```

Ye ek **UX design constraint** hai, sirf performance number nahi. Interactive UI mein 5-minute wait acceptable nahi — tumhe async/background job pattern chahiye, progress updates chahiye.

### Non-determinism

```
Same input, 5 runs:
Run 1: 6 steps  → sahi answer
Run 2: 12 steps → sahi answer (alag raste se)
Run 3: 4 steps  → adhoora answer
Run 4: 20 steps → max_steps hit, fail
Run 5: 8 steps  → sahi answer
```

**Iska matlab:**
- Traditional unit tests kaam nahi karte
- Debugging bahut mushkil hai ("us waqt aisa kyun kiya?")
- Statistical evals chahiye (100 runs, success rate measure karo)
- Ek "fix" ek path pe kaam karta hai, doosre pe nahi

### Failure modes jo sirf agents mein hote hain

| Failure | Kaisa dikhta hai |
|---|---|
| **Infinite loop** | Same tool bar-bar same args ke saath |
| **Context rot** | 20 steps ke baad original task bhool gaya |
| **Wrong path commitment** | Galat approach pakad li, chhodta nahi |
| **Premature completion** | "Done!" bola par kaam adhoora |
| **Silent failure** | Tool error ignore kiya, aage badh gaya |
| **Hallucinated progress** | "Main ne 5 files update kar di" — nahi ki thi |
| **Scope creep** | Ek cheez maangi, 10 cheezein badal di |
| **Cost explosion** | Ek run mein $50 kharch |

> 🚩 **"Hallucinated progress" sabse khatarnak hai** kyunki wo silently fail hota hai. Agent confidently bolega ki kaam ho gaya. Isliye modern prompting mein ek rule hota hai: *"Progress report karne se pehle, har claim ko is session ke tool result se verify karo."*

---

## Part 5: Sabse common galti

> **Agent banana jab workflow kaafi tha.**

### Ye smell test karo

```
❓ "Kya main is task ke steps likh sakta hoon?"

HAAN → Workflow banao. Sasta, fast, deterministic, debuggable.
NAHI → Agent consider karo.
```

### Concrete example

**❌ Over-engineered agent:**
```python
agent(
    task="Customer ka email padho, sentiment analyze karo, "
         "CRM se history nikalo, priority decide karo, ticket banao",
    tools=[read_email, analyze_sentiment, get_crm_history,
           create_ticket, notify_team]
)
# 8-15 LLM calls, 40 sec, $0.30, non-deterministic
# Aur kabhi-kabhi ye steps skip kar deta hai, ya galat order mein karta hai
```

**✅ Sahi workflow:**
```python
def handle_email(email):
    sentiment = llm_classify(email)                    # 1 call
    history   = crm.get_history(email.sender)          # code — deterministic!
    priority  = compute_priority(sentiment, history)   # code — auditable rules!
    ticket    = crm.create_ticket(email, priority)     # code
    if priority == "HIGH":
        notify_team(ticket)                            # code
    return ticket
# 1 LLM call, 3 sec, $0.01, deterministic, testable
```

**Comparison:** 30x sasta, 13x fast, poori tarah testable, aur **priority logic auditable** hai (business rules code mein hain, model ke mood mein nahi).

> 🔑 **Rule:** Jitna kaam **code** kar sakta hai, wo code se karao. LLM ko sirf wahan use karo jahan **judgement** chahiye.
>
> LLM ek expensive, unreliable, non-deterministic function call hai. Use minimize karo, eliminate nahi.

---

## Part 6: Agents kahan **actually** kaam kar rahe hain (2026)

Imaandaar picture — jahan agents production mein genuinely value de rahe hain:

| Domain | Kyun kaam karta hai |
|---|---|
| **Coding agents** ⭐ | **Verification free hai** — tests, compiler, linter. Agent apna kaam khud check kar sakta hai. |
| **Deep research** | Task genuinely open-ended, aur output (report) human review hota hai |
| **Data analysis** | Code execution se verify hota hai, iterative exploration natural hai |
| **Customer support (scoped)** | Narrow tool set, escalation path, human fallback |
| **Browser automation** | Steps genuinely unpredictable (UI badalta hai) |
| **DevOps triage** | Log exploration open-ended, par actions gated hain |

### 🔑 Pattern dekha?

> **Agents wahan kaam karte hain jahan VERIFICATION sasti hai.**

Coding agents best-performing agents hain kyunki:
```
Agent code likhta hai → tests chalata hai → fail → khud fix karta hai → pass ✅
                          ↑
                    ye feedback loop FREE hai
```

Jahan verification mehngi ya impossible hai (creative writing, strategic advice, subjective judgement), wahan agents weakly perform karte hain — aur unka non-determinism liability ban jaata hai.

> **Practical takeaway:** Agent design karte waqt sabse pehle poocho: **"Agent apna kaam khud verify kaise karega?"** Agar iska answer nahi hai, to agent probably galat tool hai.

---

## Part 7: Anthropomorphism trap

Language se dhokha na khao:

| Log kehte hain | Actually hota hai |
|---|---|
| "Agent **decide** karta hai" | Model ek probability distribution se token sample karta hai |
| "Agent **plan** banata hai" | Model plan-jaisa text generate karta hai |
| "Agent **yaad** rakhta hai" | Tum poori history dobara bhejte ho |
| "Agent **seekhta** hai" | Kuch nahi seekhta — weights same rehte hain |
| "Agent **samajhta** hai" | Pattern matching (bahut sophisticated, par pattern matching) |
| "Agent **koshish** kar raha hai" | Loop chal raha hai |

**Ye pedantic nahi hai** — ye practically matter karta hai:

```
❌ Anthropomorphic soch:
   "Agent samajh gaya hai ki kya karna hai, ab wo khud kar lega"
   → Tum monitoring, validation, limits skip kar dete ho

✅ Mechanical soch:
   "Model ne ek tool call generate ki hai. Main use validate karunga,
    allowlist check karunga, execute karunga, result wapas doonga.
    Loop pe cap hai. Cost pe cap hai. Destructive ops gated hain."
   → Tum reliable system banate ho
```

---

## Khud se pooch 🤔

1. Agent ki ek-line definition do. Workflow se **exact** farak kya hai?
2. Agent banane se pehle 4 sawaal kaunse poochne chahiye?
3. Agent mein context **quadratic-ish** kyun badhta hai? Iska cost pe kya asar?
4. Agents ke 8 failure modes batao. Kaunsa sabse khatarnak hai aur kyun?
5. Coding agents sabse successful category kyun hai? (Ek word mein pattern batao)
6. "Agent decide karta hai" ke bajaye mechanically kya hota hai? Ye distinction practically kyun matter karta hai?
7. In tasks ke liye single call / workflow / agent — kaunsa, aur kyun?
   - (a) "1000 resumes se skills extract karo"
   - (b) "Ye failing test debug karo aur fix karo"
   - (c) "Har raat sales report banao aur email karo"
   - (d) "User ki query ka jawab do hamare docs se"
   - (e) "Hamara AWS bill audit karo aur cost cutting suggest karo"

<details>
<summary>Answer 7</summary>

**(a) 1000 resumes se skills extract karo → SINGLE CALL (per resume) + Batch API**
Steps: ek. Judgement: extraction. Koi tool nahi chahiye.
→ Structured output ke saath 1 call per resume, `temperature=0`, **Batch API** (~50% sasta, real-time zaroorat nahi). Chhota fast model kaafi hai.
Agent banana yahan absurd hoga — 1000x cost, zero benefit.

**(b) Failing test debug karo aur fix karo → AGENT ✅**
- Complexity: ✅ steps pata nahi (kaunsi file? kya root cause?)
- Value: ✅ developer time bachati hai
- Viability: ✅ models isme strong hain
- Error cost: ✅ **tests verify karte hain**, git revert possible
→ Classic agent use case. Aur verification free hai (test suite) — best possible setup.

**(c) Nightly sales report → WORKFLOW (ya mostly code!)**
Steps completely known hain: query DB → format → LLM se narrative → email.
→ Cron job + 1 LLM call (narrative ke liye) + code.
Agent banana = non-determinism daalna ek predictable task mein. Har raat alag report format? Nahi chahiye.

**(d) Docs se query answer → CLASSIC RAG (single call)**
Ek retrieval + ek generation. Ye Day 3 ka bread-and-butter case hai.
→ Agentic RAG **sirf** un queries pe jo multi-hop hain — routing se decide karo (90% classic, 10% agentic).

**(e) AWS bill audit + cost cutting → AGENT ✅ (par read-only!)**
- Complexity: ✅ exploration genuinely open-ended (kaunse services? kaunse regions? kya pattern?)
- Value: ✅ potentially bade savings
- Viability: ✅ data structured hai, analysis model kar sakta hai
- Error cost: ⚠️ **agar agent resources modify kare to disaster**

→ **Agent banao, par read-only tools ke saath.** Output ek *recommendation report* ho, action nahi. Human review kare, phir execute kare.
Ye "agent proposes, human disposes" pattern high-stakes domains mein ka standard hai — aur ye exactly wo case hai jahan 4th question (error cost) ne design badal diya.
</details>

---

**Next:** [`02-agent-loop-aur-harness.md`](02-agent-loop-aur-harness.md) — poore repo ka core file.
