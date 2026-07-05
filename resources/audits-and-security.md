# Audits & Security

Pythia's security model is economic: honest resolution is the rational equilibrium because attacking the oracle costs more than it can yield (see [Economic Security](../protocol-overview/economic-security.md)). This page covers the code-security posture and how to report issues.

## Status

{% hint style="warning" %}
Pythia is **pre-mainnet**. It is deployed and exercised on the WAX testnet. A mainnet launch is explicitly gated on:

1. **External audit sign-off** of the contract suite.
2. **Multisig hardening** of every contract and vault admin key (no single deployer key should retain control of the oracle).
3. Handing parameter control to `pythiagovern`.

Do not treat testnet deployments as production, and do not integrate against mainnet until these gates are cleared and this page says so.
{% endhint %}

## What is security-critical

The highest-value surfaces to review before relying on Pythia:

* **Bond & settlement math** in `pythiaoorcle` — the payout and burned-bond calculations for OOv2 and OOv3.
* **DVM resolution & slashing** in `pythiavoting` and `pythiastake1` — snapshotting, quorum, and the token-conserving slash→reward redistribution.
* **Governance execution** in `pythiagovern` — the `target@exec` authorization requirement and the transaction-hash re-verification at execute time.
* **Registry gating** in `<finder>` — only registered contracts may request; only whitelisted collateral/identifiers are accepted.
* **`eosio.code` permissions** — every contract that sends inline actions must hold its own `eosio.code`, and governed actions must be `linkauth`-scoped to `exec`.

## Design safeguards already in place

* Commit-reveal voting with commit-time weight and round stake snapshots (anti-flash-loan).
* Unstake cooldown (default 7 days) preventing borrow-and-vote.
* All bond/fee movement deferred to settlement — nothing is paid out at dispute time.
* Permissionless, bounded cranks so the protocol stays live without privileged operators, and RAM is reclaimable.
* Deposit-recovery actions (`reclaimreq`/`reclaimbond`/`reclaimasrt`) so funds are never stranded by an abandoned flow.
* Governance timelock + allowlisted, delayed emergency path.

## Reviewing the code yourself

Because the contracts are on-chain, you can independently verify behavior:

* Read the ABIs and tables of the deployed accounts (resolve them via the [finder](network-information.md)).
* Reproduce the bond math from [Bonds, Fees & Liveness](../developers/bonds-fees-and-liveness.md) against a real settlement.
* Run a full dispute cycle on testnet ([walkthrough](../developers/resolving-disputes.md)).

## Reporting a vulnerability

If you discover a security issue, **report it privately** — do not open a public issue or disclose it on-chain in a way that could be exploited before a fix. Use the security contact published with the project's official channels (see [Links](links.md)). A responsible-disclosure and bug-bounty program is planned for launch; until it is live, coordinate privately with the team.

## Integrator responsibilities

Pythia gives you a truth machine, but **safe integration is on you**:

* Size bonds and liveness so that no false resolution is profitable for your contract's value at risk.
* Only act on **settled** results (`has_price` / `settlement_resolution`), never on unsettled proposals or assertions.
* Use liquid, non-manipulable collateral.
* Assume the DVM is the final arbiter; never depend on a specific proposer being honest.
