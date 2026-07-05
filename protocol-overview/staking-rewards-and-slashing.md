# Staking, Rewards & Slashing

Pythia's economic security comes from staked **PYTHIA**. Stakers vote in the DVM, earn emissions, and are slashed for voting incorrectly or not at all. Staking is handled by `pythiastake1`.

## Staking

Stake by transferring PYTHIA to `pythiastake1` with the memo **`stake`**. Your balance accrues rewards immediately and counts as vote weight in the DVM.

* **Unstaking is two-step.** Call `requnstake(voter, amount)` to begin a cooldown, then `execunstake(voter)` after it elapses. The cooldown (default **7 days**, bounded 1–30 days) prevents borrowing stake to swing a vote and returning it immediately.
* **Rewards vs. principal.** Emissions are paid from a dedicated **emissions vault** wallet, never from stakers' principal. Slashing redistribution moves stake between stakers without leaving the contract.

## Rewards (emissions)

Staked PYTHIA continuously accrues protocol emissions using a standard reward-per-token accumulator:

* **Emission rate** — default **0.05 PYTHIA/sec** (≈1.5M PYTHIA/year), configurable via `setemission` (governance) up to a hard cap.
* **Pro-rata** — emissions are shared across all stake in proportion to each staker's balance.
* **Claiming** — `claimreward(voter)` withdraws accrued PYTHIA; `restake(voter)` compounds it back into stake. Both can be called by the voter or their delegate; rewards always go to the staker.

## Vote-locking (veToken boost)

Stakers can **lock** their stake for a fixed duration to earn a voting-power multiplier, mirroring a vote-escrow model:

* `lock(voter, duration_seconds)` — lock for a duration between `MIN_LOCK_DURATION` (1 week) and `MAX_LOCK_DURATION` (2 years).
* Multiplier scales linearly from **1×** (no lock) up to **3×** (`MAX_LOCK_MULTIPLIER_BPS`) at maximum duration.
* `extendlock` lengthens a lock; `refreshlock` (permissionless) recomputes cached voting power as a lock decays.

The multiplier affects **voting power** (influence in the DVM), not your principal. See [Vote-Locking](../using-pythia/voting-walkthrough/vote-locking.md).

{% hint style="info" %}
Slashing math uses your **raw stake**, not your lock-boosted voting power. The boost increases your say in outcomes without increasing the amount that can be removed by a single slash beyond your actual stake.
{% endhint %}

## Slashing

When a DVM request resolves, `pythiavoting` finalizes it in two permissionless, batched phases that call into `pythiastake1`:

### Active slashing — wrong voters

`slashbatch` walks every vote submission for the resolved round:

* A voter whose revealed price matches the resolved price is **correct**.
* Any other voter (wrong answer, or committed-but-never-revealed) is slashed by `BASE_SLASH_BPS` (**0.1%**) of their vote weight, **capped at their raw stake**. The eager slash (`slashnow`) removes the stake immediately and adds it to a pool.

`rewardbatch` then pays that pool to the round's correct voters, pro-rata by weight (`credit`). The system is token-conserving: the total credited equals the total slashed.

### Passive slashing — absent stakers

Stakers who did not participate at all in a resolved request are charged a **passive (no-vote) slash**, default `DEFAULT_NO_VOTE_SLASH_BPS` = **0.05%** of raw stake per resolved request:

* At the `slashbatch` phase flip, `pythiavoting` opens a **passive event** for the request. Every participant is pre-registered as exempt.
* Non-participants are charged **lazily** — the next time they touch their stake (claim, stake more, unstake, etc.), pending passive events are applied via an internal true-up. This avoids iterating the whole staker set on-chain.
* A safety valve, `trueup(voter, max_events)`, lets anyone catch a voter up if they fall too far behind.
* Collected passive charges are redistributed to the round's correct voters via `paypassive`; after `PASSIVE_CLOSE_MIN_AGE` (60 days) `closepassive` finalizes the event and banks any undistributed residue.

### Slash rate summary

| Event | Constant | Default | Applies to |
|---|---|---|---|
| Wrong / no-reveal vote | `BASE_SLASH_BPS` | 0.1% | Vote weight (capped at raw stake) |
| No participation | `DEFAULT_NO_VOTE_SLASH_BPS` | 0.05% | Raw stake, per resolved request |
| Governance context | `GOVERNANCE_SLASH_BPS` | 0.5% | (Governance-related slashing) |
| Absolute ceiling | `MAX_SLASH_BPS` | 10% | Any single slash |

## Delegation

A staker can appoint a **delegate** with `setdelegate(voter, delegate)` (remove with `rmdelegate`). The delegate can `commitvote`/`revealvote` and claim on the staker's behalf, but **rewards and principal stay with the staker**. Delegation lets you outsource the operational work of voting without surrendering custody. See [Delegated Voting](../using-pythia/voting-walkthrough/delegated-voting.md).

## Why this matters

Rewards make honest voting profitable in expectation; slashing makes dishonest or lazy voting costly. Together they align stakers with reporting the truth, which is exactly the behavior the oracle needs. See [Economic Security](economic-security.md) for how these combine into the cost-of-corruption guarantee.
