---
description: Actions, tables, deposits, callbacks, and constants.
---

# Oracle/DVM API reference

This is the reference for the current `oovp.oracle` and `oovp.voting` ABIs. Names and symbols in a deployed network are configuration, not constants in this document.

## Shared terms

| Term                       | Meaning                                                                                  |
| -------------------------- | ---------------------------------------------------------------------------------------- |
| `extended_symbol`          | A token symbol plus its issuing contract. Both must match a configured collateral entry. |
| `ancillary_data` / `claim` | `vector<char>` bytes. Encode text deterministically, normally as UTF-8.                  |
| `identifier`               | A Finder-whitelisted unsigned identifier that defines the requested data type.           |
| liveness                   | The period in seconds in which a proposed answer or assertion can be disputed.           |
| `int128_t` price           | A signed integer. The application, identifier, and ancillary data define its scale.      |

## Deposits and memo formats

The Oracle accepts a transfer only when it is sent to the Oracle contract, originates from a whitelisted token contract, has a positive quantity, and uses one of these memos.

| Memo                   | Used before                                   | Purpose                             |
| ---------------------- | --------------------------------------------- | ----------------------------------- |
| `request:<request_id>` | `requestprice`                                | Escrows an OOv2 requester reward.   |
| `bond:<request_id>`    | `proposeprice` or `disputeprice`              | Escrows an OOv2 bond and final fee. |
| `assert`               | `asserttruth`, `assertwdflt`, or `dispassert` | Escrows an OOv3 assertion bond.     |

Deposits are consumed by their follow-up action. Use the reclaim actions when a valid deposited balance remains unused.

## `oovp.oracle`

### OOv2 value requests

| Action         | Parameters                                                                               | Authority | Effect                                                                            |
| -------------- | ---------------------------------------------------------------------------------------- | --------- | --------------------------------------------------------------------------------- |
| `requestprice` | `requester, identifier, timestamp, ancillary_data, currency, reward`                     | requester | Creates a numeric request after its reward deposit.                               |
| `setbond`      | `requester, identifier, timestamp, ancillary_data, bond`                                 | requester | Sets a custom request bond while the request is unproposed.                       |
| `setrefund`    | `requester, identifier, timestamp, ancillary_data`                                       | requester | Enables reward refund on dispute while unproposed.                                |
| `setliveness`  | `requester, identifier, timestamp, ancillary_data, liveness`                             | requester | Sets request liveness while unproposed.                                           |
| `setcallbacks` | `requester, identifier, timestamp, ancillary_data, on_proposed, on_disputed, on_settled` | requester | Configures OOv2 callback flags.                                                   |
| `seteventbase` | `requester, identifier, timestamp, ancillary_data`                                       | requester | Marks an unproposed request as event based.                                       |
| `proposeprice` | `proposer, requester, identifier, timestamp, ancillary_data, proposed_price`             | proposer  | Records a proposed numeric answer after the proposer deposit.                     |
| `disputeprice` | `disputer, requester, identifier, timestamp, ancillary_data`                             | disputer  | Challenges an active proposal after the disputer deposit and opens a DVM request. |
| `settle`       | `requester, identifier, timestamp, ancillary_data`                                       | anyone    | Settles an expired undisputed request or a DVM-resolved dispute.                  |

### OOv3 truth assertions

| Action         | Parameters                                                                                                 | Authority | Effect                                                                                                                    |
| -------------- | ---------------------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------- |
| `asserttruth`  | `asserter, claim, callback_recipient, escalation_manager, liveness, currency, bond, identifier, domain_id` | asserter  | Creates an assertion and consumes the asserter's `assert` deposit. `identifier` `0` selects the default truth identifier. |
| `assertwdflt`  | `asserter, claim, callback_recipient`                                                                      | asserter  | Creates an assertion using configured defaults.                                                                           |
| `dispassert`   | `disputer, assertion_id`                                                                                   | disputer  | Challenges an active assertion and consumes a matching `assert` deposit.                                                  |
| `settleassert` | `assertion_id`                                                                                             | anyone    | Settles an expired undisputed assertion or a DVM-resolved assertion.                                                      |

For a default truth request, DVM outcomes are `10000` for true and `0` for false. The `assertions` table's `id` is the `assertion_id` accepted by the action; the row also holds the full SHA-256 identifier for collision checking.

### Delivery, recovery, and operations

| Action          | Parameters                                         | Authority | Effect                                                                          |
| --------------- | -------------------------------------------------- | --------- | ------------------------------------------------------------------------------- |
| `notifysettled` | `requester, identifier, timestamp, ancillary_data` | anyone    | Delivers a previously settled OOv2 callback.                                    |
| `notifydisput`  | `requester, identifier, timestamp, ancillary_data` | anyone    | Delivers a previously disputed OOv2 callback.                                   |
| `notifyassert`  | `assertion_id`                                     | anyone    | Delivers an OOv3 resolution callback.                                           |
| `notifyadisp`   | `assertion_id`                                     | anyone    | Delivers an OOv3 dispute callback.                                              |
| `claimpayout`   | `payout_id`                                        | anyone    | Delivers one credited Oracle token payout.                                      |
| `cancel`        | `requester, identifier, timestamp, ancillary_data` | requester | Cancels an unproposed OOv2 request after the minimum cancellation age.          |
| `reclaimbond`   | `depositor, request_id`                            | depositor | Reclaims an unused OOv2 bond deposit.                                           |
| `reclaimreq`    | `depositor, request_id`                            | depositor | Reclaims unused or excess OOv2 reward escrow.                                   |
| `reclaimasrt`   | `depositor`                                        | depositor | Reclaims an unused OOv3 assertion deposit.                                      |
| `pruneasrt`     | `max_rows, min_age_secs`                           | anyone    | Deletes eligible terminal assertions in a bounded batch.                        |
| `syncparams`    | `identifier, currency`                             | anyone    | Refreshes the OOv3 collateral, fee, and identifier cache from Finder and Store. |

### Oracle administration

| Action         | Parameters                                         | Authority       | Effect                                                                                                        |
| -------------- | -------------------------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------- |
| `setdefault`   | `liveness`                                         | Oracle contract | Sets default liveness.                                                                                        |
| `setlimit`     | `bytes_limit`                                      | Oracle contract | Sets the ancillary-data size limit.                                                                           |
| `setburnedbps` | `bps`                                              | Oracle contract | Sets the OOv3 disputed-bond fee rate in basis points. A changed value is blocked while disputed work is open. |
| `setdefcur`    | `currency`                                         | Oracle contract | Sets the currency used by `assertwdflt`.                                                                      |
| `setfeevault`  | `vault`                                            | Oracle contract | Sets the deposit-only fee-vault recipient.                                                                    |
| `setconfig`    | `finder_contract, voting_contract, store_contract` | Oracle contract | Sets core dependency accounts.                                                                                |

### Oracle reads and tables

`oracle::has_price` reports whether an OOv2 request is settled. `oracle::get_price` returns its settled value. `oracle::getminbond` returns the current OOv3 minimum bond for a currency. `oracle::get_ancillary_limit` returns the configured byte limit.

| Table                                       | Purpose                                                                                                     |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `requests`                                  | OOv2 requests, proposal/dispute parties, escrow amounts, settings, and settlement state.                    |
| `assertions`                                | OOv3 claim, parties, bond, liveness, full identifier, and settlement state. Indexed by asserter and expiry. |
| `bonddeposits`, `requestdeps`, `assertdeps` | Pending deposits consumed by a follow-up action or reclaimed by the depositor.                              |
| `cachedcurrns`, `cachedidents`              | The `syncparams` cache for OOv3 validation and minimum-bond calculation.                                    |
| `payouts`                                   | Credited token deliveries that can be sent with `claimpayout`.                                              |
| `oracleconfig`, `oraclext`                  | Oracle configuration and extension state.                                                                   |

## `oovp.voting`

### Request intake and voting

| Action         | Parameters                                                  | Authority         | Effect                                                                                                                   |
| -------------- | ----------------------------------------------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `requestprice` | `requester, identifier, timestamp, ancillary_data`          | requester         | Queues a DVM request for the next round. With the standard launch configuration, only the registered Oracle may call it. |
| `commitvote`   | `voter, identifier, timestamp, ancillary_data, hash`        | voter or delegate | Stores a commitment during commit phase.                                                                                 |
| `batchcommit`  | `voter, commits`                                            | voter or delegate | Commits up to 50 requests in one call.                                                                                   |
| `revealvote`   | `voter, identifier, timestamp, ancillary_data, price, salt` | voter or delegate | Reveals a matching commitment during reveal phase.                                                                       |
| `batchreveal`  | `voter, reveals`                                            | voter or delegate | Reveals multiple committed votes.                                                                                        |
| `processreqs`  | `max_requests`                                              | anyone            | Resolves or rolls eligible requests after reveal; `max_requests` is at most 100.                                         |

A commitment is the SHA-256 hash of the price, salt, voter, timestamp, round ID, and identifier in the contract's canonical encoding. Generate it with the project helper or a byte-for-byte compatible implementation. Do not construct it by concatenating display strings.

### DVM administration and operations

| Action         | Parameters                                        | Authority       | Effect                                                                                                          |
| -------------- | ------------------------------------------------- | --------------- | --------------------------------------------------------------------------------------------------------------- |
| `setconfig`    | `finder_contract, stake_contract, token_contract` | Voting contract | Sets core dependency accounts.                                                                                  |
| `setphaselen`  | `phase_length`                                    | Voting contract | Sets the duration in seconds of each commit and reveal phase.                                                   |
| `setthreshold` | `gat_percentage, spat_bps`                        | Voting contract | Sets participation and agreement thresholds. GAT uses the contract's `100000000` scale; SPAT uses basis points. |
| `setmaxrolls`  | `max_rolls`                                       | Voting contract | Limits the number of rounds a request may roll.                                                                 |
| `slashbatch`   | `request_id, round_id, max_rows`                  | anyone          | Performs bounded incorrect-vote slashing and correct-weight tallying.                                           |
| `rewardbatch`  | `request_id, round_id, max_rows`                  | anyone          | Performs bounded distribution of the slash pool.                                                                |
| `pruneround`   | `round_id, max_rows`                              | anyone          | Prunes vote submissions for a completed round.                                                                  |
| `prunefreqs`   | `round_id, max_rows`                              | anyone          | Prunes vote frequencies for a completed round.                                                                  |

`slashbatch`, `rewardbatch`, `pruneround`, and `prunefreqs` require `max_rows` from 1 through 100.

### Voting reads and tables

`voting::has_price` reports whether a DVM request resolved. `voting::get_price` returns its final value. `voting::get_current_round` and `voting::get_current_phase` expose the schedule; phase `0` is commit and phase `1` is reveal.

| Table                        | Purpose                                                    |
| ---------------------------- | ---------------------------------------------------------- |
| `pricereqs`                  | Active and resolved DVM requests.                          |
| `voteinstance`               | Per-request, per-round totals and resolution state.        |
| `votesubmits`                | Individual commitments, reveals, and recorded vote weight. |
| `votefreqs`                  | Stake-weighted frequencies used to calculate an outcome.   |
| `roundconfigs`, `roundsnaps` | Per-round thresholds, deadlines, and frozen total stake.   |
| `slashjobs`, `pendrewards`   | In-progress slash/reward accounting.                       |
| `votetiming`, `votingconfig` | Global timing and dependency/threshold configuration.      |
