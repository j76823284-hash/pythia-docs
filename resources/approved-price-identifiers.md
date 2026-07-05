# Approved Price Identifiers

A **price identifier** is a `uint64` key that tells voters _what_ they are resolving and _by what methodology_. Only identifiers whitelisted in `oovp.finder` may be requested or asserted.

## How identifiers work

* Each identifier has a numeric value (`uint64`) and a human-readable label (e.g. `"BTC/USD"`), stored in the finder's `identifiers` table.
* A request/assertion references the identifier; the specific question and resolution source live in the request's **ancillary data / claim**.
* Voters resolve to what the identifier's methodology, applied to the ancillary data, says the answer should be.

## The built-in `ASSERT_TRUTH` identifier

Pythia ships one identifier used by the OOv3 flow and by governance:

| Label | Value (hex) | Value (uint64) | Resolution |
|---|---|---|---|
| `ASSERT_TRUTH` | `0x4153534552545455` | `4707197592731407445` | Binary: `NUMERICAL_TRUE` (10000) or `NUMERICAL_FALSE` (0). |

Passing `identifier = 0` to `asserttruth` automatically selects `ASSERT_TRUTH`. This identifier answers the question "is the asserted claim true?" and is the basis for governance approval and any yes/no assertion.

## Adding an identifier

Identifiers are added by the finder admin (in production, via governance):

```bash
cleos push action <finder> addident '[<uint64_identifier>,"<LABEL>"]' -p <finder>@active
```

After adding an identifier, refresh the oracle's cache so the OOv3 path recognizes it:

```bash
cleos push action <oracle> syncparams '[<uint64_identifier>,{"sym":"4,OOVP","contract":"<token>"}]' -p anyone@active
```

Remove one with `rmident`.

## Choosing an identifier value

For a numeric identifier tied to a name (e.g. `BTC/USD`), a common convention is to derive it from the first 8 bytes of a hash of the label, or to assign sequential values with a maintained mapping. The value is arbitrary as long as it is unique and agreed upon; what matters is that voters know the methodology attached to it.

## Live identifiers

The authoritative list for any network is the finder's `identifiers` table:

```bash
cleos -u <endpoint> get table <finder> <finder> identifiers
```

New identifiers are added per integration as collateral markets and data feeds come online. Because Pythia is generic, integrators typically request a new identifier (with a documented resolution methodology) as part of onboarding a new data type.
