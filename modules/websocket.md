# Module: WebSocket Streams

> Loaded on-demand by the Bybit EU Trading Skill. **Private streams require WSS authentication.** Public streams need no auth. All spot streams use `category=spot` on `stream.bybit.eu`.

## Scenario: Real-time data and order management over WebSocket

Common user intents this module handles:

- "Stream live BTCUSDT price updates" → subscribe `tickers.BTCUSDT` on `wss://stream.bybit.eu/v5/public/spot`
- "Subscribe to my order updates in real time" → authenticate then subscribe `order` on `wss://stream.bybit.eu/v5/private`
- "Place an order over WebSocket" → authenticate on `wss://stream.bybit.eu/v5/trade`, send `order.create`
- "Monitor wallet balance changes live" → authenticate then subscribe `wallet` on private stream
- "Watch the ETHUSDT order book depth 50" → subscribe `orderbook.50.ETHUSDT` on public spot stream

## Connection Endpoints

| Stream Type | URL |
|-------------|-----|
| Public spot streams | `wss://stream.bybit.eu/v5/public/spot` |
| Private streams (orders, executions, wallet, DCP) | `wss://stream.bybit.eu/v5/private` |
| WebSocket order entry (trade) | `wss://stream.bybit.eu/v5/trade` |
| System status | `wss://stream.bybit.eu/v5/public/misc/status` |
| Demo private streams | `wss://stream-demo.bybit.eu/v5/private` |

## Connection & Authentication

### WSS Authentication Handshake

Private streams (and the `/v5/trade` channel) require an `auth` op before subscribing. Send:

```json
{
  "op": "auth",
  "args": ["<apiKey>", <expires>, "<signature>"]
}
```

- `expires` — millisecond timestamp with a short window ahead of current time (e.g. `Date.now() + 5000`)
- `signature` — `HMAC_SHA256(secret, "GET/realtime" + expires)` (hex-encoded)

On success the server responds `{"op":"auth","success":true}`.

### Heartbeat (Ping/Pong)

Send `{"op":"ping"}` every 20 seconds to keep the connection alive. The server replies with `{"op":"pong","success":true}`. Without ping/pong (or incoming stream data) the connection disconnects after 10 minutes by default. Tune via `max_active_time` query parameter (min 30 s, max 600 s; also accepts `1m`, `2m`).

### Subscribe / Unsubscribe

```json
{"op": "subscribe", "args": ["tickers.BTCUSDT", "orderbook.50.ETHUSDT"]}
{"op": "unsubscribe", "args": ["tickers.BTCUSDT"]}
```

Optional `req_id` field is echoed in the response when provided.

## Public Streams Reference

| Stream | Topic Format | Auth | Push Frequency | Key Fields |
|--------|-------------|------|----------------|------------|
| Kline | `kline.{interval}.{symbol}` | No | 1–60 s | `confirm` (candle closed), open/high/low/close/volume |
| Order Price Limit | `priceLimit.{symbol}` | No | 300 ms | `buyLmt` (highest bid), `sellLmt` (lowest ask) |
| Orderbook | `orderbook.{depth}.{symbol}` | No | 10 ms (L1), 20 ms (L50), 100 ms (L200), 200 ms (L1000) | `b` (bids), `a` (asks), `u` (update ID), `seq`, `cts` |
| Ticker | `tickers.{symbol}` | No | 50 ms | `lastPrice`, `highPrice24h`, `lowPrice24h`, `volume24h`, `price24hPcnt` |
| Trade | `publicTrade.{symbol}` | No | Real-time | `T` (fill ts ms), `s` (symbol), `S` (taker side), `v` (size), `p` (price), `i` (trade ID) |
| System Status | `system.status` | No | On change | `id`, `title`, `state`, `begin`/`end` (ms), `maintainType` |

## Private Streams Reference

| Stream | Topic | Auth | Push Frequency | Key Fields |
|--------|-------|------|----------------|------------|
| DCP (Disconnect Cancel Protection) | `dcp.spot` | Yes | On event | Arms DCP; triggered when **all** subscribing connections die |
| Execution | `execution` / `execution.spot` | Yes | Real-time | `execType`, `execFee`, `execQty`, `execPrice`, `seq`, `extraFees` (EU fiat orders) |
| Fast Execution | `execution.fast` / `execution.fast.spot` | Yes | Real-time | Trade type only; fewer fields, lower latency; `orderLinkId` always `""` for maker, and also `""` when a maker order is converted to taker via a price amend |
| Order | `order` / `order.spot` | Yes | Real-time | `orderId`, `orderStatus`, `qty`, `price`, `cumExecQty`, `triggerDirection`, `ocoTriggerBy` |
| Wallet | `wallet` | Yes | On balance change | `accountType=UNIFIED`, `totalEquity`, `coin[]` balances, `spotBorrow` |

## WS Trade (Order Entry)

Connect to `wss://stream.bybit.eu/v5/trade`. Authenticate the connection using the same `{"op":"auth","args":[apiKey, expires, signature]}` handshake as private streams — `X-BAPI-TIMESTAMP` is NOT part of the auth call. Separately, each order operation message (`order.create`, `order.amend`, etc.) includes a `header` object carrying `X-BAPI-TIMESTAMP` (current timestamp in ms) and optionally `X-BAPI-RECV-WINDOW`.

Supported `op` values:

| Op | Description |
|----|-------------|
| `auth` | Authenticate the connection |
| `order.create` | Place a single spot order (same params as `POST /v5/order/create`) |
| `order.amend` | Amend a single spot order |
| `order.cancel` | Cancel a single spot order |
| `order.create-batch` | Batch place up to 10 spot orders |
| `order.amend-batch` | Batch amend up to 10 spot orders |
| `order.cancel-batch` | Batch cancel up to 10 spot orders |
| `ping` | Heartbeat |

Request format:

```json
{
  "reqId": "<optional-unique-id-max-36-chars>",
  "header": {
    "X-BAPI-TIMESTAMP": "1700000000000",
    "X-BAPI-RECV-WINDOW": "5000"
  },
  "op": "order.create",
  "args": [{ "category": "spot", "symbol": "BTCUSDT", "side": "Buy", "orderType": "Limit", "qty": "0.01", "price": "30000" }]
}
```

An ack indicates request accepted only — use the private `order` stream to confirm final order status. Batch ops are asynchronous; account rate limits are shared between WS and HTTP batch endpoints.

## Key Rules

- **Public vs private**: public topics (`tickers.*`, `orderbook.*`, `publicTrade.*`, `kline.*`, `priceLimit.*`, `system.status`) need no auth. All private topics (`order`, `execution`, `execution.fast`, `wallet`, `dcp.spot`) require auth.
- **EU hosts**: always use `stream.bybit.eu` (the EU domain). Both the public and private base URLs are on `stream.bybit.eu`.
- **Connection limits**: do not open more than 500 new connections per 5 minutes per WebSocket domain; max 1,000 connections per IP for market data.
- **Subscription limits**: Spot allows up to 10 topic args per subscription request per connection; `args` array must not exceed 21,000 characters.
- **Orderbook management**: size `0` in a delta = delete entry; non-existent price level = insert; existing = update value. Reset local book on each snapshot. `u=1` means snapshot after service restart — overwrite local state.
- **Kline** `confirm=true` means candle is closed; `false` means still open/updating.
- **DCP** triggers only when **all** private connections subscribed to `dcp.spot` are dead (past the `timeWindow` threshold). A single live connection prevents the trigger.
- **Execution** `extraFees` is populated only for spot fiat currency orders placed on the EU site (EU MiCA compliance).
- **Order stream** `cumExecFee` is deprecated for spot; use `cumFeeDetail` or `execFee` from the Execution topic. `brokerOrderPrice` is a dedicated field for EU liquidity providers.
- **WS trade** supports Spot only; demo trading is not supported on `/v5/trade`.
- **System Status**: maintenance causing interruptions shorter than 10 seconds or a single WS disconnect (immediate reconnect) is NOT announced via this topic.
- **Wallet stream**: no snapshot is pushed on successful subscription. `usdValue` is `0` for coins that cannot be used as collateral.

## Enums

**`op`** (public/private connection): `auth`, `ping`, `subscribe`, `unsubscribe`

**`op`** (WS trade `/v5/trade`): `auth`, `order.create`, `order.amend`, `order.cancel`, `order.create-batch`, `order.amend-batch`, `order.cancel-batch`, `ping`

**Kline `interval`**: `1`, `3`, `5`, `15`, `30`, `60`, `120`, `240`, `360`, `720`, `D`, `W`, `M`

**Orderbook `depth`**: `1`, `50`, `200`, `1000`

**Trade taker side `S`**: `Buy`, `Sell`

**Order `side`**: `Buy`, `Sell`

**Order `orderType`**: `Market`, `Limit`

**Order `isLeverage`**: `0` (spot), `1` (margin)

**Order `ocoTriggerBy`**: `OcoTriggerByUnknown`, `OcoTriggerByTp`, `OcoTriggerBySl`

**Order `triggerDirection`**: `1` (rise), `2` (fall)

**Order `slippageToleranceType`**: `TickSize`, `Percent`, `UNKNOWN`

**Wallet `accountType`**: `UNIFIED`
