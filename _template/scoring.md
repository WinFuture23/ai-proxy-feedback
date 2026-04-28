# Scoring System

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.

This document defines the scoring used in every bug report under
`public/<provider>/`. The system is designed to be objective,
reproducible, and easy to read at a glance.

---

## 1. Bug Severity (1 to 10)

Every bug gets a single severity number from **1** (cosmetic) to
**10** (provider unusable). Use this scale:

| Score | Label    | Meaning                                                              |
|-------|----------|----------------------------------------------------------------------|
| 9–10  | Critical | A model or core feature is completely unusable                       |
| 7–8   | High     | A common operation fails or returns wrong data ≥ 30% of the time     |
| 4–6   | Medium   | Edge case fails, or works but is inconsistent across endpoints       |
| 1–3   | Low      | Cosmetic, documentation, or minor behaviour quirks                   |

Pick the lowest score that still describes the worst symptom. When in
doubt, round **up** (so the bug is not under-prioritised).

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

## 3. Overall Grade (letter)

```
A   90 – 100   excellent, production ready
B   80 –  89   solid, minor issues
C   65 –  79   functional but real gaps
D   50 –  64   degraded, blocks common use cases
F    0 –  49   not production ready
```

Overall score = unweighted mean of all category scores.

---

## 4. Bug Status Labels

Used in the bug summary table:

| Label   | Meaning                                                          |
|---------|-------------------------------------------------------------------|
| Open    | Confirmed in this report's test run, not fixed                   |
| Partial | Some affected models / paths fixed, others still failing          |
| Fixed   | Verified resolved in this report's test run vs. previous report   |

---

## 5. Trend Indicators

Compare the new score to the previous report:

```
Δ +N    score went up by N points (improvement)
Δ −N    score went down by N points (regression)
Δ  0    no change
new     this category was not measured before
```
