# Insurance / Arbitration (OOv2)

An insurance contract pays out when an insured event occurs. The hard part is deciding, trust-minimized, **whether the event actually happened**. Pythia's optimistic oracle is the arbitrator.

## The pattern

1. A policy is created with premium, coverage, and a precise description of the insured event.
2. When a policyholder files a claim, the contract **requests** a yes/no determination from Pythia, encoding the claim in `ancillary_data`.
3. A proposer answers (claim valid / invalid). If undisputed, it settles optimistically; if disputed, the DVM arbitrates.
4. On settlement, the contract pays the claim if the resolved answer is "valid", or rejects it otherwise.

This turns a subjective "did this happen?" question into a bonded, disputable, DVM-backstopped determination.

## Encoding the claim

Be exhaustive about what qualifies and what evidence resolves it. Voters and disputers rely entirely on `ancillary_data`:

```
claim: Policy #4471 flight AA123 on 2026-07-01 was delayed >3h at arrival per <official source>.
res: 10000=claim valid (pay), 0=claim invalid (deny)
```

## Requesting arbitration

```bash
# Escrow the reward for a proposer, then request the determination
cleos push action <token> transfer '["myinsurer","<oracle>","2.0000 OOVP","request:<request_id>"]' -p myinsurer@active

cleos push action <oracle> requestprice '{
  "requester":"myinsurer",
  "identifier":<claim_identifier>,
  "timestamp":0,
  "ancillary_data":"claim: Policy #4471 ...",
  "currency":{"sym":"4,OOVP","contract":"<token>"},
  "reward":"2.0000 OOVP"
}' -p myinsurer@active
```

A `timestamp` of `0` marks the request **event-based** (the claim is about whether an event occurred, not a value at a specific time).

## Reacting to the result

Enable the settled callback and pay/deny on resolution:

```cpp
[[eosio::action]]
void myinsurer::pricesettled(uint64_t identifier, uint32_t timestamp,
                             std::vector<char> ancillary_data, int128_t price) {
    require_auth(oracle_acct);
    // find the claim by (identifier, ancillary_data)
    if (price == 10000) {         // valid → pay the policyholder
        // transfer coverage
    } else {                       // invalid → deny
        // close the claim
    }
}
```

## Why bonds matter here

Insurance is a classic target for oracle manipulation — a large fraudulent claim can be worth more than the bond. Size bonds and liveness so that **profit from a false "valid" resolution < cost of winning the dispute you'd have to win**:

* Set the request **bond** proportional to the coverage at risk (`setbond`).
* Give honest disputers time to gather evidence (`setliveness`).
* Consider `refund_on_dispute` so the reward returns to you if the claim is contested (`setrefund`).

See [Economic Security](../../protocol-overview/economic-security.md).

## Optimistic arbitrator variant

If you want a single trusted party to answer quickly but still be disputable, you can have that party be the proposer while anyone retains the right to dispute to the DVM. This "optimistic arbitrator" pattern gives fast, cheap resolution in the honest case with the full DVM backstop when contested — the same guarantees the base OOv2 flow provides, just with a designated proposer.
