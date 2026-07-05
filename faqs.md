---
description: Frequently asked questions about Pythia.
---

# FAQs

Answers to the most common questions about Pythia. See also the [DVM FAQ](protocol-overview/dvm-faq.md) for voting-specific questions and the [Glossary](resources/glossary.md) for definitions.

## General

<details>
<summary>What is Pythia?</summary>

Pythia is an **optimistic oracle** and **dispute-arbitration system** for WAX/Antelope. It lets smart contracts bring arbitrary verifiable data on-chain cheaply, and resolves disputes through a token-weighted human voting layer called the DVM. It is a from-scratch Antelope implementation of the design pioneered by UMA.
</details>

<details>
<summary>What does "optimistic" mean here?</summary>

Answers are accepted **optimistically** — assumed correct after a challenge window (liveness) unless someone disputes them. Only disputed answers go to a vote. This keeps the common case fast and cheap while preserving a decentralized backstop for the rare contested case.
</details>

<details>
<summary>How is Pythia related to UMA?</summary>

Pythia implements the same optimistic-oracle + DVM architecture as UMA, rebuilt in Antelope/C++ for WAX. The concepts map directly (OOv2/OOv3 flows, DVM voting, staking/slashing, governance-through-the-oracle). The differences are all consequences of Antelope's model — accounts and permissions, memo-based token receipt, and permissionless cranks instead of gas-paid keepers.
</details>

<details>
<summary>What is Pythia, versus Foretell and Prism?</summary>

**Pythia** is the protocol layer: oracle, DVM, staking, governance. **Foretell** is a prediction-market application that settles through Pythia. **Prism** is a benchmark / market-data product. These docs are about Pythia; Foretell and Prism are separate products that integrate with it.
</details>

<details>
<summary>Is Pythia live on mainnet?</summary>

Not yet. Pythia is deployed and exercised on the **WAX testnet**. Mainnet is gated on external audit sign-off and multisig hardening of admin keys. See [Audits & Security](resources/audits-and-security.md).
</details>

## For integrators

<details>
<summary>Should I use OOv2 or OOv3?</summary>

Use **OOv2** when you want an open set of third-party proposers answering your requests (and you'll pay a reward). Use **OOv3** when you (or your bot) produce the answer yourself and post the bond in one action. Both share the same dispute path and DVM backstop. See [Optimistic Oracle: v2 and v3](protocol-overview/optimistic-oracle-v2-and-v3.md).
</details>

<details>
<summary>How do I get data out of Pythia?</summary>

Read the settled result via the static getters — `oracle::has_price` / `oracle::get_price` (OOv2) or the `assertions` table (OOv3) — or register a callback (`pricesettled`, `assertresvd`) so your contract is notified inline. Only act on **settled** results.
</details>

<details>
<summary>Why do I transfer tokens with a memo instead of approving?</summary>

Antelope has no ERC-20-style `approve`. To fund a bond or reward you **transfer** the token to the contract with a specific memo (`assert`, `bond:<id>`, `request:<id>`, `proposal:<id>`, `stake`, `rewardpool`). The receiving contract's `on_transfer` handler records the deposit, and the follow-up action consumes it.
</details>

<details>
<summary>What happens if my transaction to settle/resolve isn't sent?</summary>

Nothing breaks — the state just waits. Every settlement/resolution step is a **permissionless crank** that anyone (a keeper, or the counterparty who wants their funds) can push. If one keeper is down, another party can settle. See [Keeper & Crank Actions](developers/keeper-and-crank-actions.md).
</details>

<details>
<summary>How do I size bonds and liveness?</summary>

So that no false resolution is profitable for the value your contract has at risk: the bond a wrong answer forfeits (and the cost of winning a DVM vote you shouldn't win) must exceed what your contract would pay out on a wrong answer. Give honest disputers enough liveness to react. See [Bonds, Fees & Liveness](developers/bonds-fees-and-liveness.md) and [Economic Security](protocol-overview/economic-security.md).
</details>

<details>
<summary>My request/assertion is stuck. How do I recover funds?</summary>

* An OOv2 request with no proposer for 7 days can be `cancel`led (reward refunded).
* Unconsumed deposits are recoverable by their depositor: `reclaimreq`, `reclaimbond`, `reclaimasrt`.
</details>

## For stakers & voters

<details>
<summary>How do I earn as a staker?</summary>

Stake OOVP (transfer with memo `stake`) and vote honestly on disputes. You earn continuous **emissions** plus a share of the stake **slashed** from wrong/absent voters. See the [Staker & Voter Guide](using-pythia/voting-walkthrough/voter-guide.md).
</details>

<details>
<summary>What happens if I vote wrong or don't vote?</summary>

A wrong (or committed-but-unrevealed) vote is slashed 0.1% of your vote weight; not participating at all incurs a smaller 0.05% passive slash per resolved request. Slashed stake is redistributed to correct voters. See [Staking, Rewards & Slashing](protocol-overview/staking-rewards-and-slashing.md).
</details>

<details>
<summary>Can I let someone vote for me?</summary>

Yes — appoint a **delegate** with `setdelegate`. The delegate votes on your behalf; you keep custody, rewards, and slashing exposure. See [Delegated Voting](using-pythia/voting-walkthrough/delegated-voting.md).
</details>

<details>
<summary>Why is unstaking slow?</summary>

Unstaking has a cooldown (default 7 days, two-step: `requnstake` then `execunstake`) so that stake can't be borrowed to swing a vote and returned immediately. Combined with commit-time snapshots, this defeats flash-loan-style manipulation.
</details>

<details>
<summary>Can I boost my voting power?</summary>

Yes — **lock** your stake for a fixed duration to earn up to a 3× voting-power multiplier (`lock`). The boost affects influence, not principal, and slashing is still computed on your raw stake. See [Vote-Locking](using-pythia/voting-walkthrough/vote-locking.md).
</details>

## Token & governance

<details>
<summary>What is the OOVP token used for?</summary>

Staking (vote weight), bonds (oracle and governance), and rewards (emissions + slashing redistribution). It has 4 decimals. "Pythia" is the brand; `OOVP` is the on-chain ticker.
</details>

<details>
<summary>How does governance work?</summary>

A proposal is submitted through `oovp.govern::propose` as an OOv3 oracle assertion. If it settles **true** (undisputed, or upheld by a DVM vote) and the timelock passes, anyone can execute its bundled on-chain actions. Governance thus uses the same security mechanism as the oracle. See [Governance](community/governance/README.md).
</details>

<details>
<summary>What can governance change?</summary>

Nearly every parameter: oracle liveness and burned-bond rate, per-collateral final fees, emission and slash rates, unstake cooldown, DVM phase length, GAT/SPAT thresholds, `max_rolls`, vote-lock parameters, and all finder registries. Sensitive actions are gated behind an `exec` permission only the governor can use.
</details>

<details>
<summary>Is there an emergency path?</summary>

Yes, but constrained: an emergency proposer (a multisig) can bypass the DVM vote only for **pre-allowlisted** targets and only after a minimum delay. This exists for critical fixes without becoming a governance backdoor. See [The Proposal Process](community/governance/the-proposal-process.md).
</details>
