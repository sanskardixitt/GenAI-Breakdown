# 02 — Agent Loop aur Harness ⭐⭐ (Core file)

> **Is file ka goal:** "Harness" concept samajhna — **yahi asli agent engineering hai.** Model commodity hai; harness tumhara product hai.
>
> 📊 **Diagram saath mein kholo:** [`08-diagrams/05-agent-loop-harness`](../08-diagrams/05-agent-loop-harness.excalidraw)

---

## Part 1: Harness kya hai?

```
        ┌─────────────────────────────────────────┐
        │              HARNESS                     │
        │  (tumhara code — YAHAN VALUE HAI)        │
        │                                          │
        │  • Loop control (kab rukna hai)          │
        │  • Tool execution + validation           │
        │  • Context management (prune, compact)   │
        │  • Error handling + retries              │
        │  • Permission gates (approval)           │
        │  • Cost/step limits                      │
        │  • Observability (traces, logs)          │
        │  • State persistence                     │
        │                                          │
        │       ┌──────────────────────┐           │
        │       │       MODEL           │          │
        │       │  (commodity — API)    │          │
        │       └──────────────────────┘           │
        └─────────────────────────────────────────┘
```

> 🔑 **Sabse important insight is repo ka:**
>
> **Model saare AI products mein same hai. Harness tumhara differentiator hai.**
>
> Do teams same model use karke completely different quality product bana sakti hain — farak harness mein hota hai. Claude Code aur ek generic chatbot same model use karte hain; farak harness mein hai.

---

## Part 2: Minimal loop → production loop

### Level 0: Naive (kabhi ship na karo)

```python
def agent(task, tools):
    messages = [{"role": "user", "content": task}]
    while True:                                  # 💀 infinite loop possible
        r = llm(messages, tools=tools)
        messages.append({"role": "assistant", "content": r.content})
        if r.stop_reason != "tool_use":
            return r
        results = [execute(tc) for tc in r.tool_calls]   # 💀 no error handling
        messages.append({"role": "user", "content": results})
```

**Kya missing hai:** step limit, cost limit, error handling, context management, validation, permissions, observability, state. **Sab kuch.**

### Level 1: Production loop

```python
from dataclasses import dataclass, field

@dataclass
class AgentConfig:
    max_steps: int = 30
    max_cost_usd: float = 2.0
    max_context_tokens: int = 150_000
    timeout_seconds: int = 600
    require_approval_for: set = field(default_factory=lambda: {
        "delete_file", "send_email", "make_payment", "deploy", "run_migration"
    })

class Agent:
    def __init__(self, tools, config, tracer):
        self.tools, self.config, self.tracer = tools, config, tracer

    def run(self, task):
        messages = [{"role": "user", "content": task}]
        state = RunState(task=task, start=now())

        for step in range(self.config.max_steps):
            # ─── GUARDS (loop se pehle) ───
            self._check_budget(state)
            self._check_timeout(state)
            messages = self._manage_context(messages, state)

            # ─── MODEL CALL ───
            with self.tracer.span("model_call", step=step):
                r = self._call_with_retry(messages)
            state.cost += estimate_cost(r.usage)
            state.tokens += r.usage.output_tokens

            messages.append({"role": "assistant", "content": r.content})

            # ─── STOP REASON HANDLING ───
            if r.stop_reason == "max_tokens":
                raise TruncatedOutput(step=step)
            if r.stop_reason == "refusal":
                return AgentResult(status="refused", detail=r.stop_details)
            if r.stop_reason == "pause_turn":
                continue                          # server-side tool resume
            if r.stop_reason != "tool_use":
                return AgentResult(status="complete", output=r,
                                   steps=step + 1, cost=state.cost)

            # ─── TOOL EXECUTION ───
            tool_calls = [b for b in r.content if b.type == "tool_use"]
            if self._is_looping(tool_calls, state):
                messages.append(self._loop_warning(tool_calls))
                continue

            results = self._execute_all(tool_calls, state)
            messages.append({"role": "user", "content": results})

        return AgentResult(status="max_steps_exceeded", steps=self.config.max_steps,
                           cost=state.cost, partial=messages)
```

Ab ise piece by piece kholte hain.

---

## Part 3: Harness ke 8 components

### Component 1: Loop control

```python
def _check_budget(self, state):
    if state.cost > self.config.max_cost_usd:
        raise BudgetExceeded(f"${state.cost:.2f} > ${self.config.max_cost_usd}")

def _check_timeout(self, state):
    if (now() - state.start).seconds > self.config.timeout_seconds:
        raise Timeout()

def _is_looping(self, tool_calls, state, window=4):
    """Same tool + same args repeat ho raha hai?"""
    sigs = [(tc.name, json.dumps(tc.input, sort_keys=True)) for tc in tool_calls]
    state.recent_calls.extend(sigs)
    for sig in sigs:
        if state.recent_calls[-window*2:].count(sig) >= 3:
            return True
    return False

def _loop_warning(self, tool_calls):
    return {"role": "user", "content":
        f"Tum '{tool_calls[0].name}' ko same arguments ke saath repeat kar rahe ho. "
        f"Ye kaam nahi kar raha. Ek ALAG approach lo, ya batao ki tum kyun "
        f"stuck ho aur kya information chahiye."}
```

> **Loop detection sabse valuable single guard hai.** Aur model ko *batana* ki wo loop kar raha hai — silently break karne se better hai. Wo actually recover kar leta hai.

### Component 2: Tool execution (safe)

```python
def _execute_all(self, tool_calls, state):
    # Parallel — ye latency ka bada win hai
    with ThreadPoolExecutor(max_workers=5) as ex:
        futures = {ex.submit(self._execute_one, tc, state): tc for tc in tool_calls}
        results = []
        for f in as_completed(futures):
            results.append(f.result())
    # Order preserve karo (tool_use order mein)
    return sorted(results, key=lambda r: [tc.id for tc in tool_calls].index(r["tool_use_id"]))

def _execute_one(self, tc, state):
    def result(content, is_error=False):
        return {"type": "tool_result", "tool_use_id": tc.id,
                "content": str(content)[:MAX_TOOL_OUTPUT], "is_error": is_error}

    # 1. ALLOWLIST
    if tc.name not in self.tools:
        return result(f"Error: unknown tool '{tc.name}'. "
                      f"Available: {list(self.tools)}", is_error=True)

    # 2. SCHEMA VALIDATION
    try:
        args = self.tools[tc.name].schema.model_validate(tc.input)
    except ValidationError as e:
        return result(f"Invalid arguments: {e}. Schema check karke retry karo.",
                      is_error=True)

    # 3. PERMISSION GATE
    if tc.name in self.config.require_approval_for:
        if not request_approval(tc.name, args):
            return result("User ne ye action DENY kar diya. Kuch aur try karo "
                          "ya user se poochho.", is_error=False)

    # 4. EXECUTE (timeout ke saath)
    try:
        with self.tracer.span("tool", name=tc.name):
            out = run_with_timeout(self.tools[tc.name].fn, args, timeout=60)
        return result(self._shape_output(out))
    except TimeoutError:
        return result("Error: tool 60s mein complete nahi hua. Chhota scope try karo.",
                      is_error=True)
    except Exception as e:
        return result(f"Error: {type(e).__name__}: {e}", is_error=True)
```

**5 layers:** allowlist → validation → permission → timeout → error capture. **Har layer zaroori hai.**

### Component 3: Tool output shaping 🔥

Ye **sabse under-appreciated** component hai:

```python
MAX_TOOL_OUTPUT = 4000     # tokens, roughly

def _shape_output(self, out):
    """Tool output ko context-friendly banao."""
    if isinstance(out, str) and len(out) > MAX_TOOL_OUTPUT * 4:
        return (out[:MAX_TOOL_OUTPUT * 2] +
                f"\n\n[... {len(out)} chars total, truncated. "
                f"Specific part chahiye to filter/offset ke saath dobara call karo ...]\n\n" +
                out[-MAX_TOOL_OUTPUT:])
    return out
```

**Kyun critical hai:**
```
❌ Ek badly-designed tool:
   read_file("app.log") → 500,000 tokens → poora context window ek call mein gaya

✅ Designed tool:
   read_file("app.log", filter="ERROR", tail=100) → 2,000 tokens
   + note: "1.2M lines total, 100 ERROR lines dikhaye. Filter change kar sakte ho."
```

> **Tool design harness ka hissa hai.** Ek acha harness tools ko aise design karta hai ki wo **high-signal, bounded** output dein. Ye context rot ka #1 preventive fix hai.

### Component 4: Context management

```python
def _manage_context(self, messages, state):
    tokens = count_tokens(messages)

    # Tier 1: purane tool results prune karo (sasta, structure safe)
    if tokens > self.config.max_context_tokens * 0.6:
        messages = prune_old_tool_results(messages, keep_last=5)
        tokens = count_tokens(messages)

    # Tier 2: compaction (mehnga — ek LLM call)
    if tokens > self.config.max_context_tokens * 0.8:
        messages = self._compact(messages, state)

    return messages

def _compact(self, messages, state):
    summary = llm_summarize(messages[1:-6], instruction=f"""
Is agent run ka summary banao. YE PRESERVE KARO:
1. Original task: {state.task}
2. Kya complete ho gaya (concrete, verified facts)
3. Kya approaches FAIL hui (aur kyun) — ye CRITICAL hai
4. Key findings / data jo aage kaam aayega
5. Pending steps

Chhote details, verbose tool output, aur exploration jo kahin nahi gayi — skip karo.
""")
    return [messages[0],
            {"role": "user", "content": f"[EARLIER PROGRESS]\n{summary}"},
            *messages[-6:]]
```

**Compaction ke 3 rules (dohra raha hoon kyunki critical hai):**
1. **Failed approaches preserve karo** — warna agent same galti dohrayega, guaranteed
2. **Original task verbatim rakho** — paraphrase mein drift hota hai
3. Compaction ek extra LLM call hai — cost/latency add karti hai

### Component 5: Error handling

**Ek fundamental principle:**

```python
# ❌ GALAT — exception raise karo
result = tool.execute(args)     # exception → poora agent crash

# ✅ SAHI — model ko batao
try:
    result = tool.execute(args)
except Exception as e:
    result = f"Error: {e}. Ye try karo: {suggest_fix(e)}"
    # → Model padhega, samjhega, alag approach lega
```

**Errors ko conversation ka hissa banao, crash nahi.** Model **actually recover karta hai** — ye tool calling ki underrated superpower hai.

```python
# Error messages ko ACTIONABLE banao
❌ "Error: 404"
✅ "Error: file 'src/uitls.py' nahi mili. Kya tumhara matlab 'src/utils.py' tha? "
   "Directory listing: [app.py, utils.py, config.py]"
```

### Component 6: Permission system

```python
class PermissionPolicy:
    ALLOW = "allow"        # chup-chaap chalao
    ASK   = "ask"          # human approval
    DENY  = "deny"         # block

POLICIES = {
    "read_file":     ALLOW,
    "search":        ALLOW,
    "list_dir":      ALLOW,
    "write_file":    ASK,       # scoped path ke andar
    "run_tests":     ALLOW,
    "git_commit":    ASK,
    "delete_file":   ASK,
    "send_email":    ASK,
    "run_migration": DENY,      # agent ko allowed hi nahi
    "prod_deploy":   DENY,
}
```

**Aur "auto-approve" mode ke liye scoping:**
```python
# Read-only tools ALWAYS allow
# Write tools allow SIRF workspace ke andar
def is_safe_write(path):
    resolved = Path(path).resolve()
    return resolved.is_relative_to(WORKSPACE.resolve())     # path traversal block
```

> **Ye security boundary tumhare code mein hai, prompt mein nahi.** Prompt injection prompt-based rules bypass kar sakta hai; wo `Path.resolve()` check bypass nahi kar sakta.

### Component 7: Observability

```python
class Tracer:
    @contextmanager
    def span(self, name, **attrs):
        span_id, t0 = uuid4(), time.time()
        self.emit({"event": "span_start", "id": span_id, "name": name, **attrs})
        try:
            yield
            self.emit({"event": "span_end", "id": span_id,
                       "duration_ms": (time.time()-t0)*1000, "status": "ok"})
        except Exception as e:
            self.emit({"event": "span_end", "id": span_id,
                       "duration_ms": (time.time()-t0)*1000,
                       "status": "error", "error": str(e)})
            raise
```

**Har agent run ke liye ye log karo:**

| Field | Kyun |
|---|---|
| `run_id` | Correlation |
| `task` | Kya maanga tha |
| `steps` | Kitne lage |
| `tool_calls` | Sequence, args, results, durations |
| `tokens` (in/out/cached) | Cost analysis |
| `cost_usd` | Budget tracking |
| `duration_ms` | Latency |
| `status` | complete / max_steps / error / refused |
| `model` + `prompt_version` | **Regression debugging** |
| `cache_hit_rate` | Optimization signal |

> **Bina tracing, agent debug karna impossible hai.** Non-determinism ki wajah se tum "reproduce" nahi kar sakte — tumhe **record** karna padta hai. Ye optional nahi hai.

### Component 8: State persistence

```python
# Long-running agents crash hote hain. Resumable banao.
def save_checkpoint(run_id, messages, state):
    db.save(run_id, {"messages": messages, "state": asdict(state),
                     "timestamp": now()})

def resume(run_id):
    ckpt = db.load(run_id)
    return Agent(...).run_from(ckpt["messages"], RunState(**ckpt["state"]))
```

---

## Part 4: System prompt design for agents

Agent ka system prompt normal chat se different hota hai:

```python
AGENT_SYSTEM = """Tum ek autonomous coding agent ho.

## Environment
- Working directory: /workspace
- Tests: `pytest tests/`
- Tum standalone chal rahe ho — user real-time nahi dekh raha

## Approach
Kaam shuru karne se pehle relevant files padho. Ek step mein ek cheez.
Change karne ke baad tests chalao.

## Verification (ye MANDATORY hai)
Progress report karne se pehle, HAR claim ko is session ke tool result se
verify karo. "Main ne X update kiya" sirf tab bolo jab tumne actually
edit tool call kiya aur wo succeed hua. Jo verified nahi hai, uske liye
explicitly bolo ki verified nahi hai.

## Autonomy
Choti decisions (variable naming, formatting, do equivalent approaches mein
se ek) khud lo aur mention kar do — poochho mat. Scope changes ya
destructive actions ke liye poochho.

## Stopping
Tum tab ruko jab:
- Task complete ho aur tests pass ho, YA
- Tum blocked ho aur tumhe information chahiye jo sirf user de sakta hai

Turn khatam karne se pehle apna aakhiri paragraph dekho. Agar wo ek plan hai,
ek sawaal hai, ya "main ab X karunga" jaisa promise hai — to wo kaam ABHI
karo tool calls se. Turn khatam sirf tab karo jab kaam poora ho.

## Communication
Tumhara text output user tool calls ke beech padhta hai. Wo tumhari thinking
nahi dekh sakta. Outcome pehle likho, detail baad mein. Poore sentences mein
likho, shorthand mein nahi.
"""
```

### Har section kaunsa failure fix karta hai

| Section | Fix karta hai |
|---|---|
| Environment | Agent galat commands try karta tha |
| Verification | **Hallucinated progress** (sabse khatarnak failure) |
| Autonomy | Agent har choti baat pe poochta tha (ya kuch bhi na poochta tha) |
| Stopping | **Premature completion** aur "I'll now do X" bolke ruk jaana |
| Communication | Agent ka output unreadable shorthand ban jaata tha |

---

## Part 5: Harness maturity levels

```
LEVEL 0: while True + tools
  → Demo mein chalega. Production mein ek raat mein $500 ka bill.

LEVEL 1: + step limit, error handling, allowlist
  → Basic safety. Crash nahi hoga.

LEVEL 2: + cost limit, timeout, loop detection, tool output shaping
  → Predictable resource usage. Ye MINIMUM production bar hai.

LEVEL 3: + context management (prune + compact), permission gates
  → Long tasks handle kar sakta hai, destructive ops safe.

LEVEL 4: + tracing, eval suite, state persistence, resumability
  → Debuggable, measurable, reliable. Ye REAL production hai.

LEVEL 5: + human-in-loop UI, sub-agents, semantic caching,
          model routing, task budgets
  → Mature product.
```

> **Reality check:** Log Level 0 pe agent bana ke "agents production-ready nahi hain" complain karte hain. Problem agents mein nahi, **harness mein** hai.
>
> **Level 2 minimum bar hai.** Uske neeche production mein na jao.

---

## Part 6: Testing agents (non-determinism ke saath)

Traditional tests kaam nahi karte. Layered approach:

```python
# LAYER 1: Tools ko normally test karo (deterministic!)
def test_search_tool():
    assert "utils.py" in search_files("utils")

# LAYER 2: Harness guards test karo (mock model se)
def test_step_limit():
    agent = Agent(tools, AgentConfig(max_steps=3), tracer)
    agent.llm = mock_always_calls_tool()      # infinite loop simulate
    result = agent.run("task")
    assert result.status == "max_steps_exceeded"
    assert result.steps == 3

def test_loop_detection():
    agent.llm = mock_repeats_same_call()
    result = agent.run("task")
    assert "ALAG approach" in result.messages[-1]["content"]

def test_denied_tool_not_executed():
    agent = Agent(tools, AgentConfig(require_approval_for={"delete_file"}), tracer)
    with patch("harness.request_approval", return_value=False):
        result = agent.run("delete everything")
    assert not deleted_anything()

# LAYER 3: End-to-end EVALS (statistical, N runs)
def test_agent_task_success_rate():
    cases = load_eval_cases()          # 20 tasks with verifiable outcomes
    results = [run_agent(c) for c in cases for _ in range(3)]   # 3x per case
    success_rate = sum(verify(r) for r in results) / len(results)
    assert success_rate > 0.80          # THRESHOLD, exact match nahi
    assert median_cost(results) < 0.50
    assert p95_steps(results) < 20
```

> **Key shift:** `assert output == expected` **nahi**. `assert success_rate > threshold` **haan**.
>
> Aur cost/steps bhi assert karo — warna accuracy improve hoti rahegi aur bill silently 10x ho jaayega.

---

## Khud se pooch 🤔

1. "Harness" kya hai? Model commodity kyun hai aur harness differentiator kyun?
2. Harness ke 8 components naam se batao.
3. Tool execution mein 5 safety layers kaunse hain? Har ek kya rokta hai?
4. Tool output shaping kyun **critical** hai? Ek concrete bad-vs-good example do.
5. Tool error pe exception raise karne ke bajaye model ko batana kyun better hai?
6. Loop detection mein model ko *batana* (silently break karne ke bajaye) kyun better hai?
7. Agents ko test karne ke 3 layers kya hain? Layer 3 mein `assert` kaisa dikhta hai aur kyun?
8. Ek agent raat bhar chala aur $400 kharch kar diya, kaam adhoora. Kaunse harness components missing the?

<details>
<summary>Answer 8</summary>

**Diagnosis: ye Level 0/1 harness hai. Minimum 4 guards missing the.**

**Directly missing (jinse ye exact failure hua):**

1. **Cost limit** — `max_cost_usd`. Ek line ka check, $400 ki bachat. Ye #1 miss hai.
2. **Step limit** — `max_steps=30`. Raat bhar chalne ka matlab hai koi cap nahi tha.
3. **Timeout** — `timeout_seconds`. Wall-clock cap.
4. **Loop detection** — "kaam adhoora" + "bahut steps" = classic loop signature. Agent probably same 3 tools ghoomta raha.

**Aur likely missing (jinse root cause aaya):**

5. **Context management** — 100+ steps ke baad context rot ho gaya hoga. Agent original task bhool gaya, apne purane failed attempts ko naya samajhne laga. Ye "kaam adhoora" ka structural karan hai.
6. **Tool output shaping** — agar ek tool ne 100K tokens return kiye, to har subsequent step pe wo dobara bhej rahe ho. Cost multiply.
7. **Tracing** — ab tumhe pata bhi nahi ki kya hua. Ye sabse painful miss hai: **tum diagnose nahi kar sakte.**
8. **Checkpointing** — agar crash hua to kuch bacha bhi nahi.

**Immediate fix (aaj hi, 30 minutes):**
```python
config = AgentConfig(
    max_steps=30,
    max_cost_usd=2.0,
    timeout_seconds=900,
    max_context_tokens=150_000,
)
```
Plus loop detection aur structured logging. Ye 4 cheezein 90% ke aise incidents rok deti hain.

**Structural fix (agla hafta):**
- Tracing add karo → phir tum actually dekh paoge ki kya hua
- Tool outputs bound karo
- Context pruning + compaction
- Ek eval suite banao jo cost aur steps bhi assert kare, sirf accuracy nahi

**Aur ek stepping-back sawaal:** Ye task **agent** hona chahiye tha? Agar 100+ steps lage aur phir bhi adhoora raha — ho sakta hai task model ki current capability se bahar tha (Question 3 of the 4-question test: **Viability**), ya scope bahut bada tha. Ek bada agent task ko 5 chhote well-scoped agent tasks mein todna often better kaam karta hai.
</details>

---

**Next:** [`03-memory-systems.md`](03-memory-systems.md)
