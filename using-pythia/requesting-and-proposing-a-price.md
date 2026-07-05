# Requesting & Proposing a Price (OOv2)

The OOv2 flow separates the party that **asks** for data (the requester) from the party that **answers** it (the proposer). It is implemented in `oovp.oracle`.

```
requestprice → proposeprice → (liveness) → settle
                    │
                    └── disputeprice → DVM → settle
```

{% hint style="info" %}
Prerequisites: your requesting contract must be registered in `oovp.finder`, and your currency + identifier must be whitelisted. See [Registering a Contract](registering-a-contract.md).
{% endhint %}

## 1. Escrow the reward, then request

The requester escrows the **reward** (not the final fee) by transferring it to the oracle with memo `request:<request_id>`, then calls `requestprice`.

The `request_id` is deterministic:

```
request_id = first 8 bytes (LE) of sha256( requester(8,LE) ‖ identifier(8,LE) ‖ timestamp(4,LE) ‖ ancillary_data )
```

Compute it off-chain, escrow the reward against it, then request:

```bash
# 1a. Escrow the reward
cleos push action <token> transfer \
  '["myapp","<oracle>","5.0000 OOVP","request:1234567890"]' -p myapp@active

# 1b. Open the request
cleos push action <oracle> requestprice '{
  "requester":"myapp",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"...",
  "currency":{"sym":"4,OOVP","contract":"<token>"},
  "reward":"5.0000 OOVP"
}' -p myapp@active
```

Use `timestamp: 0` for an **event-based** request whose answer is not tied to a specific time.

### Optional: tune the request

While the request is in the `REQUESTED` state, the requester can adjust it:

```bash
cleos push action <oracle> setliveness '{"requester":"myapp","identifier":...,"timestamp":...,"ancillary_data":"...","liveness":3600}' -p myapp@active
cleos push action <oracle> setbond     '{"requester":"myapp",...,"bond":"10.0000 OOVP"}' -p myapp@active
cleos push action <oracle> setrefund   '{"requester":"myapp",...}' -p myapp@active
cleos push action <oracle> setcallbacks '{"requester":"myapp",...,"on_proposed":true,"on_disputed":true,"on_settled":true}' -p myapp@active
```

## 2. Propose an answer

A proposer answers by first depositing **bond + final fee** (memo `bond:<request_id>`), then calling `proposeprice`:

```bash
# 2a. Deposit bond + final fee
cleos push action <token> transfer \
  '["proposer1","<oracle>","3.0000 OOVP","bond:1234567890"]' -p proposer1@active

# 2b. Propose the price
cleos push action <oracle> proposeprice '{
  "proposer":"proposer1",
  "requester":"myapp",
  "identifier":4763543210000000000,
  "timestamp":1767225600,
  "ancillary_data":"...",
  "proposed_price":6543200000000
}' -p proposer1@active
```

`proposed_price` is an `int128` in your identifier's fixed-point convention. Proposing starts the liveness clock (the request's custom liveness, or the oracle default of 2 hours).

## 3. Settle

Settlement is **permissionless** — anyone can trigger it.

* **No dispute (liveness expired):** the proposer wins. They receive their **bond + final fee + reward** back.

```bash
cleos push action <oracle> settle \
  '{"requester":"myapp","identifier":...,"timestamp":...,"ancillary_data":"..."}' -p anyone@active
```

* **Disputed:** settlement waits for the DVM, then pays the side that matches the resolved price (see [Disputing](disputing.md)).

If `on_settled` callbacks were enabled, `notifysettled` delivers the `pricesettled` callback to the requester without blocking settlement.

## Reading the result

Integrating contracts read the settled price via the oracle's static getters:

```cpp
bool ready = oracle::has_price(oracle_acct, requester, identifier, timestamp, ancillary_data);
int128_t price = oracle::get_price(oracle_acct, requester, identifier, timestamp, ancillary_data);
```

## Cancelling a stuck request

If a request sits in the `REQUESTED` state with no proposer for at least `MIN_CANCEL_AGE` (**7 days**), the requester can `cancel` it and recover the escrowed reward:

```bash
cleos push action <oracle> cancel \
  '{"requester":"myapp","identifier":...,"timestamp":...,"ancillary_data":"..."}' -p myapp@active
```

## Recovering orphaned deposits

Deposits that were never consumed can be reclaimed by their depositor:

| Action | Recovers |
|---|---|
| `reclaimreq(depositor, request_id)` | Unused/excess request (reward) escrow. |
| `reclaimbond(depositor, request_id)` | An unconsumed OOv2 bond deposit. |
| `reclaimasrt(depositor)` | An unconsumed OOv3 assertion deposit. |
