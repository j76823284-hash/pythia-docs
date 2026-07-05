# Vote-Locking for Boosted Power

Stakers can **lock** their stake for a fixed duration to earn a voting-power multiplier — a vote-escrow ("veToken") model. Locking rewards long-term aligned stakers with more say in dispute and governance outcomes.

## The multiplier

Locking scales your **voting power** linearly with lock duration:

| Lock duration | Multiplier |
|---|---|
| None (unlocked) | 1× (`BASE_MULTIPLIER_BPS` = 10000) |
| Partial | interpolated linearly |
| Maximum (`MAX_LOCK_DURATION`, 2 years) | 3× (`MAX_LOCK_MULTIPLIER_BPS` = 30000) |

Lock duration is bounded between `MIN_LOCK_DURATION` (**1 week**) and `MAX_LOCK_DURATION` (**2 years**). The multiplier decays as the lock approaches expiry, so voting power falls back toward 1× over time.

{% hint style="info" %}
The boost applies to **influence**, not principal. It increases `voting_power` (your weight in the DVM and the round-stake totals), while **slashing is computed on your raw stake** — so a boost never lets a single slash remove more than your actual staked balance.
{% endhint %}

## Lock your stake

```bash
# Lock all of myvoter's stake for 1 year
cleos push action <stake> lock '{"voter":"myvoter","duration_seconds":31536000}' -p myvoter@active
```

Your cached `voting_power` is updated to `stake × multiplier`, and the contract's `total_voting_power` is adjusted accordingly.

## Extend a lock

A lock can only be extended to a **longer** remaining duration:

```bash
cleos push action <stake> extendlock '{"voter":"myvoter","new_duration_seconds":63072000}' -p myvoter@active
```

## Refresh decayed power

As a lock decays, cached voting power drifts above the correct value until refreshed. Anyone can refresh a voter (permissionless), which keeps the round-stake totals honest:

```bash
cleos push action <stake> refreshlock '{"voter":"myvoter"}' -p keeper@active
```

## Interaction with unstaking

A lock is a commitment: your stake's voting power is boosted in exchange for keeping it staked. Plan your unstake cooldown around the lock — the boost is only meaningful while the stake remains committed.

## Governance parameters

The lock schedule is configurable by governance via `setlockparms(max_lock, min_lock, max_mult_bps)`, so the exact bounds and maximum multiplier reflect the current on-chain configuration. The defaults above are the protocol constants.
