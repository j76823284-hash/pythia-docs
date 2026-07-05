# Making an Assertion (OOv3)

An **assertion** is a single-action, bonded claim about the state of the world. If nobody disputes it before its liveness expires, it is accepted as true; if disputed, the DVM decides. This is the OOv3 flow, implemented in `pythiaoorcle`.

{% hint style="info" %}
Prerequisite: your account should exist and your bond currency must be whitelisted collateral with a configured final fee. See [Registering a Contract](registering-a-contract.md).
{% endhint %}

## Step 1 — Deposit the bond

Antelope has no ERC-20-style `approve`. Instead you **pre-deposit** the bond by transferring it to the oracle with the memo **`assert`**:

```bash
cleos push action <token> transfer \
  '["myasserter","<oracle>","2.0000 PYTHIA","assert"]' \
  -p myasserter@active
```

The oracle records the deposit under your account. The deposit must be **at least the minimum bond** for that currency (see below). You can top up with additional `assert` transfers.

## Step 2 — Assert the truth

Call `asserttruth`, which consumes your deposit and opens the assertion:

```bash
cleos push action <oracle> asserttruth '{
  "asserter":"myasserter",
  "claim":"54686520424943207072696365206f6e20323032362d30372d3031...",
  "callback_recipient":"",
  "escalation_manager":"",
  "liveness":7200,
  "currency":{"sym":"4,PYTHIA","contract":"<token>"},
  "bond":"2.0000 PYTHIA",
  "identifier":0,
  "domain_id":0
}' -p myasserter@active
```

| Field | Meaning |
|---|---|
| `claim` | The fact being asserted, as bytes (hex in cleos). Keep it self-describing and ≤ 8 KB. |
| `callback_recipient` | Contract to notify on dispute (`assertdisp`) and resolution (`assertresvd`). Empty = none. |
| `escalation_manager` | Optional pluggable arbitration policy. Empty = use the DVM directly. |
| `liveness` | Challenge window in seconds (bounded by min/max liveness). |
| `currency` | Bond token (whitelisted collateral). |
| `bond` | Bond amount; must be ≥ the minimum bond. |
| `identifier` | DVM identifier; `0` selects `ASSERT_TRUTH`. |
| `domain_id` | Optional grouping for escalation games; `0` = none. |

### The convenience wrapper

If you want protocol defaults (default liveness, the default currency, the minimum bond, and the `ASSERT_TRUTH` identifier), use `assertwdflt` — you still deposit the minimum bond first:

```bash
cleos push action <oracle> assertwdflt \
  '{"asserter":"myasserter","claim":"...","callback_recipient":""}' \
  -p myasserter@active
```

## Step 3 — Find your assertion ID

The assertion's primary key is the first 8 bytes (little-endian) of:

```
sha256( asserter(8, LE) ‖ claim ‖ block_timestamp(8, LE) )
```

Read it back from the oracle's `assertions` table (scope = oracle account) to get the `id` and full `assertion_id`:

```bash
cleos get table <oracle> <oracle> assertions --index 2 --key-type i64 -L myasserter
```

## Step 4 — Settle

After liveness expires with no dispute, anyone can settle. You recover your **full bond** with no fee:

```bash
cleos push action <oracle> settleassert '{"assertion_id":1234567890}' -p anyone@active
```

If a `callback_recipient` was set, `notifyassert` delivers the `assertresvd(assertion_id, resolution)` callback without blocking settlement.

## The minimum bond

The minimum bond is derived from the final fee and the burned-bond rate:

$$
\text{minBond} = \text{finalFee} \times \frac{10000}{\text{burnedBondBps}}
$$

At the default 50% burn rate this is **2 × finalFee**. Read it directly:

```bash
# getminbond is a static getter; integrators call it cross-contract.
# The value is also derivable from the store final fee and setburnedbps.
```

If your assertion is disputed and you win, you receive **2 × bond − oracle fee**; if you lose, you forfeit the bond. See [Bonds, Fees & Liveness](../developers/bonds-fees-and-liveness.md).

## What happens on dispute

If someone calls `dispassert` before liveness expires, the assertion escalates to the DVM. See [Disputing](disputing.md). Resolution is binary — the DVM votes `NUMERICAL_TRUE` or `NUMERICAL_FALSE` — and `settleassert` pays the winner.
