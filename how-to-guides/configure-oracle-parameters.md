---
description: Configure approved currencies, identifiers, and DVM settings.
---

# Configure Oracle parameters

Use this guide when enabling one collateral and identifier pair for a Pythia Oracle/DVM deployment.

## Before you begin

Use the deployment runbook for account creation, authority management, and network-specific commands. This guide starts after the Token, Finder, Store, Stake, Voting, and Oracle contracts exist.

## 1. Configure dependencies

In Finder, register the Oracle, voting-token, Store, and Stake implementations under the canonical interface names. Register the Oracle as a permitted financial/requesting contract where required by your Finder configuration.

Set the Oracle's `finder_contract`, `voting_contract`, and `store_contract` with `oracle::setconfig`. Set Voting's Finder, Stake, and token contracts with `voting::setconfig`.

## 2. Whitelist the currency and identifier

Add the extended collateral symbol to Finder and configure its Store final fee. Add the identifier that Oracle requests may use.

An extended symbol includes both the token symbol and issuing contract. A same-symbol token from another contract is not interchangeable.

## 3. Choose conservative Oracle parameters

Set the default liveness with `oracle::setdefault`. If using assertion defaults, set the default assertion currency with `oracle::setdefcur`. Set `oracle::setburnedbps` before opening disputed work; the contract refuses a changed burn rate while a disputed request or assertion remains unsettled.

Set a fee vault with `oracle::setfeevault` if settlement fees should go to a dedicated deposit-only account.

## 4. Refresh the OOv3 cache

Call `oracle::syncparams(identifier, currency)` for every enabled identifier/currency pair. `asserttruth` reads this cache to validate the pair and calculate the minimum bond.

Repeat this action after changing a relevant Finder whitelist or Store final fee.

## 5. Configure the DVM

Set a phase length with `voting::setphaselen`, participation and agreement thresholds with `voting::setthreshold`, and the maximum number of rolls with `voting::setmaxrolls`.

Use production-length phases and thresholds only after you have enough independent, prepared voters. The DVM cannot safely resolve disputed claims without real participation.

## 6. Verify before accepting users

Check all of the following on-chain:

* `oracle::getminbond` returns a positive amount for the configured currency.
* `asserttruth` accepts a minimum-bond test assertion.
* The Oracle can open a DVM request after a dispute.
* Voting has a valid phase schedule and staked voters can commit and reveal.

Then run one undisputed assertion and one fully disputed assertion before relying on the pair for meaningful value.
