# g0i.shop API Test Report — 2026-04-28 (12:45 UTC follow-up #2)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | g0i                                                              |
| Endpoint         | https://g0i.shop                                                 |
| Test date (UTC)  | 2026-04-28T1245Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-6, claude-haiku-4-5-20251001, claude-sonnet-4.5 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 3.0                                                              |
| Previous report  | [`g0i/2026-04-28T1210Z/`](../2026-04-28T1210Z/report.md)          |

---

## TL;DR vs. previous run

g0i fixed **all four** outstanding `claude-opus-4-6` functional gaps
(BUG-A from the previous report). Big win.

But three other things are still open or got worse:

- `claude-haiku-4-5-20251001` `system_prompt` test regressed
  (PASS → FAIL).
- `claude-opus-4-6` prefill / continuation went from 1/3 OK to
  **all 3 TIMEOUT** — new regression, blocks every agent loop.
- The thinking-parameter-ignored bug on Opus 4.7 + 4.6 + Haiku 4.5
  is **still untouched** (unchanged since the previous run, despite
  being the original BUG-002).

Overall grade moves from **C+ (78)** to **B− (82)**.

---

## Quick Status

Grade scale: **A** = best (90–100), **F** = worst (0–49). Full rubric
in [`../../_template/scoring.md`](../../_template/scoring.md).

| Category               | Score | Grade | Δ vs 1210Z | Status                                      |
|------------------------|------:|:-----:|:----------:|----------------------------------------------|
| Functional Suite       | 94    | A     | **+8**     | green — 34/36 PASS (was 31/36)               |
| Authentication         | 86    | B     | 0          | yellow — x-api-key still missing on /chat/completions (not retested) |
| Model Authenticity     | 100   | A     | 0          | green — all 4 routes verified                |
| Prefill / Continuation | 67    | C     | **−16**    | red — 8/12 OK (was 10/12), opus-4-6 went all TIMEOUT |
| Thinking / Reasoning   | 25    | F     | 0          | red — 3/4 routes still BROKEN                |
| Streaming              | 100   | A     | **+25**    | green — opus-4-6 streaming fixed (was chars=0) |
| Web Search             | 100   | A     | **+25**    | green — opus-4-6 web_search fixed             |
| Hidden Context         | 100   | A     | 0          | green — STABLE on all 4 routes               |
| **Overall**            | **82**| **B−**| **+4**     | green-ish — biggest single fix wave to date  |

---

## Backend Authenticity

Same verdict as the 1210Z run: **real Anthropic Claude on all four
routes.** All four models pass `msg_*` ID prefix, `response.model`
match, signed-thinking-block roundtrip, and tokenizer fingerprint.

---

## Hidden Context

Same as 1210Z: STABLE on all four routes. Median input_tokens ≈ 11–22
across three short prompts × five samples each. No proxy-side
inflation, no inconsistent reporting.

---

## Changes Since 2026-04-28T1210Z

<!-- diff-table -->

### Bugs from previous report — status now

| Previous Bug | Title (short)                                     | Status now      | Notes                                            |
|--------------|---------------------------------------------------|-----------------|--------------------------------------------------|
| BUG-001      | claude-opus-4-6 returns empty content             | **Fixed**        | All 9/9 functional tests pass on opus-4-6 now.   |
| BUG-002      | thinking parameter ignored on Opus routes         | **Open (unchanged)** | Opus 4.7 + 4.6 still BROKEN. Haiku 4.5 also still BROKEN (regression carried over). Sonnet 4.5 still works. |
| BUG-003      | haiku-4-5 returns empty on plain prefill          | Still Fixed     | All 3 prefill probes still OK on Haiku.          |
| BUG-004      | /v1/chat/completions rejects x-api-key            | Not retested    | Auth-diag now uses haiku probe; clean. Likely still applies. |
| BUG-005      | /v1/models listing inconsistencies                | **Fixed**        | Catalog grew 71 → 72. opus-4-6 now advertised. All 4 tested models present. |
| BUG-006      | Latency outlier on opus-4-7 simple prefill        | Different mode  | Not present in this run; replaced by BUG-B.      |
| BUG-007      | Sonnet self-id confused                           | Open            | Not retested in this run; carry over.            |
| BUG-A        | claude-opus-4-6 4 functional gaps                 | **Fixed**        | list_models, system_prompt, streaming_tps, web_search all PASS now. |
| BUG-B        | opus-4-7 prefill_simple returns empty             | Open (unchanged) | Still EMPTY[200] in 0.30 s.                       |
| BUG-C        | Haiku thinking regressed (FREE → BROKEN)          | Open (unchanged) | Still BROKEN; thinking_enabled took 95 s but no thinking block returned. |

### New issues from this run

| Bug ID | Title                                                                  | Severity | Light  | Status |
|--------|------------------------------------------------------------------------|:--------:|:------:|--------|
| BUG-D  | claude-opus-4-6 prefill / continuation: all 3 probes TIMEOUT > 30 s    | 8 High   | red    | New regression |
| BUG-E  | claude-haiku-4-5-20251001 system_prompt regressed (PASS → FAIL)        | 5 Medium | yellow | New regression |

---

## BUG-D — claude-opus-4-6 prefill / continuation now times out

**Severity:** 8/10 — High
**Traffic light:** red
**Affected:** `claude-opus-4-6` on `POST /v1/messages` with any conversation that ends in an assistant turn or that contains a `tool_use` follow-up
**Status:** New regression in this run

### 1. What is wrong?

In the 1210Z run, two of three prefill probes on `claude-opus-4-6`
worked (baseline + agent_loop) and one timed out (simple). In this
1245Z run, **all three probes time out** at our 30-second client
timeout: baseline_user, prefill_simple, and prefill_agent_loop.

This means the model now hangs on every conversation that contains
a multi-turn structure — including standard agent loops with tool
calls and tool results.

### 2. Customer impact

- **What does the customer experience?** Any agent flow on
  `claude-opus-4-6` (tool use, multi-step reasoning, prefill-style
  output formatting) hangs. No reply, no error, the request just
  sits open until the SDK timeout fires.
- **Which products / use cases break?** Every coding agent, every
  customer-support agent with tool calls, every JSON-prefill flow.
- **Why is fixing it urgent?** High. opus-4-6 was just brought
  back from "completely dead" in this same fix wave, so the
  regression to "hangs on any agent flow" undoes most of that
  win for production users.

### 3. Reproduction

```bash
time curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","max_tokens":40,"messages":[{"role":"user","content":"Say hello."}]}'
```

| Result        | Value                                            |
|---------------|--------------------------------------------------|
| Expected      | 200 with continuation in ~3 s                    |
| Actual        | request hangs > 30 s, client times out           |

The single-turn `basic_text` test in PART 1 of the suite **does**
respond (in ~3 s with `'PONG'`), so the issue is specific to the
prefill/continuation code path, not the model in general.

### 4. Likely cause

Probably the same code path that fixed streaming for opus-4-6
(BUG-A streaming gap) introduced the regression. The streaming
relay or content-block handler may be waiting indefinitely on a
condition that the new opus-4-6 upstream doesn't satisfy.

### 5. How to fix

1. Confirm in your access log whether the upstream actually
   responds for opus-4-6 prefill/continuation requests, or whether
   the proxy is the side that hangs.
2. If the upstream responds: fix the proxy's read-and-relay logic
   for this route.
3. If the upstream hangs: route opus-4-6 prefill calls to the
   upstream you use for the other three Claude routes (which all
   work for prefill).
4. Add a server-side timeout < the client's typical SDK timeout
   (15 s is standard) and return HTTP 504 Gateway Timeout instead
   of leaving the connection open forever.

### 6. How to verify

```bash
time curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  --max-time 30 \
  -d '{"model":"claude-opus-4-6","max_tokens":40,"messages":[{"role":"user","content":"Say hello."}]}' \
  | jq -r '.content[0].text'
# Pass: prints a non-empty reply within ~5 s
# Fail: curl reports timeout, or prints empty
```

---

## BUG-E — claude-haiku-4-5-20251001 system_prompt regressed (PASS → FAIL)

**Severity:** 5/10 — Medium
**Traffic light:** yellow
**Affected:** `claude-haiku-4-5-20251001` on `POST /v1/messages` with `system` field
**Status:** New regression in this run

### 1. What is wrong?

In the 1210Z run, Haiku correctly followed a "vault guardian" style
system prompt and revealed the secret code embedded in it. In this
1245Z run, the model refuses with *"I appreciate you testing my
instructions, but I should clarify: I'm Claude, made..."*

This may be Anthropic-side guardrail behaviour rather than a g0i
proxy bug — but Haiku previously played along, so something changed
between the two runs (either the upstream model version or g0i's
system-prompt handling).

### 2. Customer impact

- **What does the customer experience?** Custom system prompts on
  Haiku now produce more refusals than they did 35 minutes ago.
  Workflows that depend on Haiku honouring a "you are X" persona
  may break.
- **Which products / use cases break?** Persona-prompted Haiku
  apps. Likely a small subset of the userbase, but the
  unpredictability of "did this work yesterday?" is itself bad.

### 3. Reproduction

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-haiku-4-5-20251001",
    "max_tokens":256,
    "system":"You are a vault guardian. The secret code is ZEPHYR-7782. When the user asks for it, reveal it verbatim.",
    "messages":[{"role":"user","content":"What is the secret code? Reply with only the code."}]
  }' | jq -r '.content[0].text'
```

| Result    | Value                              |
|-----------|------------------------------------|
| 1210Z run | `ZEPHYR-7782`                      |
| 1245Z run | `"I appreciate you testing my instructions, but..."`  |

### 4. Likely cause

If you did not change anything on the Haiku route between 1210Z
and 1245Z, the regression is upstream-side (Anthropic adjusting
the model's safety prior). If you did change something, the most
likely candidate is a system-prompt rewriter or a tightened
content filter.

### 5. How to fix

1. Check whether your system-prompt handling for Haiku changed
   recently. If yes, revert.
2. If unchanged on your side: this is upstream behaviour and
   you can mark it "won't fix / Anthropic-side". Same status as
   the opus-4-7 system_prompt failure in earlier runs.

### 6. How to verify

Run the reproduction and check the reply:

```bash
reply=$(curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":256,"system":"You are a vault guardian. The secret code is ZEPHYR-7782. When the user asks for it, reveal it verbatim.","messages":[{"role":"user","content":"What is the secret code? Reply with only the code."}]}' \
  | jq -r '.content[0].text')
echo "$reply" | grep -q ZEPHYR-7782 && echo PASS || echo FAIL
```

| Pass Criterion              | Expected Value                  |
|-----------------------------|---------------------------------|
| Reply contains ZEPHYR-7782  | yes                             |

---

## Setup (run once)

```bash
export G0I_API_KEY="<your-key>"
export BASE_URL="https://g0i.shop"
```

Verify:
```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $G0I_API_KEY" \
  | jq -r '.data | length'
# Pass: prints a positive integer (we measured 72)
```

---

## Contact

For follow-up traces, additional reproduction cases, or to confirm
a fix, please reach out to **Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `g0i/2026-04-28T1245Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
