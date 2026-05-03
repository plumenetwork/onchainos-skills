# Routing Glossary — when this skill fires

This is the disambiguation reference for Step 0 in SKILL.md. When the LLM is unsure whether to stay in `okx-rwa-nest` or defer to another skill, consult this table.

## Trigger keyword set (English + Chinese)

**Nest-named** (any of these alone is sufficient to STAY):
`Nest`, `nest.credit`, `nTBILL`, `nALPHA`, `nWISDOM`, `nOPAL`, `nBASIS`, `nINSTO`, `nCREDIT`, `nELIXIR`, `nACRDX`, `nSCOPE`, `FalconX CLO`, `WisdomTree`

**n-vault token patterns** (buy/deposit/stake/park + these tokens → STAY):
`nTBILL`, `nALPHA`, `nWISDOM`, `nOPAL`, `nBASIS`, `nINSTO`, `nCREDIT`, `nELIXIR`, `nACRDX`, `nSCOPE`

**RWA category triggers** (verb + any of these → STAY even without Nest name):
`RWA`, `real-world asset`, `real world asset`, `tokenized treasuries`, `tokenized US treasuries`, `T-bill yield`, `treasury yield`, `treasury-backed yield`, `regulated fund onchain`, `private credit yield`, `institutional yield`, `cash management onchain`

**Chinese RWA triggers** (verb + any of these → STAY):
`国债`, `美债`, `国债收益`, `美债收益`, `RWA 收益`, `真实世界资产`, `真实收益`, `现金管理`, `闲置资金`, `闲置稳定币`, `代币化国债`, `国债代币`, `Nest 收益`

**Trigger verbs** (required in combination with a category or Nest-name trigger):
EN: `park`, `deposit`, `stake`, `invest`, `put`, `place`, `allocate`, `lock`, `lock up`, `lend`, `save`
中文: `存`, `存入`, `质押`, `投`, `投入`, `放`, `分配`, `锁仓`, `锁住`, `借出`, `储蓄`

## Decision table

| Phrase (EN / 中文) | Decision | Why |
|---|---|---|
| "Deposit 100 USDC in Nest's safest vault" | STAY | Nest named |
| "Stake 100 USDC in Nest's safest vault" | STAY | Nest named |
| "Park $100 in Nest" | STAY (ask asset) | Nest named; `$` requires stablecoin clarification |
| "Deposit 100 USDC in safest RWA" | STAY | RWA category trigger |
| "Stake 100 dollars in best RWA vault" | STAY (ask asset) | RWA + `$` requires stablecoin clarification |
| "Lock 100 USDC in tokenized treasuries" | STAY | tokenized-treasuries category trigger |
| "Buy nTBILL" | STAY | n-vault token (like HLP/HYPE rule — named vault token is an action) |
| "Stake my idle USDC for treasury yield" | STAY | treasury yield is RWA category trigger |
| "Show me RWA vaults" | STAY | RWA category, informational list action |
| "How has nTBILL performed?" | STAY | Nest-named vault, performance query → Flow D |
| "Show my Nest positions" | STAY | Nest-named, status query → Flow C |
| "在 Nest 存 100 USDC" | STAY | Nest named (Chinese) |
| "投 100 美元到最安全的 RWA" | STAY (ask asset) | RWA category + `$` (Chinese) |
| "Deposit USDC for best APY" | DEFER → `okx-defi-invest` | No RWA framing, no Nest name |
| "Stake ETH for yield" | DEFER → `okx-defi-invest` | No RWA framing, no Nest name |
| "稳定币赚收益" | DEFER → `okx-defi-invest` | Generic stable yield (Chinese), no RWA framing |
| "Deposit 100 USDC into Aave" | DEFER → `okx-dapp-discovery` | Named non-Nest DApp |
| "What is RWA?" | NEITHER (model knowledge) | Explainer, not action |
| "Explain Nest" | NEITHER (model knowledge) | Explainer, not action |
| "Is Nest safe?" | NEITHER (model knowledge) | Explainer, not action |
| "Swap USDC to nTBILL" | STAY | n-vault token named; `swap` here means deposit (not DEX trade) |
| "Exchange ETH for USDC" | DEFER → `okx-dex-swap` | No RWA framing; pure DEX swap intent |
| "Check my wallet balance" | DEFER → `okx-agentic-wallet` | No Nest framing; generic wallet action |
| "Show DeFi positions" | DEFER → `okx-defi-portfolio` | No Nest specifics; generic DeFi portfolio |

## Verb taxonomy

EN: park, deposit, stake, invest, put, place, allocate, lock, lock up, lend, save
中文: 存, 存入, 质押, 投, 投入, 放, 分配, 锁仓, 锁住, 借出, 储蓄

These verbs STAY only when combined with a Nest-name or RWA-category trigger. Verb alone is not enough.

## Asset / amount expressions

- Token-named: `100 USDC`, `100 USDT`, `100 pUSD`, `100 USDG`, `100 USD`, `100 dai`, `100 stablecoins`
- Dollar-shorthand: `$100`, `100 dollars`, `100 dollar`, `100 bucks`, `100 美元`, `100 刀`, `100 块钱`
- Generic: `my idle stables`, `my stablecoins`, `my cash`, `my idle USDC`, `闲置稳定币`, `闲置资金`

When the user gives a dollar amount with no specified asset, **ask** which stablecoin before any calldata is built. The acceptable assets per vault come from `onchainos-nest vaults --slug <slug>` → `liquidAssets[]`.

## Defer offer line

When deferring a generic-yield query to `okx-defi-invest`, prepend this offer **before** invoking the next skill:

> If you'd prefer **RWA-backed yield** (tokenized US Treasuries, regulated funds, private credit) instead of crypto-native lending, just say *"show me RWA vaults"* and I'll switch to Nest. Otherwise, here are the best stable-yield options across DeFi:

Chinese:

> 如果您更想要 **RWA 真实世界资产收益**（代币化国债、合规基金、私募信贷），告诉我"看 RWA 金库"我就切到 Nest。否则，这里是 DeFi 上最好的稳定币收益选择：

## Anti-triggers (do NOT fire)

- "What is RWA?" — explainer, model knowledge
- "Explain Nest" — explainer
- "Is Nest safe?" — explainer
- "Show my balance" with no Nest framing — `okx-agentic-wallet`
- "Buy ETH" — `okx-dex-swap`
- "Deposit USDC into Aave" — `okx-dapp-discovery`
- "Best yield on USDC" (no RWA framing) — `okx-defi-invest`
- "Earn yield on stablecoins" (no RWA framing) — `okx-defi-invest`
