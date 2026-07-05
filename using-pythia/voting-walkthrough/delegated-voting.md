# Delegated Voting

Voting every round is operational work: watching for requests, committing, keeping salts, and revealing on time. **Delegation** lets you hand that work to another account while keeping full custody of your stake and all of your rewards.

## How it works

A staker appoints a single **delegate** account. The delegate may `commitvote` and `revealvote` (and claim/restake) on the staker's behalf, but:

* **Custody stays with the staker.** The delegate never controls your principal and cannot unstake it.
* **Rewards stay with the staker.** Emissions and slashing redistribution always accrue to the staker, not the delegate.
* **Slashing hits the staker.** If the delegate votes wrong, the staker's stake is slashed — choose a delegate you trust to vote honestly and reliably.

Under the hood, the DVM's `has_voter_or_delegate_auth(voter)` check accepts either the voter's own signature or the registered delegate's signature.

## Set a delegate

```bash
cleos push action <stake> setdelegate '{"voter":"myvoter","delegate":"myrobot"}' -p myvoter@active
```

Now `myrobot` can vote for `myvoter`:

```bash
cleos push action <voting> commitvote '{"voter":"myvoter", ... }' -p myrobot@active
cleos push action <voting> revealvote '{"voter":"myvoter", ... }' -p myrobot@active
```

Notice the **`voter` field stays `myvoter`** — only the signing permission changes to the delegate.

## Remove a delegate

```bash
cleos push action <stake> rmdelegate '{"voter":"myvoter"}' -p myvoter@active
```

## When to delegate

* **You want to stake but not run a voting bot.** Delegate to a service or a trusted operator.
* **You manage many staking accounts.** Point them all at one operational delegate.
* **You want cold-storage custody.** Keep principal in a cold account and delegate voting to a hot key that can only vote.

{% hint style="info" %}
Delegation and [vote-locking](vote-locking.md) compose: a delegate votes with the staker's full (lock-boosted) voting power, and the boost still benefits the staker.
{% endhint %}
