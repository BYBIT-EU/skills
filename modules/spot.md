# Module: Spot Trading

> Loaded on-demand by the Bybit EU Trading Skill. Requires the account module conceptually (wallet balance, collateral info, fee rate). All endpoints are `category=spot` on `api.bybit.eu`.

## Scenario: Spot order lifecycle

Common user intents this module handles:

- "Buy 0.01 BTC with a limit order at 60000 USDT" → `POST /v5/order/create`
- "Change my limit order price to 61000" → `POST /v5/order/amend`
- "Cancel order #abc123" → `POST /v5/order/cancel`
- "Show all my open BTCUSDT orders" → `GET /v5/order/realtime`
- "What trades did I execute today?" → `GET /v5/execution/list`
- "Place 5 limit orders at once" → `POST /v5/order/create-batch`
- "How much BTC can I buy including margin?" → `GET /v5/order/spot-borrow-check`

## API Reference

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Place Order | `/v5/order/create` | POST | Yes | `category=spot`, `symbol`, `side`, `orderType`, `qty` | `isLeverage`, `marketUnit`, `price`, `orderFilter`, `triggerPrice`, `timeInForce`, `orderLinkId`, `takeProfit`, `stopLoss`, `tpLimitPrice`, `slLimitPrice`, `tpOrderType`, `slOrderType`, `slippageToleranceType`, `slippageTolerance` | Market buy qty is quote-denominated by default; set `marketUnit=baseCoin` to use base coin. `isLeverage=1` enables spot margin (see margin module). Open-order limit: 500 total, max 30 TP/SL, max 30 conditional per symbol. |
| Amend Order | `/v5/order/amend` | POST | Yes | `category=spot`, `symbol`, `orderId` or `orderLinkId` | `triggerPrice`, `qty`, `price`, `takeProfit`, `stopLoss`, `tpLimitPrice`, `slLimitPrice` | Modifies unfilled or partially filled orders only. Pass `0` for `takeProfit`/`stopLoss` to cancel them. |
| Cancel Order | `/v5/order/cancel` | POST | Yes | `category=spot`, `symbol`, `orderId` or `orderLinkId` | `orderFilter` | Must specify `orderId` or `orderLinkId`. `orderId` takes precedence when both provided. |
| Cancel All Orders | `/v5/order/cancel-all` | POST | Yes | `category=spot` | `symbol`, `orderFilter` | Cancels all open spot orders; filter by symbol or type. Response returns list of cancelled orders and a `success` flag (`1`=success, `0`=fail). |
| Batch Place Order | `/v5/order/create-batch` | POST | Yes | `category=spot`, `request[]` with each: `symbol`, `side`, `orderType`, `qty` | Per-item: `isLeverage`, `marketUnit`, `price`, `orderFilter`, `triggerPrice`, `timeInForce`, `orderLinkId`, `takeProfit`, `stopLoss`, `tpLimitPrice`, `slLimitPrice`, `tpOrderType`, `slOrderType` | Maximum 10 orders per request. Response has `result.list` (order info) and `retExtInfo.list` (per-order code/msg). |
| Batch Amend Order | `/v5/order/amend-batch` | POST | Yes | `category=spot`, `request[]` with each: `symbol`, `orderId` or `orderLinkId` | Per-item: `triggerPrice`, `qty`, `price`, `takeProfit`, `stopLoss`, `tpLimitPrice`, `slLimitPrice` | Maximum 10 orders per request. Conditional orders (`orderFilter=StopOrder`) are **not supported** for batch amend; `triggerPrice` still applies to `tpslOrder` items. |
| Batch Cancel Order | `/v5/order/cancel-batch` | POST | Yes | `category=spot`, `request[]` with each: `symbol`, `orderId` or `orderLinkId` | — | Maximum 10 orders per request. `orderId` takes precedence over `orderLinkId` when both provided. |
| Get Open & Closed Orders | `/v5/order/realtime` | GET | Yes | `category=spot` | `symbol`, `baseCoin`, `orderId`, `orderLinkId`, `openOnly`, `orderFilter`, `limit`, `cursor` | `openOnly=0` (default): open orders only (New, PartiallyFilled); `openOnly=1`: up to 500 recent closed orders (Cancelled, Rejected, Filled) per account — this cache is **cleared on server restart/release**, after which historical closed orders are only available via `/v5/order/history`. When `orderId` or `orderLinkId` is provided, `openOnly` and other filters are ignored (filter precedence: `orderId` > `orderLinkId` > `symbol` > `baseCoin`). Sorted by `createdTime` descending. |
| Get Order History | `/v5/order/history` | GET | Yes | `category=spot` | `symbol`, `baseCoin`, `orderId`, `orderLinkId`, `orderFilter`, `orderStatus`, `startTime`, `endTime`, `limit`, `cursor` | Up to 2 years of data. Time-range rules: (1) **Last 7 days** — all closed statuses queryable EXCEPT `Cancelled`, `Rejected`, `Deactivated`; (2) **Last 24 hours** — `Cancelled` (fully cancelled), `Rejected`, `Deactivated` are ALSO queryable; (3) **Older than 7 days** — only orders with fills (fully filled, or partially filled then cancelled). When both `startTime` & `endTime` are passed the range must be ≤ 7 days. `extraFees` populated for spot fiat currency orders on the EU site. |
| Get Trade History | `/v5/execution/list` | GET | Yes | `category=spot` | `symbol`, `orderId`, `orderLinkId`, `baseCoin`, `startTime`, `endTime`, `execType`, `limit [1,100] default 50`, `cursor` | Sorted by `execTime` descending. One order may produce multiple execution records. `limit` range is `[1,100]` default `50` (distinct from the order endpoints' `[1,50]` default `20`). `extraFees` populated for spot fiat currency orders on the EU site. |
| Set Disconnect Cancel All | `/v5/order/disconnected-cancel-all` | POST | Yes | `timeWindow` | `product` | Configures DCP: auto-cancels all active spot orders when the private WebSocket disconnects. `timeWindow` range [3, 300] seconds. `product` defaults to `SPOT`. Query current DCP config via `/v5/account/query-dcp-info`. |
| Get Borrow Quota | `/v5/order/spot-borrow-check` | GET | Yes | `category=spot`, `symbol`, `side` | — | Returns `maxTradeQty`/`maxTradeAmount` (includes borrowable if spot margin on) and `spotMaxTradeQty`/`spotMaxTradeAmount` (actual available balance only). |

> **NOTE — DMM Listing:** `POST /v5/order/create` is also used by Designated Market Makers (DMMs) for pre-market listing on Bybit EU. DMM UID must be whitelisted for the instrument. Place `PostOnly` limit orders to ensure maker intent. Validate instrument visibility via `GET /v5/market/instruments-info` before trading. WebSocket trading is recommended over REST for lower overhead.

## Key Rules

- All trading uses `category=spot`. No other categories are supported on Bybit EU.
- **Mainnet write operations** (`POST /v5/order/create`, `/v5/order/amend`, `/v5/order/cancel`, `/v5/order/cancel-all`, and batch variants) are CONFIRM-gated on mainnet. The skill will ask for confirmation before executing writes (see SKILL.md security flow).
- **Market buy orders** are denominated in quote currency by default (e.g. buying BTCUSDT with `qty=100` means spend 100 USDT). Set `marketUnit=baseCoin` to denominate in base currency.
- **Spot margin**: set `isLeverage=1` to enable margin borrowing on a spot order. The spot margin mode must be active and the currency set as collateral first — see the margin module.
- **TP/SL params**: `takeProfit`/`stopLoss` set trigger prices; `tpOrderType`/`slOrderType` determine the fill order type (`Market` or `Limit`); `tpLimitPrice`/`slLimitPrice` are required when using limit TP/SL. Pass `0` to cancel an existing TP or SL on amend.
- **Conditional orders** (`orderFilter=StopOrder`): assets are not reserved until the trigger fires. If there are insufficient funds at trigger time the order is cancelled.
- **tpslOrder** (`orderFilter=tpslOrder`): assets are occupied before trigger.
- **OcoOrder** / **BidirectionalTpslOrder**: supported in `cancel-all`, `realtime`, and `history` filter queries.
- **Open order limits (Spot)**: 500 total open orders per account; maximum 30 open TP/SL orders and 30 open conditional orders per symbol per account.
- **Batch endpoints** accept a maximum of 10 orders per request.
- All write operations are acknowledged synchronously but executed asynchronously. Use the WebSocket order stream for real-time status.
- **Order query priority**: `orderId` > `orderLinkId` > `symbol` > `baseCoin`. When `orderId` or `orderLinkId` is provided to `/v5/order/realtime`, `openOnly` and all other filters are ignored.
- **`openOnly=1` cache**: returns up to 500 recent closed orders (Cancelled/Rejected/Filled) per account. This cache is **cleared on server restart/release** — after a restart, historical closed orders are only available via `/v5/order/history`.
- **Order history time range** (three-tier rule):
  - **Last 7 days**: all closed statuses queryable EXCEPT `Cancelled`, `Rejected`, `Deactivated`.
  - **Last 24 hours**: `Cancelled` (fully cancelled), `Rejected`, `Deactivated` are ALSO queryable.
  - **Older than 7 days**: only orders with fills (fully filled, or partially filled then cancelled).
  - When both `startTime` & `endTime` are passed, the range must be ≤ 7 days.
- **DCP (Disconnect Cancel All)**: only available for Ins clients; VIP clients cannot access. The private WebSocket connection must subscribe to the `dcp` topic. Allow 10 seconds after configuration before querying or reconfiguring.
- `orderLinkId` must be unique and at most 36 characters (numbers, letters, dashes, underscores).
- During extreme market volatility, order and execution query endpoints may experience increased latency.

## Enums

**`category`**: `spot`

**`side`**: `Buy`, `Sell`

**`orderType`**: `Market`, `Limit`

**`timeInForce`**: `GTC`, `IOC`, `FOK`, `PostOnly`
- Market orders always use `IOC`.
- `PostOnly` is cancelled immediately if it would fill, ensuring a resting maker order.
- Default is `GTC`.

**`orderFilter`** (Place/Cancel/Query): `Order`, `tpslOrder`, `StopOrder`, `OcoOrder`, `BidirectionalTpslOrder`
- `Order`: regular active order (default)
- `tpslOrder`: TP/SL order; assets occupied before trigger
- `StopOrder`: conditional order; assets occupied only after trigger
- `OcoOrder`: one-cancels-the-other order
- `BidirectionalTpslOrder`: bidirectional TP/SL order

**`marketUnit`**: `baseCoin`, `quoteCoin`

**`slippageToleranceType`** (Place Order): `TickSize`, `Percent`
- `TickSize`: buy max = ask1 + tolerance × tickSize; sell min = bid1 − tolerance × tickSize; `slippageTolerance` integer [1, 10000]
- `Percent`: buy max = ask1 × (1 + tolerance × 0.01); sell min = bid1 × (1 − tolerance × 0.01); `slippageTolerance` [0.01, 10] up to 2 decimals
- Not supported on TP/SL or conditional orders.

**`tpOrderType`** / **`slOrderType`**: `Market`, `Limit`

**`product`** (Set Disconnect Cancel All): `SPOT`

**`openOnly`** (Get Open & Closed Orders): `0` (open orders only), `1` (up to 500 recent closed orders)
