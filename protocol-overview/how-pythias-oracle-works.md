# How Pythia's Oracle Works

Pythia's optimistic oracle lets contracts request and receive arbitrary data quickly and cheaply. It works as a generalized **escalation game** between the parties who assert data and Pythia's dispute-resolution backstop, the **Data Verification Mechanism (DVM)**.

The key idea: **most answers are never voted on.** An answer is accepted optimistically after a challenge window (its _liveness_) unless someone disputes it. Only disputed answers escalate to the DVM, where PYTHIA stakers vote on the correct outcome. This keeps the common case fast and inexpensive while preserving a decentralized, economically secured court of last resort.

```
                proposes / asserts            no dispute before liveness
   Requester  ───────────────────►  Answer  ──────────────────────────►  Settled
                                       │
                                       │  disputed within liveness
                                       ▼
                                 DVM (pythiavoting)  ──►  stakers vote  ──►  Settled
```

## The two layers

### 1. The optimistic layer — `pythiaoorcle`

The optimistic oracle verifies data quickly and optimistically. It is secured by the DVM because any answer can be escalated there for arbitration. The lifecycle has two equivalent shapes:

**OOv3 (assertions).** A single party — the **asserter** — posts a bonded claim about the state of the world in one action:

* **claim** — arbitrary bytes describing the fact being asserted (up to the ancillary-data limit, 8 KB by default).
* **currency** — the token used for the bond. Must be whitelisted in `<finder>`.
* **bond** — the stake the asserter puts behind the claim being correct. Must be at least the minimum bond (see [Bonds, Fees & Liveness](../developers/bonds-fees-and-liveness.md)).
* **liveness** — the challenge window; if it elapses with no dispute, the assertion is accepted as true.
* **identifier** / **escalation manager** / **domain** — optional parameters controlling how a dispute resolves.

**OOv2 (request → propose).** The requester and proposer are separate. A **requester** opens a price request (identifier, timestamp, ancillary data, reward); a **proposer** answers it by posting a bond; the proposal then enters its liveness window.

In both shapes:

1. A **disputer** can refute the answer within the liveness window by posting a matching bond.
2. If no one disputes before liveness expires, the answer is optimistically treated as correct and settled.
3. If disputed, the request is escalated to the DVM for arbitration.

### 2. The dispute layer — the DVM (`pythiavoting`)

The DVM is a token-weighted **commit-reveal** voting system that resolves disputes escalated from the optimistic layer.

1. On dispute, the oracle sends a price request to `pythiavoting`, stamped with the assertion/dispute context so voters and tooling can identify it.
2. The request is voted on in the next round. Voting is two-phase: a **commit** phase (voters submit a hash of their vote) followed by a **reveal** phase (voters reveal price + salt). This prevents copying and last-look manipulation.
3. Votes are weighted by each voter's **staked PYTHIA**, snapshotted at commit time so stake cannot be added mid-round to swing a result.
4. The DVM aggregates the revealed votes. If a single answer commands a supermajority and quorum thresholds are met, that answer resolves the request; otherwise the request **rolls** to the next round.
5. Correct voters are rewarded from the pool slashed from incorrect and absent voters; the oracle reads the resolved price and settles the original request.

The DVM is powerful because it introduces **human judgment** — voters reference the identifier's off-chain methodology to decide what the answer _should_ have been — while remaining trust-minimized through stake weighting and slashing.

## Why it's secure

Pythia's oracle is built around one economic invariant:

$$
\text{Cost of Corruption} > \text{Profit from Corruption}
$$

Corrupting an outcome requires controlling enough staked PYTHIA to swing a DVM vote, and doing so burns stake through slashing while the honest majority is rewarded. As long as the value that could be stolen by a false resolution stays below the cost of acquiring and risking that stake, honest resolution is the rational strategy. See [Economic Security](economic-security.md) for the full model.

## Where to go next

* **[Optimistic Oracle: v2 and v3](optimistic-oracle-v2-and-v3.md)** — the two flows in detail, and how to choose.
* **[The DVM](the-dvm.md)** — commit-reveal, snapshots, quorum, and resolution.
* **[Staking, Rewards & Slashing](staking-rewards-and-slashing.md)** — how voters are incentivized.
