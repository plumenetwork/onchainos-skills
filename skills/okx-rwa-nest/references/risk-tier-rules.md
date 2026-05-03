# Risk-Tier Mapping for `recommend`

The `recommend` subcommand ranks live vaults by underlying-asset category. Categories come from Nest's `/vaults/{slug}/details` endpoint via composition (or, when not yet available, the plugin falls back to mid-tier).

## Mapping

| Category | Risk score | Rationale | Examples |
|---|---|---|---|
| `treasuries` | 1 (lowest) | US sovereign credit, T-bills <1y | nTBILL |
| `regulated-fund` | 1 | Regulated money-market or fund-of-funds | nWISDOM |
| `mixed` | 2 | Diversified multi-category | nALPHA |
| `basis` | 2 | Market-neutral basis trade | nBASIS |
| `private-credit` | 3 (highest) | Private corporate / consumer debt | nOPAL, nINSTO, nCREDIT |
| `clo` | 3 | Collateralized loan obligations | nELIXIR |

## Risk-tier user choices

| User says | Plugin `--risk` | Behavior |
|---|---|---|
| "safest", "lowest risk", "most conservative" | `conservative` | Score 1 first; tiebreak by APY desc |
| "balanced", "medium risk" | `balanced` | Prefer atomicQueue redemption (faster) over cooldown; tiebreak by closest to 6% APY |
| "highest yield", "aggressive", "best APY" | `aggressive` | Sort by sec30d desc regardless of category |

## Fallbacks

- If a vault has no category set yet, the plugin defaults to mid-tier (score 2). This means new vaults appear "balanced" until categorized.
- If a user gives an ambiguous risk preference (e.g. "I want some yield"), default to `conservative` and tell them: *"I'll suggest the safer end. Say 'aggressive' if you want higher APY at higher risk."*
- If only one vault is live, present it regardless of score and state its category clearly.

## Disclosure

When the recommendation is `score=3` (private credit or CLO), surface a one-line risk note in the user reply:

> This vault uses [private credit / CLO] which can have higher default risk and longer redemption windows.

Do not suppress this note even when the user has explicitly asked for `--risk aggressive`. It is a disclosure, not a warning gate.

## APY field precedence

For ranking and display, use fields in this order of preference:

1. `apy.sec30d` — 30-day SEC-compliant yield (most comparable across vaults)
2. `apy.rolling30d` — 30-day rolling APY (fallback if sec30d absent)
3. `apy.rolling7d` — 7-day rolling APY (last resort)

Always label which field is shown: e.g. "5.12% (30d SEC)" vs "5.40% (7d rolling)".

## Why these tiers

The plugin's ranking is opinionated — we map categories to tiers by their typical default-risk profile. Treasuries and regulated funds carry sovereign or regulated-issuer risk (lowest). Private credit and CLO carry borrower-default risk on top (highest). Mixed/basis sit in between because they spread across multiple risk types.

The user can always override with `--risk aggressive` or by naming a vault directly. The ranking is a default, not a wall.

## `recommend` output fields

The command returns a ranked array. Each entry includes:

| Field | Displayed as |
|---|---|
| `slug` | Vault identifier (e.g. `nest-treasury-vault`) |
| `symbol` | Token name (e.g. `nTBILL`) |
| `category` | Raw category string |
| `riskScore` | 1 / 2 / 3 |
| `apy` | Formatted percent string (`5.12%`) |
| `tvl` | USD value (`$340M`) |
| `redemptionType` | `atomicQueue` (boring) or `cooldown` (nest/boringNest) |

Always show `symbol` and `apy` at minimum. Show `redemptionType` when the user has expressed a preference for liquidity.
