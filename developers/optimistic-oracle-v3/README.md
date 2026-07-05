# Optimistic Oracle v3

OOv3 is the **assertion** model: a single party posts a bonded claim, and it is accepted as true unless disputed. It collapses OOv2's request + propose into one action and is the right fit when the integrator itself produces the answer.

This section covers:

* A [Quick Start](quick-start.md) — the simplest possible assertion integration.
* [Data Asserter](data-asserter.md) — a generic framework for asserting arbitrary off-chain data.
* [Escalation Managers](escalation-managers.md) — the pluggable arbitration slot.

## Integration shape

An OOv3 integration typically:

1. **Deposits** the bond to the oracle (memo `assert`).
2. Calls **`asserttruth`** with the claim, currency, bond, liveness, and (optionally) a callback recipient and escalation manager.
3. Implements callbacks to react to disputes and resolution.
4. Reads the settled result from the oracle's `assertions` table, or reacts inside its resolution callback.

## Callbacks

If you pass a `callback_recipient`, the oracle fires inline actions to it. Implement these actions:

```cpp
// Fired when your assertion is disputed (escalated to the DVM).
[[eosio::action]]
void assertdisp(eosio::checksum256 assertion_id);

// Fired when your assertion settles. resolution = true if the assertion held.
[[eosio::action]]
void assertresvd(eosio::checksum256 assertion_id, bool resolution);
```

| Callback | Fired when | Delivery |
|---|---|---|
| `assertdisp` | Someone calls `dispassert` on your assertion. | Inline, during `dispassert`. |
| `assertresvd` | Your assertion settles. | Via the permissionless `notifyassert` action after `settleassert`. |

{% hint style="info" %}
`assertresvd` is delivered by `notifyassert` **after** settlement, so a failing callback can never block settlement of the assertion. Have a keeper call `notifyassert`, or poll the `assertions` table.
{% endhint %}

## Truth encoding

Disputed assertions resolve on the DVM as a **binary** outcome:

| Value | Meaning |
|---|---|
| `NUMERICAL_TRUE` = `10000` | The assertion was correct. |
| `NUMERICAL_FALSE` = `0` | The assertion was incorrect. |

An assertion is `settlement_resolution == true` if it went undisputed through liveness, or the DVM voted `NUMERICAL_TRUE`.

## Default identifier

Assertions default to the built-in `ASSERT_TRUTH` identifier (`0x4153534552545455`) when you pass `identifier = 0`. You can supply a different whitelisted identifier for domain-specific resolution methodologies.
