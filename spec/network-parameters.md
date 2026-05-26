# Polaris Network Parameters

**Version**: 0.1-draft
**Chain**: Arbitrum One (42161) | **Testnet**: Arbitrum Sepolia (421614)

All values are genesis proposals subject to revision before mainnet deployment.
All mutable parameters are controlled by on-chain, stake-weighted governance.

## Token addresses (Arbitrum One)

| Token | Address | Notes |
|-------|---------|-------|
| USDC (native) | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` | Not USDC.e |
| wstETH | `0x5979D7b546E38E414F7E9822514be443A4800529` | Lido wrapped stETH |

## HorizonStaking

| Parameter | Value | Notes |
|-----------|-------|-------|
| Stake token | wstETH | |
| `minimumProvisionTokens` | 0.1 wstETH | ~$250 at current rates |
| `thawingPeriod` | 2,419,200s (28 days) | Matches Horizon |
| `maxThawingPeriod` | 15,768,000s (6 months) | Matches Horizon |
| `maxSlashingPercentage` | 500,000 (50%) | Matches Horizon |
| `delegationFeeCut` | 100,000 (1%) | |

## SubgraphService

| Parameter | Value | Notes |
|-----------|-------|-------|
| `disputeDeposit` | 0.005 wstETH | ~$12–18 at current rates |
| Arbitrator | Permissionless on-chain role | Anyone may prove a valid divergence |

## GraphPayments / PaymentsEscrow

| Parameter | Value |
|-----------|-------|
| Payment token | USDC |
| Initial whitelisted tokens | USDC only |

## RewardsManager / IssuanceAllocator

| Parameter | Value | Notes |
|-----------|-------|-------|
| Reward epoch | 7 days | |
| `weeklyBudget` | 20,000 USDC | ~$1.04M/year at genesis |
| `minimumBalance` | 100,000 USDC | Emits public event if crossed |
| Eligibility oracle | Community-operated | Signer set updatable via on-chain governance |
| Eligibility validity window | 48 hours | Attestations expire if not renewed |

## On-Chain Governance

| Parameter | Value |
|-----------|-------|
| Mechanism | Stake-weighted (wstETH provisions) |
| Standard delay | 7 days |
| Emergency pause duration | 72 hours max |
| Deployer key | Renounced post-deploy |

## Testnet deployment (Arbitrum Sepolia)

Addresses TBD. Populated after first testnet deploy.
