# Module: Asset

> Loaded on-demand by the Bybit EU Trading Skill. Authentication required for all endpoints. All endpoints operate on `api.bybit.eu`. Mainnet write operations (transfers, withdraw create/cancel, deposit-to-account, Travel Rule submit) are CONFIRM-gated per SKILL.md.

## Scenario: Balances, transfers, deposits, withdrawals, and asset utilities

Common user intents this module handles:

- "What is my USDT balance in the UNIFIED account?" → `GET /v5/asset/transfer/query-account-coin-balance`
- "Show all coin balances in my FUND wallet" → `GET /v5/asset/transfer/query-account-coins-balance`
- "How much BTC can I withdraw?" → `GET /v5/asset/withdraw/withdrawable-amount`
- "Move 100 USDT from FUND to UNIFIED" → `POST /v5/asset/transfer/inter-transfer`
- "Transfer USDT from my sub-account to the main account" → `POST /v5/asset/transfer/universal-transfer`
- "Show my deposit history for the last 7 days" → `GET /v5/asset/deposit/query-record`
- "What is the deposit address for BTC on the TRC20 chain?" → `GET /v5/asset/deposit/query-address`
- "Submit Travel Rule originator info for deposit ID 12345" → `POST /v5/asset/travel-rule/deposit/submit`
- "Withdraw 500 USDT to my whitelisted address" → `POST /v5/asset/withdraw/create`
- "Show coin info for ETH including chain details" → `GET /v5/asset/coin/query-info`
- "Show my recent coin conversion history" → `GET /v5/asset/exchange/order-record`
- "Show my funding account transaction history" → `GET /v5/asset/fundinghistory`
- "List my sub-accounts available for universal transfer" → `GET /v5/asset/transfer/query-sub-member-list`
- "What are the total assets across all my accounts in BTC?" → `GET /v5/asset/total-members-assets`

## API Reference

### Balances

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Single Coin Balance | `/v5/asset/transfer/query-account-coin-balance` | GET | Yes | `accountType`, `coin` | `memberId`, `toMemberId`, `toAccountType`, `withBonus`, `withTransferSafeAmount`, `withLtvTransferSafeAmount` | `memberId` required when querying sub-account with master key. `toMemberId` required for cross-UID transferable balance. `toAccountType` required for cross-account-type transferable balance. `transferSafeAmount`/`ltvTransferSafeAmount` returned as empty string if not queried. May have increased latency during volatility. |
| Get All Coins Balance | `/v5/asset/transfer/query-account-coins-balance` | GET | Yes | `accountType` | `memberId`, `coin`, `withBonus` | `coin` is mandatory when `accountType=UNIFIED`; supports up to 10 coins per request (comma-separated). `memberId` required when using master key to check sub-account balance. May have increased latency during volatility. |
| Get Withdrawable Amount | `/v5/asset/withdraw/withdrawable-amount` | GET | Yes | `coin` | — | Returns withdrawable amounts across SPOT, FUND, UTA, and EARN wallets. SPOT wallet not returned if removed; EARN not returned if coin does not support withdrawal via Earn. `limitAmountUsd` is the USD-equivalent amount frozen by risk controls. May have increased latency during volatility. |

### Transfers

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Create Internal Transfer | `/v5/asset/transfer/inter-transfer` | POST | Yes | `transferId`, `coin`, `amount`, `fromAccountType`, `toAccountType` | — | Transfers between different account types under the **same UID**. Generate `transferId` as a UUID. CONFIRM-gated on mainnet. Response status: `STATUS_UNKNOWN`, `SUCCESS`, `PENDING`, `FAILED`. |
| Get Internal Transfer Records | `/v5/asset/transfer/query-inter-transfer-list` | GET | Yes | — | `transferId`, `coin`, `status`, `startTime`, `endTime`, `limit`, `cursor` | Returns past 7 days by default. If only `startTime`: `startTime` to `startTime+7d`; only `endTime`: `endTime-7d` to `endTime`; both: max range 7 days. Timestamps in ms but queried at second-level granularity. `limit` [1,50], default 20. |
| Create Universal Transfer | `/v5/asset/transfer/universal-transfer` | POST | Yes | `transferId`, `coin`, `amount`, `fromMemberId`, `toMemberId`, `fromAccountType`, `toAccountType` | — | Transfers between **different UIDs** (main-sub or sub-sub). Cannot transfer between the same UID. Sub-account key must have `SubMemberTransferList` permission and can only transfer to the main account. Error 131228 means insufficient balance — check transfer-safe amount via Get Single Coin Balance. CONFIRM-gated on mainnet. |
| Get Universal Transfer Records | `/v5/asset/transfer/query-universal-transfer-list` | GET | Yes | — | `transferId`, `coin`, `status`, `startTime`, `endTime`, `limit`, `cursor` | Main key needs `SubMemberTransfer` permission; sub key needs `SubMemberTransferList`. Default 7-day window; max 7 days when both times specified. Timestamps in ms, queried at second-level. `limit` [1,50], default 20. |

### Deposit

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Deposit Records (on-chain) | `/v5/asset/deposit/query-record` | GET | Yes | — | `id`, `txID`, `coin`, `startTime`, `endTime`, `limit`, `cursor` | `endTime − startTime` must be < 30 days; defaults to last 30 days. Supports master or sub UID key. `id` takes highest priority when combined with other params. txID cannot query data before Jan 1, 2024. `limit` [1,50], default 50. **EU-specific:** response includes `taxDepositRecordsId` and `taxStatus` (0 = none, 1 = pending, 2 = completed) for Austria tax reporting. |
| Get Internal Deposit Records (off-chain) | `/v5/asset/deposit/query-internal-record` | GET | Yes | — | `txID`, `startTime`, `endTime`, `coin`, `cursor`, `limit` | Off-chain (within-platform) deposits. Max range 30 days. Supports master or sub UID key. **EU-specific:** response includes `taxDepositRecordsId` and `taxStatus` fields. |
| Get Master Deposit Address | `/v5/asset/deposit/query-address` | GET | Yes | `coin` | `chainType` | Query deposit address for the master account. Use `chain` value from coin-info for `chainType`. `contractAddress` shows last 6 characters; `batchReleaseLimit` of `"-1"` means no limit. |
| Set Deposit Account | `/v5/asset/deposit/deposit-to-account` | POST | Yes | `accountType` | — | Sets the wallet type for auto-transfer after deposit (same as web GUI deposit setting). Master UID only. Funds go to FUND wallet by default. CONFIRM-gated on mainnet. |
| Get Sub Deposit Address | `/v5/asset/deposit/query-sub-member-address` | GET | Yes | `coin`, `chainType`, `subMemberId` | — | Master key only. Custodial sub-account deposit addresses cannot be obtained. `contractAddress` shows last 6 chars; `batchReleaseLimit` `"-1"` = no limit. |
| Get Sub Deposit Records (on-chain) | `/v5/asset/deposit/query-sub-member-record` | GET | Yes | `subMemberId` | `id`, `txID`, `coin`, `startTime`, `endTime`, `limit`, `cursor` | Master key only. `endTime − startTime` < 30 days; defaults to last 30 days. `id` takes priority. txID cannot query before Jan 1, 2024. **EU-specific:** `taxDepositRecordsId` and `taxStatus` in response. |
| Submit Deposit Originator Info (Travel Rule) | `/v5/asset/travel-rule/deposit/submit` | POST | Yes | `depositId`, `questionnaire` | `subAccountId` | **EU MiCA/TFR-specific.** Call when deposit `travel_rule_status=1` (pending info submission). Body uses snake_case (`deposit_id`, `sub_account_id`). `questionnaire` is a JSON string (max 16384 bytes) — see Questionnaire schema below. CONFIRM-gated on mainnet. Response `travelRuleStatus`: 0 Approved, 1 CollectInfo (re-submit), 2 Pending (poll deposit query), 3 Rejected. |

### Withdraw

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Withdraw | `/v5/asset/withdraw/create` | POST | Yes | `coin`, `address`, `amount`, `timestamp`, `accountType`, `questionnaire` | `chain` *(conditional — see Notes)*, `tag`, `forceChain`, `feeType`, `requestId` | **Security note: withdraw permission is intentionally discouraged for AI-generated API keys.** **`chain` — conditional required:** must be provided for on-chain withdrawals (`forceChain=0` auto on-chain, or `forceChain=1` force on-chain); may be omitted only for internal UID transfers (`forceChain=2`). `tag` required only if the address requires a memo/tag. Whitelist address in address book first. Rate limit: 5 req/s, max once per 10 s per chain/coin pair. Master key only. **`questionnaire` — always required on Bybit EU** (MiCA/TFR Travel Rule applies to every withdrawal). CONFIRM-gated on mainnet. |
| Cancel Withdrawal | `/v5/asset/withdraw/cancel` | POST | Yes | `id` | — | Cancel a pending withdrawal. Response `status`: 0 = fail, 1 = success. CONFIRM-gated on mainnet. |
| Get Withdrawal Address List | `/v5/asset/withdraw/query-address` | GET | Yes | — | `coin`, `chain`, `addressType`, `limit`, `cursor` | API key must have withdrawal permissions. `addressType`: 0 = OnChain; 1 = Internal Transfer (`coin`/`chain` params invalid); 2 = on-chain + internal (`coin`/`chain` invalid). Response `status`: 0 = normal, 1 = new address (24 h withdrawal restriction). `verified`: 0 = unverified, 1 = verified. `limit` [1,50], default 50. |
| Get Withdrawal Records | `/v5/asset/withdraw/query-record` | GET | Yes | — | `withdrawID`, `txID`, `coin`, `withdrawType`, `startTime`, `endTime`, `limit`, `cursor` | Master key only. `endTime − startTime` < 30 days; defaults to last 30 days. `limit` [1,50], default 50. **EU-specific:** response includes `tax`, `taxRate`, `taxType` fields (from Tax Centre configuration). |
| Get Available VASPs | `/v5/asset/withdraw/vasp/list` | GET | Yes | — | — | **EU MiCA/TFR-specific.** Lists counterparty VASP (Virtual Asset Service Provider) entities for the user's compliance zone. Use `vaspEntityId='others'` when transferring to an exchange not in the list. No request parameters. |

### Miscellaneous

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Coin Info | `/v5/asset/coin/query-info` | GET | Yes | — | `coin` | Returns chain details, withdraw/deposit status, fees, and precision. `chainDeposit`/`chainWithdraw`: 0 = suspended, 1 = normal. Empty `withdrawFee` means withdrawal not supported. `withdrawPercentageFee` is a real figure (e.g. `0.022` = 2.2%). `confirmation` credits funds for trading; `safeConfirmNumber` fully unlocks USD-equivalent funds for withdrawal. Empty `contractAddress` means no contract. |
| Get Coin Exchange Records | `/v5/asset/exchange/order-record` | GET | Yes | — | `fromCoin`, `toCoin`, `limit`, `cursor` | Returns **convert history** — records of past coin conversions. The full convert workflow (quote, execute, etc.) lives in the `convert` module. `limit` [1,50], default 10. `createdTime` is returned in seconds. Data may have up to ~5 s delay. |
| Funding Account Transaction History | `/v5/asset/fundinghistory` | GET | Yes | — | `createTimeFrom`, `createTimeTo`, `limit`, `cursor` | **EU-specific endpoint.** Returns Funding Account transaction log. `createTimeFrom` and `createTimeTo` must be used together; interval cannot exceed 7 days; defaults to last 7 days. `limit` [1,100], default 10. `ioDirection`: I = In, O = Out. Response includes localized and English business type/description fields. |
| Get Sub UID List | `/v5/asset/transfer/query-sub-member-list` | GET | Yes | — | — | Master key only. Returns up to 2000 sub-account UIDs. `transferableSubMemberIds` lists sub UIDs with universal transfer enabled. For more than 2000 sub-accounts use `/v5/user/page-subuid`. No request parameters. |
| Get Total Members Assets | `/v5/asset/total-members-assets` | GET | Yes | — | `coin` | Aggregates total assets across all master and sub-accounts, denominated in `coin` (default BTC). `stat`: 0 = normal, non-zero = exception in downstream data. `isM` returned for master account only; `type` returned for sub-accounts only. Sub-account `type` values: 1 Normal, 2 MT4, 3 Trading Bot, 4 Copy trading leader, 5 Copy trading follower, 6 Custodial, 7 Fund custodial, 8 Demo, 9 Copy Pro, 10 MT5 follow, 11 Earn vendor, 12 Earn fund. |

> **NOTE — Travel Rule Questionnaire schema** (no `/v5/` path — reference only): The `questionnaire` field accepted by `/v5/asset/withdraw/create` and `/v5/asset/travel-rule/deposit/submit` is a JSON string (max 16384 bytes) with the following structure for the EU (MiCA/TFR) compliance zone:
>
> | Field | Required | Type | Notes |
> |-------|----------|------|-------|
> | `walletType` | Yes | integer | 0 = custodial/VASP wallet; 1 = non-custodial (personal) wallet |
> | `legalType` | Yes | string | `individual` = natural person; `company` = legal entity/institution |
> | `vaspCode` | Conditional | string (max 64) | Required when `walletType=0`. Use `'others'` if the counterparty VASP is not in the VASP list. Retrieve codes via `/v5/asset/withdraw/vasp/list`. |
> | `isSelfWallet` | No | bool | Whether the wallet belongs to the user. Applicable for non-custodial wallets only. |
> | `firstName` | Conditional | string (max 128) | Required when `legalType=individual` |
> | `lastName` | Conditional | string (max 128) | Required when `legalType=individual` |
> | `companyName` | Conditional | string (max 256) | Required when `legalType=company` |
> | `transactionPurpose` | Yes | string (20–500) | EU MiCA/TFR: transaction purpose description. **Minimum 20 characters.** |
>
> Rules: field names are lowerCamelCase; country codes ISO 3166-1 alpha-3; dates ISO 8601 (YYYY-MM-DD). The `questionnaire` field takes precedence over any legacy `beneficiary`/`transactionPurpose` top-level fields.

## Key Rules

- All 24 `/v5/` endpoints require authentication.
- **`chain` is conditionally required on `/v5/asset/withdraw/create`:** it MUST be provided when `forceChain=0` (normal on-chain, default) or `forceChain=1` (force on-chain). It may be omitted only when `forceChain=2` (internal transfer by UID/email/mobile). Never treat `chain` as a plain optional param for on-chain withdrawals.
- **`questionnaire` is always required on Bybit EU for every withdrawal** (`/v5/asset/withdraw/create`). The EU MiCA/TFR Transfer of Funds Regulation mandates Travel Rule data for all withdrawals — there is no threshold below which it can be omitted on this platform.
- **Travel Rule / VASP** (`/v5/asset/travel-rule/deposit/submit`, `/v5/asset/withdraw/vasp/list`, and the `questionnaire` schema) are **EU/MiCA-specific** requirements under the Transfer of Funds Regulation (TFR). Always retrieve available VASPs from `/v5/asset/withdraw/vasp/list` before building a questionnaire; use `vaspEntityId='others'` when the counterparty is not listed.
- **Withdraw permission is intentionally discouraged for AI-generated API keys** for security. Prefer manual withdrawal via the Bybit EU web interface. If withdrawal via API is required, the key must have withdrawal permissions and all addresses must be pre-whitelisted.
- **`/v5/asset/exchange/order-record` is convert history only** — it returns past conversion records. The full convert workflow (quote request, confirmation, execution, etc.) lives in the separate `convert` module.
- **inter-transfer** operates between account types under the **same UID**; **universal-transfer** operates between **different UIDs** (main-sub or sub-sub).
- **Mainnet write operations** — inter-transfer, universal-transfer, deposit-to-account, withdraw/create, withdraw/cancel, and travel-rule/deposit/submit — are CONFIRM-gated. The skill will ask for explicit confirmation before executing these on mainnet.
- **UUID generation**: `transferId` for inter-transfer and universal-transfer must be a globally unique UUID generated by the caller.
- **EU deposit tax fields**: On-chain and off-chain deposit records include `taxDepositRecordsId` and `taxStatus` (0 = none, 1 = pending, 2 = completed) for Austria tax reporting compliance.
- **EU withdrawal tax fields**: Withdrawal records include `tax`, `taxRate`, and `taxType` fields populated from the Tax Centre configuration.
- **Funding history** (`/v5/asset/fundinghistory`) is an EU-specific endpoint for the Funding Account transaction log. `createTimeFrom`/`createTimeTo` must be paired and range ≤ 7 days.
- **Sub UID list** returns up to 2000 sub-accounts; use `/v5/user/page-subuid` for larger sets. `transferableSubMemberIds` indicates which sub UIDs support universal transfer.
- **Balance latency**: query-account-coin-balance, query-account-coins-balance, and withdrawable-amount may experience increased latency or data delays during extreme market volatility.
- **Coin info** (`withdrawFee` empty → coin does not support withdrawal; `withdrawPercentageFee` is a real ratio not a percentage string).
- **Deposit address** `batchReleaseLimit="-1"` means no batch release limit; `contractAddress` shows last 6 characters only.
- **Withdraw fee types**: `feeType=0` (default) — amount entered is what recipient receives, fee deducted on top; `feeType=1` — amount entered is the gross amount, fee is deducted from it.

## Enums

**`accountType`** (balance/transfer/deposit endpoints): `UNIFIED`, `FUND`, `CONTRACT`, `EARN`

**`withBonus`** (`query-account-coin-balance`, `query-account-coins-balance`): `0` = exclude bonus (default), `1` = include bonus

**`withTransferSafeAmount`** (`query-account-coin-balance`): `0` = false (default), `1` = true

**`withLtvTransferSafeAmount`** (`query-account-coin-balance`): `0` = false (default), `1` = true

**`transfer status`** (`inter-transfer`, `universal-transfer` response; filter in list endpoints): `STATUS_UNKNOWN`, `SUCCESS`, `PENDING`, `FAILED`

**`deposit-to-account accountType`**: `UNIFIED`, `FUND`, `EARN`

**`travelRuleStatus`** (travel-rule/deposit/submit response): `0` = Approved, `1` = CollectInfo (re-submit), `2` = Pending (poll), `3` = Rejected

**`forceChain`** (`withdraw/create`): `0` = auto (internal if Bybit address, else on-chain), `1` = force on-chain, `2` = withdraw by UID

**`feeType`** (`withdraw/create`): `0` = received amount (default), `1` = gross amount (fee deducted)

**`withdrawType`** (`withdraw/query-record`): `0` = on-chain (default), `1` = off-chain, `2` = all

**`addressType`** (`withdraw/query-address`): `0` = OnChain, `1` = Internal Transfer, `2` = on-chain + internal

**`walletType`** (questionnaire): `0` = custodial/VASP, `1` = non-custodial (personal)

**`legalType`** (questionnaire): `individual`, `company`

**`ioDirection`** (fundinghistory): `I` = In, `O` = Out
