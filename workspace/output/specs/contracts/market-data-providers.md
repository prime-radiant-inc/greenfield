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
3. Subscribe to instrument tokens for the active symbol registry.
4. Convert ticks to normalized tick events.
5. Aggregate ticks into 1-second or 5-second base candles.
6. Publish last tick time, subscribed count, reconnect count, and dropped-message
   count.
7. Expose broker state only to admin surfaces.
8. Treat missing entitlement, expired token, and market-closed states as
   explicit degraded states.

Kite realtime contract:

- Input identity is an exchange plus provider tradingsymbol.
- Subscription identity is the provider instrument token.
- Tick rows must include timestamp, last traded price, volume or quantity where
  provided, and provider symbol identity.
- Option rows must preserve underlying, expiry, strike, option side, lot size,
  and tick size from the instrument master.
- Admin broker rows must be separated from customer market-data rows.

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

Use Binance for crypto realtime and crypto historical data across spot,
futures, and options when enabled.

### Binance Spot

Use Binance spot for:

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

Spot required behavior:

1. Maintain a configurable Binance symbol registry.
2. Add, remove, and pause symbols without redeploying.
3. Subscribe to trade, aggregate trade, ticker, or kline streams depending on
   latency and volume needs.
4. Track websocket lag, reconnect count, dropped messages, and last trade time.
5. Use BTCUSDT as the default crypto benchmark.
6. Apply liquidity filters before expensive feature computation.
7. Mark low-liquidity or stale symbols as degraded instead of hiding failures.

### Binance USD-M And COIN-M Futures

Use Binance futures for:

- futures trade streams;
- futures kline streams;
- mark price streams;
- funding-rate context;
- open-interest context;
- basis/premium context where available;
- admin-only account, position, and liquidation-risk context if futures keys are
  configured.

Do not use Binance futures for:

- customer-facing leverage controls;
- customer-facing wallet/funds screens;
- customer app trading.

Futures required behavior:

1. Maintain separate registries for spot, USD-M futures, and COIN-M futures.
2. Preserve contract category in every normalized row.
3. Store mark price separately from last trade price.
4. Store funding-rate and next-funding-time context when available.
5. Store open-interest context when available.
6. Track websocket lag, reconnects, dropped messages, and last event time per
   contract.
7. Mark unavailable futures metadata as null or degraded, never fake zero.
8. Treat account and position streams as admin-only.

### Binance Options

Use Binance options only when the product explicitly enables option coverage and
the contract is liquid enough to be useful.

Use Binance options for:

- option trade or ticker streams where available;
- option kline/history where available;
- option chain metadata;
- expiry, strike, option side, and underlying mapping;
- implied-volatility and Greeks context where available.

Do not use Binance options for:

- customer-facing order placement;
- low-liquidity recommendations without explicit degradation labels;
- fake options momentum on non-options markets.

Options required behavior:

1. Maintain a separate options symbol registry.
2. Preserve underlying, expiry, strike, option side, and contract multiplier.
3. Preserve bid/ask, mark price, implied volatility, delta, gamma, theta, vega,
   and open interest when available.
4. Mark each missing options field as null, not zero.
5. Compute options-specific momentum only for options contracts.
6. Expose option-chain freshness and liquidity warnings to admin and customer
   surfaces.

### Binance Shared Requirements

Shared Binance behavior:

1. Keep public market-data streams separate from private account streams.
2. Public market-data streams should work without account keys.
3. Private account streams require admin-only API keys and must never power
   customer wallet/funds screens.
4. Every stream reports connected, reconnecting, stale, or offline state.
5. Every stream reports last event time and last normalized candle time.
6. Symbol add/remove/pause must not require deploys.
7. Provider rate-limit and disconnect behavior must degrade visibly.

Default benchmark behavior:

- Use BTCUSDT as the default spot benchmark.
- Use the matching BTC futures contract as the default futures benchmark when
  futures mode is active.
- Use the underlying spot or futures symbol as the benchmark for options
  contracts.
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
