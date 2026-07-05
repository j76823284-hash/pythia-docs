# Network Information

{% hint style="warning" %}
Pythia is currently deployed on the **WAX testnet**. A mainnet deployment is gated on external audit sign-off and multisig hardening — see [Audits & Security](audits-and-security.md).
{% endhint %}

## Chains

| Network | Chain ID | Example endpoints |
|---|---|---|
| WAX Mainnet | `1064487b3cd1a897ce03ae5b6a865651747e2e152090f99c1d19d44e01aea5a4` | `https://wax.greymass.com`, `https://api.waxsweden.org` |
| WAX Testnet | `f16b1833c747c43682f4386fca9cbb327929334a762755ebec17f6f23c9b8a12` | `https://testnet.waxsweden.org`, `https://waxtest.eosdac.io` |

WAX token precision is **8 decimals** (`WAX`); the PYTHIA protocol token is **4 decimals** (`PYTHIA`).

## Contract accounts

Pythia deploys one account per core contract:

| Role | Contract | Notes |
|---|---|---|
| Oracle | `pythiaoorcle` | Optimistic assertion flow and dispute settlement. |
| Voting | `pythiavoting` | DVM commit-reveal voting and binary truth resolution. |
| Stake | `pythiastake1` | Voting stake, delegation, rewards, and slashing. |
| Token | `pythiatoken1` | PYTHIA voting-token balances and protocol weight. |
| Store | `pythiastore1` | Fee collection and collateral accounting. |
| Governor | `pythiagovern` | Governance executed through oracle assertions. |

In addition to contract-code accounts, the protocol uses plain **vault wallets** that are not contracts:

* **Emissions vault** — pre-funded with the PYTHIA emission supply; grants `pythiastake1@eosio.code` so `claimreward`/`restake` can pay from it.
* **Oracle fee vault** — a deposit-only wallet receiving oracle settlement fees.

## Explorers

Use any WAX block explorer to inspect contract tables and actions, e.g. `bloks.io` (mainnet and testnet views) or `waxblock.io`. Table data referenced throughout these docs (`requests`, `assertions`, `pricereqs`, `voterstakes`, `proposals`, …) is public and readable via `cleos get table` or an explorer.
