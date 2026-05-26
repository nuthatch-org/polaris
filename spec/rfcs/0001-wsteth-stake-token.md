# RFC 0001: wstETH as the Polaris Stake Token

| Field | Value |
|-------|-------|
| RFC | 0001 |
| Title | wstETH as the Polaris Stake Token |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0007 (GraphToken deletion) |
| Closes issue | #1 |

---

## Summary

Replace GRT with wstETH as the staking collateral in `HorizonStaking`. Indexers provision wstETH into the staking contract; slashing, undelegation, and thawing are all denominated in wstETH.

---

## Motivation

GRT has depreciated ~99% from its all-time high and trades at ~$0.025–0.027 as of Q1 2026. Denominating security collateral in a depreciating asset means the dollar-value of slashing penalties and indexer skin-in-the-game compresses with every down-quarter, independent of any operational decision by indexers or the protocol.

wstETH is:
- A productive asset (earns Lido staking yield, ~2.5% APR rolling), reducing the opportunity cost of locking collateral
- Denominated in ETH, the canonical security collateral for Ethereum-based systems
- Stable relative to dollar terms (not pegged, but not subject to protocol-specific reflexive sell pressure)
- Audited and battle-tested at scale ($30B+ TVL as of 2025)
- Natively available on Arbitrum One via the canonical bridge

---

## Specification

### HorizonStaking changes

`GraphDirectory` currently resolves `_graphToken` (GRT) as the staking token. This is replaced with `_stakeToken`, a new `IERC20` reference pointing to wstETH on Arbitrum One:

```
wstETH (Arbitrum One): 0x5979D7b546E38E414F7E9822514be443A4800529
```

Affected functions in `HorizonStaking.sol`:

```solidity
// Before
function stake(uint256 tokens) external {
    IGraphToken(graphToken).transferFrom(msg.sender, address(this), tokens);
    ...
}

// After
function stake(uint256 tokens) external {
    IERC20(stakeToken).transferFrom(msg.sender, address(this), tokens);
    ...
}
```

The same substitution applies to `unstake`, `slash`, and any internal accounting that references the staking token address.

No changes to the staking mechanics themselves (provision model, thawing, undelegation queues, slashing percentages).

### GraphDirectory changes

```solidity
// Removed
address internal _graphToken;

// Added
address internal _stakeToken;  // wstETH
address internal _paymentToken; // USDC (see RFC 0003)
```

`_stakeToken` is set at initialisation and is not updatable (on-chain governance cannot change the stake token post-deploy without a full contract upgrade). Rationale: the stake token is a load-bearing security assumption; making it governance-updatable creates an attack vector.

### Storage layout

`HorizonStaking` is deployed behind a proxy. Any storage layout changes must be validated against the existing layout. Since we are replacing a pointer in `GraphDirectory` (which is inherited), the slot position must be checked and a storage gap adjusted if necessary.

This RFC does not specify the exact slot delta — that is a requirement for the implementing MR to verify and document.

### Events

No new events required. Existing `StakeDeposited(address indexed serviceProvider, uint256 tokens)` and related events are token-agnostic.

---

## Rationale

**Why wstETH and not ETH, rETH, or cbETH?**

wstETH is the dominant liquid staking token on Arbitrum One by liquidity and TVL. Using a non-rebasing wrapper (wstETH rather than stETH) avoids accounting complexity inside the staking contract (rebasing tokens require special handling for balances). rETH and cbETH are valid alternatives but have lower Arbitrum One liquidity, complicating indexer onboarding.

**Why not USDC as the stake token?**

Stablecoins as stake collateral are attractive for dollar-stability but don't accrue yield, have depegging risk, and introduce a regulated-asset dependency into the security model. wstETH is the better security collateral; USDC is the better payment rail.

**Why not keep GRT?**

See DESIGN-LOG.md DL-001 and the manifesto for the economic case. Summary: 99% depreciation, no yield, regulatory ambiguity, reflexive sell pressure from indexer liquidations.

---

## Alternatives considered

- **Bare ETH**: avoids wrapped-token complexity but requires special handling for ETH receive/send in Solidity and doesn't accrue staking yield.
- **rETH (Rocket Pool)**: valid choice, but lower Arbitrum One liquidity. Could be added as a whitelisted second stake token post-v1.
- **USDC**: see rationale above.

---

## Drawbacks

- Indexers must acquire wstETH rather than GRT, adding one step to onboarding (bridge wstETH to Arbitrum One, or swap on-chain).
- wstETH price fluctuates in USD terms; the dollar value of minimum provision (0.1 wstETH) changes with ETH price.
- The LidoAdapter (RFC 0002) is a v1.1 item; at genesis, indexers must wrap their own ETH to wstETH off-protocol.

---

## Security considerations

- **Transfer hook risk**: wstETH is a standard ERC-20 with no transfer hooks (no ERC-777, no fee-on-transfer). Confirm before deployment.
- **Slashing denominated in wstETH**: slashed amounts are returned in wstETH. The fisherman receives wstETH, not USDC. This is intentional but must be documented clearly for fisherman operators.
- **Storage layout safety**: the proxy upgrade path must be validated. A slot collision between the new `_stakeToken` field and existing state is a critical risk.
- **Oracle dependency for USD-denominated minimums**: if minimum provision amounts are ever expressed in USD (e.g. "must stake at least $250 worth"), a Chainlink wstETH/USD price feed is required. At genesis, minimum provision is expressed in wstETH directly, avoiding this dependency.

---

## Open questions

- [ ] Should `_stakeToken` be updateable via on-chain governance, or hardcoded? Current position: hardcoded. Arguments for updateability are welcome.
- [ ] Should rETH be whitelisted as a second stake token at genesis, or strictly wstETH-only? Current position: wstETH-only.
- [ ] What is the exact storage slot impact of replacing `_graphToken` with `_stakeToken` in `GraphDirectory`? Must be resolved before implementation can merge.

---

## Implementation notes

- Primary file: `packages/contracts/contracts/staking/HorizonStaking.sol`
- Secondary: `packages/contracts/contracts/discovery/GraphDirectory.sol`
- Test strategy: fork Arbitrum One mainnet, deploy modified staking contract, run full provision/slash/unstake cycle with wstETH
- This RFC must be accepted before RFC 0002 (LidoAdapter) begins, as LidoAdapter wraps this interface
