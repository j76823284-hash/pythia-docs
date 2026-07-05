# Voting Walkthrough

Stakers are the backbone of Pythia's security. By staking OOVP and voting honestly on disputes, you earn emissions and a share of slashed stake — and you keep the oracle trustworthy.

This section walks through everything a staker/voter needs:

| | |
|---|---|
| **[Staker & Voter Guide](voter-guide.md)** | Stake, commit, reveal, and claim rewards, step by step. |
| **[Delegated Voting](delegated-voting.md)** | Let someone else do the operational voting while you keep custody and rewards. |
| **[Vote-Locking](vote-locking.md)** | Lock stake for up to a 3× voting-power boost. |

## The voter lifecycle at a glance

```
stake OOVP ──► (dispute escalates to DVM) ──► commit vote ──► reveal vote ──► earn / get slashed ──► claim
      ▲                                                                                           │
      └───────────────────────────── restake or unstake (after cooldown) ◄───────────────────────┘
```

1. **Stake** OOVP into `oovp.stake` (transfer with memo `stake`). Your balance is your vote weight.
2. When a dispute is queued, **commit** a hashed vote during the commit phase.
3. **Reveal** your vote during the reveal phase.
4. After the round is processed, correct voters are **rewarded** from the slashed pool; wrong/absent voters are **slashed**.
5. **Claim** accrued emissions any time, or **restake** to compound.

{% hint style="warning" %}
There is no scheduler on Antelope. Rounds advance in real time, but slashing/rewards are finalized by permissionless cranks (`processreqs`, `slashbatch`, `rewardbatch`). If you run a voter, either run these cranks yourself or rely on a keeper. See [Keeper & Crank Actions](../../developers/keeper-and-crank-actions.md).
{% endhint %}

Start with the **[Staker & Voter Guide](voter-guide.md)**.
