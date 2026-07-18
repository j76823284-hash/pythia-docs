---
description: A hands-on OOv3 assertion lifecycle.
---

# Make and settle your first truth assertion

This tutorial is for developers learning Pythia. You will make an undisputed OOv3 truth assertion and settle it after its liveness period.

## What you need

* An Oracle contract with a configured Finder, Store, and Voting contract.
* A whitelisted collateral currency and identifier, followed by `syncparams` for that pair.
* An account with the chosen collateral token and permission to sign for the asserter.
* An ABI-capable Antelope client. Claims are `vector<char>` values, so encode text as UTF-8 bytes; in JavaScript, use `Array.from(Buffer.from(claim, 'utf8'))`.

For a local example, see the OOv3 assertion scenario in [`tests/integration/full-oracle-flow.test.js`](file:///tests/integration/full-oracle-flow.test.js).

## 1. Choose a claim and bond

Use a claim that another person can verify, such as `The test event completed at 12:00 UTC.` Choose a configured collateral currency and a bond at least as large as `oracle::getminbond(currency)`.

Keep these values together. An assertion is tied to its claim, asserter, block time, currency, identifier, and bond.

## 2. Deposit the bond

Transfer the bond from the asserter to the Oracle contract with exactly this memo:

```
assert
```

The transfer creates or extends the asserter's pending assertion deposit. A different token contract, token symbol, or insufficient amount is rejected. The deposit does not create an assertion by itself.

## 3. Submit `asserttruth`

Call `asserttruth` as the asserter with:

```
asserter            the account that made the deposit
claim               UTF-8 claim bytes
callback_recipient  empty name when no callback is needed
escalation_manager  empty name to use the DVM
liveness            challenge window in seconds
currency            the deposited extended symbol
bond                the deposited amount to consume
identifier          0 for the default truth identifier, or a configured identifier
domain_id           0 when no escalation-game domain is needed
```

The Oracle consumes the pending deposit atomically and writes a row to `assertions`. Record its `id`; it is the primary key used by `dispassert` and `settleassert`.

## 4. Wait through liveness

Until `expiration_time`, anyone may challenge the claim. For this tutorial, do nothing. A valid, undisputed assertion is not settled automatically: anyone may crank it once its liveness period has ended.

## 5. Settle the assertion

Call `settleassert` with the assertion row's `id`. This action is permissionless.

Read the row again. Success means:

* `settled` is `true`;
* `settlement_resolution` is `true`; and
* the asserter has a credited payout. Delivery is separate from terminal state, so use `claimpayout` if a payout remains pending.

## Next steps

To use the Oracle for numeric data, follow [Request and settle a value](file:///9782450/how-to/request-and-settle-a-value.md). To learn the contested path, follow [Dispute and resolve an assertion](file:///9782450/how-to/dispute-and-resolve-an-assertion.md).
