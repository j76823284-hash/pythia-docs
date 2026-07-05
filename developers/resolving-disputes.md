# Resolving Disputes on Testnet

This is an end-to-end walkthrough of a full dispute on the **WAX testnet**: assert (or propose) → dispute → vote → resolve → settle. It's the fastest way to understand Pythia's escalation game hands-on.

{% hint style="info" %}
On testnet, Pythia is typically configured with **short** liveness and phase lengths (down to the 30-second minimums) so a full dispute cycle completes in minutes instead of days. On mainnet these are hours/days. Get testnet accounts and endpoints from [Network Information](../resources/network-information.md).
{% endhint %}

## Setup

You'll want at least three roles (they can be three testnet accounts you control):

* an **asserter/proposer**,
* a **disputer**,
* one or more **stakers/voters** (with staked PYTHIA),
* and a **keeper** to push cranks (can be any of the above).

## Path A — dispute an OOv3 assertion

**1. Assert.**

```bash
cleos push action <token> transfer '["asserter","<oracle>","2.0000 PYTHIA","assert"]' -p asserter@active
cleos push action <oracle> asserttruth '{
  "asserter":"asserter","claim":"<hex>","callback_recipient":"","escalation_manager":"",
  "liveness":60,"currency":{"sym":"4,PYTHIA","contract":"<token>"},"bond":"2.0000 PYTHIA",
  "identifier":0,"domain_id":0}' -p asserter@active
```

**2. Dispute (within liveness).**

```bash
cleos push action <token> transfer '["disputer","<oracle>","2.0000 PYTHIA","assert"]' -p disputer@active
cleos push action <oracle> dispassert '{"disputer":"disputer","assertion_id":<id>}' -p disputer@active
```

This escalates a request to the DVM (identifier `ASSERT_TRUTH`).

## Path B — dispute an OOv2 proposal

**1. Request + propose.** Follow the [OOv2 quick start](optimistic-oracle/quick-start.md) through `proposeprice`.

**2. Dispute (within liveness).**

```bash
cleos push action <token> transfer '["disputer","<oracle>","3.0000 PYTHIA","bond:<request_id>"]' -p disputer@active
cleos push action <oracle> disputeprice '{"disputer":"disputer","requester":"myapp","identifier":...,"timestamp":...,"ancillary_data":"..."}' -p disputer@active
```

## Vote in the DVM

The dispute is queued for the **next** round. During the **commit** phase, each staker commits, then reveals in the **reveal** phase. For an `ASSERT_TRUTH` dispute the vote is binary: `10000` (asserter was right) or `0`.

```bash
# commit (hash of price+salt+voter+timestamp+round+identifier)
cleos push action <voting> commitvote '{"voter":"voter1","identifier":...,"timestamp":...,"ancillary_data":"<stamped>","hash":"<sha256 hex>"}' -p voter1@active

# reveal (in the reveal phase)
cleos push action <voting> revealvote '{"voter":"voter1","identifier":...,"timestamp":...,"ancillary_data":"<stamped>","price":0,"salt":<salt>}' -p voter1@active
```

{% hint style="warning" %}
The DVM request's ancillary data is the **stamped** version:

* OOv3: `assertionId:<hex64>,ooAsserter:<name>`
* OOv2: `<original ancillary_data>disputed by <requester> against <proposer>`

Use the stamped bytes when committing/revealing, and when computing the DVM `request_id`.
{% endhint %}

## Resolve and settle

After the reveal phase, run the cranks:

```bash
cleos push action <voting> processreqs '{"max_requests":50}' -p keeper@active
cleos push action <voting> slashbatch  '{"request_id":<dvm_req_id>,"round_id":<round>,"max_rows":100}' -p keeper@active
cleos push action <voting> rewardbatch '{"request_id":<dvm_req_id>,"round_id":<round>,"max_rows":100}' -p keeper@active
```

Then settle the original answer, which reads the DVM result and pays the winner:

```bash
# OOv3
cleos push action <oracle> settleassert '{"assertion_id":<id>}' -p keeper@active
# OOv2
cleos push action <oracle> settle '{"requester":"myapp","identifier":...,"timestamp":...,"ancillary_data":"..."}' -p keeper@active
```

## Verify the outcome

* **OOv3:** read the `assertions` row — `settled == true` and `settlement_resolution` reflects the DVM vote (`true` if voters chose `10000`).
* **OOv2:** read the `requests` row — `settled == true` and `resolved_price` is the DVM price.
* **Stakers:** correct voters were credited a share of the slashed pool; wrong/absent voters were slashed.

## Tips

* If a round fails quorum, the request **rolls** — vote again in the next round (up to `max_rolls`).
* Keep your commit `salt` until you've revealed.
* If you're scripting this, the repository's testnet scripts (e.g. a dispute test harness) automate the same sequence.
