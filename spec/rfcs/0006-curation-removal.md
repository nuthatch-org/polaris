# RFC 0006: Curation Removal — Gateway-Driven Indexing Payments

| Field | Value |
|-------|-------|
| RFC | 0006 |
| Title | Curation Removal — Gateway-Driven Indexing Payments |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | — |
| Closes issue | #6 |

---

## Summary

Delete `Curation.sol` and all bonding-curve infrastructure. Replace with gateway-driven indexing payments following the GIP-0081 pattern: gateways specify which subgraph deployments they pay for, and that signal directs indexer allocation decisions off-chain.

---

## Motivation

Curation's bonding-curve design has been acknowledged as a failure by The Graph Protocol's own GIP process. GIP-0058 ("Replacing Bonding Curves with Indexing Fees"), GIP-0070 ("Paradigm Shift"), and GIP-0081 ("Indexing Payments") collectively amount to a multi-year admission that curation produced the wrong incentives:

- Curators front-ran subgraph publications to capture bonding-curve gains.
- Curation signal correlated with expected reward weights, not with actual query demand.
- The bonding curve's mint/burn mechanics created sell pressure on GRT whenever curators exited.
- Subgraph developers were required to curate their own deployments to ensure they were indexed, adding cost and confusion.

Polaris is a greenfield deployment with no existing curators. There is nothing to migrate. The cleanest path is to delete the curation machinery entirely and replace it with the indexing-payments model that The Graph's own GIP-0081 proposes.

---

## Specification

### Deletions

The following contracts are deleted entirely:

- `Curation.sol` and all associated storage/proxy scaffolding
- `CurationPoolLib.sol` (or equivalent bonding-curve library)
- Any `RewardsManager` hooks that read curation signal (`onSubgraphSignalUpdated`, `rewardsPerSignal`, `getNewRewardsPerSignal`)

### GIP-0081 pattern: gateway-driven indexing payments

In the GIP-0081 model, gateways pay indexers directly for indexing work via the `PaymentsEscrow` / TAP v2 flow. There is no on-chain curation signal. Indexing effort is directed by:

1. **Gateway allowlists**: each gateway maintains an off-chain list of subgraph deployment IDs it is willing to pay to have indexed. Indexers subscribe to gateway allowlists.
2. **Indexing agreements** (optional, future): on-chain agreements between a gateway and an indexer for a specific deployment, with a committed payment rate. This is the direction of GIP-0087/0088 and is a v2 item for Polaris.
3. **Query fee revenue signal**: indexers observe which deployments are generating query fee revenue via `PaymentsEscrow` and self-select accordingly.

This is a fully off-chain signal mechanism. Nothing in the Polaris contracts needs to know which subgraph deployments exist or which have "signal". The contracts simply facilitate payment for work that has already been agreed upon off-chain.

### SubgraphService interaction

`SubgraphService` does not reference `Curation`. It references `RewardsManager` for eligibility checks. After curation removal, `SubgraphService` is unchanged — it does not need to know about the indexing-payment signal mechanism.

### RewardsManager: removal of curation-weighted distribution

See RFC 0004. The `rewardsPerSignal` accumulator and all curation-signal hooks are removed from `RewardsManager`. Rewards are distributed stake-weighted (see RFC 0004), not signal-weighted.

---

## Rationale

Curation's fundamental problem is that it attempted to solve an information problem (which subgraphs need indexing?) with a financial mechanism (bonding curves) in a context where the financial mechanism created perverse incentives. The correct solution to the information problem is direct gateway-to-indexer communication — which is what GIP-0081 specifies and which Polaris adopts from genesis.

---

## Alternatives considered

- **Retain curation but fix the bonding curve**: the bonding curve's incentive problems are structural, not parametric. A flat curve (no front-running advantage) is just an escrow; a steep curve is just a different front-running game. Not worth the code complexity.
- **Replace bonding curves with stake-weighted signal**: indexers signal on deployments proportional to their stake, and rewards flow to indexed deployments. Circular incentive (rewards → stake → signal → more rewards) without external demand validation. Rejected.
- **Ship curation v2 at genesis**: wait until we have a better curation mechanism. Problem: no better mechanism is fully specified. Better to launch clean and add curation v2 when there is real demand data to design against.

---

## Drawbacks

- No on-chain signal for indexers about which subgraphs to index. Off-chain gateway allowlists are less visible and less permissionless than an on-chain curation market.
- Indexers must actively monitor gateway allowlists or rely on gateway operator relationships. This slightly favours established indexers who have gateway relationships.

---

## Security considerations

None material — this is a deletion, not an addition. The reduced contract surface is strictly a security improvement.

---

## Open questions

- [ ] Should Polaris publish a standard for gateway allowlist formats (e.g. a signed JSON blob with deployment IDs and payment rates) to facilitate interoperability?
- [ ] What is the v2 curation design, if any? Defer until post-genesis query fee data provides a signal about which deployments have real demand.

---

## Implementation notes

- Delete: `Curation.sol`, `CurationPoolLib.sol`, any `ICuration` interface
- Modify: `RewardsManager.sol` (remove curation hooks — see RFC 0004)
- Modify: deployment scripts (remove curation deployment step)
- No migration needed (greenfield)
