# Module: User

> Loaded on-demand by the Bybit EU Trading Skill. Authentication required for all endpoints. All endpoints operate on `api.bybit.eu`.

## Scenario: Sub-account and API-key management

Common user intents this module handles:

- "Create a new sub-account for my trading team" → `POST /v5/user/create-sub-member`
- "List all my sub-accounts" → `GET /v5/user/query-sub-members`
- "Freeze sub-account 123456" → `POST /v5/user/frozen-sub-member`
- "Delete sub-account 123456" → `POST /v5/user/del-submember`
- "What permissions does my current API key have?" → `GET /v5/user/query-api`
- "List all API keys for sub-account 123456" → `GET /v5/user/sub-apikeys`
- "Update the permissions on my master API key" → `POST /v5/user/update-api`
- "Delete the API key for sub-account 123456" → `POST /v5/user/delete-sub-api`
- "What wallet types are available for my account?" → `GET /v5/user/get-member-type`
- "Show my referral list" → `GET /v5/user/invitation/referrals`

> **IMPORTANT — Bybit EU API key creation is BLOCKED via API (error 81007)**
> On Bybit EU, `POST /v5/user/create-sub-api` always returns error code **81007**:
> _"Bybit Europe does not support creating API Key via API."_
> API keys for sub-accounts **must be created through the Bybit EU web interface** at
> [www.bybit.eu](https://www.bybit.eu) → Account → API Management.
> Do **not** attempt programmatic API key creation on Bybit EU — it will always fail.
> The endpoint is documented here for completeness and for non-EU environments only.

---

## API Reference

### Sub-account Management

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Create Sub UID | `/v5/user/create-sub-member` | POST | Yes | `username`, `memberType` | `password`, `switch`, `isUta` (deprecated), `note` | `username` 6–16 alphanumeric chars (numbers + letters required). `memberType`: 1=normal, 6=custodial. Response `status`: 1=normal, 2=login banned, 4=frozen. `isUta` deprecated — always UTA. **CONFIRM-gated.** |
| Get Sub UID List (Limited) | `/v5/user/query-sub-members` | GET | Yes | — | — | Returns at most 1,000 sub UIDs. Use `/v5/user/submembers` for accounts with >10k sub-accounts. Master API key only. |
| Get Sub UID List (Unlimited) | `/v5/user/submembers` | GET | Yes | — | `pageSize`, `nextCursor` | For accounts with >10k sub-accounts. Up to 100 records per request. `nextCursor='0'` means no more pages. Master API key only. |
| Freeze Sub UID | `/v5/user/frozen-sub-member` | POST | Yes | `subuid`, `frozen` | — | `frozen`: 0=unfreeze, 1=freeze. No response parameters. Master API key only. **CONFIRM-gated.** |
| Delete Sub UID | `/v5/user/del-submember` | POST | Yes | `subMemberId` | — | Sub-account balance must be ≤ 0.001 USDT or deletion fails. No response parameters. Master API key only. **CONFIRM-gated.** |
| Get Fund Custodial Sub Acct | `/v5/user/escrow_sub_members` | GET | Yes | — | `pageSize`, `nextCursor` | For institutional clients querying fund custodial sub-accounts. Up to 100 records per request. `nextCursor='0'` = no more pages. Response `memberType=12` = fund custodial. |

### API-key Management

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get API Key Information | `/v5/user/query-api` | GET | Yes | — | — | Returns permissions, IP bindings, expiry, VIP/KYC level, and account type for the calling API key. Works for both master and sub API keys. `readOnly`: 0=read+write, 1=read-only. `kycRegion` returned for personal accounts (EU/MiCA context). |
| Create Sub UID API Key | `/v5/user/create-sub-api` | POST | Yes | `subuid`, `readOnly`, `permissions` | `note`, `ips` | **BLOCKED on Bybit EU — returns error 81007.** Use web UI at www.bybit.eu instead. No-IP-bound key expires after 90 days (or 7 days after password change). The returned secret cannot be retrieved later. Master API key only. |
| Get Sub Account All API Keys | `/v5/user/sub-apikeys` | GET | Yes | `subMemberId` | `limit`, `cursor` | `limit` range [1,20], default 20. Response `status`: 1=permanent, 2=expired, 3=within validity, 4=expiring soon (<7 days). `secret` always returned as `'******'`. Master API key only. |
| Modify Master API Key | `/v5/user/update-api` | POST | Yes | — | `readOnly`, `permissions` | Modifies the calling master API key only. Omit `permissions` param if not changing permissions. Affiliate permission requires removing all other permissions. `FiatBybitPay` is deprecated — use `FiatBitPay`. **CONFIRM-gated.** |
| Modify Sub API Key | `/v5/user/update-sub-api` | POST | Yes | — | `apikey` (conditional), `readOnly`, `ips`, `permissions` | `apikey` required when master manages sub key; must NOT be passed when sub key calls itself. Omit `permissions` if not changing. Fund custodial account not supported for Wallet permission. **CONFIRM-gated.** |
| Delete Master API Key | `/v5/user/delete-api` | POST | Yes | — | — | **DANGER:** The API key used to call this endpoint is **immediately invalidated**. No request or response parameters. Master API key only. **CONFIRM-gated.** |
| Delete Sub API Key | `/v5/user/delete-sub-api` | POST | Yes | — | `apikey` (conditional) | **DANGER:** Sub API key is **immediately invalidated**. `apikey` required when master manages sub key; must NOT be passed when sub key calls itself. **CONFIRM-gated.** |

### Referrals & Wallet Type

| Endpoint | Path | Method | Auth | Required Params | Optional Params | Notes |
|----------|------|--------|------|-----------------|-----------------|-------|
| Get Friend Referrals | `/v5/user/invitation/referrals` | GET | Yes | — | `status`, `size`, `cursor` | Returns referral relationships with pagination. Default page size 20, range [1,100]. `status`: 0=alive, 1=invalid; omit to return all. |
| Get UID Wallet Type | `/v5/user/get-member-type` | GET | Yes | — | `memberIds` | Master API key can query up to 200 sub UIDs (comma-separated). Master UID is always returned at top of array. Sub API key returns its own wallet types only (`memberIds` ignored). EU wallet types limited to FUND and UNIFIED. |

---

## Key Rules

- All 15 endpoints in this module require authentication. All operate on `api.bybit.eu`.
- **CRITICAL — Bybit EU blocks API key creation (error 81007):** `POST /v5/user/create-sub-api` is not functional on Bybit EU. Error code 81007 is returned: _"Bybit Europe does not support creating API Key via API."_ API keys must be created manually via the Bybit EU web interface. **Never instruct a Bybit EU user to call `create-sub-api`.**
- **Sub-account management** (`create-sub-member`, `frozen-sub-member`, `del-submember`) requires the master API key with one of: Account Transfer, Subaccount Transfer, or Withdrawal permission.
- **API-key management** (`create-sub-api`, `update-api`, `delete-api`) requires master API key with Account Transfer, Subaccount Transfer, or Withdrawal permission; these operations are CONFIRM-gated.
- **`sub-apikeys`** (`GET /v5/user/sub-apikeys`) requires a master API key only — any permission level is sufficient (no elevated permission needed).
- **`update-sub-api` and `delete-sub-api`** can be called by either a master API key (needs Account Transfer, Subaccount Transfer, or Withdrawal) OR directly by the sub API key being modified/deleted (the sub key then needs Account Transfer + Sub Member Transfer permission). These operations are CONFIRM-gated.
- **No-IP-bound API keys** expire after 90 days of creation, or after 7 days from an account password change.
- **API key secret** is only returned at creation time and cannot be retrieved later via GET.
- **Custodial sub-accounts** (`memberType=6`) have restricted permissions — Fund custodial account does not support the Wallet permission in API keys.
- **Delete Sub UID** (`del-submember`) fails if the sub-account balance exceeds 0.001 USDT. Transfer funds out first.
- **Delete Master/Sub API Key** immediately invalidates the key used for the call — the session ends the moment the request succeeds. Use with extreme caution.
- **`query-sub-members`** returns at most 1,000 sub UIDs. Accounts with >10,000 sub-accounts should use `submembers` (paginated, unlimited).
- **`get-member-type`** on EU returns wallet types limited to FUND and UNIFIED (EU spot/spot-margin UTA only — no derivatives wallet types).
- **`query-api` response fields:** `parentUid` returns `'0'` when called by the master account. `deadlineDay`/`expiredAt`/`createdAt` are only populated for keys with no IP binding or after a password change.
- **`update-api` restrictions:** For Read/Write master keys, adding or deleting `FiatP2P`, `FiatBybitPay`, `FiatBitPay`, and `FiatConvertBroker` permissions is prohibited. `Affiliate` permission requires all other permissions to be removed first.
- **Deprecated on EU:** `FiatBybitPay` permission is deprecated (use `FiatBitPay`). `NFT` permission is always `[]`. `FiatP2P`, `FiatBybitPay`, `FiatBitPay`, `FiatConvertBroker`, `BlockTrade`, and `Affiliate` are not applicable to sub-accounts (always `[]`).
- **`kycLevel`** values: `LEVEL_DEFAULT`, `LEVEL_1`, `LEVEL_2`. `kycRegion` is returned for personal accounts in EU/MiCA context.
- **Sub-account status codes:** 1=normal, 2=login banned, 4=frozen.
- **Sub-account `accountMode` codes:** `escrow_sub_members` returns only `1` (Classic) and `3` (UTA1.0). `submembers` and `query-sub-members` return the full set: 1=Classic, 3=UTA1.0, 4=UTA1.0 Pro, 5=UTA2.0, 6=UTA2.0 Pro.
- **Custodial sub-account `memberType`:** 12=fund custodial (in `escrow_sub_members` response); 6=custodial subaccount (in `create-sub-member` and sub UID list responses).

---

## Enums

**`memberType`** (`create-sub-member`, sub UID list responses): `1` (normal subaccount), `6` (custodial subaccount)

**`switch`** (`create-sub-member`): `0` (disable quick login — default), `1` (enable quick login)

**`frozen`** (`frozen-sub-member`): `0` (unfreeze), `1` (freeze)

**`readOnly`** (`create-sub-api`, `update-api`, `update-sub-api`): `0` (read and write), `1` (read only)

**`status`** (`invitation/referrals`): `0` (alive), `1` (invalid)

**`accountType`** (`get-member-type` response): `FUND`, `UNIFIED`

**`kycLevel`** (`query-api` response): `LEVEL_DEFAULT`, `LEVEL_1`, `LEVEL_2`

**`permissions` keys — master account** (`update-api`): `Spot` (`SpotTrade`), `Wallet` (`AccountTransfer`, `SubMemberTransfer`), `Exchange` (`ExchangeHistory`), `Earn` (`Earn`), `FiatP2P` (`FiatP2POrder`, `Advertising`), `FiatBybitPay` (deprecated — `FaitPayOrder`), `FiatBitPay` (`FaitPayOrder`), `FiatConvertBroker` (`FiatConvertBrokerOrder`), `Affiliate` (`Affiliate`), `Derivatives` (`DerivativesTrade`), `BlockTrade` (`BlockTrade`)

**`permissions` keys — sub-account** (`create-sub-api`, `update-sub-api`): `Spot` (`SpotTrade`), `Wallet` (`AccountTransfer`, `SubMemberTransferList`), `Derivatives` (`DerivativesTrade`), `Exchange` (`ExchangeHistory`), `Earn` (`Earn`)

**Sub-account status** (list/escrow responses): `1` (normal), `2` (login banned / forbidden login), `4` (frozen)

**`accountMode`** (`submembers`, `query-sub-members` responses): `1` (Classic), `3` (UTA1.0), `4` (UTA1.0 Pro), `5` (UTA2.0), `6` (UTA2.0 Pro). Note: `escrow_sub_members` returns only `1` (Classic) and `3` (UTA1.0).
