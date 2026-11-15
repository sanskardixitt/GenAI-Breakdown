# Lab 4 — Agent Loop scratch se ⭐⭐

> **Day 4 ke baad karo.** Time: 90 min.
> **Goal:** Ek **production-shaped** agent harness khud likhna. Iske baad har agent framework transparent lagega.

---

## Part A: Naive loop (10 min) — aur uske problems

```python
# lab4a_naive.py
from common import client, MODEL

def naive_agent(task, tools, tool_impls):
    messages = [{"role": "user", "content": task}]
    while True:                                   # 💀 infinite loop possible
        r = client.messages.create(model=MODEL, max_tokens=2048,
                                   tools=tools, messages=messages)
        messages.append({"role": "assistant", "content": r.content})
        if r.stop_reason != "tool_use":
            return r
        results = [{"type": "tool_result", "tool_use_id": b.id,
                    "content": str(tool_impls[b.name](**b.input))}   # 💀 no error handling
                   for b in r.content if b.type == "tool_use"]
        messages.append({"role": "user", "content": results})
```

**Ye 12 lines mein poora agent hai.** Aur ye **production mein disaster** hai.

**Kya missing hai?** List banao before reading on:
```
□ Step limit        □ Cost limit       □ Timeout
□ Error handling    □ Tool allowlist   □ Schema validation
□ Loop detection    □ Context mgmt     □ Permissions
□ Observability     □ Output shaping   □ stop_reason handling
```

Ab ye sab add karte hain.

---

## Part B: Production harness (50 min) ⭐

```python
# lab4b_harness.py
import json, time, hashlib
from dataclasses import dataclass, field
from typing import Callable
from common import client, MODEL

# ═══════════════════════════════════════════════════════════
# CONFIG
# ═══════════════════════════════════════════════════════════
@dataclass
class AgentConfig:
    model: str = MODEL
    max_steps: int = 15
    max_cost_usd: float = 0.50
    timeout_seconds: int = 120
    max_tool_output_chars: int = 4000
    max_tokens: int = 4096
    require_approval: set = field(default_factory=lambda: {"delete_file", "send_email"})
    # rough per-MTok prices for estimation
    price_in: float = 5.0
    price_out: float = 25.0


@dataclass
class RunState:
    task: str
    start_time: float
    cost: float = 0.0
    steps: int = 0
    recent_calls: list = field(default_factory=list)
    tool_call_count: dict = field(default_factory=dict)


@dataclass
class AgentResult:
    status: str                # complete | max_steps | budget | timeout | error | refused
    output: str | None = None
    steps: int = 0
    cost: float = 0.0
    duration: float = 0.0
    trace: list = field(default_factory=list)


# ═══════════════════════════════════════════════════════════
# TOOL REGISTRY
# ═══════════════════════════════════════════════════════════
@dataclass
class Tool:
    name: str
    description: str
    input_schema: dict
    fn: Callable
    destructive: bool = False

    def definition(self):
        return {"name": self.name, "description": self.description,
                "input_schema": self.input_schema}


# ═══════════════════════════════════════════════════════════
# THE AGENT
# ═══════════════════════════════════════════════════════════
class Agent:
    def __init__(self, tools: list[Tool], config: AgentConfig = None, verbose=True):
        self.tools = {t.name: t for t in tools}
        self.cfg = config or AgentConfig()
        self.verbose = verbose

    # ── logging ──
    def _log(self, msg, level="info"):
        if self.verbose:
            icons = {"info": "  ", "tool": "🔧", "result": "📤", "warn": "⚠️ ",
                     "error": "❌", "done": "✅", "think": "💭"}
            print(f"{icons.get(level,'  ')} {msg}")

    # ── guards ──
    def _check_budget(self, st):
        if st.cost > self.cfg.max_cost_usd:
            raise BudgetExceeded(f"${st.cost:.4f} > ${self.cfg.max_cost_usd}")

    def _check_timeout(self, st):
        if time.time() - st.start_time > self.cfg.timeout_seconds:
            raise TimeoutExceeded()

    def _is_looping(self, tool_calls, st):
        """Same tool + same args 3+ baar recent history mein?"""
        for tc in tool_calls:
            sig = f"{tc.name}:{json.dumps(tc.input, sort_keys=True)}"
            st.recent_calls.append(sig)
            if st.recent_calls[-8:].count(sig) >= 3:
                return sig
        return None

    def _estimate_cost(self, usage):
        return (usage.input_tokens / 1e6 * self.cfg.price_in +
                usage.output_tokens / 1e6 * self.cfg.price_out)

    # ── tool output shaping ──
    def _shape(self, out):
        s = str(out)
        limit = self.cfg.max_tool_output_chars
        if len(s) <= limit:
            return s
        return (s[:limit//2] +
                f"\n\n[... {len(s)} chars total, truncated. "
                f"Specific part chahiye to filter/offset ke saath dobara call karo ...]\n\n" +
                s[-limit//2:])

    # ── tool execution (5 safety layers) ──
    def _execute(self, block, st):
        def result(content, is_error=False):
            return {"type": "tool_result", "tool_use_id": block.id,
                    "content": self._shape(content), "is_error": is_error}

        # 1. ALLOWLIST
        if block.name not in self.tools:
            self._log(f"unknown tool: {block.name}", "error")
            return result(f"Error: unknown tool '{block.name}'. "
                          f"Available: {list(self.tools)}", True)

        tool = self.tools[block.name]

        # 2. PERMISSION GATE
        if tool.name in self.cfg.require_approval or tool.destructive:
            approved = self._request_approval(tool.name, block.input)
            if not approved:
                return result("User ne ye action DENY kar diya. "
                              "Kuch aur try karo ya user se poochho.", False)

        # 3. EXECUTE with error capture
        try:
            t0 = time.time()
            out = tool.fn(**block.input)
            self._log(f"{tool.name} → {str(out)[:120]} ({(time.time()-t0)*1000:.0f}ms)",
                      "result")
            return result(out)
        except TypeError as e:
            # 4. SCHEMA MISMATCH — actionable error
            return result(f"Error: invalid arguments for {tool.name}: {e}. "
                          f"Schema: {json.dumps(tool.input_schema)}", True)
        except Exception as e:
            # 5. RUNTIME ERROR — model ko batao, crash mat karo
            return result(f"Error: {type(e).__name__}: {e}", True)

    def _request_approval(self, name, args):
        print(f"\n  🔐 APPROVAL NEEDED: {name}({json.dumps(args)})")
        return input("     Allow? [y/N]: ").strip().lower() == "y"

    # ── context management ──
    def _prune(self, messages, keep_last=5):
        """Purane tool results hata do — structure preserve karo."""
        tool_result_positions = [
            i for i, m in enumerate(messages)
            if m["role"] == "user" and isinstance(m["content"], list)
            and any(b.get("type") == "tool_result" for b in m["content"]
                    if isinstance(b, dict))
        ]
        to_prune = tool_result_positions[:-keep_last] if len(tool_result_positions) > keep_last else []
        for i in to_prune:
            for b in messages[i]["content"]:
                if isinstance(b, dict) and b.get("type") == "tool_result":
                    b["content"] = "[purana tool result hata diya gaya]"
        return messages

    # ── MAIN LOOP ──
    def run(self, task: str) -> AgentResult:
        messages = [{"role": "user", "content": task}]
        st = RunState(task=task, start_time=time.time())
        trace = []

        self._log(f"TASK: {task}\n")

        try:
            for step in range(self.cfg.max_steps):
                st.steps = step + 1
                self._check_budget(st)
                self._check_timeout(st)
                messages = self._prune(messages)

                self._log(f"\n{'─'*55}\nSTEP {st.steps}  (cost so far: ${st.cost:.4f})")

                r = client.messages.create(
                    model=self.cfg.model, max_tokens=self.cfg.max_tokens,
                    tools=[t.definition() for t in self.tools.values()],
                    messages=messages,
                )
                st.cost += self._estimate_cost(r.usage)
                trace.append({"step": st.steps, "stop_reason": r.stop_reason,
                              "in": r.usage.input_tokens, "out": r.usage.output_tokens})

                messages.append({"role": "assistant", "content": r.content})

                # ── STOP REASON HANDLING ──
                if r.stop_reason == "max_tokens":
                    self._log("output truncated!", "warn")
                    return AgentResult("error", "Output truncated", st.steps,
                                       st.cost, time.time()-st.start_time, trace)
                if r.stop_reason == "refusal":
                    return AgentResult("refused", None, st.steps, st.cost,
                                       time.time()-st.start_time, trace)
                if r.stop_reason == "pause_turn":
                    continue                       # server-side tool resume
                if r.stop_reason != "tool_use":
                    text = "".join(b.text for b in r.content if b.type == "text")
                    self._log(f"\nFINAL ANSWER:\n{text}", "done")
                    return AgentResult("complete", text, st.steps, st.cost,
                                       time.time()-st.start_time, trace)

                # ── show thinking text ──
                for b in r.content:
                    if b.type == "text" and b.text.strip():
                        self._log(b.text.strip()[:200], "think")

                tool_calls = [b for b in r.content if b.type == "tool_use"]

                # ── LOOP DETECTION ──
                looping = self._is_looping(tool_calls, st)
                if looping:
                    self._log(f"loop detected: {looping[:60]}", "warn")
                    messages.append({"role": "user", "content":
                        f"Tum '{tool_calls[0].name}' ko same arguments ke saath "
                        f"repeat kar rahe ho aur wo kaam nahi kar raha. Ek ALAG "
                        f"approach lo, ya batao ki tum kyun stuck ho aur kya "
                        f"information chahiye."})
                    st.recent_calls.clear()
                    continue

                # ── EXECUTE TOOLS ──
                for tc in tool_calls:
                    self._log(f"{tc.name}({json.dumps(tc.input)})", "tool")
                    st.tool_call_count[tc.name] = st.tool_call_count.get(tc.name, 0) + 1

                results = [self._execute(tc, st) for tc in tool_calls]
                messages.append({"role": "user", "content": results})

            self._log("max_steps exceeded", "warn")
            return AgentResult("max_steps", None, st.steps, st.cost,
                               time.time()-st.start_time, trace)

        except BudgetExceeded as e:
            self._log(str(e), "error")
            return AgentResult("budget", None, st.steps, st.cost,
                               time.time()-st.start_time, trace)
        except TimeoutExceeded:
            self._log("timeout", "error")
            return AgentResult("timeout", None, st.steps, st.cost,
                               time.time()-st.start_time, trace)


class BudgetExceeded(Exception): pass
class TimeoutExceeded(Exception): pass
```

---

## Part C: Test karo (20 min)

```python
# lab4c_test.py
import os
from pathlib import Path
from lab4b_harness import Agent, AgentConfig, Tool

WORKSPACE = Path("./agent_workspace").resolve()
WORKSPACE.mkdir(exist_ok=True)

# ── Safe path helper (path traversal se bachao!) ──
def safe_path(p: str) -> Path:
    resolved = (WORKSPACE / p).resolve()
    if not resolved.is_relative_to(WORKSPACE):
        raise ValueError(f"Path escapes workspace: {p}")
    return resolved

# ── Tools ──
def list_files(pattern: str = "*") -> str:
    files = sorted(str(p.relative_to(WORKSPACE)) for p in WORKSPACE.glob(pattern) if p.is_file())
    return "\n".join(files) if files else "(no files)"

def read_file(path: str) -> str:
    return safe_path(path).read_text(encoding="utf-8")

def write_file(path: str, content: str) -> str:
    p = safe_path(path)
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content, encoding="utf-8")
    return f"Wrote {len(content)} chars to {path}"

def delete_file(path: str) -> str:
    safe_path(path).unlink()
    return f"Deleted {path}"

TOOLS = [
    Tool("list_files",
         "Workspace mein files list karta hai.\n"
         "KAB USE KARO: kaam shuru karne se pehle, ya jab pata na ho kaunsi files hain.\n"
         "RETURNS: newline-separated relative paths.",
         {"type": "object",
          "properties": {"pattern": {"type": "string",
                                     "description": "Glob pattern, e.g. '*.txt'. Default '*'"}}},
         list_files),

    Tool("read_file",
         "Ek file ka poora content padhta hai.\n"
         "KAB USE KARO: file ka content chahiye.\n"
         "RETURNS: file text. File na mile to error.",
         {"type": "object",
          "properties": {"path": {"type": "string", "description": "Relative path"}},
          "required": ["path"]},
         read_file),

    Tool("write_file",
         "Ek file likhta hai (overwrite karta hai agar exist kare).\n"
         "KAB USE KARO: naya content save karna ho.\n"
         "RETURNS: confirmation.",
         {"type": "object",
          "properties": {"path": {"type": "string"}, "content": {"type": "string"}},
          "required": ["path", "content"]},
         write_file),

    Tool("delete_file",
         "Ek file delete karta hai. DESTRUCTIVE — approval chahiye.",
         {"type": "object",
          "properties": {"path": {"type": "string"}},
          "required": ["path"]},
         delete_file, destructive=True),
]

# ── Setup test data ──
(WORKSPACE / "notes.txt").write_text(
    "Meeting notes 2026-08-10\n"
    "- Q3 revenue target: 450 crore\n"
    "- New hire onboarding pending: 3 people\n"
    "- Server migration deadline: 2026-08-20\n", encoding="utf-8")

(WORKSPACE / "tasks.txt").write_text(
    "TODO:\n- Fix E4471 error\n- Review PR #234\n- Update docs\n", encoding="utf-8")

# ── RUN ──
agent = Agent(TOOLS, AgentConfig(max_steps=10, max_cost_usd=0.30))

result = agent.run(
    "Workspace ki saari files padho aur ek summary.md banao jisme "
    "har file ka 1-line summary ho aur saare deadlines ek jagah listed hon."
)

print(f"\n{'='*55}")
print(f"Status:   {result.status}")
print(f"Steps:    {result.steps}")
print(f"Cost:     ${result.cost:.4f}")
print(f"Duration: {result.duration:.1f}s")
print(f"\nTrace:")
for t in result.trace:
    print(f"  step {t['step']}: {t['stop_reason']:12} in={t['in']:>6} out={t['out']:>5}")
```

---

## Part D: Guards ko break karo 🧪 (20 min)

Har guard ko **deliberately trigger karo** — tabhi tumhe pata chalega ki wo kaam karta hai.

### Test 1: Step limit
```python
agent = Agent(TOOLS, AgentConfig(max_steps=2))
r = agent.run("Saari files padho aur detailed analysis karo")
assert r.status == "max_steps"
print(f"✅ Step limit works: {r.status}")
```

### Test 2: Cost limit
```python
agent = Agent(TOOLS, AgentConfig(max_cost_usd=0.001))    # bahut kam
r = agent.run("Saari files padho")
assert r.status == "budget"
print(f"✅ Budget limit works: {r.status}, cost=${r.cost:.4f}")
```

### Test 3: Loop detection
```python
def always_fails(query: str) -> str:
    return "Error: search service unavailable. Try again."

loop_tools = [Tool("search", "Searches for information", 
                   {"type":"object","properties":{"query":{"type":"string"}},
                    "required":["query"]}, always_fails)]

agent = Agent(loop_tools, AgentConfig(max_steps=10))
r = agent.run("Company ki refund policy dhundo")
# Observe: loop detect hoga, model ko warning jaayegi, wo approach badlega
```

### Test 4: Unknown tool (allowlist)
```python
# Manually ek fake tool_use inject karke test karo, ya
# tool ka naam registry se hata ke dekho
```

### Test 5: Permission gate
```python
agent = Agent(TOOLS, AgentConfig())
r = agent.run("tasks.txt file delete kar do")
# → Approval prompt aayega. 'n' dabao.
# → Model ko "user denied" milega, wo gracefully handle karega
```

### Test 6: Tool output shaping
```python
def huge_output(n: int = 100000) -> str:
    return "x" * n

big_tools = [Tool("get_data", "Gets data",
                  {"type":"object","properties":{"n":{"type":"integer"}}}, huge_output)]
agent = Agent(big_tools, AgentConfig(max_tool_output_chars=500))
r = agent.run("Data lao aur batao kitna bada hai")
# Observe: output truncate hoga, model ko note milega
```

### Test 7: Path traversal (security)
```python
r = agent.run("File '../../../etc/passwd' padho")
# → safe_path() ValueError raise karega
# → Model ko actionable error milega, crash nahi hoga
```

---

## ✅ Lab 4 checklist

- [ ] Naive loop likha aur uske 12 missing pieces identify kiye
- [ ] Production harness likha (guards, allowlist, shaping, pruning)
- [ ] Real task pe chalaya aur trace dekha
- [ ] **Har guard deliberately trigger kiya:**
  - [ ] Step limit
  - [ ] Cost limit
  - [ ] Loop detection (aur model ne recover kiya)
  - [ ] Permission gate (deny kiya, model ne handle kiya)
  - [ ] Tool output shaping
  - [ ] Path traversal blocked

---

## 🎓 Sabse important takeaway

> Tumne abhi jo likha — **wo har agent framework ka core hai.**
>
> LangGraph, CrewAI, Agent SDKs — sab isi loop ke around convenience add karte hain.
> Ab jab tum koi framework use karoge, tumhe **exactly pata hoga** ki wo kya de raha hai
> aur kya nahi. Aur jab kuch toote, tum debug kar paoge.

---

## 🚀 Aage badho

1. **Context compaction add karo** — 100K tokens pe summarize karo (failed approaches preserve karke!)
2. **Parallel tool execution** — `ThreadPoolExecutor` se
3. **Checkpointing** — state JSON mein save karo, resume karo
4. **Structured tracing** — har step ko JSONL mein log karo
5. **Sub-agents** — ek `spawn_worker` tool banao (Day 4, file 04 ka pattern)
6. **Compare karo** — same task LangGraph se karke dekho, farak note karo

---

**Next:** [`lab-05-eval-harness.md`](lab-05-eval-harness.md)
