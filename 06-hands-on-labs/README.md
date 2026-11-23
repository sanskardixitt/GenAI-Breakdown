# 06 — Hands-On Labs 🛠️

> **Theory padh li? Ab code likho.** Ye labs deliberately **framework-free** hain — taaki tum samjho ki neeche kya ho raha hai.
>
> **Rule:** Har lab **khud type karo**. Copy-paste se kuch retain nahi hota.

---

## Setup (ek baar)

```powershell
# Windows PowerShell
python -m venv .venv
.venv\Scripts\Activate.ps1

pip install anthropic python-dotenv
pip install sentence-transformers rank-bm25 numpy
pip install pypdf
```

`.env` file (repo root, **`.gitignore` mein daalo**):
```
ANTHROPIC_API_KEY=sk-ant-...
```

```python
# common.py — sab labs isko import karenge
import os
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()
client = Anthropic()          # ANTHROPIC_API_KEY env se

MODEL = "claude-opus-5"
FAST_MODEL = "claude-haiku-4-5"
```

---

## Lab index

| Lab | Topic | Day | Time |
|---|---|---|---|
| [Lab 1](lab-01-tokens-embeddings.md) | Tokens, embeddings, similarity | 1 | 45 min |
| [Lab 2](lab-02-structured-output-tools.md) | Structured output + tool calling | 2 | 60 min |
| [Lab 3](lab-03-hybrid-rag.md) | Hybrid RAG + reranking ⭐ | 3 | 90 min |
| [Lab 4](lab-04-agent-loop.md) | Agent loop scratch se ⭐⭐ | 4 | 90 min |
| [Lab 5](lab-05-eval-harness.md) | Eval harness ⭐⭐ | 6 | 60 min |
| [Lab 6](lab-06-capstone.md) | Capstone: agentic RAG app | 7 | 3 ghante |

---

## Kaise karo

1. **Theory pehle padho** (us din ki files)
2. **Lab file kholo**, code khud type karo
3. **Chalao**, output dekho
4. **Todo aur theek karo** — deliberately kuch break karo, dekho kya hota hai
5. **Extend karo** — har lab ke end mein "Aage badho" section hai

> 🔑 **Sabse important lab: Lab 4 (agent loop).** Wo likhne ke baad har agent framework tumhe transparent lagega.

---

## Common utilities

Ye file bana lo — kai labs mein kaam aayegi:

```python
# utils.py
import json, time, hashlib
from functools import wraps

def timed(fn):
    """Decorator — function ka time measure karo."""
    @wraps(fn)
    def wrapper(*a, **kw):
        t0 = time.time()
        result = fn(*a, **kw)
        print(f"  ⏱️  {fn.__name__}: {(time.time()-t0)*1000:.0f}ms")
        return result
    return wrapper

def show_usage(response):
    """Token usage aur cost dikhao."""
    u = response.usage
    print(f"  📊 in={u.input_tokens} out={u.output_tokens} "
          f"cached={getattr(u, 'cache_read_input_tokens', 0)}")

def text_of(response):
    """Response se text blocks nikalo (safe)."""
    return "".join(b.text for b in response.content if b.type == "text")
```

---

## Debugging tips

Jab kuch kaam na kare:

```python
# 1. ACTUAL prompt print karo (jo bheja jaa raha hai, jo tumne socha wo nahi)
print(json.dumps(messages, indent=2)[:3000])

# 2. stop_reason HAMESHA check karo
print(f"stop_reason: {response.stop_reason}")

# 3. Poora response inspect karo
for block in response.content:
    print(f"  [{block.type}]", block)

# 4. Usage dekho (cache hit ho raha hai?)
show_usage(response)
```

> **40% bugs step 1 mein mil jaate hain** — prompt wo nahi hota jo tumne socha tha.

---

**Shuru karo:** [`lab-01-tokens-embeddings.md`](lab-01-tokens-embeddings.md)
