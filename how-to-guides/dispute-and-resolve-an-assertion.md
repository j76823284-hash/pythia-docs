---
description: Escalate an OOv3 assertion to the DVM.
---

# Dispute and resolve an assertion

Use this guide when an OOv3 claim is wrong, ambiguous, or unsupported by the available evidence.

## 1. Confirm that the assertion is challengeable

Read its `assertions` row. It must exist, be unsettled, have no `disputer`, and be before `expiration_time`. Record the row's `id`, currency, bond, identifier, assertion time, and claim.

## 2. Match the bond deposit

The disputer transfers the assertion's exact bond amount to the Oracle with:

```
assert
```

The deposit must use the assertion's extended symbol. It is held as the disputer's pending assertion deposit until consumed or reclaimed.

## 3. Call `dispassert`

Call `dispassert(disputer, assertion_id)` as the disputer. The Oracle records the dispute and opens the corresponding DVM request itself. Do not open an equivalent DVM request directly; the launch configuration restricts DVM intake to the registered Oracle.

If the assertion configured a callback recipient, use `notifyadisp` separately if callback delivery is required.

## 4. Let the DVM resolve the question

Eligible stakers commit a hash during commit phase and reveal the matching value and salt during reveal phase. Truth assertions use only these numeric outcomes:

```
true  = 10000
false = 0
```

The DVM may roll the request to a later round if it does not meet the configured participation or agreement thresholds.

## 5. Settle the assertion

Once the DVM has a result, anyone calls `settleassert(assertion_id)`.

* A `true` result pays the asserter.
* A `false` result pays the disputer.
* The winner receives both bonds minus the configured Oracle fee.

Settlement writes terminal state before attempting token delivery. Use `claimpayout` for a credited payout and `notifyassert` for a required resolution callback.
