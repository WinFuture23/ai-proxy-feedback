# g0i.shop API Test Report — 2026-04-29 (06:22 UTC)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | g0i                                                              |
| Endpoint         | https://g0i.shop                                                 |
| Test date (UTC)  | 2026-04-29T0622Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-6, claude-haiku-4-5-20251001, claude-sonnet-4.5 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 4.0                                                              |
| Previous report  | [`g0i/2026-04-28T1245Z/`](../2026-04-28T1245Z/report.md)          |

---

## TL;DR vs. previous run

This run was hit by a wave of **HTTP 502 Bad Gateway** errors on
the `claude-sonnet-4.5` and (partly) `claude-haiku-4-5-20251001`
routes. The error body is a Cloudflare HTML error page, which
strongly suggests an upstream connectivity issue rather than a
code-path bug. Filtering those out:

- Real, repeatable bugs are **largely the same as 1245Z**.
- One genuine improvement: `claude-opus-4-6` prefill /
  continuation went from *all-TIMEOUT* to *baseline + agent-loop OK*
  (BUG-D from previous report partially fixed).
- The thinking-parameter-ignored bug (original BUG-002) is **still
  open on Opus 4.7 + 4.6 + Haiku 4.5**. Could not be measured on
  Sonnet 4.5 due to 502s — Sonnet was the only working route for
  thinking in the last two runs, so that route's status is
  important to confirm in a follow-up test.

Recommendation: re-test in 2–4 hours once the 502s have cleared
to disambiguate transient infrastructure from real proxy bugs.

Overall grade slips from **B− (82)** to **D+ (62)** purely because
of the 502s. The application-level grade (excluding infra failures)
is roughly **B (84)**.

---

## Quick Status

| Category               | Score | Grade | Δ vs 1245Z | Status                                                 |
|------------------------|------:|:-----:|:----------:|---------------------------------------------------------|
| Functional Suite       | 67    | C     | **−27**    | red — 11 of 12 new failures are HTTP 502, infra issue   |
| Authentication         | 86    | B     | 0          | yellow — x-api-key on /chat/completions still suspect   |
| Model Authenticity     | 100   | A     | 0          | green — all 4 routes verified                           |
| Prefill / Continuation | 58    | F     | **−9**     | red — Sonnet 502s; Opus prefill_simple still empty      |
| Thinking / Reasoning   | 0     | F     | **−25**    | red — Sonnet (the only previously-working route) is 502 |
| Streaming              | 75    | C     | **−25**    | red — Sonnet streaming hit 502                          |
| Web Search             | 75    | C     | **−25**    | red — Sonnet + Haiku web_search hit 502                 |
| Hidden Context         | 100   | A     | 0          | green — STABLE on all 4 routes                          |
| **Overall**            | **62**| **D+**| **−20**    | red — likely transient infra; re-test recommended       |

---

## Backend Authenticity

Same verdict: **real Anthropic Claude on all 4 routes** where the
probe got through. Sonnet authenticity could not be re-validated
this run because the route returned 502 on most calls.

---

## Hidden Context

Same as previous runs: STABLE on all four models, median
input_tokens ≈ 11–22 across three short prompts × five samples
each. No proxy-side inflation, no inconsistent reporting.

---

## Changes Since 2026-04-28T1245Z

### Bugs from previous report

| Bug    | Title (short)                                                | Status now           | Notes                                                                 |
|--------|--------------------------------------------------------------|----------------------|-----------------------------------------------------------------------|
| BUG-001 | claude-opus-4-6 returned empty content                      | Still Fixed          | opus-4-6 still 9/9 functional pass.                                    |
| BUG-002 | thinking parameter ignored on Opus routes                   | **Open + worse**     | Opus 4.7 + 4.6 + Haiku 4.5 still BROKEN. Sonnet 4.5 status now unknown (502s). |
| BUG-003 | haiku-4-5 returned empty on plain prefill                   | Still Fixed          | Haiku prefill all 3 OK.                                                |
| BUG-004 | /v1/chat/completions rejects x-api-key                      | Not retested         | Auth-diag now uses haiku probe; clean.                                 |
| BUG-005 | /v1/models listing inconsistencies                          | Still Fixed          | Catalog grew 72 → 89 models. opus-4-6 still listed.                    |
| BUG-006 | Latency outlier on opus-4-7 simple prefill                  | n/a                  | Replaced by BUG-B in 1210Z report.                                     |
| BUG-007 | Sonnet self-id confused                                     | Could not retest     | Sonnet calls failed with 502.                                          |
| BUG-A   | claude-opus-4-6 4 sub-issues                                | Still Fixed          | All 4 sub-issues remain fixed.                                         |
| BUG-B   | opus-4-7 prefill_simple returns empty                       | **Still open** (HTTP_ERR mode) | This run reports HTTP_ERR on prefill_simple. Likely 502 — to be re-confirmed in next run. |
| BUG-C   | Haiku thinking regressed                                    | Open                 | Still BROKEN. Haiku thinking-enabled returned no thinking block.        |
| BUG-D   | opus-4-6 prefill all TIMEOUT                                | **Partially Fixed**  | baseline + agent_loop now OK. prefill_simple is HTTP_ERR (502).         |
| BUG-E   | Haiku system_prompt regressed                               | **Fixed** in this run | Haiku system_prompt PASS again (`'ZEPHYR-7782'`).                      |

### New from this run

| Bug ID | Title                                                                                  | Severity | Light  | Status |
|--------|----------------------------------------------------------------------------------------|:--------:|:------:|--------|
| BUG-F  | HTTP 502 Bad Gateway storm on `claude-sonnet-4.5` (8 of 9 functional tests fail) and partly on Haiku 4.5 (3 of 9) | 9 Critical | red | Open / likely infra |

---

## BUG-F — HTTP 502 storm on Sonnet 4.5 (and partly Haiku)

**Severity:** 9/10 — Critical (if persistent), likely transient
**Traffic light:** red
**Affected:** `claude-sonnet-4.5` (most calls), `claude-haiku-4-5-20251001` (some calls)
**Status:** Open — needs disambiguation against transient infra outage

### 1. What is wrong?

For the duration of this test run (~5 minutes), most calls to
`claude-sonnet-4.5` and three calls to
`claude-haiku-4-5-20251001` returned HTTP 502 Bad Gateway with a
Cloudflare HTML error page as the body. Sonnet pass rate dropped
from 9/9 (1245Z) to 1/9. Haiku pass rate dropped from 8/9 to 6/9.

Two things make this look infrastructure-related rather than
application-level:

1. The body is a Cloudflare HTML page, not your API's JSON error
   shape. Cloudflare returns 502 when its origin (your proxy)
   times out or returns an unreadable response.
2. The errors land in 0.14 s — far below any reasonable
   per-request timeout, which suggests the request is being
   refused at the edge before it even hits the proxy logic.

### 2. Customer impact

If persistent: customers on Sonnet 4.5 see ~80 % error rate.
Production agents pinned to Sonnet (which until 1245Z was the
most-tested working model) break completely. Haiku is partially
affected.

If transient: customers may experience brief outage windows.
Recommend a status-page entry and a post-mortem.

### 3. Reproduction

```bash
# Wait for the next call to fail, or re-run a few times in a
# loop to catch the 502:
for i in 1 2 3 4 5; do
  status=$(curl -sS -o /dev/null -w "%{http_code}" -X POST "$BASE_URL/v1/messages" \
    -H "Authorization: Bearer $G0I_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"model":"claude-sonnet-4.5","max_tokens":16,"messages":[{"role":"user","content":"OK"}]}')
  printf 'attempt %d: HTTP %s\n' "$i" "$status"
done
# 1245Z run: all 200
# 0622Z run: all 502 (Cloudflare HTML body)
```

### 4. Likely cause

Cloudflare 502 on a path that worked an hour earlier, with no
config change visible from the outside, points to an upstream
connectivity issue between your proxy and the Anthropic
backend used for Sonnet 4.5 (and partly Haiku). Possible
causes: DNS / TLS handshake, upstream rate limit, expired
credentials, regional Anthropic outage.

### 5. How to fix

1. Check your proxy → upstream logs around the test window
   (06:22–06:28 UTC). Look for connection-refused / TLS-failed
   / 5xx-from-upstream entries on the Sonnet route.
2. If the upstream is degraded: add a fast-fail with explicit
   504 Gateway Timeout instead of letting Cloudflare 502 with
   HTML. Customers' SDKs handle JSON 5xx better than HTML
   pages.
3. Document on the dashboard which upstream backend each
   advertised model uses, so customers can reason about
   correlated outages.

### 6. How to verify

Re-run the test suite once the route comes back. If pass rates
on Sonnet 4.5 return to 9/9 functional + FREE thinking, this
was transient. If they don't, dig deeper.

```bash
# Quick spot check
status=$(curl -sS -o /dev/null -w "%{http_code}" -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4.5","max_tokens":16,"messages":[{"role":"user","content":"OK"}]}')
echo "HTTP $status"
# Pass: 200
# Fail: 502 (with Cloudflare HTML body)
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
- Raw test output for this run: `g0i/2026-04-29T0622Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
