# Polaris Technical Specification

**Version**: 0.1-draft
**Chain**: Arbitrum One (chainId 42161)
**Testnet**: Arbitrum Sepolia (chainId 421614)
**Upstream base**: `graphprotocol/contracts` @ Horizon mainnet commit (2 Dec 2025)

---

## 1. Overview

Polaris reuses the Horizon contract architecture in full, changing three things and deleting several others:

| Change | Description |
|--------|-------------|
| Stake token | GRT → wstETH (raw ETH depositable via Lido adapter) |
| Payment token | GRT → USDC (Arbitrum One native: `0xaf88d065e77c8cC2239327C5EDb3A432268e5831`) |
| Reward source | Token inflation → USDC deposited to `IssuanceAllocator` by anyone |

Everything else — the provision model, the thawing/undelegation mechanics, the TAP v2 receipt/RAV aggregation, the SubgraphService dispute machinery, the Data Service framework — is carried forward without semantic change.

The protocol is permissionless and has no privileged operators. Anyone may run a node, index a subgraph, delegate stake, or fund the IssuanceAllocator.

---

## 2. Contract Inventory

### 2.1 Carried forward (minimal or no change)

| Contract | Change notes |
|----------|-------------|
| `HorizonStaking` | Stake token changed to wstETH. See §3. |
| `SubgraphService` | Dispute deposit recalibrated for ETH denomination. See §4. |
| `GraphTallyCollector` (TAP v2 verifier) | Unchanged. Receipt/RAV token changed to USDC downstream. |
| `PaymentsEscrow` | Payment token changed to USDC. See §5. |
| `GraphPayments` | Payment token changed to USDC. See §5. |
| `Authorizable` | Unchanged. |
| `GraphDirectory` | Updated to replace `_graphToken` with `_stakeToken` (wstETH) and `_paymentToken` (USDC). |
| `GraphProxyAdmin` | Unchanged. |

### 2.2 Redesigned

| Contract | Change notes |
|----------|-------------|
| `RewardsManager` | Inflation pathway removed entirely. Budget sourced from `IssuanceAllocator`. Eligibility gated by `RewardsEligibilityOracle` from genesis. See §6. |
| `IssuanceAllocator` | Budget input changed from minted GRT to permissionless USDC deposits. Allocation targets updatable via on-chain governance. See §7. |

### 2.3 Deleted

| Contract | Reason |
|----------|--------|
| `GraphToken` | No native token. |
| `L1GraphTokenGateway` / `L2GraphTokenGateway` | No token to bridge. |
| `BridgeEscrow` | No token bridge. |
| `Curation` | Replaced by gateway-driven indexing payments (GIP-0081 pattern). Reconsidered post-v1 if demand data warrants. |
| `EpochManager` | Rewards are continuous, not epoch-batched. Dependency removed. |
| `Staking` (legacy pre-Horizon monolith) | No migration debt; Polaris is greenfield. |
| `AllocationExchange` | Legacy allocation model not used. |
| Token vesting contracts (`TokenLockWallet`, `TokenLockManager`) | No token to vest. |
| L1-side mainnet contracts | Polaris is Arbitrum One native. No L1 deployment. |

---

## 3. HorizonStaking — Stake Token Change

### 3.1 Token

The stake token is changed from GRT to **wstETH** (`0x5979D7b546E38E414F7E9822514be443A4800529` on Arbitrum One).

Raw ETH deposits are accepted via a thin **LidoAdapter** contract:
- Accepts ETH, calls `Lido.submit()` → receives stETH, wraps to wstETH via `WstETH.wrap()`, credits `HorizonStaking` on behalf of the depositor.
- Withdrawals are wstETH only (unwrapping is the operator's responsibility).

### 3.2 Parameters (genesis values — subject to revision)

| Parameter | Value | Notes |
|-----------|-------|-------|
| `minimumProvisionTokens` | 0.1 wstETH | ~$250–350 at current rates |
| `thawingPeriod` | 2,419,200 seconds (28 days) | Matches Horizon mainnet |
| `maxThawingPeriod` | 15,768,000 seconds (6 months) | Matches Horizon mainnet |
| `delegationFeeCut` | 100,000 (1%) | Indexer's cut of delegation rewards |
| `maxSlashingPercentage` | 500,000 (50%) | Matches Horizon mainnet |

### 3.3 Slashing

Slashing is denominated in wstETH. A successful PoI-divergence dispute slashes the indexer's provision. Slashed funds are sent to the fisherman reward pool net of any protocol-reserve allocation (configurable on-chain, initially 0%).

---

## 4. SubgraphService

### 4.1 Dispute deposits

Dispute deposits are denominated in wstETH. Genesis value: **0.005 wstETH** (~$12–18 at current rates). Low enough not to deter legitimate disputes; high enough to deter spam.

### 4.2 Indexing reward eligibility

The `RewardsEligibilityOracle` (§6.2) is required from genesis. An indexer without a positive eligibility signal receives zero rewards for the relevant deployment regardless of stake weight.

### 4.3 PoI verification

PoI verification logic is unchanged from the Horizon SubgraphService. The dispute arbitrator is an on-chain role, permissionlessly callable by any address that successfully proves a divergence (same as Horizon).

---

## 5. GraphPayments and PaymentsEscrow

### 5.1 Payment token

Payment token: **USDC** (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831` on Arbitrum One).

No other payment token is whitelisted at genesis. The token whitelist is updatable via on-chain governance (not a privileged role — see §8).

### 5.2 TAP v2 (GraphTally)

The `GraphTallyCollector` is carried forward unchanged. The USDC denomination enters at `PaymentsEscrow.collect()`.

### 5.3 Gateway model

Gateways escrow USDC into `PaymentsEscrow` against a set of authorised indexers. The authorisation model (EIP-712 signed authorisations, revocable, per-indexer spend caps) is unchanged from Horizon. Anyone may operate a gateway.

---

## 6. RewardsManager

### 6.1 Inflation removal

The GRT inflation pathway is removed entirely. `RewardsManager` no longer calls `GraphToken.mint()`. The curation-signal-weighted `rewardsPerSignal` accumulator and the `subgraphAvailabilityOracle` bypass are removed.

### 6.2 Budget source

`RewardsManager` receives a periodic USDC budget from `IssuanceAllocator` (§7). Budget is distributed to eligible indexers proportionally to their stake weight across active provisions, subject to oracle eligibility.

Distribution cadence: **7-day epochs**. Indexers claim rewards by calling `RewardsManager.claim(indexer, provision)` after epoch close. Unclaimed rewards accumulate.

### 6.3 RewardsEligibilityOracle

The oracle attests per-indexer, per-deployment eligibility on a 24-hour basis. Attestations are signed EIP-712 structs submitted to `RewardsManager.setEligibility()`.

The oracle role is permissionless — any address may submit attestations. Invalid attestations (wrong signer) are rejected. The oracle signer set is updatable via on-chain governance.

Genesis eligibility criteria:
- Indexer has responded to ≥95% of sampled queries in the preceding 24h window.
- Indexer's latest PoI for the deployment is within 1 block of the canonical PoI.
- Indexer has no unresolved dispute open for >7 days.

These criteria are encoded as an off-chain convention (not hard-coded) and updated by community consensus via the PIP process.

---

## 7. IssuanceAllocator

### 7.1 Role

`IssuanceAllocator` is the sole budget source for `RewardsManager`. It holds USDC and disburses a configured weekly amount on epoch tick. It cannot mint tokens.

**Anyone may deposit USDC into `IssuanceAllocator`.** There is no permissioned depositor role.

### 7.2 Budget configuration

| Parameter | Genesis value | Notes |
|-----------|--------------|-------|
| `weeklyBudget` | 20,000 USDC | ~$1.04M/year; adjustable via on-chain governance |
| `minimumBalance` | 100,000 USDC | Emits a public event if balance falls below this |

`weeklyBudget` and allocation targets are updatable via on-chain governance only (see §8). No privileged admin role.

### 7.3 Funding

`IssuanceAllocator` is funded by community deposits. No party has a privileged right to direct these funds once deposited; deposits are irrevocable (the contract disburses to `RewardsManager` on schedule, not to depositors).

---

## 8. On-Chain Governance

Polaris has no off-chain governance body, no Foundation, and no council. All mutable parameters are controlled by on-chain governance.

**Governance mechanism**: stake-weighted voting via wstETH provisions in `HorizonStaking`. Voting weight is proportional to provision size. Any provisioned indexer may propose and vote on parameter changes.

| Parameter type | Change mechanism |
|----------------|-----------------|
| Protocol parameters | On-chain governance vote, 7-day delay |
| Emergency pause | On-chain governance vote, no delay (72h max) |
| Oracle signer rotation | On-chain governance vote, 7-day delay |
| Payment token whitelist | On-chain governance vote, 7-day delay |

There are no privileged roles at genesis. The deployer key is renounced after deployment.

---

## 9. Network parameters (genesis summary)

| Parameter | Value |
|-----------|-------|
| Chain | Arbitrum One (42161) |
| Stake token | wstETH |
| Payment token | USDC |
| Minimum provision | 0.1 wstETH |
| Thawing period | 28 days |
| Max slashing | 50% |
| Dispute deposit | 0.005 wstETH |
| Reward epoch | 7 days |
| Weekly reward budget | 20,000 USDC |
| Governance | On-chain, stake-weighted |
| Governance delay | 7 days |
| Emergency pause | 72h max, on-chain vote |

---

## 10. Open questions

- [ ] **LidoAdapter withdrawal path**: wstETH-only deposits at genesis (L1 Lido submit not available on Arbitrum); ship LidoAdapter for L1 routing in v1.1?
- [ ] **wstETH price oracle**: Chainlink wstETH/USD on Arbitrum for USD-denominated slashing penalties?
- [ ] **Curation v2**: gateway-driven allowlist (GIP-0081 pattern) is the genesis answer; sufficient long-term?
- [ ] **Oracle decentralisation**: who runs the `RewardsEligibilityOracle` at genesis before community oracle operators emerge? Fisherman monitor network?
- [ ] **IssuanceAllocator funding**: what is the realistic community deposit path at genesis to seed the weekly budget?
- [ ] **GraphTally / TAP v2 audit scope**: upstream audit covered TAP v1 on GRT/Ethereum. Additional audit scope needed for USDC/Arbitrum One deployment.
- [ ] **Fisherman portability**: is `cargopete/fishing-business` portable to Polaris with wstETH-denominated slashing?
