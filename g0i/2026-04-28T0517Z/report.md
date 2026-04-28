# g0i.shop API Test Report — 2026-04-28

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | g0i                                                              |
| Endpoint         | https://g0i.shop                                                 |
| Test date (UTC)  | 2026-04-28T0517Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-6, claude-haiku-4-5-20251001, claude-sonnet-4.5 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 1.1                                                              |

---

## How to read this report

1. **Quick Status** — score per area and overall grade. **Higher is
   better. A is the best grade, F is the worst.**
2. **Backend Authenticity** — short verdict on whether the model behind
   the API is the model the provider claims.
3. **Bug Summary** — one row per problem, sorted by severity, with a
   traffic-light indicator.
4. **Each bug** has six fixed sections: what is wrong, customer impact,
   how to reproduce, likely cause, how to fix, how to verify the fix.

The reproduction commands assume you ran the **Setup** block once
(see end of report).

---

## Quick Status

Grade scale: **A** = best (90–100), **F** = worst (0–49). Full rubric
in [`../../_template/scoring.md`](../../_template/scoring.md).

| Category               | Score (0–100) | Grade | Trend             | Status                                |
|------------------------|--------------:|:-----:|:-----------------:|----------------------------------------|
| Functional Suite       | 69            | C     | new               | yellow — 1 model dead, others good     |
| Authentication         | 86            | B     | new               | yellow — x-api-key missing on /chat/completions |
| Model Authenticity     | 75            | C     | new               | yellow — backend looks like real Anthropic, dead model drags score |
| Prefill / Continuation | 67            | C     | Δ +30             | yellow — Opus 4.7 fixed, Haiku partial |
| Thinking / Reasoning   | 50            | D     | new               | red — broken on both Opus routes       |
| Streaming              | 75            | C     | new               | yellow — works on 3 of 4 models        |
| Web Search             | 100           | A     | new               | green — works on all live models       |
| **Overall**            | **75**        | **C** | **Δ approx. +18** | **yellow — functional but real gaps**  |

---

## Backend Authenticity

We run four independent checks to detect whether g0i is actually
serving Anthropic Claude or has silently re-routed Claude requests to
a cheaper backend (OpenAI, Gemini, or open-source). The provider
should not be trusted blindly here — these checks are the only
external signal we have.

| Question                                         | Result                              | What it tells us                         |
|--------------------------------------------------|-------------------------------------|------------------------------------------|
| Does the response ID start with `msg_`?          | yes (all 4 models)                  | Anthropic-shaped response                |
| Does `response.model` match the requested model? | yes (all 4 models)                  | Routing reaches the requested route      |
| Does a signed `thinking` block roundtrip?        | **yes** (opus-4-7, haiku, sonnet)   | Strong evidence of real Anthropic backend (only Anthropic can re-validate its own signatures) |
| Tokenizer fingerprint within Anthropic range?    | mostly yes (34–52 tok / 100 chars)  | Consistent with Anthropic tokenizer      |

**Verdict: very likely real Anthropic Claude.** The thinking-signature
roundtrip is the strongest single signal — a different backend cannot
re-validate signatures it did not produce. The dead `claude-opus-4-6`
route could not be probed (skipped due to BUG-001).

---

## Resolved Since Last Report (2026-04-25)

| Previous Bug | Title                                                          | Status      |
|--------------|----------------------------------------------------------------|-------------|
| —            | claude-opus-4-7 returned HTTP 400 *"This model does not support assistant message prefill"* with `req_vrtx_*` request id | green Fixed |

All three prefill probes (baseline, simple, agent-loop) on
`claude-opus-4-7` now return HTTP 200 with real continuation text.
Thank you for the fast turnaround on the Vertex routing.

---

## Bug Summary

| Bug ID  | Title                                                            | Severity | Light  | Affected                          | Status |
|---------|------------------------------------------------------------------|:--------:|:------:|-----------------------------------|--------|
| BUG-001 | claude-opus-4-6 returns empty content for every request          | 10       | red    | claude-opus-4-6                   | Open   |
| BUG-002 | `thinking` parameter is ignored on Opus routes                   | 9        | red    | claude-opus-4-7, claude-opus-4-6  | Open   |
| BUG-003 | claude-haiku-4-5-20251001 returns empty on plain assistant prefill | 7      | red    | claude-haiku-4-5-20251001         | Open   |
| BUG-004 | /v1/chat/completions rejects x-api-key                           | 5        | yellow | /v1/chat/completions              | Open   |
| BUG-005 | /v1/models listing inconsistencies                               | 4        | yellow | /v1/models                        | Open   |
| BUG-006 | Latency outlier on opus-4-7 simple prefill (~20 s)               | 3        | green  | claude-opus-4-7                   | Open   |
| BUG-007 | Model self-identification is incorrect                           | 2        | green  | claude-sonnet-4.5                 | Open   |

---

## BUG-001 — claude-opus-4-6 returns empty content for every request

**Severity:** 10/10 — Critical
**Traffic light:** red
**Affected:** `claude-opus-4-6` on `POST /v1/messages`
**Status:** Open

### 1. What is wrong?

Every call to `claude-opus-4-6` returns HTTP 200 with an empty
`content` list. The client sees a successful response, but the
model never said anything. This makes the model unusable for any
purpose.

### 2. Customer impact

- **What does the customer experience?** The user clicks "send" and
  the assistant returns nothing. No error, no warning — just a
  blank reply. From the user's perspective the product is broken.
- **Which products / use cases break?** Anything pointed at
  `claude-opus-4-6`: chatbots, code assistants, agent loops,
  document-processing pipelines. All return empty results.
- **Why is fixing it urgent?** Customers paying for Opus 4.6 are
  effectively paying for nothing. They will either churn to a
  different provider or submit refund requests.

### 3. How to reproduce

After the **Setup** block, paste this command:

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-6",
    "max_tokens": 64,
    "messages": [{"role":"user","content":"Reply with exactly: PONG"}]
  }' | jq '{content, stop_reason, output_tokens: .usage.output_tokens}'
```

| Result    | Value                                                          |
|-----------|----------------------------------------------------------------|
| Expected  | `content: [{"type":"text","text":"PONG"}]`, `stop_reason: "end_turn"`, `output_tokens` > 0 |
| Actual    | `content: []`, `stop_reason: null`, `output_tokens: 0`         |
| Latency   | 0.3 – 0.8 s (suspiciously fast)                                |
| HTTP code | 200                                                            |

### 4. Likely cause

The route for `claude-opus-4-6` looks like it short-circuits before
reaching any backend. Three signals support this:

1. Latency is far below normal (under 1 s vs. ~3 s for other models).
2. `output_tokens: 0` — the model never produced anything.
3. `stop_reason: null` — Anthropic's API requires one of `end_turn`,
   `max_tokens`, `stop_sequence`, `tool_use`. A null value is invalid.

`claude-opus-4-6` is also missing from `/v1/models` (see BUG-005),
which suggests the model was removed from the public catalog but
its routing entry was not updated.

### 5. How to fix

Choose one of the two options:

**Option A — Restore the route (preferred if the model is supposed to work)**

1. Open your routing or proxy configuration.
2. Find the entry that maps `claude-opus-4-6` to its upstream
   (Vertex / Bedrock / direct Anthropic).
3. Confirm the upstream model id is still valid. Anthropic
   deprecated `claude-opus-4-6` in favour of `claude-opus-4-7`.
4. If the upstream returns 404 for this model id, either stop
   accepting the deprecated id (Option B) or document a transparent
   alias on the dashboard so customers know calls are being served
   by a different model.

**Option B — Remove the broken route (preferred if the model is retired)**

1. Remove `claude-opus-4-6` from any model alias / routing table.
2. Have `/v1/messages` return HTTP 410 Gone with body
   `{"error":{"type":"model_retired","message":"claude-opus-4-6 has been retired. Use claude-opus-4-7."}}`
3. Document the change on your model status page.

### 6. How to verify the fix

Paste the same reproduction command from step 2.

| Pass Criterion        | After Option A                              | After Option B |
|-----------------------|---------------------------------------------|----------------|
| HTTP status           | 200                                         | 410            |
| `content[0].text`     | non-empty real reply (e.g. `"PONG"`)        | n/a            |
| `stop_reason`         | `"end_turn"`                                | n/a            |
| `output_tokens`       | greater than 0                              | n/a            |
| Latency               | 1 s or more                                 | under 0.3 s    |

One-line check (Option A):

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","max_tokens":32,"messages":[{"role":"user","content":"Say PONG"}]}' \
  | jq -r '.content[0].text // "FAIL: empty content"'
# Pass: prints "PONG"
# Fail: prints "FAIL: empty content"
```

---

## BUG-002 — `thinking` parameter is ignored on Opus routes

**Severity:** 9/10 — Critical (blocks all reasoning use cases on Opus)
**Traffic light:** red
**Affected:** `claude-opus-4-7` and `claude-opus-4-6` on `POST /v1/messages`
**Status:** Open

### 1. What is wrong?

When a client asks Claude to use extended thinking, Opus models do
not include a `thinking` block in the response. Haiku and Sonnet
on the same proxy do include it, so the bug is route-specific.

This breaks every workflow that relies on chain-of-thought
reasoning, multi-step planning, or visible model deliberation.

### 2. Customer impact

- **What does the customer experience?** Customers paying for Opus
  to get high-quality reasoning silently lose the reasoning step.
  Their answers come back faster but lower-quality, and they have
  no error message to attribute the regression to.
- **Which products / use cases break?** Coding agents, data
  analysis pipelines, math/legal reasoning tools, multi-step
  planning agents, anything using `extended_thinking_2025` for
  better answers. The product still "works" but quality drops.
- **Why is fixing it urgent?** Opus is your premium-priced model.
  If it doesn't reason on demand, customers paying premium prices
  receive standard-quality output. This is a billing-vs-delivery
  mismatch and likely violates customer expectations.

### 3. How to reproduce

After the **Setup** block:

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 2048,
    "thinking": {"type": "enabled", "budget_tokens": 1024},
    "messages": [{"role":"user","content":"What is 7 * 9? Think first, then answer."}]
  }' | jq '[.content[].type]'
```

| Result    | Value                              |
|-----------|------------------------------------|
| Expected  | `["thinking", "text"]`             |
| Actual    | `["text"]`                         |
| HTTP code | 200                                |

Run the same command with `"model": "claude-sonnet-4.5"` and you
will see `["thinking", "text"]` — confirming the upstream supports
the feature, only the Opus routes don't pass it through.

### 4. Likely cause

The Opus route handler is dropping the `thinking` field before
forwarding the request to the upstream model. Possible places:

- A request-shaping middleware that allow-lists fields and forgot
  to add `thinking` for the Opus models.
- A model-config table where the `supports_thinking` flag is set
  to `false` for Opus by mistake.
- A different upstream (e.g. an older Vertex deployment) that does
  not yet expose extended thinking.

### 5. How to fix

1. In your request middleware, allow the `thinking` field for both
   `claude-opus-4-7` and `claude-opus-4-6` (if you keep that route).
2. If your model-config table has a per-model capability flag,
   set `supports_thinking = true` for both Opus entries.
3. If the underlying issue is the upstream, route Opus thinking
   requests to the same upstream you use for Sonnet/Haiku
   (Anthropic-direct or AWS Bedrock with the recent Claude
   offering — both pass `thinking` through unchanged).
4. Streaming with thinking: confirm `content_block_start` events
   include `{"type":"thinking"}` blocks before any `text_delta`.

### 6. How to verify the fix

Paste this command. The pass criterion is the array of content
block types.

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":2048,"thinking":{"type":"enabled","budget_tokens":1024},"messages":[{"role":"user","content":"Compute 173 * 217 step by step."}]}' \
  | jq '[.content[].type]'
# Pass: ["thinking","text"]   (or ["thinking","thinking","text"])
# Fail: ["text"]
```

| Pass Criterion                | Expected Value                           |
|-------------------------------|-------------------------------------------|
| HTTP status                   | 200                                       |
| First content block type      | `thinking`                                |
| At least one `text` block     | yes                                       |
| `thinking` block has signature| yes (`signature` field, length 64)        |

---

## BUG-003 — claude-haiku-4-5-20251001 returns empty content on plain assistant prefill

**Severity:** 7/10 — High
**Traffic light:** red
**Affected:** `claude-haiku-4-5-20251001` on `POST /v1/messages`
**Status:** Open

### 1. What is wrong?

When the conversation history ends with a plain text assistant
message (the standard "prefill" pattern from the Anthropic API),
the model returns an empty text block instead of continuing.

Important: the same model **does** continue correctly when the
last assistant turn contains a `tool_use` block followed by a
`tool_result`. Only the simple text-only prefill case is broken.

This pattern is common in agentic workflows that prefill
formatting hints (for example: `assistant: "{\"answer\":"` to
force JSON output).

### 2. Customer impact

- **What does the customer experience?** Customers using the
  prefill technique to constrain output format (e.g. force JSON,
  force a specific opening token) get empty replies. They cannot
  tell whether the prompt is bad or the proxy is broken.
- **Which products / use cases break?** JSON-output pipelines,
  function-calling shims that don't use native tool_use, "complete
  the partial response" features, custom output-formatting
  enforcement.
- **Why is fixing it urgent?** Many production agentic frameworks
  rely on this pattern. Customers may silently fall back to other
  providers if their JSON pipelines start returning empty.

### 3. How to reproduce

**Failing case** — plain text prefill:

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-haiku-4-5-20251001",
    "max_tokens": 40,
    "messages": [
      {"role":"user","content":"Say hello."},
      {"role":"assistant","content":"Hello"}
    ]
  }' | jq '.content'
```

| Result    | Value                                            |
|-----------|--------------------------------------------------|
| Expected  | `[{"type":"text","text":"! How can I help you?"}]` |
| Actual    | `[{"type":"text","text":""}]`                    |

**Working case** — agent loop with tool result (same model):

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-haiku-4-5-20251001",
    "max_tokens": 80,
    "tools": [{"name":"get_temperature","description":"Temperature lookup","input_schema":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}],
    "messages": [
      {"role":"user","content":"Weather in Berlin? Use the tool."},
      {"role":"assistant","content":[{"type":"tool_use","id":"toolu_01","name":"get_temperature","input":{"city":"Berlin"}}]},
      {"role":"user","content":[{"type":"tool_result","tool_use_id":"toolu_01","content":"11C"}]}
    ]
  }' | jq -r '.content[0].text'
# Returns a real answer, e.g. "The current temperature in Berlin is 11C..."
```

### 4. Likely cause

The proxy normalises a plain string assistant content differently
from a content-blocks array. When it sees a plain string as the
last message, it likely treats the conversation as already
complete and asks the upstream for an empty turn.

The Anthropic API itself accepts both forms — string and
content-blocks — for prefill. The bug is in g0i's request
normaliser, not the upstream.

### 5. How to fix

1. In your request normaliser, treat the trailing assistant
   message identically whether `content` is a string or an array
   of blocks.
2. Specifically: do **not** strip or replace a trailing assistant
   text-only turn. Forward it to the upstream as-is.
3. If your normaliser converts string content to blocks, make
   sure it produces `[{"type":"text","text":"<original string>"}]`
   exactly, with no whitespace tweaks.

### 6. How to verify the fix

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-haiku-4-5-20251001",
    "max_tokens":40,
    "messages":[
      {"role":"user","content":"Say hello."},
      {"role":"assistant","content":"Hello"}
    ]
  }' | jq -r '.content[0].text | length'
# Pass: prints a number greater than 0
# Fail: prints 0
```

| Pass Criterion          | Expected Value                              |
|-------------------------|---------------------------------------------|
| HTTP status             | 200                                         |
| `content[0].text` length| greater than 0                              |
| `stop_reason`           | `"end_turn"` or `"max_tokens"`              |

---

## BUG-004 — /v1/chat/completions rejects x-api-key

**Severity:** 5/10 — Medium
**Traffic light:** yellow
**Affected:** `POST /v1/chat/completions` (OpenAI-compatible endpoint)
**Status:** Open

### 1. What is wrong?

The `/v1/messages` endpoint accepts both `Authorization: Bearer`
and `x-api-key` headers for authentication. The
`/v1/chat/completions` endpoint accepts only `Bearer` and rejects
`x-api-key` with HTTP 401. This inconsistency breaks clients that
use a single auth strategy across endpoints.

### 2. Customer impact

- **What does the customer experience?** A SDK that worked on the
  Anthropic-shape endpoint suddenly returns "Could not validate
  credentials" when switched to the OpenAI-shape endpoint. The
  developer thinks the key is wrong and wastes time.
- **Which products / use cases break?** Any tooling that uses the
  OpenAI Chat Completions shape with `x-api-key` (a few SDKs, all
  the OpenAI-compatible UI clients that emit `x-api-key`).
- **Why is fixing it urgent?** Not customer-blocking (Bearer still
  works), but it's a footgun for new integrations. Easy to fix.

### 3. How to reproduce

After the **Setup** block, try the OpenAI-compatible endpoint with
`x-api-key`:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -X POST "$BASE_URL/v1/chat/completions" \
  -H "x-api-key: $G0I_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":32,"messages":[{"role":"user","content":"OK"}]}'
```

Compare with the same endpoint using `Bearer`:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -X POST "$BASE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":32,"messages":[{"role":"user","content":"OK"}]}'
```

| Endpoint                  | Auth header        | Expected | Actual |
|---------------------------|--------------------|:--------:|:------:|
| `POST /v1/messages`       | `Bearer`           | 200      | 200    |
| `POST /v1/messages`       | `x-api-key`        | 200      | 200    |
| `POST /v1/chat/completions` | `Bearer`         | 200      | 200    |
| `POST /v1/chat/completions` | `x-api-key`      | 200      | **401** |

Body returned for the failing case:
`{"detail":"Could not validate credentials"}`

### 4. Likely cause

The auth middleware for `/v1/chat/completions` only inspects the
`Authorization` header. The `/v1/messages` middleware inspects both
headers and accepts whichever is present.

### 5. How to fix

1. In the auth middleware for `/v1/chat/completions`, accept
   `x-api-key` as an alternative to `Authorization: Bearer`.
2. Pseudocode pattern used by `/v1/messages` (apply the same here):
   ```
   key = request.headers.get("authorization", "").removeprefix("Bearer ").strip()
       or request.headers.get("x-api-key", "").strip()
   if not key: return 401
   ```
3. Update the API documentation to state that both auth styles are
   accepted on **all** endpoints.

### 6. How to verify the fix

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -X POST "$BASE_URL/v1/chat/completions" \
  -H "x-api-key: $G0I_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":32,"messages":[{"role":"user","content":"OK"}]}'
# Pass: prints 200
# Fail: prints 401
```

---

## BUG-005 — /v1/models listing inconsistencies

**Severity:** 4/10 — Medium
**Traffic light:** yellow
**Affected:** `GET /v1/models`
**Status:** Open

### 1. What is wrong?

The `/v1/models` endpoint does not match the set of model ids that
are actually callable on `/v1/messages`. Some callable models are
not advertised, some advertised models are not exercised by any
test, and the naming convention (hyphen vs. dot) is mixed.

This breaks any client that uses `/v1/models` for capability
discovery (model pickers, validation layers, observability).

### 2. Customer impact

- **What does the customer experience?** Their model picker shows
  a different list than what actually responds. Models they want
  appear missing; models that appear in the picker may be retired.
- **Which products / use cases break?** Frontend model selectors,
  capability auto-detection, observability dashboards that reconcile
  listed vs. used models.
- **Why is fixing it urgent?** Not blocking, but it erodes trust in
  the API surface. Every developer who notices the drift opens a
  support ticket.

### 3. How to reproduce

After the **Setup** block:

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $G0I_API_KEY" \
  | jq -r '.data[].id' | grep -i claude | sort
```

Compare what is returned to which models actually serve
`/v1/messages`:

| Model id                          | In `/v1/models`? | Serves `/v1/messages`? |
|-----------------------------------|:----------------:|:----------------------:|
| `claude-opus-4-7`                 | yes              | yes                    |
| `claude-opus-4-6`                 | **no**           | 200 with empty content (see BUG-001) |
| `claude-haiku-4-5-20251001`       | **no**           | yes                    |
| `claude-sonnet-4.5`               | yes              | yes                    |
| `claude-haiku-4.5-clean`          | yes              | not tested             |
| `claude-opus-4.5`                 | yes              | not tested             |

Naming inconsistency: hyphen in `claude-opus-4-7` vs. dot in
`claude-sonnet-4.5`. Anthropic's canonical form uses hyphens with
an optional date suffix:
`claude-haiku-4-5-20251001`.

### 4. Likely cause

The `/v1/models` listing comes from a static catalog table that is
not synced with the routing table. Two separate sources of truth
have drifted.

### 5. How to fix

1. Reconcile the catalog with the routing table:
   - Either advertise `claude-opus-4-6` and
     `claude-haiku-4-5-20251001` if they are supposed to be
     callable, **or** remove them from the routing table if they
     are deprecated.
   - Either remove `claude-haiku-4.5-clean` and `claude-opus-4.5`
     if they are not callable, **or** add them to the test matrix.
2. Pick **one** naming style and apply it everywhere. Recommended:
   `claude-<family>-<major>-<minor>` (Anthropic style).
3. Add an automated check that fails CI if `/v1/models` and
   the routing table diverge.

### 6. How to verify the fix

```bash
# Every advertised model should respond to a basic call.
for model in $(curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $G0I_API_KEY" \
                 | jq -r '.data[].id' | grep -i claude); do
  status=$(curl -sS -o /dev/null -w "%{http_code}" -X POST "$BASE_URL/v1/messages" \
    -H "Authorization: Bearer $G0I_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d "{\"model\":\"$model\",\"max_tokens\":16,\"messages\":[{\"role\":\"user\",\"content\":\"OK\"}]}")
  printf '%-40s %s\n' "$model" "$status"
done
# Pass: every line shows status 200
# Fail: any non-200, or any model not listed but callable
```

| Pass Criterion                          | Expected Value          |
|-----------------------------------------|-------------------------|
| Every listed model returns HTTP 200     | yes                     |
| No model serves 200 while not listed    | yes                     |
| Naming convention is uniform            | hyphens, no dots        |

---

## BUG-006 — Latency outlier on opus-4-7 simple prefill (~20 s)

**Severity:** 3/10 — Low
**Traffic light:** green
**Affected:** `claude-opus-4-7` on `POST /v1/messages` with text-only prefill
**Status:** Open

### 1. What is wrong?

The simple prefill case on `claude-opus-4-7` takes around 20
seconds to return. The same model serves a single-turn baseline
in about 4 seconds and a tool-loop prefill in about 2 seconds,
so 20 s is a clear outlier.

The request does eventually succeed (this is why severity is Low),
but the latency is high enough to time out many client SDKs and
agent frameworks.

### 2. Customer impact

- **What does the customer experience?** Calls that should return
  in seconds occasionally hang for 20 seconds. SDKs with default
  timeouts (15–30 s) may treat these as failures and retry, doubling
  cost.
- **Which products / use cases break?** Latency-sensitive UIs (chat,
  IDE assistants), any SDK with default 30 s timeout.
- **Why is fixing it urgent?** Low. The request eventually succeeds,
  so most workloads tolerate it. But it makes Opus look unreliable
  next to Sonnet/Haiku.

### 3. How to reproduce

```bash
time curl -sS -o /dev/null -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-opus-4-7",
    "max_tokens":40,
    "messages":[
      {"role":"user","content":"Say hello."},
      {"role":"assistant","content":"Hello"}
    ]
  }'
```

| Result                | Value                          |
|-----------------------|--------------------------------|
| Expected total time   | ~2 to 5 s                      |
| Actual total time     | ~19 to 21 s                    |
| HTTP status           | 200                            |
| Reply content         | correct (real continuation)    |

### 4. Likely cause

Most likely the request still tries the original Vertex routing
path that returned the old `req_vrtx_*` 400 error. On rejection,
a fallback path is taken, which adds about 17 s.

This is a guess from outside; please check your proxy access log
for two consecutive upstream calls on this request id.

### 5. How to fix

1. Check the proxy access log for the failing case. Look for two
   upstream attempts in sequence — the first should be the slow
   one.
2. If a fallback chain exists, short-circuit it for routes where
   the first hop is known to fail. Cache the "first hop will fail"
   decision per (model, request shape) for at least a few minutes.
3. Alternatively, route `claude-opus-4-7` directly to the working
   upstream and skip the deprecated path entirely.

### 6. How to verify the fix

Run the reproduction five times and take the median:

```bash
for i in 1 2 3 4 5; do
  t=$( { time curl -sS -o /dev/null -X POST "$BASE_URL/v1/messages" \
    -H "Authorization: Bearer $G0I_API_KEY" \
    -H "anthropic-version: 2023-06-01" \
    -H "content-type: application/json" \
    -d '{"model":"claude-opus-4-7","max_tokens":40,"messages":[{"role":"user","content":"Say hello."},{"role":"assistant","content":"Hello"}]}' ; } 2>&1 | awk '/real/ {print $2}')
  echo "run $i: $t"
done
# Pass: every run is below 6 s
# Fail: any run above 10 s
```

| Pass Criterion                | Expected Value          |
|-------------------------------|-------------------------|
| Median latency over 5 runs    | under 5 s               |
| Worst-case latency            | under 10 s              |
| HTTP status                   | 200 every time          |

---

## BUG-007 — Model self-identification is incorrect

**Severity:** 2/10 — Low
**Traffic light:** green
**Affected:** `claude-sonnet-4.5` (most visible), other Claude routes (mild)
**Status:** Open

### 1. What is wrong?

When asked to identify itself, the model returns wrong or
self-contradicting answers. The most visible case is
`claude-sonnet-4.5` claiming to be **"Claude 3.7 Sonnet (Haiku)"**
— a non-existent name that mixes two model families.

This is **not** a model self-knowledge gap to be papered over with
a system-prompt patch. Either (a) the route really is serving the
correct upstream model and the upstream model itself does not yet
know its deployed name (an Anthropic-side training prior issue),
**or** (b) the route is silently serving a different model than
advertised. We need to know which.

### 2. Customer impact

- **What does the customer experience?** When their agent or
  application asks the model "which model are you?", the answer is
  wrong. Self-aware agentic flows that branch on model identity
  break.
- **Which products / use cases break?** Multi-model orchestration
  (where an agent picks between models based on capability),
  observability dashboards that log self-reported model id,
  audit logs in regulated environments.
- **Why is fixing it urgent?** Low. But: if the root cause turns
  out to be (b) — silent model substitution — this is suddenly
  a high-severity trust issue for paying customers.

### 3. How to reproduce

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-sonnet-4.5",
    "max_tokens":120,
    "messages":[{"role":"user","content":"Identify yourself precisely. State (a) your model name, (b) the company that created you, (c) your training data cutoff. No hedging."}]
  }' | jq -r '.content[0].text'
```

| Model           | Self-reported identity                                   |
|-----------------|----------------------------------------------------------|
| opus-4-7        | "Claude — I don't know which specific version"            |
| haiku-4-5-…     | "I am Claude. I don't have access to my exact version"   |
| sonnet-4.5      | **"Claude 3.7 Sonnet (Haiku)"** — wrong, blends two models |

### 4. Likely cause

Two possible explanations, and the difference matters:

1. **Honest training-prior gap.** The upstream model (real
   `claude-sonnet-4.5` from Anthropic) genuinely doesn't know its
   deployed name. This is an Anthropic-side issue, not g0i's.
2. **Silent model substitution.** The route called
   `claude-sonnet-4.5` is actually serving a different model
   (older Sonnet, OpenAI, Gemini, an open-source model …). The
   wrong self-id is the model truthfully reporting what it is.

The other authenticity checks in this report (msg_* prefix,
thinking-signature roundtrip, tokenizer fingerprint) currently
indicate option 1 is more likely. But option 2 is what to rule
out before closing this bug.

### 5. How to fix

**Do not** patch this by injecting an identity string into the
system prompt. Teaching the model to claim a name that may or may
not be the real one is dishonest and makes future diagnosis
impossible.

Instead:

1. Confirm internally which upstream model id is actually called
   when a customer requests `claude-sonnet-4.5`. The answer should
   be the same string, served by Anthropic (direct, Bedrock, or
   Vertex). If it is not — that is the real bug.
2. If the upstream is correct, contact Anthropic. The model's
   self-id is their content layer to fix, not yours.
3. If the upstream is **not** the model the customer asked for,
   route the request to the correct upstream **or** stop offering
   that model id. Do not silently substitute and patch the
   self-report.
4. Document on the dashboard exactly which upstream backend serves
   each advertised model id. Customers paying for Anthropic should
   know they are getting Anthropic.

### 6. How to verify the fix

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $G0I_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-sonnet-4.5",
    "max_tokens":40,
    "messages":[{"role":"user","content":"What is your exact model id? Reply with only the id."}]
  }' | jq -r '.content[0].text' | tr -d ' '
# Pass: prints "claude-sonnet-4.5"
# Fail: prints any other string
```

| Pass Criterion                  | Expected Value           |
|---------------------------------|--------------------------|
| Self-reported id                | matches the requested id |
| No mention of unrelated families | yes                     |

---

## Setup (run once)

Replace `<your-key>` with a valid g0i API key. Keep this terminal open
and reuse it for every reproduction in this report.

```bash
export G0I_API_KEY="<your-key>"
export BASE_URL="https://g0i.shop"
```

Verify the setup:

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $G0I_API_KEY" \
  | jq -r '.data | length'
# Pass: prints a positive integer (we measured 68)
```

---

## Contact

For follow-up traces, additional reproduction cases, or to confirm a
fix, please reach out to **Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `g0i/2026-04-28T0517Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
