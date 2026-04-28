# [PROVIDER] API Test Report — YYYY-MM-DD

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | [Provider name]                                                  |
| Endpoint         | https://example.com                                              |
| Test date (UTC)  | YYYY-MM-DDTHHMMZ                                                  |
| Models tested    | model-1, model-2                                                  |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 1.0                                                              |

---

## How to read this report

1. **Quick Status** — score per area and overall grade. Higher is better.
2. **Backend Authenticity** — short verdict on whether the model behind
   the API is the model the provider claims (e.g. real Anthropic vs.
   re-routed OpenAI / Gemini / open-source).
3. **Bug Summary** — one row per problem, sorted by severity, with a
   traffic-light indicator (red / yellow / green).
4. **Each bug** has six fixed sections:
   - What is wrong (plain English)
   - Customer impact (why this matters for paying users)
   - How to reproduce (copy and paste)
   - Likely cause (short engineering note)
   - How to fix (concrete steps)
   - How to verify the fix (copy and paste, with pass criterion)

The reproduction commands assume you ran the **Setup** block once
(see end of report).

---

## Quick Status

Grade scale: **A** = best (90–100), **F** = worst (0–49). Full rubric
in [`../_template/scoring.md`](../../_template/scoring.md).

| Category               | Score (0–100) | Grade | Trend             | Status                                |
|------------------------|--------------:|:-----:|:-----------------:|----------------------------------------|
| Functional Suite       | XX             | X     | Δ ±N              | one-line summary                       |
| Authentication         | XX             | X     | Δ ±N              | one-line summary                       |
| Model Authenticity     | XX             | X     | Δ ±N              | one-line summary                       |
| Prefill / Continuation | XX             | X     | Δ ±N              | one-line summary                       |
| Thinking / Reasoning   | XX             | X     | Δ ±N              | one-line summary                       |
| Streaming              | XX             | X     | Δ ±N              | one-line summary                       |
| Web Search             | XX             | X     | Δ ±N              | one-line summary                       |
| **Overall**            | **XX**         | **X** | **Δ ±N**          | **one-line summary**                   |

---

## Backend Authenticity

| Question                                | Verdict / Evidence                |
|-----------------------------------------|-----------------------------------|
| Does response shape match Anthropic?    | yes / no — see `msg_*` ID prefix  |
| Does signed thinking-block roundtrip?   | yes / no                          |
| Does tokenizer match Anthropic ranges?  | yes / no                          |
| Best guess for actual backend           | Anthropic / OpenAI / Gemini / ?   |

If any indicator says **no**, the proxy may be re-routing requests to
a different vendor than advertised. Treat with caution.

---

## Resolved Since Last Report

(Optional — list bugs marked Open in the previous report that pass now.)

| Previous Bug ID | Title | Status |
|-----------------|-------|--------|
| BUG-XXX         | …     | green Fixed |

---

## Bug Summary

| Bug ID  | Title                           | Severity | Light  | Affected         | Status |
|---------|---------------------------------|:--------:|:------:|------------------|--------|
| BUG-001 | …                               | 10 Critical | red | model / endpoint | Open   |

---

## BUG-001 — [Short title]

**Severity:** N/10 — [Critical / High / Medium / Low]
**Traffic light:** red / yellow / green
**Affected:** [model or endpoint]
**Status:** Open

### 1. What is wrong?

Two to three short sentences in plain English. State the symptom
the user sees, not the internal reason.

### 2. Customer impact

Three answers:

- **What does the customer experience?** …
- **Which products / use cases break?** …
- **Why is fixing it urgent (or not)?** …

### 3. How to reproduce

Copy and paste the **Setup** block once (end of report), then paste
this command:

```bash
curl -sS ...
```

| Result    | Value                                  |
|-----------|----------------------------------------|
| Expected  | `…` — what should come back            |
| Actual    | `…` — what actually comes back         |
| Latency   | …                                      |
| HTTP code | …                                      |

### 4. Likely cause

One short paragraph. Use simple terms. Mention the most likely
component (router, auth middleware, upstream mapping, …).

### 5. How to fix

Step by step. Aim for 3–6 numbered steps. If there are options,
label them **Option A** / **Option B** and say which is preferred.
Avoid fixes that ask the system to misrepresent identity, capabilities,
or training data — solve the underlying issue instead.

1. …
2. …
3. …

### 6. How to verify the fix

Paste this command. The pass criterion is in the table below.

```bash
curl -sS ...
```

| Pass Criterion          | Expected Value |
|-------------------------|----------------|
| HTTP status             | 200            |
| `…`                     | …              |

If the pass criterion is met, this bug is resolved.

---

## Setup (run once)

Replace `<your-key>` with a valid API key. Keep this terminal open
and reuse for every reproduction in this report.

```bash
export PROVIDER_API_KEY="<your-key>"
export BASE_URL="https://example.com"
```

Verify the setup:

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $PROVIDER_API_KEY" \
  | jq -r '.data | length'
# Pass: prints a positive integer
```

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.

Contact:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
