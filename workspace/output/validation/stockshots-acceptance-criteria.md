# StockShots Acceptance Criteria

Status: implementation brief

## Data

- A Binance symbol streams realtime ticks.
- An Indian option symbol streams realtime ticks through Kite.
- 5-second candles are stored with source and freshness metadata.
- 1-minute and 5-minute candles are resampled from the same base data.
- Indicator rows are stored per candle.
- APIs can return a continuous candle range across multiple date partitions.

## Screening

- Users can sort by one indicator.
- Users can sort by a combination of indicators.
- Users can filter and sort by a combination.
- The UI displays the active combination in plain language.
- Missing unsupported values are null or degraded, not fake zero.

## Charts

- Web renders native candle charts from JSON.
- iOS renders native candle charts from JSON.
- macOS renders native multi-chart views from JSON.
- PNG charts are not used inside app/web chart surfaces.
- Telegram can use PNG preview images.

## Alerts

- Users can create a combo alert.
- Alert evaluation uses the same screening logic as the app.
- Alerts dedupe repeated matches.
- Push and Telegram delivery both include matched fields and freshness.

## Customer Product

- Customer app shows markets, symbols, charts, alerts, and earnings.
- Customer app does not show broker funds, P&L, positions, order controls, or
  raw infrastructure diagnostics.

## Admin Product

- Admin app shows websocket health, last tick time, lag, dropped messages,
  worker heartbeats, bot state, P&L, positions, and symbol controls.
- Admin-only endpoints reject non-admin users.

## Provider Behavior

- Kite provides Indian realtime ticks and admin-only broker state.
- yfinance provides historical/fallback candles and never replaces fresher
  broker rows.
- Binance provides crypto realtime and historical data.
- Provider failures produce degraded states visible in health responses.

## Production Health

- API health reports data freshness.
- Worker health reports lag and failed jobs.
- Websocket health reports reconnects and last tick time.
- App surfaces show delayed/offline states instead of silent stale data.
