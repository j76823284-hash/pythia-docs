# Reward Vault

A generic, reusable **reward-distribution vault**: a pre-funded pool of a single reward token (typically PYTHIA) from which whitelisted "source" contracts grant rewards to users. The vault is policy-free — _why_ a reward is granted lives in the calling source contract — which keeps it reusable across integrations.

`<reward>` is optional infrastructure for application-level reward distribution. It is not part of the core oracle/DVM path.

## Memo conventions (token transfers to the reward contract)

| Memo | Purpose |
|---|---|
| `rewardpool` | Fund the pool by raising the contract's reward-token balance. |

## Actions

| Action | Auth | Description |
|---|---|---|
| `setconfig(reward_token, reward_symbol, admin)` | self | Configure the reward token and admin. |
| `addsource(source, budget)` | admin | Whitelist a source contract (lifetime `budget`; 0 = unlimited). |
| `rmsource(source)` | admin | Remove a source. |
| `accrue(source, user, amount, memo)` | source | Grant a reward to a user (best-effort; see below). |
| `claim(user)` | user | Withdraw a user's accrued reward. |

## Accrual semantics

`accrue` is **best-effort and never throws on shortfall**: it caps the grant to the available (unreserved) pool and the source's remaining budget. A depleted pool therefore can never break the calling contract's transaction. The accounting invariant is that the contract's reward-token balance always covers `reserved` (the sum of all users' unclaimed balances).

## Key tables (readable)

| Table | Contents |
|---|---|
| `rewardcfg` | Singleton: reward token, symbol, admin. |
| `poolstate` | Singleton: `reserved` (raw units owed across all users). |
| `sources` | Whitelisted source contracts with granted/budget counters. |
| `balances` | Per-user accrued (unclaimed) reward. |

## Static getter

```cpp
asset reward::get_outstanding(reward, user);
```
