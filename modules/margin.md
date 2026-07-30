# Module: Spot Margin Trade

> Loaded on-demand by the Bybit EU Trading Skill. Most endpoints require authentication; `/v5/spot-margin-trade/collateral` and `/v5/spot-margin-trade/data` are public. All endpoints are `category=spot` on `api.bybit.eu`.

## Scenario: Spot margin trade configuration and queries

Common user intents this module handles:

- "Turn on spot margin trading for my account" → `POST /v5/spot-margin-trade/switch-mode`
- "Set my spot margin leverage to 5x" → `POST /v5/spot-margin-trade/set-leverage`
- "Is spot margin enabled on my account? What leverage am I using?" → `GET /v5/spot-margin-trade/state`
- "How much USDT can I borrow for spot margin?" → `GET /v5/spot-margin-trade/max-borrowable`
- "Show me the VIP margin data and tiered collateral ratios" → `GET /v5/spot-margin-trade/data`, `GET /v5/spot-margin-trade/collateral`

## API Reference

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Toggle Margin Trade | `/v5/spot-margin-trade/switch-mode` | POST | Yes | `spotMarginMode` | — | `1`=on, `0`=off. Account must have activated spot margin (quiz) first. Response echoes `spotMarginMode`. |
| Set Leverage | `/v5/spot-margin-trade/set-leverage` | POST | Yes | `leverage` | `currency` | Leverage range [2, 10]. Must not exceed the maximum leverage of the currency. Account must have activated spot margin first. No response body parameters. |
| Get Status And Leverage | `/v5/spot-margin-trade/state` | GET | Yes | — | — | No request params. Returns `spotMarginMode` (1=on, 0=off), configured `spotLeverage` (empty string when mode is off), and `effectiveLeverage` (actual ratio, 2dp truncated). |
| Get Coin State | `/v5/spot-margin-trade/coinstate` | GET | Yes | — | `currency` | Returns per-coin spot margin leverage. `spotLeverage` returns `""` if spot margin mode is off. |
| Get Currency Data | `/v5/spot-margin-trade/currency-data` | GET | Yes | — | `currency` | Returns borrow configuration per coin: flexible/fixed manual borrow limits, coin precision, and fixed interest rate bounds. Fields return `""` when the coin's borrowable switch is disabled. |
| Get Max Borrowable Amount | `/v5/spot-margin-trade/max-borrowable` | GET | Yes | `currency` | — | Returns `currency` and `maxLoan` (max borrowable amount) for the specified coin. |
| Get Available Amount to Repay | `/v5/spot-margin-trade/repayment-available-amount` | GET | Yes | `currency` | — | Returns `lossLessRepaymentAmount` = min(spot coin available balance, coin borrow amount). |
| Get Historical Interest Rate | `/v5/spot-margin-trade/interest-rate-history` | GET | Yes | `currency` | `vipLevel`, `startTime`, `endTime` | Hourly borrowing interest rate history. Supports up to 6 months. Pass `vipLevel` as `No%20VIP` for no-VIP level. `startTime`/`endTime` must both be passed or both omitted; neither=7 days, both=max 30-day interval. |
| Get Tiered Collateral Ratio | `/v5/spot-margin-trade/collateral` | GET | No | — | `currency` | **Public endpoint** — no auth required. Returns UTA loan tiered collateral ratios per coin across quantity tiers. `maxQty` of `""` means positive infinity. |
| Get VIP Margin Data | `/v5/spot-margin-trade/data` | GET | No | — | `vipLevel`, `currency` | **Public endpoint** — no auth required. Returns VIP margin data per coin: borrowability, hourly borrow rate, liquidation order, collateral eligibility, and max borrow amount. The `collateralRatio` response field is deprecated; use Get Tiered Collateral Ratio instead. |

## Key Rules

- **Activate spot margin first**: before using `switch-mode` or `set-leverage`, the account must complete the spot margin activation quiz on the Bybit EU web interface or app. API calls will fail until activation is complete.
- **Leverage range**: `set-leverage` accepts values in the range [2, 10] only. The updated leverage must also not exceed the maximum leverage allowed for the requested currency.
- **`switch-mode` values**: `spotMarginMode=1` turns margin trade on; `spotMarginMode=0` turns it off.
- **`state` endpoint**: returns `spotMarginMode` (1=on, 0=off), `spotLeverage` (empty string `""` when mode is off), and `effectiveLeverage` (actual current leverage ratio, precision 2 decimal places truncated downward).
- **Spot orders with margin**: to use spot margin on a spot order, set `isLeverage=1` in the spot order request. This uses the leverage and collateral configuration managed by this module.
- **UTA-only**: all spot margin trade endpoints apply to the Unified Trading Account only.
- **Borrow and repayment**: `max-borrowable` gives the ceiling for borrowing a coin; `repayment-available-amount` gives the lossless repayable amount (minimum of available coin balance and outstanding borrow). Use `account/quick-repayment` to repay liabilities.
- **Interest rate history**: data is per VIP/Pro level; different users get the same historical rate for the same level (public data, though authentication is required to call the endpoint). Supports up to 6 months of history.
- **Public endpoints**: `GET /v5/spot-margin-trade/collateral` (Tiered Collateral Ratio) and `GET /v5/spot-margin-trade/data` (VIP Margin Data) do not require authentication. The deprecated `collateralRatio` field in `data` should not be used; query `collateral` for tiered ratios instead.
- **`coinstate` with margin off**: `spotLeverage` returns an empty string when the account's spot margin mode is turned off.
- **`currency-data` disabled coins**: if a coin's borrowable switch is disabled, all configuration fields for flexible and fixed manual borrow return `""`.

## Enums

**`spotMarginMode`** (Toggle Margin Trade): `1` (on), `0` (off)

**`vipLevel`** (Get Historical Interest Rate, Get VIP Margin Data): any valid VIP/Pro level string, e.g. `No VIP` (URL-encoded as `No%20VIP`), `VIP1`, `VIP2`, `Pro1`, etc.
