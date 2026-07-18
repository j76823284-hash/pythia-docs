---
description: Proposed governance and security-token policy.
---

# $PYORA tokenomics

$PYORA is the proposed governance and security token for the Pythia Oracle and DVM. Its purpose is to make manipulation of disputed outcomes costly and to give protocol participants a governed way to fund security, maintenance, and growth.

This is a proposed economic policy, not a token sale, return promise, or a statement of deployed supply. The current contracts and configuration still use the technical `OOVP`/`oovp.*` names. A deployment may use `$PYORA` only after the token symbol, supply, issuer, and distribution controls are finalized and deployed.

## What $PYORA is for

| Protocol role        | Why the token matters                                                                                       |
| -------------------- | ----------------------------------------------------------------------------------------------------------- |
| DVM stake            | Stake weights commit/reveal votes and provides slashable economic exposure.                                 |
| Delegation and locks | Holders can delegate vote signing; time locks can increase voting power under the configured staking rules. |
| Oracle bonds         | When $PYORA is an enabled collateral, asserters and disputers can lock it behind a claim.                   |
| Governance bonds     | Proposal bonds make governance spam costly.                                                                 |
| Fees and rewards     | A configured final fee, reward pool, and Store burn policy can connect protocol use to the token economy.   |

It is not equity, a revenue share, a buyback promise, or a guaranteed-yield product.

## Proposed supply policy

The starting proposal is a fixed supply of `1,000,000,000.0000 PYORA` with four decimals. Issue the full approved supply only after allocation custody, vesting controls, and the issuer authority are ready.

| Allocation                       |      Amount | Share | Initial policy                                               |
| -------------------------------- | ----------: | ----: | ------------------------------------------------------------ |
| DVM security and staking rewards | 220,000,000 |   22% | Reward reserve, voter bootstrap, keeper/disputer incentives. |
| Ecosystem and integrations       | 180,000,000 |   18% | Grants for Oracle users, tooling, audits, and builders.      |
| Community distribution           | 150,000,000 |   15% | Staged distribution after security milestones.               |
| Treasury and operations          | 180,000,000 |   18% | Multisig-held operating and audit budget.                    |
| Core contributors                | 150,000,000 |   15% | Subject to a long vesting schedule.                          |
| Strategic partners               |  80,000,000 |    8% | Subject to a long vesting schedule.                          |
| Liquidity operations             |  40,000,000 |    4% | Disclosed, limited market operations.                        |

The proposed initial policy float is `90,000,000 PYORA` (9%). These allocations are policy only until a vesting/distribution mechanism makes their restrictions enforceable.

## What the current contracts enforce

`oovp.token` can create, issue, transfer, retire, and burn a standard Antelope asset. It does not itself provide a pause, blacklist, vesting schedule, or transfer lock. `oovp.stake` pays rewards from a pre-funded balance; it does not mint them.

Therefore, do not call any balance locked, vested, or non-circulating unless an on-chain mechanism makes it so. Until then, describe it as custody policy and publish the controlling accounts and authorities.

## Security before emissions

The recommended canary emission rate is zero. Begin rewards only after a real disputed assertion completes successfully and voter participation is observable.

| Stage                  | Proposed annual emissions | Raw `setemission` rate for four decimals |
| ---------------------- | ------------------------: | ---------------------------------------: |
| Canary                 |                        0% |                                      `0` |
| First operating period |                     0.25% |                   `793` raw units/second |
| Year 2                 |                     0.50% |                  `1585` raw units/second |
| Year 3                 |                     0.75% |                  `2378` raw units/second |
| Mature operation       |             0.50% or less |                           `1585` or less |

At four decimals, `10,000` raw units equal `1.0000 PYORA`. Any emission increase should be accompanied by a published security rationale, current voter participation, reward-pool runway, and governance approval.

## DVM parameters are economic policy

The starting operational target is a 24-hour commit phase, a 24-hour reveal phase, a 10–15% Global Access Threshold (GAT), a 65% Staker Participation Threshold (SPAT), and at most four rolls. These are proposals, not safe defaults for every deployment.

GAT uses a `100000000` scale: `10000000` means 10%, `15000000` means 15%, and `50000000` means 50%. SPAT is in basis points, so `6500` means 65%.

Before accepting meaningful value, measure independent voters, delegate concentration, real reveal rates, and the cost for a disputer to obtain a bond. A protocol with high nominal stake but no active voters is not secure.

## Bond and fee economics

For a disputed OOv3 assertion with bond `B`, the asserter and disputer each post `B`. At `burned_bond_bps = 5000`, the winner receives `1.5B` and the Oracle charges `0.5B` as its fee. An undisputed assertion simply returns the asserter's bond.

The minimum OOv3 bond is:

```
minimum bond = final fee × 10,000 / burned bond bps
```

At 5,000 basis points, the minimum is twice the final fee. That minimum protects protocol mechanics; it is not necessarily sufficient for the value at risk. Integrators must choose a larger bond and longer liveness for material or ambiguous claims.

## Governance guardrails

The token should govern slowly and visibly. The current policy proposal is a `10,000 PYORA` governance proposal bond, a 72-hour execution delay, multisig-only emergency authority, and explicit governance approval for material tokenomics changes.

Track these public metrics before relaxing any limit:

* independent active voters and reveal participation;
* largest delegate and top-five delegate voting-power share;
* reward-pool runway and effective emissions;
* unresolved value relative to independent active stake; and
* dispute outcomes, roll frequency, and honest-disputer liquidity.

## Launch sequence

1. Complete an Oracle/DVM canary, including an actual disputed assertion.
2. Finalize the symbol, issuer, max supply, custody accounts, and vesting implementation.
3. Issue the approved supply into disclosed custody buckets and harden issuer authority.
4. Fund the staking reward pool but begin with zero emissions.
5. Bootstrap independent DVM voters before expanding collateral, identifiers, or value at risk.
6. Publish supply, custody, stake, delegation, and governance dashboards before public distribution.

Do not treat this sequence as a substitute for legal, tax, security, or jurisdiction-specific review.
