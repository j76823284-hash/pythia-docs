# Economic Security

Pythia's oracle is not secured by trusting any single party — it is secured by making dishonesty **more expensive than the profit it could produce**. This page explains the guarantee and the levers that maintain it.

## The core invariant

$$
\text{Cost of Corruption (CoC)} > \text{Profit from Corruption (PfC)}
$$

* **Profit from Corruption (PfC)** — the value an attacker could extract by forcing a false resolution (e.g. the funds locked in a contract that would pay out on a wrong answer).
* **Cost of Corruption (CoC)** — what it costs an attacker to control enough of the system to force that answer: acquiring a majority of staked OOVP, and the stake destroyed by slashing when the honest minority resolves against them.

As long as CoC > PfC for every integration, honest resolution is the rational equilibrium.

## The two lines of defense

### 1. Bonds (the optimistic layer)

Every optimistic answer is bonded. Proposing or asserting requires posting a bond; disputing requires posting a matching bond. The loser of a dispute forfeits their bond, part of which is burned to the fee vault. This means:

* **Spam is not free.** An unfounded assertion or a frivolous dispute risks a bond.
* **Disputing is profitable when you are right.** A correct disputer takes the wrong proposer's bond, which incentivizes an honest watcher set to challenge bad answers — you don't need Pythia to police answers; the bond market does.

The minimum bond scales with the final fee so that the economic stake is meaningful relative to the request. See [Bonds, Fees & Liveness](../developers/bonds-fees-and-liveness.md).

### 2. Stake and slashing (the DVM layer)

If an answer is disputed, it escalates to the DVM, where the outcome is decided by staked OOVP:

* Votes are **stake-weighted**, so influence requires capital at risk.
* **Wrong voters are slashed**, and the slashed stake is **paid to correct voters** — so an attacker fighting the honest majority bleeds stake to the very people resolving against them.
* **Absent stakers are slashed** too (passively), keeping participation high so the honest majority is actually present.

To corrupt a resolution an attacker must out-vote the honest stake **and** absorb the slashing losses of doing so, round after round if the request rolls.

## The levers that tune security

Governance can adjust these parameters to keep CoC > PfC as usage grows:

| Lever | Contract | Effect on security |
|---|---|---|
| **Final fee** (`setfinalfee`) | `oovp.store` | Raises the minimum bond, increasing the cost of spam and frivolous disputes. |
| **Bond size** (`setbond` / assertion bond) | `oovp.oracle` | Larger bonds raise the stake behind each answer. |
| **Liveness** (`setliveness` / assertion liveness) | `oovp.oracle` | Longer windows give honest disputers more time to catch bad answers. |
| **Burned-bond rate** (`setburnedbps`) | `oovp.oracle` | Sets how much of a losing bond is taken as an oracle fee vs. paid to the winner. |
| **Emission rate** (`setemission`) | `oovp.stake` | Higher rewards attract more honest stake, raising CoC. |
| **Slash rates** | `oovp.stake` | Higher penalties make attacking and free-riding costlier. |
| **GAT / SPAT** (`setthreshold`) | `oovp.voting` | Require broader participation/agreement before a high-value dispute resolves. |

## Guidance for integrators

Because Pythia is generic, **you** are responsible for sizing your integration so an attack is never profitable:

1. **Bond ≳ value at risk.** Set bonds and liveness such that the cost of winning a dispute you shouldn't win exceeds what your contract would pay out on a wrong answer.
2. **Choose liveness for your threat model.** Fast-moving markets want longer liveness so honest disputers can react; low-stakes data can use short liveness for speed.
3. **Whitelist your collateral and identifier.** Only whitelisted currencies and supported identifiers are accepted; use tokens with deep, non-manipulable markets.
4. **Assume the DVM is the final word.** Design your contract to read the settled price and act on it — never to depend on a specific proposer being honest.

## Fee flow and the token

Losing bonds and settlement fees flow to the oracle **fee vault**, and protocol fees collected in `oovp.store` are split between burning and treasury (default 50/50). Burning OOVP ties protocol usage to token scarcity, while the treasury funds ongoing operations. Emissions flow the other way, paying honest stakers. The net design keeps a large, honest, well-compensated staker set standing between any attacker and a false resolution.
