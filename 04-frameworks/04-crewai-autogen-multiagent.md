# 04 — CrewAI, AutoGen aur Multi-Agent Frameworks

> **Warning pehle:** Day 4 mein humne dekha ki multi-agent ka **asli justification context isolation hai**, "specialization" nahi. Ye frameworks aksar "specialization" framing bechte hain. Padho, par us lens se.

---

## Part 1: CrewAI

### Mental model

CrewAI ek **team of role-playing agents** ka metaphor use karta hai:

```python
from crewai import Agent, Task, Crew, Process

researcher = Agent(
    role="Senior Research Analyst",
    goal="AI trends pe comprehensive research karo",
    backstory="Tum 10 saal ka experience rakhne wale analyst ho...",
    tools=[search_tool, scrape_tool],
    verbose=True,
)

writer = Agent(
    role="Tech Content Writer",
    goal="Engaging technical articles likho",
    backstory="Tum ek writer ho jo complex topics ko simple banata hai...",
    tools=[],
)

research_task = Task(
    description="2026 ke top 5 AI trends research karo",
    expected_output="Ek bullet list with sources",
    agent=researcher,
)

write_task = Task(
    description="Research ke basis pe ek 800-word article likho",
    expected_output="Markdown article",
    agent=writer,
    context=[research_task],          # pichhle task ka output milega
)

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task],
            process=Process.sequential)
result = crew.kickoff()
```

### CrewAI ki strengths

| ✅ | Detail |
|---|---|
| **Intuitive** | "Role, goal, backstory" — non-engineers bhi samajh jaate hain |
| **Fast prototyping** | 30 minutes mein multi-agent demo |
| **Opinionated** | Decisions already liye hue hain — sochna kam |
| **Good defaults** | Task delegation, context passing built-in |

### CrewAI ki weaknesses

| ❌ | Detail |
|---|---|
| **"Backstory" ka value questionable** | Ye ek elaborate system prompt hai. Kya "10 saal ka experience" wala backstory actually output better karta hai? **Measure karo — aksar nahi karta.** |
| **Sequential process = workflow** | Agar steps fixed hain, ye multi-agent nahi, workflow hai. Simpler code se ho jaata |
| **Cost opacity** | Kitne LLM calls hue? Aasani se nahi dikhta |
| **Debugging** | Agents ke beech kya pass hua, trace karna mushkil |
| **Over-abstraction** | Simple task bhi 4 concepts (Agent, Task, Crew, Process) demand karta hai |

> 🚩 **Sabse honest critique:** CrewAI ka `Process.sequential` **ek workflow hai jo multi-agent ka bhes pehne hue hai.** Do agents jinke beech ek fixed handoff hai = do LLM calls. Usse framework ki zaroorat nahi.
>
> Uska `Process.hierarchical` (manager agent delegate karta hai) genuinely agentic hai — par wahan tum Day 4 wale saare multi-agent costs pay karte ho.

---

## Part 2: AutoGen / AG2

### Mental model

**Conversation-driven** multi-agent — agents aapas mein "baat" karte hain:

```python
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager

assistant = AssistantAgent("assistant", llm_config={"model": "..."})
coder     = AssistantAgent("coder", llm_config={"model": "..."},
                           system_message="Tum code likhte ho")
critic    = AssistantAgent("critic", llm_config={"model": "..."},
                           system_message="Tum code review karte ho")

user_proxy = UserProxyAgent("user_proxy",
                            code_execution_config={"work_dir": "coding"},
                            human_input_mode="NEVER")

groupchat = GroupChat(agents=[user_proxy, coder, critic], messages=[], max_round=12)
manager   = GroupChatManager(groupchat=groupchat)

user_proxy.initiate_chat(manager, message="Ek web scraper likho")
```

### AutoGen ki strengths

| ✅ | Detail |
|---|---|
| **Code execution built-in** | `UserProxyAgent` code chala sakta hai — feedback loop free |
| **Flexible conversation patterns** | Two-agent, group chat, nested chats |
| **Research heritage** | Microsoft Research se — experimentation ke liye acha |
| **Human-in-loop modes** | `ALWAYS`, `TERMINATE`, `NEVER` |

### AutoGen ki weaknesses

| ❌ | Detail |
|---|---|
| **Conversation = token explosion** | Group chat mein har agent poori conversation dekhta hai. 5 agents × 12 rounds = massive context |
| **Unpredictable termination** | Kab rukega? `max_round` pe hard stop, ya agent "TERMINATE" bole |
| **Production readiness** | Research-flavoured. Guards, budgets, observability tumhe add karne padenge |
| **Cost** | Sabse mehnga pattern — group chat mein har turn pe sab agents ka context grow |

> 🚩 **Group chat ka fundamental problem:** Ye **context isolation ka ulta** hai. Day 4 mein humne kaha tha ki multi-agent ka fayda alag contexts hain. Group chat mein sab ek shared conversation dekhte hain — matlab **context isolation ka zero fayda, aur multi-agent ki poori cost.**
>
> Ye pattern research demos mein impressive lagta hai, production economics mein bura hai.

---

## Part 3: Doosre options

| Framework | Approach | Note |
|---|---|---|
| **OpenAI Agents SDK** | Handoffs — ek agent doosre ko task pass karta hai | Simple, provider-native, guardrails built-in |
| **Semantic Kernel** | Microsoft, plugin-based, enterprise-flavoured | .NET/C# strong, Python bhi |
| **Smolagents** | HuggingFace, code-writing agents | Agent Python code likhta hai actions ke liye — interesting approach |
| **Pydantic AI** | Type-safe agents, Pydantic validation | Python-idiomatic, strong typing |
| **Atomic Agents** | Minimal, composable | Anti-bloat philosophy |
| **LangGraph** | Graph-based (file 01) | Sabse control, multi-agent bhi kar sakta hai |

---

## Part 4: 🔑 Imaandaar assessment

### Ye frameworks kya achha karte hain

1. **Demo banana** — 30 minutes mein impressive prototype ✅
2. **Concept sikhana** — multi-agent patterns samajhna aasan ho jaata hai ✅
3. **Fast iteration** — architecture explore karna ✅

### Ye kya nahi karte

1. **Cost control** — token multiplication ko manage nahi karte ❌
2. **Production guards** — budget, timeout, loop detection tumhe add karne padenge ❌
3. **Observability** — kaunsa agent kya kar raha hai, tracing basic hai ❌
4. **Accuracy** — multi-agent hone se answer better nahi hota ❌

### Ek uncomfortable sach

```
Production mein successful multi-agent systems ka bada hissa
FRAMEWORKS SE NAHI, custom orchestration se bana hai.

Kyun? Kyunki production multi-agent mein 80% kaam ye hai:
  - Har worker ka budget aur timeout
  - Partial failure handling
  - Result size limits
  - Per-thread tracing
  - Context isolation ka enforcement

Ye sab framework ka "role/goal/backstory" abstraction address nahi karta.
```

Day 4 ka orchestrator+workers code (~80 lines) **inme se kai cheezein already kar leta hai** — kyunki wo problem ke around design kiya gaya hai, metaphor ke around nahi.

---

## Part 5: Framework choose karne ka test

Multi-agent framework consider kar rahe ho? Ye poocho:

```
1. Kya mujhe multi-agent CHAHIYE HI? (Day 4 ka decision framework)
   → 80% cases mein jawab NAHI hai. Yahin ruk jao.

2. Kya mera pattern orchestrator + parallel workers hai?
   → HAAN → ~80 lines mein khud likho. Framework ka fayda kam hai.

3. Kya mujhe conversational multi-agent chahiye (agents debate karein)?
   → HAAN → AutoGen. Par cost dekh lo pehle.

4. Kya mujhe fast demo chahiye stakeholders ke liye?
   → HAAN → CrewAI. Sabse fast.

5. Kya mujhe complex conditional flow + durable state chahiye?
   → HAAN → LangGraph.

6. Kya main production ship kar raha hoon?
   → Framework ke upar guards, budgets, tracing ADD karne padenge.
     Wo effort budget mein rakho.
```

---

## Part 6: Practical recommendation

### Learning ke liye
**CrewAI se ek demo banao** (2 ghante). Multi-agent ka feel aa jaayega. Phir **wahi cheez custom likho** (3 ghante). Farak dekh ke tumhe pata chal jaayega ki framework kya de raha hai.

### Production ke liye

```
Pattern: Orchestrator + parallel workers
  → Custom (~80 lines). Day 4 ka code lagbhag ready hai.

Pattern: Complex conditional multi-step + durable state
  → LangGraph

Pattern: Agents ko genuinely conversation chahiye (rare)
  → AutoGen, par heavy guards ke saath

Pattern: Simple handoff (agent A → agent B)
  → Ye workflow hai. 2 LLM calls likho. Framework nahi chahiye.
```

### 🔑 Ek line ka advice

> **Multi-agent framework tab lo jab tumne pehle custom likh ke dekh liya ho aur tumhe pata ho ki tum kya miss kar rahe ho.**
>
> Framework se shuru karne se tum uske metaphor mein soch ne lagoge — aur "role/goal/backstory" tumhe wo sawaal nahi poochne dega jo actually matter karta hai: *"is agent ka apna context kyun chahiye?"*

---

## Part 7: Ek concrete comparison

Same task, teen tarike:

### Task: "5 competitor websites analyze karke comparison report banao"

**CrewAI (~40 lines):**
```python
researcher = Agent(role="Web Researcher", goal="...", backstory="...", tools=[scrape])
analyst    = Agent(role="Market Analyst", goal="...", backstory="...")
writer     = Agent(role="Report Writer", goal="...", backstory="...")
# 3 tasks, sequential
crew.kickoff()
```
⏱️ 30 min to write | 💰 ~$1.20 | 🐌 Sequential (slow) | 🔍 Debugging: medium

**Custom orchestrator+workers (~80 lines):**
```python
# 5 workers PARALLEL — har ek ek website
with ThreadPoolExecutor(5) as ex:
    findings = list(ex.map(analyze_site, sites))     # har ek 300-word report
# 1 synthesis call
report = llm(f"In 5 reports se comparison banao:\n{findings}")
```
⏱️ 60 min to write | 💰 ~$0.50 | ⚡ Parallel (5x fast) | 🔍 Debugging: easy

**Simple workflow (~25 lines):**
```python
data = [scrape(s) for s in sites]           # code, no LLM
summaries = [llm_summarize(d) for d in data]  # 5 calls, parallel
report = llm(f"Comparison banao:\n{summaries}")
```
⏱️ 20 min | 💰 ~$0.30 | ⚡ Fast | 🔍 Trivial debugging

**Winner:** Teesra! Kyunki **scraping ko agent ki zaroorat nahi thi** — wo deterministic code hai. Sirf summarization aur synthesis mein judgement chahiye.

> **Ye Day 4 ka lesson dohra raha hai:** Jo code kar sakta hai, code se karao. Agent sirf judgement ke liye.

---

## Khud se pooch 🤔

1. CrewAI ka `Process.sequential` "workflow in disguise" kyun hai?
2. "Backstory" ka value questionable kyun hai? Kaise verify karoge?
3. AutoGen ka group chat pattern **context isolation ka ulta** kyun hai?
4. Ye frameworks kya 4 cheezein achhi karte hain, aur kya 4 nahi karte?
5. Production multi-agent mein 80% kaam kya hai? Framework wo address karta hai?
6. Framework choose karne ke 6 sawaal kya hain? Pehla sawaal sabse important kyun hai?
7. Part 7 ke comparison mein teesra option kyun jeeta?
8. Tumhara manager kehta hai: "CrewAI se ek 6-agent system banao jo hamara customer support automate kare." Tumhara response?

<details>
<summary>Answer 8</summary>

**Main sawaal poochunga, framework se shuru nahi karunga:**

**"Customer support ke andar exactly kaunse tasks hain, aur unme se kaunse LLM judgement demand karte hain?"**

Typical breakdown:
```
1. Ticket classify karo           → 1 LLM call (single call)
2. Customer history nikalo         → CODE (CRM API)
3. Similar past tickets dhundo     → RAG
4. Draft response likho            → 1 LLM call
5. Policy compliance check          → CODE (rules) + 1 LLM call
6. Route/escalate                  → CODE (rules)
7. Ticket update karo              → CODE (API)
```

**Isme se 4 steps CODE hain, 3 steps LLM calls hain, aur ZERO steps ko agent ki zaroorat hai.**

**Mera proposal:**

```python
def handle_ticket(ticket):
    category   = llm_classify(ticket)                    # 1 call
    history    = crm.get_history(ticket.customer_id)     # code
    similar    = rag_search(ticket.body, k=3)            # RAG
    draft      = llm_draft(ticket, history, similar)     # 1 call
    if not policy_check(draft):                          # code rules
        draft = llm_revise(draft, policy_violations)     # 1 call (rare)
    if needs_human(category, ticket.sentiment):          # code rules
        return escalate(ticket, draft)
    return send(draft)
```

**3 LLM calls, ~4 seconds, ~$0.02 per ticket, fully deterministic path, testable.**

CrewAI 6-agent version: ~15-25 LLM calls, 60+ seconds, ~$0.40 per ticket, non-deterministic, aur har failure debug karne mein 6 traces.

**Manager ko kaise batau:**

> "6 agents banane se ye system 20x mehnga aur 15x slow ho jaayega, bina accuracy gain ke — kyunki inme se zyadatar steps mein koi judgement nahi hai, wo API calls aur business rules hain. Main ek workflow propose karta hoon: 3 LLM calls jahan judgement chahiye, baaki deterministic code.
>
> **Agar ye chal gaya aur hum specific cases dekhein jahan fixed workflow fail ho raha hai** — jaise complex multi-issue tickets jinme investigation chahiye — **to un cases ke liye ek agent add karenge.** Tab hamare paas evidence hoga ki agent kyun chahiye."

**Aur agar manager phir bhi insist kare:**
Ek A/B chalao. 100 tickets workflow se, 100 CrewAI se. Accuracy, cost, latency compare karo. **Data se decide karo, opinion se nahi.** Ye almost hamesha workflow ke favour mein aata hai — aur ab wo tumhara opinion nahi, measurement hai.

**Meta-lesson:** "Multi-agent banao" aksar ek **solution looking for a problem** hota hai. Sahi sawaal hai: *"is task mein judgement kahan chahiye?"* — aur uska answer usually "3 jagah, 6 nahi" hota hai.
</details>

---

**Next:** [`05-vector-databases.md`](05-vector-databases.md)
