# Lab 2 — Structured Output aur Tool Calling

> **Day 2 ke baad karo.** Time: 60 min.
> **Goal:** Tool calling ka mechanism apne haathon se dekhna — magic nahi, JSON hai.

---

## Part A: Structured output (15 min)

```python
# lab2a_structured.py
from pydantic import BaseModel, Field
from typing import Literal
from common import client, MODEL

class EmailTriage(BaseModel):
    summary: str = Field(description="Max 15 words")
    action_required: bool
    deadline: str | None = Field(description="YYYY-MM-DD ya null")
    priority: Literal["HIGH", "MEDIUM", "LOW"]
    sentiment: Literal["positive", "neutral", "negative"]

EMAIL = """
Subject: URGENT — Server migration blocked

Hi team,

Hum production migration nahi kar paa rahe kyunki staging environment
mein DB connection fail ho raha hai. Client ne 20 August tak deadline
di hai. Agar kal tak fix nahi hua to hum miss kar denge.

Please priority pe dekho.
- Rahul
"""

response = client.messages.parse(
    model=MODEL,
    max_tokens=1024,
    messages=[{"role": "user", "content": f"Is email ko triage karo:\n\n{EMAIL}"}],
    output_format=EmailTriage,
)

triage = response.parsed_output
print(f"Summary:  {triage.summary}")
print(f"Action:   {triage.action_required}")
print(f"Deadline: {triage.deadline}")
print(f"Priority: {triage.priority}")
print(f"Sentiment:{triage.sentiment}")
print(f"\nType: {type(triage)}")     # ← ek REAL Pydantic object!
```

### 🧪 Todo aur dekho

```python
# 1. stop_reason check karo
print(response.stop_reason)

# 2. max_tokens ko 20 karke chalao — kya hota hai?
#    → stop_reason == "max_tokens", parsing fail
#    → YE production bug hai jo log miss karte hain

# 3. Ek unsupported constraint daalke dekho
class Bad(BaseModel):
    score: int = Field(ge=0, le=100)      # numeric constraints often unsupported
# → SDK ise strip karke client-side validate karega, ya error dega
```

---

## Part B: Tool calling — manual loop (30 min) ⭐

Ye lab ka core hai. **Khud type karo.**

```python
# lab2b_tools.py
import json
from common import client, MODEL

# ─────────────────────────────────────────────────────────
# 1. TOOL IMPLEMENTATIONS (plain Python functions)
# ─────────────────────────────────────────────────────────
WEATHER_DB = {
    "delhi": {"temp_c": 34, "condition": "haze", "humidity": 62},
    "mumbai": {"temp_c": 29, "condition": "rain", "humidity": 88},
    "bangalore": {"temp_c": 24, "condition": "cloudy", "humidity": 71},
}

def get_weather(city: str) -> str:
    key = city.lower().strip()
    if key not in WEATHER_DB:
        # ⭐ ACTIONABLE error — model isse recover kar sakta hai
        return (f"Error: '{city}' nahi mila. "
                f"Available cities: {', '.join(WEATHER_DB)}")
    w = WEATHER_DB[key]
    return f"{city}: {w['temp_c']}°C, {w['condition']}, humidity {w['humidity']}%"

def compare_temps(city_a: str, city_b: str) -> str:
    for c in (city_a, city_b):
        if c.lower().strip() not in WEATHER_DB:
            return f"Error: '{c}' nahi mila. Available: {', '.join(WEATHER_DB)}"
    ta = WEATHER_DB[city_a.lower().strip()]["temp_c"]
    tb = WEATHER_DB[city_b.lower().strip()]["temp_c"]
    return f"{city_a}: {ta}°C, {city_b}: {tb}°C. Difference: {abs(ta-tb)}°C"

TOOLS_IMPL = {"get_weather": get_weather, "compare_temps": compare_temps}

# ─────────────────────────────────────────────────────────
# 2. TOOL DEFINITIONS (schema — model ye padhta hai)
# ─────────────────────────────────────────────────────────
TOOLS = [
    {
        "name": "get_weather",
        "description": (
            "Ek city ka current weather deta hai.\n\n"
            "KAB USE KARO: jab user ek specific city ka mausam, temperature, "
            "ya conditions poochhe.\n"
            "KAB NA KARO: do cities compare karne ke liye — uske liye "
            "compare_temps use karo.\n"
            "RETURNS: temperature, condition, humidity. City na mile to "
            "error message with available cities."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string",
                         "description": "City ka naam, e.g. Delhi"},
            },
            "required": ["city"],
        },
    },
    {
        "name": "compare_temps",
        "description": (
            "Do cities ka temperature compare karta hai.\n\n"
            "KAB USE KARO: jab user do cities ka mausam compare kare "
            "('kahan zyada garmi hai?').\n"
            "RETURNS: dono temps aur unka difference."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "city_a": {"type": "string"},
                "city_b": {"type": "string"},
            },
            "required": ["city_a", "city_b"],
        },
    },
]

# ─────────────────────────────────────────────────────────
# 3. THE LOOP  ← YEH POORA AGENT HAI
# ─────────────────────────────────────────────────────────
def run(user_query: str, max_steps: int = 8, verbose: bool = True):
    messages = [{"role": "user", "content": user_query}]

    for step in range(max_steps):
        response = client.messages.create(
            model=MODEL, max_tokens=2048, tools=TOOLS, messages=messages,
        )

        if verbose:
            print(f"\n{'='*60}\nSTEP {step+1}  |  stop_reason: {response.stop_reason}")

        # Assistant ka POORA response append karo (tool_use blocks preserve!)
        messages.append({"role": "assistant", "content": response.content})

        # ── Stop conditions ──
        if response.stop_reason == "max_tokens":
            print("⚠️  Output truncated! max_tokens badhao.")
            return None
        if response.stop_reason != "tool_use":
            text = "".join(b.text for b in response.content if b.type == "text")
            if verbose: print(f"\n✅ FINAL ANSWER:\n{text}")
            return text

        # ── Tool execution ──
        tool_results = []
        for block in response.content:
            if block.type == "text" and verbose:
                print(f"💭 Model says: {block.text}")

            if block.type == "tool_use":
                if verbose:
                    print(f"🔧 CALLING: {block.name}({json.dumps(block.input)})")

                # ⭐ ALLOWLIST — model jo bole wo mat chalao
                if block.name not in TOOLS_IMPL:
                    result, is_error = f"Error: unknown tool '{block.name}'", True
                else:
                    try:
                        result, is_error = TOOLS_IMPL[block.name](**block.input), False
                    except Exception as e:
                        result, is_error = f"Error: {type(e).__name__}: {e}", True

                if verbose:
                    print(f"📤 RESULT: {result}")

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,        # ⭐ MATCH zaroori hai
                    "content": str(result),
                    "is_error": is_error,
                })

        # ⭐ SAARE results EK message mein
        messages.append({"role": "user", "content": tool_results})

    print("⚠️  max_steps exceeded")
    return None


if __name__ == "__main__":
    run("Delhi mein mausam kaisa hai?")
```

---

## Part C: Experiments (15 min) 🧪

Ab **deliberately break karo** aur seekho:

### Experiment 1: Parallel tool calls
```python
run("Delhi aur Mumbai dono ka mausam batao")
```
**Observe:** Model ek hi response mein **do** `tool_use` blocks bhejega. Dono results ek message mein wapas gaye.

### Experiment 2: Error recovery ⭐
```python
run("Kolkata mein mausam kaisa hai?")
```
**Observe:** Tool error return karega ("Kolkata nahi mila, available: delhi, mumbai, bangalore"). **Model recover karega** — ya to available city suggest karega ya user ko batayega. Ye tool calling ki superpower hai.

### Experiment 3: Tool selection
```python
run("Delhi aur Mumbai mein kahan zyada garmi hai?")
```
**Observe:** Model `compare_temps` chunega, do baar `get_weather` nahi. **Kyun?** Description mein clearly likha tha.

### Experiment 4: Description tod do 🔴
```python
# TOOLS[0]["description"] ko badal ke sirf "Gets weather" kar do
TOOLS[0]["description"] = "Gets weather"
TOOLS[1]["description"] = "Compares weather"
run("Delhi aur Mumbai mein kahan zyada garmi hai?")
```
**Observe:** Model ab confuse ho sakta hai, do `get_weather` calls kar sakta hai. **Ye lab ka sabse important lesson hai** — tool description hi model ka poora knowledge hai.

### Experiment 5: `tool_use_id` tod do 🔴
```python
# tool_results mein "tool_use_id": "wrong_id" kar do
```
**Observe:** 400 error. Ye ek common production bug hai.

### Experiment 6: `tool_choice` force karo
```python
response = client.messages.create(
    model=MODEL, max_tokens=1024, tools=TOOLS,
    tool_choice={"type": "tool", "name": "get_weather"},
    messages=[{"role": "user", "content": "Namaste, kaise ho?"}],
)
```
**Observe:** Model ko tool use karna **padega**, chahe zaroorat na ho.

---

## Part D: Tool runner se compare karo (10 min)

```python
# lab2d_runner.py
from anthropic import beta_tool
from common import client, MODEL

@beta_tool
def get_weather(city: str) -> str:
    """Ek city ka current weather deta hai.

    Args:
        city: City ka naam, e.g. Delhi
    """
    return get_weather_impl(city)      # Part B wala function

runner = client.beta.messages.tool_runner(
    model=MODEL, max_tokens=2048,
    tools=[get_weather],
    messages=[{"role": "user", "content": "Delhi ka mausam?"}],
)

for message in runner:
    print(message.stop_reason)
```

**Compare karo:**
- Kitne lines bache? (~40 lines vs ~5)
- Schema kahan se aaya? (docstring + type hints se auto)
- Kya control kho gaya? (Har iteration pe message milta hai — kaafi control hai)

> **Ab tum jaante ho tool runner kya abstract kar raha hai.** Wahi asli value hai is lab ka.

---

## ✅ Lab 2 checklist

- [ ] Structured output se typed object mila
- [ ] `max_tokens` chhota karke truncation dekha
- [ ] Manual tool loop **khud likha** aur chalaya
- [ ] Parallel tool calls dekhe
- [ ] Error recovery dekha (model ne recover kiya)
- [ ] Tool description tod ke dekha ki model confuse ho gaya
- [ ] `tool_use_id` mismatch se 400 dekha
- [ ] Tool runner se compare kiya

---

## 🚀 Aage badho

1. **Ek naya tool add karo** — `get_forecast(city, days)` — aur dekho ki model kab kaunsa chunta hai
2. **Overlapping tools banao** deliberately (`get_weather` aur `get_current_conditions`) — dekho model kya karta hai
3. **Human approval add karo** — ek `send_alert` tool banao jo approval maange
4. **Tool output truncate karo** — ek tool banao jo 50,000 characters return kare, phir shaping add karo

---

**Next:** [`lab-03-hybrid-rag.md`](lab-03-hybrid-rag.md)
