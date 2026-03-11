# GRVT Market-Making Bot

A two-sided market-making bot for the [GRVT](https://grvt.io) perpetual futures exchange. It places limit orders on both sides of the book, captures the spread when either side fills, and manages positions back to flat with configurable risk limits.

---

## Table of Contents

- [Strategy Overview](#strategy-overview)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Running the Bot](#running-the-bot)
- [Code Structure](#code-structure)
- [Logs](#logs)
- [Risk & Safety](#risk--safety)
- [Supported Instruments](#supported-instruments)

---

## Strategy Overview

1. **Two-sided quoting** — Place a limit BUY near best bid and a limit SELL near best ask at the same time.
2. **Spread capture** — When either side fills, you earn the spread (or part of it).
3. **Position management** — After a fill, the bot places a reduce-only order on the opposite side to return to flat.
4. **Quote refresh** — If the market moves and your orders are too far from the book (beyond `stale_bps`), they are cancelled and replaced at the new best bid/ask.
5. **Emergency close** — If unrealized loss exceeds `hard_loss_pct` of equity, the bot cancels all orders and closes the position with a market order.

Order signing follows EIP-712 (typed data) and matches the GRVT SDK behaviour.

---

## Requirements

- **Python 3.8+**
- **Dependencies** (install with `pip install -r requirements.txt` if you have one, or):
  - `requests`
  - `eth-account`

No command-line arguments are required: **all settings are read from `grvt_config.json`**.

---

## Configuration

Configuration is loaded from **`grvt_config.json`** in either:

- The directory containing `bot_obf.py`, or  
- The current working directory.

You can override the config file path with:

```bash
python bot_obf.py --config /path/to/grvt_config.json
```

Optional environment variable overrides for secrets (if you prefer not to put them in the file):

- `GRVT_API_KEY` — overrides `api_key` when set
- `GRVT_SIGNER_KEY` — overrides `signer_private_key` when set  
- `GRVT_SUB_ID` — overrides `sub_account_id` when set

### Config file: `grvt_config.json`

All keys below can be set in `grvt_config.json`. Unknown keys are ignored (with a warning); some legacy keys are explicitly deprecated and not used.

#### Credentials & environment

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `api_key` | string | `""` | GRVT API key (from exchange/dashboard). |
| `signer_private_key` | string | `""` | Ethereum private key used to sign orders (no `0x` prefix is fine). |
| `sub_account_id` | string | `""` | GRVT sub-account ID to trade on. |
| `env` | string | `"testnet"` | `"testnet"` or `"prod"`. |
| `instrument` | string | `"ETH_USDT_Perp"` | Perpetual symbol, e.g. `BTC_USDT_Perp`, `SOL_USDT_Perp`. |

#### Order sizing & placement

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `order_size` | float | `0.002` | Fallback order size (in base asset) when equity is not yet available. |
| `tick_size` | float | `0.5` | Price tick size for the instrument (used for rounding). |
| `order_expiry_h` | float | `1.0` | Order time-to-live in hours (nanoseconds used for expiry in signing). |
| `trade_size_pct` | float | `0.10` | Fraction of equity per order (each side), e.g. `0.10` = 10%. |
| `max_exposure_pct` | float | `0.50` | Stop opening new quotes when position notional ≥ this fraction of equity (e.g. `0.50` = 50%). |
| `quote_offset_ticks` | int | `0` | Number of ticks away from best bid/ask; `0` = join the best. |
| `stale_bps` | float | `5.0` | Cancel and replace a quote when its price has moved more than this many basis points from the current best. |
| `dry_run` | bool | `false` | If `true`, no real orders are sent; only logs what would be done. |

#### Risk limits

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `hard_loss_pct` | float | `0.005` | Emergency market close if unrealized loss > this fraction of equity (e.g. `0.005` = 0.5%). |
| `max_position_notional` | float | `500.0` | Maximum position size in USD when equity is not used (e.g. at startup). |

#### Timing (main loop & behaviour)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cycle_sleep_sec` | float | `2.0` | Seconds to sleep between main loop cycles. |
| `session_check_sec` | float | `300` | Re-check session (and re-login if needed) every this many seconds. |
| `cancel_retry_count` | int | `3` | Number of times to retry checking that an order was cancelled. |
| `cancel_retry_delay_sec` | float | `0.25` | Delay between cancel-confirm retries (seconds). |
| `emergency_close_delay_sec` | float | `0.5` | Delay before sending the market close order in emergency close. |
| `emergency_verify_sleep_sec` | float | `1.0` | Sleep after emergency close before re-syncing position. |
| `startup_cancel_sleep_sec` | float | `0.5` | Sleep after initial cancel-all on startup. |
| `no_price_wait_sec` | float | `3` | Sleep when no price is available from the ticker. |
| `after_emergency_sleep_sec` | float | `2` | Sleep after triggering emergency close before continuing the loop. |

### Example `grvt_config.json`

```json
{
  "api_key": "your_api_key",
  "signer_private_key": "your_signer_private_key",
  "sub_account_id": "your_sub_account_id",
  "instrument": "ETH_USDT_Perp",
  "env": "testnet",

  "order_size": 0.002,
  "tick_size": 0.5,
  "order_expiry_h": 1.0,
  "dry_run": false,

  "trade_size_pct": 0.1,
  "max_exposure_pct": 0.5,
  "quote_offset_ticks": 0,
  "stale_bps": 5.0,

  "hard_loss_pct": 0.005,
  "max_position_notional": 500.0,

  "cycle_sleep_sec": 2.0,
  "session_check_sec": 300,
  "cancel_retry_count": 3,
  "cancel_retry_delay_sec": 0.25,
  "emergency_close_delay_sec": 0.5,
  "emergency_verify_sleep_sec": 1.0,
  "startup_cancel_sleep_sec": 0.5,
  "no_price_wait_sec": 3,
  "after_emergency_sleep_sec": 2
}
```

---

## Running the Bot

1. Copy or create `grvt_config.json` in the script directory or current working directory.
2. Fill in `api_key`, `signer_private_key`, and `sub_account_id` (or set `GRVT_API_KEY`, `GRVT_SIGNER_KEY`, `GRVT_SUB_ID`).
3. Set `instrument`, `env` (`testnet` or `prod`), and other parameters as needed.
4. Optionally set `dry_run: true` to test without sending real orders.
5. Run:

```bash
python bot_obf.py
```

Optional: specify a config file path:

```bash
python bot_obf.py --config /path/to/grvt_config.json
```

On **SIGINT** (Ctrl+C) or **SIGTERM**, the bot cancels all open orders for the configured instrument and sub-account, then exits.

---

## Code Structure

### Main components (`grvt_update_2.5_multiaccount.py`)

- **Constants** — `PRICE_MULTIPLIER`, `CHAIN_IDS`, `MIN_ORDER_SIZE`, `MIN_NOTIONAL`, EIP-712 order type and time-in-force mappings.
- **InstrumentCache** — Loads perpetual instruments from GRVT market-data API; provides `min_size` and metadata for signing.
- **sign_order()** — Builds EIP-712 typed data, rounds size to lot size, signs with `eth_account`, returns signature and `signed_size` for the create-order payload.
- **Config** — Dataclass-like config object; all fields above are attributes. Loaded from JSON and optionally overridden by env vars for secrets.
- **GRVTClient** — REST client: login (API key → cookie), account summary, positions, open orders, create/cancel order, cancel-all. Uses `edge`, `trades`, and `market-data` base URLs from `env`.
- **Position** — Holds `size` and `avg_entry`; provides `pnl(mark)`, `is_flat`, `is_long`, `is_short`.
- **MO (Managed Order)** — Tracks a live order: `client_oid`, `is_buy`, `price`, `size`.
- **MarketMaker** — Core loop:
  - **Size helpers** — `_min_size()`, `_min_notional()`, `_valid_size()`, `_trade_size()` (equity × `trade_size_pct` or `order_size` fallback).
  - **Submit** — Validates size/notional, builds EIP-712 order, signs, calls `create_order` (or dry-run).
  - **Cancel** — Cancels by `client_oid`; optional confirm with retries.
  - **Balance/position** — `_get_balances()` (equity, internal sub-account id), `_sync_pos()` from API.
  - **Quoting** — `_quote_prices()` (best bid/ask ± offset), `_is_stale()`, `_refresh_quotes()` (maintain buy + sell quotes; skip side if at exposure limit).
  - **Close order** — `_refresh_close()`: when position is not flat, place or refresh a reduce-only close order; only reprice when the new price is strictly better.
  - **Emergency close** — Cancel all, then market reduce-only to flat; used when unrealized loss > `hard_loss_pct` of equity.
  - **run()** — Signal handlers, startup cancel-all, then loop: session check, balances, position, ticker, open orders; risk check; refresh quotes; refresh close; cancel same-side quote when position exists; sleep `cycle_sleep_sec`.

### Config loading

- **load_config(args)** — Builds `Config()`, finds `grvt_config.json` (script dir or cwd, or `args.config` if provided). Loads JSON and sets any key that exists on `Config`; ignores deprecated keys; warns on unknown keys. Then fills in `api_key`, `signer_private_key`, `sub_account_id` from env if not set in JSON.
- **validate(cfg)** — Ensures `api_key`, `signer_private_key`, `sub_account_id` are non-empty and `env` is in `CHAIN_IDS`; exits with error otherwise.

---

## Logs

- **grvt_bot.log** — Main log: config load, cycles, quotes, submits, cancels, errors, emergency close.
- **grvt_trades.log** — Trade-level log (if used by the script).

Both use UTF-8. Console output is also logged to the main handler (with a safe stream handler for non-UTF terminals).

---

## Risk & Safety

- **Min size / min notional** — The bot uses exchange min size and min notional (from API or built-in fallbacks) to avoid rejections (e.g. 2062, 2066). Order size is derived from equity and `trade_size_pct`, then validated.
- **Exposure cap** — No new quotes are placed when position notional ≥ `max_exposure_pct` of equity (or `max_position_notional` when equity is not used).
- **Hard loss** — If unrealized loss exceeds `hard_loss_pct` of equity, the bot runs **emergency close**: cancel all orders, then one market reduce-only order to flatten, then re-sync position.
- **Close order rule** — The reduce-only close order is only re-priced when the new price is strictly better (never chase the market).

Use `dry_run: true` to verify behaviour without sending orders. Start with `testnet` and small size.

---

## Supported Instruments

The code includes fallback min order sizes and min notionals for:

- `BTC_USDT_Perp`
- `ETH_USDT_Perp`
- `SOL_USDT_Perp`
- `ARB_USDT_Perp`
- `BNB_USDT_Perp`
- `DOGE_USDT_Perp`

When the market-data API provides `min_size` (or equivalent) for an instrument, that is used; otherwise the hardcoded fallbacks apply. Other perpetuals may work if they follow the same API shape; ensure `instrument` in config matches the exchange symbol exactly (e.g. `ETH_USDT_Perp`).

---

## Summary

- **Config only** — All behaviour is driven by `grvt_config.json` (and optional env for secrets).
- **Optional CLI** — Only `--config` is supported to override the config file path.
- **Two-sided market making** — Bid and ask quotes, position management via reduce-only close, quote refresh on staleness, emergency market close on hard loss.

For issues, check `grvt_bot.log` and ensure `api_key`, `signer_private_key`, and `sub_account_id` are correct for the chosen `env` (testnet vs prod).
