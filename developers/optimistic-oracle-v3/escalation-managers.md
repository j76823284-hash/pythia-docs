# Escalation Managers

An **escalation manager (EM)** is a pluggable policy contract that customizes how an assertion is arbitrated and who is allowed to dispute it. `asserttruth` accepts an `escalation_manager` account and a `domain_id`, and the oracle reserves per-assertion policy flags for EM integration.

{% hint style="warning" %}
**Implementation status.** The `escalation_manager` parameter and the policy fields (`arbitrate_via_em`, `discard_oracle`, `validate_disputers`) are present on every assertion row, but the shipped dispute path **always routes to the DVM**. Passing an empty `escalation_manager` (the default) is fully supported and is the intended configuration today. Custom EM arbitration is a forward-looking extension — treat this page as the design and interface, not a description of active custom-EM behavior on mainnet.
{% endhint %}

## The concept

In UMA's OOv3, an escalation manager can override default behavior to:

* **Gate proposers/disputers** — restrict who may assert or dispute (`validate_disputers`).
* **Choose the arbitrator** — resolve via a custom mechanism instead of the DVM (`arbitrate_via_em`).
* **Discard the oracle result** — ignore the DVM outcome under a custom policy (`discard_oracle`).
* **Group assertions** — the `domain_id` lets an EM treat a set of assertions as one escalation game.

Pythia mirrors this interface: each assertion stores `escalation_manager`, `domain_id`, and the three policy flags, so a future EM implementation can read them per-assertion.

## How it maps to the contract

```cpp
TABLE assertion {
    // ...
    name     escalation_manager;   // pluggable EM (empty = DVM)
    uint64_t domain_id;            // grouping for escalation games
    bool     arbitrate_via_em;     // reserved: resolve via EM instead of DVM
    bool     discard_oracle;       // reserved: ignore DVM result
    bool     validate_disputers;   // reserved: gate who may dispute
    // ...
};
```

Today `asserttruth` records the `escalation_manager` and `domain_id` you pass and initializes the three flags to `false`, so:

* Disputes escalate to the **DVM** (`pythiavoting`).
* Anyone may dispute during liveness.
* The DVM result is authoritative.

## Recommended usage today

* **Leave `escalation_manager` empty** (`""`) to use the default DVM arbitration. This is the audited, exercised path.
* Use `domain_id` only as an off-chain grouping hint if it helps your application; it does not change on-chain resolution today.

## Designing for a future EM

If you are building an application that may adopt a custom EM later, keep these principles in mind so migration is smooth:

* Treat the DVM's `settlement_resolution` as the source of truth in your callbacks, and key everything off the `assertion_id`.
* Don't assume a specific disputer set; validate authorization in your own contract if you need to restrict who interacts with your assertions.
* Keep claims self-describing so that whatever arbitrator resolves them (DVM today, custom EM later) has enough context.

When custom EM arbitration ships, the policy flags above will be populated from the named `escalation_manager` at assertion time, and `dispassert` / `settleassert` will consult it. Until then, the DVM is the arbitrator for every assertion.
