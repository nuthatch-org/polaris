# RFC 0010: RewardsEligibilityOracle

| Field | Value |
|-------|-------|
| RFC | 0010 |
| Title | RewardsEligibilityOracle — Permissionless Quality-Gated Reward Eligibility |
| Status | Draft |
| Author | polarity |
| Created | 2026-05-26 |
| Depends on | RFC 0004 (RewardsManager redesign) |
| Closes issue | #10 |

---

## Summary

A new contract, `RewardsEligibilityOracle`, that gates indexer reward eligibility on off-chain quality signals submitted as on-chain EIP-712 attestations. The oracle role is permissionless at the submission level: any address may submit attestations, but only those signed by an authorised oracle signer are accepted. The signer set is updatable via on-chain governance.

---

## Motivation

The original Graph Protocol deployed without a service-quality gate on rewards. The result was that indexers could receive full rewards by staking on a deployment without meaningfully serving queries. GIP-0079 ("Indexer Rewards Eligibility Oracle") was proposed to address this but was never shipped before the Horizon transition.

Polaris ships the oracle from genesis. The rationale:
- Without a quality gate, reward distribution is purely a function of who staked the most, not who served the most queries.
- This creates a passive-income dynamic that is undesirable: large stakeholders extract rewards without providing the service the rewards are meant to incentivise.
- A quality gate aligns rewards with actual network contribution.

The oracle is not a trusted party in any meaningful sense — it cannot steal funds, mint tokens, or change protocol parameters. It can only grant or withhold eligibility for a given indexer on a given deployment. If the oracle misbehaves (withholds eligibility from good actors, grants it to bad ones), the community can rotate the oracle signer set via on-chain governance.

---

## Specification

### Contract interface

```solidity
interface IRewardsEligibilityOracle {

    struct EligibilityAttestation {
        address indexer;
        bytes32 deploymentId;
        bool eligible;
        uint256 validUntil;    // unix timestamp
        uint256 nonce;         // per-indexer-deployment nonce, prevents replay
    }

    /// @notice Submit a signed eligibility attestation.
    ///         Anyone may call this; the signature is verified on-chain.
    /// @param attestation The attestation data.
    /// @param signature   EIP-712 signature from an authorised oracle signer.
    function submitAttestation(
        EligibilityAttestation calldata attestation,
        bytes calldata signature
    ) external;

    /// @notice Check eligibility for a given indexer and deployment.
    ///         Returns false if no valid (non-expired) attestation exists.
    function isEligible(address indexer, bytes32 deploymentId) external view returns (bool);

    /// @notice Add an oracle signer. On-chain governance only; 7-day delay.
    function addSigner(address signer) external onlyGovernance;

    /// @notice Remove an oracle signer. On-chain governance only; 7-day delay.
    function removeSigner(address signer) external onlyGovernance;

    /// @notice Emergency removal of a signer. On-chain governance, no delay.
    ///         Requires supermajority (configurable, default: 2/3 of governance weight).
    function emergencyRemoveSigner(address signer) external onlyGovernance;

    event AttestationSubmitted(address indexed indexer, bytes32 indexed deploymentId, bool eligible, uint256 validUntil);
    event SignerAdded(address indexed signer);
    event SignerRemoved(address indexed signer);
}
```

### EIP-712 domain

```solidity
bytes32 public constant ATTESTATION_TYPEHASH = keccak256(
    "EligibilityAttestation(address indexer,bytes32 deploymentId,bool eligible,uint256 validUntil,uint256 nonce)"
);

bytes32 public DOMAIN_SEPARATOR; // set at initialisation with chain ID and contract address
```

### Eligibility storage

```solidity
struct EligibilityRecord {
    bool eligible;
    uint256 validUntil;
    uint256 nonce;
}

mapping(address indexer => mapping(bytes32 deploymentId => EligibilityRecord)) public eligibility;
```

`isEligible()` returns `eligibility[indexer][deploymentId].eligible && block.timestamp < eligibility[indexer][deploymentId].validUntil`.

Attestations with `validUntil` in the past are rejected at submission time. Attestations with a `nonce` less than or equal to the current stored nonce are rejected (prevents replay of old, now-expired attestations with the same content).

### Attestation validity window

Default: 48 hours (`validUntil = block.timestamp + 48 hours`). Oracle operators are expected to refresh attestations every 24 hours, providing a 24-hour buffer before expiry.

There is no on-chain enforcement of this convention — oracle operators may set any `validUntil` value they choose. The community convention is 48-hour windows; oracles that consistently set shorter windows may be rotated out by governance.

### Genesis oracle signers

At genesis, the oracle signer set is bootstrapped by the deployer (the only trusted action the deployer takes before renouncing keys). The initial signer set should be published publicly with the deployment transaction.

Candidates for genesis signers:
- Indexers running Fisherman monitor (`cargopete/fishing-business` or compatible)
- Any community operator willing to run the oracle service and publish their SLA

### Eligibility criteria (off-chain convention, not hard-coded)

The oracle computes eligibility based on data from:
1. **Gateway query logs**: indexer responded to ≥95% of sampled queries in the 24h window.
2. **Fisherman PoI monitoring**: indexer's latest PoI for the deployment is within 1 block of the canonical.
3. **Dispute status**: no unresolved dispute open >7 days.

These criteria are an off-chain convention. Different oracle operators may use slightly different criteria. The community publishes a reference oracle implementation in `node/`. On-chain, the contract only checks the signature and the `validUntil` timestamp.

---

## Rationale

**Why EIP-712 and not a simple mapping set by a trusted address?**

EIP-712 allows the oracle signer to be a hot key (for operational agility) while the signed data is cryptographically bound to the specific indexer, deployment, eligibility value, and expiry. A replay attack using an old attestation is prevented by the nonce. A simple `mapping(address => bool)` set by a trusted address would require the trusted address to be an on-chain actor (gas cost, latency) rather than an off-chain signer.

**Why permissionless submission?**

Any address can submit a valid attestation (one with a valid oracle signature). This means oracle operators do not need to interact directly with the contract — they can publish signed attestations and let indexers or keepers submit them. It also means oracle operators are not blocked by gas price spikes; anyone who has the signature can pay the gas.

**Why is the signer set updatable via governance rather than hardcoded?**

Oracle operators may become unreliable, be compromised, or leave the community. The governance rotation mechanism (with a 7-day standard delay and a no-delay emergency path) provides a way to respond without a full contract upgrade.

---

## Alternatives considered

- **On-chain PoI verification (no oracle)**: verify PoI correctness entirely on-chain. Not feasible: PoI correctness requires knowledge of the canonical chain state at a past block, which is not available to a contract on Arbitrum One.
- **Chainlink oracle**: use Chainlink's DON for attestations. Adds a third-party dependency, cost, and integration complexity. For a system that needs to attest on dozens of (indexer, deployment) pairs per day, a custom oracle is more practical and cheaper.
- **ZK proof of service quality**: cryptographic proof of query response rates, PoI correctness, etc. Interesting long-term direction but requires significant R&D. Deferred to v3.

---

## Drawbacks

- Oracle operators must run infrastructure and sign attestations every 24 hours. This is a liveness requirement; if all oracle signers go offline, all indexers become ineligible after 48 hours and no rewards are distributed.
- The off-chain criteria are a community convention, not a protocol invariant. Different oracle operators may apply criteria differently, creating inconsistency.

---

## Security considerations

- **Oracle key compromise**: if an oracle signer's key is compromised, the attacker can grant eligibility to ineligible indexers. Emergency signer removal (no delay, supermajority governance) mitigates. The impact is limited to misallocated rewards within the remaining validity window (max 48 hours worth).
- **Replay attacks**: prevented by per-(indexer, deployment) nonce that monotonically increases.
- **Oracle liveness failure**: if all signers go offline, rewards stop after 48 hours. The LowBalance event from IssuanceAllocator and community monitoring should catch this before it persists for a full epoch.
- **Governance capture**: the signer set is controlled by on-chain governance. If governance is captured by a malicious actor, they could add a compromised oracle signer. This is a general governance risk, not specific to this contract.

---

## Open questions

- [ ] Should `isEligible()` have a fallback of `true` (rewards flow if oracle is offline) or `false` (rewards stop if oracle is offline)? Current spec: `false`. The safe default is no rewards to unattested indexers; a liveness failure that stops rewards is more visible and recoverable than one that leaks rewards to undeserving actors.
- [ ] Should the oracle attest on (indexer, deploymentId) or on (indexer) globally? Per-deployment granularity is more precise but requires more attestation volume. Per-indexer is coarser but simpler. Current spec: per-deployment.
- [ ] Reference oracle implementation: should this live in `node/` or a separate `oracle/` directory?

---

## Implementation notes

- New contract: `packages/contracts/contracts/rewards/RewardsEligibilityOracle.sol`
- EIP-712 implementation: use OpenZeppelin's `EIP712` base contract
- `RewardsManager` receives the oracle address at construction (see RFC 0004)
- Test strategy: submit valid attestation, check isEligible; submit expired attestation, check rejected; rotate signer, check old signer no longer valid; run full epoch with mixed eligibility
