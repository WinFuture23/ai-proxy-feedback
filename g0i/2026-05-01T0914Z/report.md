# g0i.shop API Test Report — 2026-05-01 (09:14 UTC)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | g0i                                                              |
| Endpoint         | https://g0i.shop                                                 |
| Test date (UTC)  | 2026-05-01T0914Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-6, claude-haiku-4-5-20251001, claude-sonnet-4.5 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 5.0                                                              |
| Previous report  | [`g0i/2026-04-29T0622Z/`](../2026-04-29T0622Z/report.md)          |

---

## TL;DR vs. previous run

The HTTP 502 storm from 04-29 is **gone** for Opus and Haiku. Three
of four routes (Opus 4.7, Opus 4.6, Haiku 4.5) work normally again.

**New problem:** `claude-sonnet-4.5` now returns
`HTTP 503 "overloaded_error: Context too large..."` on six of nine
functional tests, plus all three prefill probes. Sonnet was the
one route that was fully working two reports ago, and one of the
two routes still confirmable as real Anthropic via thinking
signature (the other being Haiku, which itself has a thinking-block
regression). So Sonnet now needs investigation.

The original BUG-002 (thinking parameter ignored on Opus + Haiku
routes) is **still untouched**.

Overall grade moves from D+ (62, dragged down by transient 502s)
to **B− (78)**.

---

## Quick Status

| Category               | Score | Grade | Δ vs 0429-0622Z | Status                                             |
|------------------------|------:|:-----:|:--------------:|-----------------------------------------------------|
| Functional Suite       | 78    | C+    | **+11**        | yellow — 28/36 PASS; Sonnet hit by 503 overloaded   |
| Authentication         | 86    | B     | 0              | yellow — x-api-key on /chat/completions still suspect |
| Model Authenticity     | 75    | C     | **−25**        | yellow — Sonnet not testable due to 503             |
| Prefill / Continuation | 58    | F     | 0              | red — Opus 4.7 + 4.6 prefill_simple now HTTP 400; Sonnet all 3 503 |
| Thinking / Reasoning   | 25    | F     | **+25**        | red → yellow — Sonnet 4.5 thinking back to FREE; Opus + Haiku still BROKEN |
| Streaming              | 100   | A     | **+25**        | green — all 4 streaming tests pass (Sonnet TPS still works) |
| Web Search             | 75    | C     | 0              | yellow — Sonnet web_search is 503; others pass      |
| Hidden Context         | 100   | A     | 0              | green — STABLE on all 4 routes                      |
| **Overall**            | **78**| **C+**| **+16**        | green — 502 storm cleared, but Sonnet new 503 issue |

---

## Backend Authenticity

| Question                                          | Result                                       |
|---------------------------------------------------|-----------------------------------------------|
| Does the response ID start with `msg_`?           | yes (Opus 4.7, Opus 4.6, Haiku 4.5)           |
| Does `response.model` match the requested model?  | yes (where the route responded)               |
| Does a signed `thinking` block roundtrip?         | **yes** on Opus 4.7, Opus 4.6, Haiku 4.5      |
| Sonnet 4.5 testable?                              | **no** (HTTP 503 on the authenticity probe)  |

Verdict: real Anthropic Claude on the three working routes.
Sonnet 4.5 cannot be re-validated this run; please re-test
when 503s clear.

---

## Hidden Context

Same as previous runs: STABLE on all four models. Median
input_tokens ≈ 11–22 across three short prompts × five samples
each. No proxy-side inflation, no inconsistent reporting.

---

## Bug status update

| Bug    | Title (short)                                            | Status now            | Notes                                                |
|--------|----------------------------------------------------------|-----------------------|-------------------------------------------------------|
| BUG-001 | claude-opus-4-6 returned empty content                  | Still Fixed           | opus-4-6 still 9/9 functional pass.                   |
| BUG-002 | thinking parameter ignored on Opus routes               | **Still Open**        | Opus 4.7 + 4.6 + Haiku 4.5 still BROKEN. Sonnet 4.5 still FREE (works). |
| BUG-003 | haiku-4-5 returned empty on plain prefill               | Still Fixed           | Haiku prefill all 3 OK.                                |
| BUG-005 | /v1/models listing inconsistencies                      | Still Fixed           | Catalog now lists 89 models (was 72/71/68 in earlier runs). |
| BUG-007 | Sonnet self-id confused                                 | Could not retest      | Sonnet calls failed with 503.                          |
| BUG-A   | claude-opus-4-6 4 sub-issues                            | Still Fixed           | All 4 sub-issues remain fixed.                         |
| BUG-B   | opus-4-7 prefill_simple returns empty                   | **Different mode**    | Now HTTP 400 instead of 200/empty. New error body to be examined. |
| BUG-C   | Haiku thinking regressed                                | Open (still BROKEN)   | thinking_enabled returns no thinking block.            |
| BUG-D   | opus-4-6 prefill all TIMEOUT                            | **Improved**          | baseline + agent_loop OK. prefill_simple now HTTP 400 (different error mode, no longer hangs). |
| BUG-E   | Haiku system_prompt regressed                           | **Open again**        | Haiku system_prompt FAIL — refused.                    |
| BUG-F   | HTTP 502 storm                                          | **Cleared**           | Confirmed transient. No 502s today.                    |

### New from this run

| Bug ID | Title                                                                  | Severity | Light  | Status |
|--------|------------------------------------------------------------------------|:--------:|:------:|--------|
| BUG-K  | claude-sonnet-4.5 returns HTTP 503 `overloaded_error: Context too large` on most calls | 8 High | red    | Open / new |

---

## BUG-K — Sonnet 4.5 returns HTTP 503 "Context too large"

**Severity:** 8/10 — High
**Traffic light:** red
**Affected:** `claude-sonnet-4.5`
**Status:** Open / new

### What is wrong?

Six of nine functional tests on `claude-sonnet-4.5` failed with:

```
HTTP 503: {"type":"error","error":{"type":"overloaded_error","message":"Context too large..."}}
```

The test prompts are short — the longest is `tool_roundtrip` at
~600 input tokens. "Context too large" suggests something on the
proxy or upstream side is amplifying the prompt before it reaches
the model. Hidden-context probe on Sonnet shows STABLE 11–13
tokens for short prompts, so the inflation is not constant — it
seems to fire on certain request shapes.

### Customer impact

Sonnet 4.5 is the most popular Claude variant. If 6/9 standard
calls fail with a confusing 503, agents pinned to Sonnet break
intermittently with no clear diagnostic.

### Reproduction

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4.5","max_tokens":64,"messages":[{"role":"user","content":"Reply with: PONG"}]}'
```

Returns: `503 overloaded_error: Context too large...`

But the *streaming* probe on Sonnet 4.5 works, and the
hidden-context probe also works. So the 503 only fires for
certain non-streaming code paths.

### Likely cause

The "Context too large" message is upstream Anthropic phrasing.
Most likely the proxy is inflating context for some Sonnet
requests (perhaps duplicating system prompts, or appending a
response template) and the upstream rejects.

### Fix

1. Compare what your proxy actually sends to Anthropic for a
   short Sonnet 4.5 prompt vs. for a streaming variant — the
   non-streaming path is the bug.
2. Strip any duplicated system prompt or response wrapper from
   the non-streaming Sonnet path.
3. Add a server-side cap on outgoing context size with an
   actionable error message instead of a confusing upstream 503.

### Verify

```bash
status=$(curl -sS -o /dev/null -w "%{http_code}" -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4.5","max_tokens":64,"messages":[{"role":"user","content":"Reply with: PONG"}]}')
echo "$status"
# Pass: 200
# Fail: 503
```

---

## Setup (run once)

```bash
export G0I_API_KEY="<your-key>"
export BASE_URL="https://g0i.shop"
```

---

## Contact

For follow-up traces or to confirm a fix, please reach out to
**Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `g0i/2026-05-01T0914Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
