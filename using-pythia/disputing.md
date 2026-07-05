# Disputing

Disputing is how bad answers get caught. Anyone who believes an optimistic answer is wrong can challenge it within its liveness window by posting a matching bond. The challenge escalates the answer to the DVM, and the side that matches the DVM's resolution wins both bonds (minus the oracle fee).

Disputing is **permissionless and profitable when you are right** — this is what keeps proposers and asserters honest.

## Disputing an OOv2 proposal — `disputeprice`

While a request is in the `PROPOSED` state and liveness has not expired, deposit **bond + final fee** (memo `bond:<request_id>`) and dispute:

```bash
# Deposit the disputer bond + final fee
cleos push action <token> transfer \
  '["disputer1","<oracle>","3.0000 OOVP","bond:1234567890"]' -p disputer1@active

# Dispute the proposal
cleos push action <oracle> disputeprice '{
  "disputer":"disputer1",
  "requester":"myapp",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"..."
}' -p disputer1@active
```

This escalates a stamped request to the DVM. The stamp appended to the ancillary data is:

```
disputed by <requester> against <proposer>
```

If the request had `refund_on_dispute` enabled, the requester's reward is refunded at dispute time.

{% hint style="warning" %}
All funds stay in the oracle until settlement. Bond math and fee payment happen **once**, in `settle` / `settleassert` — never at dispute time.
{% endhint %}

## Disputing an OOv3 assertion — `dispassert`

While an assertion is active and liveness has not expired, deposit a bond equal to the asserter's bond (memo `assert`) and dispute by assertion ID:

```bash
# Deposit the disputer bond
cleos push action <token> transfer \
  '["disputer1","<oracle>","2.0000 OOVP","assert"]' -p disputer1@active

# Dispute the assertion
cleos push action <oracle> dispassert '{"disputer":"disputer1","assertion_id":1234567890}' -p disputer1@active
```

This escalates to the DVM with the OOv3 stamp:

```
assertionId:<hex64>,ooAsserter:<name>
```

and fires an `assertdisp(assertion_id)` callback if the assertion registered a callback recipient.

## What happens after a dispute

1. The dispute is queued for the DVM's **next** round.
2. OOVP stakers vote via commit-reveal (see [The DVM](../protocol-overview/the-dvm.md)). Assertion disputes resolve to `NUMERICAL_TRUE` (asserter was right) or `NUMERICAL_FALSE`.
3. Once resolved, anyone calls `settle` (OOv2) or `settleassert` (OOv3). The oracle reads the DVM result and pays the winner.

## Payouts

Let **B** be the bond and **f** the burned-bond rate (default 50%). The oracle fee is `B × f`.

* **OOv2, disputed:** the winner receives `2×B + own final fee − oracle fee` plus the reward; the fee vault receives the loser's final fee + oracle fee.
* **OOv3, disputed:** the winner receives `2×B − oracle fee` (1.5×B at the default rate); the fee vault receives the oracle fee.

So a correct disputer roughly **doubles** their bond (net of the fee), while a wrong disputer loses it. See [Bonds, Fees & Liveness](../developers/bonds-fees-and-liveness.md) for worked examples.

## Testnet dispute walkthrough

For a hands-on end-to-end dispute (propose → dispute → vote → resolve → settle) on WAX testnet, see [Resolving Disputes on Testnet](../developers/resolving-disputes.md).
