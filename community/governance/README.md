# Governance

Pythia is governed by **OOVP** stakers. The same DVM that resolves oracle disputes also ratifies protocol changes — a governance proposal is submitted as an oracle **assertion**, and if it settles as _true_ (undisputed, or upheld by a DVM vote), it becomes executable on-chain.

This makes governance and the oracle one and the same security mechanism: to pass a malicious proposal, an attacker would have to win a DVM vote against the honest staker majority, at the cost of being slashed.

## The pieces

| Contract | Role in governance |
|---|---|
| `oovp.govern` | Holds proposals, submits them as assertions, and executes approved ones. |
| `oovp.oracle` | Receives the proposal assertion and settles its truth. |
| `oovp.voting` | Resolves the assertion if it is disputed. |
| `oovp.stake` | Weights the vote and slashes wrong/absent voters. |
| `oovp.token` | The OOVP token: bonds, stake, and rewards. |

## The OOVP token

OOVP is the protocol token. It is used for:

* **Staking** — weight in DVM votes and governance.
* **Bonds** — proposal bonds and (where whitelisted) oracle bonds.
* **Rewards** — emissions to stakers and slashing redistribution.

{% hint style="info" %}
The on-chain token symbol is **OOVP** (4 decimals). "Pythia" is the protocol brand; the ticker and contract identifiers remain `OOVP`/`oovp.*` on-chain. Branding can be migrated independently of contract behavior.
{% endhint %}

## How decisions are made

1. A change is drafted as one or more concrete on-chain actions (e.g. "set the final fee for token X", "update the emission rate").
2. The authorized proposer submits it through `oovp.govern::propose` and funds the bond.
3. The proposal is asserted to the oracle. Stakers can dispute it during its liveness; a disputed proposal goes to a DVM vote.
4. If it settles **true**, anyone can execute it after the timelock. If it settles **false** (voted down), it does not execute and the bond is forfeited.

Read the details in:

* **[The Proposal Process](the-proposal-process.md)** — lifecycle, bonds, timelock, and emergency path.
* **[On-Chain (DAO) Proposals](dao-proposals.md)** — how to construct and submit an executable proposal.

## Parameters governance controls

Nearly every protocol parameter is adjustable through governance-executed actions, including: oracle liveness and burned-bond rate, per-collateral final fees, emission and slash rates, unstake cooldown, DVM phase length, GAT/SPAT thresholds, `max_rolls`, vote-lock parameters, and the finder registries (interfaces, identifiers, collateral). See [Economic Security](../../protocol-overview/economic-security.md) for how these levers keep the oracle safe.
