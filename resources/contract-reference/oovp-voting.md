# oovp.voting

The Data Verification Mechanism (DVM). Stake-weighted commit-reveal voting that resolves disputes escalated from `oovp.oracle`.

## Rounds & phases

Round and phase are derived deterministically from an anchor time and `phase_length` (default 24 h). A round = commit phase + reveal phase. New requests are queued for `current_round + 1`.

## Actions

| Action | Auth | Description |
|---|---|---|
| `requestprice(requester, identifier, timestamp, ancillary_data)` | requester (registered) | Queue a dispute for the next round. Called by the oracle on dispute. |
| `commitvote(voter, identifier, timestamp, ancillary_data, hash)` | voter or delegate | Commit a vote hash during the commit phase. |
| `batchcommit(voter, commits[])` | voter or delegate | Up to 50 commits at once. |
| `revealvote(voter, identifier, timestamp, ancillary_data, price, salt)` | voter or delegate | Reveal during the reveal phase. |
| `batchreveal(voter, reveals[])` | voter or delegate | Up to 50 reveals at once. |
| `processreqs(max_requests)` | anyone | Compute results for the previous round; resolve or roll; open slash jobs. |
| `slashbatch(request_id, round_id, max_rows)` | anyone | Slash wrong voters, tally pool, exempt participants; open passive event. |
| `rewardbatch(request_id, round_id, max_rows)` | anyone | Pay the slashed pool to correct voters pro-rata. |
| `pruneround(round_id, max_rows)` | anyone | Reclaim RAM from vote submissions. |
| `prunefreqs(round_id, max_rows)` | anyone | Reclaim RAM from vote frequencies. |

## Admin actions

| Action | Description |
|---|---|
| `setconfig(finder, stake, token)` | Wire the finder, stake, and token contracts. |
| `setphaselen(phase_length)` | Seconds per phase (30 s–7 d; default 86400). |
| `setthreshold(gat_percentage, spat_bps)` | GAT (of 1e8; default 5000000 = 5%) and SPAT (bps; default 5000 = 50%). |
| `setmaxrolls(max_rolls)` | Max times a request rolls (default 3). |

## Commit hash

```
commit_hash = sha256(
    price(16,LE) ‖ salt(16,LE) ‖ voter(8,LE) ‖
    timestamp(8,LE) ‖ round_id(4,LE) ‖ identifier(8,LE)
)
```

For `ASSERT_TRUTH` disputes, `price` must be `NUMERICAL_TRUE` (10000) or `NUMERICAL_FALSE` (0).

## Resolution rules

* **Mode** must exceed **50%** of revealed weight to win.
* **GAT**: revealed weight ≥ `gat_percentage` of the round's frozen total stake.
* **SPAT**: agreement on the mode ≥ `spat_bps`.
* If any check fails, the request **rolls** (median used as provisional value) up to `max_rolls`.

## Anti-manipulation

* Vote weight snapshotted at **commit** time.
* Round `total_stake` frozen on the first commit of the round (`roundsnaps`).
* Unstake cooldown (in `oovp.stake`) prevents borrow-and-return within a round.

## Key tables (readable)

| Table | Contents |
|---|---|
| `pricereqs` | Requests being voted on (round, resolution status). |
| `votesubmits` | Individual commit/reveal submissions per request/round. |
| `votefreqs` | Vote weight per distinct price per request/round. |
| `voteinstance` | Aggregate totals per request/round. |
| `roundsnaps` | Frozen total stake per round. |
| `slashjobs` | Pending slash/reward jobs per request. |
| `votetiming` / `votingconfig` | Singletons: phase length/anchor; thresholds and wired contracts. |

## Static getters

```cpp
bool     voting::has_price(voting, requester, id, ts, anc);
int128_t voting::get_price(voting, requester, id, ts, anc);
uint32_t voting::get_current_round(voting);
uint8_t  voting::get_current_phase(voting);   // 0=COMMIT, 1=REVEAL
```

See [The DVM](../../protocol-overview/the-dvm.md) for the full mechanism.
