---
description: Tutorials, operator guides, reference, and protocol explanations.
---

# Pythia Oracle & DVM documentation

Pythia resolves off-chain facts on WAX/Antelope. Most answers settle optimistically; contested answers are decided by the Data Verification Mechanism (DVM).

This documentation covers the Oracle and DVM only. It does not describe Foretell, token distribution operations, or a live-network deployment procedure.

## Start here

| If you want to…                                        | Read                                                                                                  |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| Learn the assertion lifecycle                          | [Make and settle your first truth assertion](tutorials/make-and-settle-your-first-truth-assertion.md) |
| Configure a new Oracle/DVM pair                        | [Configure Oracle parameters](how-to-guides/configure-oracle-parameters.md)                           |
| Resolve a numeric value                                | [Request and settle a value](how-to-guides/request-and-settle-a-value.md)                             |
| Escalate a truth claim to the DVM                      | [Dispute and resolve an assertion](how-to-guides/dispute-and-resolve-an-assertion.md)                 |
| Operate settlement and cleanup cranks                  | [Run DVM maintenance](how-to-guides/run-dvm-maintenance.md)                                           |
| Look up exact actions, tables, deposits, and callbacks | [Oracle/DVM API reference](oracle-dvm-api-reference.md)                                               |
| Understand the protocol model and trust boundaries     | [Oracle/DVM architecture](oracle-dvm-architecture.md)                                                 |
| Understand the proposed governance-token design        | [$PYORA tokenomics](explanations/usdpyora-tokenomics.md)                                              |

## Names used in these documents

`$PYORA` is the proposed public name for Pythia's governance and security token. The current source code, contract namespace, and some deployment configuration still use `OOVP` and `oovp.*`. Treat the deployed ABI, configured token symbol, and configured account names as authoritative for any live transaction.

## Status and scope

These documents describe the source in this repository. They are not a mainnet launch certificate or financial, legal, or tax advice. For a release or deployment decision, use the current [audit status](file:///9782450/audit/STATUS.md) and the relevant runbook.
