# Quick Start (OOv3)

The smallest possible OOv3 integration: deposit a bond, assert a claim, and settle it. This mirrors UMA's OOv3 `assertTruth` quick start, adapted to Antelope.

Replace `<oracle>`, `<token>` with the accounts from [Network Information](../../resources/network-information.md).

## 0. Prerequisites

* Your bond currency is whitelisted collateral (`finder::addcollat`) with a final fee (`store::setfinalfee`).
* The oracle's parameter cache is synced for that currency (`oracle::syncparams`).

See [Registering a Contract](../../using-pythia/registering-a-contract.md).

## 1. Deposit the bond

```bash
cleos push action <token> transfer \
  '["myasserter","<oracle>","2.0000 PYTHIA","assert"]' -p myasserter@active
```

The deposit must be at least the **minimum bond** (2× the final fee at the default 50% burn rate).

## 2. Assert

```bash
cleos push action <oracle> asserttruth '{
  "asserter":"myasserter",
  "claim":"<hex-encoded claim bytes>",
  "callback_recipient":"",
  "escalation_manager":"",
  "liveness":7200,
  "currency":{"sym":"4,PYTHIA","contract":"<token>"},
  "bond":"2.0000 PYTHIA",
  "identifier":0,
  "domain_id":0
}' -p myasserter@active
```

Or, for protocol defaults, deposit the minimum bond and call `assertwdflt`:

```bash
cleos push action <oracle> assertwdflt \
  '{"asserter":"myasserter","claim":"<hex>","callback_recipient":""}' -p myasserter@active
```

## 3. Find and settle the assertion

Read your assertion from the `assertions` table (by asserter secondary index) to get its `id`:

```bash
cleos get table <oracle> <oracle> assertions --index 2 --key-type i64 -L myasserter
```

After liveness with no dispute, settle it — you recover your full bond:

```bash
cleos push action <oracle> settleassert '{"assertion_id":<id>}' -p anyone@active
```

## 4. Read the result

From a contract, read the `assertions` table directly (it is public so integrators can query it):

```cpp
#include "pythiaoorcle.hpp"

oovp::oracle::assertions tbl(oracle_acct, oracle_acct.value);
auto a = tbl.find(assertion_pk);
bool ok = a != tbl.end() && a->settled && a->settlement_resolution;
```

Or register a `callback_recipient` and implement `assertresvd(assertion_id, resolution)` — see the [OOv3 overview](README.md#callbacks).

## Minimal integrating contract

A contract that asserts and reacts:

```cpp
CONTRACT myasserter : public eosio::contract {
public:
  using contract::contract;

  // Called by a keeper via notifyassert after the assertion settles.
  [[eosio::action]]
  void assertresvd(eosio::checksum256 assertion_id, bool resolution) {
    require_auth(oracle_acct);   // only the oracle may deliver this
    // record resolution keyed by assertion_id and act on it
  }
};
```

The asserter deposits its bond via a token transfer (memo `assert`) and calls `asserttruth` inline (requires `eosio.code` on its active permission).
