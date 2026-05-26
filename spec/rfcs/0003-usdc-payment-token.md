# RFC 0003: USDC as the Polaris Payment Token

| Field | Value |
|-------|-------|
| RFC | 0003 |
| Title | USDC as the Polaris Payment Token |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0007 (GraphToken deletion) |
| Closes issue | #3 |

---

## Summary

Replace GRT with USDC as the payment token in `GraphPayments` and `PaymentsEscrow`. Gateways escrow USDC; indexers receive USDC for query fees; the TAP v2 RAV flow is unchanged. Native USDC on Arbitrum One (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831`) is the sole whitelisted payment token at genesis.

---

## Motivation

Query fee payment in GRT requires gateways and consumers to (a) acquire GRT, (b) hold a volatile asset during the escrow period, and (c) account for GRT price risk between query execution and settlement. For institutional consumers — the segment Polaris explicitly targets — this is a blocker, not a friction. Treasury teams, regulated dapps, and stablecoin-native applications require predictable, dollar-denominated infrastructure costs.

USDC is:
- The dominant stablecoin on Arbitrum One by liquidity
- Natively issued by Circle on Arbitrum One (not a bridged representation)
- Accepted by every major DeFi and CeFi platform
- Familiar to institutional treasury teams
- Fully dollar-redeemable on demand (within Circle's T+1 redemption window)

---

## Specification

### Whitelisted payment tokens

`GraphPayments` maintains a mapping `mapping(address => bool) paymentTokenWhitelist`. At genesis:

```
USDC (Arbitrum One native): 0xaf88d065e77c8cC2239327C5EDb3A432268e5831  → true
```

All other tokens: `false`.

The whitelist is mutable via on-chain governance (7-day delay). No address has a unilateral right to add or remove tokens.

### USDC.e vs native USDC

This RFC specifies **native USDC** (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831`), not USDC.e (`0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8`).

USDC.e is a bridged representation of L1 USDC managed by Arbitrum's canonical bridge. Circle introduced native USDC on Arbitrum One in June 2023; native USDC supports Circle's Cross-Chain Transfer Protocol (CCTP) and is the preferred form for institutional users. USDC.e is being progressively deprecated by Circle.

Gateways that hold USDC.e must convert before escrowing. This is a one-time migration cost; the ongoing operational model targets native USDC.

### GraphPayments changes

```solidity
// Before (Horizon)
address public constant PAYMENT_TOKEN = address(GRT);

// After (Polaris)
// PAYMENT_TOKEN is not a constant; it is initialised at deploy time from GraphDirectory._paymentToken
// and is not updateable post-deploy (whitelisted tokens are updatable, but the primary settlement token is fixed)
```

The `collect()` function in `GraphPayments` pulls from `PaymentsEscrow` and transfers USDC to the indexer's nominated address. No change to the function signature.

### PaymentsEscrow changes

`deposit()` and `collect()` reference the payment token via `GraphDirectory._paymentToken`. Token is changed to USDC. No other semantic changes.

### TAP v2 / GraphTally

`GraphTallyCollector` is token-agnostic at the verifier level. The RAV (Receipt Aggregate Voucher) contains a value denominated in the payment token's native units (for USDC: 6 decimal places). RAV values are now USDC cents (10^-6 USDC), not GRT units.

RAV signing and verification are unchanged. The change is purely in the token that is transferred when `PaymentsEscrow.collect()` is called.

**Note for gateway operators**: USDC has 6 decimals. GRT has 18 decimals. Any gateway software that constructs RAV amounts must be updated to use USDC's 6-decimal precision. This is an off-chain change, not a contract change, but it must be synchronised with the contract deployment.

---

## Rationale

**Why USDC and not USDT, DAI, or another stablecoin?**

USDC has the deepest Arbitrum One liquidity of any stablecoin and is Circle-regulated with a well-understood reserve audit cadence. USDT is also viable but has historically lower institutional acceptance and no Arbitrum One native issuance. DAI (now USDS) is decentralised but has lower liquidity on Arbitrum One. The whitelist mechanism allows community governance to add additional stablecoins post-v1 without a contract upgrade.

**Why not allow GRT as a whitelisted token alongside USDC?**

The design goal is to make Polaris GRT-free for institutional consumers. Allowing GRT as a payment option re-introduces the token-acquisition friction and creates a two-tier UX. Post-v1, if the community decides to whitelist GRT, it can do so via governance.

---

## Alternatives considered

- **USDT**: viable. Defer to post-v1 governance whitelist addition.
- **USDS / sUSDS**: interesting for yield on escrowed funds, but less familiar to institutional treasury teams at this stage.
- **wstETH as payment token**: natural pairing with the stake token, but introduces ETH price volatility into query fee accounting. Not appropriate for institutional consumers.

---

## Drawbacks

- Gateway operators must migrate from GRT-denominated escrow to USDC-denominated escrow. This is a meaningful operational change for any gateway that already runs on Horizon.
- RAV amounts denominated in 6-decimal USDC rather than 18-decimal GRT. Gateway software must be updated. Risk of bugs during migration.

---

## Security considerations

- **No fee-on-transfer**: confirm USDC on Arbitrum One has no transfer fee. Circle's native USDC does not; USDC.e does not either. This must be re-confirmed if additional tokens are whitelisted post-v1.
- **Decimals handling**: 6 vs 18 decimal mismatch with upstream assumptions in the codebase. Every arithmetic operation involving token amounts must be audited for decimal correctness.
- **USDC blacklist**: Circle maintains a blacklist of addresses that cannot transfer USDC. If an indexer's payment address is blacklisted, `collect()` will revert. The protocol should handle this gracefully (allow the indexer to nominate an alternative payment address).

---

## Open questions

- [ ] Should USDT be whitelisted at genesis alongside USDC, or deferred to governance?
- [ ] How should the protocol handle a USDC Circle blacklist event on an indexer's address?
- [ ] Should escrowed USDC earn yield (e.g. deposited into sUSDS) while awaiting settlement? This would require a more complex escrow design — defer to v2.

---

## Implementation notes

- Primary files: `GraphPayments.sol`, `PaymentsEscrow.sol`, `GraphDirectory.sol`
- Secondary: update all tests that assume 18-decimal GRT token amounts to use 6-decimal USDC amounts
- Gateway software (off-chain): RAV amount construction must be updated separately from the contract work
- Must be deployed atomically with RFC 0001 (stakeToken) and RFC 0007 (GraphToken deletion) since `GraphDirectory` is modified by all three
