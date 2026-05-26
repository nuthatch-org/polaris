# RFC 0007: GraphToken and Bridge Deletion

| Field | Value |
|-------|-------|
| RFC | 0007 |
| Title | GraphToken and Bridge Deletion |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | — |
| Closes issue | #7 |

---

## Summary

Delete `GraphToken.sol`, `L1GraphTokenGateway.sol`, `L2GraphTokenGateway.sol`, `BridgeEscrow.sol`, and all associated proxy and initialisation scaffolding. Update `GraphDirectory` to remove `_graphToken` and add `_stakeToken` (wstETH) and `_paymentToken` (USDC).

---

## Motivation

Polaris has no native token. GRT serves no function in the protocol. The token contracts and bridge infrastructure are dead weight that expands the attack surface and the audit scope for no benefit.

---

## Specification

### Deleted contracts

- `GraphToken.sol` (ERC-20 + mint/burn + governor role)
- `L1GraphTokenGateway.sol`
- `L2GraphTokenGateway.sol`
- `BridgeEscrow.sol`
- Any `IGraphToken` interface files
- Token deployment and bridge initialisation scripts

### GraphDirectory changes

```solidity
// Removed
address internal _graphToken;

// Added
address internal _stakeToken;   // wstETH: 0x5979D7b546E38E414F7E9822514be443A4800529
address internal _paymentToken; // USDC:   0xaf88d065e77c8cC2239327C5EDb3A432268e5831
```

`_stakeToken` and `_paymentToken` are set at initialisation time and are not individually updatable via setter functions. A contract upgrade (via `GraphProxyAdmin`) is required to change them. This is intentional: both addresses are load-bearing security assumptions.

### Storage layout

`GraphDirectory` is a base contract inherited by `HorizonStaking` and `GraphPayments`. Removing `_graphToken` (one `address` slot, 20 bytes) and adding `_stakeToken` + `_paymentToken` (two `address` slots) changes the storage layout.

The implementing MR must:
1. Map the exact storage slots of all `GraphDirectory` fields in the current upstream layout.
2. Confirm that replacing one slot with two slots (or packing them) does not collide with any downstream storage.
3. Document the final slot assignments in the MR description.

Using a storage gap (`uint256[N] __gap`) in `GraphDirectory` is the recommended way to accommodate this without collisions.

### Cascade: IGraphToken references

Search all contracts for `IGraphToken`, `graphToken()`, `_graphToken`, and `GraphToken` and remove or replace:

- `HorizonStaking`: references to graphToken for stake → replaced by stakeToken (RFC 0001)
- `RewardsManager`: references to graphToken for mint → removed (RFC 0004)
- `SubgraphService`: any residual graphToken references → audit and remove

---

## Rationale

Straightforward. No token, no token contracts. The only non-trivial part is the storage layout delta in `GraphDirectory`, which must be handled carefully to avoid proxy storage collisions.

---

## Drawbacks

None. Reduced surface area is strictly better.

---

## Security considerations

- **Storage layout collisions**: the primary risk. See specification above. Must be verified before the MR can merge.
- **Cascade audit**: any contract that previously called `IGraphToken.mint()` or `IGraphToken.burn()` will fail to compile after deletion. The implementing MR must ensure all references are resolved, not just suppressed.

---

## Open questions

- [ ] Should `_stakeToken` and `_paymentToken` share a single storage slot (address packing) or occupy separate slots? Separate slots are cleaner and are recommended.

---

## Implementation notes

- This RFC is a prerequisite for RFC 0001, RFC 0003, and RFC 0004. It should be the first contract-layer MR to merge (or merged atomically with RFC 0001).
- After deletion, run `forge build` and ensure zero compile errors before any other RFC implementation begins.
- Grep target: `IGraphToken\|graphToken\|GraphToken` across all `.sol` files.
