# 08 — Diagrams 🎨

> Repo ke **9 sabse important concepts** ke visual maps. Har diagram ek `.excalidraw` file hai — **editable**, read-only image nahi.

---

## Kaise kholein

| Tarika | Kaise |
|---|---|
| **Browser (sabse aasan)** | [excalidraw.com](https://excalidraw.com) → `Menu → Open` → file drag-drop karo |
| **VS Code** | Extension install karo: `pomdtr.excalidraw-editor` → file pe click, bas |
| **Obsidian** | Excalidraw plugin — files direct kaam karti hain |

> **Tip:** Diagram sirf *dekho* mat. Kholo, apne notes add karo, boxes hilao, jo samajh nahi aaya us pe `?` likho. Editable hone ka yahi point hai.

---

## Diagrams — kaunsa kab padho

| # | File | Concept | Padho iske saath |
|---|---|---|---|
| 01 | [01-transformer-forward-pass.excalidraw](01-transformer-forward-pass.excalidraw) | Ek token kaise banta hai — tokenizer se sampling tak, poora forward pass | [`00-foundations/02`](../00-foundations/02-transformer-aur-attention.md) |
| 02 | [02-attention-qkv.excalidraw](02-attention-qkv.excalidraw) | Q/K/V ka asli mechanism, softmax weights, causal mask, multi-head | [`00-foundations/02`](../00-foundations/02-transformer-aur-attention.md) |
| 03 | [03-context-window-aur-kv-cache.excalidraw](03-context-window-aur-kv-cache.excalidraw) | Context window mein kya bharo aur kis **order** mein · prompt caching · lost in the middle | [`00-foundations/03`](../00-foundations/03-tokens-embeddings-context.md), [`01-llm-behaviour/04`](../01-llm-behaviour/04-context-engineering.md) |
| 04 | [04-rag-pipeline.excalidraw](04-rag-pipeline.excalidraw) | Ingest lane + query lane, hybrid retrieval, rerank, aur **RAG kyun tootta hai** | [`02-rag/*`](../02-rag/) |
| 05 | [05-agent-loop-harness.excalidraw](05-agent-loop-harness.excalidraw) | Agent loop, harness ke components, tool execution ke 5 safety layers | [`03-agents/02`](../03-agents/02-agent-loop-aur-harness.md) |
| 06 | [06-multi-agent-patterns.excalidraw](06-multi-agent-patterns.excalidraw) | 4 patterns side-by-side + "kab multi-agent, kab NAHI" | [`03-agents/04`](../03-agents/04-multi-agent-patterns.md) |
| 07 | [07-mcp-architecture.excalidraw](07-mcp-architecture.excalidraw) | Host/client/server, 3 primitives (kaun control karta hai), N×M problem, security | [`03-agents/05`](../03-agents/05-mcp-model-context-protocol.md) |
| 08 | [08-production-stack.excalidraw](08-production-stack.excalidraw) | Request ka poora safar — gateway se guardrails tak + cost/latency levers | [`05-production/*`](../05-production/) |
| 09 | [09-eval-driven-development.excalidraw](09-eval-driven-development.excalidraw) | Traces → taxonomy → golden dataset → eval → change → CI gate ka loop | [`05-production/03`](../05-production/03-eval-driven-development.md) |

---

## Har diagram se ek line ka takeaway

1. **Transformer** — Attention = *communication*, FFN = *computation*. Ye 80 baar repeat hota hai, bas.
2. **Attention** — Har token poore context ko dekh sakta hai, aur khud decide karta hai kis pe dhyaan dena hai.
3. **Context window** — Stable cheezein pehle, variable baad mein. Warna prompt cache mar jaata hai.
4. **RAG** — Generation last aur sabse chhota problem hai. Retrieval quality hi answer quality hai.
5. **Agent loop** — Model commodity hai. **Harness tumhara product hai.**
6. **Multi-agent** — Default single agent hai. Multi-agent = N× cost + N× failure surface.
7. **MCP** — M×N integrations ko M+N banata hai. Aur har server ek prompt-injection surface hai.
8. **Production** — Demo se farak: guardrails, caching, routing, observability, evals.
9. **Evals** — `assert success_rate > threshold`, **na ki** `assert output == expected`.

---

## Revision ke liye use kaise karein

Diagram kholo, **saare text notes hide/delete kar do** (Ctrl+Z se wapas aa jaayega), sirf boxes aur arrows chhodo — aur khud se har box explain karo. Jahan atko, wahi tumhara gap hai → wapas us markdown file pe jao.

> Ye "blank map" technique hai. Padhne se 5× fast revision hoti hai.

---

## Diagrams regenerate / customize karna

Ye files hand-drawn nahi hain — [`_generator.py`](_generator.py) se banti hain (taaki layout consistent rahe aur update aasan ho).

```bash
cd 08-diagrams
python _generator.py        # saari 9 files dobara likh degi
```

Kuch badalna hai? `_generator.py` mein us diagram ka function (`d01`…`d09`) edit karo aur dobara chala do.
Agar tumne file ko Excalidraw mein manually edit kiya hai, to **generator mat chalao** — wo overwrite kar dega.
