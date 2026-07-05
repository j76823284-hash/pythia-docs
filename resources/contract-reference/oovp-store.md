# oovp.store

Fee collection and the per-collateral **final-fee** schedule. The store is where each collateral's final fee lives (which anchors bond math) and where protocol fees accumulate for burning or treasury distribution.

## Transfer handling

The store has no memo requirement: **any transfer of a token that has a registered final fee is auto-credited** to that token's fee balance. Transfers of tokens without a registered final fee are silently ignored (prevents RAM griefing via dust).

## Actions

| Action | Auth | Description |
|---|---|---|
| `setfinalfee(collateral, fee_amount)` | self | Set the flat final fee for a collateral. |
| `setconfig(fee_recipient, burn_rate_bps)` | self | Set the treasury/burn recipient and burn split (bps; default 5000 = 50%). |
| `setburnable(token, burnable)` | self | Mark whether a collateral implements a holder `burn(name, asset, string)`. |
| `withdraw(to, token, quantity)` | self | Withdraw collected fees (burn or treasury). |
| `prunefees(max_rows)` | anyone | Trim the collected-fee audit log (balances unaffected). |

## Fee model

* **Final fee** — a flat per-collateral amount charged for oracle usage; it sets the minimum bond (`minBond = finalFee × 10000 / burnedBondBps`).
* **Burn split** — `burn_rate_bps` of collected fees are burned (via the token's burn, if `burnable`) and the remainder goes to the treasury recipient. Default 50/50.

## Key tables (readable)

| Table | Contents |
|---|---|
| `finalfees` | Final fee per collateral (indexed by token). |
| `feebalances` | Current collected balance per token (withdrawable/burnable). |
| `collected` | Audit log of collected fees (prunable). |
| `configs` | Singleton: fee recipient, burn rate, regular fee rate. |

## Static getters

```cpp
asset store::get_final_fee(store, collateral);                       // 0 if not configured
asset store::compute_regular_fee(store, collateral, pfc, duration);  // pfc-based regular fee
```

See [Bonds, Fees & Liveness](../../developers/bonds-fees-and-liveness.md) and [Approved Collateral Types](../approved-collateral-types.md).
