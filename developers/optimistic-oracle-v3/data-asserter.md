# Data Asserter (OOv3)

A **data asserter** is a generic pattern for bringing arbitrary off-chain data on-chain: a party asserts a `(key → value)` fact, backs it with a bond, and the data becomes trustworthy after liveness (or a DVM vote if disputed). This is the OOv3 analog of UMA's data-asserter framework.

## When to use it

* You have off-chain data (a sports result, an index value, a document hash, a computation output) that other contracts need.
* You want the data to be **self-service and disputable** rather than gated behind a request/propose market.
* You want the asserter to stand behind the data with a bond.

## Design

Keep a table of asserted data keyed by the assertion ID, and only treat a datum as final once its assertion has settled truthful.

```cpp
CONTRACT dataasserter : public eosio::contract {
public:
  using contract::contract;

  // A record of asserted data.
  TABLE datum {
    uint64_t            id;              // assertion primary key
    eosio::checksum256  assertion_id;    // full assertion id
    std::vector<char>   data_id;         // caller's key for the fact
    std::vector<char>   data;            // the asserted value
    eosio::name         asserter;
    bool                resolved;        // settled truthful?
    uint64_t primary_key() const { return id; }
  };
  using data_t = eosio::multi_index<"data"_n, datum>;

  // Register a fact BEFORE asserting so we can link the assertion callback back.
  [[eosio::action]] void assertdata(eosio::name asserter,
                                     std::vector<char> data_id,
                                     std::vector<char> data);

  // Oracle callback: mark the datum resolved iff the assertion held.
  [[eosio::action]]
  void assertresvd(eosio::checksum256 assertion_id, bool resolution);
};
```

## Flow

1. **Deposit** the bond to the oracle (memo `assert`).
2. **Assert** the fact. Encode `data_id` + `data` into the `claim` so voters can verify it, and set `callback_recipient` to your data-asserter contract:

```bash
cleos push action <oracle> asserttruth '{
  "asserter":"reporter1",
  "claim":"<hex: data_id=... value=... source=...>",
  "callback_recipient":"dataasserter",
  "escalation_manager":"",
  "liveness":7200,
  "currency":{"sym":"4,OOVP","contract":"<token>"},
  "bond":"2.0000 OOVP",
  "identifier":0,
  "domain_id":0
}' -p reporter1@active
```

3. **Resolve.** After `settleassert`, a keeper calls `notifyassert`, which fires `assertresvd(assertion_id, resolution)` to your contract. Mark the datum resolved only when `resolution == true`.

4. **Consume.** Downstream contracts read your `data` table and use only resolved entries. Because the data is only marked resolved after the assertion settled truthful, consumers inherit Pythia's optimistic + DVM guarantees.

## Making the claim verifiable

The `claim` bytes are what disputers and voters inspect. Include everything needed to independently verify the value:

* The **key/identity** of the fact (`data_id`).
* The **value** being asserted.
* The **source and methodology** (URL, API, computation) that determines the correct value.
* Any **timestamp or block reference** the value is anchored to.

Ambiguous claims are the main reason a data assertion resolves the "wrong" way — the DVM resolves to what the claim, read literally against its stated source, actually says.

## Multiple asserters / competition

Because assertion IDs incorporate the asserter and block time, several parties can assert overlapping facts. Your consuming contract decides which resolved datum to trust (e.g. first-resolved, highest-bond). Disputers keep all of them honest — a wrong assertion is a free bond for whoever disputes it.
