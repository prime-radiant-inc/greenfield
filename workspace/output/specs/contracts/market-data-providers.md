# Market Data Provider Contracts

Status: implementation brief

## Provider Rule

Providers are adapters. The rest of the product consumes normalized
`CandleFeature` rows and should not depend on provider-specific response shapes.

Every provider row must preserve:

- provider name;
- provider symbol;
- normalized exchange;
- normalized symbol;
- timestamp;
- freshness;
- entitlement or degraded reason when applicable.

## Kite / Zerodha

Use Kite for Indian realtime markets and admin-only broker operations.

Use Kite for:

- NSE/NFO/MCX realtime ticks;
- option prices;
- instrument master lookup;
- exact tradingsymbol, expiry, strike, lot size, tick size, and exchange token;
- live market-hours checks;
- admin-only positions, holdings, orders, margins, funds, and fills.

Do not use Kite for:

- public/customer authentication;
- customer chart rendering directly;
- historical research when parquet history exists;
- live order placement from the customer app.

Required behavior:

1. Refresh instruments before expiry-sensitive option work.
2. Resolve options from the instrument master; never hardcode expiry dates.
3. Subscribe to tokens for the active symbol registry.
4. Convert ticks to normalized tick events.
5. Aggregate ticks into 1-second or 5-second base candles.
6. Publish last tick time, subscribed count, reconnect count, and dropped-message
   count.
7. Expose broker state only to admin surfaces.

## yfinance

Use yfinance as a slow historical/fallback provider.

Use yfinance for:

- Indian equity candle backfill when broker history is unavailable;
- US equity candle history;
- broad universe context;
- company and earnings metadata where acceptable;
- rebuilding historical parquet for non-realtime views.

Do not use yfinance for:

- 1-second or 5-second realtime charts;
- option ticks;
- broker positions;
- order state;
- latency-sensitive alerts.

Required behavior:

1. Run through scheduled jobs, not user-request realtime paths.
2. Cache aggressively and respect rate limits.
3. Mark rows with provider and freshness.
4. Never overwrite fresher broker/exchange rows with stale yfinance rows.
5. Record coverage per symbol/timeframe/date.

## Binance

Use Binance for crypto realtime and crypto historical data.

Use Binance for:

- spot websocket ticks for BTC, ETH, SOL, and configurable altcoins;
- crypto 1-second or 5-second candles;
- historical klines for backfill;
- exchange symbol metadata;
- liquidity filters based on quote volume and range;
- optional admin-only account/wallet state if keys are configured.

Do not use Binance for:

- Indian market symbols;
- customer-facing wallet/funds screens;
- customer app trading.

Required behavior:

1. Maintain a configurable Binance symbol registry.
2. Add, remove, and pause symbols without redeploying.
3. Track websocket lag, reconnect count, dropped messages, and last trade time.
4. Use BTCUSDT as the default crypto benchmark.
5. Apply liquidity filters before expensive feature computation.
6. Mark low-liquidity or stale symbols as degraded instead of hiding failures.

## Provider Precedence

When multiple providers can produce the same candle:

1. Direct websocket/exchange rows win for realtime.
2. Provider historical rows fill gaps.
3. yfinance fills slow historical gaps.
4. Manual/imported rows are allowed only with explicit source labels.

Customer APIs return one best row per symbol/timeframe/timestamp plus
source/freshness metadata. Internal audit views may show all provider variants.
