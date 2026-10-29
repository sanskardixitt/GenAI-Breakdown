# 01 — LangChain aur LangGraph

> **Note:** Ye dono ek hi team ke hain par **bahut alag** cheezein hain. Log inhe confuse karte hain. LangChain ek toolkit hai; LangGraph ek orchestration engine hai.

---

## Part 1: LangChain — kya hai

LangChain ek **broad toolkit** hai LLM apps ke liye. Uska sabse bada asset **ecosystem** hai:

```
langchain-core        → base abstractions (Runnable, messages, prompts)
langchain             → chains, agents, retrievers
langchain-community   → 700+ integrations (loaders, stores, tools)
langchain-<provider>  → per-provider packages (anthropic, openai, ...)
langchain-text-splitters → chunking utilities
```

### LCEL (LangChain Expression Language)

LangChain ka core abstraction — pipe operator se composition:

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template(
    "Context: {context}\n\nQuestion: {question}\n\nAnswer:")
model  = ChatAnthropic(model="claude-opus-5")

chain = prompt | model | StrOutputParser()

chain.invoke({"context": docs, "question": "..."})
# Free milta hai: .stream(), .batch(), .ainvoke(), retries, fallbacks
```

**Fayda:** Composition clean hai, streaming/batching/async free milte hain.
**Nuksaan:** Debugging mein pipe ke andar kya ho raha hai wo dikhta nahi. Aur `|` operator ke peeche kaafi machinery hai.

---

## Part 2: LangChain — kab use karo, kab nahi

### ✅ Kab acha hai

| Situation | Kyun |
|---|---|
| **Bahut data sources** | 700+ community loaders — Notion, Slack, Confluence, S3, Drive... Ye genuinely weeks bachate hain |
| **Prototyping speed** | Ek RAG pipeline 30 lines mein |
| **Team standard** | Sab ek vocabulary bolte hain |
| **Provider switching** | Uniform interface |

### ❌ Kab bura hai

| Situation | Kyun |
|---|---|
| **Simple use case** | 1 LLM call ke liye ek framework? Overkill. |
| **Latest features chahiye** | Abstraction naye API params ko block karti hai (`effort`, thinking config, structured outputs ke naye shapes) |
| **Deep debugging chahiye** | Layers ke through trace karna painful |
| **Performance critical** | Overhead real hai |
| **Seekhna hai** | Framework tumhe fundamentals se door rakhega |

### 🚩 Sabse common complaint

```python
# Ye simple lagta hai...
chain = prompt | model | parser
result = chain.invoke({"q": "..."})

# ...par jab ye fail hota hai:
# - Actual prompt kya bheja gaya? (dikhta nahi)
# - Kaunse parameters model tak gaye?
# - Retry hua? Kitni baar?
# - Token usage kya tha?
# → Tracing (LangSmith) ke bina ye black box hai
```

**Isliye LangChain use karo to tracing zaroor lagao.** Bina tracing ke LangChain debug karna painful hai.

---

## Part 3: LangGraph — asli value

LangGraph **alag cheez** hai. Ye ek **stateful graph execution engine** hai.

### Core idea

```
Chain (LangChain):    A → B → C           (linear, ek direction)
Graph (LangGraph):    A → B → C
                          ↓   ↑           (cycles! conditionals! state!)
                          D ──┘
```

### Basic shape

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class State(TypedDict):
    question: str
    documents: list
    answer: str
    attempts: Annotated[int, operator.add]      # reducer — accumulate

def retrieve(state: State):
    return {"documents": search(state["question"])}

def grade(state: State):
    quality = llm_grade(state["question"], state["documents"])
    return {"quality": quality}

def rewrite(state: State):
    return {"question": llm_rewrite(state["question"]), "attempts": 1}

def generate(state: State):
    return {"answer": llm_answer(state["question"], state["documents"])}

# Graph banao
g = StateGraph(State)
g.add_node("retrieve", retrieve)
g.add_node("grade", grade)
g.add_node("rewrite", rewrite)
g.add_node("generate", generate)

g.set_entry_point("retrieve")
g.add_edge("retrieve", "grade")

# CONDITIONAL edge — yahan LangGraph ka value hai
def route(state):
    if state["quality"] == "good":       return "generate"
    if state["attempts"] >= 2:            return "generate"    # give up
    return "rewrite"

g.add_conditional_edges("grade", route,
                        {"generate": "generate", "rewrite": "rewrite"})
g.add_edge("rewrite", "retrieve")        # ← CYCLE
g.add_edge("generate", END)

app = g.compile()
app.invoke({"question": "...", "attempts": 0})
```

Ye exactly **Corrective RAG** hai (Day 3, file 04) — par declarative form mein.

---

## Part 4: LangGraph ke 4 killer features

### Feature 1: Checkpointing / durable state ⭐

```python
from langgraph.checkpoint.postgres import PostgresSaver

app = g.compile(checkpointer=PostgresSaver(conn))

config = {"configurable": {"thread_id": "user-123-session-456"}}
app.invoke({"question": "..."}, config)

# Process crash ho gaya? Server restart? Koi baat nahi:
app.invoke(None, config)      # wahin se resume, saved state se
```

**Ye single feature LangGraph ka strongest justification hai.** Long-running agents crash hote hain — checkpointing ke bina poora kaam gaya.

Khud likhna? Possible, par state serialization + resume logic non-trivial hai.

### Feature 2: Human-in-the-loop (interrupts)

```python
app = g.compile(checkpointer=saver, interrupt_before=["execute_payment"])

# Graph `execute_payment` se pehle RUK jaayega
state = app.invoke(input, config)

# Human review kare, state edit kare bhi kar sakta hai
app.update_state(config, {"amount": corrected_amount})

# Resume
app.invoke(None, config)
```

Approval workflows ke liye ye clean hai.

### Feature 3: Streaming with granularity

```python
for chunk in app.stream(input, config, stream_mode="updates"):
    print(chunk)     # har node ka output as it happens
# stream_mode: "values" | "updates" | "messages" | "debug"
```

User ko dikha sakte ho: "Searching... Grading... Generating..." — ye UX ke liye acha hai (agent latency ko manage karta hai).

### Feature 4: Time travel / branching

```python
history = list(app.get_state_history(config))
# Kisi bhi past state pe wapas jao aur alag path try karo
app.invoke(None, {"configurable": {"thread_id": "...",
                                   "checkpoint_id": history[3].config["checkpoint_id"]}})
```

Debugging aur "what-if" ke liye powerful.

---

## Part 5: LangGraph vs custom loop — imaandaar comparison

| Aspect | Custom loop | LangGraph |
|---|---|---|
| **Lines of code** | ~200 | ~100 (+ framework) |
| **Learning curve** | Zero | Medium (state, reducers, edges) |
| **Debuggability** | Full — sab tumhara code | Medium — graph execution abstract hai |
| **Checkpointing** | Tumhe likhna padega (non-trivial) | ✅ Built-in |
| **Human-in-loop** | Tumhe likhna padega | ✅ Built-in |
| **Streaming granularity** | Tumhe likhna padega | ✅ Built-in |
| **Latest API features** | ✅ Turant | ⚠️ Wrapper update ka intezaar |
| **Team onboarding** | Codebase padho | Framework docs padho |
| **Complex branching** | If-else spaghetti ban sakta hai | ✅ Declarative, visualizable |
| **Version risk** | Zero | Breaking changes possible |

### 🔑 Decision rule

```
Kya tumhe in me se KOI EK chahiye?
  - Crash-resume (durable execution)
  - Human approval mid-flow with state editing
  - Complex conditional branching (5+ paths)
  - Per-node streaming granularity
  - Multiple team members ek shared flow pe kaam kar rahe hain

HAAN → LangGraph. Value real hai.
NAHI → Custom loop. Simpler, faster, tumhara.
```

> **Meri honest position:** LangGraph un few frameworks mein se hai jinka value **genuinely** replicate karna mushkil hai. Checkpointing + interrupts khud likhna 2-3 hafte ka kaam hai, aur bugs aayenge.
>
> Par agar tumhe wo features nahi chahiye — to LangGraph sirf ek abstraction layer hai jo tumhe API se door karta hai.

---

## Part 6: LangSmith (observability)

LangChain/LangGraph ka tracing platform:

```python
import os
os.environ["LANGSMITH_TRACING"] = "true"
os.environ["LANGSMITH_API_KEY"] = "..."
# Bas. Har chain/graph run automatically trace ho jaayega.
```

**Deta hai:** Nested traces, token/cost tracking, latency breakdown, datasets, evaluations, prompt playground, A/B comparison.

**Note:** LangSmith **framework-agnostic** bhi kaam karta hai (OpenTelemetry / `@traceable` decorator se) — sirf LangChain ke liye nahi.

**Alternatives:** Langfuse (open-source, self-hostable), Phoenix (open-source, OTel-native), Braintrust (evals-first).

---

## Part 7: Migration reality check

Log poochte hain: "LangChain se hatna chahiye?"

```
Agar tum LangChain use kar rahe ho aur:

✅ Kaam kar raha hai, team samajhti hai, accuracy acchi hai
   → RAHNE DO. "Framework hataana" ek productive project nahi hai.

⚠️ Debug karna painful hai par baaki theek hai
   → Tracing add karo (LangSmith/Langfuse). Usually ye kaafi hai.

⚠️ Ek specific feature block ho raha hai (naya API param)
   → Us ek jagah escape hatch use karo (direct SDK call).
     Poora migrate na karo.

❌ Har cheez fight lag rahi hai, aur tum framework ka source zyada
   padh rahe ho apne code se
   → Core path ko custom likho. Framework ko edges pe rakho
     (loaders, splitters — jahan wo genuinely value deta hai).
```

> **"Framework hataana" ka business value zero hai** jab tak wo actively problem na de raha ho. Users ko farak nahi padta.

---

## Part 8: Practical patterns

### Pattern: LangChain sirf loading ke liye

Ye ek **bahut practical** hybrid hai:

```python
# LangChain ka BEST part use karo — integrations
from langchain_community.document_loaders import NotionDBLoader, SlackDirectoryLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

docs = NotionDBLoader(...).load() + SlackDirectoryLoader(...).load()
chunks = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100).split_documents(docs)

# Baaki sab KHUD — full control
embeddings = my_embed_batch([c.page_content for c in chunks])
my_vector_db.upsert(chunks, embeddings)

def my_rag(query):
    hits = my_hybrid_search(query, k=50)
    top = my_rerank(query, hits, k=5)
    return anthropic_client.messages.create(...)      # direct SDK
```

**Best of both:** 700 loaders ka fayda, par pipeline pe poora control aur latest API features.

### Pattern: LangGraph + direct SDK nodes

```python
def call_model(state):
    # LangChain wrapper NAHI — direct SDK
    r = anthropic_client.messages.create(
        model="claude-opus-5",
        thinking={"type": "adaptive"},
        output_config={"effort": "high"},     # naya feature, turant available
        messages=state["messages"],
        tools=TOOLS,
    )
    return {"messages": [{"role": "assistant", "content": r.content}]}

g.add_node("model", call_model)
```

**LangGraph ka orchestration + direct SDK ka feature access.** Ye combination bahut logon ke liye sweet spot hai.

---

## Khud se pooch 🤔

1. LangChain aur LangGraph mein **fundamental** farak kya hai?
2. LangChain ka sabse bada genuine value kya hai?
3. LangChain use karte waqt tracing kyun **mandatory** hai?
4. LangGraph ke 4 killer features kya hain? Kaunsa sabse strong justification hai?
5. LangGraph vs custom loop — decision rule kya hai (5 conditions)?
6. "LangChain se migrate karna chahiye?" — 4 scenarios aur unke answers?
7. "LangGraph + direct SDK nodes" pattern ka fayda kya hai?
8. Tumhari team ek approval workflow bana rahi hai: agent ek refund process kare, par ₹10,000 se upar human approval chahiye, aur agar system crash ho to wahin se resume ho. Kya use karoge?

<details>
<summary>Answer 8</summary>

**LangGraph — ye uska textbook use case hai.** Yahan dono killer features chahiye:

```python
class RefundState(TypedDict):
    request: dict
    amount: float
    validation: dict
    decision: str

g = StateGraph(RefundState)
g.add_node("validate", validate_request)          # rules + DB lookup (code)
g.add_node("assess", llm_assess_refund)           # LLM judgement
g.add_node("execute", execute_refund)             # payment API
g.add_node("notify", notify_customer)

g.set_entry_point("validate")
g.add_edge("validate", "assess")

def needs_approval(state):
    return "human_approval" if state["amount"] > 10_000 else "execute"

g.add_conditional_edges("assess", needs_approval,
                        {"human_approval": "execute", "execute": "execute"})
g.add_edge("execute", "notify")
g.add_edge("notify", END)

app = g.compile(
    checkpointer=PostgresSaver(conn),          # ← CRASH RESUME
    interrupt_before=["execute"],              # ← HUMAN APPROVAL
)
```

**Kyun LangGraph aur custom nahi:**

1. **Durable checkpointing** — payment workflow crash-safe hona **mandatory** hai. Agar process ₹50,000 refund ke beech crash ho jaaye, tumhe exact state chahiye. Ye khud likhna: state serialization + versioning + resume + idempotency = 2-3 hafte aur bugs.

2. **Human-in-the-loop with state editing** — `interrupt_before` + `update_state` exactly ye pattern hai. Approver amount correct bhi kar sakta hai, phir resume. Khud likhna = ek poora state machine + persistence layer.

3. **Audit trail** — checkpoint history se tumhe har state transition ka record milta hai. Financial workflows mein ye compliance requirement hai.

**Par ye bhi note karo:**

- **`validate` aur `execute` mein LLM nahi hona chahiye.** Validation business rules hai (code), execution payment API call hai (code). LLM sirf `assess` mein hai (judgement).
- **Idempotency** LangGraph nahi deta — tumhe payment API pe idempotency key khud lagani padegi. Resume ke baad double-refund ka risk real hai.
- **Alternative:** Agar tumhari company mein already **Temporal** hai, to wo bhi perfect fit hai (aur AI-agnostic hai — poora refund workflow, LLM sirf ek activity). Naya tool lene se pehle dekho ki durable workflow engine already hai ya nahi.

**Meta-lesson:** Ye wo case hai jahan framework ka value **genuinely** justify hota hai — kyunki wo feature (durable execution + interrupts) khud likhna mehnga aur bug-prone hai. Ye "framework hype" nahi, "framework fit" hai.
</details>

---

**Next:** [`02-llamaindex.md`](02-llamaindex.md)
