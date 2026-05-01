# LightningZeus API Test Report — 2026-05-01 (09:26 UTC)

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | LightningZeus                                                    |
| Endpoint         | https://lightningzeus.com                                        |
| Test date (UTC)  | 2026-05-01T0926Z                                                  |
| Models tested    | claude-opus-4-6, claude-opus-4-7, claude-sonnet-4.5, claude-haiku-4-5-20251001 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 3.0                                                              |
| Previous report  | [`lightningzeus/2026-04-29T0630Z/`](../2026-04-29T0630Z/report.md) |

---

## TL;DR vs. previous run

Real, sustained progress.

**Resolved:**
- Response IDs are back to `msg_*` (BUG-G fixed). Anthropic
  contract restored.
- `claude-opus-4-6` returns content again. Functional pass rate
  on this route went from 2/9 to **7/9** (`basic_text`,
  `multi_turn`, `tool_use`, `tool_roundtrip`, `image_vision`,
  `streaming_tps`, `list_models` all PASS).
- Backend authenticity confirmable again — signature roundtrip
  ACCEPTED on opus-4-6, indicating real Anthropic backend.

**Still open:**
- `system_prompt` and `web_search` on opus-4-6 still fail.
- Other three Claude routes (`opus-4-7`, `sonnet-4.5`, `haiku-4-5-20251001`) still HTTP 403 "not available".
- Hidden-context reporting is now **UNSTABLE** instead of
  consistently inflated: the same prompt sometimes reports 24
  input tokens, sometimes 6104. Customers cannot predict cost.
- 15 RPM rate limit still enforced.
- Catalog still contains `mmodel`, `OpusX-GPT`, `claudex-babe`.

Overall grade improves substantially: **F (15) → D (54)**.

---

## Quick Status

| Category               | Score | Grade | Δ vs 0429-0630Z | Status                                        |
|------------------------|------:|:-----:|:---------------:|------------------------------------------------|
| Functional Suite       | 19    | F     | **+13**         | red — opus-4-6 7/9 PASS; other 3 routes 0/9    |
| Authentication         | 100   | A     | 0               | green                                          |
| Model Authenticity     | 100   | A     | **+100**        | green — `msg_*` back, signature roundtrip ACCEPTED |
| Prefill / Continuation | 25    | F     | 0               | red — only opus-4-6 testable, 2/3 OK           |
| Thinking / Reasoning   | 0     | F     | 0               | red — all 4 routes BROKEN                      |
| Streaming              | 25    | F     | 0               | red — works only on opus-4-6                   |
| Web Search             | 0     | F     | 0               | red — opus-4-6 web_search still empty          |
| Hidden Context         | 25    | F     | **−25**         | red — UNSTABLE (24–6104 token range on same prompt) |
| Rate limits            | 0     | F     | 0               | red — 15 req/min cap still enforced            |
| **Overall**            | **54**| **D** | **+39**         | yellow — substantial backend fixes; cost predictability still a problem |

---

## Backend Authenticity (restored)

| Question                                          | Result this run                  | Previous run                       |
|---------------------------------------------------|----------------------------------|--------------------------------------|
| Does the response ID start with `msg_`?           | **yes** (e.g. `msg_tuz82x01gk1z5e27tc7hrwp2`) | no (`resp_*`)            |
| Does `response.model` match the requested model?  | yes                              | yes                                 |
| Does a signed `thinking` block roundtrip?         | **yes** (opus-4-6 ACCEPTED)       | could not test (empty content)       |
| Tokenizer fingerprint within Anthropic range?     | mixed (see Hidden Context below)  | mixed                                |

Verdict: **real Anthropic Claude on opus-4-6**, with high
confidence. The other three routes still 403, so they cannot
be probed.

---

## Hidden Context (now UNSTABLE)

| Model                      | Verdict (this run)  | input_tokens range across 5 samples per prompt | Previous run |
|----------------------------|---------------------|------------------------------------------------|---------------|
| claude-opus-4-6            | **UNSTABLE**         | 24, 24, 24, 25, 6104 for `"Reply with: OK"` (one outlier with full hidden prompt) | INFLATED-MILD ~58 |
| claude-opus-4-7            | BROKEN (403)         | n/a                                             | n/a (was 403)  |
| claude-sonnet-4.5          | BROKEN (403)         | n/a                                             | n/a (was 403)  |
| claude-haiku-4-5-20251001  | BROKEN (403)         | n/a                                             | n/a (was 403)  |

**This is the most important finding of this run.**

Five identical short prompts on the same model reported **input
token counts ranging from 24 to 6104**. That is a 254x spread
within five sequential calls. The 6104 reading suggests the
hidden prompt that was systematic in the 0428-1100 run is still
present on some requests but not others — likely a
caching / routing variability where some requests hit a path that
prepends the hidden prompt and others don't.

This is worse than a stable inflation, because customers cannot
reliably forecast their bill. With consistent inflation a
customer can multiply by a known factor; with intermittent
inflation they have to assume worst-case.

---

## Bug status update

| Bug    | Title (short)                                          | Status now            | Notes                                                                 |
|--------|--------------------------------------------------------|-----------------------|-----------------------------------------------------------------------|
| BUG-001 | Hidden context with inconsistent token reporting      | **Worse**             | Was MILD-but-stable, is now UNSTABLE — same prompt: 24 → 6104 tokens.  |
| BUG-002 | 3 of 4 advertised Claude models return HTTP 403       | Still Open            | opus-4-7, sonnet-4.5, haiku-4-5 still 403.                            |
| BUG-003 | User's `system` prompt overridden                     | Still Open            | Bracket-and-SYSTEM_OK probe still returns plain "Paris".               |
| BUG-005 | thinking parameter ignored                            | Still Open            | All 4 routes BROKEN.                                                   |
| BUG-006 | server-side web_search returns empty                  | Still Open            | opus-4-6 web_search FAIL (search_used=False).                          |
| BUG-007 | /v1/models contains typo + duplicate                  | Still Open            | Same 6-entry catalog including `mmodel`, `OpusX-GPT`, `claudex-babe`.  |
| BUG-008 | Streaming throughput slow                              | Marginal              | ~26 tok/s; still slow but workable.                                   |
| BUG-G   | Response IDs `resp_*` not `msg_*`                      | **Fixed**              | IDs now `msg_*`.                                                       |
| BUG-H   | opus-4-6 returns empty content for basic queries      | **Largely Fixed**     | 7/9 functional tests on opus-4-6 now PASS. system_prompt + web_search still fail. |
| BUG-I   | 15 RPM rate limit                                      | Still Open            | Same cap; still trips during normal test traffic.                     |
| BUG-J   | Catalog has non-Anthropic-named entries               | Still Open            | Same 6 entries.                                                        |

### New from this run

None — the issues observed are all carryovers, except the
"hidden context now intermittent" twist on BUG-001.

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
- Raw test output for this run: `lightningzeus/2026-05-01T0926Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
