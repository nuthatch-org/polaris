# RFC 0011: SubgraphService Dispute Deposit Recalibration

| Field | Value |
|-------|-------|
| RFC | 0011 |
| Title | SubgraphService Dispute Deposit Recalibration — GRT → wstETH |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0001 (wstETH stake token) |
| Closes issue | #11 |

---

## Summary

Change the `SubgraphService` dispute deposit from a GRT-denominated amount to a wstETH-denominated amount. Genesis value: 0.005 wstETH. The arbitrator role is permissionless — any address may prove a valid PoI divergence.

---

## Motivation

The dispute deposit deters frivolous disputes. It should be:
1. High enough in dollar terms to disincentivise spam.
2. Low enough not to deter legitimate disputes from small fishermen.
3. Denominated in the same asset as the stake collateral (wstETH), not in a third asset (GRT), to avoid requiring disputers to acquire a separate token.

At current GRT prices (~$0.025), the Horizon mainnet dispute deposit is a small dollar amount. The 0.005 wstETH genesis value (~$12–18 at current ETH prices) is approximately equivalent in dollar terms while being denominated in the staking collateral.

---

## Specification

### Contract change

In `SubgraphService.sol` (or wherever `disputeDeposit` is stored):

```solidity
// Before (Horizon)
uint256 public disputeDeposit; // denominated in GRT

// After (Polaris)
uint256 public disputeDeposit; // denominated in wstETH
```

The variable name and type are unchanged. The token that is transferred when a dispute is opened changes from GRT to wstETH (consistent with RFC 0001 and RFC 0007).

Genesis value: `disputeDeposit = 0.005e18` (0.005 wstETH, where wstETH has 18 decimals).

The deposit is updatable via on-chain governance (7-day delay). No privileged role required.

### Arbitrator role

The arbitrator role in `SubgraphService` is permissionless: any address may call the dispute resolution function if they can provide a valid proof of PoI divergence. There is no designated arbitrator address. This matches the Horizon design intent (disputes are verified on-chain by the contract, not by a human arbitrator).

**Dispute flow** (unchanged from Horizon semantics, only token changed):
1. Fisherman calls `dispute(indexer, deploymentId, theirPoi, canonicalPoi)` with `disputeDeposit` wstETH.
2. Contract verifies the divergence on-chain.
3. If valid: fisherman receives `disputeDeposit` back + the slash reward (a portion of the indexer's slashed wstETH). Indexer's provision is slashed.
4. If invalid: fisherman loses `disputeDeposit`. It goes to a protocol reserve (initially: discarded via burn-equivalent, or sent to address(0) — TBD).

**Note**: "verify the divergence on-chain" means the contract checks that the two PoIs are different and that the canonical PoI is attested by the `RewardsEligibilityOracle` or another on-chain authority. The precise verification mechanism is carried forward from Horizon and is not changed by this RFC.

### Dollar stability of the deposit

The 0.005 wstETH deposit value fluctuates with ETH price. At ETH = $2,000, it is ~$10. At ETH = $5,000, it is ~$25. This is acceptable: the deposit scales roughly with the value of the stake being protected, which also scales with ETH price.

If the community wishes to express the dispute deposit in USD-equivalent terms (e.g. "always worth $15"), an on-chain price oracle (Chainlink wstETH/USD) would be required. This is left to a future RFC. The genesis value is a static wstETH amount.

---

## Rationale

Denominating the dispute deposit in wstETH (the stake token) rather than USDC (the payment token) is intentional: disputes are about stake integrity, not payment flows. The fisherman is acting as a security actor in the staking layer; it is natural that their capital requirement is in the same currency as the stake.

---

## Alternatives considered

- **USDC dispute deposit**: dollar-stable, but requires fishermen to hold both wstETH (for their own staking, if they index) and USDC (for disputes). More token management overhead.
- **No dispute deposit**: removes the spam deterrent. Not recommended.

---

## Security considerations

- **Deposit loss on failed dispute**: fishermen bear real capital risk. At 0.005 wstETH, the risk is not prohibitive for legitimate disputes.
- **Deposit too low to deter spam**: at ~$12–18, a determined attacker could spam dispute calls cheaply if each dispute costs only the deposit (and the attacker expects to lose). In practice, the gas cost of a failed dispute transaction on Arbitrum One adds to the effective cost. The deposit value should be reviewed post-genesis.

---

## Open questions

- [ ] What happens to the deposit of a failed dispute? Options: (a) to the disputed indexer as compensation; (b) to the protocol reserve; (c) burned. Current lean: option (a) — to the indexer, as partial compensation for the operational disruption of an invalid dispute.
- [ ] Should there be a Chainlink wstETH/USD oracle to express the deposit in dollar-equivalent terms? Deferred to post-genesis governance.

---

## Implementation notes

- Primary file: `packages/contracts/contracts/data-service/subgraph/SubgraphService.sol` (or equivalent path)
- Modify: genesis `disputeDeposit` value from GRT amount to 0.005e18 (wstETH)
- Modify: deposit transfer token from GRT to wstETH (consistent with RFC 0001/0007 changes to GraphDirectory)
- Test: open a dispute with 0.005 wstETH deposit; verify slash; verify fisherman reward; verify deposit refund on invalid dispute
