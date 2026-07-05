# Approved Collateral Types

**Collateral** is the token used to pay rewards and post bonds. Only tokens whitelisted in `oovp.finder` (with a final fee configured in `oovp.store`) may be used.

## What makes a token usable

A token is usable as Pythia collateral when **all** of the following hold:

1. It is whitelisted in the finder (`addcollat`).
2. It has a **final fee** configured for it in the store (`setfinalfee`) — this anchors bond and minimum-bond math.
3. The oracle's parameter cache has been synced for it (`syncparams`), for the OOv3 path.

The store additionally auto-credits any transfer of a token that has a registered final fee (dust from unlisted tokens is ignored), so listing a token both enables it as collateral and enables fee collection in it.

## The `extended_symbol` model

WAX has no global token namespace — a symbol like `OOVP` only means something together with its **contract**. Everywhere collateral appears, Pythia uses an `extended_symbol`:

```json
{ "sym": "4,OOVP", "contract": "<token_contract>" }
```

Always specify both the symbol (with precision) and the contract. Two different contracts can both issue a token called `USDC`; they are distinct collateral to Pythia.

## Choosing collateral

Because a bond's economic weight is only as good as the token behind it, prefer collateral that is:

* **Liquid and hard to manipulate** — an attacker who can cheaply move the token's price can distort the value of bonds.
* **Non-blacklistable in a way that traps funds** — avoid tokens that can freeze the oracle's balance mid-dispute.
* **Appropriately priced via its final fee** — set the final fee so the minimum bond is economically meaningful for the value the token secures.

The protocol token **OOVP** is always a natural collateral (it is the bond/stake token and is used by governance). Applications frequently also list a stable collateral (e.g. a USD-pegged token on its own contract) for markets denominated in dollars.

## Adding collateral

```bash
# 1. Whitelist it
cleos push action <finder> addcollat '[{"sym":"4,USDC","contract":"<usdc_contract>"}]' -p <finder>@active
# 2. Give it a final fee
cleos push action <store>  setfinalfee '[{"sym":"4,USDC","contract":"<usdc_contract>"},"1.0000 USDC"]' -p <store>@active
# 3. Sync the oracle cache
cleos push action <oracle> syncparams '[0,{"sym":"4,USDC","contract":"<usdc_contract>"}]' -p anyone@active
```

Remove collateral with `finder::rmcollat`.

## Live collateral list

The authoritative list per network is the finder's `collaterals` table and the store's `finalfees` table:

```bash
cleos -u <endpoint> get table <finder> <finder> collaterals
cleos -u <endpoint> get table <store>  <store>  finalfees
```
