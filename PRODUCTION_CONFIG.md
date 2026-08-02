# Production Config Adjustments

Changes from the [NFI default example config](https://github.com/iterativv/NostalgiaForInfinity/blob/main/configs/exampleconfig.json) to `config.production.json`.

## Budget Constraints

| Setting | Default | Production | Reason |
|---|---|---|---|
| `dry_run_wallet` | 10000 | 250 | Running with a $250 starting wallet |
| `max_open_trades` | 6 | 6 | Matches NFI minimum recommendation. At 3x leverage, each trade gets ~$123 position |

## Futures Mode

The default config is written for spot. We're trading Binance USDT-M futures:

| Setting | Production Value |
|---|---|
| `trading_mode` | `futures` |
| `margin_mode` | `isolated` |
| `exchange.name` | `binance` |

The strategy auto-detects futures mode via `trading_mode` and enables shorting with `can_short = True` and `futures_mode_leverage = 3.0`.

## Pair Selection

The default config leaves pairlist and blacklist empty. We use:

- **VolumePairList** with 40 assets, filtered through AgeFilter (60 days), PriceFilter, SpreadFilter, and RangeStabilityFilter
- **Blacklist** covering leveraged tokens (BULL/BEAR/UP/DOWN), fiat pairs, stablecoins, and fan tokens — as recommended by NFI
- **MKR/OMNI** blacklisted due to high per-unit price causing minimum notional issues on small accounts

## Notifications

| Setting | Default | Production |
|---|---|---|
| `telegram.enabled` | not set | `false` |
| `webhook.enabled` | not set | `true` |

Telegram is disabled in favor of a custom webhook notifier at `https://freqtrade-notifier.lamualfa.dev/webhook/freqtrade`. All webhook templates (`webhookentry`, `webhookentryfill`, `webhookentrycancel`, `webhookexit`, `webhookexitfill`, `webhookexitcancel`, `webhookstatus`) are configured with freqtrade RPC placeholders.

## API Server

Enabled for remote access and monitoring:

- `listen_ip_address`: `0.0.0.0`
- `listen_port`: `8080`
- `enable_openapi`: `false`
- Credentials set via environment variables

## Data Format

| Setting | Production Value |
|---|---|
| `dataformat_ohlcv` | `feather` |
| `dataformat_trades` | `feather` |

Faster reads/writes compared to the default `jsongz`.

## Secrets

All sensitive values are empty strings in the config file and injected at runtime via environment variables:

- `exchange.key` / `exchange.secret`
- `api_server.jwt_secret_key`
- `api_server.username` / `api_server.password`
- `telegram.token` / `telegram.chat_id`

## Unchanged from Default

These remain identical to the NFI example config:

- `timeframe`: `5m` (must not be overridden)
- `stake_amount`: `unlimited`
- `tradable_balance_ratio`: `0.99`
- `unfilledtimeout`: entry 3m, exit 2m
- `order_types`: all `limit`, `stoploss_on_exchange: false`
- `entry_pricing` / `exit_pricing`: `use_order_book: true` (NFI default is `false`, but Binance futures requires order book pricing — ticker endpoint is unavailable)
- `rateLimit`: `60`
- `process_throttle_secs`: `5`
- `stoploss_on_exchange_interval`: `60`
- `stoploss_on_exchange_limit_ratio`: `0.99`

## Fork maintenance rules

The `production` branch is kept a linear superset of `iterativv:main` via rebase
(see the nfi-sync job in the parent `freqtrade-notifier` repo). Rebase replays
this branch's commits on top of upstream — so **a commit that edits a file
upstream also owns will conflict**, every time upstream touches that file.

To keep rebases conflict-free, follow these rules for any change on `production`:

1. **Never edit a file that exists in upstream.** All fork customizations live in
   fork-exclusive files (listed below). If you need to diverge from an upstream
   file, copy it to a new fork-exclusive filename and document the divergence in
   `fork-env-example.env` or here.
2. **Fork-exclusive files** (safe to edit — upstream does not have them):
   - `config.production.json` *(the production freqtrade config)*
   - `docker-compose.production.yml`
   - `docker/Dockerfile.production`
   - `docker/entrypoint.production.sh`
   - `PRODUCTION_CONFIG.md` *(this file)*
   - `fork-env-example.env` *(fork-specific env-var docs)*
3. **Shared files** (upstream owns them — do NOT edit):
   - `live-account-example.env` — fork env vars are documented in
     `fork-env-example.env` instead.
   - `configs/exampleconfig_secret.json`
   - all strategy `.py` files, `configs/*.json`, etc.

The nfi-sync job aborts (without pushing) if a rebase conflicts; resolving it
manually once is fine, but the goal is zero conflicts by design.
