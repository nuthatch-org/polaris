# RFC 0002: LidoAdapter — Raw ETH Deposit Wrapper

| Field | Value |
|-------|-------|
| RFC | 0002 |
| Title | LidoAdapter — Raw ETH Deposit Wrapper |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0001 (wstETH stake token) |
| Closes issue | #2 |

---

## Summary

A thin stateless wrapper contract that accepts raw ETH from indexers and converts it to wstETH before staking into `HorizonStaking`. This is a UX convenience — the core staking contract only accepts wstETH directly; this adapter handles the wrap for operators who prefer to deposit ETH.

---

## Motivation

Requiring indexers to manually acquire wstETH (swap on-chain or use Lido's web interface) before they can stake adds friction to onboarding. A LidoAdapter reduces this to a single transaction, improving UX without adding complexity to the core staking contract.

---

## Specification

### Constraint: no native Lido on Arbitrum One

Lido's `submit()` function — which accepts ETH and mints stETH — is an L1-only call. wstETH on Arbitrum One is a bridged representation of L1 wstETH, not natively mintable on L2.

This means the "accept ETH → Lido.submit() → wrap → stake" flow does not work on Arbitrum One without an L1 round-trip.

**Two options:**

**Option A — wstETH-only at genesis (recommended for v1)**

Do not ship LidoAdapter at genesis. Document clearly that operators must bring their own wstETH. The adapter is a v1.1 deliverable.

Pros: no new attack surface, no cross-chain complexity, shorter audit scope.
Cons: marginally worse UX for operators who want to deposit raw ETH.

**Option B — LidoAdapter via L1 cross-chain deposit**

A more complex adapter that:
1. Receives ETH on Arbitrum One
2. Bridges ETH to L1 via Arbitrum's canonical bridge
3. Calls Lido `submit()` on L1
4. Bridges resulting wstETH back to Arbitrum One
5. Calls `HorizonStaking.stake()` on behalf of the original caller

This requires a two-phase, two-transaction flow (bridge latency is 15–60 minutes each way) and a keeper bot to complete the L1-side step. This is a meaningfully complex system with a non-trivial attack surface.

**This RFC recommends Option A for v1. Option B is documented here for completeness and deferred to a future RFC.**

### Option A: wstETH-only at genesis

No new contract. The adapter slot in the architecture is reserved; the contract is not deployed.

Documentation in `node/` and the indexer onboarding guide should include the three-step setup:
1. Acquire ETH
2. Wrap to wstETH via Lido's web UI or `wstETH.wrap(stETH)` / `stETH.submit()` on L1, then bridge to Arbitrum One via the canonical bridge
3. Call `HorizonStaking.stake(amount)`

Or more conveniently, swap directly to wstETH on Arbitrum One via Uniswap or Curve (wstETH/WETH pool exists on Arbitrum One with deep liquidity).

### Option B: LidoAdapter (v1.1 placeholder interface)

```solidity
interface ILidoAdapter {
    /// @notice Accepts ETH, converts to wstETH via Lido on L1, and stakes into HorizonStaking.
    /// @dev Two-phase: this call initiates the bridge. A keeper completes on L1.
    ///      Caller receives a receipt NFT redeemable once the round-trip completes.
    /// @param staker The address to credit the stake to in HorizonStaking.
    function depositETH(address staker) external payable returns (uint256 receiptId);

    /// @notice Called by keeper after L1-side completion. Completes the stake.
    /// @param receiptId The receipt ID from depositETH.
    function completeDeposit(uint256 receiptId) external;
}
```

This interface is not implemented in v1. It is specified here so that the storage layout and any cross-contract dependencies can be planned without blocking v1.

---

## Rationale

The Option A / Option B split exists because the naive implementation ("accept ETH, call Lido.submit()") simply doesn't work on Arbitrum One. The two real options are "don't do it in v1" and "do it properly with a two-phase bridge flow in v1.1". Attempting a half-measure (e.g. using a DEX swap instead of Lido) would mean the adapter is not actually a Lido adapter — it's a swap wrapper, with different security properties and no yield accrual between deposit and wrap.

---

## Alternatives considered

- **DEX swap wrapper**: accept ETH, swap to wstETH via Uniswap on Arbitrum One, stake. Simpler than the L1 bridge path, but the indexer gets wstETH at spot price (no Lido discount) and the contract has DEX price-impact/slippage exposure. Deferred: if shipped, it should be a separate "SwapAndStake" contract, not the LidoAdapter.

---

## Drawbacks

- v1 leaves a small UX friction for operators who want to start from raw ETH. In practice, most institutional operators already hold wstETH or have easy access to it.

---

## Security considerations

- Option A has no new attack surface.
- Option B (v1.1): the two-phase bridge flow introduces keeper trust (who can complete the deposit on behalf of the user?), bridge failure modes (what happens if the L1 Lido submit succeeds but the wstETH bridge back fails?), and receipt NFT security (can someone steal a receipt and claim someone else's stake?). These must be fully specified in the v1.1 RFC before implementation.

---

## Open questions

- [ ] Is there an Arbitrum One native path to mint wstETH without an L1 round-trip that we have missed?
- [ ] Is a DEX-swap wrapper (SwapAndStake) worth shipping in v1 for UX, as a separate contract from LidoAdapter?

---

## Implementation notes

- v1: no contract to implement. Update `node/` docs and operator onboarding guide with wstETH acquisition instructions.
- v1.1: design the two-phase bridge keeper system in a new RFC before implementation.
