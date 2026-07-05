# Quick Start (OOv2)

The smallest possible OOv2 integration: request a price, have it proposed, and settle it. This walkthrough uses `cleos`; the same calls apply from any Antelope SDK (WharfKit, eosjs, etc.).

Replace `<oracle>`, `<token>`, `<finder>`, `<store>` with the accounts from [Network Information](../../resources/network-information.md).

## 0. Prerequisites

* Your requester account is registered: `finder::regcontract`.
* Your currency is whitelisted (`finder::addcollat`) and has a final fee (`store::setfinalfee`).
* Your identifier is supported (`finder::addident`).

See [Registering a Contract](../../using-pythia/registering-a-contract.md).

## 1. Compute the request ID

```
request_id = first 8 bytes (LE) of
  sha256( requester(8,LE) ‖ identifier(8,LE) ‖ timestamp(4,LE) ‖ ancillary_data )
```

Compute this off-chain (any keccak/sha256 lib) so you can escrow against it.

## 2. Escrow the reward and request

```bash
cleos push action <token> transfer \
  '["myapp","<oracle>","5.0000 OOVP","request:<request_id>"]' -p myapp@active

cleos push action <oracle> requestprice '{
  "requester":"myapp",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"q:Will it rain in NYC on 2026-07-01? res:1=yes,0=no",
  "currency":{"sym":"4,OOVP","contract":"<token>"},
  "reward":"5.0000 OOVP"
}' -p myapp@active
```

(`ancillary_data` is `bytes`; pass it as hex in real calls. It is shown here as text for readability.)

## 3. Propose an answer

A proposer deposits **bond + final fee** and answers:

```bash
cleos push action <token> transfer \
  '["proposer1","<oracle>","3.0000 OOVP","bond:<request_id>"]' -p proposer1@active

cleos push action <oracle> proposeprice '{
  "proposer":"proposer1",
  "requester":"myapp",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"...",
  "proposed_price":10000
}' -p proposer1@active
```

## 4. Settle after liveness

If no one disputes before liveness expires:

```bash
cleos push action <oracle> settle '{
  "requester":"myapp",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"..."
}' -p anyone@active
```

The proposer gets their bond + final fee + reward back, and the resolved price is written to the request row.

## 5. Read the result

From a contract:

```cpp
#include "oovp.oracle.hpp"

if (oovp::oracle::has_price(oracle_acct, get_self(), identifier, timestamp, ancillary_data)) {
    int128_t price = oovp::oracle::get_price(oracle_acct, get_self(), identifier, timestamp, ancillary_data);
    // ... act on price
}
```

From the CLI, read the `requests` table:

```bash
cleos get table <oracle> <oracle> requests --lower <request_id> --limit 1
```

## Next

* Add **callbacks** so your contract reacts automatically — see the [OOv2 overview](README.md#callbacks).
* Handle the **disputed** path — see [Disputing](../../using-pythia/disputing.md) and [Resolving Disputes on Testnet](../resolving-disputes.md).
* Build something real — [Prediction Market](prediction-market.md) or [Insurance](insurance.md).
