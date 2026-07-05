# oovp.token

The **OOVP** protocol token — a standard `eosio.token` implementation with ANT-style additions (holder burn, explicit RAM `open`/`close`). Voting power is determined by `oovp.stake`, not by token balance snapshots, so the token itself is a plain fungible token.

* **Symbol:** `OOVP`, 4 decimals (`1.0000 = 10000` raw units).
* **Uses:** staking, bonds, proposal bonds, rewards.

## Actions

| Action | Auth | Description |
|---|---|---|
| `create(issuer, maximum_supply)` | self | Create the token with a max supply. |
| `issue(to, quantity, memo)` | issuer | Issue tokens up to max supply. |
| `retire(quantity, memo)` | issuer | Reduce current **supply** only. |
| `burn(from, quantity, memo)` | from | Holder burn: reduce both **supply** and **max_supply**. |
| `transfer(from, to, quantity, memo)` | from | Standard transfer. |
| `open(owner, symbol, ram_payer)` | ram_payer | Open a zero balance row (explicit RAM control). |
| `close(owner, symbol)` | owner | Close a zero balance row. |

## Burn semantics

| Action | Effect |
|---|---|
| `retire` | `supply -= quantity` |
| `burn` | `supply -= quantity` **and** `max_supply -= quantity` |

`burn` permanently removes units from the lifetime cap, which is how protocol fees can deflate OOVP. (It does not erase unrelated unissued headroom that existed before the burn.)

## Key tables (readable)

| Table | Scope | Contents |
|---|---|---|
| `accounts` | owner | Per-account balances. |
| `stat` | symbol code | Supply, max supply, issuer. |

## Static getters

```cpp
asset token::get_balance(token, owner, symbol);
asset token::get_supply(token, symbol);
```

{% hint style="info" %}
The token contract's behavior is derived from the ANT token-standard repo and adapted into the Pythia protocol stack. The on-chain ticker remains **OOVP** while "Pythia" is the protocol brand.
{% endhint %}
