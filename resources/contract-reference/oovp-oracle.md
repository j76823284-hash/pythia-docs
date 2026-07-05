# oovp.oracle

The optimistic oracle. Hosts both the **OOv2** (request → propose → dispute → settle) and **OOv3** (assert → dispute → settle) flows, and escalates disputes to `oovp.voting`.

## Memo conventions (token transfers to the oracle)

| Memo | Purpose |
|---|---|
| `request:<request_id>` | Escrow the **reward** for an OOv2 request. |
| `bond:<request_id>` | Deposit **bond + final fee** for an OOv2 propose/dispute. |
| `assert` | Deposit a bond for an OOv3 assert/dispute. |

Deposited tokens must be whitelisted collateral. Deposits are consumed by the corresponding action; unused deposits are recoverable (`reclaimreq` / `reclaimbond` / `reclaimasrt`).

## OOv2 actions

| Action | Auth | Description |
|---|---|---|
| `requestprice(requester, identifier, timestamp, ancillary_data, currency, reward)` | requester | Open a price request; reward pre-escrowed via `request:<id>`. |
| `setbond(requester, …, bond)` | requester | Override bond (while `REQUESTED`). |
| `setliveness(requester, …, liveness)` | requester | Override liveness (30 s–7 d). |
| `setrefund(requester, …)` | requester | Refund reward to requester on dispute. |
| `setcallbacks(requester, …, on_proposed, on_disputed, on_settled)` | requester | Enable callbacks. |
| `seteventbase(requester, …)` | requester | Mark request event-based. |
| `proposeprice(proposer, requester, identifier, timestamp, ancillary_data, proposed_price)` | proposer | Answer a request (bond+fee pre-deposited). |
| `disputeprice(disputer, requester, identifier, timestamp, ancillary_data)` | disputer | Dispute a proposal; escalate to DVM. |
| `settle(requester, identifier, timestamp, ancillary_data)` | anyone | Settle (EXPIRED or DVM-RESOLVED). |
| `notifysettled(requester, …)` | anyone | Deliver the `pricesettled` callback. |
| `cancel(requester, identifier, timestamp, ancillary_data)` | requester | Cancel a stuck `REQUESTED` request after 7 days; refund reward. |

**State machine:** `INVALID → REQUESTED → PROPOSED → (EXPIRED | DISPUTED) → RESOLVED → SETTLED`.

## OOv3 actions

| Action | Auth | Description |
|---|---|---|
| `asserttruth(asserter, claim, callback_recipient, escalation_manager, liveness, currency, bond, identifier, domain_id)` | asserter | Post a bonded claim. `identifier=0` ⇒ `ASSERT_TRUTH`. |
| `assertwdflt(asserter, claim, callback_recipient)` | asserter | Assert with protocol defaults (default liveness/currency, min bond). |
| `dispassert(disputer, assertion_id)` | disputer | Dispute an active assertion; escalate to DVM. |
| `settleassert(assertion_id)` | anyone | Settle after liveness or DVM resolution. |
| `notifyassert(assertion_id)` | anyone | Deliver the `assertresvd` callback. |
| `pruneasrt(max_rows, min_age_secs)` | anyone | Reclaim RAM from terminal settled assertions (`min_age_secs ≥ 300`). |
| `syncparams(identifier, currency)` | anyone | Refresh the whitelist/final-fee cache. |

**Assertion ID:** `sha256(asserter(8,LE) ‖ claim ‖ block_time(8,LE))`; primary key = first 8 bytes (LE).

## Admin actions

| Action | Description |
|---|---|
| `setconfig(finder, voting, store)` | Wire the core contracts. |
| `setdefault(liveness)` | Default liveness (default 7200). |
| `setlimit(bytes)` | Ancillary/claim size cap (default 8192). |
| `setburnedbps(bps)` | Burned-bond rate (default 5000; 0 ⇒ default). |
| `setdefcur(currency)` | Default currency for `assertwdflt` (must be whitelisted). |
| `setfeevault(vault)` | Deposit-only wallet receiving settlement fees. |

## Deposit-recovery actions

| Action | Auth | Recovers |
|---|---|---|
| `reclaimreq(depositor, request_id)` | depositor | Unused/excess request (reward) escrow. |
| `reclaimbond(depositor, request_id)` | depositor | Unconsumed OOv2 bond deposit. |
| `reclaimasrt(depositor)` | depositor | Unconsumed OOv3 assert deposit. |

## Callbacks fired to integrators

| Callback (action on recipient) | Args | Fired |
|---|---|---|
| `priceproposed` / `pricedisputed` / `pricesettled` | `(identifier, timestamp, ancillary_data, price)` | OOv2, if enabled via `setcallbacks`. |
| `assertdisp` | `(assertion_id)` | On `dispassert`, if a recipient is set. |
| `assertresvd` | `(assertion_id, resolution)` | Via `notifyassert` after settlement. |

## Key tables (readable)

| Table | Scope | Contents |
|---|---|---|
| `requests` | oracle | OOv2 price requests (state, proposer, disputer, prices, bonds). |
| `assertions` | oracle | OOv3 assertions (claim, asserter, disputer, bond, settled, resolution). |
| `oracleconfig` | oracle | Singleton: liveness, limits, wired contracts, burned bps, default currency. |
| `cachedcurrns` / `cachedidents` | oracle | Synced whitelist + final-fee cache. |

## Static getters

```cpp
bool     oracle::has_price(oracle, requester, id, ts, anc);
int128_t oracle::get_price(oracle, requester, id, ts, anc);
asset    oracle::getminbond(oracle, currency);   // minBond = finalFee × 10000 / burnedBps
```

See [Bonds, Fees & Liveness](../../developers/bonds-fees-and-liveness.md) for the settlement math.
