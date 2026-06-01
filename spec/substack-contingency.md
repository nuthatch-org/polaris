# The Graph Needs a Lifeboat. We've Drawn One Up in USDC.

*A contingency plan for a network that has quietly bet its security on the price of its own token.*

---

There is a polite fiction at the centre of The Graph, and it is this: that a decentralised data network can run, indefinitely, on a token it prints itself.

The fiction is comfortable because it has mostly worked. Indexers index, delegators delegate, gateways pay, and the protocol tops everyone up with freshly issued GRT. Nobody has to look too closely at where the money actually comes from. But "mostly worked" and "will keep working" are different claims, and the gap between them is exactly the size of a contingency plan.

So here is one. It's called **Polaris**. It is The Graph's own Horizon contracts — the same staking, the same provisions, the same TAP payment rails — with three substitutions and a great deal subtracted. Stake is **wstETH**. Payments are **USDC**. Rewards are funded by **voluntary USDC deposits** rather than by inflation. There is no native token, no Foundation, no council, and very nearly no governance at all.

It is, in one line: *The Graph, denominated in things that are actually worth something, governed by almost nobody.*

Let me explain why that might matter rather more than it sounds.

## Three problems the network would rather not discuss

**1. The rewards are inflation wearing a high-vis jacket.**

The bulk of what indexers earn is not query fees. It is issuance — new GRT, minted on a schedule, diluting every holder to pay the people securing the network. This is fine right up until it isn't. Indexers have costs denominated in dollars (servers, salaries, electricity that is rude enough to insist on being paid in fiat), so they sell the GRT. That sell pressure is structural and permanent. The protocol's security budget is, in effect, a slow bleed from token holders to operators, intermediated by an emission curve that nobody voted for in any meaningful sense.

A network that pays for its own security by debasing its own savings has not solved the funding problem. It has merely arranged for the bill to arrive quietly.

**2. The collateral is as stable as the weather.**

Stake is the security model. Slash an indexer and you take their stake; the credibility of that threat is the credibility of the network. But GRT is a volatile asset, so the real value of every provision lurches around with the market. The cost of attacking the network on a green day is not the cost on a red one. You are securing a database with a security deposit whose value is set by sentiment on Crypto Twitter. This is not a criticism unique to The Graph — it is the default condition of token-collateralised protocols — but defaults are exactly what a contingency plan exists to question.

**3. The governance is decentralised in the way a buffet is free.**

The Graph has a Foundation, a Council multisig, an off-chain GIP process, and a roadmap stewarded by a small number of very competent people who are, nonetheless, a small number of people. None of this is sinister. Most of it is sensible. But "a multisig of trusted parties can change the protocol" and "credibly neutral, unstoppable infrastructure" are not the same sentence, and the network has been telling itself they are for some time. Every privileged key is a point of failure, capture, or simply error. A genuine contingency plan has to assume the trusted parties are unavailable — that is rather the point of a contingency.

## Why a contingency, specifically

The data doesn't stop mattering if the token has a bad year. Subgraphs still need indexing. Dapps still need to query them. The *useful* part of The Graph — thousands of indexers running real infrastructure against real demand — is entirely separable from the GRT economics layered on top of it.

That separability is the opportunity. If the token economics seize up, if governance is captured, if issuance becomes politically impossible to sustain, the network needs somewhere the actual work can continue without missing a block. Not a competitor. A **lifeboat** — the same crew, the same engine, in a hull that doesn't depend on the thing that's sinking.

Polaris is built to be that hull. It is deliberately the *same software*, because a contingency you have to relearn is a contingency you won't reach for in a crisis.

## What changes, concretely

- **Stake in wstETH.** Collateral is a yield-bearing claim on staked ETH — an asset whose value isn't a referendum on the protocol's own narrative. Indexers provision wstETH; slashing, thawing, and undelegation are all denominated in it. (Raw-ETH deposits via a Lido adapter are on the roadmap; more on the honest limitations below.)
- **Pay in USDC.** Gateways escrow USDC; indexers collect USDC through the unchanged TAP v2 / GraphTally rails. A RAV is now worth a predictable number of actual dollars. Revolutionary, I know.
- **Rewards funded by deposits, not dilution.** There is no `mint()`. An `IssuanceAllocator` contract holds USDC that *anyone* may deposit — no whitelist, no privileged funder — and disburses a fixed weekly budget to indexers, stake-weighted and gated by a quality oracle. The security budget becomes an explicit, visible, opt-in line item rather than an invisible tax on holders.
- **No token. None.** GraphToken, both bridges, the vesting contracts, the L1 deployment — all deleted. You cannot have a token-price problem if you do not have a token. This is the kind of insight that sounds trite until you notice how much architecture it removes.

## "Even more limited governance" is the feature

Here is the part that will annoy people, so I'll say it plainly.

Polaris has *less* governance than The Graph, on purpose. No Foundation. No council. No off-chain process. The deployer key is renounced after launch. The only governance is on-chain, stake-weighted, with a mandatory delay — and even that controls a deliberately small set of parameters.

The instinct in this industry is to treat governance surface as a feature: more knobs, more committees, more "community input." But every knob is something that can be turned against you, and every committee is something that can be captured or simply wander off. A contingency network's job is to be *boring and unstoppable*. The less of it there is to govern, the less there is to break, capture, or argue about at three in the morning when it actually matters.

Minimal governance isn't a shortcut. It's the whole thesis. Credible neutrality is something you achieve by *removing* discretion, not by distributing it more politely.

## The honest bit

A contingency plan that oversells itself is just optimism with extra steps, so:

Polaris today is a **specification and a fork in progress**, not a live network. The contracts are inherited from audited Horizon code, but the modifications — the wstETH plumbing, the redesigned rewards, the USDC treasury — are pre-audit and would need a serious one before anyone stakes a real ETH on them. A few questions are genuinely open: wstETH can't be minted natively on Arbitrum, so raw-ETH deposits wait for v1.1; the rewards-eligibility oracle needs credibly neutral operators from day one or it becomes the very chokepoint we set out to remove; and "fund it with voluntary deposits" only works if someone actually volunteers.

These are real. They are also exactly the kind of problem you want to have *thought through before the emergency*, rather than discovered during it. That is what a contingency plan is for: not to predict the iceberg, but to have already sketched the lifeboat by the time anyone's looking for it.

The Graph may never need Polaris. That would be the best outcome — a fire extinguisher that gathers dust is doing its job. But "we'll be fine, the token will hold, the Foundation will steer" is a strategy only until it isn't, and the people running the actual infrastructure deserve a hull that could float on something firmer than faith.

I'm not publishing code today, and I'm not asking anyone to migrate anything. This is an idea put on the table while the table is still calm: that the useful part of The Graph can be cleanly separated from the token economics layered over it, and that the separation is worth designing *before* it's urgent rather than after.

Disagree with it. Improve it. Or file it away against the day it's needed. A contingency is most valuable precisely when no one thinks they'll have to use it.
