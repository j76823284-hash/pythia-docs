---
description: Operate settlement and cleanup cranks.
---

# Run DVM maintenance

Use this guide to operate permissionless settlement and cleanup cranks. Run these actions from a monitored keeper or allow any third party to call them.

## Settle pending Oracle work

1. Find expired, undisputed OOv2 requests and call `settle`.
2. Find expired, undisputed assertions and call `settleassert`.
3. For disputed requests or assertions, wait until Voting resolves the corresponding DVM request, then call the appropriate settlement action.
4. Deliver outstanding callbacks with `notifysettled`, `notifydisput`, `notifyassert`, or `notifyadisp` only when the recipient expects them.
5. Deliver credited token payouts with `claimpayout`.

Settlement is permissionless. Do not couple a terminal settlement to a callback or recipient-token notification; Pythia separates them deliberately.

## Advance the DVM

After a reveal phase ends, call `voting::processreqs(max_requests)`. The action accepts at most 100 requests per call. It either resolves eligible requests or rolls them according to the configured limit.

For a resolved round, run these bounded phases until complete:

1. `slashbatch(request_id, round_id, max_rows)` tallies correct votes and slashes incorrect votes.
2. `rewardbatch(request_id, round_id, max_rows)` pays the slash pool to correct voters by voting weight.

Both batch actions require `max_rows` from 1 to 100. Persist progress between transactions; do not assume a large round finishes in one call.

## Prune only terminal data

* `pruneasrt(max_rows, min_age_secs)` removes eligible settled assertions after the minimum retention age.
* `pruneround(round_id, max_rows)` removes vote-submission rows for completed rounds.
* `prunefreqs(round_id, max_rows)` removes vote-frequency rows for completed rounds.

Pruning is permissionless and bounded. Never prune a current or future voting round. Monitor RAM, unresolved work, DVM roll counts, and callback/payout queues before and after every keeper run.
