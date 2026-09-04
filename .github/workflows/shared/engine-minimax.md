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
# MiniMax pricing for run-cost reporting, in dollars per 1M tokens.
models:
  providers:
    anthropic:
      models:
        MiniMax-M3:
          cost:
            input: 0.6
            output: 2.4
            cache_read: 0.12
# `MiniMax-M3` is absent from AWF's built-in pricing catalog. AWF v0.27.44
# supports `models.default-ai-credits-pricing`, but gh-aw v0.83.4 does not
# propagate that field from imported shared workflows into the AWF config.
# Keep the budget disabled here rather than duplicate pricing in every importer.
max-ai-credits: -1
engine:
  id: copilot
  # Free Copilot tier (free_limited_copilot) has 0 premium interactions.
  # The alias keys 'auto'/'mini'/'gpt-5-mini' all resolve to premium
  # models (claude-sonnet-5, claude-haiku-4.5, gpt-5.4-mini) that 400
  # with 'The requested model is not supported'. 'copilot/auto' is NOT an
  # alias key, so the resolver passes it through and normalizes it to
  # 'auto' for the Copilot CLI, which then uses its native Auto mode and
  # dynamically selects a model available to the subscription (verified
  # working on the free tier).
  model: copilot/auto
safe-outputs:
  threat-detection:
    # Detection has an independent budget and the same unknown-model constraint.
    max-ai-credits: -1
    # Detection uses the stateful Claude CLI, so it retains normal recovery.
    engine:
      id: copilot
---
