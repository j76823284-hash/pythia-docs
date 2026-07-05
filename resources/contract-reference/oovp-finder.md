# Finder Registry

The service locator and registry. Maps interface names to contract accounts and holds the identifier, collateral, and contract whitelists that gate access to the oracle and DVM.

## Actions

| Action | Auth | Description |
|---|---|---|
| `setaddress(interface_name, implementation)` | self | Map an interface name to an account. |
| `addident(identifier, identifier_utf)` | self | Whitelist a price identifier. |
| `rmident(identifier)` | self | Remove an identifier. |
| `addcollat(token)` | self | Whitelist a collateral (`extended_symbol`). |
| `rmcollat(token)` | self | Remove a collateral. |
| `regcontract(contract_account, creator)` | creator | Register a contract so it may submit requests. |
| `unregcontrct(contract_account)` | self | Unregister a contract. |

## Interface names

| Name | Role |
|---|---|
| `oracle` | Optimistic oracle. |
| `votingtoken` | PYTHIA token. |
| `store` | Fee store. |
| `staking` | Stake contract. |
| `governor` | Governance. |
| `feed` | Price feed (optional module). |
| `synths` | Synthetic assets (optional module). |

## Key tables (readable)

| Table | Contents |
|---|---|
| `addresses` | Interface name → implementation account. |
| `identifiers` | Whitelisted price identifiers (numeric + label + active flag). |
| `collaterals` | Whitelisted collateral tokens. |
| `contracts` | Registered contracts allowed to request. |

## Static getters

```cpp
name finder::get_implementation(finder, interface_name);         // reverts if not found
bool finder::is_identifier_supported(finder, identifier);
bool finder::is_collateral_whitelisted(finder, extended_symbol); // matches contract + symbol
bool finder::is_contract_registered(finder, account);
```

The finder is the **source of truth** for live contract accounts on any network — see [Network Information](../network-information.md).
