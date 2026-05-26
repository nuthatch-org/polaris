# GitLab Issues — Polaris v1

Paste each section below into a new GitLab issue at https://gitlab.com/polarity1/polaris/-/issues/new.
Issues should be created in order (#1 first) so that cross-references resolve correctly.

Apply labels: `rfc::required` until the RFC MR is merged, then `rfc::accepted`. Use `area::contracts` for contract issues.

---

## Issue #1 — HorizonStaking: replace GRT stake token with wstETH

**Title**: `contracts: replace GRT stake token with wstETH in HorizonStaking`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Replace all references to GRT as the staking collateral in `HorizonStaking` with wstETH (`0x5979D7b546E38E414F7E9822514be443A4800529` on Arbitrum One).

This change is the load-bearing architectural difference between Polaris and Horizon. Indexers provision wstETH; slashing, thawing, and undelegation are all denominated in wstETH. `GraphDirectory` is updated to replace `_graphToken` with `_stakeToken` (wstETH) and `_paymentToken` (USDC).

**RFC**: `spec/rfcs/0001-wsteth-stake-token.md` — this issue is blocked on that RFC being merged.

**Acceptance criteria**:
- [ ] RFC 0001 merged to `main`
- [ ] `HorizonStaking.stake()`, `unstake()`, `slash()` pull/push wstETH, not GRT
- [ ] `GraphDirectory._stakeToken` returns wstETH address
- [ ] Storage layout delta documented and verified (no proxy collisions)
- [ ] Existing Horizon stake/slash/unstake tests ported and passing with wstETH
- [ ] Fork test on Arbitrum One: full provision → query → slash → unstake cycle with live wstETH
- [ ] `DESIGN-LOG.md` DL-001 updated with `Implemented` status

**References**:
- Design log: DL-001
- RFC: `spec/rfcs/0001-wsteth-stake-token.md`
- Depends on: #7 (GraphToken deletion, for `GraphDirectory` changes)
- Blocks: #2 (LidoAdapter)

---

## Issue #2 — LidoAdapter: raw ETH → wstETH deposit wrapper

**Title**: `contracts: LidoAdapter for raw ETH deposits (v1.1)`

**Labels**: `rfc::required`, `area::contracts`, `milestone::v1.1`

**Description**:

Specify and (in v1.1) implement a `LidoAdapter` contract that accepts raw ETH from indexers and converts it to wstETH before staking into `HorizonStaking`.

**Critical constraint**: Lido's `submit()` function is L1-only. wstETH on Arbitrum One is a bridged token; it cannot be minted natively on L2. The RFC must resolve whether a two-phase L1 bridge flow is viable or whether v1 simply requires wstETH-only deposits.

**RFC**: `spec/rfcs/0002-lido-adapter.md` — RFC resolves the L1/L2 constraint and decides the v1 scope.

**Acceptance criteria**:
- [ ] RFC 0002 merged to `main` (resolves the L1/L2 constraint, decides v1 scope)
- [ ] If RFC decides wstETH-only at genesis: operator onboarding docs updated in `node/` with wstETH acquisition instructions; issue closes as `wont-implement` for v1 and reopens for v1.1
- [ ] If RFC decides v1.1 two-phase bridge: separate MR implements `LidoAdapter.sol`; full test suite for the two-phase keeper flow
- [ ] `DESIGN-LOG.md` DL-002 updated

**References**:
- Design log: DL-002
- RFC: `spec/rfcs/0002-lido-adapter.md`
- Depends on: #1 (wstETH stake token)

---

## Issue #3 — GraphPayments: replace GRT payment token with USDC

**Title**: `contracts: replace GRT payment token with USDC in GraphPayments and PaymentsEscrow`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Replace GRT with native USDC (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831`) as the whitelisted payment token in `GraphPayments` and `PaymentsEscrow`. The TAP v2 / GraphTally flow is unchanged; the denomination of RAV values changes from 18-decimal GRT to 6-decimal USDC.

This must be deployed atomically with #1 and #7 as all three touch `GraphDirectory`.

**RFC**: `spec/rfcs/0003-usdc-payment-token.md`

**Acceptance criteria**:
- [ ] RFC 0003 merged to `main`
- [ ] `GraphPayments` whitelist: only USDC at genesis
- [ ] `PaymentsEscrow.deposit()` and `collect()` transfer USDC
- [ ] All test amounts updated for 6-decimal USDC (not 18-decimal GRT)
- [ ] Gateway documentation updated: RAV amounts are now USDC cents (10^-6 USDC)
- [ ] Fork test: gateway escrows USDC → indexer serves queries → RAV submitted → indexer collects USDC
- [ ] `DESIGN-LOG.md` DL-003 updated

**References**:
- Design log: DL-003
- RFC: `spec/rfcs/0003-usdc-payment-token.md`
- Depends on: #7 (GraphToken + GraphDirectory changes)
- Related: #4 (RewardsManager also receives USDC from IssuanceAllocator)

---

## Issue #4 — RewardsManager: remove inflation, add USDC treasury budget

**Title**: `contracts: redesign RewardsManager — remove GRT inflation, add USDC epoch budget`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Remove the GRT `mint()` call and all curation-signal-weighted reward logic from `RewardsManager`. Replace with a 7-day epoch model where USDC is drawn from `IssuanceAllocator` and distributed stake-weighted across eligible provisions.

This is the most significant contract change in Polaris. The RFC specifies the reward distribution formula, epoch mechanics, pull-based claim model, and the oracle eligibility gate.

**RFC**: `spec/rfcs/0004-rewards-manager-redesign.md`

**Acceptance criteria**:
- [ ] RFC 0004 merged to `main`
- [ ] No `GraphToken.mint()` call anywhere in `RewardsManager`
- [ ] `rewardsPerSignal`, curation hooks, and `EpochManager` dependency removed
- [ ] `advanceEpoch()` callable by any address; emits `EpochAdvanced`
- [ ] `disburse()` called on `IssuanceAllocator` at epoch advance
- [ ] Distribution formula: stake-weighted, oracle-eligibility-gated
- [ ] `claim(indexer, deploymentId)` transfers USDC to indexer; pull-based
- [ ] Unit tests: 0-budget epoch, partial-budget epoch, mixed eligibility
- [ ] Fork test: 3 epochs, 5 indexers with varying eligibility, verify claim amounts
- [ ] `DESIGN-LOG.md` DL-004 updated

**References**:
- Design log: DL-004
- RFC: `spec/rfcs/0004-rewards-manager-redesign.md`
- Depends on: #5 (IssuanceAllocator), #10 (RewardsEligibilityOracle), #6 (curation removal), #9 (EpochManager decoupling)

---

## Issue #5 — IssuanceAllocator: permissionless USDC disbursement contract

**Title**: `contracts: new IssuanceAllocator — permissionless USDC deposits, epoch disbursement`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Implement a new `IssuanceAllocator` contract. It accepts permissionless USDC deposits (no whitelist, no minimum) and disburses a configured weekly amount to `RewardsManager` on epoch tick. Deposits are irrevocable. The weekly budget is configurable via on-chain governance.

This is the funding mechanism that replaces GRT inflation. Anyone can deposit USDC to fund indexer rewards.

**RFC**: `spec/rfcs/0005-issuance-allocator.md`

**Acceptance criteria**:
- [ ] RFC 0005 merged to `main`
- [ ] `deposit(uint256 amount)` callable by any address; no minimum; emits `Deposited`
- [ ] No `withdraw()` function; deposits are irrevocable
- [ ] `disburse()` callable only by registered allocation targets
- [ ] `LowBalance` event fires when balance < `minimumBalance`
- [ ] `setAllocationTargets()` and `setMinimumBalance()` gated behind on-chain governance (7-day delay)
- [ ] No admin key; deployer renounced post-deploy
- [ ] Genesis config: `weeklyAmount = 20_000e6` to `RewardsManager`; `minimumBalance = 100_000e6`
- [ ] Unit tests: deposit, 3-epoch disburse cycle, balance depletion behaviour, LowBalance events
- [ ] `DESIGN-LOG.md` DL-005 updated

**References**:
- Design log: DL-005
- RFC: `spec/rfcs/0005-issuance-allocator.md`
- Blocks: #4 (RewardsManager depends on IssuanceAllocator)

---

## Issue #6 — Curation: delete bonding-curve infrastructure

**Title**: `contracts: delete Curation.sol and bonding-curve infrastructure`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Delete `Curation.sol`, `CurationPoolLib.sol`, and all associated bonding-curve infrastructure. Remove curation-signal hooks from `RewardsManager`. Replace indexing signal with gateway-driven indexing payments (GIP-0081 pattern) operating entirely off-chain.

Polaris is greenfield; there are no existing curators to migrate.

**RFC**: `spec/rfcs/0006-curation-removal.md`

**Acceptance criteria**:
- [ ] RFC 0006 merged to `main`
- [ ] `Curation.sol` and `CurationPoolLib.sol` deleted
- [ ] No `ICuration` references remain in any contract
- [ ] `RewardsManager` curation hooks removed (coordinate with #4)
- [ ] Deployment script removes curation deployment step
- [ ] `forge build` compiles cleanly after deletion
- [ ] `DESIGN-LOG.md` DL-006 updated

**References**:
- Design log: DL-006
- RFC: `spec/rfcs/0006-curation-removal.md`
- Related: #4 (curation hooks removed from RewardsManager)

---

## Issue #7 — GraphToken: delete token and bridge contracts

**Title**: `contracts: delete GraphToken, L1/L2 gateways, BridgeEscrow; update GraphDirectory`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Delete `GraphToken.sol`, `L1GraphTokenGateway.sol`, `L2GraphTokenGateway.sol`, `BridgeEscrow.sol`, and all associated proxy and initialisation scaffolding.

Update `GraphDirectory` to remove `_graphToken` and add `_stakeToken` (wstETH) and `_paymentToken` (USDC). Storage layout must be verified to avoid proxy collisions.

This RFC is a prerequisite for #1, #3, and #4 and should be the first contract MR to merge (or merged atomically with #1).

**RFC**: `spec/rfcs/0007-graphtoken-deletion.md`

**Acceptance criteria**:
- [ ] RFC 0007 merged to `main`
- [ ] `GraphToken.sol`, `L1GraphTokenGateway.sol`, `L2GraphTokenGateway.sol`, `BridgeEscrow.sol` deleted
- [ ] All `IGraphToken`/`graphToken()`/`_graphToken` references resolved across all contracts
- [ ] `GraphDirectory._stakeToken` and `._paymentToken` added; `._graphToken` removed
- [ ] Storage layout delta documented in MR; no proxy collisions
- [ ] `forge build` compiles cleanly
- [ ] `DESIGN-LOG.md` DL-007 updated

**References**:
- Design log: DL-007
- RFC: `spec/rfcs/0007-graphtoken-deletion.md`
- Blocks: #1, #3, #4

---

## Issue #8 — TokenLock: delete vesting contracts

**Title**: `contracts: delete TokenLockWallet and TokenLockManager`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Delete `TokenLockWallet.sol` and `TokenLockManager.sol`. No token to vest. No migration needed.

**RFC**: `spec/rfcs/0008-vesting-deletion.md`

**Acceptance criteria**:
- [ ] RFC 0008 merged to `main`
- [ ] `TokenLockWallet.sol` and `TokenLockManager.sol` deleted
- [ ] No references remain in other contracts or deployment scripts
- [ ] `forge build` compiles cleanly
- [ ] `DESIGN-LOG.md` DL-008 updated

**References**:
- Design log: DL-008
- RFC: `spec/rfcs/0008-vesting-deletion.md`
- Can be merged alongside #7

---

## Issue #9 — EpochManager: decouple from RewardsManager

**Title**: `contracts: remove EpochManager as hard dependency of RewardsManager`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Remove `EpochManager` as a hard dependency of `RewardsManager`. Epoch tracking for reward distribution moves into `IssuanceAllocator` (7-day wall-clock epochs). `EpochManager` is retained as a standalone deployable utility but is not part of the core deployment.

Remove `_epochManager` from `GraphDirectory` (coordinate storage layout with #7).

**RFC**: `spec/rfcs/0009-epoch-manager-decoupling.md`

**Acceptance criteria**:
- [ ] RFC 0009 merged to `main`
- [ ] `RewardsManager` does not call `IEpochManager.currentEpoch()` or any EpochManager function
- [ ] `GraphDirectory._epochManager` removed; storage gap adjusted
- [ ] `EpochManager.sol` retained but not deployed in core deployment script
- [ ] Can be bundled with the #4 (RewardsManager) implementation MR
- [ ] `DESIGN-LOG.md` DL-009 updated

**References**:
- Design log: DL-009
- RFC: `spec/rfcs/0009-epoch-manager-decoupling.md`
- Depends on: #4 (RewardsManager redesign), #7 (GraphDirectory changes)

---

## Issue #10 — RewardsEligibilityOracle: quality-gated reward eligibility from genesis

**Title**: `contracts: new RewardsEligibilityOracle — EIP-712 attestations, permissionless submission`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Implement a new `RewardsEligibilityOracle` contract. Any address may submit EIP-712 signed attestations; only attestations from authorised oracle signers are accepted. The signer set is updatable via on-chain governance (7-day standard delay; no-delay emergency path).

`RewardsManager` calls `isEligible(indexer, deploymentId)` before including a provision in reward distribution. Attestations expire after 48 hours if not renewed. This ships from genesis — not as a future upgrade.

**RFC**: `spec/rfcs/0010-rewards-eligibility-oracle.md`

**Acceptance criteria**:
- [ ] RFC 0010 merged to `main`
- [ ] `submitAttestation()` callable by any address; signature verified on-chain; replay-protected by nonce
- [ ] `isEligible()` returns `false` for expired attestations and unattested (indexer, deployment) pairs
- [ ] `addSigner()` / `removeSigner()` gated by on-chain governance (7-day delay)
- [ ] `emergencyRemoveSigner()` gated by on-chain governance (no delay, supermajority)
- [ ] EIP-712 domain separator correctly includes chain ID and contract address
- [ ] Unit tests: valid attestation, expired attestation, wrong signer, nonce replay, signer rotation
- [ ] Integration test: full epoch with oracle-eligible and oracle-ineligible indexers; verify reward split
- [ ] Reference oracle implementation published in `node/oracle/`
- [ ] `DESIGN-LOG.md` DL-010 updated

**References**:
- Design log: DL-010
- RFC: `spec/rfcs/0010-rewards-eligibility-oracle.md`
- Blocks: #4 (RewardsManager calls `isEligible()`)

---

## Issue #11 — SubgraphService: recalibrate dispute deposit for wstETH

**Title**: `contracts: recalibrate SubgraphService disputeDeposit from GRT to wstETH`

**Labels**: `rfc::required`, `area::contracts`

**Description**:

Change the dispute deposit token from GRT to wstETH (consistent with the stake token change in #1). Genesis value: 0.005 wstETH (~$12–18 at current ETH prices). Value is updatable via on-chain governance.

The arbitrator role is permissionless — any address may prove a PoI divergence on-chain.

**RFC**: `spec/rfcs/0011-dispute-deposit-recalibration.md`

**Acceptance criteria**:
- [ ] RFC 0011 merged to `main`
- [ ] Dispute deposit token changed to wstETH; genesis value `0.005e18`
- [ ] Dispute deposit updatable via on-chain governance (7-day delay)
- [ ] Arbitrator role permissionless (no designated address)
- [ ] Unit tests: valid dispute (fisherman wins), invalid dispute (fisherman loses deposit), deposit returned on valid dispute
- [ ] Fork test: open dispute with 0.005 wstETH; verify slash; verify fisherman reward
- [ ] `DESIGN-LOG.md` DL-011 updated

**References**:
- Design log: DL-011
- RFC: `spec/rfcs/0011-dispute-deposit-recalibration.md`
- Depends on: #1 (wstETH as stake/deposit token)
