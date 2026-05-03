# Nest REST API Cookbook

Base URL: `https://api.nest.credit/v1`. All endpoints return `{ data: ... }` wrappers; the
plugin's `NestApi` client unwraps automatically. When curling directly, drill into `.data`.

**Slug → symbol mapping** (slugs are kebab-case; symbols start with `n` uppercase):

| Slug | Symbol | Notes |
|---|---|---|
| `nest-treasury-vault` | nTBILL | Tokenized US Treasuries |
| `nest-alpha-vault` | nALPHA | Diversified multi-strategy |
| `nest-wisdom-vault` | nWISDOM | WisdomTree regulated fund |
| `nest-opal-vault` | nOPAL | Private credit |
| `nest-basis-vault` | nBASIS | Basis-trade yield |
| `nest-credit-vault` | nCREDIT | Private credit |
| `nest-acrdx-vault` | nACRDX | Accredited-DX credit |
| `nest-elixir-vault` | nELIXIR | Elixir strategy (verify) |
| `nest-insto-vault` | nINSTO | Institutional private credit |
| `nest-scope-vault` | nSCOPE | Scope strategy (verify) |

---

## Endpoints

### `GET /vaults`

Lists all vaults in the live registry.

**Params**

| Name | Required | Default | Description |
|---|---|---|---|
| `live` | no | `false` | `true` = only return currently live vaults |

**Key response fields** (`data[]` items)

| Field | Type | Description |
|---|---|---|
| `slug` | string | Kebab-case identifier used in all other endpoints |
| `name` | string | Human-readable vault name |
| `symbol` | string | Token symbol (e.g. `nTBILL`) |
| `vaultType` | string | `boring` / `nest` / `boringNest` — controls deposit/withdraw flow |
| `vaultAddress` | string | On-chain vault token address |
| `tellerContractAddress` | string | Teller entry point for boring vaults |
| `accountantAddress` | string | Rate provider / accountant contract |
| `decimals` | number | Share token decimals |
| `liquidAssets` | array | Accepted deposit tokens, each with `address`, `chainId`, `symbol` |
| `nestVaults` | array | Inner nest vault addresses (for `boringNest` types) |
| `tvl` | string | Current TVL in USD |
| `sec30d` | string | 30-day SEC-compliant yield figure |
| `numHolders` | number | Count of distinct token holders |

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults?live=true" | jq '.data[] | {slug, symbol, vaultType, tvl, sec30d}'
```

**Consumed by:** `onchainos-nest vaults` (no `--slug`)

---

### `GET /vaults/{slug}`

Returns a single vault object — same shape as one element from `GET /vaults`.

**Params:** none (slug is path param)

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault" | jq '.data | {slug, symbol, vaultType, vaultAddress, decimals, liquidAssets}'
```

**Consumed by:** `onchainos-nest vaults --slug nest-treasury-vault`

---

### `GET /vaults/{slug}/details`

Adds performance and liquidity metrics to the base vault shape.

**Extra fields beyond base vault**

| Field | Type | Description |
|---|---|---|
| `apy.rolling7d` | string | 7-day rolling APY (percent) |
| `apy.rolling30d` | string | 30-day rolling APY (percent) |
| `apy.sec30d` | string | 30-day SEC-compliant APY |
| `tvl30DayChange` | string | TVL percentage change over 30 days |
| `tokenPrice` | string | Current share price in USD |
| `liquidFunds` | string | Immediately liquid USD in the vault |
| `pendingRedemptions` | string | USD value of queued redemption requests |

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault/details" \
  | jq '.data | {apy, tvl30DayChange, tokenPrice, liquidFunds, pendingRedemptions}'
```

**Consumed by:** `onchainos-nest history --vault nest-treasury-vault`

---

### `GET /vaults/{slug}/positions`

Returns the vault's asset breakdown: liquid assets, yield-bearing positions, and supply by chain.

**Key response fields** (`data` object)

| Field | Type | Description |
|---|---|---|
| `vaultAddress` | string | Vault token contract address |
| `positions.liquidAssets` | array | Liquid-asset positions with `address`, `balance`, `usdValue` |
| `positions.yieldAssets` | array | Yield positions (protocol, underlying, balance, usdValue) |
| `supplyByChain` | object | `{ chainId: totalSupply }` map |

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault/positions" \
  | jq '.data | {vaultAddress, supplyByChain, "liquidCount": (.positions.liquidAssets | length)}'
```

**Consumed by:** `onchainos-nest status --vault nest-treasury-vault` (per-vault breakdown, optional)

---

### `GET /vaults/{slug}/last-price-update`

Returns the most recent share-price data points, one entry per chain the vault is deployed on.

**Key response fields** (`data[]` items)

| Field | Type | Description |
|---|---|---|
| `chainId` | number | Chain where this price update occurred |
| `price` | string | Share price in USD at update time |
| `updatedAt` | string | ISO-8601 timestamp of the price update |
| `blockNumber` | number | Block at which the price was committed on-chain |

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault/last-price-update" \
  | jq '.data[] | {chainId, price, updatedAt, blockNumber}'
```

**Consumed by:** `onchainos-nest history --vault nest-treasury-vault`

---

### `GET /vaults/{slug}/recent-transactions`

Returns the last 7 days of on-chain activity for the vault (deposits, withdrawals, solver fills).

**Key response fields** (`data[]` items)

| Field | Type | Description |
|---|---|---|
| `txHash` | string | Transaction hash |
| `type` | string | `deposit` / `withdraw` / `redeem` / `requestRedeem` |
| `user` | string | User address |
| `assetAmount` | string | Input/output asset amount (UI units) |
| `shareAmount` | string | Vault share amount minted or burned |
| `timestamp` | string | ISO-8601 timestamp |
| `chainId` | number | Chain where the tx occurred |

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault/recent-transactions" \
  | jq '.data[] | {type, user, assetAmount, shareAmount, timestamp}'
```

**Consumed by:** `onchainos-nest history --vault nest-treasury-vault`

---

### `GET /vaults/{slug}/token-minimums`

Returns minimum deposit amounts per accepted asset, used for pre-flight validation before building deposit calldata.

**Key response fields** (`data[]` items)

| Field | Type | Description |
|---|---|---|
| `tokenAddress` | string | ERC-20 asset address |
| `symbol` | string | Asset symbol (e.g. `USDC`) |
| `chainId` | number | Chain this minimum applies to |
| `minimumDeposit` | string | Minimum deposit in UI units |
| `minimumDepositUsd` | string | USD equivalent of the minimum |

**Curl**

```bash
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault/token-minimums" \
  | jq '.data[] | {symbol, chainId, minimumDeposit, minimumDepositUsd}'
```

**Consumed by:** `onchainos-nest build-deposit` (internal validation — rejects `--amount` below minimum before emitting calldata)

---

### `GET /vaults/{slug}/user/{user}/pending-redemptions`

Returns all pending and claimable redemption requests for a specific user in a specific vault.

**Params:** `slug` and `user` are both path params; no query params.

**Key response fields** (`data[]` items)

| Field | Type | Description |
|---|---|---|
| `requestId` | string | Unique redemption request ID |
| `shareAmount` | string | Shares submitted for redemption (UI units) |
| `wantToken` | string | Asset address the user requested to receive |
| `status` | string | `pending` / `claimable` / `expired` / `fulfilled` |
| `currentClaimableAssets` | string | USD value claimable right now (0 until fulfilled) |
| `deadline` | string | ISO-8601 expiry (AtomicQueue requests only) |
| `earliestClaimTime` | string | Earliest claim time (nest cooldown vaults) |

**Curl**

```bash
USER=0xYourAddressHere
curl -s "https://api.nest.credit/v1/vaults/nest-treasury-vault/user/${USER}/pending-redemptions" \
  | jq '.data[] | {requestId, status, shareAmount, currentClaimableAssets, deadline}'
```

**Consumed by:** `onchainos-nest pending-redemptions --address <user> --vault nest-treasury-vault`

---

### `GET /user/{address}/compliance`

Compliance gate check. Returns a `predicateMessage` that must be passed to the deposit contract. Non-compliant addresses return `200 OK` with `isCompliant: false` and an empty signatures array — not an HTTP error.

**Params**

| Name | Required | Default | Description |
|---|---|---|---|
| `chainId` | yes | — | Chain where the deposit will occur (e.g. `1` for Ethereum) |
| `isNewProxy` | no | `false` | `true` for nest/boringNest vaults (routes through `NEW_PREDICATE_PROXY`) |

**Key response fields** (`data` object)

| Field | Type | Description |
|---|---|---|
| `isCompliant` | boolean | `true` = user may proceed to deposit |
| `message` | string | Human-readable reason when `isCompliant: false` |
| `predicateMessage` | object | Signed compliance payload to pass to the deposit contract |
| `predicateMessage.expireByBlockNumber` | number | Expiry — **interpreted as Unix timestamp** (see Notes) |
| `predicateMessage.signatures` | array | Compliance signatures validated on-chain by PredicateProxy |

**Curl**

```bash
USER=0xYourAddressHere
curl -s "https://api.nest.credit/v1/user/${USER}/compliance?chainId=1&isNewProxy=false" \
  | jq '.data | {isCompliant, message, "expiry": .predicateMessage.expireByBlockNumber}'
```

**Consumed by:** `onchainos-nest eligibility --address <user> --chain-id 1 [--is-new-proxy]`

---

### `GET /user/{address}/details`

Returns an aggregate summary of the user's holdings and performance across all Nest vaults.

**Params:** `address` is a path param; no query params.

**Key response fields** (`data` object)

| Field | Type | Description |
|---|---|---|
| `totalValueUsd` | string | Total USD value across all vaults |
| `weightedApy` | string | APY weighted by position size |
| `positions` | array | Per-vault breakdown with `slug`, `shareBalance`, `valueUsd`, `apy` |
| `firstDepositAt` | string | ISO-8601 timestamp of user's first deposit |

**Curl**

```bash
USER=0xYourAddressHere
curl -s "https://api.nest.credit/v1/user/${USER}/details" \
  | jq '.data | {totalValueUsd, weightedApy, "vaultCount": (.positions | length)}'
```

**Consumed by:** `onchainos-nest status --address <user>` (aggregate view, no `--vault` flag)

---

## Notes

### `expireByBlockNumber` is a Unix timestamp

The field name is a legacy misnomer. The `PredicateProxy` contract interprets it as a **Unix
timestamp in seconds**, not a block number. A value of `99999999` represents approximately
1973-11-29 — any deposit attempt with that value will revert with "transaction expired". Always
use a value that is a few minutes in the future relative to the current Unix time. The plugin
generates this correctly via `eligibility`; this note is for anyone calling the compliance
endpoint directly.

### `{ data: ... }` envelope

Every endpoint wraps its payload in `{ "data": ... }`. The `NestApi` client inside the plugin
unwraps this transparently — plugin output already contains only the inner value. When scripting
directly against the REST API, always add `| jq '.data'` (or `.data[]` for arrays).

### Compliance is not an error envelope

When an address fails the compliance check, the API returns `HTTP 200` with
`{ data: { isCompliant: false, message: "...", signatures: [] } }`. There is no HTTP 4xx/5xx.
Treat `isCompliant: false` as a hard stop — surface `message` verbatim to the user.

### Per-vault contract addresses are live

`tellerContractAddress`, `accountantAddress`, `vaultAddress`, and `nestVaults[]` are fetched from
the API at runtime. Do not hardcode them. Only the universal system contracts
(`OLD_PREDICATE_PROXY`, `NEW_PREDICATE_PROXY`, `ATOMIC_QUEUE`, `ATOMIC_SOLVER`, `MULTICALL3`)
are vendored — see `references/system-contracts.md`.
