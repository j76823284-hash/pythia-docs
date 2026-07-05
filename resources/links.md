# Links

## Pythia

* **Documentation** — this book.
* **Protocol contracts** — the `oovp.*` Antelope/C++ contract suite (source repository).
* **Official channels** (website, Discord, X/Twitter, security contact) — published with the project's launch materials. Until mainnet, treat any address you find as unofficial unless it is confirmed through an official channel.

{% hint style="info" %}
Pythia is the **oracle / DVM / staking / governance** protocol. Related products — **Foretell** (prediction markets) and **Prism** (benchmark / market data) — integrate with Pythia but are separate products with their own sites. Don't conflate them.
{% endhint %}

## WAX / Antelope

* **WAX** — [https://wax.io](https://wax.io)
* **Antelope** — [https://antelope.io](https://antelope.io)
* **Antelope CDT (Contract Development Toolkit)** — [https://github.com/AntelopeIO/cdt](https://github.com/AntelopeIO/cdt)
* **Leap / Spring (nodeos)** — [https://github.com/AntelopeIO](https://github.com/AntelopeIO)
* **Block explorer** — [https://bloks.io](https://bloks.io) (mainnet + testnet), [https://waxblock.io](https://waxblock.io)

## Public endpoints

| Network | Endpoints |
|---|---|
| WAX Mainnet | `https://wax.greymass.com`, `https://api.waxsweden.org` |
| WAX Testnet | `https://testnet.waxsweden.org`, `https://waxtest.eosdac.io` |

Chain IDs and account resolution: [Network Information](network-information.md).

## Prior art

Pythia is a from-scratch Antelope implementation of the optimistic-oracle and DVM design pioneered by UMA. Their documentation and contracts are an excellent conceptual reference:

* **UMA Protocol** — [https://uma.xyz](https://uma.xyz)
* **UMA docs** — [https://docs.uma.xyz](https://docs.uma.xyz)
* **UMA contracts** — [https://github.com/UMAprotocol/protocol](https://github.com/UMAprotocol/protocol)

Where Pythia differs from UMA, it is because of Antelope's model (accounts and permissions instead of `msg.sender`, memo-based token receipt instead of `approve`, permissionless cranks instead of keepers-with-gas, and multi-index tables instead of Solidity mappings). Those differences are documented throughout this book.
