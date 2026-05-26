# RFC 0005: IssuanceAllocator — Permissionless USDC Disbursement

| Field | Value |
|-------|-------|
| RFC | 0005 |
| Title | IssuanceAllocator — Permissionless USDC Disbursement |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0003 (USDC payment token) |
| Closes issue | #5 |

---

## Summary

A new contract, `IssuanceAllocator`, that accepts permissionless USDC deposits and disburses a configured weekly amount to `RewardsManager` on epoch tick. No privileged depositor. No minting. Deposits are irrevocable. The weekly budget and allocation targets are updatable via on-chain governance.

---

## Motivation

Polaris needs a bridge between "USDC deposited by community participants who want to fund indexer rewards" and "USDC disbursed to RewardsManager on a predictable schedule". The `IssuanceAllocator` is that bridge.

The design goals are:
1. **Permissionless deposits**: anyone may contribute to the reward pool. No whitelist, no minimum, no KYC.
2. **Predictable disbursement**: `RewardsManager` receives a known USDC amount each epoch, not whatever happened to be in the contract at epoch-tick time.
3. **Explicit, not inflationary**: the total USDC in the allocator is visible on-chain. When it runs out, it runs out. There is no printing press.
4. **Governable parameters**: the weekly budget and the set of allocation targets (which contracts receive USDC) are updatable via on-chain governance, allowing the community to adjust the subsidy rate as query-fee revenue grows.

---

## Specification

### Contract interface

```solidity
interface IIssuanceAllocator {

    struct AllocationTarget {
        address recipient;
        uint256 weeklyAmount; // in USDC, 6 decimals
    }

    /// @notice Deposit USDC into the allocator. Permissionless.
    /// @param amount Amount of USDC (6 decimals) to deposit.
    function deposit(uint256 amount) external;

    /// @notice Disburse the configured weekly amount to all allocation targets.
    ///         Callable only by a registered allocation target (i.e. RewardsManager).
    ///         Returns the amount actually disbursed (may be less than configured if balance is insufficient).
    function disburse() external returns (uint256 disbursed);

    /// @notice Update allocation targets. On-chain governance only; 7-day delay.
    function setAllocationTargets(AllocationTarget[] calldata targets) external onlyGovernance;

    /// @notice Update the minimum balance alert threshold.
    function setMinimumBalance(uint256 amount) external onlyGovernance;

    /// @notice Current USDC balance.
    function balance() external view returns (uint256);

    /// @notice Total USDC deposited over the contract's lifetime.
    function totalDeposited() external view returns (uint256);

    /// @notice Total USDC disbursed over the contract's lifetime.
    function totalDisbursed() external view returns (uint256);

    event Deposited(address indexed depositor, uint256 amount);
    event Disbursed(address indexed recipient, uint256 amount, uint256 epoch);
    event LowBalance(uint256 balance, uint256 minimumBalance);
    event AllocationTargetsUpdated(AllocationTarget[] targets);
}
```

### Deposit mechanics

```solidity
function deposit(uint256 amount) external {
    IERC20(USDC).transferFrom(msg.sender, address(this), amount);
    totalDeposited += amount;
    emit Deposited(msg.sender, amount);
    if (IERC20(USDC).balanceOf(address(this)) < minimumBalance) {
        emit LowBalance(IERC20(USDC).balanceOf(address(this)), minimumBalance);
    }
}
```

Deposits are irrevocable. There is no `withdraw()` function. USDC deposited into `IssuanceAllocator` is committed to the reward pool; it cannot be retrieved by the depositor or by governance.

This is a deliberate design choice: it makes the contract maximally trustworthy (no rug pathway) at the cost of depositor flexibility.

### Disburse mechanics

```solidity
function disburse() external returns (uint256 disbursed) {
    require(isAllocationTarget(msg.sender), "not a target");
    AllocationTarget storage target = allocationTargets[msg.sender];
    uint256 available = IERC20(USDC).balanceOf(address(this));
    disbursed = available >= target.weeklyAmount ? target.weeklyAmount : available;
    if (disbursed > 0) {
        IERC20(USDC).transfer(msg.sender, disbursed);
        totalDisbursed += disbursed;
        emit Disbursed(msg.sender, disbursed, currentEpoch);
    }
    if (IERC20(USDC).balanceOf(address(this)) < minimumBalance) {
        emit LowBalance(IERC20(USDC).balanceOf(address(this)), minimumBalance);
    }
}
```

`disburse()` is called by `RewardsManager` at epoch advance. It is not permissionless — only registered allocation targets may call it. This prevents a third party from draining the allocator by repeatedly calling `disburse()`.

The "only registered targets may call `disburse()`" access control is the only privileged function in the contract. The set of registered targets is updatable via on-chain governance.

### Genesis configuration

| Parameter | Value |
|-----------|-------|
| `weeklyAmount` for RewardsManager | 20,000 USDC |
| `minimumBalance` | 100,000 USDC (~5 weeks of disbursements) |
| Allocation targets at genesis | `[{recipient: RewardsManager, weeklyAmount: 20_000e6}]` |

### Governance controls

`setAllocationTargets()` and `setMinimumBalance()` are callable only via on-chain governance with a 7-day delay. This means adjusting the weekly budget requires a governance vote and a week's notice — indexers can plan around it.

There is no `pause()` function. There is no emergency withdrawal. The contract can be effectively zeroed by setting `weeklyAmount` to 0 via governance, but that takes 7 days.

---

## Rationale

**Why irrevocable deposits?**

Allowing withdrawal would make the contract's balance a governance attack vector (e.g. a large depositor who also holds governance power could drain the allocator). Irrevocability eliminates this. Depositors should treat a contribution as a donation to the protocol's bootstrap period.

**Why not route yield on deposited USDC?**

The simplest version of this contract holds plain USDC. A v2 upgrade could deposit idle USDC into sUSDS or Aave to earn yield, reducing the rate at which the balance depletes. This is left to a future RFC to avoid audit scope creep in v1.

**Why a minimum balance alert rather than an automatic top-up?**

An automatic top-up (e.g. via a Zodiac Roles modifier on some external treasury) requires trusting that external treasury's security model. The `LowBalance` event is a signal for off-chain actors to respond; the protocol itself does not assume any external top-up mechanism exists.

---

## Alternatives considered

- **Streaming (Sablier / Superfluid)**: route USDC from a stream into RewardsManager directly. Avoids needing a separate IssuanceAllocator contract, but couples the reward system to a third-party streaming protocol, adding audit surface and a dependency failure mode.
- **RewardsManager holds its own USDC buffer**: simpler, but conflates the disbursement logic with the reward distribution logic. Separating them makes each contract easier to audit and upgrade independently.

---

## Drawbacks

- Irrevocable deposits may deter casual contributors who want optionality. The counterargument is that the protocol needs committed capital, not hot-money deposits.
- No yield on idle USDC. At $20k/week disbursement and a buffer of $100k minimum, the idle balance could be earning ~$2,000–5,000/year in sUSDS. Left to v2.

---

## Security considerations

- **Re-entrancy in `disburse()`**: USDC transfer must follow checks-effects-interactions.
- **Access control on `disburse()`**: the `isAllocationTarget()` check is the sole access control. It must be impossible for a non-target to call `disburse()` even via a delegatecall or proxy pattern.
- **No admin key**: the contract has no `owner`, no `pauser`, no privileged address beyond on-chain governance. Verify at deployment that the deployer key is renounced.
- **USDC blacklist**: if the `IssuanceAllocator` address itself is blacklisted by Circle, it cannot hold or transfer USDC. Vanishingly unlikely, but document as a known risk.

---

## Open questions

- [ ] Should depositors receive a non-transferable ERC-1155 receipt as a record of their contribution (no economic rights, purely informational)?
- [ ] v1.1: yield on idle USDC via sUSDS/Aave. Worth specifying the interface now so v1 storage layout can accommodate it?
- [ ] Should `disburse()` be callable by anyone (with the USDC going to the configured targets) rather than only by the targets themselves? Simpler access model; different attack surface.

---

## Implementation notes

- New contract: `packages/contracts/contracts/rewards/IssuanceAllocator.sol`
- Deploy before `RewardsManager` (RewardsManager constructor takes IssuanceAllocator address)
- Test: deposit, advance 3 epochs, verify disbursement amounts, verify LowBalance events fire at threshold
- Fork test: deploy on Arbitrum Sepolia with live USDC, run a full epoch cycle
