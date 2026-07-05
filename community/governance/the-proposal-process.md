# The Proposal Process

A Pythia governance proposal moves through four stages: **propose → bond → assert & vote → execute**. This mirrors UMA's "propose to the oracle, execute if approved" model, adapted to Antelope inline actions and the OOv3 assertion flow.

## 1. Propose

Only the configured **proposer contract** may create proposals (`propose` checks `proposer == proposer_contract`). This is set once by governance via `setproposer` and cannot be zeroed afterward. A proposal bundles:

* `transactions` — 1 to 10 concrete on-chain actions to execute if approved.
* `ancillary_data` — a human-readable description voters can reference.

```bash
cleos push action <govern> propose '{
  "proposer":"<proposer_contract>",
  "transactions":[ { "to":"<store>", "action_name":"setfinalfee",
                     "authorization":[{"actor":"<store>","permission":"exec"}],
                     "data":"<serialized args>" } ],
  "ancillary_data":"Raise WAXUSDC final fee to 2.0000"
}' -p <proposer_contract>@active
```

The proposal ID is derived from `sha256(proposer ‖ block_time)`, and the exact `transactions` bytes are hashed and stored — voters approve **that specific hash**, and execution re-checks the bytes match.

## 2. Bond

The proposal is inert until the bond is funded. The proposer transfers the **minimum bond** in PYTHIA to `pythiagovern` with memo `proposal:<proposal_id>`:

```bash
cleos push action <token> transfer \
  '["<proposer_contract>","<govern>","500.0000 PYTHIA","proposal:1234567890"]' \
  -p <proposer_contract>@active
```

When the bond is fully funded, `pythiagovern`'s transfer handler automatically:

1. Forwards the bond to `pythiaoorcle` (memo `assert`).
2. Submits the proposal as an **`asserttruth`** assertion (asserter = the governor), using the configured voting period as liveness.
3. Records the assertion's primary key and sets the **timelock** (`executable_after = now + voting_period + execution_delay`).

Over-deposits are refunded automatically.

## 3. Assert & vote

The proposal is now a live oracle assertion:

* If **no one disputes** it before the voting period (liveness) elapses, it settles **true** optimistically.
* If a staker **disputes** it, it escalates to a DVM vote. Voters resolve it `NUMERICAL_TRUE` (approve) or `NUMERICAL_FALSE` (reject).

`pythiagovern::is_approved` reads the oracle's `assertions` table by the stored key and returns true only if the assertion is **settled and resolved true**, the asserter is the governor, and the stored claim matches — a pending or rejected assertion is not approved.

## 4. Execute

After the assertion settles true **and** the timelock passes, execution is permissionless:

```bash
cleos push action <govern> execute '{"proposal_id":1234567890}' -p anyone@active
```

`execute`:

* Re-verifies the `transactions` bytes hash to the approved hash.
* Executes each action. Every governance transaction must carry exactly one authorization of the form **`target@exec`** — governed contracts expose sensitive actions under an `exec` permission that only the governor can use.
* Returns the bond to the proposer.

If the proposal was rejected or never approved, `execute` fails and the bond is not returned (it was consumed as the assertion bond and forfeited on a lost dispute).

## Cancelling

A proposal can be cancelled **before** its assertion is submitted (i.e. before the bond is fully funded) by the proposer or the governor, refunding any partial bond:

```bash
cleos push action <govern> cancel '{"proposal_id":1234567890}' -p <proposer_contract>@active
```

Once the oracle assertion has been submitted, the proposal can no longer be cancelled — it must run its course.

## Emergency path

For critical fixes, an **emergency proposer** (a multisig, set via `setemergency`) can bypass the DVM vote — but only within tight limits:

* Every target must be **allowlisted** in advance via `setemergtgt(target, true)`.
* A minimum **emergency delay** must elapse since the proposal was created (`emergency_delay`, ≥ 1 hour, configurable up to 7 days).
* The `transactions` bytes must match the stored hash.

```bash
cleos push action <govern> emergexec '{"proposal_id":1234567890}' -p <emergency_proposer>@active
```

This path exists so a compromised parameter can be fixed quickly, while the allowlist + delay keep it from becoming a governance backdoor.

## Timelock summary

| Delay | Set by | Purpose |
|---|---|---|
| `voting_period` | `setvoteperiod` (1–30 days) | Assertion liveness / DVM voting window. |
| `execution_delay` | `setexecdelay` | Extra timelock after approval before `execute` is allowed. |
| `emergency_delay` | `setemrgdelay` (1 hour–7 days) | Minimum age before `emergexec` may run. |
