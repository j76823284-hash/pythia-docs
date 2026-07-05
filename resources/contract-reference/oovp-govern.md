# pythiagovern

Governance. Proposals are submitted as OOv3 oracle assertions; if the assertion settles truthful, the proposal's bundled transactions become executable after a timelock.

## Memo conventions (token transfers to the govern contract)

| Memo | Purpose |
|---|---|
| `proposal:<proposal_id>` | Fund a proposal's bond. When fully funded, the governor auto-submits the oracle assertion. |

## Actions

| Action | Auth | Description |
|---|---|---|
| `propose(proposer, transactions[], ancillary_data)` | proposer (== configured proposer contract) | Create a proposal (1–10 transactions). |
| `execute(proposal_id)` | anyone | Execute an approved proposal after the timelock. |
| `emergexec(proposal_id)` | emergency proposer | Execute via the emergency path (allowlisted targets, after delay). |
| `cancel(proposal_id)` | proposer or self | Cancel before the assertion is submitted; refund partial bond. |

## Admin actions

| Action | Description |
|---|---|
| `setconfig(finder)` | Wire the finder (used to resolve oracle + voting token). |
| `setproposer(account)` | Set the authorized proposer contract (cannot be zeroed once set). |
| `setemergency(account)` | Set the emergency proposer (e.g. a multisig). |
| `setminbond(bond)` | Minimum proposal bond. |
| `setvoteperiod(seconds)` | Assertion liveness / DVM voting window (1–30 days). |
| `setexecdelay(seconds)` | Timelock after approval before `execute` is allowed. |
| `setemrgdelay(seconds)` | Minimum age before `emergexec` (1 h–7 d). |
| `setemergtgt(target, allowed)` | Allowlist a target for emergency execution. |

## Proposal transaction shape

Each transaction must be authorized **`{actor: to, permission: "exec"}`**:

```cpp
struct transaction_data {
    name                          to;             // target contract
    name                          action_name;    // action to call
    std::vector<permission_level> authorization;  // exactly [{to, "exec"}]
    std::vector<char>             data;            // serialized args
};
```

## Approval semantics

`is_approved(proposal_id)` returns true only when the proposal's oracle assertion is **`settled && settlement_resolution`**, the asserter is the governor, and the stored claim still matches the proposal. `execute` additionally re-verifies the `transactions` bytes hash to the approved hash.

## Timelock

`executable_after = assertion_time + voting_period + execution_delay`. `execute` enforces `now ≥ executable_after`.

## Key tables (readable)

| Table | Contents |
|---|---|
| `proposals` | Proposals: transactions, hashes, bond, `dvm_request_id` (assertion PK), `executable_after`. |
| `emergprops` | Emergency execution records. |
| `emergtargets` | Allowlisted emergency targets. |
| `govconfig` | Singleton: wired contracts, proposer, emergency proposer, bond, periods, delays. |

See [The Proposal Process](../../community/governance/the-proposal-process.md) and [On-Chain Proposals](../../community/governance/dao-proposals.md).
