# Polaris Contracts

Fork of `graphprotocol/contracts` + `graphprotocol/horizon` at the Horizon mainnet commit (2 Dec 2025).

Polaris changes three things from Horizon and deletes several others. See [DESIGN-LOG.md](DESIGN-LOG.md) for the full modification inventory. See [NOTICE](NOTICE) for upstream attribution.

## What changed

| Area | Change |
|------|--------|
| Stake token | GRT → wstETH (+ LidoAdapter for raw ETH deposits) |
| Payment token | GRT → USDC |
| Reward source | Token inflation → permissionless USDC deposits via IssuanceAllocator |
| Curation | Deleted. Gateway-driven indexing payments (GIP-0081 pattern) from genesis. |
| RewardsManager | Redesigned — no mint(), budget from IssuanceAllocator, eligibility from oracle |
| GraphToken + bridges | Deleted. No native token. |
| L1 deployment | Deleted. Arbitrum One native only. |

## Setup

```bash
# Dependencies
forge install

# Compile
forge build

# Test
forge test
```

Requires [Foundry](https://book.getfoundry.sh/).

## Network addresses

Populated after testnet deployment. See `deployments/` once available.

## Audit status

Pre-audit. The upstream Horizon contracts were audited by OpenZeppelin and Dedaub (see upstream repo for reports). Polaris modifications require a new audit covering:
- LidoAdapter
- RewardsManager redesign
- IssuanceAllocator
- wstETH/USDC token parameter propagation across HorizonStaking, SubgraphService, GraphPayments, PaymentsEscrow

## Licence

Apache-2.0. See [NOTICE](NOTICE) for upstream attribution.
