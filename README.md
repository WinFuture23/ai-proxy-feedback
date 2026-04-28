# AI Proxy Test Reports

© 2026 Sebastian Kuhbach of WinFuture.de — All rights reserved.

Black-box test results and bug reports for Anthropic-compatible AI API proxies.

Every bug report follows a fixed template — see
[`_template/bug-report-template.md`](_template/bug-report-template.md).
Scoring rules are documented in
[`_template/scoring.md`](_template/scoring.md).

Each provider has its own subfolder. Reports are dated and immutable —
new test runs land in a new dated subfolder rather than overwriting the
previous one, so the regression history stays auditable.

## Structure

```
g0i/
  YYYY-MM-DD/
    report.md      Provider-facing bug report (issues, repros, fixes)
    results.txt    Raw test output (sanitized — no API keys)

codemax/
  YYYY-MM-DD/
    ...
```

## Methodology

The full test suite is intentionally provider-agnostic: every script
takes `--api-key`, `--base-url`, and `--models` so the same battery
runs against any Anthropic-compatible proxy.

Test categories:

1. **Functional suite** — list_models, basic_text, system_prompt,
   multi_turn, tool_use, tool_roundtrip, image_vision, streaming_tps,
   web_search.
2. **Auth diagnostics** — Bearer vs x-api-key on /v1/messages and
   /v1/chat/completions, with and without auth.
3. **Authenticity probe** — msg_* IDs, response.model, self-ID,
   thinking-block signature roundtrip, tokenizer fingerprint.
4. **Prompt-leak probe** — repeat_above, dump_system, translate_system,
   first_500_chars, compliance_audit. Detects hidden system prompts
   injected by the proxy.
5. **Prefill regression** — baseline_user, prefill_simple,
   prefill_agent_loop. Reproduces the
   "conversation must end with user message" bug class on Vertex-routed
   Anthropic deployments.
6. **Thinking-enforcement probe** — variants without/with disabled/with
   enabled thinking field. Detects whether the proxy honours the
   client's `thinking` parameter.

## Reproducing locally

The reproduction commands in each report use `$PROVIDER_API_KEY` as a
placeholder. Set it in your shell, never paste it into a request that
gets logged.

```bash
git clone https://github.com/WinFuture23/ai-proxy-feedback
cd ai-proxy-feedback

# Each report's curl snippets are self-contained and runnable
# given a $PROVIDER_API_KEY env var.
```

## Source

This repo is a curated, public-facing mirror. The full test suite
(and per-provider workflows that produce the result files) live in a
separate private repository. Bug reports and sanitized result files
are mirrored here automatically on each new run.
