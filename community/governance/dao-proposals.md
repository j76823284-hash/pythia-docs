# On-Chain (DAO) Proposals

An on-chain proposal is a bundle of concrete Antelope actions that execute atomically if the proposal is approved. This is how the protocol changes its own parameters, moves treasury funds, or upgrades registry entries — all under staker control.

## Anatomy of a proposal transaction

Each entry in a proposal's `transactions` array is a `transaction_data`:

```cpp
struct transaction_data {
    name                          to;             // target contract
    name                          action_name;    // action to call
    std::vector<permission_level> authorization;  // must be exactly [{to, "exec"}]
    std::vector<char>             data;            // serialized action arguments
};
```

Rules enforced at execution time:

* **1–10 transactions** per proposal.
* Each `to` account must exist.
* **Exactly one authorization**, and it must be **`{actor: to, permission: "exec"}`**.

The `exec` permission requirement is the crux of governance safety: governed contracts link their sensitive actions to an `exec` permission whose authority is the governor contract (via `eosio.code`). So only a proposal that the DVM approved — executed by `oovp.govern` — can invoke those actions.

## Building the `data` field

`data` is the **serialized arguments** for `action_name`, exactly as the target ABI expects. Produce it the same way you would for any Antelope action, e.g. with `cleos convert pack_action_data` or your SDK's serializer:

```bash
cleos convert pack_action_data <store> setfinalfee \
  '{"collateral":{"sym":"4,WAXUSDC","contract":"<collat_token>"},"fee_amount":"2.0000 WAXUSDC"}'
# → hex string to place in transactions[i].data
```

## Example: change the DVM emission rate

A proposal with a single action that calls `oovp.stake::setemission`:

```json
{
  "proposer": "<proposer_contract>",
  "transactions": [
    {
      "to": "<stake>",
      "action_name": "setemission",
      "authorization": [{ "actor": "<stake>", "permission": "exec" }],
      "data": "<packed setemission args>"
    }
  ],
  "ancillary_data": "PIP-7: reduce staking emission to 0.04 OOVP/sec"
}
```

Submit it via `propose`, fund the bond, and — if it settles true — execute it. See [The Proposal Process](the-proposal-process.md).

## Example: multi-action proposal

Proposals can bundle related changes so they take effect together (up to 10 actions):

```json
"transactions": [
  { "to": "<finder>", "action_name": "addcollat",   "authorization": [{"actor":"<finder>","permission":"exec"}], "data": "..." },
  { "to": "<store>",  "action_name": "setfinalfee",  "authorization": [{"actor":"<store>","permission":"exec"}],  "data": "..." },
  { "to": "<oracle>", "action_name": "syncparams",   "authorization": [{"actor":"<oracle>","permission":"exec"}], "data": "..." }
]
```

## Writing effective ancillary data

`ancillary_data` is what stakers read to decide how to vote (and, if disputed, what the DVM references). Make it unambiguous:

* State the change, the exact new value, and the rationale.
* Reference an off-chain proposal document / discussion where relevant.
* Ensure it matches the actual `transactions` — voters approve the transaction **hash**, and any mismatch between description and bytes should be a reason to dispute.

## Approval semantics

A proposal is **approved** only when its oracle assertion is `settled == true` **and** `settlement_resolution == true`. Concretely:

* **Undisputed** for the full voting period → optimistically true → approved.
* **Disputed** and DVM votes `NUMERICAL_TRUE` → approved.
* **Disputed** and DVM votes `NUMERICAL_FALSE` → rejected; cannot execute.

This is verified by `is_approved`, which also checks the asserter is the governor and the stored claim still matches the proposal — so a proposal cannot be executed against a different or forged assertion.
