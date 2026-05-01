# Nest System Contracts

The plugin (`@plumenetwork/onchainos-nest-plugin`) consumes a vendored copy of `system-contracts.json`. This file is the human-readable mirror — addresses must match the JSON byte-for-byte.

**Last verified on-chain:** 2026-05-01

## Ethereum (chainId 1)

| Contract | Address | Used by |
|---|---|---|
| `OLD_PREDICATE_PROXY` | `0x6104fe10ca937a086ba7adbd0910a4733d380cb6` | Boring vault deposits and `depositAndBridge` |
| `NEW_PREDICATE_PROXY` | `0xfC0c4222B3A0c9B060C0B959DEc62442036b9035` | Nest / boringNest vault deposits |
| `ATOMIC_QUEUE` | `0x228c44bb4885c6633f4b6c83f14622f37d5112e5` | Boring vault withdraw requests |
| `ATOMIC_SOLVER` | `0x77fb098A1C28a5b50BFAdb69Ca1bEE515a7FC974` | Settles boring withdrawals |
| `MULTICALL3` | `0xcA11bde05977b3631167028862bE2a173976CA11` | Read batching |

Per-vault contract addresses (Teller, Accountant, BoringVault, NestVault) are **not hardcoded**. They are fetched live from Nest's API at runtime, which means new vaults appear automatically without skill or plugin updates.

## Verification procedure

When proposing a release or upstream PR, re-run the on-chain bytecode checks (see `_internal/nest-agents-port-notes.md` for the procedure) and update the `verifiedAt` field in `system-contracts.json`. The plugin loads `verifiedAt` and surfaces it in `--version` output so downstream agents know the freshness of the vendored data.

## Why these addresses are universal

The `OLD_PREDICATE_PROXY`, `NEW_PREDICATE_PROXY`, and `ATOMIC_QUEUE` are deployed on multiple chains via CREATE2-deterministic deployment, which means the same address resolves to the same logical contract on every chain that has a Nest deployment (Ethereum, Plume, plus any future chains Nest adds). The plugin treats them as universal constants per chain — when a chain is added, the same address is expected to be live there.

If a future chain breaks the CREATE2 invariant (different address on that chain), this file gains a per-chain entry under `chains.<chainId>` with the chain-specific override.

## Other chains

Currently only Ethereum (chainId 1) is committed to the JSON. When OKX adds Plume (chainId 98866) to its supported wallet chain list, this file gains a `"98866"` entry by re-running the bytecode verification on a Plume RPC. Adding the entry is a single-line PR — no schema or plugin code changes.
