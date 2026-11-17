# 50 Interview Questions (with answers)

> **Kaise use karo:** Pehle **likhkar** answer do, phir dekho. Padh ke "haan pata hai" bolna self-deception hai.
>
> ⭐ = frequently asked

---

## Foundations (1-10)

**Q1 ⭐ Transformer ne RNN ki kaunsi do problems solve ki?**
<details><summary>Answer</summary>

1. **Sequential processing** — RNN parallel nahi ho sakta (h3 ke liye h2 chahiye). Attention mein saare tokens ek saath process hote hain → GPU ka full use.
2. **Information bottleneck** — RNN poora past ek fixed vector mein squeeze karta tha, purani baat dhul jaati thi. Attention mein har token har token ko directly dekh sakta hai, distance matter nahi karta.
</details>

**Q2 Q, K, V kya hain? Ek analogy do.**
<details><summary>Answer</summary>
Library search: **Q** = tumhara sawaal, **K** = kitabon ke labels, **V** = actual content. Q·K se relevance score, softmax se weights, phir V ka weighted average. Har token apna Q banata hai aur sab tokens ke K se match karke decide karta hai kis pe dhyaan dena hai.
</details>

**Q3 ⭐ Attention quadratic kyun hai? Uska practical asar?**
<details><summary>Answer</summary>
n tokens → n×n attention matrix (har token har token ko dekhta hai). 10x context = 100x comparisons.
**Asar:** long context mehnga hai; isliye FlashAttention, sliding window, sparse attention jaise optimizations exist karte hain; aur isliye "sab context mein daal do" strategy galat hai.
</details>

**Q4 ⭐⭐ Prompt caching prefix-match pe kyun kaam karta hai?**
<details><summary>Answer</summary>
Prompt caching = **KV cache ko requests ke beech reuse karna**. Attention causal hai — token #500 ka K,V tokens #1-499 pe depend karta hai. Agar token #10 badla, #11 se aage sab K,V badal jaate hain, reuse impossible.
**Consequence:** stable content pehle (system, tools, examples), variable baad mein. `datetime.now()` system prompt mein = cache hit rate zero, **bina kisi error ke**.
</details>

**Q5 Positional encoding kyun zaroori hai?**
<details><summary>Answer</summary>
Attention ek **set operation** hai — usme order ka concept nahi. "kutta aadmi ko kaata" aur "aadmi kutta ko kaata" ka same math hoga. RoPE (rotary) aaj standard hai kyunki wo relative positions naturally encode karta hai aur extrapolate karta hai — **isliye 1M context windows possible hain**.
</details>

**Q6 Hindi English se 3-4x zyada tokens kyun leti hai?**
<details><summary>Answer</summary>
BPE tokenizer training corpus se seekha hai, jo English-heavy tha. Common English words ek token, Devanagari characters aksar 1-2 characters per token. **Practical:** Indian language products ka token budget 3-4x planning chahiye.
</details>

**Q7 ⭐ "Strawberry mein kitne r" wala problem kyun hota hai?**
<details><summary>Answer</summary>
Model "strawberry" ko `["str","aw","berry"]` dekhta hai — individual letters nahi. Character-level operations uske liye indirect inference hain. **Fix:** aise kaam code se karao, model se nahi.
</details>

**Q8 Prefill aur decode mein farak?**
<details><summary>Answer</summary>
**Prefill:** input parallel process, compute-bound, time ∝ input length, = TTFT.
**Decode:** token-by-token sequential, **memory-bandwidth-bound** (GPU math units idle), time ∝ output length.
**Isliye:** output tokens 4-5x mehnge hain, aur output length latency ka sabse bada lever hai.
</details>

**Q9 Raw pretrained model se sawaal poochne pe kya hota hai?**
<details><summary>Answer</summary>
Wo jawab nahi deta — wo **text continue** karta hai. "Bharat ki rajdhani kya hai?" pe wo aur sawaal likh sakta hai (kyunki internet pe aisa sawaal usually question-list ka part hota hai). Assistant banane ke liye SFT + preference tuning chahiye.
</details>

**Q10 ⭐ Fine-tuning kab karo aur kab nahi?**
<details><summary>Answer</summary>
**Karo:** consistent format/style, brand voice, chhote model ko narrow task pe match karana, long prompt internalize karke cost bachana.
**Mat karo:** naya factual knowledge daalna (→ RAG), changing information, "model smart ho jaaye", jab eval suite nahi hai.
**Order:** Prompting → Few-shot → RAG → Fine-tuning. LIMA insight: fine-tuning **behaviour** sikhata hai, knowledge nahi.
</details>

---

## Prompting & Reasoning (11-18)

**Q11 ⭐⭐ "CRITICAL: YOU MUST" 2026 mein kyun nuksaan karta hai?**
<details><summary>Answer</summary>
Purane models weakly instructions follow karte the, isliye emphasis chahiye tha. Aaj ke models system prompt ko bahut closely follow karte hain — wahi emphasis ab **over-triggering** karta hai (tool har baar call hoga chahe zaroorat na ho). Aur jab har instruction "CRITICAL" hai, koi bhi critical nahi hai. **Anxious prompt = anxious, hedging model.**
</details>

**Q12 Prohibition list ke 2 problems?**
<details><summary>Answer</summary>
1. **Anchoring** — X ka zikr karne se X ki probability badh sakti hai
2. Model ko coherent principle ki jagah ek maze navigate karna padta hai
**Fix:** positive framing — "ye karo" > "ye na karo".
</details>

**Q13 ⭐ Chain-of-Thought kaam kyun karta hai?**
<details><summary>Answer</summary>
**Generated tokens hi compute hain.** Har reasoning token ek naya forward pass hai jisme model previous reasoning ko attend kar sakta hai. Reasoning tokens = model ka scratchpad = extra compute. Ye prompt trick nahi, **computational** insight hai.
</details>

**Q14 ⭐ Test-time compute naya scaling axis kyun hai?**
<details><summary>Answer</summary>
Purana scaling: bada model, zyada data, zyada training compute (mehnga, ek baar ka).
Naya: **inference pe** zyada sochne do — same model, **per-request tunable**. Ek hi model easy aur hard dono handle kar sakta hai, aur improvement ke liye retrain nahi karna padta.
</details>

**Q15 ⭐ Reasoning models ke saath prompting ULTA kyun hai?**
<details><summary>Answer</summary>
- "Think step by step" → redundant (already sochta hai), latency badhata hai
- Step-by-step method dictate karna → over-constraining, uska plan usually better
- "Verify your answer" → already verify karta hai, over-verification hoti hai
- Verbose scaffolding → quality **girti** hai
**Sahi:** goal do, constraints do, success criteria do. Method uska.
</details>

**Q16 Reasoning model ke saath `max_tokens` khatarnak kyun?**
<details><summary>Answer</summary>
`max_tokens` thinking + answer **dono** cover karta hai. Agar 1000 rakha aur thinking 900 le gaya, answer ke liye 100 bache → truncated output. Generous rakho (16K-64K high effort pe) aur `stop_reason` check karo.
</details>

**Q17 Structured output "guarantee" kaise deta hai?**
<details><summary>Answer</summary>
**Constrained decoding.** Grammar (schema) har step pe batata hai ki kaunse tokens valid hain; baaki sab ki probability force se zero. Isliye output schema-invalid ho hi nahi sakta. Ye prompt engineering nahi, decoding-level enforcement hai.
</details>

**Q18 Structured output ke 2 gotchas?**
<details><summary>Answer</summary>
1. **Refusal** — safety refuse kare to output schema follow nahi karega
2. **Truncation** — `max_tokens` hit ho gaya to JSON adhoora, invalid
Dono ke liye `stop_reason` check karo.
</details>

---

## RAG (19-30)

**Q19 ⭐⭐ RAG debug kaise karoge systematically?**
<details><summary>Answer</summary>
**Retrieval aur generation ALAG measure karo.**
```
Recall@5 low, Gen accuracy high (perfect context ke saath) → RETRIEVAL problem
Recall@5 high, Gen accuracy low                            → GENERATION problem
Dono low                                                    → pehle retrieval fix karo
```
80% failures retrieval mein hote hain, par log 80% time prompt pe kharch karte hain.
</details>

**Q20 ⭐⭐ Recall@20 = 0.95, Recall@5 = 0.60. Diagnosis aur fix?**
<details><summary>Answer</summary>
**Ranking problem, retrieval problem nahi.** Sahi chunk 95% baar retrieve ho raha hai (top-20 mein hai), par 35% baar rank 6-20 pe hai.
**Fix: Reranking.** top-50 retrieve → cross-encoder rerank → top-5 pass. Expected 0.60 → ~0.88.
Chunking/embedding badalna time waste hoga — wo retrieval fixes hain, aur retrieval theek hai.
</details>

**Q21 ⭐ Hybrid search kyun chahiye? Ek example do.**
<details><summary>Answer</summary>
Vector search **exact identifiers** miss karta hai — `E4471` ka embedding space mein koi meaning nahi. Query "error code E4471 ka fix" pe vector search "error handling best practices" laayega. BM25 exact match pakad leta hai.
Aur embeddings **negation-blind** hain ("baarish hui" vs "baarish nahi hui" ≈ 0.95 similarity).
Dono **complementary** hain — isliye hybrid.
</details>

**Q22 ⭐ RRF kya hai aur score normalization se better kyun?**
<details><summary>Answer</summary>
Reciprocal Rank Fusion: `score = Σ 1/(k + rank)`, `k=60`. **Scores bhool jao, sirf ranks use karo.**
BM25 (0-30) aur cosine (0-1) ko normalize karna fragile hai. RRF scale-free hai, tuning almost zero, aur jo docs dono methods mein aate hain wo naturally top pe aa jaate hain.
</details>

**Q23 ⭐ Bi-encoder aur cross-encoder mein farak?**
<details><summary>Answer</summary>
**Bi-encoder:** query aur doc alag encode, phir cosine. Docs pre-compute ho sakte hain → fast. Par query aur doc ek dusre ko "dekh" nahi sakte.
**Cross-encoder:** query+doc **ek saath** process, tokens ek dusre pe attend karte hain → bahut accurate, par offline precompute impossible → slow.
**Isliye 2-stage:** bi-encoder se 50 candidates (recall), cross-encoder se top-5 (precision).
</details>

**Q24 Cross-encoder "aboutness vs answerhood" gap kaise fix karta hai?**
<details><summary>Answer</summary>
Query "2024 revenue kitna tha?" pe:
- "Revenue growth strategies..." → bi-encoder 0.82 (topic match), cross-encoder 0.11 (jawab nahi deta)
- "FY2024: ₹450 crore" → bi-encoder 0.71 (chhota chunk), cross-encoder 0.96 ✅
Embeddings "aboutness" measure karti hain, cross-encoder "answerhood".
</details>

**Q25 Parent-document retrieval kaise trade-off todta hai?**
<details><summary>Answer</summary>
Chhote chunks = precise search par context kat jaata hai. Bade chunks = poora context par embedding dhundhli.
**Solution:** child chunks (400 tokens) **embed** karo, parent chunks (2000 tokens) **return** karo. Precise search + full context, dono.
</details>

**Q26 ⭐ Chunk enrichment kya hai aur kaam kyun karta hai?**
<details><summary>Answer</summary>
Chunk ke aage header path prepend karo:
```
❌ "Ye 26 hafte ki hoti hai..."           → "ye" kya? standalone meaningless
✅ "[HR Policy | Leave > Parental Leave]
    Ye 26 hafte ki hoti hai..."           → ab query se match karega
```
Chhota change, bada asar (teams often 10-15 points report karte hain).
</details>

**Q27 ⭐⭐ ACL prompt mein kyun nahi ho sakti?**
<details><summary>Answer</summary>
"Confidential documents mention na karo" ek **instruction** hai — prompt injection se bypass ho sakti hai. Aur model ne wo data **dekh liya** hai, to leak ka risk hamesha rahega.
**Sahi:** retrieval mein pre-filter — `WHERE access_level IN (user.roles)`. Model ne data dekha hi nahi → leak **impossible**.
Ye enterprise RAG ka #1 audit failure hai.
</details>

**Q28 ⭐ Aggregation query RAG se kyun nahi ho sakti?**
<details><summary>Answer</summary>
"Kitne active contracts hain?" — vector search 5 chunks laayega jinme contract ka zikr hai. Model un 5 se count karke **confidently galat** number dega.
**Fix:** router. Query classify karo → aggregation → SQL, realtime → API, lookup → RAG, summary → full context.
"Sab kuch RAG" ek architecture mistake hai.
</details>

**Q29 RAG prompt mein "nahi mila" escape hatch kyun critical hai?**
<details><summary>Answer</summary>
Bina explicit permission ke model **hamesha** kuch generate karega — uska training objective "text produce karna" hai, "abstain karna" nahi. Escape hatch ke bina hallucination guaranteed hai. **"Mujhe nahi pata" ek galat answer se better hai**, especially enterprise mein.
</details>

**Q30 Citation validation hallucination canary kaise banta hai?**
<details><summary>Answer</summary>
```python
cited = set(re.findall(r'\[Doc (\d+)\]', answer))
if cited - set(range(1, len(chunks)+1)):  # Doc 7 jab sirf 5 the
    → model confabulate kar raha hai
```
Ye automatic detection hai — tumhe manually padhna nahi padta.
</details>

---

## Agents (31-40)

**Q31 ⭐⭐ Agent aur workflow mein exact farak?**
<details><summary>Answer</summary>
**Workflow:** *tum* decide karte ho ki kya karna hai (steps code mein hardcoded).
**Agent:** *model* decide karta hai (kaunsa tool, kitni baar, kab rukna).
Baaki sab isse follow hota hai — determinism, cost predictability, debuggability.
</details>

**Q32 ⭐ Agent banane se pehle kaunse 4 sawaal?**
<details><summary>Answer</summary>
1. **Complexity** — steps pehle se unspecifiable hain?
2. **Value** — outcome extra cost/latency justify karta hai?
3. **Viability** — model is task type mein capable hai?
4. **Error cost** — galti pakdi aur recover ki jaa sakti hai?
Koi ek "nahi" = agent na banao.
</details>

**Q33 ⭐⭐ "Harness" kya hai? Model commodity kyun hai?**
<details><summary>Answer</summary>
Harness = tumhara code jo model ke around hai: loop control, tool execution+validation, context management, error handling, permissions, limits, observability, state.
**Model sab products mein same hai** — API se aata hai. Do teams same model se completely different quality product bana sakti hain; farak harness mein hota hai. **Harness tumhara differentiator hai.**
</details>

**Q34 Tool execution ke 5 safety layers?**
<details><summary>Answer</summary>
1. **Allowlist** — unknown tool reject
2. **Schema validation** — invalid args pe actionable error
3. **Permission gate** — destructive ops pe approval
4. **Timeout** — hanging tool block na kare
5. **Error capture** — exception ko tool_result banao, crash nahi
</details>

**Q35 ⭐ Tool error pe exception raise karna kyun bura hai?**
<details><summary>Answer</summary>
Exception = poora agent crash. Error ko **model ko batao** (`is_error: true`, actionable message) — model **actually recover karta hai** (spelling fix karega, alag approach lega). Ye tool calling ki underrated superpower hai.
Error message actionable ho: "City not found; available: [...]" ≫ "Invalid input".
</details>

**Q36 ⭐ Tool output shaping kyun critical hai?**
<details><summary>Answer</summary>
Ek badly-designed tool poora context window ek call mein bhar sakta hai:
```
❌ read_file("app.log") → 500,000 tokens
✅ read_file("app.log", filter="ERROR", tail=100) → 2,000 tokens
   + note: "1.2M lines total, filter change kar sakte ho"
```
Ye context rot ka #1 preventive fix hai.
</details>

**Q37 Context rot kya hai? 4 fixes?**
<details><summary>Answer</summary>
Long agent runs mein context degrade hota hai — purane tool results, stale exploration. Symptoms: repeat, failed approach dobara, original task bhool jaana, loop.
**Fixes:** (1) tool result pruning, (2) compaction (failed approaches **preserve** karke!), (3) state externalize (file mein), (4) fresh context restart.
</details>

**Q38 ⭐ Multi-agent ka asli justification?**
<details><summary>Answer</summary>
**Context isolation** — "150K tokens padho, 2K wapas lao". Har sub-agent ka apna context window; main agent ka clean rehta hai. Plus parallelism.
"Specialization" weak argument hai — same model hai, sirf alag system prompt. Ek agent ko dono instructions do, same result, kam complexity.
</details>

**Q39 Coding agents sabse successful category kyun hain?**
<details><summary>Answer</summary>
**Verification sasti hai** — tests, compiler, linter. Agent code likhta hai → tests chalata hai → fail → khud fix karta hai. Ye feedback loop free hai.
Jahan verification mehngi/impossible hai (creative writing, strategic advice), wahan agents weakly perform karte hain.
**Design question:** "Agent apna kaam khud verify kaise karega?" — agar answer nahi hai, agent shayad galat tool hai.
</details>

**Q40 "Hallucinated progress" sabse khatarnak failure kyun?**
<details><summary>Answer</summary>
Wo **silently** fail hota hai — agent confidently bolega "maine 5 files update kar di" jab ki nahi ki. Koi error nahi, koi alert nahi.
**Fix (prompt mein):** "Progress report karne se pehle, har claim ko is session ke tool result se verify karo. Jo verified nahi hai, uske liye explicitly bolo."
</details>

---

## Frameworks & Production (41-50)

**Q41 ⭐ "Kaunsa AI framework use karun" galat sawaal kyun hai?**
<details><summary>Answer</summary>
"AI framework" ek category nahi hai — 8 alag layers hain (model access, RAG, orchestration, storage, eval, obs, UI), har ek ka apna tooling. Sahi sawaal: **"har layer pe kya use karun?"**
</details>

**Q42 ⭐ Framework choose karne ka rule?**
<details><summary>Answer</summary>
**"Kya main ye 200 lines mein likh sakta hoon?"** HAAN → likho.
Framework lo jab: 20+ integrations chahiye, durable execution chahiye, team ko shared vocabulary chahiye.
**Aur:** pehle bina framework likho, phir framework tab lo jab pata ho ki wo **kaunsa specific problem** solve kar raha hai.
</details>

**Q43 LangGraph ka strongest justification?**
<details><summary>Answer</summary>
**Checkpointing / durable execution.** Long-running agents crash hote hain; checkpointing ke bina poora kaam gaya. State serialization + resume khud likhna 2-3 hafte aur bugs.
Plus: human-in-the-loop interrupts with state editing, complex conditional branching, per-node streaming.
Agar in me se kuch nahi chahiye → custom loop simpler hai.
</details>

**Q44 ⭐ 4 agent approaches kya hain (harness/deployment axes)?**
<details><summary>Answer</summary>
| | Harness | Deployment |
|---|---|---|
| Manual loop | Tum | Tum |
| Tool runner | SDK (loop only) | Tum |
| Coding agent SDK | SDK (+ built-in tools) | Tum |
| Managed platform | Provider | **Provider** |
Sabse common confusion: tool runner ≠ coding agent SDK ≠ managed platform.
</details>

**Q45 ⭐ pgvector default kyun hona chahiye?**
<details><summary>Answer</summary>
Ek DB (data + vectors + full-text), ACID transactions, joins with business data, SQL metadata filtering (pre-filter!), mature ops/backup, sasta, aur team already jaanti hai.
10M vectors tak comfortable. Zyadatar products wahan pahunchte hi nahi.
Naya system lena = naya deployment, monitoring, sync problem, failure mode.
</details>

**Q46 ⭐⭐ Cost ke top 3 levers?**
<details><summary>Answer</summary>
1. **Model routing** (60-80%) — chhota model jahan chalega, cascade pattern
2. **Prompt caching** (50-90% input) — stable prefix, cache hit rate monitor karo
3. **Output length** (30-50%) — cost aur latency dono fix
Ye teen milkar aksar 80%+ cut kar dete hain. Infra optimize karne se pehle ye karo.
</details>

**Q47 ⭐ Bill 3x ho gaya. Pehla diagnostic step?**
<details><summary>Answer</summary>
**Cache hit rate check karo.** Sabse common cause (40%) ek cache invalidator hai — koi ne prompt prefix mein kuch dynamic daal diya (timestamp, per-user content, tool order). Koi error nahi aata, silently full price.
Phir: model distribution (routing toota?), token breakdown (input/output kya badha?), top expensive requests (agent loop?).
</details>

**Q48 ⭐⭐ AI mein traditional tests kyun kaam nahi karte?**
<details><summary>Answer</summary>
1. **Non-determinism** — same input, alag output
2. **No single correct answer** — 10 summaries sab sahi ho sakti hain
3. **Quality graded hai, binary nahi**
**Shift:** `assert output == expected` → `assert quality_metric > threshold`. Aur cost/steps bhi assert karo.
</details>

**Q49 ⭐ LLM judge ke 5 rules? Kaunsa sabse skip hota hai?**
<details><summary>Answer</summary>
1. Strong model use karo
2. Rubric do, "acha hai kya?" nahi
3. **Judge ko VALIDATE karo** (50 cases khud grade, agreement <80% = rubric fix) ⭐
4. Position bias randomize karo
5. `temperature=0`
**Rule 3 sabse skip hota hai** — bina validation ke tum ek unmeasured metric measure kar rahe ho aur uske hisaab se optimize kar rahe ho.
</details>

**Q50 ⭐⭐ Prompt injection ko SQL injection kehna kyun sahi hai?**
<details><summary>Answer</summary>
Dono mein **data aur instructions ek hi channel mein** hain. Model architecturally distinguish nahi kar sakta — system prompt ki "authority" seekhi hui hai, enforced nahi.
Aur SQL injection ka fix "better query likhna" nahi tha — **parameterized queries** (architecture) tha. Yahan bhi:
- **Privilege separation** (reader/actor, structured boundary)
- **Least privilege tools** (khatarnak tool exist hi na kare)
- **Approval gates**
- **Egress control**
Prompt-level defence (delimiters) ek layer hai, solution nahi.
</details>

---

## Bonus: behavioural questions

**"Ek AI project batao jo fail hua aur kyun."**
> Structure: kya banaya → kya assume kiya → kya galat nikla → kaise pata chala (metric!) → kya seekha.
> Strong answer mein **measurement** hoga: "humein pata chala kyunki Recall@5 0.6 tha" ≫ "users complain kar rahe the".

**"Tum AI ke saath kaise up-to-date rehte ho?"**
> Papers + provider changelogs + apne evals pe naye models test karna. "Twitter padhta hoon" weak answer hai; "jab naya model aata hai to apne eval suite pe chalata hoon aur measure karta hoon" strong hai.

**"AI product mein sabse bada risk kya hai?"**
> Do achhe answers: (1) **Bina eval ke ship karna** — tum blind ho aur regression invisible hai. (2) **Security** — prompt injection + over-privileged tools, kyunki blast radius bada hai aur fix architecture-level hai.

---

**Next:** [`system-design-cases.md`](system-design-cases.md)
