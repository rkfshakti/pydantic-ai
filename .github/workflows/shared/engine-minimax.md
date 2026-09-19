---
# Shared runtime + engine config for the Pydantic AI gh-aw shim (MiniMax backend).
#
# Registers as the built-in `claude` engine and only overrides `command`, so
# gh-aw runs its full Claude proxy + credential-injection machinery.
#
# ANTHROPIC_BASE_URL MUST be a compile-time literal (not a ${{ vars.* }}
# expression): gh-aw derives the api-proxy target host AND the
# `--anthropic-api-base-path` from its parsed URL path at compile time. With a
# vars expression the path can't be parsed, so the proxy drops the `/anthropic`
# prefix and the upstream returns 404. Only ANTHROPIC_API_KEY stays a secret
# (injected by the AWF api-proxy, excluded from the agent container).
# MiniMax exposes an Anthropic-compatible API at https://api.minimax.io/anthropic.
#
# The checked-out workspace is mounted no-exec in the AWF sandbox, so a
# pre-step stages a launcher in gh-aw's exec-able /tmp/gh-aw/bin that runs
# `uv run --script` against the workspace harness.
#
# Required repo variable:
#   GH_AW_MODEL — model name forwarded as `--model <name>` to the harness.
# Required secret:
#   MINIMAX_API_KEY — API key injected by the AWF api-proxy.
#
# Usage:
#   imports:
#     - shared/engine-minimax.md
runtimes:
  uv: {}
# No `models.providers` pricing overlay for `MiniMax-M3`, deliberately. gh-aw compiles
# that block into `apiProxy.providers`, which is the AWF API proxy's *AI-credits
# accounting* table rather than a reporting field: pricing a model there starts charging
# it credits. AWF then enforces a hard cap of 10,000 credits per run that nothing in this
# frontmatter can lift — `max-ai-credits` only omits gh-aw's own budget, and the schema
# types it `exclusiveMinimum: 0`. One ordinary request costs tens of thousands, so every
# agent job 403s on its first call with `ai_credits_limit_exceeded`.
#
# Nothing here needs that overlay: the weekly spend report reads token counts from each
# run's `agent_usage.json` artifact, never a pricing table.
max-ai-credits: -1
engine:
  id: copilot
  # BYOK: route Copilot CLI inference to Ollama Cloud's OpenAI-compatible
  # endpoint (minimax-m3, verified HTTP 200 with the repo's OLLAMA_API_KEY).
  # BYOK needs no GitHub Copilot subscription — the CLI authenticates to the
  # provider directly. COPILOT_PROVIDER_BASE_URL is a compile-time literal
  # so gh-aw adds ollama.com to the AWF allow-list automatically.
  model: minimax-m3
  env:
    COPILOT_PROVIDER_BASE_URL: https://ollama.com/v1
    COPILOT_PROVIDER_API_KEY: ${{ secrets.OLLAMA_API_KEY }}
    # The Ollama BYOK engine is stateless; an outer retry repeats the whole task.
    GH_AW_HARNESS_MAX_RETRIES: "0"
safe-outputs:
  threat-detection:
    # Detection has an independent budget and the same unknown-model constraint.
    max-ai-credits: -1
    # Detection uses the stateful CLI, so it retains normal recovery.
    engine:
      id: copilot
      env:
        GH_AW_HARNESS_MAX_RETRIES: "3"
---
