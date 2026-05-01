# Codemax API Test Report — 2026-05-01 (09:51 UTC)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | Codemax                                                          |
| Endpoint         | https://api.codemax.pro                                          |
| Test date (UTC)  | 2026-05-01T0951Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-7-thinking, claude-sonnet-4-6, claude-haiku-4-5-20251001 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 3.0                                                              |
| Previous report  | [`codemax/2026-04-29T0635Z/`](../2026-04-29T0635Z/report.md)      |

---

## TL;DR vs. previous run

**Real fixes shipped.** The two top bugs from the previous report
are resolved:

- **BUG-001 (streaming returns no text deltas):** FIXED.
  All four models now stream `text_delta` events normally.
  Throughput in the 28–50 tok/s range.
- **BUG-002 (`web_search_20250305` emits as plain text):** FIXED
  in shape — the `[TOOL_CALL]\n{tool => …}` text dump is gone.
  However, the tool still does not actually invoke
  (`search_used = False` on all four models). Different failure
  mode, partial fix.
- **BUG-003 (basic_text truncated to first character):** FIXED
  on `claude-opus-4-7-thinking`, `claude-sonnet-4-6`,
  `claude-haiku-4-5-20251001`. Still failing on
  `claude-opus-4-7` — but now with **HTTP 529 "overloaded_error"**
  from upstream Anthropic (74-second waits). Looks like
  upstream rate-limit / overload, not the original truncation
  bug.
- **BUG-004 (image_vision empty on Sonnet 4-6):** FIXED.
  Sonnet 4-6 image_vision now PASS.

**Still open:**

- **BUG-002 web_search:** model no longer dumps `[TOOL_CALL]`
  text but still does not invoke the tool. `search_used=False`
  on all four routes.
- **BUG-005:** `claude-opus-4-7-thinking` still missing from
  `/v1/models` (only 3 advertised).
- **BUG-006:** MCP `understand_image` still HTTP 404, **and**
  MCP `web_search` is now also broken — new path
  `Cannot POST /v1/coding_plan/search`. So both tools in
  `codemax-mcp` are now broken.

**New observation:**

- Hidden-context probe now reports **~232 tokens** of injected
  prompt (was ~47 yesterday, +5×). Reporting is stable across
  all four models and three short prompts.

Overall grade improves D (52) → **C+ (78)**.

---

## Quick Status

| Category               | Score | Grade | Δ vs 0429-0635Z | Status                                                |
|------------------------|------:|:-----:|:---------------:|--------------------------------------------------------|
| Functional Suite       | 78    | C+    | **+14**         | yellow — 28/36 PASS (was 23). 3/4 streaming fixed; opus-4-7 still hit by upstream 529 |
| Authentication         | 100   | A     | 0               | green                                                  |
| Model Authenticity     | 100   | A     | 0               | green — real Anthropic backend confirmed               |
| Streaming              | 100   | A     | **+100**        | green — fixed (was F 0)                                |
| Web Search (server)    | 25    | F     | **+25**         | yellow → red — text dump gone, but tool not invoked    |
| MCP Tools              | 0     | F     | **−50**         | red — both `web_search` (new 404) and `understand_image` (old 404) now fail |
| Hidden System Prompt   | 25    | F     | **−25**         | yellow → red — context grew from ~47 to ~232 tokens (5×)|
| Hidden Context (probe) | 25    | F     | **−25**         | red — INFLATED-MODERATE (was MILD). Stable but 5× larger.|
| **Overall**            | **78**| **C+**| **+26**         | yellow → green — biggest single fix wave on Codemax    |

---

## Backend Authenticity

Same verdict: **real Anthropic Claude on all four routes.**
msg_* IDs ✓, response.model match ✓, signed-thinking-block
roundtrip ACCEPTED ✓ on all four models, tokenizer fingerprint
within range.

---

## Hidden Context — context grew, reporting still stable

The new `tests/test_hidden_context.py` probe sends each of three
short prompts five times and reports the input_tokens
distribution.

| Model                      | Verdict             | Median input_tokens | Δ vs 0429-0635Z |
|----------------------------|---------------------|---------------------|------------------|
| claude-opus-4-7            | **INFLATED-MODERATE** | 232                 | +185             |
| claude-opus-4-7-thinking   | **INFLATED-MODERATE** | 232                 | +185             |
| claude-sonnet-4-6          | **INFLATED-MODERATE** | 232                 | +185             |
| claude-haiku-4-5-20251001  | **INFLATED-MODERATE** | 232                 | +185             |

The hidden coding-assistant prompt grew approximately 5× in 24
hours. All four routes are inflated by exactly the same amount —
suggesting one shared injection layer that was extended.
Reporting is stable across runs (variance = 0).

For comparison: g0i is at ~13 tokens (no hidden context),
LightningZeus is between 24 and 6104 tokens (UNSTABLE).

---

## Bug status update

| Bug    | Title (short)                                            | Status now             | Notes                                                          |
|--------|----------------------------------------------------------|------------------------|----------------------------------------------------------------|
| BUG-001 | Streaming returns no text deltas                        | **Fixed**              | All 4 routes stream `text_delta` events normally.              |
| BUG-002 | web_search emits as plain text                          | **Partially fixed**    | No more `[TOOL_CALL]` text dump. But the tool still doesn't get invoked (`search_used=False` on all 4). |
| BUG-003 | basic_text truncated to first character                 | **Mostly fixed**       | 3/4 models PASS now. opus-4-7 still fails — upstream HTTP 529, not truncation. |
| BUG-004 | image_vision returns empty on Sonnet 4-6                | **Fixed**              | Sonnet 4-6 image_vision PASS. All 4 routes pass image_vision. |
| BUG-005 | claude-opus-4-7-thinking not in /v1/models              | Open                   | Still missing.                                                 |
| BUG-006 | MCP understand_image returns HTTP 404                   | **Worse**              | `understand_image` still 404 on `/v1/coding_plan/vlm`. **And** `web_search` MCP tool is now also 404 (new path `/v1/coding_plan/search`). Both tools broken. |

### New from this run

| Bug ID | Title                                                                     | Severity | Light  | Status |
|--------|---------------------------------------------------------------------------|:--------:|:------:|--------|
| BUG-7  | Hidden coding-assistant prompt grew ~5× (47 → 232 tokens) in 24 hours      | 4 Medium | yellow | Open / new |
| BUG-8  | MCP `web_search` returns HTTP 404 on new path `/v1/coding_plan/search`    | 6 Medium | yellow | Open / new |
| BUG-9  | `claude-opus-4-7` returns HTTP 529 "overloaded_error" on basic queries (3 of 9 tests, ~74 s wait each) | 7 High | red    | Open / likely upstream-side |

---

## BUG-7 — Hidden coding-assistant prompt grew ~5×

**Severity:** 4/10 — Medium
**Traffic light:** yellow
**Affected:** all `/v1/messages` requests
**Status:** Open / new

### What is wrong?

The token-stability probe reports a median of **232 input
tokens** for trivial 10-token user messages (e.g. "Reply with:
OK"). Yesterday's reading was 47. The hidden prompt grew about
5× in 24 hours.

### Customer impact

All four routes pay roughly 25× the visible prompt size in
hidden context. For short-prompt workflows, this is a real
cost overhead. For long-prompt workflows, the relative
overhead is small but the absolute tokens still flow to the
upstream meter.

### Reproduction

```bash
for i in 1 2 3; do
  curl -sS -X POST "$BASE_URL/v1/messages" \
    -H "Authorization: Bearer $CODEMAX_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"model":"claude-haiku-4-5-20251001","max_tokens":16,"messages":[{"role":"user","content":"Reply with: OK"}]}' \
    | jq '.usage.input_tokens'
done
# Expected: ~10
# Actual: 230 (consistent across calls)
```

### Fix

Trim or remove the coding-assistant system prompt. If you
want to keep it, expose an opt-out via a header or request
parameter, and disclose the token overhead on the dashboard.

---

## BUG-8 — MCP `web_search` now also 404

**Severity:** 6/10 — Medium
**Traffic light:** yellow
**Affected:** `codemax-mcp@1.0.4` tool `web_search`
**Status:** Open / new

### What is wrong?

In yesterday's run, `codemax-mcp`'s `web_search` tool worked
(returned real BTC price quotes from web sources). Today it
fails with:

```
Error: HTTP 404: Cannot POST /v1/coding_plan/search
```

Combined with the long-standing `understand_image` failure
(`Cannot POST /v1/coding_plan/vlm`), **both tools advertised
by your MCP server are now broken**.

### Customer impact

Customers using Claude Code or any MCP-based workflow with
`codemax-mcp` get errors on every tool call. The MCP server
advertises tools that 404 immediately.

### Reproduction

Spin up the MCP server (`npx codemax-mcp`) and call:

```json
{"method":"tools/call","params":{"name":"web_search","arguments":{"query":"current Bitcoin price USD","num_results":3}}}
```

Returns: `Error: HTTP 404: Cannot POST /v1/coding_plan/search`

### Fix

Same shape as the `understand_image` fix recommendation:
either restore the `/v1/coding_plan/search` endpoint, or
update the `codemax-mcp` package to call the correct path
(presumably `/v1/messages` with the
`web_search_20250305` server tool, the same way standard
Anthropic-API web_search calls work).

---

## BUG-9 — opus-4-7 returns HTTP 529 on basic queries

**Severity:** 7/10 — High (likely upstream-side)
**Traffic light:** red
**Affected:** `claude-opus-4-7`
**Status:** Open / likely upstream

### What is wrong?

Three of the nine functional tests on `claude-opus-4-7`
(basic_text, system_prompt, multi_turn) failed with:

```
HTTP 529: {"type":"error","error":{"type":"overloaded_error","message":"Anthropic's API is..."}}
```

Each call hung for ~74 seconds before returning.

### Customer impact

Customers using `claude-opus-4-7` for short non-streaming
calls see ~30 % failure rate plus 74-second waits when the
upstream is overloaded. The other Claude routes on Codemax
(opus-4-7-thinking, sonnet-4-6, haiku-4-5) are not affected
by this 529.

### Likely cause

This is an Anthropic-side overloaded_error. Codemax may not
be the responsible party.

### Recommended action

1. Add a fast-fail timeout (3–5 s) on the opus-4-7 route
   when upstream returns 529. Returning a clean 503 to the
   customer immediately is better than a 74-second hang.
2. If you have multiple upstream paths, route 529 retries
   to a less-loaded backend.

---

## Setup (run once)

```bash
export CODEMAX_API_KEY="<your-key>"
export BASE_URL="https://api.codemax.pro"
```

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $CODEMAX_API_KEY" \
  | jq -r '.data | length'
# Pass: prints a positive integer (we measured 3)
```

---

## Contact

For follow-up traces or to confirm a fix, please reach out to
**Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `codemax/2026-05-01T0951Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
