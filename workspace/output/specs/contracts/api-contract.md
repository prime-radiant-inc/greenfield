# StockShots API Contract

Status: implementation brief

## Authentication

Customer APIs require an authenticated user or a valid product key. Admin APIs
require admin entitlement. Public unauthenticated access may only expose static
marketing or login pages.

## Customer APIs

### Markets

`GET /v1/markets`

Returns supported markets, freshness, and availability.

### Symbols

`GET /v1/symbols?exchange={exchange}`

Returns tradable/watchable symbols for a market.

### Candles

`GET /v1/candles?exchange={exchange}&symbol={symbol}&timeframe={timeframe}&from={timestamp}&to={timestamp}&limit={count}`

Returns continuous candle rows. The API hides storage partitioning.

Response shape:

- `exchange`
- `symbol`
- `timeframe`
- `source`
- `freshness_state`
- `candles`

Each candle contains:

- `timestamp`
- `open`
- `high`
- `low`
- `close`
- `volume`
- optional indicator fields if requested.

### Indicators

`GET /v1/indicators?exchange={exchange}&symbol={symbol}&timeframe={timeframe}&limit={count}`

Returns indicator history for chart tapes and detail panels.

### Screening

`POST /v1/screen`

Request:

- `exchange`
- `timeframe`
- `mode`: `sort`, `filter`, or `filter_sort`
- `fields`: list of indicator fields
- `thresholds`: optional minimum/maximum values
- `limit`

Response:

- matching rows;
- visible explanation of matched fields;
- freshness state;
- degraded reason if applicable.

### Alert Rules

`GET /v1/alerts`

`POST /v1/alerts`

`DELETE /v1/alerts/{id}`

Alert rules use the same field-combo model as screening.

### Earnings

`GET /v1/earnings?market={market}&from={date}&to={date}`

Returns earnings/company-event rows plus symbol signal context when available.

### Health

`GET /v1/health`

Returns market data freshness, worker health, and degraded reasons safe for
customer display.

## Admin APIs

### Ingestion Health

`GET /v1/admin/ingestion`

Returns provider connections, last tick times, lag, reconnects, dropped
messages, and active symbol counts.

### Worker Health

`GET /v1/admin/workers`

Returns worker heartbeats, queue lag, job failures, and write lag.

### Bot State

`GET /v1/admin/bots`

Returns bot state, tracked symbols, paper/live mode, open decisions, and recent
actions.

### Broker State

`GET /v1/admin/positions`

`GET /v1/admin/pnl`

`GET /v1/admin/funds`

Admin-only broker state. Must never be available to customer apps.

### Symbol Registry

`POST /v1/admin/symbols`

`DELETE /v1/admin/symbols/{exchange}/{symbol}`

`POST /v1/admin/symbols/{exchange}/{symbol}/pause`

Controls realtime symbol subscriptions.
