# Codemax API Test Report — 2026-04-28

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.

| Field            | Value                                                            |
|------------------|------------------------------------------------------------------|
| Provider         | Codemax                                                          |
| Endpoint         | https://api.codemax.pro                                          |
| Test date (UTC)  | 2026-04-28                                                       |
| Models tested    | claude-opus-4-7, claude-opus-4-7-thinking, claude-sonnet-4-6, claude-haiku-4-5-20251001 |
| Tester           | Sebastian Kuhbach (https://winfuture.de)                         |
| Methodology      | https://github.com/WinFuture23/ai-proxy-feedback                 |
| Report version   | 1.0                                                              |

---

## How to read this report

1. **Quick Status** — score per area and overall grade.
2. **Bug Summary** — one row per problem, sorted by severity.
3. **Each bug** has five fixed sections:
   - What is wrong (plain English)
   - How to reproduce (copy and paste)
   - Likely cause (short engineering note)
   - How to fix (concrete steps)
   - How to verify the fix (copy and paste, with pass criterion)

The reproduction commands assume you ran the **Setup** block once
(see end of report).

---

## Quick Status

| Category               | Score (0–100) | Grade | Trend             | Status                                |
|------------------------|--------------:|:-----:|:-----------------:|----------------------------------------|
| Functional Suite       | 64            | D     | Δ −22 vs 2026-04-26 | regression on basic_text + streaming + web_search |
| Authentication         | 100           | A     | Δ 0               | all auth combinations return 200       |
| Model Authenticity     | 100           | A     | Δ 0               | real Anthropic backend confirmed       |
| Streaming              | 0             | F     | Δ −100            | no text deltas on any model            |
| Web Search (server)    | 0             | F     | Δ 0               | tool emits as plain text, not blocks   |
| MCP Tools              | 50            | D     | Δ 0               | web_search OK, understand_image 404    |
| Hidden System Prompt   | 50            | D     | Δ 0               | partial leak via prompt-injection probe |
| **Overall**            | **52**        | **D** | Δ −24             | **degraded since previous run**        |

Scoring rules: see [`../_template/scoring.md`](../_template/scoring.md).

---

## Bug Summary

| Bug ID  | Title                                                            | Severity | Affected                          | Status |
|---------|------------------------------------------------------------------|:--------:|-----------------------------------|--------|
| BUG-001 | Streaming returns no text deltas (chars = 0)                     | 9        | all 4 models, `stream: true`      | Open   |
| BUG-002 | `web_search_20250305` tool emits as plain text instead of blocks | 8        | all 4 models                      | Open   |
| BUG-003 | `basic_text` replies truncated to first character or empty       | 7        | opus-4-7, sonnet-4-6, haiku-4-5   | Open   |
| BUG-004 | `image_vision` returns empty on `claude-sonnet-4-6`              | 5        | claude-sonnet-4-6                 | Open   |
| BUG-005 | `claude-opus-4-7-thinking` not advertised in /v1/models          | 3        | /v1/models                        | Open   |
| BUG-006 | MCP `understand_image` returns HTTP 404                          | 5        | codemax-mcp /v1/coding_plan/vlm   | Open   |

---

## BUG-001 — Streaming returns no text deltas (chars = 0)

**Severity:** 9/10 — Critical
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

### 2. How to reproduce

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

### 3. Likely cause

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

### 4. How to fix

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

### 5. How to verify the fix

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

### 2. How to reproduce

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

### 3. Likely cause

The proxy is intercepting the upstream response, finding the
structured `server_tool_use` blocks, and serializing them as a
human-readable text dump (`[TOOL_CALL]\n{tool => ...}`) before
forwarding. A previous version probably stripped them entirely;
this newer version stringifies them.

### 4. How to fix

1. In your response post-processing, do not mutate
   `server_tool_use` or `web_search_tool_result` content blocks.
   Forward them unchanged.
2. If you need to log tool calls for observability, log them
   server-side. Do not put a debug dump into `content`.
3. If the upstream is not actually producing `server_tool_use`
   blocks (some non-Anthropic backends won't), then either return
   HTTP 400 for `web_search_20250305` requests or proxy them to
   a backend that does.

### 5. How to verify the fix

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

### 2. How to reproduce

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

### 3. Likely cause

Same root cause as BUG-001 with extra symptom: the proxy forces
extended thinking on by default. The model spends the entire
`max_tokens` budget on hidden thinking, leaving 0 or 1 tokens for
the visible text.

`claude-opus-4-7-thinking` is unaffected because it has thinking
enabled explicitly with a separate budget — the visible text gets
its own allocation.

### 4. How to fix

1. Stop forcing `thinking: enabled` on requests that did not ask
   for it. Default to `thinking: disabled` (or no field) and only
   enable thinking when the client opts in.
2. If you want to keep thinking on by default for product reasons,
   reserve a portion of `max_tokens` for the visible text. A
   reasonable rule: cap thinking at `min(budget_tokens,
   max_tokens / 2)` so at least half the budget is left for text.
3. Document the policy on the dashboard so clients know what
   `max_tokens` actually means on Codemax.

### 5. How to verify the fix

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
**Affected:** `claude-sonnet-4-6`
**Status:** Open

### 1. What is wrong?

When `claude-sonnet-4-6` is asked about a small base64-encoded
image (32×32 solid red PNG, prompt `"What colour?"`), the response
is HTTP 200 with an empty `text` block. The other three models
correctly answer `"Red"`. The image input itself is the same byte
sequence in all four cases.

### 2. How to reproduce

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

### 3. Likely cause

Same forced-thinking story as BUG-003. `claude-sonnet-4-6`
appears to spend its entire `max_tokens: 512` budget on hidden
thinking before reaching the visible text, leaving no room for
the answer.

### 4. How to fix

Same as BUG-003: stop forcing thinking on by default, or reserve
a portion of `max_tokens` for visible text. Once BUG-003 is
resolved, this bug should also disappear — please verify with the
command in step 5.

### 5. How to verify the fix

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
**Affected:** `GET /v1/models`
**Status:** Open

### 1. What is wrong?

`claude-opus-4-7-thinking` accepts requests on `/v1/messages` and
returns valid responses (it's the only model not affected by
BUG-001 and BUG-003 in this report), but it is not listed in the
output of `/v1/models`. Clients that use `/v1/models` for model
discovery will not see it.

### 2. How to reproduce

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

### 3. Likely cause

`claude-opus-4-7-thinking` is treated internally as a routing
alias rather than as a first-class model. The catalog generator
walks the routing config but skips alias entries.

### 4. How to fix

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

### 5. How to verify the fix

```bash
curl -sS "$BASE_URL/v1/models" -H "Authorization: Bearer $CODEMAX_API_KEY" \
  | jq -r '.data[].id' | grep -c thinking
# Pass after Option A: prints 1 (the thinking variant is listed)
# Pass after Option B: prints 0 AND a request with the thinking model id returns success with response.model="claude-opus-4-7"
```

---

## BUG-006 — MCP `understand_image` returns HTTP 404

**Severity:** 5/10 — Medium
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

### 2. How to reproduce

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

### 3. Likely cause

The MCP server posts to `https://api.codemax.pro/v1/coding_plan/vlm`,
which does not exist. Either the endpoint was never deployed, or
its path changed and the MCP client was not updated.

### 4. How to fix

1. Confirm the intended VLM endpoint for `understand_image`. Is
   it `/v1/messages` with an image content block? A separate
   vision endpoint? A future-planned route?
2. Update `codemax-mcp` (the npm package) to call the correct
   endpoint with the correct request shape. Bump the version.
3. Alternatively, route `/v1/coding_plan/vlm` to `/v1/messages`
   with an Anthropic-shaped image payload using the requested
   model (`claude-opus-4-7` works for vision in our tests).

### 5. How to verify the fix

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
fix:

- **Sebastian Kuhbach** — https://winfuture.de
- Report repo: https://github.com/WinFuture23/ai-proxy-feedback
- Raw test output for this run: `codemax/2026-04-28/results.txt`
