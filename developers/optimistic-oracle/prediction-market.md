# Prediction Market (OOv2)

A prediction market lets users trade on the outcome of a real-world event ("Will X happen by date D?"). Pythia resolves the event; your market contract pays out accordingly.

## The pattern

1. **Create a market** with a question, an outcome encoding, and a resolution time. Encode the question in the request's `ancillary_data` so it is unambiguous.
2. Users **trade** shares of outcomes while the market is open.
3. At resolution time, the market **requests** the outcome from Pythia (OOv2 `requestprice`) with a reward.
4. A proposer answers; if undisputed, it settles optimistically; if disputed, the DVM decides.
5. The market **reads the settled price** (or reacts in its `pricesettled` callback) and lets winners redeem.

## Encoding the outcome

Choose a simple, unambiguous encoding and document it in `ancillary_data`. A common binary encoding:

```
price = 10000  → YES   (1.0000 at 4-decimal fixed point)
price = 0      → NO
price = 5000   → 50/50 / unresolvable (split)
```

Because voters and disputers reference `ancillary_data` to determine the correct answer, spell out the exact resolution source and the encoding, e.g.:

```
q: Will BTC close above $100k on 2026-07-01 (UTC) per <source>?
res: 10000=yes, 0=no, 5000=unresolvable
```

## Requesting resolution

When the market closes, request the outcome. Enable the settled callback so your contract is notified:

```bash
# Set the settled callback once, right after requesting
cleos push action <oracle> setcallbacks '{
  "requester":"mymarket","identifier":...,"timestamp":...,"ancillary_data":"...",
  "on_proposed":false,"on_disputed":false,"on_settled":true
}' -p mymarket@active
```

Implement the callback:

```cpp
[[eosio::action]]
void mymarket::pricesettled(uint64_t identifier, uint32_t timestamp,
                            std::vector<char> ancillary_data, int128_t price) {
    require_auth(oracle_acct);              // only the oracle may call this
    // locate the market by (identifier, timestamp, ancillary_data),
    // record the resolved outcome, and open redemption for winners
}
```

The oracle delivers `pricesettled` via the permissionless `notifysettled` action after `settle`, so a keeper (or the winner claiming) triggers the notification. Alternatively, poll `oracle::has_price` and read `oracle::get_price` when redeeming.

## Sizing bonds and liveness

Set bonds and liveness relative to the value locked in the market so that disputing a wrong answer is profitable and manipulating the outcome is not:

* **Liveness** long enough for honest disputers to notice a wrong proposal (`setliveness`).
* **Bond** large enough that a bad proposal risks meaningful capital (`setbond`, default 2× final fee).

See [Bonds, Fees & Liveness](../bonds-fees-and-liveness.md) and [Economic Security](../../protocol-overview/economic-security.md).

## Disputes

If a proposal is wrong, anyone disputes it and it goes to the DVM. Your market should treat resolution as final only once `has_price` is true — do not pay out on an unsettled proposal. If you enabled `on_disputed`, you can pause redemption while a dispute is pending.
