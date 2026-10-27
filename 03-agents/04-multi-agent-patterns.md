# 04 — Multi-Agent Patterns (hype vs reality)

> **Is file ka goal:** Multi-agent ko honestly assess karna. Ye 2024-25 ka sabse over-hyped topic tha — par 2026 mein iska ek **genuine, narrow** justification hai.
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/06-multi-agent-patterns`](../08-diagrams/06-multi-agent-patterns.excalidraw) — 4 patterns side-by-side.

---

## Part 1: Asli justification — context isolation

Log kehte hain multi-agent ka fayda "specialization" hai. **Ye mostly galat hai.**

```
❌ Weak justification: "Ek agent coding mein expert, ek writing mein"
   → Same model hai! "Specialization" sirf ek alag system prompt hai.
   → Ek hi agent ko dono instructions de do — same result, kam complexity.

✅ Strong justification: CONTEXT ISOLATION
   → Har agent ka apna context window hai.
   → Sub-agent 100K tokens padhta hai, 500 tokens ka summary return karta hai.
   → Main agent ka context clean rehta hai.
```

### Ye concrete example dekho

```
❌ SINGLE AGENT:
   Task: "In 50 files ko analyze karke architecture report banao"

   Step 1:  read file1.py    → 3K tokens context mein
   Step 2:  read file2.py    → 6K tokens
   ...
   Step 50: read file50.py   → 150K tokens 💥
   Step 51: report likho     → model context mein kho gaya, quality kharab

✅ MULTI-AGENT:
   Main agent → 5 sub-agents ko 10-10 files do
       Sub-agent 1: 30K tokens padha → 400 token summary return
       Sub-agent 2: 30K tokens padha → 400 token summary
       ... (parallel!)
   Main agent ko: 2,000 tokens milte hain (150K nahi)
   Main agent: clean context mein acha report likhta hai ✅
```

> 🔑 **Multi-agent ka asli fayda: "150K tokens padho, 2K tokens wapas lao."**
>
> Aur bonus: parallelism (5 sub-agents ek saath chal sakte hain → 5x fast).

---

## Part 2: Patterns

### Pattern 1: Orchestrator + Workers (sabse useful) ⭐

```
                 ORCHESTRATOR
                (plan + synthesize)
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    Worker 1       Worker 2       Worker 3
   (files 1-17)   (files 18-34)  (files 35-50)
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                 ORCHESTRATOR
                 (final report)
```

```python
def orchestrate(task):
    # 1. Plan — kaam ko independent chunks mein todo
    subtasks = llm_plan(f"""Is task ko INDEPENDENT subtasks mein todo jo
parallel chal sakte hain. Har subtask self-contained ho — worker ko
main conversation nahi dikhegi.

Task: {task}""")

    # 2. Fan out (PARALLEL — ye latency ka bada win hai)
    with ThreadPoolExecutor(max_workers=5) as ex:
        results = list(ex.map(lambda st: run_worker(st), subtasks))

    # 3. Synthesize
    return llm(f"Task: {task}\n\nWorker reports:\n{format(results)}\n\n"
               f"In reports se ek unified answer banao.")

def run_worker(subtask):
    """Worker ka apna FRESH context hai."""
    return Agent(tools=WORKER_TOOLS, config=WORKER_CONFIG).run(f"""
{subtask.description}

Tumhe sirf ye subtask karna hai. Poora context nahi hai — jo diya hai
usi se kaam karo. Apna finding ek CONCISE report mein do (max 300 shabd),
har claim ke saath source (file path / line number).
""")
```

**Kab use karo:** Task genuinely parallel-izable ho — bahut files, bahut sources, bahut records.

### Pattern 2: Sequential pipeline

```
Agent A (research) → Agent B (draft) → Agent C (review) → output
```

```python
def pipeline(topic):
    research = research_agent.run(f"Research karo: {topic}")
    draft    = writer_agent.run(f"Research: {research}\n\nEk article likho")
    review   = editor_agent.run(f"Draft: {draft}\n\nReview karo aur improve karo")
    return review
```

> 🚩 **Ruko!** Ye zyadatar **workflow** hai, multi-agent nahi. Steps fixed hain, tum decide kar rahe ho. Agar har stage ek simple LLM call se ho sakta hai, to ye 3 LLM calls ka workflow hai — teen "agents" ka orchestration nahi.
>
> **Agent tab hai jab har stage khud loop kare tools ke saath.**

### Pattern 3: Reviewer / Critic loop

```python
def generate_with_review(task, max_rounds=3):
    output = generator.run(task)
    for _ in range(max_rounds):
        review = reviewer.run(f"""Task: {task}
Output: {output}

Issues dhundo. Agar output acceptable hai to sirf "APPROVED" likho.
Warna specific, actionable issues list karo.""")
        if "APPROVED" in review:
            break
        output = generator.run(f"{task}\n\nPichhla attempt: {output}\n"
                               f"Reviewer feedback: {review}\n\nImprove karo.")
    return output
```

**Kyun kaam karta hai:** Reviewer ka **fresh context** hai — wo generator ke reasoning se biased nahi hai. Ye "self-critique" se better kaam karta hai.

**Kab:** High-quality output chahiye, cost matter nahi karta (2-6x calls).

> ⚠️ **2026 note:** Naye reasoning models **already** self-verify karte hain. Unke saath explicit reviewer loop kabhi **redundant** ho jaata hai — aur "verify karo" instruction over-verification cause karta hai. Pehle bina reviewer measure karo, phir decide karo.

### Pattern 4: Debate / ensemble

```python
def ensemble(question, n=3):
    answers = [agent.run(f"{question}\n\nApproach angle {i} se socho: {ANGLES[i]}")
               for i in range(n)]
    return llm(f"Question: {question}\n\n{n} independent answers:\n{answers}\n\n"
               f"Inko compare karo aur best synthesis do. Jahan wo disagree "
               f"karte hain, wahan explicitly batao.")
```

**Kab:** High-stakes decisions, jahan ek galat answer mehnga ho. Cost N×+1.

### Pattern 5: Router (technically multi-agent nahi, par useful)

```python
def route(query):
    intent = classifier.run(query)          # chhota fast model
    return {
        "coding":   coding_agent,
        "research": research_agent,
        "data":     data_agent,
    }[intent].run(query)
```

**Fayda:** Har agent ke paas sirf **uske** tools hain — chhota context, better tool selection, sasta.

> **Ye sabse practical "multi-agent" pattern hai** aur sabse kam glamorous. Ek chhota classifier + specialized agents = clean architecture.

---

## Part 3: Multi-agent ki real lagat 💸

Ye section imaandaari ke liye hai:

| Cost | Detail |
|---|---|
| **Token multiplication** | Har agent apna system prompt + tools + context lekar chalta hai. 5 agents = 5x fixed overhead |
| **Context re-establishment** | Sub-agent ko brief karna padta hai (usne main conversation nahi dekhi) |
| **Coordination overhead** | Orchestrator plan banata hai, results padhta hai, synthesize karta hai — extra calls |
| **Error compounding** | Agent A ki 10% error rate + Agent B ki 10% = 19% combined |
| **Latency** | Sequential mein add hoti hai; parallel mein slowest determine karta hai |
| **Debugging hell** | "Kaunse agent ne galti ki?" — 5 traces padhna padta hai |

### Concrete numbers

```
Single agent:      15 steps,  80K tokens,  $0.40,  60 sec
Multi-agent (5):   35 steps, 250K tokens,  $1.30, 45 sec (parallel)
                                                   ↑
                              3x cost, thoda fast
```

**Kab ye trade worth hai:** Jab single agent ka context genuinely blow ho raha ho aur quality gir rahi ho. **Warna nahi.**

---

## Part 4: 🔑 Decision framework

```
Kya single agent ka context BHAR raha hai?
   │
   ├─ NAHI → ❌ Multi-agent na banao. Complexity bekaar hai.
   │
   └─ HAAN → Kya kaam INDEPENDENT chunks mein tut sakta hai?
              │
              ├─ NAHI → ❌ Multi-agent na banao.
              │          Context management (prune/compact) use karo.
              │
              └─ HAAN → Kya sub-results CONCISE ho sakte hain?
                         │
                         ├─ NAHI → ❌ Fayda nahi (context phir bhar jaayega)
                         │
                         └─ HAAN → ✅ Orchestrator + Workers banao
```

### Multi-agent se PEHLE ye try karo

| Try karo | Kyun |
|---|---|
| Better tool design (bounded output) | Context bharne ka #1 karan |
| Context pruning + compaction | Sasta, simple |
| Task ko chhote agent runs mein todo | Sequential, ek-ek karke |
| Retrieval improve karo | Kam chunks, better chunks |
| `max_steps` tighten karo | Wandering rokega |

> **90% cases mein inme se koi ek kaam kar jaata hai.** Multi-agent last resort hai, first instinct nahi.

---

## Part 5: Agar multi-agent bana rahe ho — 6 rules

### Rule 1: Sub-agent ko poora brief do
Sub-agent ne main conversation **nahi dekhi**. Uska task self-contained hona chahiye:

```python
❌ "Analyze this file"                    # kaunsi file? kis liye? kya dhundhna hai?
✅ """Ye file analyze karo: src/auth/session.py

Context: Hum ek security audit kar rahe hai. Ye Django app hai.
Dhundho: authentication bypasses, session fixation, insecure defaults.
Report format: har finding ke liye — file:line, issue, severity, suggested fix.
Max 300 shabd. Kuch na mile to "no findings" likho."""
```

### Rule 2: Return format constrain karo
```python
"Apna finding ek 300-shabd ke report mein do. Raw file contents,
full logs, ya intermediate exploration return NA karo — sirf conclusions
aur unke evidence (file:line)."
```
Warna sub-agent 20K tokens return karega aur poora fayda khatam.

### Rule 3: Sub-agents ko delegate karne ki permission na do
```python
WORKER_CONFIG = AgentConfig(can_spawn_subagents=False, max_steps=15)
```
Nested delegation = exponential cost aur debugging nightmare. **Ek level kaafi hai.**

### Rule 4: Har sub-agent pe apna budget
```python
WORKER_CONFIG = AgentConfig(max_steps=15, max_cost_usd=0.30, timeout_seconds=120)
```

### Rule 5: Partial failure handle karo
```python
results = [r for r in worker_results if r is not None]
if len(results) < len(subtasks) * 0.5:
    raise TooManyWorkerFailures()
# Aur orchestrator ko batao ki kaunse subtasks fail hue
```

### Rule 6: Har thread ko alag trace karo
```python
tracer.span("worker", worker_id=i, subtask=st.description, parent_run=run_id)
```
Warna debug karna impossible hai.

---

## Part 6: Sub-agent tool descriptions

Agar tum orchestrator ko "spawn worker" tool de rahe ho, uska description **critical** hai — kyunki modern models delegation ke baare mein **ulta behave** karte hain model-to-model:

```python
# Kuch models UNDER-delegate karte hain (unhe encourage karna padta hai)
# Kuch models OVER-delegate karte hain (unhe rokna padta hai)
```

**Over-delegation rokne ke liye (ye zyada common problem hai 2026 mein):**

```
## Sub-agents ka use
Sub-agent spawn karna mehnga hai: wo apna context banata hai, dobara
explore karta hai, report deta hai, aur phir tum wo report padhte ho.
Sirf tab delegate karo jab payoff is overhead se clearly zyada ho.

DELEGATE KARO:
- Genuinely independent, parallel kaam (alag modules, wide multi-file search)

DELEGATE NA KARO:
- Jo tum khud 2-4 tool calls mein kar sakte ho
- Verification ya double-checking ke liye — wo tumhare apne loop mein karo
- Ek chhote task ko kai hisson mein todne ke liye

Agar ek sub-agent se kaam ho sakta hai, ek hi use karo. Jab delegate karo,
to pehli baar mein POORA brief do — launch karke phir dobara brief karna
mehnga hai. Aur delegate karne ke baad uska kaam dobara khud na karo.
```

---

## Part 7: Frameworks (Day 5 ka preview)

| Framework | Approach | Note |
|---|---|---|
| **LangGraph** | Graph of nodes, explicit state | Sabse control, sabse used |
| **CrewAI** | "Crew" of role-based agents | Fast prototyping, opinionated |
| **AutoGen / AG2** | Conversational multi-agent | Research-flavoured |
| **OpenAI Agents SDK** | Handoffs between agents | Simple, provider-native |
| **Provider agent platforms** | Managed multi-agent (roster/coordinator) | No infra, vendor-coupled |
| **Custom** | Tum khud likho | 200 lines, poora control ⭐ |

> **Imaandaar advice:** Orchestrator + workers pattern **200 lines mein** khud likh sakte ho (upar ka code lagbhag poora hai). Framework tab lo jab tumhe uska specific feature chahiye — durable execution, visual debugging, ya team ko ek standard chahiye.
>
> Day 5 mein detail se dekhenge.

---

## Khud se pooch 🤔

1. Multi-agent ka **asli** justification kya hai? "Specialization" wala argument weak kyun hai?
2. Orchestrator+Workers pattern context ko kaise bachata hai? Ek concrete number example do.
3. Multi-agent ki 6 real costs kya hain?
4. Decision framework ke 3 sawaal kaunse hain?
5. Multi-agent se pehle 5 cheezein kya try karni chahiye?
6. Sub-agent ko poora brief kyun dena padta hai? Ek bad-vs-good example do.
7. Nested delegation (sub-agent ka sub-agent) kyun mana hai?
8. Ek team kehti hai: "Humne 8 specialized agents banaye — CodeAgent, TestAgent, DocsAgent, ReviewAgent, SecurityAgent, PerfAgent, DeployAgent, MonitorAgent." Tumhara assessment?

<details>
<summary>Answer 8</summary>

**Ye classic multi-agent over-engineering hai.** Red flags:

**1. "Specialization" ke naam pe split hua hai, context isolation ke naam pe nahi.**
CodeAgent aur TestAgent same model hain, sirf alag system prompt. Ek hi agent ko dono capabilities do — same output, 1/8 complexity. Sawaal ye poochho: *"Kya inme se koi ek agent apna context bhar raha tha?"* Agar nahi, split ka koi justification nahi.

**2. Zyadatar sequential hai — matlab ye workflow hai, multi-agent nahi.**
Code → Test → Review → Deploy ek **fixed pipeline** hai. Steps pata hain, order pata hai. Ye 4 LLM calls ka workflow hona chahiye (ya kuch steps to plain code — deploy aur monitor mein LLM ka kya kaam?).

**3. Error compounding.**
Har agent 90% reliable ho to 8-agent chain: 0.9^8 = **43% end-to-end success rate.** Ye production mein unusable hai.

**4. Cost.**
8 agents = 8x system prompts + 8x tool definitions + 8x context re-establishment + orchestration calls. Likely 10-15x single agent ka cost.

**5. Debugging.**
Ek failure = 8 traces padhne padenge. Aur non-determinism ki wajah se reproduce bhi nahi hoga.

**Mera recommendation:**

```
1. Sab kuch collapse karo ek WORKFLOW mein:
   - Code likhna: 1 agent (tools: read, write, edit, bash, run_tests)
     → ye ek genuine agent hai, kyunki verification free hai (tests)
   - Review: 1 LLM call (fresh context — ye genuinely valuable hai)
   - Deploy + Monitor: PLAIN CODE. Yahan LLM ka koi kaam nahi.
   - Docs: 1 LLM call, ya code agent ke andar hi

2. Security aur Perf ko agent na banao — inhe TOOLS banao:
   - `run_security_scan()` → existing SAST tool chalao
   - `run_benchmark()` → existing profiler chalao
   → Deterministic, fast, reliable. LLM sirf results interpret kare.

3. Sub-agents SIRF wahan jahan context bharta ho:
   - "50 files ka security audit" → haan, workers use karo
   - "ye ek function review karo" → nahi, main agent kar lega
```

**Result:** 1 agent + 2 LLM calls + kuch deterministic tools. Shayad 85% success rate (43% ke bajaye), 1/10 cost, aur debug ho sakta hai.

**Meta-lesson:** Multi-agent architecture ka test ye nahi hai ki "kitne agents hain" — test ye hai ki **"har agent ka apna context kyun chahiye?"** Agar har agent ke liye ye sawaal answer nahi kar sakte, to wo agent nahi hona chahiye.
</details>

---

**Next:** [`05-mcp-model-context-protocol.md`](05-mcp-model-context-protocol.md)
