# Scoring System

This document defines the scoring used in every bug report under
`<provider>/`. The system is designed to be objective, reproducible,
and easy to read at a glance.

---

## At a glance

**Higher is better.** Grade **A** is the best score, **F** is the worst.

| Grade | Score range | Meaning                       |
|:-----:|------------:|-------------------------------|
| **A** | 90 – 100    | Excellent, production ready   |
| **B** | 80 – 89     | Solid, only minor issues      |
| **C** | 65 – 79     | Functional but real gaps      |
| **D** | 50 – 64     | Degraded, blocks common usage |
| **F** |  0 – 49     | Not production ready          |

Each test category gets its own grade. The overall grade is the
average across all categories, rounded to the nearest letter.

---

## 1. Bug Severity (1 to 10)

Every bug gets a single severity number from **1** (cosmetic) to
**10** (provider unusable). Use this scale:

| Score | Label    | Traffic light | Meaning                                                  |
|:-----:|----------|:-------------:|----------------------------------------------------------|
| 9–10  | Critical | red           | A model or core feature is completely unusable           |
| 7–8   | High     | red           | A common operation fails or returns wrong data ≥ 30 %    |
| 4–6   | Medium   | yellow        | Edge case fails, or works but is inconsistent            |
| 1–3   | Low      | green         | Cosmetic, documentation, or minor behaviour quirks       |

Pick the lowest score that still describes the worst symptom. When
in doubt, round **up** (so the bug is not under-prioritised).

---

## 2. Category Score (0 to 100)

Test results are grouped into seven categories. Each category has a
score from **0** (everything fails) to **100** (everything passes).

| Category               | What it tests                                                         |
|------------------------|------------------------------------------------------------------------|
| Functional Suite       | basic_text, system_prompt, multi_turn, tool_use, tool_roundtrip, image_vision, list_models |
| Authentication         | Bearer / x-api-key on /v1/messages and /v1/chat/completions           |
| Model Authenticity     | msg_* IDs, response.model match, thinking-signature roundtrip, tokenizer fingerprint |
| Prefill / Continuation | baseline_user, prefill_simple, prefill_agent_loop                     |
| Thinking / Reasoning   | thinking field absent, disabled, enabled                               |
| Streaming              | chars > 0, ttft present, tps reasonable                                |
| Web Search             | server tool actually invoked                                           |

Score formula per category:

```
score = round( passed_probes / total_probes * 100 )
```

When a model is unusable for unrelated reasons (see Critical bugs),
its probes still count as failed in every category. We do not
"exclude" dead models, because the customer cannot exclude them either.

---

## 3. Bug Status Labels

Used in the bug summary table:

| Label   | Traffic light | Meaning                                                 |
|---------|:-------------:|----------------------------------------------------------|
| Open    | red           | Confirmed in this report's test run, not fixed           |
| Partial | yellow        | Some affected models / paths fixed, others still failing |
| Fixed   | green         | Verified resolved in this report's test run vs. previous |

---

## 4. Trend Indicators

Compare the new score to the previous report:

```
Δ +N    score went up by N points (improvement)
Δ −N    score went down by N points (regression)
Δ  0    no change
new     this category was not measured before
```

---

## 5. Customer Impact

Every bug report includes a **Customer Impact** section. It explains,
in plain English, how the bug affects the *paying customer* of the
proxy — not the proxy's own engineers. Examples:

- "Users cannot stream Claude responses, so your chat UI looks broken
  even though the API reports success."
- "Agents using tool calls will silently lose every tool call, so
  any product built on agentic flows breaks today."

The section answers three questions:

1. What does the customer **experience**?
2. Which **products / use cases** does this break?
3. Why is fixing it **urgent** (or not)?

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.

Contact: https://winfuture.de · https://t.me/wf_sebastian · sk@winfuture.de
