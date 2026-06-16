## Summary
CryptoWatchBot is a personal Telegram bot that lets each user privately maintain a watchlist of crypto tickers and receive two kinds of alerts: absolute price thresholds and percentage moves over a 1-hour window. Users add/remove coins with inline buttons or by typing a ticker, query prices on demand, schedule an optional morning summary in their IANA timezone, and set quiet hours to suppress alerts. The bot throttles repeat alerts with a cooldown and retries price-feed failures quietly. The owner (bot operator) can view usage and top-firing alerts.

## Audience
- Primary: individual Telegram users who want lightweight, private crypto price alerts and simple on-demand checks.
- Admin: the bot owner who needs operational metrics (user count, most-fired alerts).

## Core entities
- User (telegram_user_id, display_name, timezone IANA, quiet_hours_start/end, morning_summary_time, cooldown_minutes, created_at)
- WatchlistItem (id, user_id, symbol, normalized_coin_id, added_via, active_flag)
- Alert (id, user_id, watchlist_item_id, type {threshold|percent}, threshold_value, direction {above|below}/percent_direction {up|down|any}, percent_window_minutes, last_triggered_at, cooldown_minutes, enabled)
- PriceSnapshot (coin_id, symbol, price_usd, timestamp, source)
- Notification (id, user_id, alert_id, coin_id, price_old, price_new, percent_change, triggered_at)
- OwnerMetrics (daily_user_count, alerts_fired_by_alert_id, alerts_fired_by_symbol)

## Integrations & notification targets
- Price data provider: CoinGecko public API (primary). A secondary fallback provider: Binance public API (symbol-level) for tickers supported by Binance.
  - Rationale: wide coin coverage (CoinGecko) and simple no-auth access for MVP.
- Messaging: Telegram Bot API (messages, inline keyboards, callbacks).
- Storage: PostgreSQL for persistent data (users, watchlists, alerts, snapshots, metrics). Redis (optional) for short-term caches, rate-limiting, and distributed locks for scheduled checks.
- Background job runner / scheduler: cron-like worker (e.g., celery/beat, BullMQ, or a managed scheduler) to poll prices and run per-user scheduled tasks (morning summary).
- Monitoring/logging: Sentry or similar for runtime errors.

## Interaction flows
- Onboarding (/start)
  - Bot asks user to set IANA timezone (required). Stores timezone before enabling scheduled features.
  - Offers quick-start inline buttons: Add BTC, Add ETH, Add TON, Add custom ticker.
- Adding/removing coins
  - Inline buttons add the normalized symbol to the user's watchlist.
  - "Add custom ticker" opens a prompt to type a ticker or coin id; bot attempts to resolve via CoinGecko search and confirms matches; if ambiguous, offers top matches inline.
  - Remove uses a Manage Watchlist screen with inline toggles per coin.
- Creating alerts
  - UI options per watchlist item: "Set threshold alert" -> choose direction (above/below) and enter USD value; "Set percent alert" -> enter percent and choose direction or "both"; percent window defaults to 60 minutes.
  - Commands supported for power users: /alert add BTC threshold below 60000, /alert add BTC percent 5 down
- On-demand price checks (/price)
  - /price BTC — returns current USD price and 24h change
  - /price — returns a table of current prices for the user's watchlist and which alerts (if any) are currently tripped (without triggering notifications)
- Morning summary (optional)
  - /summary set HH:MM — user picks time in their IANA timezone; bot sends a single message summarizing current prices and notable moves since previous summary.
  - /summary off to disable.
- Quiet hours
  - /quiet set START END in local timezone (e.g., 23:00 07:00). Bot suppresses alerts inside the quiet window; alerts that trigger during quiet hours are queued for delivery at quiet end (optionally aggregated into a single message).
- Alert evaluation & delivery
  - Price poll frequency: every minute for tracked symbols (batching requests across users). For percent alerts, use rolling-window computation comparing now vs price 60 minutes ago.
  - When an alert condition is met, send exactly one alert message describing: coin symbol, old price (timestamped), new price, absolute USD change, percent change, alert type that fired.
  - After firing, set last_triggered_at and suppress further alerts for that same alert for cooldown_minutes (default 240 minutes). If quiet hours are active at trigger time, queue notification to be sent at quiet end; do not consider queued delivery as a retrigger.
  - To avoid false alerts from transient feed issues: require that the price feed returned a valid numeric price for both now and the historical point; if a feed error occurs, retry up to 3 times with short backoff; if still failing, do not trigger and log an incident.
- Unknown ticker / typo handling
  - If user-provided ticker cannot be resolved, reply with: "I couldn't find 'FOOBAR'; did you mean: ..." and present top matches from CoinGecko with inline buttons to select.

## Persistence
- PostgreSQL schema containing tables for users, watchlist_items, alerts, price_snapshots (time-series, optional TTL for older rows), notifications, owner_metrics.
- Redis used for ephemeral locks, cooldowns, and scheduled job coordination.

## Retry & failure behavior
- Price fetch: on transient HTTP/network errors, retry up to 3 times with exponential backoff; if a coin's feed fails for a minute, skip firing alerts for that coin until healthy.
- Scheduled worker idempotency: use per-alert locks to ensure single evaluation across workers.

## Owner/admin features
- /admin register <secret> to designate an owner account (first-time setup) or during deployment the owner Telegram ID can be set in environment.
- /admin stats — shows: total users, active users (used in last 30 days), total alerts configured, top 10 most-fired alerts and top 10 coins by alert count (time-window selectable: day/week/month).
- Access control: /admin commands only respond to designated owner ID(s).

## Payments
- None in MVP. No paid tiers or in-app purchases; telemetry for owner metrics only (no user-identifying public leaderboard).

## Non-goals
- No automatic trading/exchange order placement.
- No group chat watchlists — per-user private only for MVP.
- No complex portfolio valuation or multi-currency pricing (USD only initially).

## Assumptions & defaults
- Price provider: CoinGecko primary, Binance as fallback — chosen for broad coverage and no-auth access for MVP.
- Percent-window: 60 minutes — the user requested "in an hour" so alerts compare now vs 60 minutes prior.
- Cooldown after alert: 240 minutes (4 hours) — prevents spam from wobbling without asking the owner; configurable per-user/alert.
- Poll frequency: 1 minute for batch price polling — balances timeliness with API rate limits; worker batches multiple symbols into single CoinGecko requests when possible.
- Quiet-hours behavior: suppress and queue alerts to deliver once quiet hours end (as a single aggregated message) — avoids surprising the user while ensuring important triggers are not lost.
- Owner registration: owner grants admin by running /admin register <secret> (or provisioning owner_id via env) — secure admin access without a separate dashboard.
- Unknown ticker handling: use CoinGecko search and present top matches inline; require explicit user confirmation before adding to watchlist — prevents silent mis-subscribes.
- Data retention: keep price snapshots for 30 days at full resolution, older data archived or downsampled — supports percent-window computations and limits storage growth.

If you want any of the defaults changed (price provider, cooldown, percent window, cooldown behavior, or whether queued alerts are aggregated vs. sent individually), tell me which one to adjust and I will update the brief.