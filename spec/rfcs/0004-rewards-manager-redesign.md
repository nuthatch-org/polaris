# RFC 0004: RewardsManager Redesign — Inflation Removal and Treasury Budget

| Field | Value |
|-------|-------|
| RFC | 0004 |
| Title | RewardsManager Redesign — Inflation Removal and Treasury Budget |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0005 (IssuanceAllocator), RFC 0010 (RewardsEligibilityOracle) |
| Closes issue | #4 |

---

## Summary

Remove the GRT inflation pathway from `RewardsManager` entirely. Replace it with a USDC budget drawn from `IssuanceAllocator` on a 7-day epoch cadence. Reward eligibility is gated by `RewardsEligibilityOracle` from genesis. Distribution is stake-weighted across eligible provisions.

---

## Motivation

The existing `RewardsManager` in Horizon calls `GraphToken.mint()` to create new GRT for indexer rewards. This is the mechanism that produced ~$27.5 million in inflationary rewards in 2025 against ~$750k of organic query fee revenue — a ~36:1 subsidy ratio that has not declined despite five years of network operation.

The problems with inflation-funded rewards in GRT:
1. Every indexer that receives GRT rewards must sell some to cover USD-denominated costs, creating constant sell pressure.
2. The GRT price decline means the dollar value of rewards shrinks every quarter even if GRT issuance in token terms is constant or increasing.
3. New GRT supply is paid for by all existing GRT holders via dilution, including delegators who may not understand this.
4. The inflation creates a regulatory ambiguity (is GRT a security?) that prevents institutional deployment.

The Polaris alternative: USDC rewards funded by a permissionlessly-deposited `IssuanceAllocator`. The budget is explicit, visible, and in dollars. It does not require minting anything. It does not dilute anything. It runs out if it is not replenished — which is a feature, not a bug.

---

## Specification

### Removed from Horizon RewardsManager

- `GraphToken.mint()` call
- `rewardsPerSignal` accumulator (curation-signal-weighted issuance)
- `subgraphAvailabilityOracle` — the curation bypass mechanism
- `EpochManager` hard dependency (epoch tracking moves to `IssuanceAllocator`)
- `onSubgraphAllocationUpdated` / `onSubgraphSignalUpdated` hooks (curation-related)
- `getNewRewardsPerSignal()` and all related view functions

### Added to RewardsManager

#### Epoch model

```solidity
uint256 public currentEpoch;
uint256 public epochStartTime;
uint256 public constant EPOCH_DURATION = 7 days;

/// @notice Advance the epoch. Callable by anyone once EPOCH_DURATION has elapsed.
function advanceEpoch() external {
    require(block.timestamp >= epochStartTime + EPOCH_DURATION, "epoch not over");
    _snapshotEpoch();
    currentEpoch++;
    epochStartTime = block.timestamp;
    emit EpochAdvanced(currentEpoch, epochStartTime);
}
```

Anyone may call `advanceEpoch()`. There is no privileged keeper.

#### Budget request

At epoch advance, `RewardsManager` requests the configured USDC amount from `IssuanceAllocator`:

```solidity
function _snapshotEpoch() internal {
    uint256 budget = IIssuanceAllocator(issuanceAllocator).disburse();
    epochBudget[currentEpoch] = budget;
    totalEligibleStake[currentEpoch] = _computeEligibleStake();
}
```

If `IssuanceAllocator` has insufficient balance, `disburse()` returns the available amount (which may be 0). The protocol does not revert; it records a 0-budget epoch.

#### Eligibility check

Before computing `_computeEligibleStake()`, each active provision is checked against `RewardsEligibilityOracle`:

```solidity
function _computeEligibleStake() internal view returns (uint256 total) {
    for each active (indexer, provision) {
        if (IRewardsEligibilityOracle(oracle).isEligible(indexer, provision.deploymentId)) {
            total += provision.tokens;
        }
    }
}
```

Ineligible provisions contribute zero to the denominator and receive zero rewards.

#### Reward accumulator

```solidity
mapping(uint256 epoch => mapping(address indexer => mapping(bytes32 deploymentId => uint256 rewardShare)))
    public epochRewards;
```

Rewards are not pushed; they are pull-based. Indexers call `claim(indexer, deploymentId)` to collect accumulated USDC.

#### Distribution formula

```
rewardShare(indexer, deploymentId, epoch) =
    (provision.tokens / totalEligibleStake[epoch]) * epochBudget[epoch]
```

Computed lazily at `claim()` time (or eagerly at epoch snapshot — TBD based on gas analysis).

#### Claim function

```solidity
function claim(address indexer, bytes32 deploymentId) external returns (uint256 amount) {
    // sum unclaimed epochs
    // transfer USDC to indexer's payment address
    // mark epochs as claimed
    emit RewardsClaimed(indexer, deploymentId, amount);
}
```

No access control on `claim()` — anyone may trigger a claim on behalf of an indexer (rewards go to the indexer's nominated address, not the caller).

### Events

```solidity
event EpochAdvanced(uint256 indexed epoch, uint256 startTime);
event EpochBudgetSet(uint256 indexed epoch, uint256 budget);
event RewardsClaimed(address indexed indexer, bytes32 indexed deploymentId, uint256 amount);
event IneligibleProvision(address indexed indexer, bytes32 indexed deploymentId, uint256 indexed epoch);
```

---

## Rationale

**Pull-based rather than push-based rewards**: pushing rewards on every epoch tick to all indexers is gas-prohibitive if the indexer set is large. Pull-based accumulation lets indexers batch claims across epochs.

**Anyone can advance the epoch**: removes the need for a privileged keeper. Gas cost of `advanceEpoch()` is borne by whoever calls it — in practice, this will be indexers or a public keeper bot. The epoch cannot be advanced more than once per `EPOCH_DURATION`.

**Zero-budget epochs rather than reverting**: if `IssuanceAllocator` runs dry, the protocol continues functioning (gateways still pay, indexers still serve queries) — they just receive no subsidy for that epoch. This avoids a scenario where a depleted allocator bricks the reward system.

**Stake-weighted rather than signal-weighted**: curation signal has been deleted (RFC 0006). The natural replacement is stake weight, which at least has a slashability argument: indexers with more skin in the game receive proportionally more reward.

---

## Alternatives considered

- **Epoch-push model**: compute and push rewards to all indexers at epoch tick. Gas cost scales linearly with indexer count; impractical beyond ~100 indexers. Rejected.
- **RAV-proportional rewards**: distribute rewards proportionally to TAP RAV values rather than stake. More aligned with actual work done, but RAVs are off-chain and can be gamed or withheld. Deferred to v2 as an optional weighting factor.
- **No subsidy at all**: pure query-fee-only model. Correct long-term, but without a subsidy bridge, Polaris cannot attract indexers during the bootstrap period when query fees are nascent. Deferred as a protocol parameter (weeklyBudget can be set to 0 by governance when the network reaches self-sustainability).

---

## Drawbacks

- Rewards are not automatic — someone must call `advanceEpoch()` and indexers must call `claim()`. This is operational overhead vs. Horizon's more automated distribution.
- If IssuanceAllocator goes to zero, indexers receive no subsidy without any protocol-level alarm. The `minimumBalance` event from `IssuanceAllocator` is the only signal.
- The lazy computation of `rewardShare` at claim time is gas-efficient but means the gas cost of `claim()` scales with the number of unclaimed epochs.

---

## Security considerations

- **Re-entrancy in `claim()`**: USDC transfer must follow checks-effects-interactions. Mark epochs as claimed before transferring.
- **Oracle manipulation**: if the `RewardsEligibilityOracle` is compromised, all rewards could flow to a single ineligible indexer or be withheld from all legitimate indexers. The oracle signer rotation mechanism (RFC 0010) mitigates but does not eliminate this.
- **Epoch advance griefing**: advancing the epoch costs gas but cannot be profitably gamed (it doesn't change who gets rewards, only when the snapshot happens). Low risk.
- **Division precision**: stake-weighted distribution involves integer division. At low indexer counts, rounding errors accumulate. The remaining USDC after all claims (dust) should accrue to the next epoch's budget, not sit unclaimable.

---

## Open questions

- [ ] Lazy vs. eager epoch reward computation? Eager (at `advanceEpoch()`) is simpler but gas-heavy if the indexer set is large.
- [ ] Should there be a maximum number of unclaimed epochs before rewards expire? Prevents unbounded storage growth.
- [ ] RAV-weighted subsidies as an optional multiplier on top of stake-weighted base? Deferred to v2, but worth noting here.
- [ ] Who pays gas for `advanceEpoch()`? A public keeper incentive (small USDC tip from the epoch budget to the caller) could ensure reliable epoch advancement.

---

## Implementation notes

- Primary file: `packages/contracts/contracts/rewards/RewardsManager.sol`
- Must be deployed after `IssuanceAllocator` (RFC 0005) and `RewardsEligibilityOracle` (RFC 0010)
- All upstream tests that test reward accrual against GRT mint must be replaced
- Fork test: deploy on Arbitrum One fork, seed `IssuanceAllocator` with USDC, run 3 epochs with mixed eligible/ineligible indexers, verify claim amounts
