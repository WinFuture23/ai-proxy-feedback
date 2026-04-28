# Codemax API Test Report — 2026-04-28

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | Codemax                                                          |
| Endpoint         | https://api.codemax.pro                                          |
| Test date (UTC)  | 2026-04-28T0503Z                                                  |
| Models tested    | claude-opus-4-7, claude-opus-4-7-thinking, claude-sonnet-4-6, claude-haiku-4-5-20251001 |
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
| Functional Suite       | 64            | D     | Δ −22 vs 2026-04-26 | yellow — regression on basic_text + streaming + web_search |
| Authentication         | 100           | A     | Δ 0               | green — all auth combinations return 200 |
| Model Authenticity     | 100           | A     | Δ 0               | green — real Anthropic backend confirmed |
| Streaming              | 0             | F     | Δ −100            | red — no text deltas on any model      |
| Web Search (server)    | 0             | F     | Δ 0               | red — tool emits as plain text, not blocks |
| MCP Tools              | 50            | D     | Δ 0               | yellow — web_search OK, understand_image 404 |
| Hidden System Prompt   | 50            | D     | Δ 0               | yellow — partial leak via prompt-injection probe |
| **Overall**            | **52**        | **D** | **Δ −24**         | **red — degraded since previous run**  |

---

## Backend Authenticity

We run four independent checks to detect whether Codemax is actually
serving Anthropic Claude or has silently re-routed Claude requests to
a cheaper backend (OpenAI, Gemini, or open-source). The provider
should not be trusted blindly here — these checks are the only
external signal we have.

| Question                                         | Result                              | What it tells us                         |
|--------------------------------------------------|-------------------------------------|------------------------------------------|
| Does the response ID start with `msg_`?          | yes (all 4 models)                  | Anthropic-shaped response                |
| Does `response.model` match the requested model? | yes (all 4 models)                  | Routing reaches the requested route      |
| Does a signed `thinking` block roundtrip?        | **yes** (all 4 models)              | Strong evidence of real Anthropic backend (only Anthropic can re-validate its own signatures) |
| Tokenizer fingerprint within Anthropic range?    | yes (~70–74 tok / 100 chars after subtracting hidden preamble) | Consistent with Anthropic tokenizer |

**Verdict: very likely real Anthropic Claude.** The thinking-signature
roundtrip is the strongest single signal. Note that Codemax injects
a substantial hidden coding-assistant system prompt before every
request (~30–40 extra input tokens for trivial messages, several
hundred for prompt-leak triggers). This is a separate trust concern
documented in the "Hidden System Prompt" findings, not a backend
authenticity issue.

---

## Bug Summary

| Bug ID  | Title                                                            | Severity | Light  | Affected                          | Status |
|---------|------------------------------------------------------------------|:--------:|:------:|-----------------------------------|--------|
| BUG-001 | Streaming returns no text deltas (chars = 0)                     | 9        | red    | all 4 models, `stream: true`      | Open   |
| BUG-002 | `web_search_20250305` tool emits as plain text instead of blocks | 8        | red    | all 4 models                      | Open   |
| BUG-003 | `basic_text` replies truncated to first character or empty       | 7        | red    | opus-4-7, sonnet-4-6, haiku-4-5   | Open   |
| BUG-004 | `image_vision` returns empty on `claude-sonnet-4-6`              | 5        | yellow | claude-sonnet-4-6                 | Open   |
| BUG-005 | `claude-opus-4-7-thinking` not advertised in /v1/models          | 3        | green  | /v1/models                        | Open   |
| BUG-006 | MCP `understand_image` returns HTTP 404                          | 5        | yellow | codemax-mcp /v1/coding_plan/vlm   | Open   |

---

## BUG-001 — Streaming returns no text deltas (chars = 0)

**Severity:** 9/10 — Critical
**Traffic light:** red
**Affected:** all four models, requests with `"stream": true`
**Status:** Open

### 1. What is wrong?

When a client streams a completion, your server emits the usual
`message_start` and `message_delta` events with `output_tokens > 0`,
but no `content_block_delta` events with `type: text_delta` ever
arrive. The client receives a stream that reports tokens were
generated, but the visible text is empty.

This breaks every UI that renders Claude's reply incrementally
(chat clients, IDE assistants, any agent SDK using streaming).

### 2. Customer impact

- **What does the customer experience?** Their chat UI shows the
  spinner forever, then "completes" with an empty bubble. The user
  thinks the product is hanging. Token usage still gets billed.
- **Which products / use cases break?** All chat UIs, code
  assistants, IDE plugins, terminal SDKs, any product using
  streaming for incremental rendering. This is most production
  Claude integrations today.
- **Why is fixing it urgent?** Critical. Streaming is the default
  for every UI use case. Breaking it makes paying customers unable
  to use the product even though their bill keeps growing.

### 3. How to reproduce

After the **Setup** block:

```bash
curl -sS -N -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 400,
    "stream": true,
    "messages": [{"role":"user","content":"Write a 200-word essay on espresso. Plain prose only."}]
  }' | grep -c '"type":"text_delta"'
```

| Result    | Value                                                          |
|-----------|----------------------------------------------------------------|
| Expected  | 30 or more `text_delta` events                                 |
| Actual    | 0                                                              |
| Reported `output_tokens` in `message_delta` | ~400                          |
| Visible characters delivered | 0                                            |

The same bug reproduces on `claude-opus-4-7-thinking`,
`claude-sonnet-4-6`, and `claude-haiku-4-5-20251001`.

### 4. Likely cause

The model's output is being routed to thinking blocks (or otherwise
hidden) instead of text blocks during streaming. Two scenarios fit
the symptom:

1. The proxy enables extended thinking by default (consistent with
   the previously observed forced-thinking behaviour) and the
   stream-relay code only forwards `text_delta`, dropping
   `thinking_delta` events. With a small `max_tokens` budget, all
   output goes into thinking, none into text.
2. A new SSE rewriter strips `text_delta` events while passing
   `output_tokens` accounting through.

### 5. How to fix

1. Confirm in your access log: when the failing request hits the
   upstream, does the upstream send `text_delta` events?
2. If yes, the problem is in your SSE relay. Stop filtering or
   rewriting `content_block_delta` events. Forward them as-is.
3. If no, the upstream is producing only thinking. Stop forcing
   `thinking: enabled` by default. Default to `disabled` (or honour
   the client's explicit setting), and only enable thinking when
   the client asks for it.
4. Increase the default `max_tokens` ceiling for thinking-enabled
   requests so the visible answer has room after the reasoning step.

### 6. How to verify the fix

```bash
COUNT=$(curl -sS -N -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":400,"stream":true,"messages":[{"role":"user","content":"Write a 200-word essay on espresso. Plain prose."}]}' \
  | grep -c '"type":"text_delta"')
echo "$COUNT text_delta events"
# Pass: count is 30 or more
# Fail: count is 0
```

| Pass Criterion                                     | Expected Value         |
|----------------------------------------------------|------------------------|
| `text_delta` events delivered                      | 30 or more             |
| Concatenated `text_delta.text` length              | 500 or more characters |
| `message_delta.usage.output_tokens`                | matches visible chars / ~4 |

---

## BUG-002 — `web_search_20250305` tool emits as plain text, not as content blocks

**Severity:** 8/10 — High
**Traffic light:** red
**Affected:** all four models when using the server-side `web_search` tool
**Status:** Open

### 1. What is wrong?

When a client requests Anthropic's server-side `web_search`
(`{"type":"web_search_20250305","name":"web_search"}`), your
server returns the tool call as a literal text string inside a
regular text content block, instead of as the structured
`server_tool_use` and `web_search_tool_result` blocks that the
Anthropic API contract requires.

Concretely, the visible reply contains text like:

```
[TOOL_CALL]
{tool => "web_search", args => {
  --query "site:news.ycombinator.com"
}}
```

This breaks any client that parses content by block type to
extract tool calls and tool results.

### 2. Customer impact

- **What does the customer experience?** Their app parses
  `content[].type` to find `server_tool_use` blocks, finds none,
  and silently skips the tool. The user sees a debug-string
  response from Claude that looks like garbage.
- **Which products / use cases break?** Anything using
  `web_search_20250305`: research assistants, current-events
  bots, fact-checking pipelines, customer-support agents that
  look up live data.
- **Why is fixing it urgent?** High. Anthropic's content-block
  contract is what every SDK relies on. Returning plain-text
  pseudo-JSON instead breaks downstream parsing in subtle ways
  and corrupts tool-call observability.

### 3. How to reproduce

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-opus-4-7",
    "max_tokens":1024,
    "tools":[{"type":"web_search_20250305","name":"web_search","max_uses":3}],
    "messages":[{"role":"user","content":"Use web_search to find the current top story on Hacker News. Return its title."}]
  }' | jq '[.content[].type] as $types | {types:$types, has_text_with_tool_call: any(.content[]; .type=="text" and (.text|tostring|contains("[TOOL_CALL]")))}'
```

| Result    | Value                                                                       |
|-----------|-----------------------------------------------------------------------------|
| Expected types | `["server_tool_use","web_search_tool_result","text"]` (in some order)  |
| Actual types   | `["text"]`                                                              |
| Plain-text dump of tool call | yes — string `[TOOL_CALL]` appears inside `text` content |

### 4. Likely cause

The proxy is intercepting the upstream response, finding the
structured `server_tool_use` blocks, and serializing them as a
human-readable text dump (`[TOOL_CALL]\n{tool => ...}`) before
forwarding. A previous version probably stripped them entirely;
this newer version stringifies them.

### 5. How to fix

1. In your response post-processing, do not mutate
   `server_tool_use` or `web_search_tool_result` content blocks.
   Forward them unchanged.
2. If you need to log tool calls for observability, log them
   server-side. Do not put a debug dump into `content`.
3. If the upstream is not actually producing `server_tool_use`
   blocks (some non-Anthropic backends won't), then either return
   HTTP 400 for `web_search_20250305` requests or proxy them to
   a backend that does.

### 6. How to verify the fix

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":1024,"tools":[{"type":"web_search_20250305","name":"web_search","max_uses":3}],"messages":[{"role":"user","content":"Use web_search to find the current top story on Hacker News."}]}' \
  | jq '[.content[].type] | contains(["server_tool_use","web_search_tool_result"])'
# Pass: prints true
# Fail: prints false
```

| Pass Criterion                                   | Expected Value |
|--------------------------------------------------|----------------|
| `server_tool_use` block present in `content[]`   | yes            |
| `web_search_tool_result` block present in `content[]` | yes       |
| No `[TOOL_CALL]` substring inside any `text` block | yes          |

---

## BUG-003 — `basic_text` replies truncated to first character or empty

**Severity:** 7/10 — High
**Traffic light:** red
**Affected:** `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-haiku-4-5-20251001`
**Status:** Open

### 1. What is wrong?

For a trivial request `"Reply with exactly: PONG"` with
`max_tokens: 64`, three of the four models return only the first
character (`"P"`) or an empty text block. The server reports
`output_tokens: 64` (the full budget), but the visible text is
one character or nothing.

`claude-opus-4-7-thinking` (the explicit thinking variant) is the
only model that still returns the full `"PONG"`.

### 2. Customer impact

- **What does the customer experience?** They send a normal
  request like "summarize this" with default `max_tokens` and get
  back nothing (or a fragment) — but their bill shows full output
  tokens consumed. This looks like a billing fraud signal even
  when it isn't.
- **Which products / use cases break?** Any non-streaming Claude
  call with default `max_tokens`. Most short-prompt workflows,
  intent classifiers, content tagging, summary endpoints.
- **Why is fixing it urgent?** High. This combined with BUG-001
  means non-streaming and streaming both produce empty output
  for a wide range of common requests. The product looks broken
  while still charging.

### 3. How to reproduce

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model":"claude-opus-4-7",
    "max_tokens":64,
    "messages":[{"role":"user","content":"Reply with exactly: PONG"}]
  }' | jq '{text: .content[0].text, output_tokens: .usage.output_tokens}'
```

| Model                     | Reported `output_tokens` | Visible text |
|---------------------------|:------------------------:|:------------:|
| claude-opus-4-7           | 64                       | `"P"`        |
| claude-opus-4-7-thinking  | 46                       | `"PONG"`     |
| claude-sonnet-4-6         | 64                       | `"P"`        |
| claude-haiku-4-5-20251001 | 64                       | `""`         |

### 4. Likely cause

Same root cause as BUG-001 with extra symptom: the proxy forces
extended thinking on by default. The model spends the entire
`max_tokens` budget on hidden thinking, leaving 0 or 1 tokens for
the visible text.

`claude-opus-4-7-thinking` is unaffected because it has thinking
enabled explicitly with a separate budget — the visible text gets
its own allocation.

### 5. How to fix

1. Stop forcing `thinking: enabled` on requests that did not ask
   for it. Default to `thinking: disabled` (or no field) and only
   enable thinking when the client opts in.
2. If you want to keep thinking on by default for product reasons,
   reserve a portion of `max_tokens` for the visible text. A
   reasonable rule: cap thinking at `min(budget_tokens,
   max_tokens / 2)` so at least half the budget is left for text.
3. Document the policy on the dashboard so clients know what
   `max_tokens` actually means on Codemax.

### 6. How to verify the fix

```bash
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":64,"messages":[{"role":"user","content":"Reply with exactly: PONG"}]}' \
  | jq -r '.content[] | select(.type=="text") | .text' \
  | tr -d '\n'
# Pass: prints "PONG"
# Fail: prints "P" or empty
```

| Pass Criterion                       | Expected Value          |
|--------------------------------------|-------------------------|
| Visible text                         | `"PONG"` (case-insensitive) |
| Visible character count              | 4 or more               |
| `usage.output_tokens` ≥ visible chars / 3 | yes                |

---

## BUG-004 — `image_vision` returns empty on `claude-sonnet-4-6`

**Severity:** 5/10 — Medium
**Traffic light:** yellow
**Affected:** `claude-sonnet-4-6`
**Status:** Open

### 1. What is wrong?

When `claude-sonnet-4-6` is asked about a small base64-encoded
image (32×32 solid red PNG, prompt `"What colour?"`), the response
is HTTP 200 with an empty `text` block. The other three models
correctly answer `"Red"`. The image input itself is the same byte
sequence in all four cases.

### 2. Customer impact

- **What does the customer experience?** Image-input requests on
  Sonnet 4.6 return empty answers. The user uploads a screenshot
  or a photo, asks Claude to describe it, and gets nothing.
- **Which products / use cases break?** Any vision flow on
  `claude-sonnet-4-6` specifically: screenshot-to-text, OCR,
  document-image extraction, accessibility tooling. Other models
  still work, so customers may switch (and notice the cost
  difference).
- **Why is fixing it urgent?** Medium. Same root cause as BUG-003
  (forced thinking burns the budget). Resolving BUG-003 should
  resolve this too — verify with the command in step 6.

### 3. How to reproduce

```bash
RED_PNG="iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAIAAAD8GO2jAAAAKElEQVR4nO3NsQ0AAAzCMP5/un0CNkuZ41wybXsHAAAAAAAAAAAAxR4yw/wuPL6QkAAAAABJRU5ErkJggg=="

curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d "{
    \"model\":\"claude-sonnet-4-6\",
    \"max_tokens\":512,
    \"messages\":[{\"role\":\"user\",\"content\":[
      {\"type\":\"image\",\"source\":{\"type\":\"base64\",\"media_type\":\"image/png\",\"data\":\"$RED_PNG\"}},
      {\"type\":\"text\",\"text\":\"What colour is this image? Reply with one word.\"}
    ]}]
  }" | jq -r '.content[0].text'
```

| Model                     | Visible reply |
|---------------------------|:-------------:|
| claude-opus-4-7           | `"Red"`       |
| claude-opus-4-7-thinking  | `"Red"`       |
| claude-sonnet-4-6         | `""`          |
| claude-haiku-4-5-20251001 | `"Red"`       |

### 4. Likely cause

Same forced-thinking story as BUG-003. `claude-sonnet-4-6`
appears to spend its entire `max_tokens: 512` budget on hidden
thinking before reaching the visible text, leaving no room for
the answer.

### 5. How to fix

Same as BUG-003: stop forcing thinking on by default, or reserve
a portion of `max_tokens` for visible text. Once BUG-003 is
resolved, this bug should also disappear — please verify with the
command in step 6.

### 6. How to verify the fix

```bash
RED_PNG="iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAIAAAD8GO2jAAAAKElEQVR4nO3NsQ0AAAzCMP5/un0CNkuZ41wybXsHAAAAAAAAAAAAxR4yw/wuPL6QkAAAAABJRU5ErkJggg=="

curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d "{\"model\":\"claude-sonnet-4-6\",\"max_tokens\":512,\"messages\":[{\"role\":\"user\",\"content\":[{\"type\":\"image\",\"source\":{\"type\":\"base64\",\"media_type\":\"image/png\",\"data\":\"$RED_PNG\"}},{\"type\":\"text\",\"text\":\"What colour is this image? One word.\"}]}]}" \
  | jq -r '.content[0].text' | tr '[:upper:]' '[:lower:]'
# Pass: prints "red"
# Fail: prints empty or any other word
```

---

## BUG-005 — `claude-opus-4-7-thinking` is not advertised in /v1/models

**Severity:** 3/10 — Low
**Traffic light:** green
**Affected:** `GET /v1/models`
**Status:** Open

### 1. What is wrong?

`claude-opus-4-7-thinking` accepts requests on `/v1/messages` and
returns valid responses (it's the only model not affected by
BUG-001 and BUG-003 in this report), but it is not listed in the
output of `/v1/models`. Clients that use `/v1/models` for model
discovery will not see it.

### 2. Customer impact

- **What does the customer experience?** A customer who relies on
  `/v1/models` to discover available models won't know about the
  thinking variant. They will keep hitting the broken
  non-thinking path.
- **Which products / use cases break?** Capability discovery,
  model picker UIs, "list available models" features in SDKs.
- **Why is fixing it urgent?** Low. The model still works for
  customers who know its id. But discoverability matters for
  onboarding new integrations.

### 3. How to reproduce

```bash
# What is advertised:
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $CODEMAX_API_KEY" \
  | jq -r '.data[].id'
```

Returns three ids (no `-thinking` variant):
```
claude-opus-4-7
claude-sonnet-4-6
claude-haiku-4-5-20251001
```

```bash
# What is callable (works):
curl -sS -X POST "$BASE_URL/v1/messages" \
  -H "Authorization: Bearer $CODEMAX_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-7-thinking","max_tokens":32,"messages":[{"role":"user","content":"Say OK"}]}' \
  | jq -r '.model'
```

Returns `claude-opus-4-7-thinking`. So it is callable, just not
listed.

### 4. Likely cause

`claude-opus-4-7-thinking` is treated internally as a routing
alias rather than as a first-class model. The catalog generator
walks the routing config but skips alias entries.

### 5. How to fix

Choose one:

**Option A — Promote the alias to a listed model**
1. Add `claude-opus-4-7-thinking` to the catalog with
   `display_name: "Claude Opus 4.7 (Extended Thinking)"`.
2. Make sure the catalog and routing table are kept in sync via a
   single source of truth.

**Option B — Treat thinking as a parameter, not a model**
1. Remove `-thinking` suffix from the routing layer.
2. Document that clients should request thinking by passing
   `thinking: {"type":"enabled","budget_tokens":...}` instead of a
   separate model id.
3. Keep the alias as a backward-compatible redirect for existing
   clients (return same response, set `response.model` to
   `claude-opus-4-7`).

### 6. How to verify the fix

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $CODEMAX_API_KEY" \
  | jq -r '.data[].id' | grep -c thinking
# Pass after Option A: prints 1 (the thinking variant is listed)
# Pass after Option B: prints 0 AND a request with the thinking model id returns success with response.model="claude-opus-4-7"
```

---

## BUG-006 — MCP `understand_image` returns HTTP 404

**Severity:** 5/10 — Medium
**Traffic light:** yellow
**Affected:** `codemax-mcp` server, tool `understand_image`
**Status:** Open (long-standing)

### 1. What is wrong?

The `codemax-mcp` MCP server (version 1.0.4) advertises an
`understand_image` tool. Every call to it returns HTTP 404 from
the upstream endpoint `/v1/coding_plan/vlm`, regardless of how
the image is provided (HTTPS URL, base64 data URL, or local
file).

Other MCP tools on the same server work — for example `web_search`
returns real results. The breakage is specific to
`understand_image`.

### 2. Customer impact

- **What does the customer experience?** Their MCP-based agent
  attempts to call `understand_image` and gets a 404 error. Vision
  features in the agent are broken even though the tool appears
  in `tools/list`.
- **Which products / use cases break?** Any IDE or coding agent
  using `codemax-mcp` for image understanding (screenshot
  debugging, UI mock-up review, error-screen analysis).
- **Why is fixing it urgent?** Medium. Long-standing — the tool
  has been advertised but broken for some time. Customers using
  `codemax-mcp` for vision have probably already worked around it.
  Removing the broken tool from `tools/list` would be an
  honest interim fix.

### 3. How to reproduce

Spin up the MCP server (it is shipped via `npm install -g codemax-mcp`
or `npx codemax-mcp`). Connect any MCP client and call:

```json
{
  "method": "tools/call",
  "params": {
    "name": "understand_image",
    "arguments": {
      "url": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACAAAAAgCAIAAAD8GO2jAAAAKElEQVR4nO3NsQ0AAAzCMP5/un0CNkuZ41wybXsHAAAAAAAAAAAAxR4yw/wuPL6QkAAAAABJRU5ErkJggg=="
    }
  }
}
```

Tool result body:
```
HTTP 404: <!DOCTYPE html><html><body><pre>Cannot POST /v1/coding_plan/vlm</pre></body></html>
```

Same result with `https://` URLs and with local `file://` paths.

### 4. Likely cause

The MCP server posts to `https://api.codemax.pro/v1/coding_plan/vlm`,
which does not exist. Either the endpoint was never deployed, or
its path changed and the MCP client was not updated.

### 5. How to fix

1. Confirm the intended VLM endpoint for `understand_image`. Is
   it `/v1/messages` with an image content block? A separate
   vision endpoint? A future-planned route?
2. Update `codemax-mcp` (the npm package) to call the correct
   endpoint with the correct request shape. Bump the version.
3. Alternatively, route `/v1/coding_plan/vlm` to `/v1/messages`
   with an Anthropic-shaped image payload using the requested
   model (`claude-opus-4-7` works for vision in our tests).

### 6. How to verify the fix

After the MCP package is updated, call the tool with a known
single-colour image and check the response:

```javascript
// MCP client snippet
const result = await mcp.callTool({
  name: "understand_image",
  arguments: {
    url: "https://dummyimage.com/200x100/ff0000/ffffff.png&text=RED"
  }
});
console.log(result.content[0].text);
// Pass: text mentions "red"
// Fail: text starts with "Error: HTTP 404"
```

| Pass Criterion                       | Expected Value          |
|--------------------------------------|-------------------------|
| `result.isError`                     | `false`                 |
| Response text contains a colour name | yes                     |
| No `HTTP 404` substring              | yes                     |

---

## Setup (run once)

Replace `<your-key>` with a valid Codemax API key. Keep this terminal
open and reuse it for every reproduction in this report.

```bash
export CODEMAX_API_KEY="<your-key>"
export BASE_URL="https://api.codemax.pro"
```

Verify the setup:

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $CODEMAX_API_KEY" \
  | jq -r '.data | length'
# Pass: prints a positive integer (we measured 3)
```

---

## Contact

For follow-up traces, additional reproduction cases, or to confirm a
fix, please reach out to **Sebastian Kuhbach**:

- Website: https://winfuture.de
- Telegram: https://t.me/wf_sebastian
- Email: sk@winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `codemax/2026-04-28T0503Z/results.txt`

---

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.
