---
description: Verified off-chain facts for WAX and Antelope.
---

# Pythia Oracle & DVM

**An optimistic oracle that can record any verifiable truth or piece of data on the WAX blockchain.** 🔮

> _Pythia was the high priestess of the Temple of Apollo at Delphi — the mouthpiece of the most consulted oracle of the ancient world. Kings and merchants paid for her answers and traded on them. Pythia does the same thing with cryptography: every question is a request, and the optimistic oracle is the priestess that finally speaks the answer._

Pythia is an **optimistic oracle** and **dispute-arbitration system** built natively for [Antelope](https://antelope.io) chains (WAX). It lets smart contracts bring arbitrary, verifiable data on-chain cheaply and quickly, backed by a token-weighted human voting layer that resolves disputes. Pythia is a from-scratch Antelope/C++ implementation of the design pioneered by [UMA](https://uma.xyz) — the same optimistic-oracle and Data Verification Mechanism (DVM) architecture, rebuilt for WAX's account, table, and resource model.

Pythia's oracle provides verified on-chain data for use cases such as:

1. Prediction markets
2. Insurance and claims arbitration
3. Custom derivatives and synthetic assets
4. On-chain governance execution

The oracle is designed to be modular and extensible. Two flows are live inside the same `pythiaoorcle` contract: **OOv2** (request/propose) and **OOv3** (assertions). Which one you use depends on your application.

***

## Core concepts

Learn the fundamentals of Pythia's optimistic oracle.

|                                                                                      |                                                                           |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- |
| [**How the Oracle Works**](protocol-overview/how-pythias-oracle-works.md)            | The optimistic escalation game between proposers, disputers, and the DVM. |
| [**Optimistic Oracle v2 & v3**](protocol-overview/optimistic-oracle-v2-and-v3.md)    | The two request flows and how to choose between them.                     |
| [**The DVM**](protocol-overview/the-dvm.md)                                          | Commit-reveal, stake-weighted voting that resolves disputes.              |
| [**Staking, Rewards & Slashing**](protocol-overview/staking-rewards-and-slashing.md) | How stakers secure the oracle and earn emissions.                         |

### Which oracle flow fits your use case?

Pythia's single oracle contract exposes both flows. Choose based on how your application produces answers:

| OOv2 (request → propose)                                                                      | OOv3 (assert)                                                                                          |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Your contract **requests** data; a third-party **proposer** answers it.                       | Your contract (or a bot) **asserts** an answer and posts the bond itself, in one action.               |
| The request carries the parameters proposers/disputers must follow (reward, bond, liveness).  | The asserter chooses bond, liveness, currency, and an optional escalation manager.                     |
| Primary use cases: prediction markets, insurance, event resolution with an open proposer set. | Primary use cases: data assertions, governance approval, self-resolving markets, content/verification. |

The dispute path and DVM backstop are identical for both.

***

## Developer quickstart

Once you have chosen a flow, dive into the guides:

{% tabs %}
{% tab title="OOv2" %}
OOv2 suits protocols that want third parties to propose answers to open data requests.

* [**Prediction Market**](developers/optimistic-oracle/prediction-market.md) — identify and settle real-world events.
* [**Insurance / Arbitration**](developers/optimistic-oracle/insurance.md) — verify and resolve claims.
* [**Quick Start**](developers/optimistic-oracle/quick-start.md) — the minimal request → propose → settle loop.
{% endtab %}

{% tab title="OOv3" %}
OOv3 suits protocols that just need asserted data verified.

* [**Quick Start**](developers/optimistic-oracle-v3/quick-start.md) — the simplest possible assertion.
* [**Data Asserter**](developers/optimistic-oracle-v3/data-asserter.md) — assert arbitrary off-chain data on-chain.
* [**Escalation Managers**](developers/optimistic-oracle-v3/escalation-managers.md) — pluggable arbitration policy.
{% endtab %}
{% endtabs %}

***

## Community & governance

The **PYTHIA** token secures Pythia's oracle through decentralized governance and economic guarantees against corruption. Stakers vote on disputes and governance proposals, earning emission rewards for honest participation and losing stake (slashing) for incorrect or absent votes.

**Governance proposals** are executed through the oracle itself: a proposal is submitted as an OOv3 assertion, and if it settles as _true_ it becomes executable on-chain.

|                                                                          |                                                                  |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| [**Governance**](community/governance/)                                  | The PYTHIA token, staking, and how the DVM secures the protocol. |
| [**The Proposal Process**](community/governance/the-proposal-process.md) | How changes are proposed and ratified.                           |
| [**On-Chain Proposals**](community/governance/dao-proposals.md)          | Submitting executable on-chain actions through `pythiagovern`.   |

***

## Resources & support

* Browse the [**Contract Reference**](resources/contract-reference/) for every action, table, and memo convention.
* Check [**Network Information**](resources/network-information.md) for chain IDs, endpoints, and contract accounts.
* Review the [**Audits & Security**](resources/audits-and-security.md) page before integrating.

{% hint style="warning" %}
**Status.** Pythia is deployed and exercised on the **WAX testnet**. A mainnet launch is gated on external audit sign-off and multisig hardening of the admin accounts — see [Audits & Security](resources/audits-and-security.md). Treat everything here as pre-mainnet unless a network page says otherwise.
{% endhint %}
