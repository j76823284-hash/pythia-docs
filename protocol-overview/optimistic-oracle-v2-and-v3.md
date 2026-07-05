# Optimistic Oracle: v2 and v3

Pythia ships **both** oracle flows in a single contract, `pythiaoorcle`. They share the same dispute path and DVM backstop but differ in how an answer is produced.

{% hint style="info" %}
Both flows live in `pythiaoorcle`. You do not deploy anything to use them — you register your contract in `<finder>` and call the relevant actions. See [Registering a Contract](../using-pythia/registering-a-contract.md).
{% endhint %}

## OOv2 — request then propose

OOv2 is a two-party flow. It suits applications with an **open set of proposers** answering requests.

### State machine

```
INVALID → REQUESTED → PROPOSED → (EXPIRED | DISPUTED) → RESOLVED → SETTLED
```

### Lifecycle

1. **`requestprice`** — a registered requester opens a request for `(identifier, timestamp, ancillary_data, currency, reward)`. The reward is pre-escrowed to the oracle (token transfer with memo `request:<request_id>`). The requester does **not** post the final fee — that is posted by proposers/disputers.
2. **Optional configuration** — before a proposal arrives, the requester may call `setbond`, `setliveness`, `setcallbacks`, `setrefund`, or `seteventbase` to tune the request.
3. **`proposeprice`** — a proposer answers with a price and posts **bond + final fee** (transfer with memo `bond:<request_id>`). This starts the liveness clock.
4. **Challenge window** — during liveness anyone can **`disputeprice`** by posting a matching bond + final fee. This escalates the request to the DVM.
5. **Settlement** — `settle` is permissionless:
   * **Undisputed (EXPIRED):** the proposer is right; they recover their bond + final fee plus the reward.
   * **Disputed (RESOLVED):** the oracle reads the DVM's resolved price. The side that matches it wins, taking back both bonds minus the oracle fee, plus the reward. See [Bonds, Fees & Liveness](../developers/bonds-fees-and-liveness.md) for the exact math.

### Request parameters

Requests can be customized while in the `REQUESTED` state:

| Action | Effect |
|---|---|
| `setbond` | Override the proposer/disputer bond (default: 2× final fee). |
| `setliveness` | Override the challenge window (bounded by min/max liveness). |
| `setrefund` | Refund the reward to the requester if the request is disputed. |
| `setcallbacks` | Fire `priceproposed` / `pricedisputed` / `pricesettled` callbacks to the requester. |
| `seteventbase` | Mark the request event-based (timestamp `0`). |

## OOv3 — assert

OOv3 collapses request and proposal into one bonded **assertion**. It suits applications where the integrator (or a bot) both produces and stands behind the answer.

### Lifecycle

1. **Deposit** — the asserter transfers the bond to the oracle with memo `assert`.
2. **`asserttruth`** — the asserter posts the claim in one action, choosing `currency`, `bond`, `liveness`, `identifier`, an optional `callback_recipient`, an optional `escalation_manager`, and a `domain_id`. `assertwdflt` is a convenience wrapper using protocol defaults (default liveness, default currency, minimum bond, the `ASSERT_TRUTH` identifier).
3. **Challenge window** — anyone can **`dispassert`** during liveness by posting a matching bond (memo `assert`). This escalates to the DVM and fires an `assertdisp` callback if one is registered.
4. **`settleassert`** — permissionless:
   * **Undisputed:** after liveness, the asserter recovers their full bond with **no fee**.
   * **Disputed:** the oracle reads the DVM result. `ASSERT_TRUTH` disputes resolve to `NUMERICAL_TRUE` (10000) or `NUMERICAL_FALSE` (0). The winner takes both bonds minus the oracle fee.
5. **`notifyassert`** — delivers the resolution callback (`assertresvd`) to the callback recipient without blocking settlement.

### The assertion ID

Each assertion is keyed by:

```
assertion_id = sha256( asserter(8 bytes, LE) ‖ claim ‖ block_timestamp(8 bytes, LE) )
```

The first 8 bytes (little-endian) of that hash are the table primary key. When a dispute is escalated, the DVM request is stamped:

```
assertionId:<hex64>,ooAsserter:<name>
```

so voters and off-chain tooling can unambiguously parse the assertion context.

## Choosing a flow

| Question | Use OOv2 | Use OOv3 |
|---|---|---|
| Who produces the answer? | A third-party proposer answers your request. | You (or your bot) assert the answer yourself. |
| Do you need an open proposer market? | Yes. | No. |
| Do you want a reward paid to the answerer? | Yes — set a reward. | No — the asserter is not paid a reward; they simply recover their bond. |
| Do you need pluggable arbitration? | No. | Yes — attach an escalation manager. |
| Typical products | Prediction markets, insurance, event resolution. | Data assertions, governance approval, self-resolving markets. |

{% hint style="success" %}
Pythia's own **governance** contract uses OOv3: `pythiagovern::propose` submits the proposal as an assertion and checks that it settled `true` before allowing execution. See [On-Chain Proposals](../community/governance/dao-proposals.md).
{% endhint %}

## Shared parameter cache

To keep `asserttruth` cheap, the oracle caches the collateral whitelist, final-fee schedule, and identifier whitelist on-chain. After adding a currency or identifier in `<finder>`/`pythiastore1`, call the permissionless **`syncparams`** action to refresh the cache. See the [Deployment & Setup Checklist](../resources/deployment-checklist.md).
