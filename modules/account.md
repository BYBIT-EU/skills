# Module: Account

> Loaded on-demand by the Bybit EU Trading Skill. Authentication required for all endpoints. All endpoints operate on `api.bybit.eu` under the Unified Trading Account (UTA).

## Scenario: Account management, balances, and configuration

Common user intents this module handles:

- "What is my wallet balance?" → `GET /v5/account/wallet-balance`
- "Show my transaction log for the last 7 days" → `GET /v5/account/transaction-log`
- "Switch my margin mode to portfolio margin" → `POST /v5/account/set-margin-mode`
- "Turn off collateral for ETH" → `POST /v5/account/set-collateral-switch`
- "How much can I transfer out for BTC and USDT?" → `GET /v5/account/withdrawal`
- "What is my fee rate for BTCUSDT?" → `GET /v5/account/fee-rate`
- "Repay all my spot margin liabilities" → `POST /v5/account/quick-repayment`
- "Get my trade analysis data for BTCUSDT this week" → `GET /v5/account/trade-info-for-analysis`

## API Reference

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Wallet Balance | `/v5/account/wallet-balance` | GET | Yes | `accountType=UNIFIED` | `coin` | Returns UTA wallet balance. Multiple coins allowed, comma-separated (e.g. `USDT,USDC`). Currencies with 0 assets/liabilities are not returned by default. `availableToWithdraw` is deprecated — use `/v5/account/withdrawal`. |
| Get Account Info | `/v5/account/info` | GET | Yes | — | — | Returns margin mode, account status, and UTA settings. Response fields `dcpStatus`, `timeWindow`, `smpGroup` are deprecated — use Get DCP Info and Get SMP Group ID instead. No `category` parameter. |
| Get Fee Rate | `/v5/account/fee-rate` | GET | Yes | `category=spot` | `symbol` | Returns `takerFeeRate` and `makerFeeRate` per spot symbol. |
| Get Collateral Info | `/v5/account/collateral-info` | GET | Yes | — | `currency` | Returns borrow rate, loanable amount, and collateral eligibility per coin. `collateralRatio` is deprecated — use Get Tiered Collateral Ratio. `freeBorrowingAmount` is deprecated, always empty. |
| Set Collateral Coin | `/v5/account/set-collateral-switch` | POST | Yes | `coin`, `collateralSwitch` | — | Toggle collateral on/off for a single coin. **USDT and USDC cannot be set.** No response body. |
| Batch Set Collateral Coin | `/v5/account/set-collateral-switch-batch` | POST | Yes | `request` (array of `{coin, collateralSwitch}`) | — | Toggle collateral for multiple coins in one call. **USDT and USDC cannot be set.** Each item requires `coin` and `collateralSwitch`. |
| Set Margin Mode | `/v5/account/set-margin-mode` | POST | Yes | `setMarginMode` | — | Sets the UTA margin mode. Default is `REGULAR_MARGIN`. On failure, `reasons` array contains `reasonCode`/`reasonMsg`; empty on success. Portfolio margin may require account equity ≥ 1000 USDC (observed in error response; not a documented hard threshold). |
| Get Borrow History | `/v5/account/borrow-history` | GET | Yes | — | `currency`, `startTime`, `endTime`, `limit`, `cursor` | Interest records for the last 2 years. Default window 30 days. When both `startTime`/`endTime` passed, range ≤ 30 days. `limit` [1,50], default 20. |
| Repay Liability | `/v5/account/quick-repayment` | POST | Yes | — | `coin` | Repays spot margin liabilities. If `coin` not specified, repays all coins. Repayment order follows `liquidationOrder` from VIP margin data. |
| Get Pay Info | `/v5/account/pay-info` | GET | Yes | — | `coin` | Repayment collateral info for a coin before repaying liabilities. `coin` must have outstanding liabilities; if omitted, returns aggregated totals. `borrowInfo.coin` only returned when `coin` is passed. |
| Get Transaction Log | `/v5/account/transaction-log` | GET | Yes | — | `accountType`, `category`, `currency`, `baseCoin`, `type`, `transSubType`, `startTime`, `endTime`, `limit`, `cursor` | Supports up to 2 years of data. Default range 24 hours; max 7 days when both times passed. `limit` [1,50], default 20. **EU-specific:** `extraFees` (withholding tax / WHT) is populated for spot fiat currency orders placed on the EU site; empty string otherwise. May have increased latency during volatility. |
| Get DCP Info | `/v5/account/query-dcp-info` | GET | Yes | — | — | Returns Disconnected-Cancel-All-Prevention (DCP) configuration. Supports Spot only. `timeWindow` [3,300] seconds, default 10. Only configured accounts return data; others get empty. |
| Get SMP Group ID | `/v5/account/smp-group` | GET | Yes | — | — | Returns the self-match-prevention (SMP) group ID. Default `smpGroup` is 0 if no group assigned. |
| Get Trade Behaviour Config | `/v5/account/user-setting-config` | GET | Yes | — | — | Returns limit-price-exceeds-bid/ask behaviour setting (`lpaSpot`), Delta Neutral status (`deltaEnable`), and Spot MNT fee deduction flag (`smsef`). |
| Set Price Limit Behaviour | `/v5/account/set-limit-px-action` | POST | Yes | `category=spot`, `modifyEnable` | — | Configures how the system handles limit orders whose price exceeds the allowed range. `modifyEnable=false` (default): reject; `modifyEnable=true`: auto-adjust to nearest allowed price. No response body. Query current config via `/v5/account/user-setting-config`. |
| Get Account Instruments Info | `/v5/account/instruments-info` | GET | Yes | `category=spot` | `symbol` | User-specific instrument specifications (includes `isPublicRpi` and `myRpiPermission` flags). Custodial sub-accounts do not support queries. `minOrderQty`/`maxOrderQty`/`maxOrderAmt` are deprecated — check `minOrderAmt` and `maxLimitOrderQty`/`maxMarketOrderQty` by order type. May have increased latency during volatility. |
| Get Trade Info For Analysis | `/v5/account/trade-info-for-analysis` | GET | Yes | `symbol` | `startTime`, `endTime` | Aggregated spot trade analysis (execution values, qty, fees, and daily `sumPriceList` breakdown) for a symbol. Spot-only. |
| Get Transferable Amount (Unified) | `/v5/account/withdrawal` | GET | Yes | `coinName` | — | Query the available amount to transfer from the Unified wallet. Supports **up to 20 coins** per request, comma-separated (e.g. `BTC,USDC,USDT,SOL`). `availableWithdrawalMap` keys each coin to its amount; `availableWithdrawal` is for the first coin only. May have increased latency during volatility. |

## Key Rules

- All 18 endpoints in this module require authentication (API key). All are UTA-only.
- **Margin modes** for `set-margin-mode`: `ISOLATED_MARGIN`, `REGULAR_MARGIN` (cross margin), `PORTFOLIO_MARGIN`. Default is `REGULAR_MARGIN`. Portfolio margin may require account equity ≥ 1000 USDC (observed in error response; not a documented hard threshold).
- **Collateral restrictions**: `USDT` and `USDC` **cannot** have their collateral switch set via `set-collateral-switch` or `set-collateral-switch-batch`. When `marginCollateral=false`, the `collateralSwitch` field is meaningless.
- **`/v5/account/withdrawal` is not a withdrawal endpoint** — it queries the transferable (withdrawable) amount for up to 20 comma-separated coins in the Unified wallet. Use the asset module for actual withdrawal operations.
- **EU `extraFees` (WHT)**: In `transaction-log` responses, the `extraFees` field carries withholding tax amounts for spot fiat currency orders on the EU site. It is an empty string for all other order types.
- **Deprecated response fields** to avoid relying on:
  - `wallet-balance`: `accountLTV`, `free`, `availableToWithdraw`, `availableToBorrow`
  - `collateral-info`: `collateralRatio`, `freeBorrowingAmount`
  - `account/info`: `dcpStatus`, `timeWindow`, `smpGroup`
  - `account/instruments-info`: `minOrderQty`, `maxOrderQty`, `maxOrderAmt`, `innovation`
- **Limit price behaviour** (`set-limit-px-action` + `user-setting-config`): The allowed buy range is `Min[Max(Index, Index×(1+x%)+2min avg premium), Index×(1+y%)]`; sell range is symmetric. `x%` and `y%` come from `priceLimitRatioX`/`priceLimitRatioY` in instruments info.
- **DCP (Disconnected-Cancel-All-Prevention)**: Requires prior configuration via an account manager. Supports Spot only. `timeWindow` range is [3, 300] seconds.
- **Borrow history** max query range is 2 years. When only `startTime` is passed, the window is `startTime` to `startTime+30d`; only `endTime`: `endTime-30d` to `endTime`; both: `endTime - startTime ≤ 30 days`.
- **Transaction log** supports up to 2 years. Default window is 24 hours. When both times passed, `endTime - startTime ≤ 7 days`. Paginated via `cursor`/`nextPageCursor`.
- During extreme market volatility, `instruments-info`, `transaction-log`, and `withdrawal` may experience increased latency or data delays.
- **Account instruments info**: `maxLimitOrderQty`, `maxMarketOrderQty`, and `postOnlyMaxLimitOrderSize` are adjusted bi-monthly (3rd and 17th, 08:00 UTC+8).
- **`pay-info`** should be called before `quick-repayment` to check repayment collateral requirements. `coin` must have outstanding liabilities or an error is returned.
- **SMP group** `smpGroup` is 0 by default if the account has no assigned group.
- **Account instruments info** includes `symbolType` (e.g. `xstocks`) and `xstockMultiplier` for tokenised stock pairs: `stock_price = token_price / multiplier`, `stock_qty = token_qty × multiplier`, default multiplier 1.

## Enums

**`accountType`** (`wallet-balance`, `transaction-log`): `UNIFIED`

**`setMarginMode`** (`set-margin-mode`): `ISOLATED_MARGIN`, `REGULAR_MARGIN`, `PORTFOLIO_MARGIN`

**`collateralSwitch`** (`set-collateral-switch`, `set-collateral-switch-batch`): `ON`, `OFF`

**`category`** (`fee-rate`, `set-limit-px-action`, `instruments-info`, `transaction-log`, `trade-info-for-analysis`): `spot`

**`modifyEnable`** (`set-limit-px-action`): `true` (auto-adjust order price), `false` (reject order — default)

**`type`** (`transaction-log`): transaction log type enum (`typeuta-translog` — see inventory for full list)
