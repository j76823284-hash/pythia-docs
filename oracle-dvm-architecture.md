---
description: Oracle flows, DVM voting, trust boundaries, and governance.
---

# Oracle/DVM architecture

Pythia turns a question that a contract cannot answer by itself into a result that a contract can read on-chain. It uses optimistic settlement for routine work and a stake-weighted DVM only when someone disputes the answer.

This is an explanation of the model, not a deployment guide. For action names and fields, use the [API reference](oracle-dvm-api-reference.md).

## The protocol boundary

| Contract      | Responsibility                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| `oovp.oracle` | Accepts value requests and truth assertions, holds economic deposits, and records final Oracle state. |
| `oovp.voting` | Queues disputed questions, runs commit/reveal rounds, and records DVM results.                        |
| `oovp.stake`  | Supplies stake, delegation, voting power, and slashable exposure.                                     |
| `oovp.finder` | Resolves the configured implementation addresses and whitelists.                                      |
| `oovp.store`  | Holds final-fee configuration and receives configured Oracle fees.                                    |

The Finder separates a protocol role from an account name. Integrations should resolve configured implementations instead of assuming fixed production account names.

## Two Oracle entry points

### OOv2 value request

Use this flow for a numeric value whose scale and meaning are defined by its identifier and ancillary data.

```
requester deposit → requestprice → proposer deposit → proposeprice
                                               │
                              no dispute ──────┼────── dispute
                                               │
                                          settle│      DVM → settle
```

The requester escrows a reward. A proposer and, if necessary, a disputer each escrow the required bond and final fee. The Oracle does not pay either side when the dispute is opened; it waits for settlement so economic state has one terminal decision point.

### OOv3 truth assertion

Use this flow for a claim that resolves to true or false.

```
asserter deposit → asserttruth → liveness ──→ settleassert(true)
                                  │
                                  └─ dispute → DVM → settleassert(true|false)
```

The asserter and disputer each post the same bond. An undisputed assertion returns the asserter's bond after liveness. A disputed assertion pays the winning side from both bonds minus the configured Oracle fee.

The assertion's compact `id` is its table key. The Oracle also stores the full SHA-256 assertion ID, so a truncated-key collision fails rather than merging two claims.

## What the DVM decides

The DVM receives only disputed Oracle questions. Its normal intake is the Oracle itself, which prevents another registered contract from pre-creating a matching request.

1. The Oracle creates a DVM request for the next voting round.
2. A staker or its delegate commits a hash of its answer and secret salt during commit phase.
3. The same account reveals the answer and salt during reveal phase.
4. `processreqs` checks the configured participation and agreement conditions. It either records a result or rolls the request to another round within the configured limit.
5. The Oracle reads that result and completes settlement.

For truth assertions, only `10000` (true) and `0` (false) are valid revealed values. Numeric requests may use the application-defined integer value.

## Stake, snapshots, and delegation

Vote weight is taken when a voter commits. The DVM freezes total stake on the first commit in a round, then uses that snapshot for threshold checks. This stops a stake movement after the round begins from changing the quorum denominator.

Delegation changes who may sign a vote; it does not remove the original staker's economic exposure. Operators should monitor both raw stake and delegated voting-power concentration.

## Finality without callback risk

Settlement updates the terminal Oracle row before token delivery or an application callback. Payouts are recorded as credits and callbacks are retried through separate actions. A failing recipient contract therefore cannot revert a valid price or assertion result.

This separation creates operational work: keepers must watch pending payout and callback rows as well as unresolved requests.

## Bounded work and retention

Antelope CPU and RAM are protocol constraints. Pythia uses bounded, permissionless actions for result processing, slash/reward distribution, and pruning. A keeper should repeat small batches instead of attempting an unbounded sweep.

Terminal assertion rows and completed-round voting data can be pruned after their retention conditions are met. Active and future-round vote data must remain intact.

## Trust assumptions

Pythia does not make external facts intrinsically true. Its security depends on:

* at least one capable watcher challenging a false optimistic answer before liveness expires;
* enough independent stake participating to meet DVM thresholds when a dispute occurs;
* bonds, liveness, fees, and value-at-risk limits that make honest disputes economically viable; and
* correct Finder, token, and permission configuration.

Use longer liveness and larger bonds as the ambiguity or value at risk increases.
