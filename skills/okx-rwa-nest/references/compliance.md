# Compliance & Geo Rules

Nest enforces compliance per-deposit via a signed `predicateMessage`. The flow:

1. Skill calls `onchainos-nest eligibility --address <user> --chain-id 1 --is-new-proxy <bool>`.
2. Plugin hits `GET /user/<addr>/compliance` and returns the signed predicate.
3. `build-deposit` consumes the predicate; the on-chain `PredicateProxy.deposit` validates the signatures.

## US Block (defense-in-depth)

The `eligibility` subcommand accepts `--country <ISO2>`. If the user self-attests `US` (case-insensitive), the plugin hard-blocks BEFORE any HTTP call to Nest. This is defense-in-depth — the underlying compliance API would block US persons too, but we don't want to leak any data to Nest's API for users we know are blocked.

The skill MUST ask the user to self-attest a country code if it has not been provided. **Never proceed without an explicit country.** Don't infer from IP or wallet history.

## Predicate Expiry

The predicate signature is time-bound. Two fields:

- `expireByBlockNumber` — legacy field name, **interpreted as a Unix timestamp in seconds** by the contract (not a block number — see `references/api-cookbook.md` Notes for the full explanation). Used by `OLD_PREDICATE_PROXY` (boring vaults).
- `expireByTime` — the canonical Unix-epoch field, used by the new (nest/boringNest) PredicateProxy (`NEW_PREDICATE_PROXY`).

The `eligibility` subcommand generates both fields correctly. If you are calling the compliance API directly (e.g. scripting), ensure the expiry value is at least a few minutes in the future relative to the current Unix time — a hardcoded value like `99999999` represents 1973 and will always revert.

If the user takes too long between `eligibility` and `build-deposit` broadcast, the on-chain check fails with `Predicate.validateSignatures: transaction expired`. The skill auto-reruns `eligibility` (max 2 retries) then stops. See Retry Protocol below.

## Sanctions / KYT

Nest's compliance API is the authoritative source for sanctions screening. We do not run our own KYT. If `data.isCompliant: false`, surface `data.message` verbatim (when present) — never override.

Non-compliant responses come back as `HTTP 200` with `{ isCompliant: false, message: "...", signatures: [] }`. There is no HTTP 4xx/5xx error code. Treat `isCompliant: false` as a hard stop regardless.

## Trust Boundary

We treat the `predicateMessage` as an opaque, time-bound credential. The plugin:

- never logs it
- never persists it
- never embeds it in error messages
- validates only the structural shape (zod schema)
- relies on on-chain validation for signature correctness

The signatures are verified by `PredicateProxy` at deposit time using a registered set of signer addresses configured in the contract by Nest's protocol team. We don't know the signer set; we just pass the predicate through.

## What we do NOT check

- We do not call Nest's API to verify a predicate was signed correctly. The on-chain contract does that.
- We do not gate on user-provided KYC documents. Nest's API + on-chain proxy is the gate.
- We do not store any compliance state per-user. Each deposit re-fetches a fresh predicate.
- We do not interpret or translate compliance denial reasons. Surface `data.message` as-is.

## Compliance vs Security Scan

Compliance (`eligibility`) and security scanning (`okx-security tx-scan`) are independent gates that both run before every deposit broadcast. They are not interchangeable:

| Gate | What it checks | Who enforces |
|---|---|---|
| `eligibility` | Geo/sanctions eligibility of the depositor | Nest API + on-chain PredicateProxy |
| `tx-scan` | Transaction payload safety (phishing, unlimited approvals, etc.) | OKX Security |

Both must pass. A compliance-approved user can still be blocked by `tx-scan` if the calldata looks suspicious.

## `isNewProxy` routing

Pass `--is-new-proxy` (i.e. `isNewProxy: true`) when the target vault is of type `nest` or `boringNest`. The compliance API uses this to route the predicate to `NEW_PREDICATE_PROXY`. For `boring` vault types, omit the flag (defaults `false`), which routes to `OLD_PREDICATE_PROXY`. The vault type is available from `onchainos-nest vaults --slug <slug>` → `.vaultType`.

## Region Codes

Currently only `US` triggers the local hard-block. Other regions are evaluated by Nest's compliance API. If a user's region is restricted, the API returns `isCompliant: false` with a message — surface it verbatim.

## Retry Protocol

On predicate expiry at broadcast time:

1. Re-run `onchainos-nest eligibility --address <user> --chain-id <id> [--is-new-proxy]`.
2. Save new `predicateMessage` to `/tmp/predicate.json`.
3. Re-run `build-deposit` with `--predicate-message @/tmp/predicate.json`.
4. If the second attempt expires (e.g. user was idle), repeat once more (retry 2).
5. After 2 retries without a successful broadcast, stop and tell the user to restart the deposit.

Never silently swallow the expiry error. Always inform the user when a retry is happening.
