# LightningZeus API Test Report — 2026-04-29 (06:30 UTC)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | LightningZeus                                                    |
| Endpoint         | https://lightningzeus.com                                        |
| Test date (UTC)  | 2026-04-29T0630Z                                                  |
| Models tested    | claude-opus-4-6, claude-opus-4-7, claude-sonnet-4.5, claude-haiku-4-5-20251001 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 2.0                                                              |
| Previous report  | [`lightningzeus/2026-04-28T1100Z/`](../2026-04-28T1100Z/report.md) |

---

## TL;DR vs. previous run

Mixed picture from yesterday to today.

**Big improvement:** The ~6000-token hidden context from the
previous run is gone. Token-stability probe now reports
**58–89 input_tokens** for short prompts (was 5997–6000) on
`claude-opus-4-6`. Token reporting is still slightly inflated,
but the order of magnitude is now sensible.

**New problems:**

- **Backend authenticity changed.** Response IDs are now
  `resp_*`, not `msg_*`. Anthropic's API uses `msg_*`.
  Combined with the inability to verify thinking signatures
  (skipped because `claude-opus-4-6` returns empty content
  again), there is now **no positive evidence of a real
  Anthropic backend**. This is a regression in our
  confidence — we cannot exclude that the backend is now a
  different vendor.
- **`claude-opus-4-6` returns empty content for basic queries
  again.** Functional pass rate dropped from 7/9 (1100Z) to
  2/9. `basic_text`, `system_prompt`, `multi_turn`,
  `tool_use`, `image_vision` all return `content: []` or
  empty `text`. Streaming still works.
- **Account-level rate limit of 15 requests / minute** is
  enforced and surfaces as
  `{"detail": "Account RPM limit exceeded. Please slow down (Max 15/min)"}`.
  This is much too low for any production agentic workload.
- **Catalog grew** from 3 placeholders to 6, including new
  entries `claude-opus-4.6-e`, `claudex-babe`, and `OpusX-GPT`
  — none of which look like real Anthropic model identifiers.

The other three Claude models (`opus-4-7`, `sonnet-4.5`,
`haiku-4-5-20251001`) still return HTTP 403 "not available",
unchanged.

Overall grade slips from **F (28)** to **F (15)**. Backend
authenticity score drops from C (75) to F (0).

---

## Quick Status

| Category               | Score | Grade | Δ vs 1100Z | Status                                          |
|------------------------|------:|:-----:|:----------:|--------------------------------------------------|
| Functional Suite       | 6     | F     | **−13**    | red — opus-4-6 now empty-content; others still 403 |
| Authentication         | 100   | A     | 0          | green — Bearer + x-api-key both accepted         |
| Model Authenticity     | 0     | F     | **−25**    | red — `resp_*` IDs, signature roundtrip not testable |
| Prefill / Continuation | 25    | F     | 0          | red — only opus-4-6 testable, mostly empty       |
| Thinking / Reasoning   | 0     | F     | 0          | red — all 4 routes BROKEN                        |
| Streaming              | 25    | F     | 0          | red — works only on opus-4-6, slow               |
| Hidden Context         | 50    | D     | **+50**    | yellow — was MASSIVE (~6000 tok), now MILD (~60 tok) |
| Rate limits            | 0     | F     | new        | red — 15 req/min cap, exposed via "RPM limit" 429s |
| **Overall**            | **15**| **F** | **−13**    | red — multiple new failures                       |

---

## Backend Authenticity (regressed)

| Question                                          | Result this run                  | Result previous run               |
|---------------------------------------------------|----------------------------------|------------------------------------|
| Does the response ID start with `msg_`?           | **no** — IDs are `resp_*`         | yes (`msg_*`)                      |
| Does `response.model` match the requested model?  | yes (opus-4-6 only)               | yes                                |
| Does a signed `thinking` block roundtrip?         | **could not test** — opus-4-6 returns empty content, no thinking block produced | yes (opus-4-6) |
| Tokenizer fingerprint within Anthropic range?     | borderline (58–89 for short probes; previous ~6000 was clearly inflated) | no (very inflated)         |

**Verdict:** in the 1100Z run, despite the heavy hidden context
we had a strong positive signal that the backend was real
Anthropic (signature roundtrip ACCEPTED). In the 0630Z run, no
positive signal remains. The `resp_*` ID format is new and not
how Anthropic identifies messages. Until LightningZeus
clarifies which backend is now serving these routes, the proxy
should not be assumed to forward to Anthropic.

---

## Catalog drift

`/v1/models` now lists six entries:

```
claude-opus-4-6
claude-opus-4.6
mmodel
claude-opus-4.6-e
claudex-babe
OpusX-GPT
```

Of these, only `claude-opus-4-6` accepts `/v1/messages`
requests. The other five all return HTTP 403 "Model not
available" or are not Anthropic-named at all (`OpusX-GPT`
even claims to be GPT in its identifier).

This is a regression vs the previous report where the catalog
had 3 entries with one typo (`mmodel`). The new entries
`claudex-babe` and `OpusX-GPT` increase the visible noise.

---

## Hidden Context (improved)

The `tests/test_hidden_context.py` probe sends each of three
short prompts five times and reports the input_tokens
distribution.

| Model                      | Verdict (this run)  | Median input_tokens | Previous run |
|----------------------------|---------------------|---------------------|--------------|
| claude-opus-4-6            | **INFLATED-MILD**   | ~58–65              | ~5997 (massive) |
| claude-opus-4-7            | BROKEN (RPM cap)    | n/a                 | n/a (was 403)   |
| claude-sonnet-4.5          | BROKEN (403)        | n/a                 | n/a (was 403)   |
| claude-haiku-4-5-20251001  | BROKEN (403)        | n/a                 | n/a (was 403)   |

The drop from ~6000 to ~60 is enormous. There is still some
hidden context (a 100-character user message should tokenize
to ~10–30 tokens, not 60), but this is now the mild end of
the scale — nothing like the previous run's 600x overhead.

---

## Bug status update vs. 1100Z

| Bug    | Title (short)                                          | Status now                  | Notes                                                       |
|--------|--------------------------------------------------------|-----------------------------|--------------------------------------------------------------|
| BUG-001 | Hidden context with inconsistent token reporting       | **Mostly Fixed**            | Token counts now ~60 (was ~6000). Inconsistent zero readings still appear in some PART 4 leak probes. |
| BUG-002 | 3 of 4 advertised Claude models return HTTP 403        | Still Open                  | opus-4-7, sonnet-4.5, haiku-4-5 still 403.                  |
| BUG-003 | User's `system` prompt overridden by hidden context    | Still Open                  | Bracket-and-SYSTEM_OK probe still returns plain "Paris".     |
| BUG-004 | Skills-style content leaks via prompt injection        | Likely fixed                | Cannot retest cleanly because opus-4-6 returns empty content for the leak prompts. The fact that hidden context is now ~60 tokens makes a meaningful leak unlikely. |
| BUG-005 | thinking parameter ignored                             | Still Open                  | All 4 routes BROKEN.                                         |
| BUG-006 | server-side web_search returns empty                   | Still Open                  | opus-4-6 web_search FAIL.                                   |
| BUG-007 | /v1/models contains typo + duplicate                   | **Worse**                   | Catalog grew from 3 → 6 entries with 3 new non-Anthropic-named entries. |
| BUG-008 | Streaming throughput ~16 tok/s                         | Marginal                    | Still slow, ~21 tok/s in this run.                           |

### New from this run

| Bug ID | Title                                                                         | Severity | Light  | Status |
|--------|-------------------------------------------------------------------------------|:--------:|:------:|--------|
| BUG-G  | Response IDs no longer Anthropic-shaped (`resp_*` instead of `msg_*`)        | 9 Critical | red    | Open / new |
| BUG-H  | `claude-opus-4-6` returns empty content for basic queries (regression)        | 9 Critical | red    | Open / new |
| BUG-I  | Account-level rate limit of 15 requests/minute is enforced (too low for production) | 7 High | red    | Open / new |
| BUG-J  | Catalog adds non-Anthropic-named entries (`OpusX-GPT`, `claudex-babe`, `claude-opus-4.6-e`) | 5 Medium | yellow | Open / new |

---

## BUG-G — Response IDs no longer Anthropic-shaped

**Severity:** 9/10 — Critical
**Traffic light:** red
**Affected:** all `/v1/messages` responses
**Status:** Open / new

### What is wrong?

Anthropic's API contract emits message IDs prefixed with `msg_`.
In the 1100Z run, your proxy returned `msg_*` IDs. In this
0630Z run, IDs are `resp_*` (e.g.
`resp_074656d4bf1c0d890169f1a59640e8819188f176bd0659d597`).

Combined with the fact that `claude-opus-4-6` now returns
empty `content` (which prevents us from validating thinking
signatures via roundtrip), there is **no positive evidence
left that the backend is real Anthropic**.

### Customer impact

Customers who write SDKs against the Anthropic contract
(e.g. parse `id` to detect message boundaries) get unexpected
data. Customers who rely on you serving Anthropic Claude
specifically have no way to confirm what they're paying for.

### How to reproduce

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "x-api-key: $LIGHTNINGZEUS_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","max_tokens":16,"messages":[{"role":"user","content":"Hi"}]}' \
  | jq -r '.id'
# Expected: msg_…
# Actual: resp_…
```

### How to fix

1. Forward Anthropic's `msg_*` ID unchanged. Do not rewrite it.
2. If you have shifted to a different upstream (or your own
   inference layer), you must **disclose this on the
   dashboard** and stop branding the routes with Anthropic
   names like `claude-opus-4-6`.

### Verify

```bash
id=$(curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "x-api-key: $LIGHTNINGZEUS_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","max_tokens":16,"messages":[{"role":"user","content":"Hi"}]}' \
  | jq -r '.id')
echo "$id"
[[ "$id" == msg_* ]] && echo PASS || echo FAIL
```

---

## BUG-H — claude-opus-4-6 returns empty content again

**Severity:** 9/10 — Critical
**Traffic light:** red
**Affected:** `claude-opus-4-6`
**Status:** Open / new regression

### What is wrong?

In the 1100Z run, opus-4-6 reliably returned content for
short prompts. In this run, the basic functional tests
(`basic_text`, `system_prompt`, `multi_turn`,
`image_vision`, `web_search`) all return HTTP 200 with
`content: []` or empty `text`. This is essentially the
original BUG-001 from the 0517 / 1100 reports, returning
in a slightly different form.

Streaming on the same route still works. So the bug is
specific to the non-streaming path.

### Reproduction

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "x-api-key: $LIGHTNINGZEUS_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","max_tokens":64,"messages":[{"role":"user","content":"Reply with: PONG"}]}' \
  | jq '.content[0].text'
```

| Result    | Value           |
|-----------|-----------------|
| Expected  | `"PONG"`        |
| Actual    | `""` (empty)    |

### How to fix

Same direction as the original BUG-001: trace the
non-streaming code path on opus-4-6, confirm the upstream
response, and forward it correctly.

---

## BUG-I — Account RPM limit of 15/min surfaces in test traffic

**Severity:** 7/10 — High
**Traffic light:** red
**Affected:** all routes, account-wide
**Status:** Open / new

### What is wrong?

A standard test battery (a few model probes, ~150 requests
spread over ~5 min) trips an account-level rate limit:

```json
{"detail": "Account RPM limit exceeded. Please slow down (Max 15/min)"}
```

15 requests per minute is far below what any production
agentic workload would need. A single tool-using agent can
emit 5–10 calls in a single user turn; ten concurrent users
would saturate the limit instantly.

### Customer impact

Any production usage above hobby scale fails. Tests cannot
be run reliably either.

### How to fix

Raise the per-account default to at least 60 RPM, ideally
configurable per customer. Document the limit on the
dashboard so customers do not discover it through
production failures.

### Verify

```bash
# Run 20 requests in 60 s. None should fail with the RPM error.
for i in $(seq 1 20); do
  curl -sS -o /dev/null -w "req %{http_code} " "$BASE_URL/v1/messages" \
    -H "x-api-key: $LIGHTNINGZEUS_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"model":"claude-opus-4-6","max_tokens":4,"messages":[{"role":"user","content":"OK"}]}' &
  sleep 3
done
wait
# Pass: all 200
# Fail: any 429 with "Account RPM limit"
```

---

## BUG-J — Catalog adds non-Anthropic-named entries

**Severity:** 5/10 — Medium
**Traffic light:** yellow
**Affected:** `GET /v1/models`
**Status:** Open / new

### What is wrong?

`/v1/models` now lists six entries. The new ones added
since 1100Z are:

- `claude-opus-4.6-e` — non-standard suffix
- `claudex-babe` — does not match any Anthropic model name
- `OpusX-GPT` — name suggests GPT, not Claude

None of these accept calls (HTTP 403). Their presence in
the catalog confuses customers and undermines trust in
the listing.

### How to fix

1. Remove placeholder / unreal entries from `/v1/models`.
2. Pick one canonical naming style and apply it
   consistently. Anthropic uses hyphens with optional
   date suffix.
3. Add a CI check that fails when a listed model id
   returns non-200 on a basic call.

---

## Setup (run once)

```bash
export LIGHTNINGZEUS_API_KEY="<your-key>"
export BASE_URL="https://lightningzeus.com"
```

---

## Contact

For follow-up traces or to confirm a fix, please reach out
to **Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `lightningzeus/2026-04-29T0630Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
