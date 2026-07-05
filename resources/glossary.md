# Glossary

Key terms used throughout Pythia's documentation.

**Ancillary data** — Arbitrary bytes attached to a request describing the question and its resolution methodology. Voters and disputers rely on it to determine the correct answer. Capped at the oracle's `ancillary_bytes_limit` (default 8 KB).

**Assertion (OOv3)** — A bonded claim about the state of the world posted in a single action (`asserttruth`). Accepted as true if not disputed before liveness expires.

**Asserter** — The party who posts an assertion and its bond.

**ASSERT_TRUTH** — The built-in binary identifier (`0x4153534552545455`) used for truth assertions and governance. Resolves to `NUMERICAL_TRUE` (10000) or `NUMERICAL_FALSE` (0).

**Basis points (bps)** — Hundredths of a percent; 10000 bps = 100%. Most rates in Pythia are expressed in bps.

**Bond** — Collateral posted behind an answer (or a dispute). Forfeited if you lose; recovered (plus the loser's, minus the oracle fee) if you win.

**Burned-bond rate** — The fraction of a losing bond taken as an oracle fee on a disputed settlement (default 50%, `setburnedbps`). Also drives the OOv3 minimum bond.

**Callback** — An inline action the oracle fires to an integrating contract on propose/dispute/settle (OOv2: `priceproposed`/`pricedisputed`/`pricesettled`) or dispute/resolution (OOv3: `assertdisp`/`assertresvd`).

**Collateral** — A whitelisted token used for rewards and bonds. Represented as an `extended_symbol` (symbol + contract).

**Commit-reveal** — Two-phase voting: submit a hash of your vote (commit), then reveal price + salt. Prevents copying and last-look manipulation.

**Cost of Corruption (CoC)** — What an attacker must spend/risk to force a false resolution. Must exceed the Profit from Corruption.

**Crank** — A permissionless action anyone can call to advance protocol state (settle, resolve, slash, prune), since Antelope has no scheduled transactions.

**Delegate** — An account authorized to vote on a staker's behalf while the staker keeps custody and rewards.

**DVM (Data Verification Mechanism)** — Pythia's stake-weighted commit-reveal voting layer (`pythiavoting`) that resolves disputes.

**Emissions** — Protocol rewards paid to stakers continuously from the emissions vault (default 0.05 PYTHIA/sec).

**Escalation manager** — A pluggable arbitration policy slot on OOv3 assertions. Reserved on-chain; the shipped path routes disputes to the DVM.

**Extended symbol** — An Antelope `(symbol, contract)` pair. Required because WAX has no global token namespace.

**Final fee** — A flat per-collateral fee (`pythiastore1`) that anchors bond and minimum-bond math.

**Finder** — The registry (`<finder>`) mapping interface names to accounts and holding the identifier/collateral/contract whitelists.

**GAT (Global Access Threshold)** — Minimum participation for a DVM result to count: revealed weight ≥ `gat_percentage` of the round's frozen stake (default 5%).

**Governor** — The governance contract (`pythiagovern`).

**Identifier** — A `uint64` key selecting the resolution methodology for a request/assertion.

**Liveness** — The challenge window during which an answer can be disputed (default 2 h; 30 s–7 d).

**Minimum bond** — `finalFee × 10000 / burnedBondBps`; the smallest bond an OOv3 assertion may post (2× final fee at the default 50% burn).

**Mode** — The most-voted price by weight in a DVM round. Must exceed 50% of revealed weight to win.

**OOv2** — The request → propose → dispute → settle flow.

**OOv3** — The assert → dispute → settle flow.

**PYTHIA** — The protocol token (4 decimals). Used for staking, bonds, and rewards.

**Optimistic oracle** — An oracle that accepts answers optimistically after a challenge window, escalating only disputes to a voting backstop.

**Passive slash** — A small penalty (default 0.05%) charged to stakers who did not participate in a resolved request, applied lazily.

**Profit from Corruption (PfC)** — The value an attacker could extract from a false resolution.

**Proposer** — In OOv2, the party who answers a request and posts a bond.

**Roll** — Re-queuing a DVM request to the next round when it fails to resolve (up to `max_rolls`, default 3).

**Rounds & phases** — The DVM's fixed schedule: each round is a commit phase followed by a reveal phase (default 24 h each).

**Slashing** — Removing stake from voters who vote incorrectly or don't vote; redistributed to correct voters.

**SPAT (Staker Participation / agreement Threshold)** — Minimum agreement on the mode for a result to count (default 50%, in bps).

**Stamp** — Context appended to a dispute's DVM request. OOv3: `assertionId:<hex>,ooAsserter:<name>`. OOv2: `disputed by <requester> against <proposer>`.

**Vote-locking (veToken)** — Locking stake for a fixed duration to earn up to a 3× voting-power multiplier.
