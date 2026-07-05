# Network Information

{% hint style="warning" %}
Pythia is currently deployed on the **WAX testnet**. A mainnet deployment is gated on external audit sign-off and multisig hardening — see [Audits & Security](audits-and-security.md). The **finder registry is the single source of truth** for live contract accounts on any network; resolve them on-chain (see below) rather than hardcoding.
{% endhint %}

## Chains

| Network | Chain ID | Example endpoints |
|---|---|---|
| WAX Mainnet | `1064487b3cd1a897ce03ae5b6a865651747e2e152090f99c1d19d44e01aea5a4` | `https://wax.greymass.com`, `https://api.waxsweden.org` |
| WAX Testnet | `f16b1833c747c43682f4386fca9cbb327929334a762755ebec17f6f23c9b8a12` | `https://testnet.waxsweden.org`, `https://waxtest.eosdac.io` |

WAX token precision is **8 decimals** (`WAX`); the OOVP protocol token is **4 decimals** (`OOVP`).

## Resolving live contract accounts

Every inter-contract lookup in Pythia goes through `oovp.finder`. To find the live account for any interface on a given network, read the finder's `addresses` table:

```bash
cleos -u <endpoint> get table <finder> <finder> addresses
```

The canonical interface names (keys) are:

| Interface name | Resolves to | Role |
|---|---|---|
| `oracle` | `oovp.oracle` | Optimistic oracle (OOv2 + OOv3). |
| `votingtoken` | `oovp.token` | OOVP protocol token. |
| `store` | `oovp.store` | Fee / final-fee schedule. |
| `staking` | `oovp.stake` | DVM staking, rewards, slashing. |
| `governor` | `oovp.govern` | Governance. |
| `feed` | `oovp.feed` | Price feed (optional module). |
| `synths` | `oovp.synths` | Synthetic assets (optional module). |

The DVM voting contract (`oovp.voting`) and the reward vault (`oovp.reward`) are wired into the contracts that use them via their own `setconfig` actions rather than a dedicated finder interface name.

## Account model

Pythia deploys one account per contract. On mainnet the account naming follows this scheme (final names are published in the deployment registry and the on-chain finder at launch):

| Role | Contract | Notes |
|---|---|---|
| Token | `oovp.token` | Protocol voting token (OOVP). |
| Finder | `oovp.finder` | Interface + whitelist registry. |
| Store | `oovp.store` | Fees + final-fee schedule. |
| Stake | `oovp.stake` | Staking; grows RAM over time. |
| Voting | `oovp.voting` | DVM; grows RAM over time. |
| Oracle | `oovp.oracle` | OOv2 + OOv3; grows RAM over time. |
| Governor | `oovp.govern` | Governance. |
| Reward | `oovp.reward` | Generic reward vault (used by app integrations). |

In addition to contract-code accounts, the protocol uses plain **vault wallets** that are not contracts:

* **Emissions vault** — pre-funded with the OOVP emission supply; grants `oovp.stake@eosio.code` so `claimreward`/`restake` can pay from it.
* **Oracle fee vault** — a deposit-only wallet receiving oracle settlement fees.

{% hint style="info" %}
The on-chain account names on the current WAX testnet deployment differ from the mainnet scheme above (they use a testnet-specific prefix and are subject to change). Always resolve the live accounts from the finder on the network you're targeting rather than assuming a fixed name.
{% endhint %}

## Explorers

Use any WAX block explorer to inspect contract tables and actions, e.g. `bloks.io` (mainnet and testnet views) or `waxblock.io`. Table data referenced throughout these docs (`requests`, `assertions`, `pricereqs`, `voterstakes`, `proposals`, …) is public and readable via `cleos get table` or an explorer.
