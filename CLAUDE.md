# Operating playbook for Claude Code in this repo

This repository can place **real equity orders** on a user's Robinhood
Agentic Trading account via a `robinhood-trading` MCP server that the user
registers locally (see README → "Live trading via Robinhood Agentic
Trading"). This document is the *only* sanctioned path for doing so.
Follow it exactly. If the `robinhood-trading` MCP is not registered in the
current Claude Code session, refuse and tell the user to run
`claude mcp add robinhood-trading --transport http <robinhood-mcp-url>`.

## Safety preamble — non-negotiable

- Never call any `robinhood-trading` MCP tool unless **all** of the following are true in the current turn:
  1. The user explicitly asked you to execute / submit / place / send trades.
  2. The hedge fund was run with `--live` (you will see `"execution_mode": "live"` in its output JSON).
  3. The environment variable `HEDGE_FUND_LIVE_TRADING_ALLOWED=true` is set.
  4. You have shown the decision summary table (see below) and the user has answered **y** to a confirmation prompt **in this turn**.
- If any of (1)–(4) fails, do not call the MCP. Print why and stop.
- Approval from a previous turn does not carry forward. Re-confirm on every batch.
- Never add a broker MCP server to the user's Claude config without explicit user instruction in the same turn. (This repo does not ship an `.mcp.json` for the broker — the user registers it locally.)
- Never invent or guess order parameters the user did not provide (e.g., limit prices, time-in-force). The hedge fund emits market-equivalent decisions only; pass action + ticker + quantity unchanged.
- On any MCP error mid-batch, stop the batch and report. Do not retry blindly. Do not skip ahead to the next ticker without the user's say-so.

## Standard workflow

### 1. Run the hedge fund

```bash
poetry run python src/main.py --tickers <TICKERS> --output-file decisions.json [--live] [--confirm-per-trade] [--show-reasoning]
```

- Default is dry-run. Add `--live` only when the user asked to actually trade.
- If the user wants approval per ticker (not per batch), add `--confirm-per-trade`.

### 2. Read and summarize decisions

Read `decisions.json`. It contains:

```
{
  "decisions": { "<TICKER>": { "action", "quantity", "confidence", "reasoning" }, ... },
  "analyst_signals": { ... "risk_management_agent": { "<TICKER>": { "current_price", ... } } ... },
  "execution_mode": "dry_run" | "live",
  "confirm_per_trade": true | false
}
```

Print a table to the user:

| Ticker | Action | Qty | Est. Notional | Confidence | Reasoning (1 line) |

`Est. Notional = quantity × analyst_signals.risk_management_agent.<ticker>.current_price`. Truncate `reasoning` to ~80 chars.

### 3. Gate

- If `execution_mode != "live"`: print `Dry run — re-run with --live to submit orders.` and stop. Do not call the MCP.
- If `execution_mode == "live"` and `HEDGE_FUND_LIVE_TRADING_ALLOWED` is unset or not `true`: refuse. Tell the user to set the env var.
- Otherwise: ask `Submit N orders to Robinhood? (y/N)` (or per-ticker if `confirm_per_trade` is true). Treat anything other than a clean `y` / `yes` as no.

### 4. Submit

For each decision where `action != "hold"`, call the matching Robinhood MCP tool with `symbol = <ticker>` and `quantity = <quantity>`:

- `buy` — open or add to a long
- `sell` — close or reduce a long
- `short` — open or add to a short
- `cover` — close or reduce a short
- `hold` — skip; do not call the MCP

Use the MCP tool's own parameter names as discovered at call time; do not hard-code field names here. Equities only — Robinhood Agentic Trading is equities-only in beta. If a user asks to trade options/crypto/futures via this MCP, refuse and explain.

### 5. Record

Append one JSON line per submission to `trades.log` in the repo root:

```
{"ts": "<iso8601>", "ticker": "...", "action": "...", "quantity": N, "request": {...}, "response": {...}, "order_id": "...", "status": "..."}
```

Include the MCP tool name in `request.tool`. On error, write the error in `response` and stop the batch.

### 6. Report back

Summarize: N submitted, M skipped (holds), K failed (with first error). Print order IDs.

## Common mistakes to avoid

- Calling an MCP tool to "check" or "test" — there is no safe test call against a live brokerage; any submission is real. Use dry-run for testing.
- Treating a `hold` as a position close. `hold` means no action — skip it.
- Re-submitting a failed order without the user's say-so.
- Editing `.env` to set `HEDGE_FUND_LIVE_TRADING_ALLOWED=true` yourself. The user sets this in their shell; you do not.

## When the user asks something off-script

If the user asks you to do something not covered above — e.g., cancel an open order, check positions, fetch balances — those are likely supported by the Robinhood MCP but outside this playbook's automated path. Treat each as a single, explicit, user-confirmed MCP call. Do not chain them into the decision-execution flow.
