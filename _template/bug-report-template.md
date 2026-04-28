# [PROVIDER] API Test Report — YYYY-MM-DD

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | [Provider name]                                                  |
| Endpoint         | https://example.com                                              |
| Test date (UTC)  | YYYY-MM-DD                                                       |
| Models tested    | model-1, model-2                                                  |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 1.0                                                              |

---

## How to read this report

1. **Quick Status** — see the score per area and the overall grade.
2. **Bug Summary** — one row per problem, sorted by severity.
3. **Each bug** in detail with five fixed sections:
   - **What is wrong** — plain English
   - **How to reproduce** — copy and paste into a terminal
   - **Likely cause** — short engineering explanation
   - **How to fix** — concrete steps
   - **How to verify the fix** — copy and paste, with a clear pass criterion

The reproduction commands assume you ran the **Setup** block once
(see end of report).

---

## Quick Status

| Category               | Score (0–100) | Grade | Trend  | Status              |
|------------------------|---------------:|:-----:|:------:|---------------------|
| Functional Suite       | XX             | X     | Δ ±N   | one-line summary    |
| Authentication         | XX             | X     | Δ ±N   | one-line summary    |
| Model Authenticity     | XX             | X     | Δ ±N   | one-line summary    |
| Prefill / Continuation | XX             | X     | Δ ±N   | one-line summary    |
| Thinking / Reasoning   | XX             | X     | Δ ±N   | one-line summary    |
| Streaming              | XX             | X     | Δ ±N   | one-line summary    |
| Web Search             | XX             | X     | Δ ±N   | one-line summary    |
| **Overall**            | **XX**         | **X** | Δ ±N   | one-line summary    |

Scoring rules: see [`_template/scoring.md`](../_template/scoring.md).

---

## Resolved Since Last Report

(Optional — list bugs marked Open in the previous report that pass now.)

| Previous Bug ID | Title | Status |
|-----------------|-------|--------|
| BUG-XXX         | …     | Fixed  |

---

## Bug Summary

| Bug ID  | Title                           | Severity (1–10) | Affected         | Status |
|---------|---------------------------------|:---------------:|------------------|--------|
| BUG-001 | …                               | 10 Critical     | model / endpoint | Open   |

---

## BUG-001 — [Short title]

**Severity:** N/10 — [Critical / High / Medium / Low]
**Affected:** [model or endpoint]
**Status:** Open

### 1. What is wrong?

Two to three short sentences in plain English. State the symptom
the user sees, not the internal reason.

### 2. How to reproduce

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

### 3. Likely cause

One short paragraph. Use simple terms. Mention the most likely
component (router, auth middleware, upstream mapping, …).

### 4. How to fix

Step by step. Aim for 3–6 numbered steps. If there are options,
label them **Option A** / **Option B** and say which is preferred.

1. …
2. …
3. …

### 5. How to verify the fix

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

## Contact

For follow-up traces, additional reproduction cases, or to confirm a
fix:

- **Sebastian Kuhbach** — https://winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
