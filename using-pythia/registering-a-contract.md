# Registering a Contract

Before your contract can request prices from Pythia, it must be **registered** in the `<finder>` registry, and the currency and identifier it uses must be whitelisted. This gate prevents spam and griefing from arbitrary accounts.

{% hint style="info" %}
Throughout these guides, `<finder>`, `<oracle>`, `<store>`, `<voting>`, `<stake>`, and `<token>` are placeholders for the deployed contract accounts. Get the real account names from [Network Information](../resources/network-information.md).
{% endhint %}

## What the registry controls

`<finder>` is a service locator plus three whitelists:

| Registry | Purpose | Admin action |
|---|---|---|
| Interface → address | Resolve `oracle`, `votingtoken`, `store`, `staking`, `governor` to accounts. | `setaddress` |
| Identifier whitelist | Which price identifiers may be requested/asserted. | `addident` / `rmident` |
| Collateral whitelist | Which tokens may be used for rewards and bonds. | `addcollat` / `rmcollat` |
| Contract registry | Which accounts may submit requests. | `regcontract` / `unregcontrct` |

## 1. Register your contract

Registration is authorized by the **creator**, so you register your own contract:

```bash
cleos push action <finder> regcontract \
  '{"contract_account":"myapp","creator":"myapp"}' \
  -p myapp@active
```

Once registered, `myapp` may call `pythiaoorcle::requestprice` and `pythiavoting::requestprice`.

## 2. Ensure your identifier is supported

Price identifiers are `uint64` values with a human-readable label. Adding an identifier is an **admin** action on the finder:

```bash
cleos push action <finder> addident \
  '{"identifier":4763543210000000000,"identifier_utf":"BTC/USD"}' \
  -p <finder>@active
```

Check the [Approved Price Identifiers](../resources/approved-price-identifiers.md) page for existing identifiers. For OOv3 truth assertions you can use the built-in `ASSERT_TRUTH` identifier (`0x4153534552545455`) — pass `identifier = 0` to `asserttruth` to select it automatically.

{% hint style="warning" %}
An identifier is only a **key** for a resolution methodology. What voters actually reference to resolve a dispute is the off-chain methodology associated with that identifier, plus your request's ancillary data. Design ancillary data to be unambiguous.
{% endhint %}

## 3. Ensure your collateral is whitelisted

The token you use for rewards and bonds must be a whitelisted collateral, and (for bond math) it should have a **final fee** configured in `pythiastore1`:

```bash
# Whitelist the collateral (admin)
cleos push action <finder> addcollat \
  '{"token":{"sym":"4,PYTHIA","contract":"<token>"}}' \
  -p <finder>@active

# Set its final fee (admin, on the store)
cleos push action <store> setfinalfee \
  '{"collateral":{"sym":"4,PYTHIA","contract":"<token>"},"fee_amount":"1.0000 PYTHIA"}' \
  -p <store>@active
```

See [Approved Collateral Types](../resources/approved-collateral-types.md).

## 4. Refresh the oracle's parameter cache

The OOv3 assertion path reads a cached copy of the whitelist and final-fee schedule to stay cheap. After adding or changing an identifier or collateral, refresh the cache (permissionless):

```bash
cleos push action <oracle> syncparams \
  '{"identifier":0,"currency":{"sym":"4,PYTHIA","contract":"<token>"}}' \
  -p anyone@active
```

## You're ready

With your contract registered and your identifier + collateral whitelisted, continue to:

* **[Making an Assertion (OOv3)](making-an-assertion.md)** — the single-action assert flow.
* **[Requesting & Proposing a Price (OOv2)](requesting-and-proposing-a-price.md)** — the request/propose flow.
