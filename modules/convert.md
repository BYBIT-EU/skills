# Module: Convert

> Loaded on-demand by the Bybit EU Trading Skill. Authentication required for all endpoints (API key permission: **Exchange** for crypto convert; **Convert** for small-balance). All endpoints operate on `api.bybit.eu`. Mainnet write operations (execute/confirm quote steps) are CONFIRM-gated per SKILL.md.

## Scenario: Crypto convert, small-balance sweep, and fiat on/off-ramp

Common user intents this module handles:

- "What coins can I convert from my UNIFIED wallet?" → `GET /v5/asset/exchange/query-coin-list`
- "Get me a quote to swap 100 USDT into BTC" → `POST /v5/asset/exchange/quote-apply`
- "Confirm and execute the convert quote" → `POST /v5/asset/exchange/convert-execute`
- "Check the status of my last convert (quoteTxId XYZ)" → `GET /v5/asset/exchange/convert-result-query`
- "Show my full crypto convert history" → `GET /v5/asset/exchange/query-convert-history`
- "Which of my small-balance coins can I sweep to USDT?" → `GET /v5/asset/covert/small-balance-list`
- "Get a quote to convert my small balances (BTC, XRP) to USDT" → `POST /v5/asset/covert/get-quote`
- "Execute the small-balance convert quote" → `POST /v5/asset/covert/small-balance-execute`
- "Show my small-balance convert history" → `GET /v5/asset/covert/small-balance-history`
- "What fiat/crypto pairs can I trade?" → `GET /v5/fiat/query-coin-list`
- "What is the reference price for EUR-USDT?" → `GET /v5/fiat/reference-price`
- "Get a fiat convert quote to buy BTC with EUR" → `POST /v5/fiat/quote-apply`
- "Execute the fiat convert quote" → `POST /v5/fiat/trade-execute`
- "Check the status of my fiat trade" → `GET /v5/fiat/trade-query`
- "Show my fiat convert history" → `GET /v5/fiat/query-trade-history`
- "Show my current fiat balance" → `GET /v5/fiat/balance-query`

## API Reference

### Regular Crypto Convert (`/v5/asset/exchange/*`)

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Convert Coin List | `/v5/asset/exchange/query-coin-list` | GET | Yes | `accountType` | `coin`, `side` | `accountType`: `eb_convert_funding` or `eb_convert_uta`. `side=0`: fromCoin list (balance given if held; coin ignored); `side=1`: toCoin list (requires `coin`). `singleFromMinLimit`/`singleFromMaxLimit` define per-transaction limits. `accuracyLength` gives coin precision. Rate limit 100 req/s. API key permission: Exchange. |
| Request a Quote | `/v5/asset/exchange/quote-apply` | POST | Yes | `accountType`, `fromCoin`, `toCoin`, `requestCoin`, `requestAmount` | `fromCoinType`, `toCoinType`, `paramType`, `paramValue`, `requestId` | `requestCoin` must equal `fromCoin` (err 700002 if set to `toCoin`). Quote valid for **15 seconds** — `expiredTime` returned. Balance pre-checked at this stage. If real-time rate vs quoted rate differ > 0.5% at confirm, exchange is rejected (err 32024). Rate limit 50 req/s. API key permission: Exchange. **Read-only POST — no CONFIRM required.** |
| Confirm a Quote | `/v5/asset/exchange/convert-execute` | POST | Yes | `quoteTxId` | — | Executes the conversion asynchronously. Must be called before the quote expires (15 s). Calling more than once returns err 700009 (quoteTxId already used). Can fail if funds were transferred out after quote. Poll `/v5/asset/exchange/convert-result-query` for final status. Rate limit 50 req/s. API key permission: Exchange. **CONFIRM-gated on mainnet.** |
| Get Convert Status | `/v5/asset/exchange/convert-result-query` | GET | Yes | `quoteTxId`, `accountType` | — | Query final status and details of a convert by `quoteTxId`. `exchangeStatus`: `init`, `processing`, `success`, `failure`. Err 700004 if `quoteTxId` incorrect or mismatched with `accountType`. Rate limit 100 req/s. API key permission: Exchange. |
| Get Convert History | `/v5/asset/exchange/query-convert-history` | GET | Yes | — | `accountType`, `index`, `limit` | Returns all confirmed convert records. `accountType` supports comma-separated multiple types (e.g. `eb_convert_funding,funding`); all types if omitted. `index` is page number starting from 1. `limit` default 20, up to 100. From Sep 10 2025 web-executed converts also appear here. Rate limit 100 req/s. API key permission: Exchange. |

### Small-Balance Convert (`/v5/asset/covert/*`)

> **NOTE:** The URL path prefix `covert` (not `convert`) is the **real, intentional API path** used by Bybit. This spelling must be preserved exactly when making requests.

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Small Balance Coins | `/v5/asset/covert/small-balance-list` | GET | Yes | `accountType` | `fromCoin` | Returns coins with USDT-equivalent value under 10 USDT. Only Unified wallet (`eb_convert_uta`) supported. `supportToCoins` returns `["MNT","USDT","USDC"]`. Total per-transaction amount must be between 1.0e-8 and 200 USDT. `toAmount`/`exchangeRate`/`feeInfo`/`taxFeeInfo` in response are reserved — ignore. Rate limit 10 req/s. API key permission: Convert. |
| Request a Quote | `/v5/asset/covert/get-quote` | POST | Yes | `accountType`, `fromCoinList`, `toCoin` | — | `fromCoinList` is array of up to 20 source coin names. `toCoin` must be one of `MNT`, `USDT`, or `USDC`. Only Unified wallet (`eb_convert_uta`) supported. Quote valid for **30 seconds** — `expireTime` returned. Custody accounts (Copper, Fireblock, etc.) not supported. In UTA, simultaneous multi-coin converts may partially execute. Includes `taxFeeInfo`/`totalTaxFeeInfo` EU/MiCA compliance fields. Rate limit 5 req/s. API key permission: Convert. **Read-only POST — no CONFIRM required.** |
| Confirm a Quote | `/v5/asset/covert/small-balance-execute` | POST | Yes | `quoteId` | — | Executes the small-balance conversion asynchronously. Must be called before quote expires (30 s). `exchangeTxId` equals `quoteId`. `status`: `init`, `processing`, `success`, `failure`, `partial_fulfillment`. Poll `/v5/asset/covert/small-balance-history` for final status. Rate limit 5 req/s. API key permission: Convert. **CONFIRM-gated on mainnet.** |
| Get Exchange History | `/v5/asset/covert/small-balance-history` | GET | Yes | — | `accountType`, `quoteId`, `startTime`, `endTime`, `cursor`, `size` | Can query records from API or web/app, both Unified and Funding wallets. `quoteId` has highest priority when querying. `size` default 50, max 100. Each record contains `subRecords` per fromCoin plus `taxFeeInfo`/`totalTaxFeeInfo` EU/MiCA compliance fields. `exchangeSource`: `small_asset_uta`, `small_asset_funding`. Rate limit 10 req/s. API key permission: Convert. |

### Fiat Convert (`/v5/fiat/*`)

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Trading Pair List | `/v5/fiat/query-coin-list` | GET | Yes | — | `side` | `side=0`: buy (buy crypto, sell fiat); `side=1`: sell (sell crypto, buy fiat). Returns `fiats[]` and `cryptos[]` with precision, disable flag, and `singleFromMin/MaxLimit`. Example fiats: EUR, GBP, GEL; example cryptos: USDT, BTC, ETH. |
| Get Reference Price | `/v5/fiat/reference-price` | GET | Yes | `symbol` | — | `symbol` format: `fiat-crypto`, e.g. `EUR-USDT`. Returns `buys[]` and `sells[]` with `unitPrice` (1 crypto = x fiat) and `paymentMethod`. EU/SEPA payment methods supported (e.g. Cash Balance, SEPA, Apple Pay, Google Pay, Credit Card). |
| Request a Quote | `/v5/fiat/quote-apply` | POST | Yes | `fromCoin`, `fromCoinType`, `toCoin`, `toCoinType`, `requestAmount` | `requestCoinType` | `fromCoinType`/`toCoinType`: `fiat` or `crypto`. `requestCoinType` defaults to `fiat`. Master UID API key only. Response includes `quoteTxId` (used for confirm) and `expiredTime` (ms timestamp). The fiat quote expiry duration is NOT documented — honor the `expiredTime` value returned in the response; do not assume a fixed window. **Read-only POST — no CONFIRM required.** |
| Confirm a Quote | `/v5/fiat/trade-execute` | POST | Yes | `quoteTxId`, `subUserId` | `webhookUrl`, `MerchantRequestId` | Executes the fiat convert asynchronously. `quoteTxId` from `/v5/fiat/quote-apply`. `webhookUrl` (max 256 chars) triggers the Trade Notify webhook callback on order completion/failure. `MerchantRequestId` max 36 chars. **CONFIRM-gated on mainnet.** |
| Get Convert Status | `/v5/fiat/trade-query` | GET | Yes | — *(one of the below required)* | `tradeNo`, `merchantRequestId` | Exactly one of `tradeNo` or `merchantRequestId` must be provided. Returns single trade object with `status`, `exchangeRate`, from/to coin details, amounts, `createdAt`, `subUserId`. `status`: `processing`, `success`, `failed`. |
| Get Convert History | `/v5/fiat/query-trade-history` | GET | Yes | — | `index`, `limit`, `startTime`, `endTime` | Paginated fiat convert trade history. `index` starts at 1 (default 1). `limit` range 20–100, default 20, capped at 100. `startTime`/`endTime` are millisecond timestamps. `status`: `processing`, `success`, `failed`. |
| Get Balance | `/v5/fiat/balance-query` | GET | Yes | — | `currency` | `currency` uses ISO 4217 fiat codes (e.g. `EUR`, `GBP`). Omit to return all fiat balances. Response: `totalBalance`, `balance` (available), `frozenBalance`, `currency` per fiat. |

## Notes

> **NOTE — Convert Guideline** (no REST endpoint — workflow reference only):
> The 4-step crypto convert workflow is:
> 1. **Get Convert Coin List** (`/v5/asset/exchange/query-coin-list`) — verify fromCoin and toCoin availability.
> 2. **Request a Quote** (`/v5/asset/exchange/quote-apply`) — balance pre-checked; `quoteTxId` and `expiredTime` returned. Quote expires in **15 seconds**.
> 3. **Confirm a Quote** (`/v5/asset/exchange/convert-execute`) — must be called within 15 s using `quoteTxId`; execution is asynchronous.
> 4. **Get Convert Status** (`/v5/asset/exchange/convert-result-query`) — poll until `exchangeStatus=success` or `failure`.
>
> Always surface `expireTime`/`expiredTime` and `quoteTxId` in user-facing confirmations so the user understands the execution deadline.
>
> **Error codes:** 32024 = rate threshold exceeded (real-time rate vs quoted rate > 0.5%); 700002 = `requestCoin` must equal `fromCoin`; 700006 = amount below min limit; 700007 = amount above max limit; 700009 = `quoteTxId` already used; 700010 = INS loan user cannot convert; 700011 = illegal operation (quoted with user A, confirmed with user B).
>
> **Rate limits (not upgradable):** `query-coin-list` 100/s; `quote-apply` 50/s; `convert-execute` 50/s; `convert-result-query` 100/s; `query-convert-history` 100/s.

> **NOTE — Trade Notify** (webhook callback — not a callable REST endpoint):
> Bybit POSTs a JSON payload to the `webhookUrl` you supply in `/v5/fiat/trade-execute` whenever a fiat-convert trade changes status. This is an **inbound webhook** that your server must receive, not an API you call.
>
> Webhook body fields:
>
> | Field | Type | Notes |
> |-------|------|-------|
> | `tradeNo` | string | Trade order number |
> | `status` | string | `processing`, `success`, or `failed` |
> | `quoteTxId` | string | Quote transaction ID |
> | `exchangeRate` | string | Executed exchange rate |
> | `fromCoin` | string | Coin sold |
> | `fromCoinType` | string | `fiat` or `crypto` |
> | `toCoin` | string | Coin bought |
> | `toCoinType` | string | `fiat` or `crypto` |
> | `fromAmount` | string | Amount sold |
> | `toAmount` | string | Amount received |
> | `createdAt` | string | Trade creation timestamp |
>
> Authentication: HTTP headers include `Content-Type`, `timestamp`, and `publicKey`. Bybit uses RSA_SHA256 to sign callbacks. Secure your webhook endpoint using IP whitelisting.

## Key Rules

- **Quote-then-execute model**: For ALL three convert sub-systems (crypto, small-balance, fiat), always request a quote first to obtain a `quoteTxId` / `quoteId`, then confirm within the expiry window. The quote request is read-only (POST but no state change) — no CONFIRM required. The execute/confirm step is a mainnet write operation — **CONFIRM-gated**.
- **Expiry windows**: Crypto convert quotes expire in **15 seconds** (`expiredTime`); small-balance quotes expire in **30 seconds** (`expireTime`); fiat quote expiry duration is NOT documented — always honor the `expiredTime` (ms) value returned in the `/v5/fiat/quote-apply` response; do not assume a fixed window. Always display this deadline to the user before asking for CONFIRM.
- **`covert` path spelling is intentional**: The small-balance API paths use `/v5/asset/covert/` (not `/v5/asset/convert/`). This is the real Bybit API path — preserve this spelling exactly when constructing requests.
- **API key permissions differ**: Crypto convert endpoints require the **Exchange** permission; small-balance convert endpoints require the **Convert** permission. Fiat convert (`/v5/fiat/*`): API-key permission is not specified in the EU docs — verify your key's permissions against the Bybit EU fiat API documentation.
- **Small-balance only supports Unified wallet**: `accountType` must be `eb_convert_uta`; Funding wallet is not supported for small-balance operations.
- **Small-balance `toCoin`** must be one of `MNT`, `USDT`, or `USDC`; up to 20 source coins per transaction; per-transaction USDT-equivalent value must be between 1.0e-8 and 200 USDT.
- **Crypto convert** uses `accountType`: `eb_convert_funding` (Funding wallet via API) or `eb_convert_uta` (Unified wallet via API).
- **Fiat convert** requires the master UID API key; `subUserId` is required on the confirm step (`/v5/fiat/trade-execute`).
- **Async execution**: All three execute/confirm endpoints are asynchronous. After calling the execute endpoint, poll the corresponding status/history endpoint for the final result — do not assume immediate success.
- **EU/MiCA compliance fields**: Small-balance responses include `taxFeeInfo` and `totalTaxFeeInfo` for EU tax reporting. Fiat convert is subject to EU regulatory requirements.
- **Spot-to-Convert fallback**: For base-base pairs (e.g. BTCDOGE) or unlisted spot symbols, do NOT call `/v5/order/create`. Route to this module's crypto convert flow.
- **Custody accounts not supported** for small-balance convert (Copper, Fireblock, etc.).
- **Convert history** (crypto): starting September 10, 2025, web/app-executed converts also appear in `/v5/asset/exchange/query-convert-history` alongside API-executed ones.

## Enums

**`accountType`** (crypto convert): `eb_convert_funding`, `eb_convert_uta`

**`accountType`** (convert history extended): `eb_convert_funding`, `eb_convert_uta`, `funding`, `funding_fiat`, `funding_fbtc_convert`, `funding_block_trade`

**`accountType`** (small-balance): `eb_convert_uta`

**`side`** (crypto coin list): `0` = fromCoin list, `1` = toCoin list

**`side`** (fiat coin list): `0` = buy (buy crypto, sell fiat), `1` = sell (sell crypto, buy fiat)

**`exchangeStatus`** (crypto convert): `init`, `processing`, `success`, `failure`

**`status`** (small-balance): `init`, `processing`, `success`, `failure`, `partial_fulfillment`

**`exchangeSource`** (small-balance history): `small_asset_uta`, `small_asset_funding`

**`status`** (fiat convert): `processing`, `success`, `failed`

**`toCoin`** (small-balance quote): `MNT`, `USDT`, `USDC`

**`fromCoinType` / `toCoinType` / `requestCoinType`** (fiat): `fiat`, `crypto`

**`supportConvert`** (small-balance): `1`, `2`
