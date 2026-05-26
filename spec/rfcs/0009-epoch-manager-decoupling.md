# RFC 0009: EpochManager Decoupling from RewardsManager

| Field | Value |
|-------|-------|
| RFC | 0009 |
| Title | EpochManager Decoupling from RewardsManager |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0004 (RewardsManager redesign) |
| Closes issue | #9 |

---

## Summary

Remove `EpochManager` as a hard dependency of `RewardsManager`. Epoch tracking for reward distribution moves into `IssuanceAllocator`. `EpochManager` is retained as a standalone deployable utility for subgraph developers who want block-based epoch tooling, but the reward system no longer depends on it.

---

## Motivation

In Horizon, `RewardsManager` uses `EpochManager` to determine when reward periods begin and end. This coupling made sense when rewards were driven by per-epoch GRT issuance. In Polaris, rewards are driven by a 7-day wall-clock cadence managed by `IssuanceAllocator`. The `EpochManager`'s block-based epoch model is irrelevant to this — and keeping the dependency adds an extra contract to the upgrade path and the audit scope.

---

## Specification

### RewardsManager changes

`RewardsManager` no longer calls `IEpochManager(epochManager).currentEpoch()`. It tracks its own `currentEpoch` counter and `epochStartTime` (see RFC 0004).

Remove from `GraphDirectory`:
```solidity
// Removed from GraphDirectory
address internal _epochManager;
```

`RewardsManager` still emits `uint256 epoch` in events for human readability, but this epoch number is internal to the rewards system, not correlated with `EpochManager`'s block-based epochs.

### EpochManager status

`EpochManager.sol` is retained in the codebase as a standalone contract but:
- Is not deployed as part of the core Polaris protocol
- Has no dependency from any core contract
- Can be deployed independently by developers who find it useful
- Is removed from the canonical deployment script

### GraphDirectory changes

Remove `_epochManager` from `GraphDirectory`. Storage gap must be adjusted to avoid layout collisions (same concern as RFC 0007).

---

## Drawbacks

- Subgraph developers who relied on `EpochManager.currentEpoch()` for on-chain epoch logic must use an alternative. The standalone `EpochManager` deployment covers this use case.

---

## Security considerations

Reduced dependency surface. No new risks.

---

## Implementation notes

- Modify: `RewardsManager.sol` (remove EpochManager calls — done as part of RFC 0004 implementation)
- Modify: `GraphDirectory.sol` (remove `_epochManager` — coordinate with RFC 0007 storage layout work)
- Keep: `EpochManager.sol` as a standalone contract; just don't deploy it in the core script
- Trivial MR, can be bundled with RFC 0004 implementation
