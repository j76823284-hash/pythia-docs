# pythiastake1

Staking, delegation, emission rewards, slashing, and vote-locking for the DVM.

## Memo conventions (token transfers to the stake contract)

| Memo | Purpose |
|---|---|
| `stake` | Stake the transferred PYTHIA for the sender. |
| `emission` | Fund emissions — reserved for the configured emissions vault. |

## Staking actions

| Action | Auth | Description |
|---|---|---|
| `requnstake(voter, amount)` | voter | Begin the unstake cooldown. |
| `execunstake(voter)` | anyone | Complete the unstake after cooldown. |
| `claimreward(voter)` | voter or delegate | Withdraw accrued emissions (paid to the staker). |
| `restake(voter)` | voter or delegate | Compound accrued emissions back into stake. |
| `setdelegate(voter, delegate)` | voter | Appoint a voting delegate. |
| `rmdelegate(voter)` | voter | Remove the delegate. |

## Vote-locking actions

| Action | Auth | Description |
|---|---|---|
| `lock(voter, duration_seconds)` | voter | Lock stake for a boost (1×→3× by duration). |
| `extendlock(voter, new_duration_seconds)` | voter | Extend to a longer lock. |
| `refreshlock(voter)` | anyone | Recompute decayed voting power. |

## Slashing / reward internals (called by voting)

| Action | Auth | Description |
|---|---|---|
| `slashnow(voter, amount)` | voting | Eagerly slash an exact amount (phase 0). |
| `credit(voter, amount)` | voting | Credit a correct voter's share (phase 1). |
| `updatereward(voter, correct_vote, slash_amount)` | voting | Update reward/slash accounting. |
| `regexempt(voter, request_id)` | voting | Exempt a participant from the passive slash. |
| `openpassive(request_id, round_id, correct_weight, snapshot_total_stake)` | voting | Open the passive (no-vote) event. |
| `regpassive(voter, request_id, weight)` | voting | Register a correct voter's passive-pool claim. |
| `applyslash(voter, max_traversals)` | anyone | Apply pending active slashes in batches. |
| `trueup(voter, max_events)` | anyone | Apply pending passive events (safety valve). |
| `paypassive(seq, max_rows)` | anyone | Distribute a passive event's pool to correct voters. |
| `closepassive(seq, max_rows)` | anyone | Finalize a passive event after 60 days; bank residue. |

## Admin actions

| Action | Description |
|---|---|
| `setvotingtkn(token_contract, token_symbol)` | Set the PYTHIA token contract + symbol. |
| `setvotingctr(voting_contract)` | Set the DVM voting contract. |
| `setemitvault(vault)` | Set the emissions vault (must grant `stake@eosio.code`). |
| `setemission(rate)` | Emission rate (raw units/sec; default 500 = 0.05 PYTHIA/sec). |
| `setcooldown(seconds)` | Unstake cooldown (1–30 days; default 7 days). |
| `setnovote(bps)` | Passive (no-vote) slash rate (default 5 = 0.05%). |
| `setlockparms(max_lock, min_lock, max_mult_bps)` | Vote-lock schedule. |
| `fundvault(amount)` | One-shot: move emission excess into the emissions vault. |

## Slash constants

| Constant | Default | Meaning |
|---|---|---|
| `BASE_SLASH_BPS` | 10 (0.1%) | Wrong / no-reveal vote, on vote weight (capped at raw stake). |
| `DEFAULT_NO_VOTE_SLASH_BPS` | 5 (0.05%) | Passive no-vote slash, on raw stake, per resolved request. |
| `GOVERNANCE_SLASH_BPS` | 50 (0.5%) | Governance-context slashing. |
| `MAX_SLASH_BPS` | 1000 (10%) | Ceiling for any single slash. |

## Key tables (readable)

| Table | Contents |
|---|---|
| `voterstakes` | Per-voter stake, rewards, delegate, lock, voting power. |
| `stakeconfig` | Singleton: emission rate, cumulative stake, cooldown, wired contracts, lock params. |
| `passevents` / `passclaims` / `nvexempts` | Passive (no-vote) slash accounting. |

## Static getters

```cpp
uint128_t stake::get_stake(stake, voter);       // lock-boosted voting power
uint128_t stake::get_raw_stake(stake, voter);   // raw slashable stake
uint128_t stake::get_total_stake(stake);        // total voting power (or cumulative stake)
name      stake::get_delegate(stake, voter);
```

See [Staking, Rewards & Slashing](../../protocol-overview/staking-rewards-and-slashing.md).
