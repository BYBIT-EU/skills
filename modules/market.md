# Module: Market Data

> Loaded on-demand by the Bybit EU Trading Skill. No authentication required. All endpoints are `category=spot` on `api.bybit.eu`.

## Scenario: Spot market data queries

Common user intents this module handles:

- "What is the current price of BTCUSDT?" → `GET /v5/market/tickers`
- "Show me the order book for ETHUSDT with depth 20" → `GET /v5/market/orderbook`
- "Get the last 100 candles for BTCUSDT on the 1-hour chart" → `GET /v5/market/kline`
- "What instruments are available on Bybit EU spot?" → `GET /v5/market/instruments-info`

## API Reference

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Instruments Info | `/v5/market/instruments-info` | GET | No | `category=spot` | `symbol`, `symbolType`, `status` | Returns precision, order limits, price filters. `symbolType=xstocks` for tokenised stocks. |
| Get Kline | `/v5/market/kline` | GET | No | `symbol`, `interval` | `category`, `start`, `end`, `limit` | `category` defaults to `spot`. `limit` [1,1000], default 200. Sorted descending by startTime. |
| Get Order Price Limit | `/v5/market/price-limit` | GET | No | `symbol` | `category` | Returns `buyLmt` (highest bid price) and `sellLmt` (lowest ask price) for the symbol. |
| Get Orderbook | `/v5/market/orderbook` | GET | No | `category=spot`, `symbol` | `limit` | `limit` [1,1000], default 1. Bids descending; asks ascending. Each entry is [price, size]. |
| Get Recent Public Trades | `/v5/market/recent-trade` | GET | No | `category=spot`, `symbol` | `limit` | `limit` [1,60], default 60. Response includes `isRPITrade` flag (EU-specific). |
| Get Tickers | `/v5/market/tickers` | GET | No | `category=spot` | `symbol` | Omit `symbol` to return all spot tickers. Includes 24h OHLCV, best bid/ask, last price. |
| Get Server Time | `/v5/market/time` | GET | No | — | — | No request params. Returns `timeSecond` and `timeNano`. |

## Key Rules

- All market data endpoints are **public** — no API key required.
- All endpoints on Bybit EU are **`category=spot` only**. Derivatives and options categories are not supported.
- **Kline** candle array order: `[startTime, openPrice, highPrice, lowPrice, closePrice, volume, turnover]`. The `closePrice` is the last traded price when the candle is not yet closed. Results sorted descending by `startTime`.
- **Orderbook** `limit` range is [1, 1000], default 1. The `u` (Update ID) field corresponds to the 1000-level WebSocket orderbook stream; `seq` is the cross-sequence number (smaller = earlier).
- **Recent trades** `limit` range is [1, 60], default 60. `side` indicates the taker side.
- **Instruments Info**: `maxLimitOrderQty`, `maxMarketOrderQty`, and `postOnlyMaxLimitOrderSize` are adjusted bi-monthly (3rd and 17th of each month, 08:00 UTC+8) — do not cache these values long-term.
- During extreme market volatility, Instruments Info and Tickers may experience increased latency or data delays.
- **EU tokenised stocks**: `symbolType=xstocks` identifies tokenised stock trading pairs. `xstockMultiplier` converts between token and underlying stock units: `stock_price = token_price / multiplier`, `stock_qty = token_qty * multiplier`. Default multiplier is 1. The `innovation` field is deprecated in favour of `symbolType`.
- `stTag` in Instruments Info references the Bybit EU special-treatment (ST) label applied to certain pairs.

## Enums

**`category`** (all endpoints): `spot`

**`interval`** (Get Kline): `1`, `3`, `5`, `15`, `30`, `60`, `120`, `240`, `360`, `720`, `D`, `W`, `M`

**`status`** (Get Instruments Info): `Trading`

**`side`** (Get Recent Public Trades response): `Buy`, `Sell`
