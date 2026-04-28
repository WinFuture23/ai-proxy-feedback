# g0i.shop API Test Report — 2026-04-28 (12:10 UTC follow-up)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | g0i                                                              |
| Endpoint         | https://g0i.shop                                                 |
| Test date (UTC)  | 2026-04-28T1210Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-6, claude-haiku-4-5-20251001, claude-sonnet-4.5 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 2.0                                                              |
| Previous report  | [`g0i/2026-04-28T0517Z/`](../2026-04-28T0517Z/report.md)          |

---

## TL;DR vs. previous run

After g0i deployed fixes between 05:17 UTC and 12:10 UTC, **3 of 7 bugs
are resolved or substantially improved**, **1 bug had a partial
regression** (Haiku thinking), and **1 new regression appeared** (Opus
4.7 simple prefill went from OK to empty). Scores moved from
overall **C (75)** to overall **C+ (78)**.

---

## How to read this report

This report is a follow-up to the 2026-04-28T0517Z run. It uses the
same template structure but adds a **Changes Since Last Report**
section that shows, per bug, whether the issue is now Fixed, Partially
Fixed, Open, or Regressed.

Grade scale: **A** = best (90–100), **F** = worst (0–49). Full rubric
in [`../../_template/scoring.md`](../../_template/scoring.md).

---

## Quick Status

| Category               | Score | Grade | Δ vs 0517Z | Status                                |
|------------------------|------:|:-----:|:----------:|----------------------------------------|
| Functional Suite       | 86    | B     | **+17**    | green — 31/36 PASS; opus-4-6 alive     |
| Authentication         | 86    | B     | 0          | yellow — x-api-key still missing on /chat/completions (not retested) |
| Model Authenticity     | 100   | A     | **+25**    | green — opus-4-6 now testable, all 4 models verified |
| Prefill / Continuation | 83    | B     | **+16**    | green — 10/12 OK (was 8/12)             |
| Thinking / Reasoning   | 25    | F     | **−25**    | red — Haiku regressed from FREE to BROKEN |
| Streaming              | 75    | C     | 0          | yellow — works on 3/4 (opus-4-6 still broken) |
| Web Search             | 75    | C     | **−25**    | yellow — opus-4-6 still empty           |
| Hidden Context (new)   | 100   | A     | new        | green — STABLE on all 4 models, ≈ message length |
| **Overall**            | **78**| **C+**| **+3**     | yellow — improved but new regressions   |

---

## Backend Authenticity (updated)

The previous run could not probe `claude-opus-4-6` because the route
returned empty content. This run **could** probe it, and it passes
all four authenticity checks. Three of three other models also pass.

| Question                                         | Result                                          |
|--------------------------------------------------|-------------------------------------------------|
| Does the response ID start with `msg_`?          | yes (all 4 models)                              |
| Does `response.model` match the requested model? | yes (all 4 models)                              |
| Does a signed `thinking` block roundtrip?        | **yes** (all 4 models, including opus-4-6 now)  |
| Tokenizer fingerprint within Anthropic range?    | yes (34–52 tok / 100 chars)                     |

**Verdict: real Anthropic Claude on all four routes.** The
thinking-signature roundtrip succeeds across the board — only the
genuine Anthropic backend can validate signatures it produced.
Confidence is now **higher** than in the previous report because
opus-4-6 is no longer skipped.

---

## Hidden Context — new probe, all 4 models STABLE

A new probe (`tests/test_hidden_context.py`) sends each of three
trivial prompts five times and reports the distribution of
`usage.input_tokens`. The intent is to detect any hidden system
prompt or unstable usage reporting.

| Model                      | Verdict | Median input_tokens (range across 3 prompts × 5 samples) |
|----------------------------|---------|----------------------------------------------------------|
| claude-opus-4-7            | STABLE  | 18–22 (range 0 within each prompt)                       |
| claude-opus-4-6            | STABLE  | 11–18                                                    |
| claude-haiku-4-5-20251001  | STABLE  | 11–18                                                    |
| claude-sonnet-4.5          | STABLE  | 11–18                                                    |

g0i is clean on this probe: no inflated token counts, no instability
between calls of the same prompt. (For comparison: LightningZeus
showed input_tokens of 5997–6000 for the same probe.)

---

## Changes Since 2026-04-28T0517Z

<!-- diff-table -->

| Previous Bug | Title (short)                                              | Severity then | Status now      | Notes                                                                 |
|--------------|------------------------------------------------------------|--------------|-----------------|-----------------------------------------------------------------------|
| BUG-001      | claude-opus-4-6 returns empty content for every request    | 10 Critical  | **Mostly Fixed** | 5/9 functional tests pass now (was 0/9). Baseline + agent-loop prefill OK. signature_roundtrip ACCEPTED. 4 sub-issues remain — see new BUG-A. |
| BUG-002      | `thinking` parameter ignored on Opus routes                | 9 Critical   | **Open + new regression** | Opus 4.7 + 4.6 still BROKEN. Haiku 4.5 **regressed** from FREE to BROKEN. Sonnet 4.5 still works.   |
| BUG-003      | claude-haiku-4-5 returns empty on plain assistant prefill  | 7 High       | **Fixed**        | All 3 prefill probes on Haiku now return real continuation text.       |
| BUG-004      | /v1/chat/completions rejects x-api-key                     | 5 Medium     | Not retested    | Auth-diag part now uses opus-4-6 model probe; 403 now is route-related, masking the auth check. Will fix the test. |
| BUG-005      | /v1/models listing inconsistencies                         | 4 Medium     | **Partially Fixed** | Catalog grew from 68 → 71 models. Haiku 4.5 now advertised. Opus 4.6 still missing. |
| BUG-006      | Latency outlier on opus-4-7 simple prefill (~20 s)         | 3 Low        | **Different mode** | The 20 s spike is gone. But prefill_simple now returns EMPTY in 0.30 s — see new BUG-B. |
| BUG-007      | Model self-identification confused                         | 2 Low        | Open            | Sonnet 4.5 still says "Claude 3.7 Sonnet (Haiku)" when asked.          |

### New bugs from this run

| Bug ID | Title                                                                  | Severity | Light  | Status |
|--------|------------------------------------------------------------------------|:--------:|:------:|--------|
| BUG-A  | claude-opus-4-6 still has 4 functional gaps (system_prompt, streaming, web_search, list_models) | 6 Medium | yellow | Open   |
| BUG-B  | claude-opus-4-7 prefill_simple now returns empty content (was OK in 0517Z) | 7 High | red    | Open / Regression |
| BUG-C  | claude-haiku-4-5 thinking-enabled regressed (FREE → BROKEN)            | 7 High   | red    | Open / Regression |

---

## BUG-002 (still open) — now reproducible on three models

This is the bug g0i replied to with a quote from the Backend
Authenticity section of the previous report, which was not actually
addressing this issue. Below is a fresh, single-line verification:

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":2048,"thinking":{"type":"enabled","budget_tokens":1024},"messages":[{"role":"user","content":"What is 7*9? Think first."}]}' \
  | jq '[.content[].type]'
```

| Result    | Value                              |
|-----------|------------------------------------|
| Expected  | `["thinking", "text"]`             |
| Actual    | `["text"]`                         |
| HTTP      | 200                                |

Same probe with `"model": "claude-sonnet-4.5"` returns
`["thinking","text"]` correctly — confirming the upstream supports
the feature and the bug is route-specific to the Opus + Haiku-4.5
routes.

---

## BUG-A — claude-opus-4-6 still has 4 functional gaps

**Severity:** 6/10 — Medium
**Traffic light:** yellow
**Affected:** `claude-opus-4-6` on `POST /v1/messages`
**Status:** Open

### 1. What is wrong?

The model now responds (no longer empty content). But four of the
nine functional tests still fail on this route:

| Test          | Status | Detail                                                                  |
|---------------|--------|-------------------------------------------------------------------------|
| list_models   | FAIL   | Model not advertised in `/v1/models`                                    |
| system_prompt | FAIL   | Model refuses safe-but-secret-style system prompt: *"I'm sorry, but I can't help with that."* — looks like Anthropic-side guardrail, not a proxy bug |
| streaming_tps | FAIL   | `chars=0`, `ttft=n/a` — model produces 400 output_tokens but no `text_delta` events arrive |
| web_search    | FAIL   | `search_used=False`, empty reply                                         |

### 2. Customer impact

- system_prompt failure is likely Anthropic-side and outside your
  control.
- streaming_tps failure is real and breaks any chat UI on opus-4-6.
- web_search failure means the server tool is not wired up for this
  route.
- list_models failure means discoverability is incomplete.

### 3. Reproduction

Identical to the previous report's BUG-001 reproduction; the model
now returns content for trivial queries but the four tests above
still fail.

### 4. Likely cause

Possibly the same underlying upstream that returned content=[]
before now returns text but does not stream. Web_search is a
separate upstream/tool wiring issue. List_models is a catalog sync
issue (see also BUG-005 partial fix).

### 5. How to fix

1. Confirm the upstream for opus-4-6 supports streaming. If so,
   forward stream chunks unmodified.
2. Wire `web_search_20250305` for opus-4-6 the same way it is wired
   for the other three Claude routes (which all pass web_search).
3. Add opus-4-6 to `/v1/models` listing.

### 6. How to verify

```bash
# streaming
curl -sS -N -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","max_tokens":200,"stream":true,"messages":[{"role":"user","content":"Write 100 words on espresso."}]}' \
  | grep -c '"type":"text_delta"'
# Pass: > 10
# Fail: 0
```

---

## BUG-B — claude-opus-4-7 prefill_simple now returns empty

**Severity:** 7/10 — High
**Traffic light:** red
**Affected:** `claude-opus-4-7` on `POST /v1/messages` with text-only prefill
**Status:** Open / Regression

### 1. What is wrong?

In the 2026-04-28T0517Z run, the simple-prefill probe on
`claude-opus-4-7` returned a real continuation in ~19.8 s. In this
run, the same probe returns HTTP 200 with `content: []` in 0.30 s.

It looks like the slow-fallback path that previously produced a
correct (but slow) response was replaced with a fast-fail path that
returns empty content.

### 2. Customer impact

Prefill / continuation patterns no longer work on Opus 4.7. Agent
frameworks that depend on `assistant: "<prefix>"` to constrain
output now silently get empty replies.

### 3. Reproduction

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-opus-4-7","max_tokens":40,
    "messages":[
      {"role":"user","content":"Say hello."},
      {"role":"assistant","content":"Hello"}
    ]
  }' | jq '{content, latency_hint:"~0.30s"}'
```

| Result    | Value                              |
|-----------|------------------------------------|
| Expected  | `content[0].text` non-empty (continuation from "Hello") |
| Actual    | `content: []`                      |
| Latency   | ~0.30 s                            |

### 4. Likely cause

A code path was changed to short-circuit the request when it
detects the trailing-assistant pattern. In the 0517Z run the slow
Vertex retry produced output; now it appears that retry was removed
or replaced.

### 5. How to fix

Restore the working code path from the 0517Z run (which had real
continuations, just slow). Ideally route the request directly to
the upstream that was returning content correctly.

### 6. How to verify

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":40,"messages":[{"role":"user","content":"Say hello."},{"role":"assistant","content":"Hello"}]}' \
  | jq -r '.content[0].text // "FAIL: empty"'
# Pass: prints a non-empty continuation (e.g. "! How can I help...")
# Fail: prints "FAIL: empty"
```

---

## BUG-C — claude-haiku-4-5 thinking enabled regressed (FREE → BROKEN)

**Severity:** 7/10 — High
**Traffic light:** red
**Affected:** `claude-haiku-4-5-20251001`
**Status:** Open / Regression

### 1. What is wrong?

In the 0517Z run, Haiku correctly returned a `thinking` block when
`thinking={"type":"enabled","budget_tokens":1024}` was set. In this
run, the same request returns no thinking block — the model is
treated as if thinking were disabled. The probe also took **78 s**
to complete (vs ~2 s for the no-thinking variants), which suggests
the upstream did do reasoning work but the proxy stripped it from
the response.

Sonnet 4.5 still works correctly on this probe; Opus 4.7 + 4.6 are
both BROKEN as before (BUG-002).

### 2. Customer impact

Reasoning workloads on Haiku now silently lose the reasoning step
and produce only the final answer. Customers paying for thinking
get billed for the slow upstream call but cannot read the thinking
block.

### 3. Reproduction

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":2048,"thinking":{"type":"enabled","budget_tokens":1024},"messages":[{"role":"user","content":"What is 7*9?"}]}' \
  | jq '[.content[].type]'
```

| Result    | Value                              |
|-----------|------------------------------------|
| Expected  | `["thinking", "text"]`             |
| Actual    | `["text"]`                         |
| Latency   | ~78 s                              |

### 4. Likely cause

Looks like a response-shaping middleware that strips thinking
blocks. The latency suggests the upstream did the thinking work,
but the proxy did not forward the result.

### 5. How to fix

In your response post-processing, do not drop or rewrite content
blocks of `type: thinking` or `type: redacted_thinking`. Forward
them unchanged.

### 6. How to verify

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":2048,"thinking":{"type":"enabled","budget_tokens":1024},"messages":[{"role":"user","content":"Compute 173*217 step by step."}]}' \
  | jq '[.content[].type] | contains(["thinking"])'
# Pass: prints true
# Fail: prints false
```

---

## Setup (run once)

```bash
export G0I_API_KEY="<your-key>"
export BASE_URL="https://g0i.shop"
```

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $G0I_API_KEY" \
  | jq -r '.data | length'
# Pass: prints a positive integer (we measured 71)
```

---

## Contact

For follow-up traces, additional reproduction cases, or to confirm
a fix, please reach out to **Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `g0i/2026-04-28T1210Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
