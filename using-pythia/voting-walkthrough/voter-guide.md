# Staker & Voter Guide

This guide takes you from zero to a claimed reward. All actions are on `pythiastake1` (staking) and `pythiavoting` (voting).

## 1. Stake

Stake by transferring PYTHIA to the stake contract with the memo **`stake`**:

```bash
cleos push action <token> transfer \
  '["myvoter","<stake>","100.0000 PYTHIA","stake"]' -p myvoter@active
```

Your staked balance is now your vote weight and immediately begins accruing emissions. Check it:

```bash
cleos get table <stake> <stake> voterstakes --lower myvoter --limit 1
```

## 2. Wait for a request in the current round

Disputes escalate to the DVM and are queued for the next round. List active requests:

```bash
cleos get table <voting> <voting> pricereqs
```

Each row has `identifier`, `timestamp`, `ancillary_data`, and `last_voting_round`. You vote on requests whose `last_voting_round` equals the **current** round (derive round/phase from the `votetiming` singleton, or use a client helper).

## 3. Commit your vote

During the **commit** phase, submit a hash of your vote. The hash binds your vote to your account and the round so it cannot be replayed or copied:

```
commit_hash = sha256(
    price(16, LE) ‖ salt(16, LE) ‖ voter(8, LE) ‖
    timestamp(8, LE) ‖ round_id(4, LE) ‖ identifier(8, LE)
)
```

Pick a **random 128-bit salt** and keep it safe — you need it to reveal.

```bash
cleos push action <voting> commitvote '{
  "voter":"myvoter",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"...",
  "hash":"<32-byte sha256 hex>"
}' -p myvoter@active
```

Vote weight is snapshotted **now**, at commit time. Batch multiple commits with `batchcommit` (max 50).

{% hint style="danger" %}
If you lose your salt you cannot reveal, and an unrevealed commit counts as a **missed vote** (subject to passive slashing). Store `(price, salt)` for every commit until you have revealed it.
{% endhint %}

## 4. Reveal your vote

During the **reveal** phase, reveal `price` and `salt`. The contract recomputes the hash and rejects mismatches:

```bash
cleos push action <voting> revealvote '{
  "voter":"myvoter",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"...",
  "price":6543200000000,
  "salt":839172049182740000000000000000000000
}' -p myvoter@active
```

For OOv3 assertion disputes (identifier `ASSERT_TRUTH`), `price` must be `10000` (true) or `0` (false). Batch reveals with `batchreveal` (max 50).

## 5. Let the round finalize

After the reveal phase, anyone runs the resolution cranks:

```bash
cleos push action <voting> processreqs '{"max_requests":50}' -p keeper@active
cleos push action <voting> slashbatch  '{"request_id":...,"round_id":...,"max_rows":100}' -p keeper@active
cleos push action <voting> rewardbatch '{"request_id":...,"round_id":...,"max_rows":100}' -p keeper@active
```

* If you voted with the resolved majority, your stake is **credited** a pro-rata share of the slashed pool.
* If you voted wrong or never revealed, your stake is **slashed** (`BASE_SLASH_BPS`, 0.1%).
* If you did not participate at all, a passive slash (0.05%) is applied lazily on your next stake interaction.

## 6. Claim (or restake) rewards

Emissions accrue continuously. Withdraw them, or compound:

```bash
# Withdraw accrued PYTHIA emissions
cleos push action <stake> claimreward '{"voter":"myvoter"}' -p myvoter@active

# Or compound them back into stake
cleos push action <stake> restake '{"voter":"myvoter"}' -p myvoter@active
```

## 7. Unstake (two-step, with cooldown)

Unstaking is deliberately slow to prevent vote manipulation:

```bash
# Begin the cooldown (default 7 days)
cleos push action <stake> requnstake '{"voter":"myvoter","amount":"50.0000 PYTHIA"}' -p myvoter@active

# After the cooldown elapses
cleos push action <stake> execunstake '{"voter":"myvoter"}' -p myvoter@active
```

## Best practices

* **Reveal promptly.** Don't wait until the last block of the reveal phase.
* **Automate.** Voting reliably every round is what avoids passive slashing; consider a [delegate](delegated-voting.md) or a bot.
* **Keep salts.** One lost salt per round is one missed vote.
* **Follow the methodology.** Resolve to what the identifier's methodology says the answer _should_ be, not what benefits you — the majority (and slashing) enforce this.
