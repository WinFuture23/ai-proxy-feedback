# Codemax API Test Report — 2026-04-29 (06:35 UTC)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | Codemax                                                          |
| Endpoint         | https://api.codemax.pro                                          |
| Test date (UTC)  | 2026-04-29T0635Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-7-thinking, claude-sonnet-4-6, claude-haiku-4-5-20251001 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 2.0                                                              |
| Previous report  | [`codemax/2026-04-28T0503Z/`](../2026-04-28T0503Z/report.md)      |

---

## TL;DR vs. previous run

**No bugs from the previous report appear to have been fixed.**
Functional pass rate is unchanged at 23/36. The same six issues
(BUG-001 streaming, BUG-002 web_search-as-text, BUG-003 basic_text
truncated, BUG-004 image_vision empty on sonnet, BUG-005
opus-4-7-thinking not in catalog, BUG-006 MCP understand_image
404) all reproduce.

The new `tests/test_hidden_context.py` probe confirms what
prompt-leak measurements already suggested: Codemax injects a
**mild hidden prompt of ~47 tokens** consistently on every
request, across all four models. Reporting is stable (no
inconsistency). For comparison: g0i = ~13 tokens (no hidden
context), LightningZeus = ~60 tokens (down from ~6000).

Overall grade unchanged at **D (52)**.

---

## Quick Status

| Category               | Score | Grade | Δ vs 0503Z | Status                                                |
|------------------------|------:|:-----:|:----------:|--------------------------------------------------------|
| Functional Suite       | 64    | D     | 0          | red — 23/36 PASS, identical fail set                   |
| Authentication         | 100   | A     | 0          | green — all auth combinations return 200               |
| Model Authenticity     | 100   | A     | 0          | green — real Anthropic backend confirmed               |
| Streaming              | 0     | F     | 0          | red — chars=0 on all 4 models, no text_delta events    |
| Web Search (server)    | 0     | F     | 0          | red — `[TOOL_CALL]` text dump instead of structured blocks |
| MCP Tools              | 50    | D     | 0          | yellow — web_search OK, understand_image 404          |
| Hidden System Prompt   | 50    | D     | 0          | yellow — ~47 tokens (mild), consistent across models  |
| Hidden Context (probe) | 50    | D     | new        | yellow — INFLATED-MILD on all 4 routes (median ≈ 47 tokens) |
| **Overall**            | **52**| **D** | 0          | red — same set of unresolved issues                    |

---

## Backend Authenticity

Same verdict as previous run: **real Anthropic Claude on all four
routes.** msg_* IDs ✓, response.model match ✓, signed-thinking-block
roundtrip ACCEPTED ✓ on all four models, tokenizer fingerprint in
range ✓.

---

## Hidden Context — new probe data

The new `tests/test_hidden_context.py` probe sends each of three
short prompts five times and reports the input_tokens distribution.

| Model                      | Verdict        | Median input_tokens | Note               |
|----------------------------|----------------|---------------------|---------------------|
| claude-opus-4-7            | INFLATED-MILD  | 47                  | Stable, no variance  |
| claude-opus-4-7-thinking   | INFLATED-MILD  | 47                  | Stable               |
| claude-sonnet-4-6          | INFLATED-MILD  | 47                  | Stable               |
| claude-haiku-4-5-20251001  | INFLATED-MILD  | 47                  | Stable               |

47 tokens of hidden context on every call is mild but non-zero.
A bare "Reply with: OK" should tokenize to ~10 tokens; you're
adding ~30–35 on top. Stable reporting (variance = 0 across 5
samples per prompt) is good.

For comparison: g0i is at ~13 tokens (no hidden prompt). The
~30 token gap is consistent with the leaked
"coding-assistant" system prompt documented in earlier reports.

---

## Bugs from previous report — status now

| Bug    | Title (short)                                            | Status now | Notes                                                   |
|--------|----------------------------------------------------------|------------|----------------------------------------------------------|
| BUG-001 | Streaming returns no text deltas (chars = 0)             | Open       | All 4 routes still chars=0, ttft=n/a.                   |
| BUG-002 | web_search emits as plain text, not blocks               | Open       | All 4 routes still produce `[TOOL_CALL]\n{tool => ...}` |
| BUG-003 | basic_text truncated to first character / empty          | Open       | opus-4-7, sonnet-4-6, haiku-4-5 still affected. opus-4-7-thinking still passes (separate thinking budget). |
| BUG-004 | image_vision returns empty on claude-sonnet-4-6          | Inverted   | Sonnet image_vision now PASSes; opus-4-7 image_vision PASSes; **system_prompt on sonnet-4-6 now FAILs** (regression from PASS) |
| BUG-005 | claude-opus-4-7-thinking not in /v1/models               | Open       | Still missing.                                           |
| BUG-006 | MCP understand_image returns HTTP 404                    | Open       | Long-standing.                                           |

### Notable reshuffle on Sonnet 4-6

Sonnet's image_vision PASSed in this run (was FAIL in 0503Z),
but its system_prompt FAILed (was PASS in 0503Z). Net change is
zero (1 ⇄ 1), but the inversion is worth flagging in case it
points to internal routing changes you'd want to audit.

---

## Notes on what's unchanged

The four headline bugs from the previous report — streaming,
web_search emitted as text, basic_text truncated, MCP
understand_image — all reproduce verbatim. The same fixes
described in the 0503Z report still apply. Happy to re-test
once a build is deployed.

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
- Raw test output for this run: `codemax/2026-04-29T0635Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
