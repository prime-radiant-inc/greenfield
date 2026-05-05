# StockShots Test Vectors

Status: implementation brief

## Candle Partition Read

Given:

- `features/NFO/ABC/5s/2026-05-05.parquet`
- `features/NFO/ABC/5s/2026-05-06.parquet`

When:

- a client requests candles from `2026-05-05T09:30:00` to
  `2026-05-06T09:35:00`

Then:

- the API reads both partitions;
- returns one continuous candle array sorted by timestamp;
- includes source and freshness metadata;
- does not expose partition filenames in the response.

## Realtime Plus Persisted Merge

Given:

- persisted 5-second candles exist until `09:20:00`;
- websocket buffer has candles from `09:20:05` to `09:21:00`;

When:

- the client requests the latest 120 5-second candles;

Then:

- the response merges persisted and live rows;
- duplicate timestamps are resolved deterministically;
- the newest websocket row wins when timestamps match;
- freshness shows live if recent rows exist.

## Indicator Combo Sort

Given three symbols:

- symbol A has high momentum, high activity, trend aligned;
- symbol B has high momentum, low activity, trend aligned;
- symbol C has low momentum, high activity, trend not aligned.

When:

- user sorts by `momentum + activity + trend`;

Then:

- symbol A ranks above B and C;
- the response explains which fields matched;
- the UI shows the active combo, not a mysterious score.

## Combo Alert

Given:

- user creates alert for `activity >= 2`, `momentum > 0`, `trend aligned`;
- symbol A first satisfies the combo at `10:15:00`;

When:

- the alert scheduler evaluates the latest rows;

Then:

- one alert is sent;
- repeated identical rows are deduped;
- the alert message includes symbol, timeframe, matched fields, and freshness;
- Telegram may include a PNG preview but links to the native chart.

## Customer/Admin Boundary

Given:

- a non-admin customer account;

When:

- the user opens the mobile app or web app;

Then:

- the user can see markets, charts, alerts, and earnings;
- the user cannot see broker funds, P&L, positions, order controls, bot controls,
  or raw websocket diagnostics.

## Provider Precedence

Given:

- yfinance provides a 5-minute candle for a timestamp;
- broker historical data later provides the same timestamp;
- websocket realtime data provides a newer row;

Then:

- broker historical data replaces yfinance for the overlapping timestamp;
- websocket data wins for realtime overlap;
- source metadata reflects the winning provider.

## Binance Futures Mark Price

Given:

- a Binance USD-M futures contract has trade ticks and mark-price updates;

When:

- the latest futures row is returned;

Then:

- last traded price and mark price are separate fields;
- funding context is included when available;
- missing open-interest context is null or degraded;
- customer UI does not expose leverage or account controls.

## Binance Options Missing Greeks

Given:

- a Binance options contract has price data but missing Greeks;

When:

- the option appears in a screen or chart;

Then:

- missing Greeks are null;
- the row remains visible only if liquidity/freshness rules pass;
- options momentum fields may be computed;
- non-options markets still return null for options-only fields.

## Kite Option Resolution

Given:

- a user wants the current at-the-money NIFTY call and put;

When:

- the symbol registry is refreshed;

Then:

- expiry, strike, option side, lot size, tick size, and instrument token come
  from the provider instrument master;
- no expiry date is hardcoded;
- websocket subscriptions use instrument tokens, not guessed symbols.
