# Keeper & Crank Actions

Antelope has **no deferred or scheduled transactions**. Every step that would be "automatic" on other chains is instead a **permissionless action** that anyone can call. In production these are driven by a **keeper** — a small bot that watches state and pushes the right crank at the right time.

This page lists every crank, what it does, and how to sequence them.

## Why cranks exist

Because the chain can't call a contract for you, Pythia exposes settlement, resolution, slashing, and RAM-reclaim as explicit actions. They are:

* **Permissionless** — no special authority required (you only pay CPU/NET).
* **Bounded** — each takes a `max_rows`/`max_requests` bound so it never exceeds a transaction's resource limits. Call repeatedly until done.
* **Idempotent-safe** — they no-op or revert cleanly when there is nothing to do.

## Oracle cranks (`pythiaoorcle`)

| Action | What it does | When |
|---|---|---|
| `settle(...)` | Settle an OOv2 request (undisputed or DVM-resolved). | After liveness, or after the DVM resolves a dispute. |
| `settleassert(assertion_id)` | Settle an OOv3 assertion. | After liveness, or after DVM resolution. |
| `notifysettled(...)` | Deliver the OOv2 `pricesettled` callback. | After `settle`, if callbacks were enabled. |
| `notifyassert(assertion_id)` | Deliver the OOv3 `assertresvd` callback. | After `settleassert`, if a callback recipient was set. |
| `syncparams(identifier, currency)` | Refresh the whitelist/final-fee cache. | After changing collateral or identifiers in finder/store. |
| `pruneasrt(max_rows, min_age_secs)` | Erase terminal settled assertions to reclaim RAM. | Periodically; `min_age_secs` ≥ 300 s. |

## DVM cranks (`pythiavoting`)

Run these **in order** after a round's reveal phase ends:

| Order | Action | What it does |
|---|---|---|
| 1 | `processreqs(max_requests)` | Compute results for the previous round; resolve or roll each request; open slash jobs. |
| 2 | `slashbatch(request_id, round_id, max_rows)` | Slash wrong voters, tally the pool, exempt participants; open the passive event on the final batch. |
| 3 | `rewardbatch(request_id, round_id, max_rows)` | Pay the slashed pool to correct voters pro-rata; register passive claims. |
| 4 | `pruneround(round_id, max_rows)` | Reclaim RAM from vote submissions. |
| 4 | `prunefreqs(round_id, max_rows)` | Reclaim RAM from vote frequencies. |

`processreqs` must be called during the **commit** phase of the following round (i.e. between rounds). `slashbatch` must fully complete (phase flips to 1) before `rewardbatch`.

## Stake cranks (`pythiastake1`)

| Action | What it does | When |
|---|---|---|
| `execunstake(voter)` | Complete an unstake after cooldown. | After `requnstake` cooldown elapses. |
| `applyslash(voter, max_traversals)` | Apply pending active slashes to a voter in batches. | If a voter accrued many pending slashes. |
| `trueup(voter, max_events)` | Apply pending passive (no-vote) slash events. | Safety valve for voters far behind. |
| `paypassive(seq, max_rows)` | Distribute a passive event's pool to correct voters. | As passive charges accumulate. |
| `closepassive(seq, max_rows)` | Finalize a passive event and bank the residue. | After `PASSIVE_CLOSE_MIN_AGE` (60 days). |
| `refreshlock(voter)` | Recompute a decaying vote-lock's cached power. | Periodically for locked voters. |

## Store crank (`pythiastore1`)

| Action | What it does |
|---|---|
| `prunefees(max_rows)` | Trim the collected-fee audit log to reclaim RAM (balances are unaffected). |

## A minimal keeper loop

A production keeper typically runs a loop like:

```
every N seconds:
  # settle finished optimistic answers
  for each expired assertion:      settleassert(id); notifyassert(id)
  for each expired OOv2 request:   settle(...);       notifysettled(...)

  # advance the DVM between rounds
  if in commit phase and previous round unprocessed:
      processreqs(50)
      for each open slash job:     slashbatch(...); then rewardbatch(...)

  # housekeeping
  occasionally: pruneasrt, pruneround, prunefreqs, prunefees, paypassive
```

{% hint style="info" %}
Keepers should run with a **low-privilege, linkauth-scoped** hot key that can only call these crank actions and cannot move funds. See the keeper role in the [Deployment & Setup Checklist](../resources/deployment-checklist.md).
{% endhint %}

Because every crank is permissionless, the protocol is **liveness-robust**: if one keeper goes down, anyone — including the counterparties who want their funds settled — can push the cranks.
