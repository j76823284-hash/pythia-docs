# DVM FAQ

Common questions about Pythia's Data Verification Mechanism (`oovp.voting`) and staking (`oovp.stake`).

### What is the DVM?

The Data Verification Mechanism is Pythia's court of last resort. When an optimistic answer is disputed, OOVP stakers vote on the correct outcome using a two-phase commit-reveal process. See [The DVM](the-dvm.md).

### Who can vote?

Anyone who has staked OOVP into `oovp.stake`. Voting weight equals your staked balance (optionally boosted by a vote-lock multiplier) snapshotted at commit time. You can vote directly, or appoint a **delegate** who votes on your behalf while you keep the rewards. See [Delegated Voting](../using-pythia/voting-walkthrough/delegated-voting.md).

### How long does a dispute take to resolve?

A disputed request is queued for the **next** round and voted on during that round's commit + reveal phases. With the default 24-hour phase length, a round is 48 hours, so resolution typically completes within one to a few days depending on when the dispute lands and whether the request rolls. Phase length is configurable.

### What happens if not enough people vote?

The request fails quorum and **rolls** to the next round, up to `max_rolls` times (default 3). Two thresholds must be met to resolve:

* **GAT** — participation must reach at least `gat_percentage` of the round's frozen total stake (default 5%).
* **SPAT** — agreement on the winning answer must reach at least `spat_bps` (default 50%).

### How is the winning answer chosen?

The most-voted price by weight (the **mode**) wins, but only if it holds **more than 50%** of the revealed vote weight and clears GAT + SPAT. If no answer clears 50%, the request has no mode and rolls; the median is used as a provisional value in the meantime.

### Why commit-reveal instead of just voting?

If votes were public as they were cast, voters could copy the emerging majority ("herding") or a large holder could vote last to swing the result. Committing a hash first, then revealing, hides everyone's vote until the reveal phase, so votes are independent.

### Can someone flash-borrow tokens to swing a vote?

No. Two snapshots block this:

* Your vote weight is snapshotted at **commit** time, so stake added between commit and reveal is ignored.
* The round's **total stake** is frozen on the first commit of the round, so just-in-time stake cannot inflate or deflate the GAT denominator.

Additionally, unstaking is subject to a cooldown (default 7 days), so stake cannot be borrowed and returned within a round.

### What do I earn for voting?

Two things:

1. **Emissions** — staked OOVP continuously accrues protocol emissions (default 0.05 OOVP/sec, shared pro-rata across all stake).
2. **Slashing redistribution** — when you vote with the resolved majority, you receive a pro-rata share of the stake slashed from voters who were wrong or absent.

### What happens if I vote incorrectly or don't vote?

* **Wrong vote:** your stake is slashed by `BASE_SLASH_BPS` (default 0.1%) of your vote weight, capped at your raw stake. The slashed amount is redistributed to correct voters.
* **No vote (passive):** you are charged a smaller passive slash (default 0.05%) per resolved request, applied lazily the next time you interact with your stake.

Slash rates are bounded by `MAX_SLASH_BPS` (10%). See [Staking, Rewards & Slashing](staking-rewards-and-slashing.md).

### How are disputes on OOv3 assertions voted?

Assertion disputes use the `ASSERT_TRUTH` identifier and are **binary**: reveals must be `NUMERICAL_TRUE` (10000, "the assertion was correct") or `NUMERICAL_FALSE` (0). The assertion settles truthful only if the DVM resolves `NUMERICAL_TRUE`.

### Does resolution happen automatically?

No — Antelope has no scheduled/deferred transactions. Resolution is driven by **permissionless cranks** that anyone (typically a keeper) can call: `processreqs`, then `slashbatch`, then `rewardbatch`. Settlement of the underlying request/assertion (`settle` / `settleassert`) is likewise permissionless. See [Keeper & Crank Actions](../developers/keeper-and-crank-actions.md).

### What is "rolling"?

If a request cannot resolve in its round (no supermajority, or quorum not met), it is re-queued into the next round to be voted on again. It can roll up to `max_rolls` times before the provisional median stands.
