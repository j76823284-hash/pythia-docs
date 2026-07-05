# The DVM (Data Verification Mechanism)

The **Data Verification Mechanism** is Pythia's dispute-resolution backstop. It is a stake-weighted **commit-reveal** voting system implemented in `pythiavoting`. When an optimistic answer is disputed, the oracle escalates a price request to the DVM, and PYTHIA stakers vote on the correct outcome.

## Rounds and phases

Voting is organized into fixed **rounds**. Each round has two phases of equal length (`phase_length`, default **24 hours**, configurable between 30 s and 7 days):

```
│◄────── round N ──────►│◄────── round N+1 ─────►│
│  COMMIT  │  REVEAL    │  COMMIT   │  REVEAL     │
```

Round and phase are derived deterministically from an on-chain anchor time and `phase_length`, so anyone can compute the current round/phase locally. A newly disputed request is queued for the **next** round (`current_round + 1`).

## Commit-reveal voting

Commit-reveal prevents voters from copying each other and prevents last-look manipulation.

### Commit phase — `commitvote`

A voter (or their delegate) submits a **hash** of their intended vote:

```
commit_hash = sha256(
    price(16 bytes, LE) ‖
    salt (16 bytes, LE) ‖
    voter(8 bytes, LE)  ‖
    timestamp(8 bytes, LE) ‖
    round_id(4 bytes, LE)  ‖
    identifier(8 bytes, LE)
)
```

At commit time the contract snapshots the voter's stake weight — this is the **authoritative vote weight**, so adding stake between commit and reveal does not increase influence. On the **first commit of a round**, the contract also freezes the round's `total_stake` in a snapshot (used for the quorum check). Multiple votes can be committed at once with `batchcommit`.

### Reveal phase — `revealvote`

The voter reveals `price` and `salt`. The contract recomputes the commit hash and rejects any mismatch. The revealed vote is tallied at the weight snapshotted during commit. Multiple reveals can be batched with `batchreveal`.

{% hint style="warning" %}
A vote that is committed but never revealed does **not** count and is treated as a missed vote for slashing purposes. Losing your salt means losing your reveal.
{% endhint %}

For disputes escalated from OOv3 assertions (identifier `ASSERT_TRUTH`), reveals are constrained to the binary set: `NUMERICAL_TRUE` (10000) or `NUMERICAL_FALSE` (0).

## Resolution

After a round's reveal phase ends, anyone can call **`processreqs`** (permissionless, batched) to compute results for the previous round. For each request:

1. **Mode.** The most-voted price by weight is computed. To win, the mode must command **more than 50%** of the revealed vote weight. If no price exceeds 50%, the request has no mode.
2. **Quorum checks.** A mode that clears 50% must additionally satisfy two thresholds:
   * **GAT (Global Access Threshold)** — total revealed weight must be at least `gat_percentage` of the frozen round stake (default **5%**). This ensures enough of the network participated.
   * **SPAT (Staker Participation / agreement Threshold)** — agreement on the mode must be at least `spat_bps` (default **50%**). This is measured as mode-weight ÷ revealed-weight.
3. **Resolve or roll.**
   * If mode + GAT + SPAT are all satisfied, the request **resolves** to the mode price. A slashing job is created.
   * Otherwise the request **rolls** to the next round (up to `max_rolls`, default **3**), falling back to the median as a provisional value while it re-votes.

Once resolved, the oracle reads the price via `voting::get_price` / `voting::has_price` and settles the original OOv2 request or OOv3 assertion.

## Anti-manipulation design

| Mechanism | What it prevents |
|---|---|
| **Commit-reveal** | Copying others' votes; last-look manipulation. |
| **Commit-time weight snapshot** | Adding stake after seeing how a vote is going. |
| **Round `total_stake` snapshot** | Flash-loan / just-in-time stake to distort the GAT denominator. |
| **>50% mode + SPAT** | A plurality minority resolving a request. |
| **GAT** | A tiny quorum resolving a high-value dispute. |
| **Rolling + `max_rolls`** | Permanently stuck or low-participation requests. |
| **Slashing** | Voting against the resolved outcome, or not voting at all. |

## Slashing and rewards

Resolution is finalized by two permissionless, batched cranks:

1. **`slashbatch`** — walks the round's vote submissions. Wrong voters are slashed eagerly (capped at their raw stake) and the slashed amount is tallied into a pool; correct voters are parked for payout; every participant is registered as **exempt** from the passive no-vote slash. At the final batch, a passive slash event is opened against non-participants.
2. **`rewardbatch`** — pays the slashed pool to the round's correct voters, pro-rata by weight, and registers their claim on the passive (no-vote) pool.

Non-participating stakers are charged a small **passive slash** (default **0.05%** per resolved request) lazily, the next time they touch their stake. See [Staking, Rewards & Slashing](staking-rewards-and-slashing.md) for the full accounting, including the true-up and pay-out cranks.

## RAM housekeeping

Because Antelope tables cost RAM, the DVM exposes permissionless prune cranks once a round is fully resolved: **`pruneround`** (vote submissions) and **`prunefreqs`** (vote frequencies). See [Keeper & Crank Actions](../developers/keeper-and-crank-actions.md).
