# Contract Reference

Pythia is a suite of Antelope/C++ (CDT, C++20) contracts. Each contract is a single account with its own actions and tables. This reference documents every public action, the key tables you can read, and the memo conventions used for token transfers.

## The contracts

| Contract | Role | Reference |
|---|---|---|
| `pythiaoorcle` | Optimistic oracle — OOv2 (request/propose) + OOv3 (assert) flows. | [pythiaoorcle](oovp-oracle.md) |
| `pythiavoting` | DVM — commit-reveal, stake-weighted dispute resolution. | [pythiavoting](oovp-voting.md) |
| `pythiastake1` | Staking, delegation, emissions, slashing, vote-locking. | [pythiastake1](oovp-stake.md) |
| `pythiagovern` | Governance proposals executed via oracle assertions. | [pythiagovern](oovp-govern.md) |
| `pythiastore1` | Fee collection and per-collateral final-fee schedule. | [pythiastore1](oovp-store.md) |
| Finder registry | Service locator + identifier/collateral/contract registries. | [Finder Registry](oovp-finder.md) |
| `pythiatoken1` | PYTHIA protocol token (eosio.token + ANT-style burn/close). | [pythiatoken1](oovp-token.md) |
| Reward vault | Generic pre-funded reward vault for app integrations. | [Reward Vault](oovp-reward.md) |

## Dependency / deploy order

```
pythiatoken1 → <finder> → pythiastore1 → pythiastake1 → pythiavoting → pythiaoorcle → pythiagovern → <reward>
```

See the [Deployment & Setup Checklist](../deployment-checklist.md) for the full bring-up.

## Cross-contract static getters

Contracts expose `static` read-only methods so other contracts can query their tables without knowing internal layout. These are the intended integration surface:

```cpp
stake::get_stake(stake, voter)                 // uint128_t (lock-boosted voting power)
stake::get_raw_stake(stake, voter)             // uint128_t (raw slashable stake)
stake::get_total_stake(stake)                  // uint128_t
stake::get_delegate(stake, voter)              // name
finder::get_implementation(finder, iface)      // name
finder::is_identifier_supported(finder, id)    // bool
finder::is_collateral_whitelisted(finder, ext) // bool
finder::is_contract_registered(finder, acct)   // bool
voting::has_price(voting, requester, id, ts, anc)  // bool
voting::get_price(voting, requester, id, ts, anc)  // int128_t
oracle::has_price(oracle, requester, id, ts, anc)  // bool
oracle::get_price(oracle, requester, id, ts, anc)  // int128_t
oracle::getminbond(oracle, currency)           // asset
store::get_final_fee(store, collateral)        // asset
reward::get_outstanding(reward, user)          // asset
```

## Conventions

* **Basis points (bps).** Percentages are bps unless noted (10000 = 100%).
* **Fixed point.** Prices and amounts are integers in the token/identifier precision. PYTHIA is 4 decimals (`1.0000 = 10000`).
* **Memos are the API for transfers.** WAX has no `approve`; you pre-fund a contract by transferring tokens with a specific memo. Each contract page lists its memo conventions.
* **`extended_symbol`.** Tokens are always `(symbol, contract)` pairs.
