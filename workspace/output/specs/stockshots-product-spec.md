# StockShots Product Specification

Status: implementation brief
Audience: new implementation team
Scope: backend, cloud workers, web, iOS, macOS, widget, Telegram, admin tools

## Product Definition

StockShots is a cloud-computed market signal product. It ingests market data,
turns ticks and historical candles into a canonical candle-plus-indicator
dataset, lets users sort and filter symbols by indicator combinations, renders
native charts in every app, and sends alerts when selected market conditions
appear.

The product is not a broker terminal for customers. Customer surfaces explain
market movement and alert users. Admin surfaces may expose broker state,
operational health, bot state, paper/live decision support, and P&L.

## User-Facing Product Flow

1. A user opens StockShots and sees a market tape plus a ranked symbol list.
2. The user chooses a market and timeframe.
3. The user sorts or filters symbols by a visible indicator combination:
   momentum, acceleration, activity, range, trend, band position, candle trend,
   previous high/low context, and freshness.
4. The user opens a symbol and sees a native line/candlestick chart using the
   same candle rows computed in the cloud.
5. The user saves an alert rule from the same visible indicator combination.
6. The alert system sends push and Telegram notifications when the rule appears.
7. Admin users open a separate admin surface for websocket health, bots, worker
   lag, broker positions, funds, P&L, and controls.

## Non-Negotiable Product Rules

- One canonical candle-feature row shape feeds every consumer.
- Apps and web render native charts from JSON candle data.
- PNG chart images are allowed only for Telegram/share previews.
- Do not expose an unexplained "score" as the main product concept.
- Sorting and filtering must show the indicator combination being used.
- Customer apps must not expose funds, broker positions, P&L, order placement,
  live trading controls, or raw infrastructure diagnostics.
- Admin apps may expose operational and broker details.
- Every row must carry source and freshness metadata.
- Unsupported or unavailable values are null or explicitly degraded, never fake
  zero.
- Realtime rows and historical rows use the same API shape.

## Markets

The product must support these markets, with graceful degradation when data is
unavailable:

- Indian equities.
- Indian index and equity options.
- Indian commodities if provider entitlement exists.
- Binance spot crypto.
- Binance USD-M futures where enabled.
- Binance COIN-M futures where enabled.
- Binance options where enabled and sufficiently liquid.
- US equities through historical providers.
- Earnings/company-event context where available.

Options-specific behavior:

- Option symbols must be resolved from provider instruments, not by hardcoding
  expiry dates.
- The default NFO option universe should be one at-the-money call and one
  at-the-money put per enabled underlying, plus an explicitly configured index
  option pack.
- Options momentum fields apply only to options markets. Non-options markets
  should return null for options-only values.

Crypto-specific behavior:

- BTCUSDT is the default crypto benchmark.
- Users/admins can add, remove, or pause Binance spot, futures, and options
  symbols independently.
- Expensive realtime computation should prioritize liquid symbols.
- Futures rows must preserve contract type, funding context, mark price, and
  open-interest context when available.
- Binance options rows must preserve expiry, strike, option side, implied
  volatility context, and underlying price context when available.

## Canonical Data Model

### CandleFeature

Each row represents one symbol, one market, one timeframe, and one candle close
timestamp.

Identity fields:

- `exchange`
- `symbol`
- `timeframe`
- `timestamp`
- `source`
- `freshness_state`
- `computed_at`

OHLCV fields:

- `open`
- `high`
- `low`
- `close`
- `volume`
- `vwap`
- `ltp`
- `daily_change_pct`

Indicator fields:

- `rvol`
- `ratr`
- `atr`
- `atr_pct`
- `rrs`
- `rrs_rising`
- `rrs2`
- `rrs2_rising`
- `ha_state`
- `ema_3`
- `ema_8`
- `ema_20`
- `ema_50`
- `ema_200`
- `ema200_pct`
- `bb_pct`
- `prev_high`
- `prev_low`
- `above_prev_high`
- `below_prev_low`
- `hh`
- `ll`

Optional tick-level fields:

- `trade_count`
- `buy_volume`
- `sell_volume`
- `volume_delta`
- `spread`
- `imbalance`

Plain-language labels:

- `rrs` means Momentum.
- `rrs_rising` means Acceleration.
- `rrs2` means Options momentum.
- `rrs2_rising` means Options acceleration.
- `rvol` means Activity.
- `atr` and `ratr` mean Range.
- `bb_pct` means Band position.
- `ha_state` means Candle trend.
- `ema_*` means Trend line.

## Storage Model

Store candle-feature history as parquet files in object storage. Store users,
alert rules, auth, symbol registry, worker state, and audit events in a
transactional database.

Recommended parquet partition:

```text
features/{exchange}/{symbol}/{timeframe}/{yyyy-mm-dd}.parquet
```

Example:

```text
features/NFO/NIFTY_CALL_OPTION/5s/2026-05-05.parquet
```

The date in the filename is a physical storage partition, not a user-facing
product concept. APIs should return continuous candle ranges by reading all
needed partitions.

Why date partitions:

- today's chart can read today's file instead of years of 5-second candles;
- one bad day can be recomputed without rewriting full history;
- cloud object storage works better with immutable daily chunks;
- workers can process dates independently;
- old history remains stable while live rows keep updating.

## Backend Services

Build five backend domains. These can be deployed as separate services or as
modules in one service, but their responsibilities must remain separate.

### Ingestion

Responsibilities:

- subscribe to realtime provider streams for broker ticks, exchange trades,
  klines, mark prices, funding context, and account/admin streams where enabled;
- fetch historical candles;
- normalize provider events into internal tick/candle events;
- maintain symbol registry;
- track reconnects, last tick time, lag, and dropped messages.

Outputs:

- normalized ticks;
- 1-second or 5-second base candles;
- provider health snapshots.

### Feature Computation

Responsibilities:

- aggregate ticks into base candles;
- resample base candles to higher timeframes;
- compute indicator fields;
- write candle-feature rows to parquet;
- publish latest rows to a low-latency cache;
- mark stale and degraded rows.

### Screening

Responsibilities:

- read latest candle-feature rows;
- apply a user-selected indicator combination;
- sort/filter symbols;
- return explainable rows with the matched fields.

Screening modes:

- single-field sort;
- multi-field sort;
- combo filter plus sort;
- saved preset;
- alert dry run.

### Alerts

Responsibilities:

- evaluate saved indicator combinations;
- dedupe repeated alerts;
- send mobile push;
- send Telegram text;
- optionally generate a PNG preview for Telegram;
- store alert history.

### Admin Operations

Responsibilities:

- show ingestion fleet health;
- show websocket lag and dropped messages;
- show worker heartbeats;
- show bot state;
- show broker positions, funds, and P&L;
- allow symbol add/remove/pause;
- allow admin-only paper/live decision support.

## Consumer Surfaces

### Web

The web product must be authenticated. Anonymous access should not expose live
market data.

Screens:

- Markets: sortable/filterable symbol list.
- Scanner: indicator-combo presets and results.
- Symbol detail: native chart, indicator tape, multi-timeframe context.
- Compare: multiple symbols/charts.
- Alerts: create and manage alert combinations.
- Earnings: earnings/calendar context.
- Admin: websocket and bot health for admins only.

### iOS Customer App

Tabs:

- Today.
- Discover.
- Scanner.
- Alerts.
- More.

Customer app capabilities:

- market tape;
- ranked symbol cards;
- sortable/filterable symbol list;
- native line and candlestick charts;
- multi-timeframe chart detail;
- indicator tape;
- push alerts;
- earnings context.

### iOS/macOS Admin App

Admin capabilities:

- all customer analytics;
- websocket fleet health;
- bot health;
- symbol registry controls;
- broker positions;
- funds;
- P&L;
- paper/live decision-support controls;
- Telegram controls;
- worker/scheduler health.

### macOS

The macOS app is a dense control-room workspace:

- multi-chart grid;
- scanner table;
- symbol detail inspector;
- alert builder;
- admin health panels.

### Widget

The widget should show lightweight market state:

- top movers;
- active setups;
- market mood;
- alert count;
- freshness state.

### Telegram

Telegram is an alert and digest surface, not the system of record.

Expected commands:

- `/top`
- `/top {exchange} {timeframe}`
- `/combo {fields}`
- `/alert create {exchange} {timeframe} {fields}`
- `/earnings`
- `/health`

Telegram may use PNG chart previews. It must link back to native app/web charts
for full interaction.

## Cloud Architecture

Minimum production cloud:

- API service.
- Realtime ingestion workers.
- Historical backfill workers.
- Feature computation workers.
- Screening/alert scheduler.
- Object storage for parquet.
- Transactional database for metadata.
- Cache for latest rows and dedupe state.
- Queue or stream for tick/candle jobs.
- Observability for lag, freshness, errors, and user-visible outages.

Required health signals:

- API uptime;
- provider connection state;
- last tick time per symbol;
- candle write lag;
- feature compute lag;
- parquet write lag;
- alert delivery lag;
- dropped/rejected message count;
- worker heartbeat;
- stale symbol count.

## Build Order

1. Define the `CandleFeature` contract.
2. Build one provider adapter for Binance realtime.
3. Build one provider adapter for Kite realtime.
4. Store 5-second candles.
5. Compute core indicators for 5-second candles.
6. Resample to 15-second, 1-minute, and 5-minute rows.
7. Build candle and screen APIs.
8. Build web scanner and native chart.
9. Build iOS customer scanner and native chart.
10. Build alert rules and Telegram delivery.
11. Build admin health panels.
12. Build macOS multi-chart control room.
13. Expand symbol coverage.
14. Add earnings/company context.

## Explicit Non-Goals For MVP

- Direct customer trading.
- Game/training surface before the data product is stable.
- Multiple charting engines.
- Multiple ranking systems.
- Multiple backend API versions.
- Separate data stores per consumer.
- Customer-facing broker funds or P&L.
- Unexplained score-first UI.
