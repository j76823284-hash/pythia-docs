# Optimistic Oracle v2

The OOv2 flow is the **request → propose → dispute → settle** model. Your contract opens a data request; a third-party proposer answers it; anyone can dispute; the DVM is the backstop. It is implemented in `oovp.oracle`.

Use OOv2 when you want an **open set of proposers** competing to answer your requests, and you want to pay a reward for correct answers.

This section covers:

* A [Quick Start](quick-start.md) — the minimal request/propose/settle loop from a contract.
* [Prediction Market](prediction-market.md) — resolve real-world events into a market.
* [Insurance / Arbitration](insurance.md) — verify and pay out claims.

## Integration shape

A contract integrating OOv2 typically:

1. **Registers** itself in `oovp.finder` (`regcontract`) and ensures its identifier + collateral are whitelisted.
2. **Escrows a reward** to the oracle (memo `request:<request_id>`) and calls `requestprice`.
3. Optionally sets **callbacks** so it is notified inline when the answer is proposed, disputed, or settled.
4. **Reads the settled price** via `oracle::has_price` / `oracle::get_price`, or reacts inside its `pricesettled` callback.

## Callbacks

If you enable callbacks with `setcallbacks`, the oracle fires inline actions **to your contract**. Implement these actions with this exact signature:

```cpp
// identifier, timestamp, ancillary_data, price
[[eosio::action]]
void priceproposed(uint64_t identifier, uint32_t timestamp,
                   std::vector<char> ancillary_data, int128_t price);

[[eosio::action]]
void pricedisputed(uint64_t identifier, uint32_t timestamp,
                   std::vector<char> ancillary_data, int128_t price);

[[eosio::action]]
void pricesettled(uint64_t identifier, uint32_t timestamp,
                  std::vector<char> ancillary_data, int128_t price);
```

| Callback | Fired when | Enabled by |
|---|---|---|
| `priceproposed` | A proposer answers your request. | `setcallbacks(..., on_proposed=true, ...)` |
| `pricedisputed` | The proposal is disputed (escalated to DVM). | `setcallbacks(..., on_disputed=true, ...)` |
| `pricesettled` | The request settles. Delivered via `notifysettled` after `settle`. | `setcallbacks(..., on_settled=true, ...)` |

{% hint style="info" %}
`pricesettled` is delivered by the permissionless `notifysettled` action **after** `settle`, so settlement is never blocked by a failing callback. Have a keeper call `notifysettled` (or handle settlement by polling `has_price`).
{% endhint %}

## Required permissions

The oracle sends inline actions (token transfers, DVM requests, callbacks) using its own `eosio.code` permission. Your integrating contract, if it sends inline actions to the oracle, must likewise have `eosio.code` on its active permission. See the [Deployment & Setup Checklist](../../resources/deployment-checklist.md).
