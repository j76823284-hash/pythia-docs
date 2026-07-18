---
description: Resolve an OOv2 numeric request.
---

# Request and settle a value

Use the OOv2 value-request flow when an application needs a numeric answer, such as a price or measured value. Use a truth assertion instead when the question is simply whether a claim is true.

## 1. Define one immutable request

A request is identified by its requester, identifier, timestamp, and ancillary bytes. Preserve these exact values for every later action. Changing even one field refers to a different request.

Generate the request ID with the project's canonical helper or the contract's hash algorithm. Do not invent an ID: the Oracle verifies the full request hash as well as its compact primary key.

## 2. Deposit the requester reward

Transfer the reward token to the Oracle with this memo:

```
request:<request_id>
```

The transfer must use a whitelisted extended symbol. The first deposit must meet the configured final-fee floor; `requestprice` then consumes the requested reward from this escrow.

## 3. Open the request

Call `requestprice` as the requester with the same identifier, timestamp, ancillary data, currency, and reward used to define the request. While the request remains unproposed, the requester may set a custom bond, liveness, callbacks, event-based behaviour, or refund-on-dispute behaviour.

## 4. Propose a value

The proposer transfers the required bond plus final fee to the Oracle with:

```
bond:<request_id>
```

The proposer then calls `proposeprice` with the unchanged request fields and the numeric value. The proposal starts the request's liveness period.

## 5. Settle or dispute

* For an undisputed proposal, call `settle` after liveness expires.
* To challenge it before expiry, the disputer makes the same `bond:<request_id>` deposit and calls `disputeprice`.
* A dispute opens the DVM request automatically. After the DVM resolves it, call `settle` with the original request fields.

`settle` is permissionless. Read the `requests` row or use `oracle::has_price` before allowing a dependent contract to use the result.

## Recover unused funds

Use `reclaimreq` for unused or excess request escrow and `reclaimbond` for an orphaned bond deposit. A requester can use `cancel` only for an unproposed request that has passed the configured minimum cancellation age.
