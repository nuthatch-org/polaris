# Polaris Design Log

Per-contract record of every divergence from the Horizon mainnet baseline.
Each entry references the corresponding GitLab issue where discussion lives.

---

## DL-001: HorizonStaking — stake token GRT → wstETH

**Issue**: #1
**Status**: Specified, not implemented

Replace all references to `GraphToken` / GRT in `HorizonStaking.sol` with wstETH
(`IERC20` parameterised at constructor / initializer).

Affected:
- `GraphDirectory._graphToken` → replaced by `GraphDirectory._stakeToken` pointing to wstETH
- `stake(uint256 tokens)` — pulls wstETH from `msg.sender`
- `unstake(uint256 tokens)` — returns wstETH to operator
- `slash(address serviceProvider, uint256 tokens, uint256 reward, address rewardDestination)` — denominated in wstETH

**Open question**: stake token hardcoded at deploy time or a governance parameter? Current preference: hardcoded, to simplify the audit surface.

---

## DL-002: LidoAdapter — raw ETH → wstETH deposit wrapper

**Issue**: #2
**Status**: Specified, not implemented

New contract. Thin stateless wrapper:

```solidity
function depositETH(address staker) external payable {
    uint256 stETH = ILido(LIDO).submit{value: msg.value}(address(0));
    uint256 wstETH = IWstETH(WSTETH).wrap(stETH);
    IERC20(WSTETH).approve(address(horizonStaking), wstETH);
    IHorizonStaking(horizonStaking).stake(wstETH);
}
```

No withdrawal path — operators withdraw wstETH directly from HorizonStaking.

**Known issue**: Lido's `submit()` is an L1-only call. wstETH on Arbitrum One is a bridged token, not natively mintable on L2. Options:
- (a) Accept wstETH-only deposits at genesis; no adapter.
- (b) Route ETH to L1 via a cross-chain deposit path.

**Current lean**: wstETH-only deposits at genesis. LidoAdapter ships as v1.1.

Arbitrum One wstETH: `0x5979D7b546E38E414F7E9822514be443A4800529`

---

## DL-003: GraphPayments + PaymentsEscrow — payment token GRT → USDC

**Issue**: #3
**Status**: Specified, not implemented

`GraphPayments` holds a whitelist of accepted payment tokens. At genesis, only USDC is whitelisted:
- Arbitrum One USDC: `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` (native USDC, not USDC.e)

No semantic change to the escrow or payment flow. The `paymentType` enum, RAV collector pattern, and fee cut mechanism are unchanged.

---

## DL-004: RewardsManager — inflation pathway removed, treasury budget introduced

**Issue**: #4
**Status**: Specified, not implemented

**Deleted**:
- `GraphToken.mint()` call
- `rewardsPerSignal` accumulator and all curation-signal-weighted logic
- `subgraphAvailabilityOracle` curation bypass
- `EpochManager` dependency

**Added**:
- `IssuanceAllocator` as the sole budget source (`IERC20(USDC).transferFrom(issuanceAllocator, ...)`)
- `RewardsEligibilityOracle` eligibility check before any reward credit
- 7-day epoch tick, advanceable by any caller
- Per-indexer, per-provision reward accumulator

**Reward distribution formula** (draft):

```
rewardShare(indexer, provision) =
    (provisionTokens(indexer) / totalEligibleProvisionTokens) * epochBudget
```

`totalEligibleProvisionTokens` = sum of provision sizes for all (indexer, provision) pairs with a positive oracle eligibility flag in the current epoch.

---

## DL-005: IssuanceAllocator — permissionless USDC treasury disbursement

**Issue**: #5
**Status**: Specified, not implemented

New contract (modelled on GIP-0076 design, adapted for USDC rather than GRT issuance).

**Key property**: anyone may deposit USDC. No permissioned depositor role.

```solidity
struct AllocationTarget {
    address recipient;
    uint256 weeklyAmount; // in USDC (6 decimals)
}
```

At genesis, single target: `RewardsManager`. Allocation target list is mutable via on-chain governance (7-day delay). Deposits are irrevocable — the contract disburses to `RewardsManager` on schedule.

---

## DL-006: Curation.sol — deleted

**Issue**: #6
**Status**: Specified

`Curation.sol` and the bonding-curve infrastructure are removed entirely. No migration path needed (Polaris is greenfield).

Indexing effort is directed by gateway-driven indexing payments (GIP-0081 pattern): gateways specify which subgraph deployments they are willing to pay for; that signal drives indexer allocation decisions off-chain.

Curation v2 — if warranted — is deferred post-v1 based on real demand data.

---

## DL-007: GraphToken + bridges — deleted

**Issue**: #7
**Status**: Specified

`GraphToken.sol`, `L1GraphTokenGateway.sol`, `L2GraphTokenGateway.sol`, `BridgeEscrow.sol`, and all associated proxy and initialisation scaffolding are removed.

`GraphDirectory` updated: `_graphToken` removed; `_stakeToken` (wstETH) and `_paymentToken` (USDC) added.

---

## DL-008: Token vesting contracts — deleted

**Issue**: #8
**Status**: Specified

`TokenLockWallet.sol`, `TokenLockManager.sol` — removed. No token to vest.

---

## DL-009: EpochManager — removed as core dependency

**Issue**: #9
**Status**: Specified

`EpochManager` retained as a deployable utility but no longer a hard dependency of `RewardsManager`. The reward epoch is tracked internally in `IssuanceAllocator`.

---

## DL-010: RewardsEligibilityOracle — integrated from genesis

**Issue**: #10
**Status**: Specified, not implemented

Permissionless oracle role: any address may submit attestations; invalid attestations (wrong signer) are rejected on-chain. Oracle signer set updatable via on-chain governance.

```solidity
function setEligibility(
    address indexer,
    bytes32 deploymentId,
    bool eligible,
    uint256 validUntil
) external // no access control; signature verified on-chain
```

`validUntil` is a Unix timestamp. `RewardsManager` checks `block.timestamp < validUntil` before crediting rewards. Attestations expire after 48 hours if not renewed.

---

## DL-011: SubgraphService — dispute deposit recalibration

**Issue**: #11
**Status**: Specified, not implemented

`disputeDeposit` changed from a GRT amount to a wstETH amount.

Genesis value: **0.005 wstETH**. Approximately equivalent in dollar terms to the Horizon mainnet GRT deposit at current GRT prices, but denominated in the same asset as the stake collateral.

The dispute arbitrator role is permissionless (on-chain) — any address may prove a valid PoI divergence and claim the fisherman reward.
